# AWS orchestrator switch for the fa dark factory — design

Date: 2026-10-09
Status: draft for owner review
Supersedes: Phase 1 (AgentCore executor inside GitHub Actions) and Phase 2
(Step Functions orchestrator) of
`2026-10-08-agentcore-executor-design.md`. Its M0 AWS foundation, account
isolation (§6), security baseline (§7) and decisions (§10) stay in force.

## 1. Goal

The owner (EPAM AI Technology Manager assessment) builds a "dark factory" by
reusing IstiN's fa + dmtools machine factory. One config line per app repo
selects the orchestrator:

- `github-actions` — today's upstream factory, untouched.
- `aws` — AWS Step Functions orchestrates, Amazon Bedrock AgentCore runs the
  legs.

Both modes stay working on the same app repo so the assessment can compare
them on the same issues.

### Success criteria

1. Flipping the one line moves a repo between modes; only one orchestrator
   is ever active.
2. In `aws` mode an assigned issue becomes a merged PR with no GitHub Actions
   runner doing orchestration or agent work (CI on PRs stays on GitHub
   Actions).
3. The factory's business logic — rules, labels, caps, agent packs, prompts,
   verdict handling — is IstiN's code, unchanged; parity tests prove the
   ported leg steps decide the same as upstream.
4. Measured per mode on the same issue set (snake canary): cost per leg,
   wake-up latency, lead time issue → merge, rework rounds.
5. Cinema ticket booking app (FastAPI + React) built by the factory: the
   minimum assessment demo.

### Non-goals

- Re-implementing the factory's rules as Step Functions states (rejected
  approach B).
- Moving CI, branch protection or the tracker off GitHub.
- Any pull request to IstiN's repositories. Changes live in the owner's
  forks (`rustembuild/*`).

## 2. What is reused, adapted and new

| Piece | Source | Status |
|---|---|---|
| dmtools CLI (`epam/dmtools-dart`, linux-arm64 build) | upstream release | reused unchanged |
| SM engine `smAgent.js`, rules `sm_github.json`, `githubSource`, `factoryState`, `common/ci.js` | IstiN/dmtools-agents | reused unchanged |
| Merge bot `machine_merge` (`js/sm/mergeBot.js`) | IstiN/dmtools-agents | reused unchanged |
| Agent packs `story_development`, `bug_development`, `pr_review`, `pr_rework` + hooks, verdict parser | IstiN/dmtools-agents | reused unchanged |
| fa CLI (coding agent) | IstiN/flutter_agent_harness | reused unchanged |
| Runner JSONs, `.dmtools/config.js` format, label taxonomy, caps | app repo / upstream | reused; config gains one key |
| git-push guard, session/memory persistence to git | IstiN/dmtools-agentic-workflows kit | reused unchanged |
| GitHub-mode workflows (`factory-*.yml`) | IstiN/dmtools-agentic-workflows | reused unchanged |
| `scripts/run-teammate-local.sh` (install + `dmtools run` without GHA) | IstiN/dmtools-agents | reused as the leg runner core |
| Guard, verdict apply, cycle close, job markers from `factory-teammate.yml` | IstiN/dmtools-agentic-workflows | ported to the leg runner, parity-tested |
| `smProvider.dispatchLeg` / `activeMachineRuns` | IstiN/dmtools-agents | adapted in `rustembuild/dmtools-agents` (aws branch) |
| GitHub App, webhook receiver, tick + leg state machines, locks, Terraform | new | new, in `rustembuild/dark-factory-aws` |

## 3. The switch

`.dmtools/config.js` in the app repo:

```js
module.exports = {
  machineAuthor: 'ai-teammate',
  orchestrator: 'github-actions', // or 'aws'; absent = 'github-actions'
  sm: { runners: { bug, story, review, rework } } // unchanged
};
```

- `github-actions`: no behaviour change anywhere.
- `aws`: the four GitHub Actions stubs in the app repo (`machine-sm`,
  `ai-teammate`, `machine-merge`, `sm-kicker`) exit at their first step when
  the config says `aws`, so GitHub never dispatches legs or ticks. The AWS
  side ignores repos whose config does not say `aws` (the tick reads the
  config from the default branch before acting).
- A flip takes effect on the next wake-up. Legs already running finish in the
  mode that started them.

## 4. Architecture (aws mode)

