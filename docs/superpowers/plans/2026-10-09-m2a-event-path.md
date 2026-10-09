# M2a — Event Path Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Labelling or assigning a canary issue leads to a merged PR with no human and no local AWS login: GitHub App webhook → receiver → tick (IstiN's engine + merge bot, unchanged logic) → `fa-ac-leg`, plus a 15-minute safety tick.

**Architecture:** One Dart binary (`events`) runs as two container Lambdas: `fa-ac-webhook` (behind an API Gateway HTTP API) verifies GitHub deliveries, starts dev legs on issue events and wakes the tick; `fa-ac-sm-tick` runs `dmtools run sm_github` and `dmtools run machine_merge` for one repo. A Step Functions state machine `fa-ac-tick` serialises ticks per repo with a DynamoDB lock and a dirty flag; EventBridge Scheduler starts it every 15 minutes. IstiN's engine dispatches legs through a small overlay on `js/common/scm.js` that starts `fa-ac-leg` executions instead of GitHub workflow runs.

**Tech Stack:** Dart 3.12 (AOT, `package:crypto`), bash, Node 20 (engine unit tests only), Terraform 1.16.3 + aws provider 6.65.0, AWS Lambda (container, arm64), API Gateway HTTP API, Step Functions (JSONata), DynamoDB, EventBridge Scheduler, GitHub Actions.

**Spec:** `docs/superpowers/specs/2026-10-09-aws-orchestrator-switch-design.md` (this fork, branch `docs/agentcore-executor-spec`). Implementation repo: `rustembuild/dark-factory-aws` (`~/projects/dark-factory-aws`). M1 results: `docs/m1-findings.md` there.

## Rulings that refine the spec (recorded before execution)

- **R-A (adapter seam).** Spec §5.4 names `smProvider.dispatchLeg/activeMachineRuns`. At the pinned tag, `sm_github` (`js/smAgent.js`) dispatches through `js/common/scm.js` (`triggerWorkflow`, `listWorkflowRuns`); `smProvider`'s dispatch is only used by the older `machine_sm` pack. The overlay therefore adds an scm provider `github-sfn` that wraps the GitHub provider and overrides those two methods for `ai-teammate.yml` only.
- **R-B (no fork, overlay patch).** Spec §5.4 says "fork `rustembuild/dmtools-agents` and publish its packs". Instead, `dark-factory-aws/engine/` keeps a unified diff against the pinned tag plus its tests, and the events image builds a patched `sm_github` zip from IstiN's released zip. Same unchanged rules/guards/limits, no fork release machinery (its release workflow bumps `versions.json` on `main` and needs a PAT), and the patch fails loudly if upstream drifts. Legs keep IstiN's registry unchanged.
- **R-C (dev legs from the webhook).** Upstream starts dev legs from the consumer stub's `issues: [assigned, labeled]` trigger, not from the engine. The webhook receiver does the same: on `issues.assigned`, or `issues.labeled` with label `agent:dev|agent:review|agent:rework`, it starts `fa-ac-leg` directly (`leg` empty → guard derives it from labels, `sender` = event sender, `trigger` = the action). It reads only `action`, `label.name`, `issue.number`, `sender.login`, `repository.full_name`, `installation.id` from the event (spec §5.2 said "never reads further event content"; this is the minimum that preserves the upstream behaviour).
- **R-D (one image, container Lambdas, no SigV4).** dmtools' QuickJS library needs glibc ≥ 2.38, so `provided.al2023` (glibc 2.34) cannot run the tick. Both Lambdas run one `debian:trixie-slim` image (`fa-ac-events`) whose Dart binary implements the Lambda Runtime API and shells out to the `aws` CLI like the leg runner does — no SigV4 client.
- **R-E (roles).** New roles: `fa-ac-events` (both Lambdas) and `fa-ac-scheduler` (EventBridge Scheduler). `fa-ac-tick` reuses `fa-ac-leg-invoker` (its trust already covers `stateMachine:fa-ac-*`), extended with `lambda:InvokeFunction` on `fa-ac-sm-tick` and `states:StartExecution` on `fa-ac-tick`/`fa-ac-leg`. Both new roles join the fence's known-role list and the $50 hard stop.
- **R-F (fence compaction moves into M2a).** The fence renders to 6133 of 6144 characters; M2a adds `scheduler:*` and two role ARNs (~115 characters). Task 1 compacts it to ≤ 6000 characters after the additions.
- **R-G (CI validation).** `sm_github` validates PRs by dispatching `ciWorkflow` (`gh workflow run`) and reading `event=workflow_dispatch` runs. The canary's `ci.yml` gains `workflow_dispatch:`; the tick passes `ciWorkflow: 'ci.yml'` per repo.
- **R-H (merge bot self-tick).** `machine_merge` runs with `selfTick: false` (its self-tick dispatches `machine-sm.yml` on GitHub).
- **R-I (GitHub stubs).** Spec §3's "stubs skip in aws mode" applies to repos that carry IstiN's GitHub stubs; the canary has none. Deferred to M3a (both modes on one repo).
- **R-J (bot sender).** Review legs start from the `labeled` event the factory's own App adds (`agent:review`). The canary's `AGENTCORE_ALLOWED_ACTORS` becomes `agzyamov,rustembuild-factory[bot]`; strangers stay refused.

## Global Constraints

