# AgentCore executor + Step Functions orchestrator for the fa dark factory

- Date: 2026-10-08
- Status: partly superseded (2026-10-09) — Phase 1 (AgentCore executor inside GitHub Actions), Phase 2 (Step Functions orchestrator) and the M1–M5 milestones are replaced by `2026-10-09-aws-orchestrator-switch-design.md`. M0, account isolation (§6), security (§7) and decisions (§10) stay in force.
- Owner: rustembuild

## 1. Context and goal

Purpose: an EPAM AI Technology Manager assessment — build a **dark factory**
and develop an app with it. The factory reuses fa (this fork) and IstiN's
dmtools factory (`IstiN/dmtools-agentic-workflows`, `IstiN/dmtools-agents`),
whose own runbook defines the term
(`dmtools-agents/.github/skills/setup-dark-factory/SKILL.md`): tracker intake →
SM agent → specialised agents → PR → required checks → label approval → merge.

Today every leg (dev / review / rework) runs entirely on a GitHub-hosted
runner VM inside `factory-teammate.yml`. This design adds:

1. **Phase 1 — AgentCore executor:** a leg can run in an Amazon Bedrock
   AgentCore Runtime microVM, selected by one config line.
2. **Phase 2 — Step Functions orchestrator:** the dev → review → rework →
   merge loop as an explicit state graph, as an alternative to the GitHub
   Actions label/dispatch state machine.

### Success criteria

- `executor: 'agentcore'` in a consumer's `.dmtools/config.js` makes its legs
  run in AgentCore; removing it restores today's behaviour exactly.
- A throwaway **web snake game** repo is built by the factory end to end
  (issue → merged PR) with legs executing in AgentCore (M3a — factory canary).
- The assessment app, a **web-based cinema ticket booking** app, is built by
  the factory the same way (M3b — the minimum assessment demo).
- Phase 2: the same loop driven by Step Functions, visible in the console.
- No one but the owner can cause spend in the owner's AWS account (§6).
- Metrics per leg: duration, AgentCore cost, LLM tokens, review rounds.

### Non-goals

- **No PRs to IstiN's upstream repos.** All factory changes live in the
  owner's forks.
- GitLab orchestrator, Azure sandbox executor — the contract allows them; not
  built here.
- VPC-isolated networking (listed as future hardening).

### Honest constraints (for the presentation)

- AgentCore sessions are fixed at **2 vCPU / 8 GB** (non-adjustable); the
  free public-repo `ubuntu-24.04` runner has 4 vCPU / 16 GB. CPU-bound work
  (flutter analyze/test) may be slower; the win is startup (pre-baked
  toolchain image) and per-use billing while the agent waits on the LLM.
- Session max lifetime 8 h; idle timeout 15 min (kept alive via
  `HealthyBusy`); container image ≤ 2 GB; ARM64 only.
- AgentCore is a runtime, not an orchestrator — loops live in the
  orchestrator (GitHub Actions or Step Functions).

## 2. Architecture: two plug points, one contract

```
ORCHESTRATOR (decides the next leg)      GitHub Actions (IstiN's factory) | Step Functions (phase 2)
        │  LegSpec
        ▼
EXECUTOR (where one leg runs)            inline (CI runner VM, default) | agentcore (phase 1)
        │  LegResult
        ▼
run-leg.sh — the leg's work steps, identical in every executor
```

- The orchestrator never knows how a leg runs; the executor never decides
  what runs next.
- GitHub labels/comments remain the human-visible record. Under Step
  Functions the execution is the source of truth and labels mirror it.

### 2.1 Control vs work steps

`factory-teammate.yml` (job 2) is split:

| Control (stays in orchestrator) | Work (moves into `run-leg.sh`) |
|---|---|
| event guard / leg derivation | fetch factory-setup + kit, resolve parent config |
| mark job started / done | install fa + dmtools (no-op when pre-baked) |
| apply review verdict | checkout, git author + push credentials |
| close review / rework cycle | restore fa session, write issue/PR into input folder |
| publish token usage | run agent (`dmtools run` with the runner json) |
| | persist fa project memory, quarantine poisoned session, collect crash log |

The `inline` executor runs `run-leg.sh` as a step — behaviour identical to
today. The `agentcore` executor runs the same script inside the microVM.

### 2.2 Contract

`LegSpec` and `LegResult` are JSON documents validated against one shared JSON
Schema (`schema/leg-spec.schema.json`, `schema/leg-result.schema.json` in the
dmtools-agents fork).