```
GitHub (issues/PRs/labels/CI) ──webhook──▶ API Gateway ─▶ fa-ac-webhook (Lambda)
        ▲                                                        │ start / mark dirty
        │                         EventBridge Scheduler (15 min) ┤
        │                                                        ▼
        │                                         Step Functions fa-ac-tick
        │                                           lock repo (DynamoDB)
        │                                           fa-ac-sm-tick (Lambda container):
        │                                             dmtools run sm_github   (unchanged)
        │                                             dmtools run machine_merge (unchanged)
        │◀── labels / merge / CI dispatch ───────────   └─ dispatchLeg ─▶ fa-ac-dispatch-leg
        │                                           unlock; loop once if dirty      │
        │                                                                           ▼
        │                                         Step Functions fa-ac-leg (one per leg)
        │                                           lock item (DynamoDB)
        │                                           InvokeAgentRuntime + task token
        │                                           wait for callback, heartbeat 15 min, cap 4 h
        └── PR / comments / labels / verdict ◀──── AgentCore session: run-leg.sh
                                                    unlock; wake a tick
```

The factory is level-triggered: every tick re-derives state from GitHub, so
events only shorten the wait. A lost event costs latency, never correctness.

## 5. Components

### 5.1 GitHub App `rustembuild-factory`

- Owned by the `rustembuild` organization, installed on the selected app
  repos only. Not the existing df App.
- Repository permissions: issues, pull requests, contents, checks, actions
  (read/write); metadata (read).
- Webhook events: `issues`, `issue_comment`, `pull_request`,
  `pull_request_review`, `pull_request_review_comment`, `check_run`,
  `check_suite`, `workflow_run`.
- Private key and webhook secret: Secrets Manager `fa-ac-github-app`. CI
  writes the values from GitHub environment secrets; nothing is set by hand.
- Tokens: the tick and every leg mint an installation token scoped to the
  one app repo, valid at most 1 hour. They replace the PAT
  (`SOURCE_GITHUB_TOKEN`) and the "silent" `github.token` (CI is dispatch-only,
  so update-branch pushes by the App trigger no CI).

### 5.2 Webhook receiver `fa-ac-webhook`

- API Gateway HTTP API, route `POST /github`, throttled (burst 20, rate 5/s).
- Lambda in Dart (AOT, `provided.al2023`, arm64).
- Order of checks, each one drop-and-200 on failure:
  1. `X-Hub-Signature-256` HMAC against the webhook secret (constant-time).
  2. Installation ID equals the configured one.
  3. Repository full name is in the allowlist.
  4. Event type is in the list of §5.1.
- Never reads further event content. Effect: start `fa-ac-tick` for the repo;
  if the repo lock is held, set the repo's `dirty` flag instead.

### 5.3 Tick state machine `fa-ac-tick` (Standard)

1. Acquire the repo lock: DynamoDB `fa-ac-locks`, item `repo#<owner/name>`,
   conditional put with a 20-minute TTL. Held → set `dirty`, end.
2. Run Lambda `fa-ac-sm-tick` (container image, arm64, 15-minute timeout):
   - fetch an installation token;
   - check out `.dmtools/config.js` from the default branch; stop unless
     `orchestrator: 'aws'`;
   - `dmtools run sm_github` with the same `jobParams` the GitHub stub passes
     today (`repo`, `ciWorkflow`, `validationChecks`, `machineAuthor`,
     `statePublish`, `parallelWorkers`), the App token as source and silent
     token, and `DMTOOLS_PACK_REGISTRY` pinned to an `AGENTS_VERSION` release;
   - `dmtools run machine_merge`.
3. Release the lock. If `dirty` was set during the run, clear it and go to 1
   (at most 3 passes per execution).
- EventBridge Scheduler starts a tick every 15 minutes per allowlisted repo
  (safety net, replaces cron + `factory-sm-kicker`).

### 5.4 Engine adapter (fork `rustembuild/dmtools-agents`)

- `js/common/smProvider.js`: when the consumer config says
  `orchestrator: 'aws'`:
  - `dispatchLeg({issue|pr, leg, reason, ref})` runs
    `fa-ac-dispatch-leg` through `cli_execute_command`;
  - `activeMachineRuns()` runs `fa-ac-active-legs` and returns the same shape
    the GitHub branch returns (so in-flight dedup and `workflowBudget` work
    unchanged).
- No rule, guard, limit, label or pack changes. Unit tests in the engine's
  existing harness (`js/unit-tests`).
- The packs are published as a release of the fork; `DMTOOLS_PACK_REGISTRY`
  points at it in aws mode. GitHub mode keeps IstiN's registry.