- All AWS changes go through Terraform applied by CI (`deploy.yml`); no `aws` write calls outside CI except starting/stopping executions for checks.
- Every AWS resource is named `fa-ac-*`, region `us-east-1`; IAM roles use path `/fa-ac/` and `permissions_boundary = local.boundary_arn`.
- Never touch df-agentcore (eu-central-1, `darkfactory*`/`df-agentcore*`) or `sbx-*` resources.
- No PRs to upstream repos (IstiN/*, epam/*). PRs only in `rustembuild/*`.
- Conventional Commits; PR title matches `^(feat|fix|docs|refactor|perf|test|build|ci|chore|revert)(\([a-z0-9._/-]+\))?!?: [^ ].{0,70}$`; commit trailer `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`.
- Actions pinned by full SHA (checkout `11d5960a326750d5838078e36cf38b85af677262`, setup-node `39370e3970a6d050c480ffad4ff0ed4d3fdee5af`, setup-terraform `b9cd54a3c349d3f38e8881555d616ced269862dd`, configure-aws-credentials as already used in `deploy.yml`).
- Pins: dmtools `v0.1.44` (sha256 `e6af54ff9228d221fad92da46fd2782110aa8b3722befb0132a1deb963c8cf5e`), gh `2.102.0`, IstiN/dmtools-agents tag `agents-rel-20261008-215056` (`sm_github` 0.1.36, `machine_merge` 0.1.35).
- Secrets never on argv or in logs; tokens via env/stdin; outputs through `redact` (runner/leg/lib.sh).
- Execution names (shared contract between Dart and JS): `leg-<slug>-<anchor>-<leg>-<epochMs>`, where `slug` = lower-cased `owner-repo` with every char outside `[a-z0-9-]` replaced by `-`, truncated to 40; `anchor` = `gh-<n>` or `pr-<n>`; `leg` ∈ `dev|review|rework|auto`. Tick executions: `tick-<slug>-<epochMs>`.
- Leg input (unchanged from M1 plus `trigger`): `{"repo":"o/n","issue":<int>|null,"pr":<int>|null,"leg":"dev|review|rework|"(""→auto),"reason":str,"ref":str,"sender":str,"trigger":"dispatch|labeled|assigned"}`.

## Review Focus

1. **Duplicate deliveries / bursts** (GitHub retries, label storms): at most one leg per anchor runs — the item lock and the engine's in-flight list (RUNNING executions) both hold; a second tick while one runs sets `dirty` instead of running twice. Tests: Task 4 (dirty path), Task 6 (list-executions → in-flight titles).
2. **Forged or foreign webhooks**: bad signature, other installation, repo not allowlisted, unknown event → 200 with no AWS call. Tests: Task 5.
3. **Factory's own label events**: `agent:review` added by the bot must start the review leg (sender `rustembuild-factory[bot]` allowlisted), while status-label churn (`status:*`, `ai_*`) starts nothing. Tests: Task 5.
4. **Engine overlay drift**: the patch must fail the build if IstiN's `scm.js` at the pinned tag changes shape, and non-`ai-teammate.yml` workflows (CI dispatch probes) must still go to GitHub. Tests: Task 3.
5. **Tick on a repo not in aws mode / config missing**: the tick exits cleanly without running the engine. Tests: Task 6.

---

### Task 1: Fence — compaction, Scheduler, two new roles (bootstrap)

**Files:**
- Modify: `terraform/bootstrap/fence.tf`
- Modify: `terraform/bootstrap/tests/bootstrap.tftest.hcl`

**Interfaces:**
- Produces: fence allows `scheduler:*` (us-east-1); `OnlyKnownRoles` lists `fa-ac-leg-runtime`, `fa-ac-leg-invoker`, `fa-ac-budget-action`, `fa-ac-events`, `fa-ac-scheduler`.

- [ ] **Step 1: Measure today's size.** Render the policy the way the infra reference did: copy `fence.tf` and a stub `locals.tf` (`account = "111111111111"`, `boundary_arn`, `bootstrap_role_arn` as in `main.tf`) into a scratch dir with an `output "fence" { value = local.fence_policy }`, run `terraform init && terraform apply -auto-approve`, then `terraform output -raw fence | python3 -c 'import json,sys; print(len(json.dumps(json.loads(sys.stdin.read()),separators=(",",":"))))'`. Expected: `6133`.

- [ ] **Step 2: Write the failing tests** in `terraform/bootstrap/tests/bootstrap.tftest.hcl` (add a run block; keep existing asserts, update the OnlyKnownRoles one):

```hcl
run "fence_fits_and_allows_m2a" {
  command = plan
  assert {
    condition     = length(jsonencode(jsondecode(local.fence_policy))) <= 6000
    error_message = "Fence must stay <= 6000 minified chars (limit 6144) to leave headroom."
  }
  assert {
    condition     = contains(flatten([for s in jsondecode(local.fence_policy).Statement : try(tolist(s.Action), [s.Action]) if s.Effect == "Allow"]), "scheduler:*")
    error_message = "Fence must allow scheduler:* (EventBridge Scheduler) in us-east-1."
  }
  assert {
    condition = toset([for s in jsondecode(local.fence_policy).Statement : s.NotResource if try(s.Sid, "") == "OnlyKnownRoles"][0]) == toset([
      for n in ["fa-ac-leg-runtime", "fa-ac-leg-invoker", "fa-ac-budget-action", "fa-ac-events", "fa-ac-scheduler"] :
      "arn:aws:iam::111111111111:role/fa-ac/${n}"
    ])
    error_message = "Only the five known roles may be created or retrusted."
  }
}
```

If the existing test file references Sids that Step 4 renames, update those references in the same edit.

- [ ] **Step 3: Run to verify it fails.** `cd terraform/bootstrap && terraform init -backend=false && terraform test`. Expected: FAIL on `scheduler:*` and the role list.

- [ ] **Step 4: Implement.** In `fence.tf`: add `"fa-ac-events", "fa-ac-scheduler"` to `fa_ac_role_names`; add `"scheduler:*"` to the `IdNamedServices` Allow action list. Then compact until the size assert passes, without weakening any Deny: shorten every Sid to ≤ 12 characters (e.g. `NeverTouchSbxDatabricksDemoByTag` → `NoSbxTag`, `NeverTouchDfAgentcoreByTag` → `NoDfTag`), drop `/*` duplicates where the same pattern already ends in `*` (e.g. `df-agentcore-*` and `df-agentcore-*/*` are both covered by `df-agentcore-*`), merge the two by-tag Deny statements into one statement whose Condition uses `"StringEquals": {"aws:ResourceTag/Project": ["df-agentcore", "sbx-databricks-demo"]}`. Keep the region conditions and every Deny's effect identical. Record the before/after size in the commit message.

- [ ] **Step 5: Run tests.** `terraform fmt -check -recursive && terraform validate && terraform test`. Expected: PASS. Re-run Step 1's measurement; expected ≤ 6000.

- [ ] **Step 6: Commit and PR.** Branch `feat/m2a-fence`, commit `feat(bootstrap): fence room for the event path (scheduler, events and scheduler roles)`, open the PR, wait for CI green, merge (bootstrap applies first in `deploy.yml`), and confirm the `bootstrap` job is green.

---

### Task 2: Foundation — events ECR repo, roles, webhook secret, hard stop

**Files:**
- Create: `terraform/foundation/events.tf`
- Modify: `terraform/foundation/iam.tf` (invoker policy), `terraform/foundation/budget.tf:274,294`, `terraform/foundation/secrets.tf`, `terraform/foundation/main.tf` (outputs)
- Modify: `.github/workflows/deploy.yml` (Sync secrets)
- Test: `terraform/foundation/tests/*.tftest.hcl` (add `events.tftest.hcl`)

**Interfaces:**
- Consumes: Task 1 fence.
- Produces (outputs, `sensitive = true`): `events_ecr_repository_url`, `events_role_arn`, `scheduler_role_arn`; secret container `fa-ac-webhook-secret` (SecretString = the raw webhook secret); invoker can `lambda:InvokeFunction` on `function:fa-ac-sm-tick` and `states:StartExecution` on `stateMachine:fa-ac-tick` and `stateMachine:fa-ac-leg`.

- [ ] **Step 1: Write the failing test** `terraform/foundation/tests/events.tftest.hcl` (mirror the existing mock_provider setup in that folder):

```hcl
run "event_path_foundation" {
  command = plan
  assert {
    condition     = aws_ecr_repository.events.name == "fa-ac-events" && aws_ecr_repository.events.image_tag_mutability == "IMMUTABLE"
    error_message = "fa-ac-events ECR repo must exist with immutable tags."
  }
  assert {
    condition     = aws_iam_role.events.name == "fa-ac-events" && aws_iam_role.events.path == "/fa-ac/" && aws_iam_role.events.permissions_boundary != null
    error_message = "fa-ac-events role must live under /fa-ac/ with the boundary."
  }
  assert {
    condition     = aws_iam_role.scheduler.name == "fa-ac-scheduler" && strcontains(aws_iam_role.scheduler.assume_role_policy, "scheduler.amazonaws.com")
    error_message = "fa-ac-scheduler role must trust EventBridge Scheduler."
  }
  assert {
    condition     = toset(aws_budgets_budget_action.hard_stop.definition[0].iam_action_definition[0].roles) == toset(["fa-ac-leg-runtime", "fa-ac-leg-invoker", "fa-ac-events", "fa-ac-scheduler"])
    error_message = "The $50 hard stop must cover the new roles."
  }
  assert {
    condition     = aws_secretsmanager_secret.webhook.name == "fa-ac-webhook-secret"
    error_message = "Webhook secret container must exist."
  }
}
```

- [ ] **Step 2: Run** `cd terraform/foundation && terraform init -backend=false && terraform test`. Expected: FAIL (resources undefined).

- [ ] **Step 3: Implement** `terraform/foundation/events.tf`:

```hcl
resource "aws_ecr_repository" "events" {
  name                 = "fa-ac-events"
  image_tag_mutability = "IMMUTABLE"
  image_scanning_configuration { scan_on_push = true }
}

resource "aws_ecr_lifecycle_policy" "events" {
  repository = aws_ecr_repository.events.name
  policy = jsonencode({ rules = [{ rulePriority = 1, description = "keep last 10",
    selection = { tagStatus = "any", countType = "imageCountMoreThan", countNumber = 10 },
  action = { type = "expire" } }] })
}

resource "aws_iam_role" "events" {
  name                 = "fa-ac-events"
  path                 = "/fa-ac/"
  permissions_boundary = local.boundary_arn
  assume_role_policy = jsonencode({ Version = "2012-10-17", Statement = [{
    Effect = "Allow", Principal = { Service = "lambda.amazonaws.com" }, Action = "sts:AssumeRole",
  Condition = { StringEquals = { "aws:SourceAccount" = local.account } } }] })
}

resource "aws_iam_role_policy" "events" {
  name = "fa-ac-events"
  role = aws_iam_role.events.id
  policy = jsonencode({ Version = "2012-10-17", Statement = [
    { Effect = "Allow", Action = ["logs:CreateLogGroup", "logs:CreateLogStream", "logs:PutLogEvents"],
    Resource = "arn:aws:logs:${local.region}:${local.account}:log-group:/aws/lambda/fa-ac-*" },
    { Effect = "Allow", Action = "secretsmanager:GetSecretValue",
    Resource = [aws_secretsmanager_secret.github_app.arn, aws_secretsmanager_secret.webhook.arn] },
    { Effect = "Allow", Action = "states:StartExecution", Resource = [
      "arn:aws:states:${local.region}:${local.account}:stateMachine:fa-ac-tick",
    "arn:aws:states:${local.region}:${local.account}:stateMachine:fa-ac-leg"] },
    { Effect = "Allow", Action = "states:ListExecutions",
    Resource = "arn:aws:states:${local.region}:${local.account}:stateMachine:fa-ac-leg" },
  ] })
}

resource "aws_iam_role" "scheduler" {
  name                 = "fa-ac-scheduler"
  path                 = "/fa-ac/"
  permissions_boundary = local.boundary_arn
  assume_role_policy = jsonencode({ Version = "2012-10-17", Statement = [{
    Effect = "Allow", Principal = { Service = "scheduler.amazonaws.com" }, Action = "sts:AssumeRole",
  Condition = { StringEquals = { "aws:SourceAccount" = local.account } } }] })
}

resource "aws_iam_role_policy" "scheduler" {
  name = "fa-ac-scheduler"
  role = aws_iam_role.scheduler.id
  policy = jsonencode({ Version = "2012-10-17", Statement = [{ Effect = "Allow", Action = "states:StartExecution",
  Resource = "arn:aws:states:${local.region}:${local.account}:stateMachine:fa-ac-tick" }] })
}
```

Use the resource names the existing `secrets.tf` gives the App secret (if it is not `aws_secretsmanager_secret.github_app`, use the real name). In `secrets.tf` add `aws_secretsmanager_secret.webhook` named `fa-ac-webhook-secret` with the same recovery window as the others. In `iam.tf` add to the invoker policy:

```hcl
{ Effect = "Allow", Action = "lambda:InvokeFunction", Resource = "arn:aws:lambda:${local.region}:${local.account}:function:fa-ac-sm-tick" },
{ Effect = "Allow", Action = "states:StartExecution", Resource = [
  "arn:aws:states:${local.region}:${local.account}:stateMachine:fa-ac-tick",
  "arn:aws:states:${local.region}:${local.account}:stateMachine:fa-ac-leg"] },
```

In `budget.tf` add `aws_iam_role.events.name` and `aws_iam_role.scheduler.name` to both the budget-action role's `iam:AttachRolePolicy` resource list (L274) and the hard stop's `roles` (L294). Add the three outputs to `main.tf` (`sensitive = true`). In `deploy.yml`'s "Sync secrets" step add `sync fa-ac-webhook-secret-raw` handling: the webhook secret is not JSON, so add a separate branch — if `FA_AC_WEBHOOK_SECRET` (env secret, mapped from `secrets.FA_AC_WEBHOOK_SECRET`) is non-empty and differs from the stored value, `aws secretsmanager put-secret-value --secret-id fa-ac-webhook-secret --secret-string file://<0600 temp file>`; never echo it.

- [ ] **Step 4: Run** `terraform fmt -check -recursive && terraform validate && terraform test`. Expected: PASS.

- [ ] **Step 5: Owner-free secret.** The controller generates the webhook secret: `openssl rand -hex 32 | gh secret set FA_AC_WEBHOOK_SECRET --repo rustembuild/dark-factory-aws --env aws-deploy` (value never printed).

- [ ] **Step 6: Commit, PR, merge after green CI, confirm deploy green** and that `aws secretsmanager describe-secret --secret-id fa-ac-webhook-secret` shows a `LastChangedDate`. Commit: `feat(foundation): events image repo, roles and webhook secret`.

---

### Task 3: Engine overlay — `github-sfn` scm provider (patch + tests + pack builder)

**Files:**
- Create: `engine/README.md`, `engine/aws-sfn.patch`, `engine/test_scmSfn.js`, `engine/build-packs.sh`, `engine/pins.env`
- Modify: `.github/workflows/test.yml` (new job `engine`)

**Interfaces:**
- Consumes: IstiN/dmtools-agents at `agents-rel-20261008-215056`.
- Produces: `engine/build-packs.sh <outdir>` writes `<outdir>/sm_github-0.1.36-aws.zip` (+`.sha256`) and `<outdir>/machine_merge-0.1.35.zip` (+`.sha256`). In the patched engine, `createScm({scm:{provider:'github-sfn'}, repository:{owner,repo}})` returns the GitHub provider with:
  - `triggerWorkflow(owner, repo, 'ai-teammate.yml', inputsJson, ref)` → runs `aws stepfunctions start-execution --state-machine-arn "$FA_AC_LEG_SM_ARN" --name <leg-name> --input '<json>'` via `cli_execute_command`;
  - `listWorkflowRuns(status, 'ai-teammate.yml', limit, owner, repo)` → for `status === 'in_progress'` returns `{workflow_runs: [{id, status:'in_progress', display_title:'▶ <leg> (SM) · <anchor>', created_at}]}` built from RUNNING executions whose name starts with `leg-<slug>-`; for any other status returns `{workflow_runs: []}`;
  - every other workflow file and every other method unchanged (GitHub).

- [ ] **Step 1: Fetch the pinned source** (scratch, not committed): `git clone --depth 1 --branch agents-rel-20261008-215056 https://github.com/IstiN/dmtools-agents.git /tmp/dta`. Read `js/common/scm.js` (1113 lines; `createScm` at L1081) and `js/unit-tests/node_testRunner.js` to learn its CLI (how to run one test file). Confirm in `js/smAgent.js` (L372 `createTargetScm`) that the project config's `scm` key reaches `createScm` (it copies `effectiveConfig`); record what you saw in `engine/README.md`.

- [ ] **Step 2: Write the failing test** `engine/test_scmSfn.js` using the harness globals (`suite`, `test`, `assert`, `loadModule`, `makeRequire`):

```js
// Tests for the github-sfn scm provider (dark-factory-aws overlay).
function load(mocks) {
    var mod = loadModule('js/common/scm.js', makeRequire({}, mocks), mocks);
    return mod.createScm({ scm: { provider: 'github-sfn' }, repository: { owner: 'rustembuild', repo: 'fa-canary-snake' } });
}

suite('scm github-sfn', function () {
    test('ai-teammate.yml dispatch starts a fa-ac-leg execution', function () {
        var cmds = [];
        var scm = load({ cli_execute_command: function (a) { cmds.push(a.command); return '{"executionArn":"x"}'; },
                         github_trigger_workflow: function () { throw new Error('must not call GitHub'); } });
        scm.triggerWorkflow('rustembuild', 'fa-canary-snake', 'ai-teammate.yml',
            JSON.stringify({ issue: '5', leg: 'review', reason: "sm: it's green" }), 'ai/gh-5');
        assert.equal(cmds.length, 1);
        var c = cmds[0];
        assert.contains(c, 'aws stepfunctions start-execution --state-machine-arn "$FA_AC_LEG_SM_ARN" --name leg-rustembuild-fa-canary-snake-gh-5-review-');
        assert.contains(c, "\"repo\":\"rustembuild/fa-canary-snake\"");
        assert.contains(c, '"issue":5');
        assert.contains(c, '"pr":null');
        assert.contains(c, '"ref":"ai/gh-5"');
        assert.contains(c, '"trigger":"dispatch"');
        assert.contains(c, "it'\\''s green");   // single quote escaped for /bin/sh
    });

    test('PR-anchored dispatch uses pr-N and issue null', function () {
        var cmd = '';
        var scm = load({ cli_execute_command: function (a) { cmd = a.command; return '{}'; } });
        scm.triggerWorkflow('rustembuild', 'fa-canary-snake', 'ai-teammate.yml',
            JSON.stringify({ issue: '', leg: 'rework', reason: 'r', pr: '77' }), 'main');
        assert.contains(cmd, '--name leg-rustembuild-fa-canary-snake-pr-77-rework-');
        assert.contains(cmd, '"issue":null');
        assert.contains(cmd, '"pr":77');
    });

    test('other workflows still dispatch on GitHub', function () {
        var gh = null;
        var scm = load({ cli_execute_command: function () { throw new Error('no aws for CI'); },
                         github_trigger_workflow: function (o, r, f) { gh = f; return {}; } });
        scm.triggerWorkflow('rustembuild', 'fa-canary-snake', 'ci.yml', '{}', 'main');
        assert.equal(gh, 'ci.yml');
    });

    test('in_progress lists RUNNING legs of this repo as runs with stub titles', function () {
        var listing = JSON.stringify({ executions: [
            { name: 'leg-rustembuild-fa-canary-snake-gh-5-review-1791558717000', startDate: '2026-10-09T15:00:00Z', executionArn: 'a1' },
            { name: 'leg-rustembuild-fa-canary-snake-pr-77-auto-1791558717001', startDate: '2026-10-09T15:01:00Z', executionArn: 'a2' },
            { name: 'leg-other-repo-gh-1-dev-1791558717002', startDate: '2026-10-09T15:02:00Z', executionArn: 'a3' },
            { name: 'm1-dev-5-1791554168', startDate: '2026-10-09T15:03:00Z', executionArn: 'a4' }
        ] });
        var scm = load({ cli_execute_command: function () { return listing; } });
        var res = scm.listWorkflowRuns('in_progress', 'ai-teammate.yml', 50, 'rustembuild', 'fa-canary-snake');
        var runs = res.workflow_runs;
        assert.equal(runs.length, 2);
        assert.equal(runs[0].display_title, '▶ review (SM) · gh-5');
        assert.equal(runs[1].display_title, '▶ auto (SM) · pr-77');
        assert.equal(runs[0].status, 'in_progress');
        assert.equal(runs[0].created_at, '2026-10-09T15:00:00Z');
    });

    test('other statuses report no legs (counted once, under in_progress)', function () {
        var scm = load({ cli_execute_command: function () { throw new Error('must not list'); } });
        assert.deepEqual(scm.listWorkflowRuns('queued', 'ai-teammate.yml', 50, 'rustembuild', 'fa-canary-snake'), { workflow_runs: [] });
    });

    test('a failed listing throws so the engine treats it as unknown, not idle', function () {
        var scm = load({ cli_execute_command: function () { throw new Error('AccessDenied'); } });
        assert.throws(function () { scm.listWorkflowRuns('in_progress', 'ai-teammate.yml', 50, 'rustembuild', 'fa-canary-snake'); });
    });
});
```

- [ ] **Step 3: Run it against unpatched source to see it fail.** `cp engine/test_scmSfn.js /tmp/dta/js/unit-tests/ && cd /tmp/dta && node js/unit-tests/node_testRunner.js js/unit-tests/test_scmSfn.js` (adjust to the runner's real argument convention found in Step 1). Expected: FAIL (provider `github-sfn` falls back to plain GitHub).

- [ ] **Step 4: Implement the overlay in `/tmp/dta/js/common/scm.js`**, then save it as `engine/aws-sfn.patch` (`cd /tmp/dta && git diff > ~/projects/dark-factory-aws/engine/aws-sfn.patch`). Add above `createScm`:

```js
// dark-factory-aws overlay (aws orchestrator): legs run as Step Functions
// executions of fa-ac-leg instead of ai-teammate.yml workflow runs.
var SFN_LEG_WORKFLOW = 'ai-teammate.yml';

function _sfnSlug(owner, repo) {
    return (String(owner) + '-' + String(repo)).toLowerCase().replace(/[^a-z0-9-]/g, '-').slice(0, 40);
}

function _sfnShellQuote(s) {
    return "'" + String(s).replace(/'/g, "'\\''") + "'";
}

function _createGithubSfnProvider(owner, repo) {
    var base = _createGithubProvider(owner, repo);
    var baseTrigger = base.triggerWorkflow;
    var baseList = base.listWorkflowRuns;
    base.triggerWorkflow = function (o, r, workflowFile, payload, ref) {
        if (workflowFile !== SFN_LEG_WORKFLOW) return baseTrigger(o, r, workflowFile, payload, ref);
        var inp = typeof payload === 'string' ? JSON.parse(payload || '{}') : (payload || {});
        var issue = inp.issue ? parseInt(inp.issue, 10) : null;
        var pr = inp.pr ? parseInt(inp.pr, 10) : null;
        var anchor = issue ? 'gh-' + issue : 'pr-' + pr;
        var leg = inp.leg || 'auto';
        var name = 'leg-' + _sfnSlug(o || owner, r || repo) + '-' + anchor + '-' + leg + '-' + Date.now();
        var input = JSON.stringify({ repo: (o || owner) + '/' + (r || repo), issue: issue, pr: issue ? null : pr,
            leg: inp.leg || '', reason: inp.reason || '', ref: ref || '', sender: '', trigger: 'dispatch' });
        return cli_execute_command({ command: 'aws stepfunctions start-execution --state-machine-arn "$FA_AC_LEG_SM_ARN" --name ' +
            name + ' --input ' + _sfnShellQuote(input) + ' --output json' });
    };
    base.listWorkflowRuns = function (status, workflowId, limit, o, r) {
        if (workflowId !== SFN_LEG_WORKFLOW) return baseList(status, workflowId, limit, o, r);
        if (status !== 'in_progress') return { workflow_runs: [] };
        var prefix = 'leg-' + _sfnSlug(o || owner, r || repo) + '-';
        var raw = cli_execute_command({ command: 'aws stepfunctions list-executions --state-machine-arn "$FA_AC_LEG_SM_ARN" --status-filter RUNNING --max-items 100 --output json' });
        var text = (raw && (raw.output || raw.stdout)) || raw;
        var data = typeof text === 'string' ? JSON.parse(text) : text;
        var runs = [];
        (data.executions || []).forEach(function (e) {
            if (String(e.name).indexOf(prefix) !== 0) return;
            var m = /-((?:gh|pr)-\d+)-(dev|review|rework|auto)-\d+$/.exec(e.name);
            if (!m) return;
            runs.push({ id: e.executionArn, status: 'in_progress', display_title: '▶ ' + m[2] + ' (SM) · ' + m[1],
                created_at: e.startDate, updated_at: e.startDate, html_url: '' });
        });
        return { workflow_runs: runs };
    };
    return base;
}
```

and inside `createScm`, before `return _createGithubProvider(owner, repo);`:

```js
    if (provider === 'github-sfn') {
        return _createGithubSfnProvider(owner, repo);
    }
```

(If the test shows `cli_execute_command` returning an object, keep the `raw.output || raw.stdout` fallback; the engine reference says it returns trimmed stdout as a string.)

- [ ] **Step 5: Run the new test and IstiN's existing scm/smAgent tests** against the patched tree (`test_scmSfn.js`, plus whichever `test_scm*.js` / `test_smAgent.js` exist, through the node runner). Expected: new tests PASS; existing tests keep their pre-patch result (record any pre-existing failures in `engine/README.md` as baseline, so CI compares against it).

- [ ] **Step 6: Write `engine/pins.env` and `engine/build-packs.sh`:**

```bash
# engine/pins.env
AGENTS_TAG=agents-rel-20261008-215056
SM_GITHUB_VERSION=0.1.36
MACHINE_MERGE_VERSION=0.1.35
```

```bash
#!/usr/bin/env bash
# Builds the packs the tick runs: IstiN's released sm_github zip with the
# aws-sfn overlay applied to js/common/scm.js, and the unchanged machine_merge
# zip. Fails if the patch no longer applies or a released zip changed.
set -euo pipefail
here="$(cd "$(dirname "$0")" && pwd)"; source "$here/pins.env"
out="${1:?usage: build-packs.sh <outdir>}"; mkdir -p "$out"
work="$(mktemp -d)"; trap 'rm -rf "$work"' EXIT
base="https://github.com/IstiN/dmtools-agents/releases/download/$AGENTS_TAG"
fetch() {  # fetch <asset> -> verified file in $work
  curl -fsSLo "$work/$1" "$base/$1"; curl -fsSLo "$work/$1.sha256" "$base/$1.sha256"
  (cd "$work" && echo "$(awk '{print $1}' "$1.sha256")  $1" | sha256sum -c - >/dev/null)
}
fetch "sm_github-$SM_GITHUB_VERSION.zip"
fetch "machine_merge-$MACHINE_MERGE_VERSION.zip"
git -c advice.detachedHead=false clone -q --depth 1 --branch "$AGENTS_TAG" https://github.com/IstiN/dmtools-agents.git "$work/src"
(cd "$work/src" && git apply --check "$here/aws-sfn.patch" && git apply "$here/aws-sfn.patch")
mkdir "$work/sm" && (cd "$work/sm" && unzip -q "$work/sm_github-$SM_GITHUB_VERSION.zip")
cmp -s "$work/sm/js/common/scm.js" <(cd "$work/src" && git show HEAD:js/common/scm.js) \
  || { echo "released sm_github scm.js differs from the tag source" >&2; exit 1; }
cp "$work/src/js/common/scm.js" "$work/sm/js/common/scm.js"
python3 -I - "$work/sm" <<'PY'
import hashlib, json, pathlib, sys
root = pathlib.Path(sys.argv[1]); m = root / "manifest.json"; man = json.loads(m.read_text())
p = "js/common/scm.js"
man["files"][p] = hashlib.sha256((root / p).read_bytes()).hexdigest()
m.write_text(json.dumps(man, indent=2) + "\n")
PY
(cd "$work/sm" && zip -qrX "$out/sm_github-$SM_GITHUB_VERSION-aws.zip" .)
cp "$work/machine_merge-$MACHINE_MERGE_VERSION.zip" "$out/"
for z in "$out"/*.zip; do (cd "$out" && sha256sum "$(basename "$z")" | awk '{print $1}' > "$(basename "$z").sha256"); done
echo "packs: $(ls "$out" | tr '\n' ' ')"
```

Before writing the manifest code, open the released zip's `manifest.json` and confirm the `files` map format (path → sha256 hex). If the format differs, adapt the Python to it and say so in `engine/README.md`.

- [ ] **Step 7: Add CI job `engine`** to `.github/workflows/test.yml` (ubuntu-24.04; checkout; setup-node 20 pinned): clone the tag into `/tmp/dta`, `git apply engine/aws-sfn.patch`, copy `engine/test_scmSfn.js` into `js/unit-tests/`, run the node runner on it (must pass), then `bash engine/build-packs.sh /tmp/packs` (must succeed and list both zips).

- [ ] **Step 8: Commit, PR, merge after green.** Commit: `feat(engine): github-sfn scm overlay so the engine dispatches legs to fa-ac-leg`.

---

### Task 4: Dart — Lambda runtime loop and Step Functions starter

**Files:**
- Modify: `pubspec.yaml` (add `crypto: ^3.0.6` to dependencies), `lib/leg_runner.dart` (exports)
- Create: `lib/src/lambda_runtime.dart`, `lib/src/sfn.dart`
- Test: `test/lambda_runtime_test.dart`, `test/sfn_test.dart`

**Interfaces:**
- Consumes: `ProcessRunner` typedef from `lib/src/results.dart`.
- Produces:
  - `typedef LambdaHandler = Future<Object?> Function(Map<String, Object?> event);`
  - `Future<void> runLambdaLoop(LambdaHandler handler, {required String api, HttpClient? client, int? maxInvocations})` — `api` = `AWS_LAMBDA_RUNTIME_API` (host:port); GET `/2018-06-01/runtime/invocation/next`, then POST `/2018-06-01/runtime/invocation/<requestId>/response` with the JSON result, or `/error` with `{"errorMessage":..., "errorType":...}` when the handler throws.
  - `String slugOf(String repo)`; `String legExecutionName(String repo, {int? issue, int? pr, required String leg, required int nowMs})`; `String tickExecutionName(String repo, {required int nowMs})` — formats per Global Constraints.
  - `class SfnStarter { SfnStarter({required ProcessRunner run}); Future<bool> start(String stateMachineArn, String name, Map<String, Object?> input); }` — runs `aws stepfunctions start-execution --state-machine-arn <arn> --name <name> --input <json> --output json`; returns true on exit 0 or when stderr contains `ExecutionAlreadyExists`; false otherwise.

- [ ] **Step 1: Write failing tests.** `test/sfn_test.dart`:

```dart
import 'dart:convert';
import 'dart:io';
import 'package:leg_runner/leg_runner.dart';
import 'package:test/test.dart';

void main() {
  test('names follow the shared contract', () {
    expect(slugOf('RustemBuild/fa_canary.snake'), 'rustembuild-fa-canary-snake');
    expect(legExecutionName('rustembuild/fa-canary-snake', issue: 5, leg: 'review', nowMs: 17),
        'leg-rustembuild-fa-canary-snake-gh-5-review-17');
    expect(legExecutionName('o/r', pr: 9, leg: '', nowMs: 1), 'leg-o-r-pr-9-auto-1');
    expect(tickExecutionName('o/r', nowMs: 3), 'tick-o-r-3');
    expect(slugOf('a' * 30 + '/' + 'b' * 30).length, 40);
  });

  test('start passes input as one argv element and treats duplicates as started', () async {
    final calls = <List<String>>[];
    var stderr = '';
    final s = SfnStarter(run: (exe, args) async {
      calls.add([exe, ...args]);
      return ProcessResult(1, stderr.isEmpty ? 0 : 255, '{}', stderr);
    });
    expect(await s.start('arn:sm', 'n1', {'repo': 'o/r'}), isTrue);
    expect(calls.single, ['aws', 'stepfunctions', 'start-execution', '--state-machine-arn', 'arn:sm',
        '--name', 'n1', '--input', jsonEncode({'repo': 'o/r'}), '--output', 'json']);
    stderr = 'An error occurred (ExecutionAlreadyExists)';
    expect(await s.start('arn:sm', 'n1', {}), isTrue);
    stderr = 'AccessDenied';
    expect(await s.start('arn:sm', 'n1', {}), isFalse);
  });
}
```

`test/lambda_runtime_test.dart` — start a local `HttpServer` on `127.0.0.1:0` that serves two invocations (`Lambda-Runtime-Aws-Request-Id: r1` body `{"a":1}`, then `r2` body `{"boom":true}`) and records posts; handler returns `{'ok': event['a']}` or throws `StateError('bad')` when `boom`; run `runLambdaLoop(handler, api: '127.0.0.1:<port>', maxInvocations: 2)`; expect a POST to `/2018-06-01/runtime/invocation/r1/response` with body `{"ok":1}` and a POST to `/2018-06-01/runtime/invocation/r2/error` whose JSON has `errorType` `StateError` and an `errorMessage` containing `bad`.

- [ ] **Step 2: Run** `dart pub get && dart test test/sfn_test.dart test/lambda_runtime_test.dart`. Expected: FAIL (undefined names).

- [ ] **Step 3: Implement** `lib/src/sfn.dart`:

```dart
import 'dart:convert';
import 'results.dart';

String slugOf(String repo) {
  final s = repo.toLowerCase().replaceAll('/', '-').replaceAll(RegExp(r'[^a-z0-9-]'), '-');
  return s.length > 40 ? s.substring(0, 40) : s;
}

String legExecutionName(String repo, {int? issue, int? pr, required String leg, required int nowMs}) {
  final anchor = issue != null ? 'gh-$issue' : 'pr-$pr';
  return 'leg-${slugOf(repo)}-$anchor-${leg.isEmpty ? 'auto' : leg}-$nowMs';
}

String tickExecutionName(String repo, {required int nowMs}) => 'tick-${slugOf(repo)}-$nowMs';

class SfnStarter {
  SfnStarter({required ProcessRunner run}) : _run = run;
  final ProcessRunner _run;

  Future<bool> start(String stateMachineArn, String name, Map<String, Object?> input) async {
    final r = await _run('aws', ['stepfunctions', 'start-execution', '--state-machine-arn', stateMachineArn,
        '--name', name, '--input', jsonEncode(input), '--output', 'json']);
    return r.exitCode == 0 || '${r.stderr}'.contains('ExecutionAlreadyExists');
  }
}
```

`lib/src/lambda_runtime.dart`:

```dart
import 'dart:convert';
import 'dart:io';

typedef LambdaHandler = Future<Object?> Function(Map<String, Object?> event);

Future<void> runLambdaLoop(LambdaHandler handler, {required String api, HttpClient? client, int? maxInvocations}) async {
  final http = client ?? HttpClient();
  final base = 'http://$api/2018-06-01/runtime/invocation';
  for (var n = 0; maxInvocations == null || n < maxInvocations; n++) {
    final req = await http.getUrl(Uri.parse('$base/next'));
    final res = await req.close();
    final id = res.headers.value('lambda-runtime-aws-request-id') ?? '';
    final body = await utf8.decodeStream(res);
    Object? out;
    String? errType, errMsg;
    try {
      final ev = body.isEmpty ? <String, Object?>{} : jsonDecode(body);
      out = await handler(ev is Map<String, Object?> ? ev : <String, Object?>{'value': ev});
    } catch (e) {
      errType = e.runtimeType.toString();
      errMsg = e.toString();
    }
    final post = await http.postUrl(Uri.parse(errType == null ? '$base/$id/response' : '$base/$id/error'));
    post.headers.contentType = ContentType.json;
    post.write(jsonEncode(errType == null ? out : {'errorMessage': errMsg, 'errorType': errType}));
    await (await post.close()).drain<void>();
  }
}
```

Export both from `lib/leg_runner.dart`.

- [ ] **Step 4: Run** `dart analyze --fatal-infos && dart test`. Expected: PASS (all old tests too).

- [ ] **Step 5: Commit** `feat(events): Lambda runtime loop and Step Functions starter` (PR is opened at the end of Task 6 together with Tasks 5-6, branch `feat/m2a-events`).

---

### Task 5: Dart — webhook handler

**Files:**
- Create: `lib/src/webhook.dart`
- Test: `test/webhook_test.dart`

**Interfaces:**
- Consumes: Task 4 (`SfnStarter`, `legExecutionName`, `tickExecutionName`).
- Produces: `class WebhookConfig { final String secret; final int installationId; final Set<String> repos; final String tickArn; final String legArn; }` and `Future<Map<String, Object?>> handleWebhook(Map<String, Object?> apiEvent, WebhookConfig cfg, SfnStarter sfn, {int Function() nowMs})` returning an API Gateway v2 response `{'statusCode': 200, 'body': '<word>'}` where word ∈ `bad-signature|other-installation|repo-not-allowed|ignored-event|ok`.

Behaviour, in order (each failure → 200 with that word, no AWS call):
1. Body = `apiEvent['body']` (base64-decode when `isBase64Encoded == true`); header `x-hub-signature-256` (headers are lower-case in v2) must equal `'sha256=' + hex(HMAC-SHA256(secret, rawBody))`, compared in constant time.
2. JSON `installation.id` == `cfg.installationId`.
3. `repository.full_name` ∈ `cfg.repos`.
4. Header `x-github-event` ∈ `{issues, issue_comment, pull_request, pull_request_review, pull_request_review_comment, check_run, check_suite, workflow_run}`.
5. If event `issues` and (`action == 'assigned'` or (`action == 'labeled'` and `label.name` ∈ `{agent:dev, agent:review, agent:rework}`)): start `fa-ac-leg` with name `legExecutionName(repo, issue: n, leg: '', nowMs)` and input `{repo, issue: n, pr: null, leg: '', reason: 'webhook: issues.<action>', ref: '', sender: sender.login, trigger: action}`.
6. Always: start `fa-ac-tick` with name `tickExecutionName(repo, nowMs)` and input `{repo}`.
7. Return `ok`.

- [ ] **Step 1: Write failing tests** `test/webhook_test.dart` — helper `signed(Map payload, String event)` builds the apiEvent with the right signature (`Hmac(sha256, utf8.encode('s3cret'))`), a recording fake `SfnStarter` (subclass overriding `start`). Cases:
  1. bad signature → `bad-signature`, no starts;
  2. base64 body with a correct signature → `ok`;
  3. installation 999 → `other-installation`;
  4. repo `evil/x` → `repo-not-allowed`;
  5. event `push` → `ignored-event`;
  6. `issues.labeled` with `agent:dev` by `agzyamov` → two starts: leg (name `leg-rustembuild-fa-canary-snake-gh-5-auto-42`, input has `sender: agzyamov`, `trigger: labeled`, `leg: ''`, `issue: 5`) then tick (`tick-rustembuild-fa-canary-snake-42`, input `{repo}`);
  7. `issues.labeled` with `status:In Review` → only the tick;
  8. `issues.labeled` with `agent:review` by `rustembuild-factory[bot]` → leg + tick, sender kept verbatim;
  9. `pull_request.synchronize` → only the tick;
  10. a leg start that returns false still returns `ok` and still starts the tick (log a warning line without the payload).

  Use `nowMs: () => 42`, `cfg = WebhookConfig(secret: 's3cret', installationId: 169607698, repos: {'rustembuild/fa-canary-snake'}, tickArn: 'arn:tick', legArn: 'arn:leg')`.

- [ ] **Step 2: Run** `dart test test/webhook_test.dart`. Expected: FAIL.

- [ ] **Step 3: Implement** `lib/src/webhook.dart` (use `package:crypto` `Hmac`/`sha256`; constant-time compare: equal lengths and XOR-accumulate over code units; never log the body or the secret).

- [ ] **Step 4: Run** `dart analyze --fatal-infos && dart test`. Expected: PASS.

- [ ] **Step 5: Commit** `feat(events): GitHub webhook handler`.

---

### Task 6: Tick script, events entrypoint, events image

**Files:**
- Create: `runner/tick/tick.sh`, `bin/events.dart`, `Dockerfile.events`, `scripts/contract-events.sh`
- Create tests: `tests/tick/test_tick.sh` (reuses `tests/leg/harness.sh` and fakes `tests/leg/bin/{gh,aws,dmtools}`)
- Modify: `tests/leg/run-all.sh` (also run `tests/tick/test_*.sh`), `.github/workflows/test.yml` (build + contract-test the events image in the `dart` job), `.dockerignore` (keep `engine/` in the context)

**Interfaces:**
- Consumes: Task 3 packs (`/opt/packs/sm_github-0.1.36-aws.zip`, `/opt/packs/machine_merge-0.1.35.zip`), Task 4 runtime loop, Task 5 `handleWebhook`, `runner/leg/{lib.sh,app-token.sh}`.
- Produces:
  - `bin/events.dart`: `FA_AC_HANDLER=webhook` → loads `WebhookConfig` from env (`FA_AC_REPOS` JSON object keyed by repo, `FA_AC_INSTALLATION_ID`, `FA_AC_TICK_SM_ARN`, `FA_AC_LEG_SM_ARN`) and the secret from Secrets Manager (`aws secretsmanager get-secret-value --secret-id fa-ac-webhook-secret`, cached for the container's life); `FA_AC_HANDLER=tick` → runs `/app/tick/tick.sh` with the event JSON on stdin and returns its last stdout line parsed as JSON (non-zero exit → throw).
  - `tick.sh` stdin `{"repo":"o/n"}`, stdout last line `{"repo":..,"status":"ok|skipped","reason":..,"sm":<rc>,"merge":<rc>}`; exit 0 unless infra failure (secret/token), then exit 1.
  - Image `fa-ac-events`: `ENTRYPOINT ["/app/events"]`, user non-root, `HOME=/tmp/home`, binaries `aws`, `gh`, `git`, `curl`, `jq`, `openssl`, `dmtools` (+QuickJS lib), packs in `/opt/packs`.

- [ ] **Step 1: Write failing tick tests** `tests/tick/test_tick.sh` (source `tests/leg/harness.sh`; extend the fake `gh` rules as the leg tests do). Cases:
  1. repo not in `FA_AC_REPOS` → stdout `"status":"skipped"`, `"reason":"repo-not-allowed"`, no dmtools call;
  2. `.dmtools/config.js` without `orchestrator: 'aws'` (served by the fake `gh api repos/o/r/contents/.dmtools/config.js`) → `skipped`/`not-aws`;
  3. aws repo → fake dmtools log shows, in order, `run /opt/packs/sm_github-0.1.36-aws.zip` with override containing `"repo":"o/r"`, `"ciWorkflow":"ci.yml"`, `"machineAuthor":"rustembuild-factory[bot]"`, `"silentToken":"<token>"` masked in logs, then `run /opt/packs/machine_merge-0.1.35.zip` with `"selfTick":false`; the written `/tmp/.../work/.dmtools/config.js` ends with the overlay line `module.exports.scm = { provider: 'github-sfn' };`; `FA_AC_LEG_SM_ARN` and `HOME` reach dmtools' env; the token never appears in stdout;
  4. sm exit 3 → merge still runs, stdout `"sm":3`, script exit 0;
  5. App secret unreadable → exit 1.
  Use env `LEG_SECRETS=env APP_ID=1 INSTALLATION_ID=2 APP_PRIVATE_KEY=k LEG_TOKEN=t0k PACKS=/opt/packs` to bypass Secrets Manager and minting in tests, the same pattern as `factory-leg.sh`.

- [ ] **Step 2: Run** `bash tests/tick/test_tick.sh`. Expected: FAIL (script missing).

- [ ] **Step 3: Implement `runner/tick/tick.sh`:**

```bash
#!/usr/bin/env bash
# One SM tick for one repo: IstiN's sm_github (with the github-sfn overlay)
# then machine_merge, as the GitHub stubs run them (factory-sm.yml,
# factory-merge.yml). Reads {"repo":"o/n"} on stdin; last stdout line is JSON.
set -uo pipefail
here="$(cd "$(dirname "$0")" && pwd)"; leg="${LEG_LIB_DIR:-/app/leg}"
source "$leg/lib.sh"
PACKS="${PACKS:-/opt/packs}"; source "${ENGINE_PINS:-/opt/packs/pins.env}"
event="$(cat)"; REPO="$(jq -r '.repo // empty' <<<"$event")"
out() { jq -nc --arg repo "$REPO" --arg s "$1" --arg r "${2:-}" --argjson sm "${3:-null}" --argjson mg "${4:-null}" \
  '{repo:$repo,status:$s,reason:$r,sm:$sm,merge:$mg}'; }
cfg="$(jq -c --arg r "$REPO" '.[$r] // empty' <<<"${FA_AC_REPOS:-{\}}")"
[[ -n "$REPO" && -n "$cfg" ]] || { out skipped repo-not-allowed; exit 0; }
CI_WORKFLOW="$(jq -r '.ciWorkflow // "ci.yml"' <<<"$cfg")"

if [[ "${LEG_SECRETS:-aws}" == aws ]]; then
  app="$(aws secretsmanager get-secret-value --secret-id "${FA_AC_GITHUB_APP_SECRET:-fa-ac-github-app}" --query SecretString --output text 2>/dev/null)" \
    || { echo "tick: cannot read the GitHub App secret" >&2; exit 1; }
  APP_ID="$(jq -r '.app_id // empty' <<<"$app")"; INSTALLATION_ID="$(jq -r '.installation_id // empty' <<<"$app")"
  APP_PRIVATE_KEY="$(jq -r '.private_key // empty' <<<"$app")"; unset app
fi
export APP_ID INSTALLATION_ID APP_PRIVATE_KEY
if [[ -n "${LEG_TOKEN:-}" ]]; then TOKEN="$LEG_TOKEN"
else TOKEN="$(bash "$leg/app-token.sh" "$REPO")" || { echo "tick: cannot mint a token for $REPO" >&2; exit 1; }; fi
unset APP_PRIVATE_KEY
export REDACT_VARS="TOKEN GH_TOKEN SOURCE_GITHUB_TOKEN"
export GH_TOKEN="$TOKEN" SOURCE_GITHUB_TOKEN="$TOKEN" GH_REPO="$REPO" GITHUB_REPOSITORY="$REPO"
export HOME="${HOME:-/tmp/home}" GIT_TERMINAL_PROMPT=0 GH_PROMPT_DISABLED=1 GH_NO_UPDATE_NOTIFIER=1 GH_CONFIG_DIR=/tmp/gh
mkdir -p "$HOME"
work="$(mktemp -d)"; trap 'rm -rf "$work"' EXIT; mkdir -p "$work/.dmtools"; cd "$work"
gh api "repos/$REPO/contents/.dmtools/config.js" -H 'Accept: application/vnd.github.raw' > .dmtools/config.js 2>/dev/null \
  || { out skipped no-config; exit 0; }
grep -Eq "orchestrator:[[:space:]]*['\"]aws['\"]" .dmtools/config.js || { out skipped not-aws; exit 0; }
MACHINE_AUTHOR="$(sed -nE "s/.*machineAuthor:[[:space:]]*['\"]([][A-Za-z0-9._,-]+)['\"].*/\1/p" .dmtools/config.js | head -1)"
printf '\nmodule.exports.scm = { provider: %s };\n' "'github-sfn'" >> .dmtools/config.js

sm_override="$(jq -nc --arg repo "$REPO" --arg cw "$CI_WORKFLOW" --arg ma "$MACHINE_AUTHOR" --arg t "$TOKEN" \
  '{params:{jobParams:({repo:$repo,ciWorkflow:$cw,silentToken:$t,sourceToken:$t} + (if $ma=="" then {} else {machineAuthor:$ma} end))}}')"
dmtools run "$PACKS/sm_github-$SM_GITHUB_VERSION-aws.zip" "$sm_override" 2>&1 | redact >&2
sm_rc=${PIPESTATUS[0]}
mg_override="$(jq -nc --arg repo "$REPO" --arg cw "$CI_WORKFLOW" '{params:{jobParams:{repo:$repo,ciWorkflow:$cw,selfTick:false}}}')"
dmtools run "$PACKS/machine_merge-$MACHINE_MERGE_VERSION.zip" "$mg_override" 2>&1 | redact >&2
mg_rc=${PIPESTATUS[0]}
out ok "" "$sm_rc" "$mg_rc"
```

Note: the override carries the token as a jobParam (as `factory-sm.yml` does); it is passed as one argv element to dmtools inside the single-tenant Lambda sandbox, never printed (the redact pipe masks it in dmtools' own output). If the reviewer flags argv exposure, the alternative is `jobParams` without tokens plus `SILENT_GH_TOKEN` env — check whether `smAgent.js` reads `silentToken` from env before changing.

- [ ] **Step 4: Run** `bash tests/tick/test_tick.sh`. Expected: PASS. Run `bash tests/leg/run-all.sh` (all leg tests still pass; requires node locally or rely on CI).

- [ ] **Step 5: Implement `bin/events.dart`** — reads env, builds `SfnStarter(run: Process.run)`, selects the handler by `FA_AC_HANDLER`, calls `runLambdaLoop(handler, api: Platform.environment['AWS_LAMBDA_RUNTIME_API']!)`. The tick handler: `Process.start('/app/tick/tick.sh', [])`, write `jsonEncode(event)` to stdin, close, forward its stderr to this process's stderr line by line (CloudWatch), collect stdout, on non-zero exit throw `StateError('tick failed: exit <rc>')`, else `jsonDecode(lastNonEmptyLine)`.

- [ ] **Step 6: Implement `Dockerfile.events`** (mirror the leg Dockerfile's pinned, checksum-verified installs):

```dockerfile
FROM --platform=linux/arm64 dart:stable AS build
WORKDIR /src
COPY pubspec.yaml pubspec.lock ./
RUN dart pub get
COPY lib ./lib
COPY bin ./bin
RUN mkdir -p /out && dart compile exe bin/events.dart -o /out/events

FROM --platform=linux/arm64 debian:trixie-slim AS packs
RUN apt-get update && apt-get install -y --no-install-recommends ca-certificates curl git unzip zip python3 && rm -rf /var/lib/apt/lists/*
COPY engine /engine
RUN bash /engine/build-packs.sh /opt/packs && cp /engine/pins.env /opt/packs/

FROM --platform=linux/arm64 debian:trixie-slim
ARG GH_VERSION=2.102.0
ARG GH_SHA256=7862c86c72f43df3a2d93ddde6f473285b4e2af61b494849846827e513ef6484
ARG DMTOOLS_VERSION=v0.1.44
ARG DMTOOLS_SHA256=e6af54ff9228d221fad92da46fd2782110aa8b3722befb0132a1deb963c8cf5e
RUN apt-get update \
 && apt-get install -y --no-install-recommends awscli jq ca-certificates curl git openssl \
 && rm -rf /var/lib/apt/lists/*
RUN curl -fsSLo /tmp/gh.tgz "https://github.com/cli/cli/releases/download/v${GH_VERSION}/gh_${GH_VERSION}_linux_arm64.tar.gz" \
 && echo "${GH_SHA256}  /tmp/gh.tgz" | sha256sum -c - \
 && tar -xzf /tmp/gh.tgz -C /tmp && install -m 0755 "/tmp/gh_${GH_VERSION}_linux_arm64/bin/gh" /usr/local/bin/gh && rm -rf /tmp/gh*
RUN curl -fsSLo /tmp/dm.tgz "https://github.com/epam/dmtools-dart/releases/download/${DMTOOLS_VERSION}/dmtools-linux-arm64.tar.gz" \
 && echo "${DMTOOLS_SHA256}  /tmp/dm.tgz" | sha256sum -c - \
 && mkdir -p /tmp/dm /opt/dmtools/bin/native/quickjs && tar -xzf /tmp/dm.tgz -C /tmp/dm \
 && cp /tmp/dm/dmtools/dmtools /opt/dmtools/bin/dmtools.bin \
 && cp /tmp/dm/dmtools/native/quickjs/libquickjs_bridge.so /opt/dmtools/bin/native/quickjs/ \
 && printf '%s\n' '#!/bin/sh' 'DIR=$(CDPATH= cd -- "$(dirname -- "$0")" && pwd)' 'export JSR_QUICKJS_LIB="$DIR/native/quickjs/libquickjs_bridge.so"' 'exec "$DIR/dmtools.bin" "$@"' > /opt/dmtools/bin/dmtools \
 && chmod 0755 /opt/dmtools/bin/dmtools /opt/dmtools/bin/dmtools.bin && rm -rf /tmp/dm /tmp/dm.tgz
COPY --from=packs /opt/packs /opt/packs
COPY --from=build /out/events /app/events
COPY --chmod=0755 runner/tick /app/tick
COPY --chmod=0755 runner/leg/lib.sh runner/leg/app-token.sh /app/leg/
ENV PATH=/opt/dmtools/bin:$PATH HOME=/tmp/home
USER 10001
ENTRYPOINT ["/app/events"]
```

- [ ] **Step 7: Implement `scripts/contract-events.sh <image>`:** asserts arm64, size < 1 GB, non-root uid, tools present (`aws gh git curl jq openssl dmtools`), the QuickJS lib loads (`python3` is not in this image — use `dmtools --version` plus `ldd /opt/dmtools/bin/native/quickjs/libquickjs_bridge.so | grep -q 'not found'` must be false), both pack zips and `.sha256` exist and verify, and the image runs one invocation against a local fake Runtime API: start the container with `-e AWS_LAMBDA_RUNTIME_API=host.docker.internal:<port> --add-host host.docker.internal:host-gateway -e FA_AC_HANDLER=tick -e FA_AC_REPOS='{}'`, serve one `{"repo":"x/y"}` invocation from a tiny `python3 -m http.server`-style script on the host (written inline in the contract script), and expect a POST to `/response` whose body has `"reason":"repo-not-allowed"`.

- [ ] **Step 8: Wire CI.** In `test.yml`'s `dart` job, after the leg contract test: `docker build -f Dockerfile.events -t fa-ac-events:ci . && bash scripts/contract-events.sh fa-ac-events:ci`. In `tests/leg/run-all.sh` also run `../tick/test_*.sh`.

- [ ] **Step 9: Run** locally what you can (`dart analyze --fatal-infos && dart test && bash tests/tick/test_tick.sh`); push branch `feat/m2a-events` (Tasks 4–6), open the PR, wait for CI green (it builds and contract-tests the image), merge. Commit for this task: `feat(events): tick script, events entrypoint and image`.

---

### Task 7: Leg runner — `trigger` field and the tick wake-up

**Files:**
- Modify: `lib/src/spec.dart` (L78 optional strings), `runner/leg/factory-leg.sh` (guard call `ACTOR=`), `terraform/runtime/leg_state_machine.tf` (Payload + wake tick)
- Test: `test/spec_test.dart`, `tests/leg/test_factory_leg.sh`, `terraform/runtime/tests/*.tftest.hcl`

**Interfaces:**
- Consumes: Task 2 invoker permissions.
- Produces: LegSpec optional `trigger` ∈ `{dispatch, labeled, assigned}` (anything else → `SpecException`); factory-leg passes `ACTOR=${TRIGGER:-dispatch}` to the guard (so the #544 manual-rework override check applies to label events, as upstream); `fa-ac-leg` forwards `trigger` and, after releasing the lock (both success and failure paths), starts `fa-ac-tick` with `{repo}` (failure to start is caught and ignored).

- [ ] **Step 1: Failing tests.** `test/spec_test.dart`: a factory spec with `"trigger":"labeled"` parses and keeps it; `"trigger":"push"` throws `SpecException`. `tests/leg/test_factory_leg.sh`: with spec `trigger: labeled` and sender `stranger` lacking write permission (fake `gh api repos/o/r/collaborators/stranger/permission` → `read`) and `leg: rework`, the guard skips with `override-refused`; with no trigger, the existing dispatch behaviour is unchanged. Terraform test: the leg definition contains `'trigger': $spec.trigger` and a state `WakeTick` of type Task with resource `arn:aws:states:::aws-sdk:sfn:startExecution`.

- [ ] **Step 2: Run** `dart test test/spec_test.dart`, `bash tests/leg/test_factory_leg.sh` (CI), `cd terraform/runtime && terraform test`. Expected: FAIL.

- [ ] **Step 3: Implement.** `spec.dart`: add `'trigger'` to the optional-string list and validate the value set. `factory-leg.sh`: read `TRIGGER` from the spec the same way `SENDER` is read, and replace `ACTOR=dispatch` with `ACTOR="${TRIGGER:-dispatch}"`. `leg_state_machine.tf`: add `'trigger': $spec.trigger` to the Payload object; change `ReleaseLock` → `Next = "WakeTick"` (keep its Catch → `WakeTick`), and `ReleaseLockAfterFailure` → `Next = "WakeTickAfterFailure"`; add:

```hcl
WakeTick = {
  Type      = "Task"
  Resource  = "arn:aws:states:::aws-sdk:sfn:startExecution"
  Arguments = { StateMachineArn = "arn:aws:states:us-east-1:${data.aws_caller_identity.current.account_id}:stateMachine:fa-ac-tick", Input = "{% $string({'repo': $spec.repo}) %}" }
  Catch     = [{ ErrorEquals = ["States.ALL"], Next = "Done" }]
  Next      = "Done"
}
WakeTickAfterFailure = {
  Type      = "Task"
  Resource  = "arn:aws:states:::aws-sdk:sfn:startExecution"
  Arguments = { StateMachineArn = "arn:aws:states:us-east-1:${data.aws_caller_identity.current.account_id}:stateMachine:fa-ac-tick", Input = "{% $string({'repo': $spec.repo}) %}" }
  Catch     = [{ ErrorEquals = ["States.ALL"], Next = "Failed" }]
  Next      = "Failed"
}
```

(Use whatever account-id source `terraform/runtime` already has; add `data "aws_caller_identity" "current" {}` only if none exists. `Done` must still output `$result`; `Failed` must still use `$failure`.)

- [ ] **Step 4: Run** the three test suites. Expected: PASS.

- [ ] **Step 5: Commit** on branch `feat/m2a-runtime` (shared with Task 8): `feat(leg): trigger field and tick wake-up after each leg`.

---

### Task 8: Runtime — tick state machine, Lambdas, API, schedules, deploy wiring

**Files:**
- Create: `terraform/runtime/events.tf`, `terraform/runtime/tick_state_machine.tf`, `terraform/runtime/tests/events.tftest.hcl`
- Modify: `terraform/runtime/main.tf` (vars `events_image_uri`, `factory_repos`, `installation_id`), `.github/workflows/deploy.yml` (image job builds both images; runtime job passes both digests; new step "Configure App webhook")

**Interfaces:**
- Consumes: Tasks 2, 6, 7.
- Produces: `fa-ac-webhook` and `fa-ac-sm-tick` Lambdas (image `fa-ac-events@sha256:…`, arm64, role `fa-ac-events`), API `fa-ac-webhook` route `POST /github`, state machine `fa-ac-tick`, schedules `fa-ac-tick-<slug>` every 15 minutes, output `webhook_url` (not sensitive — it is a public endpoint by design; it contains no account ID).

Variables in `main.tf`:

```hcl
variable "events_image_uri" {
  type = string
  validation {
    condition     = can(regex("^[0-9]{12}\\.dkr\\.ecr\\.us-east-1\\.amazonaws\\.com/fa-ac-events@sha256:[0-9a-f]{64}$", var.events_image_uri))
    error_message = "events_image_uri must be a digest-pinned fa-ac-events image."
  }
}
variable "factory_repos" {
  type    = map(object({ ci_workflow = string }))
  default = { "rustembuild/fa-canary-snake" = { ci_workflow = "ci.yml" } }
}
variable "installation_id" {
  type    = number
  default = 169607698
}
```

- [ ] **Step 1: Failing test** `terraform/runtime/tests/events.tftest.hcl` (mock provider like the existing runtime tests; set `events_image_uri` to a valid fake digest): assert both Lambdas use `package_type = "Image"`, `architectures = ["arm64"]`, image = the var; `fa-ac-sm-tick` timeout 900 and memory ≥ 2048, env has `FA_AC_HANDLER = "tick"`, `FA_AC_LEG_SM_ARN`, `FA_AC_REPOS` = `{"rustembuild/fa-canary-snake":{"ciWorkflow":"ci.yml"}}`; `fa-ac-webhook` timeout ≤ 10, env `FA_AC_HANDLER = "webhook"`, `FA_AC_INSTALLATION_ID = "169607698"`; the API route key is `POST /github` with throttling burst 20 / rate 5; one schedule per repo with `schedule_expression = "rate(15 minutes)"` targeting `fa-ac-tick`; `fa-ac-tick` uses the invoker role and contains states `AcquireRepoLock`, `MarkDirty`, `RunTick`, `ReleaseRepoLock`, `CheckDirty`.

- [ ] **Step 2: Run** `cd terraform/runtime && terraform test`. Expected: FAIL.

- [ ] **Step 3: Implement `events.tf`:** two `aws_lambda_function`s (`image_uri = var.events_image_uri`, `role` = foundation `events_role_arn`), log groups `/aws/lambda/fa-ac-webhook` and `/aws/lambda/fa-ac-sm-tick` with `retention_in_days = 30`, `aws_apigatewayv2_api` (HTTP) `fa-ac-webhook`, `aws_apigatewayv2_integration` (AWS_PROXY, payload format 2.0), route `POST /github`, `$default` stage with `auto_deploy = true` and `default_route_settings { throttling_burst_limit = 20, throttling_rate_limit = 5 }`, `aws_lambda_permission` for `apigateway.amazonaws.com` scoped to the API's execution ARN, and `aws_scheduler_schedule` per `factory_repos` key (name `fa-ac-tick-${slug}` where slug follows the shared contract, `flexible_time_window { mode = "OFF" }`, target `arn` = tick state machine, `role_arn` = foundation `scheduler_role_arn`, `input = jsonencode({ repo = each.key })`). Output `webhook_url = "${aws_apigatewayv2_stage.default.invoke_url}github"` (check whether `invoke_url` ends with `/`).

- [ ] **Step 4: Implement `tick_state_machine.tf`** (JSONata, STANDARD, role = invoker, logging like `fa-ac-leg`, `TimeoutSeconds = 3600`):

```
Prepare         Pass   Assign repo=$states.input.repo, lockKey='repo#'&repo, dirtyKey='dirty#'&repo, pass=0 → AcquireRepoLock
AcquireRepoLock dynamodb:putItem {pk:$lockKey, owner:$states.context.Execution.Id, expiresAt: now+1200}
                ConditionExpression "attribute_not_exists(pk) OR expiresAt < :now"
                Catch ConditionalCheckFailedException → MarkDirty;  Next → ClearDirty
MarkDirty       dynamodb:putItem {pk:$dirtyKey, expiresAt: now+1800} → Busy (Succeed {status:'dirty'})
ClearDirty      dynamodb:deleteItem {pk:$dirtyKey}  (Catch States.ALL → RunTick) → RunTick
RunTick         arn:aws:states:::lambda:invoke  FunctionName fa-ac-sm-tick, Payload {repo}, TimeoutSeconds 900,
                Retry Lambda.ServiceException/TooManyRequests x2; Catch States.ALL → ReleaseRepoLock (Assign failed=true) → ReleaseRepoLock
ReleaseRepoLock dynamodb:deleteItem {pk:$lockKey} ConditionExpression "#o = :me"; Catch States.ALL → CheckDirty → CheckDirty
CheckDirty      dynamodb:deleteItem {pk:$dirtyKey} ReturnValues ALL_OLD; Assign wasDirty = $exists($states.result.Attributes)
                Catch States.ALL → Done → Loop?
Loop?           Choice: wasDirty and pass < 2 → Again (Pass, Assign pass=pass+1 → AcquireRepoLock); else Done
Done            Succeed (Output {repo, passes: pass+1})
```

Write it as HCL `jsonencode` with the same JSONata style as `leg_state_machine.tf` (`{% %}` expressions, `$millis()`/`$floor($millis()/1000)` for now). Validate locally with `aws stepfunctions validate-state-machine-definition --definition file://...` if credentials are at hand; CI's runtime job validates anyway.

- [ ] **Step 5: Wire `deploy.yml`.** Image job: build and push both images (leg runner as today; `fa-ac-events` from `Dockerfile.events`, same "skip if tag exists" logic, repo URL from foundation output `events_ecr_repository_url`); outputs `image_digest` and `events_image_digest`. Runtime job: export `TF_VAR_events_image_uri=<acct>.dkr.ecr.us-east-1.amazonaws.com/fa-ac-events@$EVENTS_IMAGE_DIGEST`. New final step "Configure App webhook" (env `aws-deploy`, secrets `FA_AC_GITHUB_APP_JSON`, `FA_AC_WEBHOOK_SECRET`): build the App JWT with `runner/leg/app-token.sh`'s `app_jwt` (source the file; `APP_ID`, `APP_PRIVATE_KEY` from the JSON via jq into env, never echoed), then `curl -fsS -X PATCH https://api.github.com/app/hook/config -H @- -d <json>` (Authorization header on stdin, as `app-token.sh` does) with `{"url": <webhook_url>, "content_type": "json", "secret": <secret>, "insecure_ssl": "0"}` built by `jq -n` from env; skip when either secret is empty.

- [ ] **Step 6: Run** `terraform fmt -check -recursive && terraform validate && terraform test` in `terraform/runtime`. Expected: PASS.

- [ ] **Step 7: Commit, push `feat/m2a-runtime` (Tasks 7–8), PR, merge after green, watch `deploy.yml`.** Commit: `feat(runtime): tick state machine, event Lambdas, webhook API and schedules`. After deploy: `aws stepfunctions describe-state-machine --name`-equivalent checks (via `list-state-machines`) show `fa-ac-tick`; `aws lambda get-function --function-name fa-ac-sm-tick` shows the digest; `aws scheduler get-schedule --name fa-ac-tick-rustembuild-fa-canary-snake` exists.

---

### Task 9: Canary readiness, App events, end-to-end run, findings

**Files:**
- Modify (canary `rustembuild/fa-canary-snake`): `.github/workflows/ci.yml` (add `workflow_dispatch:`)
- Create (dark-factory-aws): `docs/m2a-findings.md`

**Interfaces:**
- Consumes: everything above.

- [ ] **Step 1: Canary CI dispatchable.** PR in the canary adding `workflow_dispatch:` under `on:` in `ci.yml`; merge after green. Verify `gh workflow run ci.yml --repo rustembuild/fa-canary-snake --ref main` starts a run.

- [ ] **Step 2: Allowlist the factory bot.** `gh variable set AGENTCORE_ALLOWED_ACTORS --repo rustembuild/fa-canary-snake --body 'agzyamov,rustembuild-factory[bot]'`.

- [ ] **Step 3: Owner step — App events.** The owner opens https://github.com/organizations/rustembuild/settings/apps/rustembuild-factory, checks the webhook URL was set by CI, ticks **Active**, and subscribes to events: Issues, Issue comment, Pull request, Pull request review, Pull request review comment, Check run, Check suite, Workflow run. Save.

- [ ] **Step 4: Smoke the webhook.** In the App's "Advanced → Recent deliveries" the `ping` shows 200 (body `ignored-event`). Redeliver nothing yet.

- [ ] **Step 5: Safety tick check.** Within 15 minutes `aws stepfunctions list-executions --state-machine-arn <fa-ac-tick>` shows a SUCCEEDED `tick-…` execution whose `fa-ac-sm-tick` log (`/aws/lambda/fa-ac-sm-tick`) shows `sm_github` and `machine_merge` ran with exit codes and no token text.

- [ ] **Step 6: End-to-end.** The owner's account (`agzyamov`) creates canary issue "Snake: add a difficulty selector (easy/normal/hard) that changes the starting speed; persist the choice in localStorage; tests for the speed mapping" and labels it `agent:dev`. Without starting anything by hand, observe and record timestamps for: webhook delivery 200 → dev leg execution → PR opened → CI dispatched by the SM (`ai_validating`) → `agent:review` → review leg (via webhook label event) → verdict → (rework round if requested) → `pr_approved` → merge by `machine_merge` → issue closed. If any hop stalls for > 30 minutes, investigate (logs of the tick Lambda, `fa-ac-leg` executions, the guard's `[leg]` lines), fix through a PR, and continue.

- [ ] **Step 7: Findings.** Write `docs/m2a-findings.md` (hop timings table, executions, defects found and fix PRs, anything deferred) in a PR to dark-factory-aws; merge after green.

---

## Self-review notes

- Spec coverage (M2 row, §5.1–5.6, §7 allowlist): App events (Task 9), receiver (5, 8), tick + lock + dirty + schedule (8), engine adapter (3, per R-A/R-B), leg tools (folded into the overlay — `dispatchLeg`/`activeMachineRuns` map to `triggerWorkflow`/`listWorkflowRuns`), leg wake-up (7), fence + roles + hard stop (1, 2), auto-merge (`machine_merge` in the tick, 6). Stubs-skip deferred (R-I). Hardening (R14, 4 h cap) is M2b.
- The leg state machine still caps legs at 3600 s (M1); M2b raises it with token refresh.