`LegSpec`: `repo`, `ref`, `leg` (dev|review|rework), `issue` | `pr`,
`runnerConfigPath`, `factoryRef`, `dmtoolsVersion`, `faVersion`, `attempt`,
`sessionCacheKey`, `callback` (`{type: "s3"}` or
`{type: "sfn", taskToken}`), `maxSeconds`.

`LegResult`: `status` (`ok` | `agent_failed` | `infra_error`), `branch`,
`prNumber`, `verdict` (review legs: the existing verdict JSON), `tokenUsage`,
`durationSeconds`, `logsUrl` (private CloudWatch link), `exitCode`.

### 2.3 Config

```js
// consumer .dmtools/config.js
module.exports = {
  machineAuthor: 'ai-teammate',
  executor: 'agentcore',            // 'inline' (default) | 'agentcore'
  sm: { runners: { /* unchanged */ } }
};
```

AWS details come from repository variables (non-secret):
`AGENTCORE_RUNTIME_ARN`, `AGENTCORE_ROLE_ARN` (OIDC role),
`AGENTCORE_REGION`, `AGENTCORE_BUCKET`, `AGENTCORE_ENABLED` (kill switch).
The orchestrator is not a config key: it is whichever trigger is deployed,
with `ORCHESTRATOR=stepfunctions` making the GitHub Actions stub exit early.

## 3. Phase 1 — AgentCore executor

### 3.1 Leg-runner image (`fa-leg-runner`)

- ARM64, listens on `0.0.0.0:8080`, exposes `POST /invocations` and
  `GET /ping` (AgentCore HTTP protocol contract).
- **Base layer:** a small HTTP shim, git, gh, fa, dmtools CLI, `run-leg.sh`.
  Built and pushed to ECR by CI when fa/dmtools versions change.
- **Toolchain layer:** per target app, so the base stays generic. For the
  canary and the cinema app: Python 3.12 + uv (FastAPI backend) and Node LTS
  (React + Vite frontend). Total ≤ 2 GB.

### 3.2 Leg lifecycle