### 5.5 Leg tools `fa-ac-dispatch-leg`, `fa-ac-active-legs` (Dart, in the tick image)

- `fa-ac-dispatch-leg`: starts `fa-ac-leg` with execution name
  `leg-<repo-slug>-<issue|pr>-<n>-<leg>-<epoch>` and input
  `{repo, anchor, leg, reason, ref}`. Returns success if the item lock is
  already held (the SM treats the leg as in flight).
- `fa-ac-active-legs`: lists `RUNNING` executions of `fa-ac-leg` for the repo
  and maps them to `<config>:<ticket>` keys.

### 5.6 Leg state machine `fa-ac-leg` (Standard)

1. Acquire the item lock: `fa-ac-locks`, item `item#<repo>#<anchor>`,
   conditional put, TTL 5 hours. Held → end with `Busy`.
2. `InvokeAgentRuntime` on `fa_ac_leg_runner` with the LegSpec (§5.7) and a
   task token (`waitForTaskToken`), `HeartbeatSeconds` 900,
   `TimeoutSeconds` 14400 (today's 240-minute cap).
3. Retry once on `infra_error` (session failed to start, runtime throttled).
   No retry on `agent_failed`: the rules decide what happens next.
4. Always: release the item lock, start a tick for the repo.
5. On final failure: comment on the anchor with the reason and the execution
   link; no label changes beyond what the leg runner already applied.

### 5.7 Leg runner (AgentCore image)

- The M0 Dart shim gains the LegSpec fields `taskToken`, `repo`, `anchor`,
  `leg`, `reason`, `ref`. It sends `SendTaskHeartbeat` every 5 minutes and
  `SendTaskSuccess`/`SendTaskFailure` at the end; the S3 `result.json`
  stays as the audit record.
- `run-leg.sh`, in order:
  1. Guard (ported from `factory-teammate.yml` `guard` job): resolve the
     runner from `sm.runners`, exactly one anchor, derive the leg from labels
     when empty, skip on `blocked`, owner-override check for manual
     `agent:rework`, allowlist check (§7).
  2. Install/run via `run-teammate-local.sh` (IstiN's): checkout, seed the
     agent input from the anchor, restore the fa session, `dmtools run
     <runner.json>` with the same env (`AI_AGENT_PROVIDER=fa`, provider queue,
     keys from Secrets Manager `fa-ac-llm-*`, App token as `GH_TOKEN`), the
     git-push guard first on `PATH`, the trace staleness watchdog.
  3. Post steps (ported, run even after failure): token usage, job done
     marker, apply the review verdict (same fallback chain), close the
     review/rework cycle, persist fa memory and session to git, link the run.
- Image: the M0 image plus the dmtools CLI, fa CLI, `gh`, git, pinned by
  version and checksum. Size stays under the 2 GB AgentCore limit.

### 5.8 Infrastructure (repo `rustembuild/dark-factory-aws`, Terraform via CI)

- New: API Gateway, `fa-ac-webhook`, `fa-ac-sm-tick` (+ ECR repo),
  `fa-ac-tick`, `fa-ac-leg`, `fa-ac-locks`, Scheduler schedules, the
  `fa-ac-github-app` and `fa-ac-llm-*` secrets, one role per component.
- The leg invoker role trusts `states.amazonaws.com` (from the `fa-ac-leg`
  state machine) instead of the fork's GitHub OIDC subject.
- The fence (bootstrap) is extended for `apigateway`, `dynamodb`,
  `scheduler`, `states`, `lambda`, `iam:PassRole` — all limited to `fa-ac-*`
  names in us-east-1. The new roles are added to the fence's known-role list.
- The $50 hard stop also covers the new execution roles.

## 6. Error handling

| Failure | Behaviour |
|---|---|
| Webhook lost, GitHub outage | 15-minute safety tick re-derives state |
| Two wake-ups at once | Repo lock admits one; the other sets `dirty` |
| Same leg dispatched twice | Item lock + unique execution name; the SM also sees it via `activeMachineRuns` |
| Leg hangs | Missed heartbeat after 15 min fails the leg; lock released; rules' caps and escalation apply |
| Session fails to start | One retry, then `infra_error` comment on the anchor |
| Agent fails, bad PR, red CI | Unchanged rules: red-head cap 3, rework rounds 2, `needs-human`, `blocked` |
| Budget reaches $50 | Hard stop denies the runtime/invoker/execution roles; legs fail; the next tick comments that the factory is paused |
| Lock left behind by a crash | TTL expiry (20 min repo, 5 h item) |
| Mode flipped mid-flight | Running legs finish; the next wake-up honours the new mode |

## 7. Security

- Webhook: signature first, then installation, repo and event allowlists,
  throttling; content never trusted.
- Who can start agents (aws-mode guard): the actor who assigned or labelled
  and the issue author must be on `AGENTCORE_ALLOWED_ACTORS` (repo variable,
  read at leg start); comments by others are dropped from the agent input.
  App repos are private.
- Tokens are App installation tokens, one repo, ≤ 1 hour. No PAT in aws mode.
- Everything is Terraform through CI; the fence keeps all new resources
  inside `fa-ac-*` names in us-east-1; df-agentcore and sbx-databricks-demo
  stay denied.

## 8. Testing

- Dart unit tests: signature verification (good, bad, missing), filters,
  dirty-flag coalescing, execution naming, active-leg mapping, LegSpec
  parsing, heartbeat/callback calls (fake client).
- Engine adapter unit tests in `rustembuild/dmtools-agents`.
- Parity tests: fixtures (issue/PR state, labels, review outputs) run
  through the ported guard and verdict steps and through the upstream
  workflow's logic; decisions must match.
- State machines: AWS `TestState` per state (lock busy, heartbeat timeout,
  infra retry, unlock in the finally path).
- Terraform: `terraform test` per root; IAM simulator checks on the extended
  fence.
- End to end on the private snake canary repo, same issues in both modes.

## 9. Milestones

| | Milestone | Done when |
|---|---|---|
| M0 (finish) | Direct runtime smoke from the owner's CLI (30 s and ~1000 s legs), findings doc | result in S3; idle timeout survived |
| M1 | Leg path: image with dmtools + fa, `run-leg.sh`, `fa-ac-leg`, locks, parity tests | a leg started by hand on a canary issue opens a PR |
| M2 | Event path: GitHub App, webhook, `fa-ac-tick` + engine Lambda, engine adapter, the switch, stubs skip in aws mode | assigning an issue → merged PR, aws mode end to end |
| M3a | Snake canary in both modes | same issue set, numbers per mode |
| M3b | Cinema booking app via the factory | minimum assessment demo |
| M4 | Metrics and comparison, hardening, presentation | assessment-ready |

The fork's GitHub smoke workflow and its `agentcore` environment are retired:
in aws mode GitHub never calls AgentCore.

## 10. Risks

- Porting the GitHub-loop steps of `factory-teammate.yml` (guard, verdict,
  cycle close) is the largest piece; parity tests gate it.
- The dmtools pack registry must be mirrorable to the owner's fork release
  (`DMTOOLS_PACK_REGISTRY` override is supported by the workflows; confirm
  in M1).
- Upstream changes to `smProvider.js` must be merged into the fork; keep the
  adapter small and isolated.
- One month is tight: if M2 slips, M3b runs in whichever mode is solid and
  the comparison uses the snake numbers.

## 11. Decisions

- Orchestrator switch `github-actions | aws`, both kept working (owner,
  2026-10-09).
- Events via a GitHub App webhook, not a GitHub Actions forwarder or polling
  (owner, 2026-10-09).
- Approach A: unchanged engine as a step; AWS replaces wake-up, leg
  execution, locking and tokens only (owner, 2026-10-09).
- Leg runner built on IstiN's `run-teammate-local.sh`.
- CI stays on GitHub Actions in both modes.
- Coding agent in aws-mode legs: Claude Code on the owner's Claude
  subscription (`claude setup-token` → `CLAUDE_CODE_OAUTH_TOKEN` in the LLM
  secret), not fa (owner, 2026-10-09). fa's Anthropic adapter is API-key
  only, and reusing a subscription outside Claude Code is not allowed. A repo
  picks the agent with `agentProvider: 'fa' | 'claude-code'` in
  `.dmtools/config.js` (default `fa`); IstiN's packs run unchanged through
  his `claude-code` provider plus a `claude` shim in the leg image.
- Target repos need CI: the review verdict labels `pr_approved` only after
  the PR's checks are green (learned in M1; see dark-factory-aws
  `docs/m1-findings.md`).
- Leg-runner base image needs glibc ≥ 2.38 (dmtools' QuickJS library); the
  image uses `debian:trixie-slim` (learned in M1).