1. The orchestrator assumes the AWS role via GitHub OIDC (no stored AWS keys)
   and calls `InvokeAgentRuntime` with the `LegSpec`; session id is
   deterministic: `<repo>-<issue|pr>-<leg>-<attempt>` (padded to the
   service's minimum id length).
2. `/invocations` validates the spec, replies `{accepted}`, starts
   `run-leg.sh` in the background; `/ping` returns `HealthyBusy` until it
   ends, then `Healthy`.
3. `run-leg.sh` streams logs to CloudWatch and delivers the `LegResult`:
   - `callback.type = s3` → writes `s3://<bucket>/legs/<sessionId>/result.json`;
   - `callback.type = sfn` → `SendTaskSuccess` / `SendTaskFailure`.
4. GitHub Actions waiting step (`wait-agentcore-leg`) polls S3 for
   `result.json`, printing status lines only (§6.5), and fails at
   `maxSeconds`.

### 3.3 Secrets

Never in the payload or the image. The runtime execution role reads from
Secrets Manager: the fine-grained GitHub token, `ZAI_CODE_KEY`,
`KIMI_REVIEW_KEY`. `run-leg.sh` exports them into the agent's environment.

### 3.4 Session / memory cache

`setup/cache.sh` gains a storage backend switch: GitHub cache (inline) or S3
(agentcore), same keys. A ticket can change executor and still resume its fa
conversation.

### 3.5 Failures

| Situation | Result | Orchestrator action |
|---|---|---|
| fa exits non-zero / red build | `agent_failed` | existing behaviour (as inline) |
| no result by `maxSeconds`, session crash, 8 h cap | `infra_error` | retry once with `attempt+1`, then label `needs-human` |
| `AGENTCORE_ENABLED=false` | leg not started | job exits with a clear message |

Re-running is safe: pushes go to the leg's own branch; session id includes
the attempt.

## 4. Phase 2 — Step Functions orchestrator

Standard workflow, one execution per ticket, name `gh-<issue>-<generation>`.

```
MarkStarted → DevLeg → ReviewLeg → Choice(verdict)
                          ▲          ├ approved            → WaitForChecks → Merge → Done
                          │          ├ changes & round < N → ReworkLeg ─┐
                          └──────────┼──────────────────────────────────┘
                                     └ changes & round ≥ N → NeedsHuman (wait) → resume
```

- **Legs:** `.waitForTaskToken` tasks; a Lambda starts the AgentCore session
  with `callback = {type: sfn, taskToken}`; `HeartbeatSeconds` /
  `TimeoutSeconds` catch dead sessions (`infra_error` → retry once → human).
- **Waits on humans / CI:** `NeedsHuman` and `WaitForChecks` are task-token
  waits completed by the webhook bridge (`/resume` comment or approval from
  an allowlisted user; green `check_suite`).
- **Reuse:** GitHub side effects (mark started/done, apply verdict, close
  cycle) import the existing JS modules from the dmtools-agents fork inside
  Node Lambdas. `MAX_AUTO_REWORK_ROUNDS` becomes the loop bound N.
- **One execution per ticket:** the bridge refuses to start if one is
  running for the issue (replaces the GitHub concurrency group).
- **Coexistence:** `ORCHESTRATOR=stepfunctions` makes `ai-teammate.yml` exit
  early; flipping it back restores the GitHub Actions loop.

## 5. Repositories

| Repo | Contents |
|---|---|
| `rustembuild/dmtools-agentic-workflows` (fork, never PR'd) | `run-leg.sh` extraction, executor switch in `factory-teammate.yml`, `wait-agentcore-leg` step, cache backend switch |
| `rustembuild/dmtools-agents` (fork, never PR'd) | `executor` key in `configLoader.js`, LegSpec/LegResult schemas, unit tests |
| `rustembuild/dark-factory-aws` (new) | leg-runner Dockerfile + shim (Dart, same toolchain as fa; S3 via the aws CLI in the image — Dart has no official AWS SDK); Terraform applied only through GitHub Actions: ECR, AgentCore Runtime, S3, Secrets, IAM, Budgets, Step Functions, Lambdas, API Gateway + WAF |
| `rustembuild/flutter_agent_harness_agentcore` (this fork) | first test consumer: `uses:` → own fork, the config line, `agentcore` environment |
| new canary repo (snake game) | throwaway web app to shake out the factory, set up via the setup-dark-factory runbook |
| new app repo (cinema booking) | the assessment app, same setup |

Fork hygiene: changes go mostly into new files with small hook points in
existing ones; consumers pin `uses:` to a fork commit SHA (not `@main`);
weekly `git merge upstream/main` into the forks.

## 6. Shared AWS account — never touch df-agentcore

The AWS account (its ID is kept out of public repos and passed via
variables) already hosts the owner's earlier dark factory, `df-agentcore`
(eu-central-1; tag `Project=df-agentcore`; names `darkfactory-*`,
`df-agentcore-*`; CI role `GitHubActionsRole`; Terraform state in
`df-agentcore-state-<account>`). This project must not change or depend on
any of it beyond reading the shared GitHub OIDC provider.

- **Region fence:** everything regional lives in **us-east-1**; df-agentcore
  is entirely in eu-central-1.
- **Names and tags:** every resource is prefixed `fa-ac-` (`fa_ac_` where
  AWS forbids hyphens) and tagged `Project=fa-agentcore` through provider
  `default_tags`. IAM roles and policies live under path `/fa-ac/`.
- **Shared OIDC provider:** AWS allows one per URL per account; df-agentcore
  created it. This project only **reads** it (Terraform `data` source) and
  never manages it.
- **Own state:** Terraform state in a new bucket `fa-ac-state-<account>`
  (us-east-1, versioned, S3-native lock). Never df-agentcore's bucket.
- **Own CI role `fa-ac-ci`, not AdministratorAccess:** allowed only in
  us-east-1 for regional services; IAM only under `/fa-ac/` and every role
  it creates must carry the `fa-ac-boundary` permissions boundary; explicit
  Deny on resources tagged `Project=df-agentcore`, on names `darkfactory-*`,
  `df-agentcore-*`, `GitHubActionsRole`, and on eu-central-1.
- **Terraform through GitHub Actions only**, matching df-agentcore's rule —
  except the one-time bootstrap (state bucket + `fa-ac-ci` role), which the
  owner applies locally once with admin credentials because CI cannot create
  its own role. It touches no df-agentcore resource.
- **Budget scoped by region:** the account budget would count df-agentcore's
  spend, so the $20 alert / $50 stop budget filters on Region `us-east-1`
  (df-agentcore is entirely eu-central-1; the M0 preflight confirms nothing
  else spends in us-east-1). A region filter needs no cost-allocation-tag
  activation and does not depend on AgentCore propagating tags to billing.
  The hard stop attaches deny-all only to this project's roles.
- **Teardown** destroys only this project's state; `prevent_destroy` stays
  on the state bucket.

## 7. Security — public repo, private AWS bill

Assume any agent session can be fully taken over by prompt injection.

1. **Who can start a leg:** only `workflow_dispatch` and `issues: assigned`
   (both need write/triage access); never `pull_request_target` or
   `issue_comment`. Keep the existing `ai-teammate` author gate; add an actor
   allowlist (owner + bot).
2. **Who gets AWS credentials:** GitHub OIDC only. The IAM trust policy
   accepts only
   `repo:rustembuild/flutter_agent_harness_agentcore:environment:agentcore`
   (and the app repo's equivalent). The `agentcore` environment is limited to
   `main`, optionally with required reviewer.
3. **Blast radius:** the shared-account fences of §6. Execution role: read 3 named secrets, write its S3 prefix and log group,
   `states:SendTaskSuccess/Failure` — nothing else. GitHub token: fine-grained,
   single repo, no admin, `main` protected. LLM keys with provider spend caps.
4. **Spend caps:** max 2 concurrent legs, max N legs/day, runtime
   `maxLifetime` = observed longest leg + margin; AWS Budgets alert at $20 and
   a budget action at $50 attaching deny-all to the execution role;
   `AGENTCORE_ENABLED=false` kill switch.
5. **Public logs:** Actions logs are public — the waiting step prints only
   status lines; agent output stays in private CloudWatch.
6. **Webhook (phase 2):** verify GitHub HMAC signature, then repo and sender
   allowlist, before any execution starts; WAF rate limit.

## 8. Testing

- **Refactor parity:** same canary issue through the `inline` executor before
  and after extracting `run-leg.sh`; existing dmtools-agents unit tests pass;
  new tests for the `executor` config key.
- **Shim:** run the ARM64 image locally (the owner's machine is ARM64):
  `/ping` states, accept + background run, `result.json` written, task-token
  callback against a stub.
- **Contract:** both sides validate against the shared schemas.
- **Infrastructure:** `terraform test` with mocked providers in CI asserts
  trust subjects, the shared-account fences (§6) and budget values.
- **Security negatives:** fork / PR OIDC subject denied (IAM policy
  simulator); bad webhook signature and non-allowlisted sender dropped;
  budget action attaches deny; kill switch stops new legs.
- **E2E:** a canary issue through dev → review → rework → merge on each
  orchestrator.

## 9. Milestones (~5 weeks)

| | Milestone | Done when |
|---|---|---|
| M0 | Bootstrap in the shared account, fenced CI role, budget, kill switch; hello-world shim deployed through CI | cold start measured, limits confirmed, df-agentcore unchanged |
| M1 | `run-leg.sh` extracted in the fork | inline parity canary green |
| M2 | leg-runner image, agentcore executor, S3 cache | this fork's legs run in AgentCore via the one config line |
| M3a | canary repo: factory builds a web snake game (GHA orchestrator + AgentCore executor) | issue → merged PR → playable game, no human code |
| **M3b** | app repo: factory builds the web cinema ticket booking app | **minimum assessment demo** |
| M4 | Step Functions orchestrator + webhook bridge | canary loop visible in the console |
| M5 | metrics (cost/leg, rounds, lead time), hardening, presentation | assessment-ready |

## 10. Decisions

- Web stack (canary + cinema app): Python FastAPI backend + React (Vite) frontend.
- AWS region: us-east-1 (shared account; df-agentcore stays in eu-central-1).
- Infrastructure as code: Terraform through GitHub Actions (not CDK), matching df-agentcore.
- AWS Budgets: alert at $20/month, hard stop (deny-all action) at $50/month.
  Notifications go to the subscriber-less SNS topic `fa-ac-budget` (owner
  choice 2026-10-09: no alert email; the stop acts on its own).
- All Terraform, the bootstrap included, applies through CI (`deploy.yml`:
  bootstrap → foundation → image → runtime). The bootstrap runs as
  `fa-ac-bootstrap`, which cannot change itself; changing that role is the
  only local owner apply left.
- Public repos: `main` is pull-request-only, outside contributors' workflow
  runs need approval, interaction limits stay at collaborators-only
  (renewed every 6 months), secret scanning + push protection are on.
- App repos (M3a snake, M3b cinema) are private: only the owner can file or
  comment on the issues agents read. Agents still run only for allowlisted
  assigners/authors and read only allowlisted comments (M1 requirement).
- Conventions: Conventional Commits; org-wide PR template and `.gitmessage`
  in the public `rustembuild/.github`; PR titles checked in CI; semantic
  releases (tag + notes, nothing committed back) in our own repos —
  dark-factory-aws now, the app repos from M3a. The fa fork keeps
  upstream's release pipeline.
