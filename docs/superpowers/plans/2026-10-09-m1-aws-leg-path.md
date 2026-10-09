# M0 finish + M1 (AWS leg path) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Close M0 with a direct AgentCore smoke test, then build the AWS-mode leg path: one factory leg (dev/review/rework) started by hand as a Step Functions execution runs IstiN's unchanged agent packs inside AgentCore and reports back through labels, comments and a PR.

**Architecture:** The Dart shim from M0 gains Step Functions callbacks (heartbeat, success, failure). A bash leg runner ports only the GitHub-Actions glue of IstiN's `factory-teammate.yml` (guard, input seeding, markers, verdict application, cycle close, session/memory persistence) and calls IstiN's scripts unchanged (`review-verdict.sh`, `fa-session.sh`, `git-push-guard.sh`, credential installer, packs via `dmtools run`). The state machine `fa-ac-leg` locks the item in DynamoDB, invokes the runtime with a task token and releases the lock. A GitHub App issues one-hour installation tokens.

**Tech Stack:** Dart 3.12 (shim), bash 5 + jq + gh 2.102.0 (leg runner), dmtools CLI v0.1.44, fa v1.0.535, IstiN packs `agents-rel-20261008-215056`, Terraform 1.16.3 / hashicorp/aws 6.65.0, AWS Step Functions (JSONata), DynamoDB, Secrets Manager, Bedrock AgentCore Runtime.

**Spec:** `docs/superpowers/specs/2026-10-09-aws-orchestrator-switch-design.md` (branch `docs/agentcore-executor-spec` of the fa fork). Port details: `docs/superpowers/plans/2026-10-09-leg-port-reference.md` (same branch). All code tasks happen in `~/projects/dark-factory-aws` (repo `rustembuild/dark-factory-aws`, private) on a feature branch per task group; `main` is pull-request-only.

## Global Constraints

- Region us-east-1 only. Every AWS name starts with `fa-ac-` (AgentCore runtimes `fa_ac_`). Tag `Project=fa-agentcore` comes from provider `default_tags`.
- All AWS resource changes go through Terraform applied by CI (`deploy.yml`: bootstrap → foundation → image → runtime). No `aws` CLI create/update/delete calls from a workstation. Read-only calls and `invoke-agent-runtime` / `start-execution` for tests are fine.
- Never put the AWS account ID, secrets, tokens or private keys in git, logs, PR text or commit messages. Mask the account ID in pasted output (`<ACCT>`).
- Never touch df-agentcore (eu-central-1, `darkfactory-*`, `df-agentcore-*`) or sbx-databricks-demo (`sbx-ard-*`).
- No pull requests to IstiN's repositories. GitHub changes only in `rustembuild/*`.
- Commits: Conventional Commits, PR title = squash commit; trailer `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>` (or the model that wrote it). Repo-local git identity per each repo's conventions.
- Pin every third-party artifact by version and sha256 (images, tarballs, zips); pin GitHub Actions by commit SHA.
- Business logic is IstiN's: do not change rules, labels, caps, verdict decisions or packs. Port only GitHub-Actions glue, with the FT line range cited in a comment above each ported block.
- M1 caps a leg at 1 hour: runtime `max_lifetime` stays 3600, `fa-ac-leg` `TimeoutSeconds` 3600, shim `maxSeconds` 3300. (App installation tokens live 1 hour; the spec's 4-hour cap needs token refresh, planned for M2.)
- The AWS-mode guard refuses to run when the repo variable `AGENTCORE_ALLOWED_ACTORS` is missing or empty (deny by default).

## Review Focus

1. Issue/PR titles, bodies and labels containing quotes, `$()`, backticks, newlines or `=` must never be evaluated as shell: guard output is parsed line by line, never `eval`ed, and markers edit files, not command strings. (Tests in Task 4 and Task 6.)
2. A secret that does not exist yet or is empty (first deploy before the owner fills it) must end the leg as `infra_error` with a clear reason before any GitHub write. (Test in Task 7.)
3. App token minting failing (repo not in the installation, bad key) must end as `infra_error` before cloning. (Test in Task 7.)
4. Step Functions saying the task is gone (timed out, aborted) while the leg still runs must kill the leg's process group so a paid session does not keep working. (Test in Task 2.)
5. A missing `AGENTCORE_ALLOWED_ACTORS` variable must refuse, not allow everyone. (Test in Task 4.)

## File map (dark-factory-aws)

| Path | Responsibility |
|---|---|
| `scripts/smoke-direct.sh` | M0 direct invoke of the hello leg + wait for S3 result |
| `docs/m0-findings.md` | M0 measurements and limits |
| `lib/src/spec.dart` | LegSpec validation (hello + factory legs) |
| `lib/src/callback.dart` | `TaskCallback` interface + aws CLI implementation |
| `lib/src/job.dart` | runs the leg command, heartbeats, delivers the result |
| `lib/src/config.dart` | builds writer + callback from env/spec |
| `bin/leg_runner.dart` | entrypoint wiring |
| `runner/run-leg.sh` | dispatcher: `hello` → `hello.sh`, else `factory-leg.sh` |
| `runner/hello.sh` | M0 hello leg (moved) |
| `runner/leg/lib.sh` | logging, GitHub-env loader, label helpers |
| `runner/leg/guard.sh` | port of FT guard (109-390) + allowlist |
| `runner/leg/markers.sh` | job markers in the issue body (FT 856-886, 1167-1204) |
| `runner/leg/verdict.sh` | apply review verdict (FT 1206-1420) |
| `runner/leg/session.sh` | fa session restore/persist/quarantine (FT 587-602, 1482-1538, 1571-1605) |
| `runner/leg/tokens.sh` | token usage publish (FT 1061-1165) |
| `runner/leg/app-token.sh` | GitHub App installation token (JWT via openssl) |
| `runner/leg/factory-leg.sh` | the leg: secrets → token → clone → guard → setup → agent → post |
| `tests/leg/harness.sh`, `tests/leg/bin/gh`, `tests/leg/bin/dmtools` | bash test harness and fakes |
| `tests/leg/test_*.sh`, `tests/leg/run-all.sh` | leg runner tests |
| `Dockerfile`, `scripts/contract-test.sh` | image with pinned tools; contract checks |
| `terraform/bootstrap/fence.tf` | fence: DynamoDB + Step Functions callbacks |
| `terraform/foundation/{main,iam,locks,secrets}.tf` | locks table, secrets, roles |
| `terraform/runtime/{main,leg_state_machine}.tf` | runtime env, `fa-ac-leg` |
| `.github/workflows/{test,deploy}.yml` | leg tests in CI; secret sync |

---

### Task 1: M0 finish — direct smoke, findings, retire the GitHub smoke path

**Files:**
- Create: `scripts/smoke-direct.sh`, `docs/m0-findings.md`

**Interfaces:**
- Consumes: deployed runtime `fa_ac_leg_runner`, bucket `fa-ac-legs-<acct>`.
- Produces: findings used by M1 sizing (cold start, idle-timeout behaviour).

- [ ] **Step 1: Write the smoke script**

```bash
#!/usr/bin/env bash
# M0: invoke the hello leg directly (owner credentials) and wait for its
# LegResult in S3. Read-only except for the paid session itself.
# Usage: scripts/smoke-direct.sh <sleepSeconds>
set -euo pipefail
sleep_s="${1:-30}"
[[ "$sleep_s" =~ ^[0-9]{1,4}$ ]] || { echo "sleepSeconds must be 0-9999" >&2; exit 64; }
export AWS_REGION=us-east-1
acct="$(aws sts get-caller-identity --query Account --output text)"
arn="$(aws bedrock-agentcore-control list-agent-runtimes \
  --query "agentRuntimes[?agentRuntimeName=='fa_ac_leg_runner'].agentRuntimeArn | [0]" --output text)"
[[ "$arn" == arn:* ]] || { echo "runtime fa_ac_leg_runner not found" >&2; exit 1; }
sid="m0-smoke-$(date +%s)-$(openssl rand -hex 8)"   # >= 33 characters
payload="$(jq -nc --arg s "$sid" --argjson z "$sleep_s" \
  '{sessionId:$s, leg:"hello", maxSeconds:3300, callback:{type:"s3"}, hello:{sleepSeconds:$z}}')"
out="$(mktemp)"; trap 'rm -f "$out"' EXIT
start="$(date +%s)"
aws bedrock-agentcore invoke-agent-runtime --agent-runtime-arn "$arn" \
  --runtime-session-id "$sid" --content-type application/json \
  --cli-binary-format raw-in-base64-out --payload "$payload" "$out" >/dev/null
accepted="$(date +%s)"
echo "accepted after $((accepted - start))s: $(head -c 200 "$out")"
until aws s3api head-object --bucket "fa-ac-legs-$acct" --key "legs/$sid/result.json" >/dev/null 2>&1; do
  (( $(date +%s) - start < sleep_s + 600 )) || { echo "no LegResult after $((sleep_s + 600))s" >&2; exit 1; }
  sleep 10
done
echo "result: $(aws s3 cp "s3://fa-ac-legs-$acct/legs/$sid/result.json" -)"
echo "wall clock: $(( $(date +%s) - start ))s"
```

- [ ] **Step 2: Run the short smoke**

Run: `chmod +x scripts/smoke-direct.sh && scripts/smoke-direct.sh 30 2>&1 | sed -E 's/[0-9]{12}/<ACCT>/g'`
Expected: `accepted after Ns: {"accepted":true...}`, then `result: {"status":"ok","exitCode":0,...,"durationSeconds":3x}`. Record N (cold start) and the wall clock.

- [ ] **Step 3: Run the long smoke (beyond the 15-minute idle timeout)**

Run: `scripts/smoke-direct.sh 1000 2>&1 | sed -E 's/[0-9]{12}/<ACCT>/g'` (run in the background; takes ~17 min)
Expected: `"status":"ok"` with `durationSeconds` ≈ 1000. Proves a busy session (`HealthyBusy`) survives past `idle_runtime_session_timeout` 900.

- [ ] **Step 4: Write `docs/m0-findings.md`**

```markdown
# M0 findings (2026-10-09)

| Measurement | Value |
|---|---|
| Cold start (invoke → accepted) | <N from Step 2> s |
| 30 s hello leg wall clock | <value> s |
| 1000 s hello leg | status ok, <durationSeconds> s — survives the 900 s idle timeout while HealthyBusy |
| Image | arm64, <size> MB |
| Budget | fa-ac-monthly $50, AUTOMATIC hard stop armed (STANDBY) |
| Isolation | df-agentcore / sbx-databricks-demo / OIDC provider / GitHubActionsRole snapshot: unchanged |

Limits confirmed: 2 vCPU / 8 GB per session, max lifetime 3600 s (M0/M1 cap),
session id ≥ 33 characters, payload delivered as raw JSON.
```

Fill each `<...>` with the measured value before committing (no placeholders in the committed file).

- [ ] **Step 5: Retire the GitHub smoke path (spec §9)**

```bash
gh api -X DELETE repos/rustembuild/flutter_agent_harness_agentcore/environments/agentcore
git -C ~/projects/flutter_agent_harness_agentcore worktree remove ~/projects/fa-agentcore-smoke
git -C ~/projects/flutter_agent_harness_agentcore branch -D feat/agentcore-smoke
```

Expected: environment gone (`gh api repos/rustembuild/flutter_agent_harness_agentcore/environments --jq '.total_count'` → 0); the branch was never pushed. Also update `github/README.md` in dark-factory-aws: replace the `agentcore` environment bullet with "Retired 2026-10-09 (spec 2026-10-09 §9): in aws mode GitHub never invokes AgentCore." and delete `github/fork-agentcore-environment.json`.

- [ ] **Step 6: Commit and open the PR**

```bash
git checkout -b chore/m0-finish
git add scripts/smoke-direct.sh docs/m0-findings.md github/README.md github/fork-agentcore-environment.json
git commit -m "chore: close M0 with a direct smoke test and findings"
git push -u origin chore/m0-finish
gh pr create --repo rustembuild/dark-factory-aws --base main --title "chore: close M0 with a direct smoke test and findings" --body-file <(printf '%s\n' "## What" "Direct AgentCore smoke script and M0 findings; retire the fork's GitHub smoke path." "" "## How it was verified" "Smoke runs at 30 s and 1000 s (results in docs/m0-findings.md).")
```

---

### Task 2: Shim — factory LegSpec + Step Functions callbacks (Dart)

**Files:**
- Modify: `lib/src/spec.dart`, `lib/src/job.dart`, `lib/src/config.dart`, `bin/leg_runner.dart`, `lib/leg_runner.dart`
- Create: `lib/src/callback.dart`
- Test: `test/spec_test.dart`, `test/job_test.dart`, `test/config_test.dart`, `test/callback_test.dart`

**Interfaces:**
- Consumes: existing `parseLegSpec`, `runLeg`, `ResultWriter`, `BusyTracker`, `ProcessRunner`.
- Produces:
  - LegSpec factory fields: `repo` (`owner/name`), exactly one of `issue`/`pr` (positive int), `leg` ∈ `dev|review|rework|auto`, optional `reason`, `ref`, `sender`, `runUrl` (strings), optional `taskToken` (string ≤ 4096).
  - `abstract interface class TaskCallback { Future<bool> heartbeat(); Future<void> succeed(Map<String, Object?> result); Future<void> fail(String error, String cause); }` — `heartbeat()` returns false when the task is gone.
  - `class AwsCliTaskCallback implements TaskCallback` (`AwsCliTaskCallback(String token, {ProcessRunner? run})`).
  - `TaskCallback? callbackFor(Map<String, Object?> spec, {ProcessRunner? run})`.
  - `runLeg(spec, writer, busy, command, {TaskCallback? callback, Duration heartbeatEvery = const Duration(minutes: 5)})`.
  - Error names sent with `fail`: the LegResult `status` (`agent_failed` or `infra_error`); Step Functions retries on `infra_error`.

- [ ] **Step 1: Write failing spec tests** (append to `test/spec_test.dart`)

```dart
  group('factory legs', () {
    Map<String, Object?> base() => {
          'sessionId': 'a' * 36,
          'leg': 'dev',
          'maxSeconds': 3300,
          'callback': {'type': 's3'},
          'repo': 'rustembuild/fa-canary-snake',
          'issue': 1,
        };
    List<int> enc(Map<String, Object?> m) => utf8.encode(jsonEncode(m));

    test('accepts an issue-anchored dev leg', () {
      expect(parseLegSpec(enc(base()))['repo'], 'rustembuild/fa-canary-snake');
    });
    test('accepts auto, review and rework', () {
      for (final leg in ['auto', 'review', 'rework']) {
        expect(parseLegSpec(enc({...base(), 'leg': leg}))['leg'], leg);
      }
    });
    test('rejects an unknown leg', () {
      expect(() => parseLegSpec(enc({...base(), 'leg': 'deploy'})), throwsA(isA<SpecException>()));
    });
    test('rejects a malformed repo', () {
      for (final repo in ['noslash', 'a/b/c', 'a b/c', '../x/y', '']) {
        expect(() => parseLegSpec(enc({...base(), 'repo': repo})), throwsA(isA<SpecException>()), reason: repo);
      }
    });
    test('requires exactly one of issue and pr', () {
      expect(() => parseLegSpec(enc({...base(), 'pr': 2})), throwsA(isA<SpecException>()));
      expect(() => parseLegSpec(enc({...base()}..remove('issue'))), throwsA(isA<SpecException>()));
      expect(parseLegSpec(enc({...base()..remove('issue'), 'pr': 2}))['pr'], 2);
    });
    test('rejects non-positive anchors', () {
      expect(() => parseLegSpec(enc({...base(), 'issue': 0})), throwsA(isA<SpecException>()));
      expect(() => parseLegSpec(enc({...base(), 'issue': '1'})), throwsA(isA<SpecException>()));
    });
    test('optional strings must be strings', () {
      for (final key in ['reason', 'ref', 'sender', 'runUrl', 'taskToken']) {
        expect(() => parseLegSpec(enc({...base(), key: 5})), throwsA(isA<SpecException>()), reason: key);
      }
    });
    test('hello legs need no repo', () {
      final hello = {'sessionId': 'b' * 36, 'leg': 'hello', 'maxSeconds': 10, 'callback': {'type': 's3'}};
      expect(parseLegSpec(enc(hello))['leg'], 'hello');
    });
  });
```

- [ ] **Step 2: Run to verify failure**

Run: `dart test test/spec_test.dart`
Expected: FAIL (e.g. "rejects an unknown leg" — no exception thrown).

- [ ] **Step 3: Implement factory validation** in `lib/src/spec.dart` (after the callback check, before `return decoded;`)

```dart
  if (leg != 'hello') _checkFactoryLeg(decoded, leg);
  return decoded;
}

const factoryLegs = {'dev', 'review', 'rework', 'auto'};
final _repo = RegExp(r'^[A-Za-z0-9_.-]+/[A-Za-z0-9_.-]+$');

void _checkFactoryLeg(Map<String, Object?> spec, String leg) {
  if (!factoryLegs.contains(leg)) {
    throw SpecException('leg must be hello, ${factoryLegs.join(', ')}');
  }
  final repo = spec['repo'];
  if (repo is! String || !_repo.hasMatch(repo) || repo.contains('..')) {
    throw SpecException('repo must be owner/name');
  }
  final issue = spec['issue'];
  final pr = spec['pr'];
  final hasIssue = issue != null;
  final hasPr = pr != null;
  if (hasIssue == hasPr) {
    throw SpecException('exactly one of issue and pr is required');
  }
  final anchor = hasIssue ? issue : pr;
  if (anchor is! int || anchor < 1) {
    throw SpecException('issue/pr must be a positive integer');
  }
  for (final key in ['reason', 'ref', 'sender', 'runUrl', 'taskToken']) {
    final value = spec[key];
    if (value != null && value is! String) {
      throw SpecException('$key must be a string');
    }
  }
  final token = spec['taskToken'];
  if (token is String && (token.isEmpty || token.length > 4096)) {
    throw SpecException('taskToken must be 1-4096 characters');
  }
}
```

Delete the old `return decoded;` line it replaces, so the function ends with the two lines shown.

- [ ] **Step 4: Run spec tests**

Run: `dart test test/spec_test.dart`
Expected: PASS (all old and new tests).

- [ ] **Step 5: Write failing callback tests** — `test/callback_test.dart`

```dart
import 'dart:io';

import 'package:leg_runner/leg_runner.dart';
import 'package:test/test.dart';

void main() {
  late List<List<String>> calls;
  late ProcessResult Function(List<String>) answer;

  ProcessRunner fake() => (exe, args) async {
        calls.add([exe, ...args]);
        return answer(args);
      };

  setUp(() {
    calls = [];
    answer = (_) => ProcessResult(1, 0, '', '');
  });

  test('heartbeat calls send-task-heartbeat and reports alive', () async {
    final cb = AwsCliTaskCallback('tok', run: fake());
    expect(await cb.heartbeat(), isTrue);
    expect(calls.single, ['aws', 'stepfunctions', 'send-task-heartbeat', '--task-token', 'tok']);
  });

  test('heartbeat reports gone on TaskTimedOut / TaskDoesNotExist', () async {
    for (final err in ['TaskTimedOut', 'TaskDoesNotExist']) {
      answer = (_) => ProcessResult(1, 254, '', 'An error occurred ($err)');
      expect(await AwsCliTaskCallback('tok', run: fake()).heartbeat(), isFalse, reason: err);
    }
  });

  test('heartbeat treats other failures as alive', () async {
    answer = (_) => ProcessResult(1, 255, '', 'Could not connect');
    expect(await AwsCliTaskCallback('tok', run: fake()).heartbeat(), isTrue);
  });

  test('succeed sends the result as task output', () async {
    await AwsCliTaskCallback('tok', run: fake()).succeed({'status': 'ok'});
    expect(calls.single, ['aws', 'stepfunctions', 'send-task-success', '--task-token', 'tok', '--task-output', '{"status":"ok"}']);
  });

  test('fail sends error and a cause capped at 32768 characters', () async {
    await AwsCliTaskCallback('tok', run: fake()).fail('infra_error', 'x' * 40000);
    final c = calls.single;
    expect(c.sublist(0, 7), ['aws', 'stepfunctions', 'send-task-failure', '--task-token', 'tok', '--error', 'infra_error']);
    expect(c[8].length, 32768);
  });

  test('callbackFor returns null without a token', () {
    expect(callbackFor({'leg': 'hello'}), isNull);
    expect(callbackFor({'taskToken': 'tok'}), isA<AwsCliTaskCallback>());
  });
}
```

- [ ] **Step 6: Run to verify failure**

Run: `dart test test/callback_test.dart`
Expected: FAIL to compile (`AwsCliTaskCallback` undefined).

- [ ] **Step 7: Implement `lib/src/callback.dart`**

```dart
import 'dart:convert';
import 'dart:io';

import 'results.dart';

/// Step Functions task-token callback (spec 2026-10-09 §5.7).
abstract interface class TaskCallback {
  /// False when Step Functions says the task is gone (timed out, aborted).
  Future<bool> heartbeat();
  Future<void> succeed(Map<String, Object?> result);
  Future<void> fail(String error, String cause);
}

/// Uses the aws CLI baked into the image; the runtime role grants
/// states:SendTask*.
class AwsCliTaskCallback implements TaskCallback {
  AwsCliTaskCallback(this.token, {ProcessRunner? run}) : _run = run ?? Process.run;

  final String token;
  final ProcessRunner _run;
  static const _maxCause = 32768;

  @override
  Future<bool> heartbeat() async {
    final r = await _run('aws', ['stepfunctions', 'send-task-heartbeat', '--task-token', token]);
    if (r.exitCode == 0) return true;
    final err = '${r.stderr}';
    if (err.contains('TaskTimedOut') || err.contains('TaskDoesNotExist')) return false;
    stderr.writeln('heartbeat failed, assuming the task is alive: $err');
    return true;
  }

  @override
  Future<void> succeed(Map<String, Object?> result) => _send([
        'send-task-success', '--task-token', token, '--task-output', jsonEncode(result),
      ]);

  @override
  Future<void> fail(String error, String cause) => _send([
        'send-task-failure', '--task-token', token, '--error', error,
        '--cause', cause.length > _maxCause ? cause.substring(0, _maxCause) : cause,
      ]);

  Future<void> _send(List<String> args) async {
    final r = await _run('aws', ['stepfunctions', ...args]);
    if (r.exitCode != 0) {
      stderr.writeln('aws stepfunctions ${args.first} failed (${r.exitCode}): ${r.stderr}');
    }
  }
}

TaskCallback? callbackFor(Map<String, Object?> spec, {ProcessRunner? run}) {
  final token = spec['taskToken'];
  return token is String ? AwsCliTaskCallback(token, run: run) : null;
}
```

Export it: add `export 'src/callback.dart';` to `lib/leg_runner.dart`.

- [ ] **Step 8: Run callback tests**

Run: `dart test test/callback_test.dart`
Expected: PASS.

- [ ] **Step 9: Write failing job tests** (append to `test/job_test.dart`; reuse the file's existing helpers for a temp script command and a `FileResultWriter` — read the file first and follow its style)

```dart
  group('with a task callback', () {
    late _RecordingCallback cb;
    setUp(() => cb = _RecordingCallback());

    test('ok leg reports success with the result', () async {
      await runLeg(_spec(maxSeconds: 10), writer, busy, script('exit 0'), callback: cb);
      expect(cb.events, ['succeed:ok']);
    });

    test('failed leg reports failure with the status as error', () async {
      await runLeg(_spec(maxSeconds: 10), writer, busy, script('exit 3'), callback: cb);
      expect(cb.events, ['fail:agent_failed:exit 3']);
    });

    test('heartbeats while the leg runs', () async {
      await runLeg(_spec(maxSeconds: 10), writer, busy, script('sleep 1'),
          callback: cb, heartbeatEvery: const Duration(milliseconds: 200));
      expect(cb.events.where((e) => e == 'heartbeat').length, greaterThanOrEqualTo(3));
    });

    test('kills the leg when the task is gone', () async {
      cb.alive = false;
      final sw = Stopwatch()..start();
      await runLeg(_spec(maxSeconds: 30), writer, busy, script('sleep 30'),
          callback: cb, heartbeatEvery: const Duration(milliseconds: 200));
      expect(sw.elapsed, lessThan(const Duration(seconds: 10)));
      expect(cb.events.last, 'fail:infra_error:task-gone');
    });
  });
```

and at file end:

```dart
class _RecordingCallback implements TaskCallback {
  final events = <String>[];
  bool alive = true;
  @override
  Future<bool> heartbeat() async { events.add('heartbeat'); return alive; }
  @override
  Future<void> succeed(Map<String, Object?> r) async => events.add('succeed:${r['status']}');
  @override
  Future<void> fail(String e, String c) async => events.add('fail:$e:$c');
}
```

If `test/job_test.dart` has no `_spec`/`script` helpers with these shapes, add them: `_spec({required int maxSeconds})` returns `{'sessionId': 's' * 36, 'leg': 'hello', 'maxSeconds': maxSeconds, 'callback': {'type': 's3'}}`; `script(String body)` writes `#!/usr/bin/env bash\n$body\n` to a temp file, `chmod +x`, and returns `[path]`.

- [ ] **Step 10: Run to verify failure**

Run: `dart test test/job_test.dart`
Expected: FAIL to compile (`callback` named parameter undefined).

- [ ] **Step 11: Implement callbacks in `lib/src/job.dart`**

Change the signature and body of `runLeg`:

```dart
Future<void> runLeg(
  Map<String, Object?> spec,
  ResultWriter writer,
  BusyTracker busy,
  List<String> command, {
  TaskCallback? callback,
  Duration heartbeatEvery = const Duration(minutes: 5),
}) async {
  final stopwatch = Stopwatch()..start();
  final sessionId = spec['sessionId'] as String;
  try {
    final result = await _execute(spec, command, callback, heartbeatEvery)
      ..['sessionId'] = sessionId
      ..['durationSeconds'] = stopwatch.elapsedMilliseconds / 1000;
    try {
      await writer.write(sessionId, result);
    } catch (error) {
      stderr.writeln('could not deliver LegResult for session $sessionId: $error');
    }
    if (callback != null) {
      if (result['status'] == 'ok') {
        await callback.succeed(result);
      } else {
        await callback.fail(
          result['status'] as String,
          (result['reason'] as String?) ?? 'exit ${result['exitCode']}',
        );
      }
    }
  } finally {
    busy.finish();
  }
}
```

In `_execute`, add the two parameters and replace the wait block:

```dart
Future<Map<String, Object?>> _execute(
  Map<String, Object?> spec,
  List<String> command,
  TaskCallback? callback,
  Duration heartbeatEvery,
) async {
  // ... unchanged up to `final int code;`
    var gone = false;
    Timer? beat;
    if (callback != null) {
      beat = Timer.periodic(heartbeatEvery, (_) async {
        if (gone) return;
        if (!await callback.heartbeat()) {
          gone = true;
          await Process.run('bash', ['-c', 'kill -KILL -- -${process.pid}']);
        }
      });
    }
    final int code;
    try {
      code = await process.exitCode.timeout(
        Duration(seconds: spec['maxSeconds'] as int),
      );
    } on TimeoutException {
      await Process.run('bash', ['-c', 'kill -KILL -- -${process.pid}']);
      await process.exitCode;
      return {'status': 'infra_error', 'exitCode': null, 'reason': 'timeout'};
    } finally {
      beat?.cancel();
    }
    if (gone) {
      return {'status': 'infra_error', 'exitCode': code, 'reason': 'task-gone'};
    }
  // ... unchanged from `if (code == 126 || code == 127)`
```

- [ ] **Step 12: Wire the entrypoint** — `bin/leg_runner.dart`

```dart
  final heartbeat = Duration(
    seconds: int.tryParse(env['LEG_HEARTBEAT_SECONDS'] ?? '') ?? 300,
  );
  await serve(
    InternetAddress.anyIPv4,
    8080,
    busy,
    (spec) => runLeg(spec, writer, busy, command,
        callback: callbackFor(spec), heartbeatEvery: heartbeat),
  );
```

- [ ] **Step 13: Run all Dart checks**

Run: `dart analyze --fatal-infos && dart test`
Expected: `No issues found!` and all tests pass.

- [ ] **Step 14: Commit**

```bash
git checkout -b feat/m1-leg-path
git add lib bin test
git commit -m "feat(shim): factory LegSpec and Step Functions callbacks"
```

---

### Task 3: Leg runner test harness and dispatcher

**Files:**
- Create: `tests/leg/harness.sh`, `tests/leg/bin/gh`, `tests/leg/bin/dmtools`, `tests/leg/run-all.sh`, `runner/hello.sh`, `runner/leg/lib.sh`, `tests/leg/test_lib.sh`
- Modify: `runner/run-leg.sh`, `.github/workflows/test.yml`

**Interfaces:**
- Produces (test harness, used by Tasks 4-7):
  - `setup_case` — fresh `CASE_DIR`, fake `gh`/`dmtools` first on `PATH`, empty rules/log.
  - `gh_rule <glob> <output> [exit]` — the fake `gh` prints `<output>` (printf `%b`) and exits `[exit]` (default 0) for the first rule whose glob matches the full argument string; unmatched calls print nothing, exit 0. Every call is appended to `$FAKE_GH_LOG`; a `--body-file F` argument is copied to `$CASE_DIR/body.<n>`.
  - `make_repo` — creates `$CASE_DIR/repo` (git repo with `.dmtools/config.js` naming the four runners) and `cd`s into it.
  - `assert_eq <actual> <expected> <name>`, `assert_file_contains <file> <text> <name>`, `assert_file_lacks <file> <text> <name>`, `finish`.
- Produces (runtime): `runner/leg/lib.sh` functions `log`, `warn`, `die`, `load_github_env <file>` (exports `KEY=VALUE` and `KEY<<DELIM` blocks), `labels_have <labels> <word>` (grep -qw semantics).

- [ ] **Step 1: Write the fakes and harness**

`tests/leg/bin/gh`:

```bash
#!/usr/bin/env bash
# Fake gh for leg runner tests: rules are "<glob>\t<exit>\t<output>" lines.
args="$*"
n=$(( $(wc -l < "${FAKE_GH_LOG:-/dev/null}" 2>/dev/null || echo 0) + 1 ))
printf '%s\n' "$args" >> "${FAKE_GH_LOG:-/dev/null}"
prev=""
for a in "$@"; do
  if [[ "$prev" == --body-file ]]; then cp "$a" "${CASE_DIR:-/tmp}/body.$n"; fi
  prev="$a"
done
while IFS=$'\t' read -r pattern code output; do
  [[ -n "$pattern" ]] || continue
  # shellcheck disable=SC2053
  if [[ "$args" == $pattern ]]; then
    [[ -n "$output" ]] && printf '%b\n' "$output"
    exit "$code"
  fi
done < "${FAKE_GH_RULES:-/dev/null}"
exit 0
```

`tests/leg/bin/dmtools`:

```bash
#!/usr/bin/env bash
# Fake dmtools: records its arguments, writes a trace line, exits FAKE_DMTOOLS_EXIT.
printf '%s\n' "$*" >> "${CASE_DIR:-/tmp}/dmtools.log"
[[ -n "${FA_LOG_FILE:-}" ]] && echo trace >> "$FA_LOG_FILE"
[[ -n "${FAKE_DMTOOLS_OUTPUT:-}" ]] && printf '%s\n' "$FAKE_DMTOOLS_OUTPUT"
sleep "${FAKE_DMTOOLS_SLEEP:-0}"
exit "${FAKE_DMTOOLS_EXIT:-0}"
```

`tests/leg/harness.sh`:

```bash
# shellcheck shell=bash
# Leg runner test harness: fakes on PATH, assertions, counters.
TESTS_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
ROOT="$(cd "$TESTS_DIR/../.." && pwd)"
LEG="$ROOT/runner/leg"
PASS=0
FAIL=0

setup_case() {
  CASE_DIR="$(mktemp -d)"
  export CASE_DIR
  export FAKE_GH_RULES="$CASE_DIR/gh.rules" FAKE_GH_LOG="$CASE_DIR/gh.log"
  : > "$FAKE_GH_RULES"; : > "$FAKE_GH_LOG"
  export PATH="$TESTS_DIR/bin:$ORIG_PATH"
  cd "$CASE_DIR" || exit 1
}
ORIG_PATH="${ORIG_PATH:-$PATH}"

gh_rule() { printf '%s\t%s\t%s\n' "$1" "${3:-0}" "$2" >> "$FAKE_GH_RULES"; }

make_repo() {
  mkdir -p "$CASE_DIR/repo/.dmtools" && cd "$CASE_DIR/repo" || exit 1
  git init -q -b main .
  git config user.name test; git config user.email test@example.com
  cat > .dmtools/config.js <<'JS'
module.exports = { machineAuthor: 'ai-teammate', sm: { runners: {
  bug: '.dmtools/runners/fa-bug-dev.json', story: '.dmtools/runners/fa-story-dev.json',
  review: '.dmtools/runners/fa-review.json', rework: '.dmtools/runners/fa-rework.json' } } };
JS
  git add -A && git commit -qm init
}

assert_eq() {
  if [[ "$1" == "$2" ]]; then PASS=$((PASS + 1)); else FAIL=$((FAIL + 1)); printf 'FAIL %s\n  expected: [%s]\n  actual:   [%s]\n' "$3" "$2" "$1"; fi
}
assert_file_contains() {
  if grep -qF -- "$2" "$1" 2>/dev/null; then PASS=$((PASS + 1)); else FAIL=$((FAIL + 1)); printf 'FAIL %s: %s lacks [%s]\n' "$3" "$1" "$2"; fi
}
assert_file_lacks() {
  if grep -qF -- "$2" "$1" 2>/dev/null; then FAIL=$((FAIL + 1)); printf 'FAIL %s: %s has [%s]\n' "$3" "$1" "$2"; else PASS=$((PASS + 1)); fi
}
finish() { echo "$(basename "$0"): passed=$PASS failed=$FAIL"; (( FAIL == 0 )); }
```

`tests/leg/run-all.sh`:

```bash
#!/usr/bin/env bash
# Runs every tests/leg/test_*.sh; fails if any fails.
set -u
cd "$(dirname "$0")" || exit 1
rc=0
for t in test_*.sh; do bash "$t" || rc=1; done
exit "$rc"
```

- [ ] **Step 2: Write the failing lib test** — `tests/leg/test_lib.sh`

```bash
#!/usr/bin/env bash
source "$(dirname "$0")/harness.sh"
source "$LEG/lib.sh"

setup_case
printf 'A=1\nB<<EOF_X\nline1\nline2\nEOF_X\nC=x=y\n' > env.txt
( load_github_env env.txt; printf '%s|%s|%s' "$A" "$B" "$C" > out.txt )
assert_eq "$(cat out.txt)" "1|line1
line2|x=y" "load_github_env handles KEY=VALUE and heredoc blocks"

labels="bug agent:review status:In Development"
labels_have "$labels" agent:review && r=yes || r=no
assert_eq "$r" yes "labels_have finds a word"
labels_have "$labels" review && r=yes || r=no
assert_eq "$r" yes "labels_have uses grep -w semantics like upstream"
labels_have "$labels" agent:dev && r=yes || r=no
assert_eq "$r" no "labels_have misses absent labels"
finish
```

- [ ] **Step 3: Run to verify failure**

Run: `chmod +x tests/leg/bin/* tests/leg/run-all.sh && bash tests/leg/test_lib.sh`
Expected: FAIL (`runner/leg/lib.sh: No such file`).

- [ ] **Step 4: Implement `runner/leg/lib.sh`**

```bash
# shellcheck shell=bash
# Shared helpers for the AWS-mode leg runner.
log()  { printf '[leg] %s\n' "$*" >&2; }
warn() { printf '[leg] WARN: %s\n' "$*" >&2; }
die()  { printf '[leg] ERROR: %s\n' "$*" >&2; exit 1; }

# Exports what a GitHub Actions step would have written to $GITHUB_ENV:
# KEY=VALUE lines and KEY<<DELIM ... DELIM blocks.
load_github_env() {
  local file="$1" line key delim value
  [[ -f "$file" ]] || return 0
  while IFS= read -r line || [[ -n "$line" ]]; do
    if [[ "$line" =~ ^([A-Za-z_][A-Za-z0-9_]*)\<\<(.+)$ ]]; then
      key="${BASH_REMATCH[1]}"; delim="${BASH_REMATCH[2]}"; value=""
      while IFS= read -r line && [[ "$line" != "$delim" ]]; do
        value+="${value:+$'\n'}$line"
      done
      export "$key=$value"
    elif [[ "$line" =~ ^([A-Za-z_][A-Za-z0-9_]*)=(.*)$ ]]; then
      export "${BASH_REMATCH[1]}=${BASH_REMATCH[2]}"
    fi
  done < "$file"
}

# Upstream tests labels with `grep -qw` on a space-joined list (FT 297-364).
labels_have() { grep -qw -- "$2" <<<"$1"; }
```

- [ ] **Step 5: Move the hello leg and add the dispatcher**

`git mv runner/run-leg.sh runner/hello.sh`, then create `runner/run-leg.sh`:

```bash
#!/usr/bin/env bash
# Leg dispatcher called by the shim with the LegSpec path.
set -euo pipefail
here="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
spec="$1"
if [[ "$(jq -r .leg "$spec")" == hello ]]; then
  exec "$here/hello.sh" "$spec"
fi
exec "$here/leg/factory-leg.sh" "$spec"
```

- [ ] **Step 6: Run the lib test**

Run: `chmod +x runner/run-leg.sh runner/hello.sh && bash tests/leg/test_lib.sh`
Expected: `test_lib.sh: passed=4 failed=0` (four assertions).

- [ ] **Step 7: Add the leg tests to CI** — in `.github/workflows/test.yml` add a job:

```yaml
  leg:
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@11d5960a326750d5838078e36cf38b85af677262 # v4
      - name: Leg runner tests
        run: bash tests/leg/run-all.sh
```

(ubuntu-24.04 ships bash, git, jq, node, python3, openssl.)

- [ ] **Step 8: Commit**

```bash
git add tests/leg runner .github/workflows/test.yml
git commit -m "test(leg): bash harness, fakes and leg dispatcher"
```

---

### Task 4: Guard (port of FT 109-390 + allowlist)

**Files:**
- Create: `runner/leg/guard.sh`, `tests/leg/test_guard.sh`

**Interfaces:**
- Consumes: `lib.sh`; cwd = repo checkout with `.dmtools/config.js`.
- Env in: `REPO`, `ISSUE` or `PR`, `INPUT_LEG` (dev|review|rework|empty), `INPUT_REASON`, `ACTOR` (`dispatch` in AWS mode), `SENDER`, `ALLOWED_ACTORS` (comma-separated logins).
- Produces on stdout, one per line: `run=true|false`, `runner=<path>`, `config=factory-agents/<agent>.json`, `kind=dev|review|rework`, `anchor=gh-N|pr-N`, `slot=bug|story|review|rework|`, `reason=<skip reason>`. Exit 1 only for "both/neither anchors" and missing/empty config (FT rows 1-2, 164-176).
- Skip reasons (AWS mode additions marked *): `pr-unresolved`, `pr-not-open`, `issue-unresolved`, `issue-not-open`, `agent-skip`, `blocked`, `override-refused`, `in-progress`, `no-leg`, `allowlist-missing`*, `author-not-allowed`*, `sender-not-allowed`*.

- [ ] **Step 1: Write the failing tests** — `tests/leg/test_guard.sh`

```bash
#!/usr/bin/env bash
source "$(dirname "$0")/harness.sh"

# guard <env assignments...>: runs the guard, prints its output.
guard() { env REPO=o/r ACTOR=dispatch ALLOWED_ACTORS=alice "$@" bash "$LEG/guard.sh"; }
issue_rules() { # labels assignees state title author
  gh_rule "issue view 7 --repo o/r --json labels*" "$1"
  gh_rule "issue view 7 --repo o/r --json assignees*" "$2"
  gh_rule "issue view 7 --repo o/r --json state*" "${3:-OPEN}"
  gh_rule "issue view 7 --repo o/r --json title*" "${4:-Add snake}"
  gh_rule "api repos/o/r/issues/7 --jq .user.login" "${5:-alice}"
}
get() { grep "^$1=" <<<"$2" | cut -d= -f2-; }

setup_case; make_repo
out="$(guard ISSUE=7 PR=8 2>/dev/null)"; rc=$?
assert_eq "$rc" 1 "row 1: both anchors is an error"

setup_case; make_repo
out="$(guard 2>/dev/null)"; rc=$?
assert_eq "$rc" 1 "row 2: no anchor is an error"

setup_case; make_repo; rm .dmtools/config.js
out="$(guard ISSUE=7 2>/dev/null)"; rc=$?
assert_eq "$rc" 1 "missing config is an error"

setup_case; make_repo
gh_rule "pr view 8 --repo o/r --json state*" ""
assert_eq "$(get reason "$(guard PR=8)")" pr-unresolved "row 3"

setup_case; make_repo
gh_rule "pr view 8 --repo o/r --json state*" "MERGED"
assert_eq "$(get reason "$(guard PR=8)")" pr-not-open "row 4"

setup_case; make_repo
gh_rule "pr view 8 --repo o/r --json state*" "OPEN"
gh_rule "pr view 8 --repo o/r --json labels*" "true"
assert_eq "$(get reason "$(guard PR=8)")" blocked "row 5"

setup_case; make_repo
gh_rule "pr view 8 --repo o/r --json state*" "OPEN"
gh_rule "pr view 8 --repo o/r --json labels*" "false"
gh_rule "api repos/o/r/issues/8 --jq .user.login" "alice"
out="$(guard PR=8 INPUT_REASON="SM: Rework requested")"
assert_eq "$(get run "$out")|$(get kind "$out")|$(get runner "$out")|$(get anchor "$out")" \
  "true|rework|.dmtools/runners/fa-rework.json|pr-8" "rows 6+8: reason 'rework' selects the rework leg"

setup_case; make_repo
gh_rule "pr view 8 --repo o/r --json state*" "OPEN"
gh_rule "pr view 8 --repo o/r --json labels*" "false"
gh_rule "api repos/o/r/issues/8 --jq .user.login" "alice"
out="$(guard PR=8 INPUT_LEG=review)"
assert_eq "$(get kind "$out")|$(get config "$out")" "review|factory-agents/pr_review.json" "row 9"

setup_case; make_repo
gh_rule "pr view 8 --repo o/r --json state*" "OPEN"
gh_rule "pr view 8 --repo o/r --json labels*" "false"
gh_rule "api repos/o/r/collaborators/mallory/permission -q .permission" "read"
gh_rule "api repos/o/r/issues/8 --jq .user.login" "alice"
out="$(env REPO=o/r ACTOR=labeled SENDER=mallory ALLOWED_ACTORS=alice,mallory PR=8 INPUT_LEG=rework bash "$LEG/guard.sh")"
assert_eq "$(get reason "$out")" override-refused "row 7: read-only sender cannot force rework"

setup_case; make_repo; issue_rules "agent:dev" "" "CLOSED"
assert_eq "$(get reason "$(guard ISSUE=7)")" issue-not-open "row 13"

setup_case; make_repo; issue_rules "agent:dev agent:skip" ""
assert_eq "$(get reason "$(guard ISSUE=7)")" agent-skip "row 14"

setup_case; make_repo; issue_rules "agent:dev blocked" ""
assert_eq "$(get reason "$(guard ISSUE=7)")" blocked "row 15"

setup_case; make_repo; issue_rules "" "" OPEN "[Bug] snake wraps"
out="$(guard ISSUE=7 INPUT_LEG=dev)"
assert_eq "$(get slot "$out")|$(get config "$out")|$(get anchor "$out")" "bug|factory-agents/bug_development.json|gh-7" "row 16: explicit dev, bug title"

setup_case; make_repo; issue_rules "agent:rework" ""
out="$(guard ISSUE=7)"
assert_eq "$(get runner "$out")|$(get slot "$out")|$(get kind "$out")|$(get config "$out")" \
  ".dmtools/runners/fa-rework.json||dev|factory-agents/story_development.json" "row 17 quirk: label rework records slot empty, kind dev"

setup_case; make_repo; issue_rules "agent:review" ""
assert_eq "$(get kind "$(guard ISSUE=7)")" review "row 18"

setup_case; make_repo; issue_rules "ai_developed agent:dev" ""
assert_eq "$(get reason "$(guard ISSUE=7)")" in-progress "row 19"

setup_case; make_repo; issue_rules "" "ai-teammate"
out="$(guard ISSUE=7)"
assert_eq "$(get run "$out")|$(get slot "$out")" "true|story" "row 20: assigned to the agent"

setup_case; make_repo; issue_rules "enhancement" ""
assert_eq "$(get reason "$(guard ISSUE=7)")" no-leg "row 22"

setup_case; make_repo; issue_rules "agent:dev" ""
out="$(env REPO=o/r ACTOR=dispatch ISSUE=7 bash "$LEG/guard.sh")"
assert_eq "$(get reason "$out")" allowlist-missing "missing allowlist refuses"

setup_case; make_repo; issue_rules "agent:dev" "" OPEN "Add snake" mallory
assert_eq "$(get reason "$(guard ISSUE=7)")" author-not-allowed "stranger's issue refused"

setup_case; make_repo; issue_rules "agent:dev" ""
assert_eq "$(get reason "$(guard ISSUE=7 SENDER=mallory)")" sender-not-allowed "stranger's trigger refused"

setup_case; make_repo; issue_rules 'agent:dev $(touch pwned) `id`' ""
out="$(guard ISSUE=7)"
assert_eq "$(get run "$out")" true "hostile label text is data"
[[ -e pwned ]] && r=executed || r=safe
assert_eq "$r" safe "hostile label text never executes"
finish
```

- [ ] **Step 2: Run to verify failure**

Run: `bash tests/leg/test_guard.sh`
Expected: FAIL (guard.sh missing).

- [ ] **Step 3: Implement `runner/leg/guard.sh`**

```bash
#!/usr/bin/env bash
# Leg guard: port of factory-teammate.yml guard job (FT 109-390) at
# dmtools-agentic-workflows ca33362b, plus the AWS-mode allowlist (spec
# 2026-10-09 §7). Prints key=value lines; never evaluated as shell.
set -uo pipefail
here="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
source "$here/lib.sh"
AGENT_HANDLE="${AGENT_HANDLE:-ai-teammate}"
runner="" config="" kind="" anchor="" slot=""

emit() {
  printf 'run=%s\nrunner=%s\nconfig=%s\nkind=%s\nanchor=%s\nslot=%s\nreason=%s\n' \
    "$1" "$runner" "$config" "$kind" "$anchor" "$slot" "${2:-}"
  exit 0
}
skip() { log "skip: $1"; runner="" config="" kind=""; emit false "$1"; }

# FT 154-162: exactly one anchor.
[[ -n "${ISSUE:-}" && -n "${PR:-}" ]] && die "exactly one of issue and pr"
[[ -z "${ISSUE:-}" && -z "${PR:-}" ]] && die "exactly one of issue and pr"

# FT 164-181: runners from .dmtools/config.js ('|' keeps empty fields).
[[ -f .dmtools/config.js ]] || die ".dmtools/config.js not found"
IFS='|' read -r RUNNER_BUG RUNNER_STORY RUNNER_REVIEW RUNNER_REWORK < <(node -e "
const c=require(process.cwd()+'/.dmtools/config.js');const r=(c.sm&&c.sm.runners)||{};
console.log([r.bug||r.dev||'',r.story||r.dev||'',r.review||'',r.rework||''].join('|'))")
[[ -n "$RUNNER_BUG$RUNNER_STORY$RUNNER_REVIEW$RUNNER_REWORK" ]] || die "sm.runners is empty"

# FT 192-204: #544 owner override for manual rework.
rework_override_ok() {
  [[ "${ACTOR:-}" != labeled || -z "${SENDER:-}" ]] && return 0
  local perm
  perm="$(gh api "repos/$REPO/collaborators/$SENDER/permission" -q .permission 2>/dev/null || true)"
  case "$perm" in admin|maintain|write) return 0 ;; esac
  warn "rework override by $SENDER refused (permission: ${perm:-none})"
  return 1
}

# Spec 2026-10-09 §7 (AWS mode only): deny by default.
allowed() { [[ ",${ALLOWED_ACTORS// /}," == *",$1,"* ]]; }
check_allowlist() {
  [[ -n "${ALLOWED_ACTORS// /}" ]] || skip allowlist-missing
  local author
  author="$(gh api "repos/$REPO/issues/$1" --jq .user.login 2>/dev/null || true)"
  [[ -n "$author" ]] && allowed "$author" || skip author-not-allowed
  if [[ -n "${SENDER:-}" ]] && ! allowed "$SENDER"; then skip sender-not-allowed; fi
}

slot_config() {  # FT 375-381
  case "$slot" in
    bug)    config=factory-agents/bug_development.json;   kind=dev ;;
    story)  config=factory-agents/story_development.json; kind=dev ;;
    review) config=factory-agents/pr_review.json;         kind=review ;;
    rework) config=factory-agents/pr_rework.json;         kind=dev ;;
    *)      config=factory-agents/story_development.json; kind=dev ;;
  esac
}

if [[ -n "${PR:-}" ]]; then
  anchor="pr-$PR"
  state="$(gh pr view "$PR" --repo "$REPO" --json state -q .state 2>/dev/null || true)"
  [[ -n "$state" ]] || skip pr-unresolved                        # FT 211-219
  [[ "$state" == OPEN ]] || skip pr-not-open                     # FT 220
  blocked="$(gh pr view "$PR" --repo "$REPO" --json labels \
    -q '([.labels[].name]|index("blocked"))!=null' 2>/dev/null || echo false)"
  [[ "$blocked" == true ]] && skip blocked                       # FT 227-233
  check_allowlist "$PR"
  leg="${INPUT_LEG:-}"                                           # FT 234-237
  if [[ -z "$leg" ]] && grep -qi rework <<<"${INPUT_REASON:-}"; then leg=rework; fi
  if [[ "$leg" == rework ]]; then                                # FT 238-249
    rework_override_ok || skip override-refused
    runner="$RUNNER_REWORK"; config=factory-agents/pr_rework.json; kind=rework; slot=rework
  else                                                           # FT 251-257
    runner="$RUNNER_REVIEW"; config=factory-agents/pr_review.json; kind=review; slot=review
  fi
  emit true
fi

anchor="gh-$ISSUE"                                               # FT 259
LABELS="$(gh issue view "$ISSUE" --repo "$REPO" --json labels -q '[.labels[].name]|join(" ")' 2>/dev/null)" \
  || { warn "label refresh failed"; LABELS=""; }                 # FT 271-277
ASSIGNEES="$(gh issue view "$ISSUE" --repo "$REPO" --json assignees -q '[.assignees[].login]|join(" ")' 2>/dev/null)" \
  || { warn "assignee refresh failed"; ASSIGNEES=""; }
state="$(gh issue view "$ISSUE" --repo "$REPO" --json state -q .state 2>/dev/null || true)"
[[ -n "$state" ]] || skip issue-unresolved                       # FT 285-290
[[ "$state" == OPEN ]] || skip issue-not-open                    # FT 291-295
labels_have "$LABELS" agent:skip && skip agent-skip              # FT 297
labels_have "$LABELS" blocked && skip blocked                    # FT 306
check_allowlist "$ISSUE"

bug_or_story() {                                                 # FT 319-323, 358-362
  local title
  title="$(gh issue view "$ISSUE" --repo "$REPO" --json title -q .title 2>/dev/null || true)"
  if grep -qiE '^\[?bug\]' <<<"$title" || grep -qwi bug <<<"$LABELS"; then
    runner="$RUNNER_BUG"; slot=bug
  else
    runner="$RUNNER_STORY"; slot=story
  fi
}

if [[ -n "${INPUT_LEG:-}" ]]; then                               # FT 313-326
  case "$INPUT_LEG" in
    rework) runner="$RUNNER_REWORK"; slot=rework ;;
    review) runner="$RUNNER_REVIEW"; slot=review ;;
    dev)    bug_or_story ;;
    *)      runner="" ;;
  esac
elif labels_have "$LABELS" agent:rework && ! grep -qF 'status:In Rework' <<<"$LABELS"; then
  if rework_override_ok; then runner="$RUNNER_REWORK"; slot=""; else runner=""; fi  # FT 327-338
elif labels_have "$LABELS" agent:review; then                    # FT 339-344
  runner="$RUNNER_REVIEW"; slot=review
elif grep -qE 'status:In Development|status:In Review|status:In Rework|in progress|ai_developed|ai_pr_reviewed|_wip|rework-round-' <<<"$LABELS"; then
  skip in-progress                                               # FT 345-354
elif grep -qw -- "$AGENT_HANDLE" <<<"$ASSIGNEES" || labels_have "$LABELS" agent:dev; then
  bug_or_story                                                   # FT 355-364
fi
[[ -n "$runner" ]] || skip no-leg                                # FT 387-390
slot_config                                                      # FT 365-386
emit true
```

- [ ] **Step 4: Run guard tests**

Run: `chmod +x runner/leg/guard.sh && bash tests/leg/test_guard.sh`
Expected: `test_guard.sh: passed=<n> failed=0`.

- [ ] **Step 5: Commit**

```bash
git add runner/leg/guard.sh tests/leg/test_guard.sh
git commit -m "feat(leg): port the factory guard with an AWS-mode allowlist"
```

---

### Task 5: Verdict application (port of FT 1206-1420)

**Files:**
- Create: `runner/leg/verdict.sh`, `tests/leg/test_verdict.sh`

**Interfaces:**
- Env in: `REPO`, `ISSUE`, `MACHINE_AUTHOR`, `MAX_ROUNDS` (default 2), `VERDICT_SH` (IstiN's `review-verdict.sh` or pack `loop/verdict.sh`), `RUN_OUTPUT`, `VERDICT_POLL_SECONDS` (30), `VERDICT_DEADLINE_SECONDS` (600). cwd = checkout.
- Exit 0 except "verdict missing" (exit 1), as upstream.
- Uses `$VERDICT_SH decide` (prints shell assignments `decision=… escalate=… next_round=… rounds_done=… round_labels=… diagnosis=…`) and `$VERDICT_SH threads <file>` unchanged.

- [ ] **Step 1: Write the failing tests** — `tests/leg/test_verdict.sh`

```bash
#!/usr/bin/env bash
source "$(dirname "$0")/harness.sh"

fake_verdict() { # decision escalate next_round round_labels
  cat > "$CASE_DIR/verdict.sh" <<EOF
#!/usr/bin/env bash
if [[ "\$1" == decide ]]; then
  printf 'decision=%q\nescalate=%q\nnext_round=%q\nrounds_done=1\nround_labels=%q\ndiagnosis=%q\n' '$1' '$2' '$3' '$4' 'json:none'
else
  echo "- thread one"
fi
EOF
  chmod +x "$CASE_DIR/verdict.sh"
}
run_verdict() {
  env REPO=o/r ISSUE=7 MACHINE_AUTHOR=bot MAX_ROUNDS=2 VERDICT_SH="$CASE_DIR/verdict.sh" \
    RUN_OUTPUT=/dev/null VERDICT_POLL_SECONDS=0 VERDICT_DEADLINE_SECONDS=1 bash "$LEG/verdict.sh"
}
common_rules() {
  gh_rule "issue view 7 --repo o/r --json labels*" "rework-round-1"
  gh_rule "pr list --repo o/r --state open --json number,body,headRefName" '[{"number":12,"body":"Closes #7","headRefName":"ai/gh-7"}]'
}

setup_case; make_repo; common_rules; fake_verdict rework false 2 rework-round-1
run_verdict; rc=$?
assert_eq "$rc" 0 "rework exits 0"
assert_file_contains "$FAKE_GH_LOG" "issue edit 7 --repo o/r --remove-label needs-human" "rework clears needs-human"
assert_file_contains "$FAKE_GH_LOG" "issue edit 7 --repo o/r --remove-label rework-round-1" "rework strips old rounds"
assert_file_contains "$FAKE_GH_LOG" "issue edit 7 --repo o/r --add-label agent:rework" "rework arms the agent"
assert_file_contains "$FAKE_GH_LOG" "issue edit 7 --repo o/r --add-label rework-round-2" "rework counts the round"
order="$(grep -n -e 'add-label agent:rework' -e 'add-label rework-round-2' "$FAKE_GH_LOG" | cut -d: -f1 | tr '\n' ' ')"
assert_eq "$(awk '{print ($1 < $2) ? "ok" : "bad"}' <<<"$order")" ok "agent:rework is added before the counter (FT 1357-1365)"

setup_case; make_repo; common_rules; fake_verdict rework true 3 rework-round-2
run_verdict; rc=$?
assert_eq "$rc" 0 "escalation exits 0"
assert_file_contains "$FAKE_GH_LOG" "issue edit 7 --repo o/r --add-label needs-human" "escalation asks a human"
assert_file_contains "$FAKE_GH_LOG" "issue comment 7 --repo o/r --body-file" "escalation comments"
assert_file_contains "$(ls "$CASE_DIR"/body.* | tail -1)" "Auto-rework loop capped" "escalation text"
assert_file_lacks "$FAKE_GH_LOG" "add-label agent:rework" "escalation does not re-arm"

setup_case; make_repo; common_rules; fake_verdict unknown false "" ""
run_verdict; rc=$?
assert_eq "$rc" 1 "missing verdict fails the step"
assert_file_contains "$(ls "$CASE_DIR"/body.* | tail -1)" "Review verdict missing" "missing verdict comment"

setup_case; make_repo; common_rules; fake_verdict approve false "" rework-round-1
gh_rule "pr view 12 --repo o/r --json statusCheckRollup*" 'SUCCESS\nSKIPPED'
run_verdict; rc=$?
assert_eq "$rc" 0 "green approve exits 0"
assert_file_contains "$FAKE_GH_LOG" "issue edit 7 --repo o/r --add-label pr_approved" "green approve arms the merge"

setup_case; make_repo; common_rules; fake_verdict approve false "" ""
gh_rule "pr view 12 --repo o/r --json statusCheckRollup*" 'SUCCESS\nFAILURE'
run_verdict
assert_file_lacks "$FAKE_GH_LOG" "add-label pr_approved" "red checks never approve"

setup_case; make_repo; common_rules; fake_verdict approve false "" ""
gh_rule "pr view 12 --repo o/r --json statusCheckRollup*" 'PENDING'
run_verdict
assert_file_lacks "$FAKE_GH_LOG" "add-label pr_approved" "pending checks never approve"

setup_case; make_repo; fake_verdict approve false "" ""
gh_rule "issue view 7 --repo o/r --json labels*" ""
gh_rule "pr list --repo o/r --state open --json number,body,headRefName" '[{"number":3,"body":"Closes #70","headRefName":"x"}]'
run_verdict
assert_file_lacks "$FAKE_GH_LOG" "add-label pr_approved" "approve without a linked PR labels nothing (#70 is not #7)"
finish
```

- [ ] **Step 2: Run to verify failure**

Run: `bash tests/leg/test_verdict.sh`
Expected: FAIL (verdict.sh missing).

- [ ] **Step 3: Implement `runner/leg/verdict.sh`**

```bash
#!/usr/bin/env bash
# Apply the review verdict: port of factory-teammate.yml FT 1206-1420. The
# decision itself is IstiN's review-verdict.sh (`decide`), unchanged.
set -uo pipefail
here="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
source "$here/lib.sh"
N="$ISSUE"; R="$REPO"
MAX_ROUNDS="${MAX_ROUNDS:-2}"
POLL="${VERDICT_POLL_SECONDS:-30}"; DEADLINE="${VERDICT_DEADLINE_SECONDS:-600}"
tmp="$(mktemp -d)"; trap 'rm -rf "$tmp"' EXIT

labels="$(gh issue view "$N" --repo "$R" --json labels --jq '[.labels[].name]|join(" ")' 2>/dev/null || true)"
gh pr list --repo "$R" --state open --json number,body,headRefName > "$tmp/prs.json" 2>/dev/null || echo '[]' > "$tmp/prs.json"
pr="$(jq -r --arg n "$N" '[.[] | select(
    ((.body // "") | test("(?i)\\b(closes|fixes|resolves)\\s+#" + $n + "\\b"))
    or (.headRefName | test("^" + $n + "[-_.]"))
    or (.headRefName | test("(^|[-_./])gh-0*" + $n + "$"))) | .number][0] // empty' "$tmp/prs.json")"

if [[ -n "$pr" ]]; then
  gh pr view "$pr" --repo "$R" --json comments,reviews \
    --jq '[(.comments//[])[]|{author:(.author.login//""),body}]+[(.reviews//[])[]|{author:(.author.login//""),body}]' \
    > "$tmp/pr-comments.json" 2>/dev/null || echo '[]' > "$tmp/pr-comments.json"
else
  echo '[]' > "$tmp/pr-comments.json"
fi
gh issue view "$N" --repo "$R" --json comments --jq '[(.comments//[])[]|{author:(.author.login//""),body}]' \
  > "$tmp/issue-comments.json" 2>/dev/null || echo '[]' > "$tmp/issue-comments.json"

decision="" escalate="" next_round="" rounds_done="" round_labels="" diagnosis=""
eval "$(MAX_ROUNDS="$MAX_ROUNDS" PR_REVIEW_JSON=outputs/pr_review.json PR_REVIEW_JSON_ALT="outputs/gh-$N/pr_review.json" \
  RUN_OUTPUT="$RUN_OUTPUT" PR_COMMENTS_JSON="$tmp/pr-comments.json" ISSUE_COMMENTS_JSON="$tmp/issue-comments.json" \
  MACHINE_AUTHOR="${MACHINE_AUTHOR:-}" ISSUE_LABELS="$labels" bash "$VERDICT_SH" decide)"
log "verdict: decision=$decision escalate=$escalate next_round=$next_round diagnosis=$diagnosis"

ensure_label() { gh label create "$1" --repo "$R" --color "$2" --force >/dev/null 2>&1 || true; }
strip_round_labels() { local l; for l in $round_labels; do gh issue edit "$N" --repo "$R" --remove-label "$l" || true; done; }

if [[ "$decision" == rework && "$escalate" == true ]]; then      # FT 1315-1355
  strip_round_labels
  ensure_label needs-human B60205
  gh issue edit "$N" --repo "$R" --add-label needs-human || true
  {
    echo "🛑 **Auto-rework loop capped** — ${rounds_done} automatic rework rounds without an APPROVE verdict (cap: ${MAX_ROUNDS}). The machine stopped labeling \`agent:rework\`."
    echo
    if [[ -n "$pr" ]]; then
      echo "### Unresolved review threads on PR #$pr"
      gh api graphql -F owner="${R%/*}" -F name="${R#*/}" -F number="$pr" -f query='query($owner:String!,$name:String!,$number:Int!){repository(owner:$owner,name:$name){pullRequest(number:$number){reviewThreads(first:100){nodes{isResolved path line comments(first:1){nodes{body author{login}}}}}}}}' \
        > "$tmp/threads.json" 2>/dev/null && bash "$VERDICT_SH" threads "$tmp/threads.json" || true
      echo
      echo "_If no threads are listed, the findings live in the PR general comments._"
    else
      echo "_No linked open PR found for this issue._"
    fi
    echo
    echo "**Next step (human):** fix the remaining findings directly and re-label \`agent:review\` for a fresh review — or re-label \`agent:rework\` to re-arm the agent loop with a fresh round cap."
  } > "$tmp/escalate.md"
  gh issue comment "$N" --repo "$R" --body-file "$tmp/escalate.md" || true
  exit 0
fi

if [[ "$decision" == rework ]]; then                             # FT 1357-1365 (order is load-bearing)
  gh issue edit "$N" --repo "$R" --remove-label needs-human || true
  strip_round_labels
  gh issue edit "$N" --repo "$R" --add-label agent:rework
  ensure_label "rework-round-${next_round}" BFD4F2
  gh issue edit "$N" --repo "$R" --add-label "rework-round-${next_round}"
  exit 0
fi

if [[ "$decision" != approve ]]; then                            # FT 1367-1388
  ensure_label needs-human B60205
  gh issue edit "$N" --repo "$R" --add-label needs-human || true
  {
    echo "🛑 **Review verdict missing** — the review leg finished, but no verdict (APPROVE / REQUEST_CHANGES / BLOCK) was found in any source, so the cycle cannot be labeled."
    echo
    echo "Per-source diagnosis: \`${diagnosis}\`"
    echo
    echo "_Next step (human):_ read the review leg's run output, then label \`pr_approved\` (approve) or \`agent:rework\` (changes requested) — or re-run the review leg."
  } > "$tmp/missing.md"
  gh issue comment "$N" --repo "$R" --body-file "$tmp/missing.md" || true
  exit 1
fi

[[ -n "$pr" ]] || { warn "approve but no linked open PR for #$N"; exit 0; }   # FT 1390-1393

deadline=$((SECONDS + DEADLINE))                                 # FT 1394-1416
while :; do
  states="$(gh pr view "$pr" --repo "$R" --json statusCheckRollup \
    --jq '.statusCheckRollup[]|if .status and .status != "COMPLETED" then "PENDING" else (.conclusion // .state // "PENDING") end' 2>/dev/null || true)"
  if grep -qE 'FAILURE|ERROR|CANCELLED|TIMED_OUT|ACTION_REQUIRED|STARTUP_FAILURE|STALE' <<<"$states"; then
    warn "PR #$pr has failing checks; not approving"; exit 0
  fi
  if [[ -n "$states" ]] && ! grep -qvE '^(SUCCESS|SKIPPED|NEUTRAL)$' <<<"$states"; then break; fi
  (( SECONDS >= deadline )) && { warn "PR #$pr checks still pending; not approving"; exit 0; }
  sleep "$POLL"
done
strip_round_labels                                               # FT 1417-1420
gh issue edit "$N" --repo "$R" --remove-label needs-human || true
gh issue edit "$N" --repo "$R" --add-label pr_approved
```

- [ ] **Step 4: Run verdict tests**

Run: `chmod +x runner/leg/verdict.sh && bash tests/leg/test_verdict.sh`
Expected: `test_verdict.sh: passed=<n> failed=0`.

- [ ] **Step 5: Commit**

```bash
git add runner/leg/verdict.sh tests/leg/test_verdict.sh
git commit -m "feat(leg): port review verdict application"
```

---

### Task 6: Markers, sessions and token usage

**Files:**
- Create: `runner/leg/markers.sh`, `runner/leg/session.sh`, `runner/leg/tokens.sh`, `tests/leg/test_markers.sh`, `tests/leg/test_session.sh`, `tests/leg/test_tokens.sh`

**Interfaces:**
- `markers.sh start|done` — env `REPO ISSUE KIND RUNNER RUN_URL` (+ `OUTCOME` success|failure|cancelled for `done`). Pure helpers sourced by tests: `marker_add <file> <line>`, `marker_finish <file> <run_url> <mark> <hh:mm>`.
- `session.sh restore|persist|quarantine` — env `ANCHOR CONFIG_NAME RUN_ID FA_LOG_FILE RUN_OUTPUT`, cwd = checkout with remote `origin`.
- `tokens.sh` — env `REPO KIND CONFIG_NAME ANCHOR RUN_OUTPUT`; never fails (exit 0). Helper `token_sums <file>` prints `<prompt> <completion>`.

- [ ] **Step 1: Write the failing tests**

`tests/leg/test_markers.sh`:

```bash
#!/usr/bin/env bash
source "$(dirname "$0")/harness.sh"
source "$LEG/markers.sh"

setup_case
printf 'Build a snake game.' > body.md
marker_add body.md "- [ ] ▶ dev (fa-story-dev) · [run](https://x/run/1) · started 10:00 ⏳"
assert_eq "$(cat body.md)" "Build a snake game.

## Machine jobs
- [ ] ▶ dev (fa-story-dev) · [run](https://x/run/1) · started 10:00 ⏳" "first marker adds the section"
marker_add body.md "- [ ] ▶ review (fa-review) · [run](https://x/run/2) · started 11:00 ⏳"
assert_eq "$(grep -c '^## Machine jobs' body.md)" 1 "second marker reuses the section"
marker_finish body.md "https://x/run/1" "✅" "10:30"
assert_file_contains body.md "- [x] ▶ dev (fa-story-dev) · [run](https://x/run/1) · started 10:00 · done 10:30 ✅" "done marker flips only its line"
assert_file_contains body.md "- [ ] ▶ review (fa-review) · [run](https://x/run/2) · started 11:00 ⏳" "other lines untouched"

setup_case
printf 'Body with `code` and $(echo x) and "quotes"' > body.md
marker_add body.md "- [ ] ▶ dev (a) · [run](u) · started 01:00 ⏳"
assert_file_contains body.md 'Body with `code` and $(echo x) and "quotes"' "issue text is preserved verbatim"
finish
```

`tests/leg/test_session.sh`:

```bash
#!/usr/bin/env bash
source "$(dirname "$0")/harness.sh"

setup_case
git init -q --bare origin.git
make_repo; git remote add origin "$CASE_DIR/origin.git"; git push -q origin main
mkdir -p .dmtools/fa-sessions/x && echo s1 > .dmtools/fa-sessions/x/state
env ANCHOR=gh-7 CONFIG_NAME=story_development bash "$LEG/session.sh" persist
assert_eq "$(git -C "$CASE_DIR/origin.git" show fa-sess/gh-7:.dmtools/fa-sessions/x/state)" s1 "persist pushes the session branch"

cd "$CASE_DIR" && rm -rf repo && git clone -q origin.git repo && cd repo || exit 1
env ANCHOR=gh-7 bash "$LEG/session.sh" restore
assert_eq "$(cat .dmtools/fa-sessions/x/state)" s1 "restore brings the session back"
assert_eq "$(git diff --cached --name-only | wc -l | tr -d ' ')" 0 "restore leaves nothing staged"

touch -d '10 minutes ago' trace.log; : > run-output.txt
env ANCHOR=gh-7 RUN_ID=42 FA_LOG_FILE=trace.log RUN_OUTPUT=run-output.txt bash "$LEG/session.sh" quarantine
[[ -d .dmtools/fa-sessions-quarantine/run-42 ]] && r=moved || r=kept
assert_eq "$r" moved "silent trace quarantines the session"
assert_file_contains run-output.txt "FA-SESSION-QUARANTINE" "quarantine is logged"
git -C "$CASE_DIR/origin.git" rev-parse --verify -q fa-sess/gh-7 >/dev/null && r=present || r=deleted
assert_eq "$r" deleted "quarantine deletes the remote session branch"
finish
```

`tests/leg/test_tokens.sh`:

```bash
#!/usr/bin/env bash
source "$(dirname "$0")/harness.sh"
source "$LEG/tokens.sh"

setup_case
printf 'x fa-tokens: {"input": 100, "output": 20}\nnoise\nfa-tokens: {"input": 5, "output": {"nested": 1}, "x": 1}\n' > out.txt
assert_eq "$(token_sums out.txt)" "105 20" "sums input/output across lines"

setup_case
printf 'fa-tokens: {"input": 10, "output": 2}\n' > out.txt
gh_rule "api repos/o/r --jq .default_branch" "main"
gh_rule "api repos/o/r/contents/data/fa-tokens.json?ref=factory-data" '{"sha":"abc","content":"e30="}'
env REPO=o/r KIND=dev CONFIG_NAME=story_development ANCHOR=gh-7 RUN_OUTPUT=out.txt bash "$LEG/tokens.sh"; rc=$?
assert_eq "$rc" 0 "publish exits 0"
assert_file_contains "$FAKE_GH_LOG" "-X PUT repos/o/r/contents/data/fa-tokens.json" "publish PUTs the ledger"
assert_file_contains "$FAKE_GH_LOG" "message=leg tokens — issue-7 dev [skip ci]" "commit message per FT"

setup_case
: > out.txt
env REPO=o/r KIND=dev CONFIG_NAME=x ANCHOR=gh-7 RUN_OUTPUT=out.txt bash "$LEG/tokens.sh"; rc=$?
assert_eq "$rc|$(wc -l < "$FAKE_GH_LOG" | tr -d ' ')" "0|0" "no tokens, no calls"
finish
```

- [ ] **Step 2: Run to verify failure**

Run: `bash tests/leg/test_markers.sh; bash tests/leg/test_session.sh; bash tests/leg/test_tokens.sh`
Expected: FAIL (scripts missing).

- [ ] **Step 3: Implement `runner/leg/markers.sh`**

```bash
#!/usr/bin/env bash
# Job markers in the issue body: port of FT 856-886 (start) and 1167-1204 (done).
set -uo pipefail

marker_add() {  # <body file> <line>
  printf '%s\n' "$(cat "$1")" > "$1"
  if grep -q '^## Machine jobs' "$1"; then
    printf '%s\n' "$2" >> "$1"
  else
    printf '\n## Machine jobs\n%s\n' "$2" >> "$1"
  fi
}

marker_finish() {  # <body file> <run url> <mark> <hh:mm>
  awk -v url="($2)" -v mark="$3" -v now="$4" '
    !done && index($0, url) && /^- \[ \] / {
      sub(/^- \[ \] /, "- [x] ")
      if (match($0, /started [0-9][0-9]:[0-9][0-9]/)) $0 = substr($0, 1, RSTART + RLENGTH - 1) " · done " now " " mark
      done = 1
    }
    { print }' "$1" > "$1.new" && mv "$1.new" "$1"
}

if [[ "${BASH_SOURCE[0]}" == "$0" ]]; then
  here="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
  source "$here/lib.sh"
  f="$(mktemp)"; trap 'rm -f "$f" "$f.orig"' EXIT
  gh issue view "$ISSUE" --repo "$REPO" --json body -q .body > "$f" || { warn "marker: cannot read #$ISSUE"; exit 0; }
  case "$1" in
    start)
      leg="${KIND:-dev} ($(basename "${RUNNER:-agent}" .json))"
      marker_add "$f" "- [ ] ▶ ${leg} · [run](${RUN_URL}) · started $(date -u '+%H:%M') ⏳" ;;
    done)
      case "${OUTCOME:-}" in success) mark="✅" ;; failure) mark="❌" ;; cancelled) mark="🚫" ;; *) mark="⚠️ ${OUTCOME:-unknown}" ;; esac
      cp "$f" "$f.orig"
      marker_finish "$f" "$RUN_URL" "$mark" "$(date -u '+%H:%M')"
      cmp -s "$f" "$f.orig" && { warn "marker: no start line for $RUN_URL"; exit 0; } ;;
  esac
  gh issue edit "$ISSUE" --repo "$REPO" --body-file "$f" || warn "marker: edit failed"
fi
```

- [ ] **Step 4: Implement `runner/leg/session.sh`**

```bash
#!/usr/bin/env bash
# fa session store on branch fa-sess/<anchor>: port of FT 587-602 (restore),
# 1482-1538 (quarantine), 1571-1605 (persist).
set -uo pipefail
here="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
source "$here/lib.sh"
B="fa-sess/${ANCHOR}"

case "$1" in
  restore)
    if git fetch -q --depth 1 origin "refs/heads/$B" 2>/dev/null; then
      git checkout FETCH_HEAD -- .dmtools/fa-sessions 2>/dev/null && git restore --staged .dmtools/fa-sessions
      log "session restored from $B"
    else
      log "no session branch $B; fresh session"
    fi ;;
  quarantine)
    [[ -f "${FA_LOG_FILE:-}" ]] || exit 0
    age=$(( $(date +%s) - $(stat -c %Y "$FA_LOG_FILE") ))
    (( age < ${FA_TRACE_QUARANTINE_SILENCE_MINUTES:-5} * 60 )) && exit 0
    [[ -d .dmtools/fa-sessions ]] || exit 0
    mkdir -p .dmtools/fa-sessions-quarantine
    mv .dmtools/fa-sessions ".dmtools/fa-sessions-quarantine/run-${RUN_ID}"
    echo "🧟 FA-SESSION-QUARANTINE: session moved aside after a silent trace (${age}s)" >> "$RUN_OUTPUT"
    git push -q origin --delete "$B" 2>/dev/null || warn "could not delete $B" ;;
  persist)
    [[ -n "$(find .dmtools/fa-sessions -type f 2>/dev/null | head -1)" ]] || exit 0
    git config user.name "dm.ai"; git config user.email "dm.ai@epam.com"
    snap="$(mktemp -d)"; cp -a .dmtools/fa-sessions "$snap/"
    git switch -q --orphan "$B" 2>/dev/null || git switch -q -C "$B"
    git rm -rq --cached . >/dev/null 2>&1 || true
    rm -rf .dmtools/fa-sessions; mkdir -p .dmtools; cp -a "$snap/fa-sessions" .dmtools/
    git add -f .dmtools/fa-sessions
    git diff --cached --quiet && exit 0
    git commit -qm "fa sessions: GH-${ANCHOR} (${CONFIG_NAME:-agent}) [skip ci]"
    git push -q origin "HEAD:$B" --force || warn "session push failed" ;;
  *) die "usage: session.sh restore|persist|quarantine" ;;
esac
```

- [ ] **Step 5: Implement `runner/leg/tokens.sh`**

```bash
#!/usr/bin/env bash
# Token usage ledger on branch factory-data: port of FT 1061-1165. Never fails.
set -uo pipefail

token_sums() {  # <run output> -> "<prompt> <completion>"
  python3 - "$1" <<'PY'
import json, re, sys
text = open(sys.argv[1], encoding="utf-8", errors="replace").read()
p = c = 0
for m in re.finditer(r"fa-tokens:\s*", text):
    i, depth = m.end(), 0
    if i >= len(text) or text[i] != "{":
        continue
    for j in range(i, len(text)):
        depth += {"{": 1, "}": -1}.get(text[j], 0)
        if depth == 0:
            try:
                d = json.loads(text[i:j + 1])
                p += d.get("input", 0) if isinstance(d.get("input"), int) else 0
                c += d.get("output", 0) if isinstance(d.get("output"), int) else 0
            except ValueError:
                pass
            break
print(p, c)
PY
}

if [[ "${BASH_SOURCE[0]}" == "$0" ]]; then
  here="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
  source "$here/lib.sh"
  grep -q 'fa-tokens:' "$RUN_OUTPUT" 2>/dev/null || exit 0
  read -r prompt completion < <(token_sums "$RUN_OUTPUT")
  leg="${KIND:-dev}"; [[ "${CONFIG_NAME:-}" == *rework* ]] && leg=rework
  key="${ANCHOR/#gh-/issue-}"
  row="$(jq -nc --arg leg "$leg" --arg at "$(date -u +%FT%TZ)" --argjson p "$prompt" --argjson c "$completion" \
    '{leg:$leg, at:$at, prompt:$p, completion:$c, total:($p+$c)}')"
  cutoff="$(date -u -d '14 days ago' +%FT%TZ)"
  default="$(gh api "repos/$REPO" --jq .default_branch 2>/dev/null || echo main)"
  sha0="$(gh api "repos/$REPO/git/ref/heads/$default" --jq .object.sha 2>/dev/null || true)"
  [[ -n "$sha0" ]] && gh api -X POST "repos/$REPO/git/refs" -f ref=refs/heads/factory-data -f sha="$sha0" >/dev/null 2>&1 || true
  for attempt in 1 2 3; do
    cur="$(gh api "repos/$REPO/contents/data/fa-tokens.json?ref=factory-data" 2>/dev/null || echo '{}')"
    sha="$(jq -r '.sha // empty' <<<"$cur")"
    body="$(jq -r '.content // "e30="' <<<"$cur" | base64 -d 2>/dev/null)"
    [[ -n "$body" ]] || body='{}'
    new="$(jq -c --arg key "$key" --argjson row "$row" --arg cutoff "$cutoff" \
      '.[$key] = ((.[$key] // []) + [$row]) | with_entries(.value |= (map(select((.at // "") >= $cutoff)) | .[-40:])) | with_entries(select(.value | length > 0))' <<<"$body")"
    args=(-X PUT "repos/$REPO/contents/data/fa-tokens.json" -f branch=factory-data
      -f "message=leg tokens — $key $leg [skip ci]" -f "content=$(printf '%s' "$new" | base64 -w0)")
    [[ -n "$sha" ]] && args+=(-f "sha=$sha")
    gh api "${args[@]}" >/dev/null 2>&1 && exit 0
    sleep "$((attempt * 2))"
  done
  warn "token ledger update failed"
  exit 0
fi
```

- [ ] **Step 6: Run the three test files**

Run: `chmod +x runner/leg/*.sh && bash tests/leg/test_markers.sh && bash tests/leg/test_session.sh && bash tests/leg/test_tokens.sh`
Expected: each file reports `failed=0`.

- [ ] **Step 7: Commit**

```bash
git add runner/leg tests/leg
git commit -m "feat(leg): port job markers, session store and token ledger"
```

---

### Task 7: App token and the leg itself (`factory-leg.sh`)

**Files:**
- Create: `runner/leg/app-token.sh`, `runner/leg/factory-leg.sh`, `tests/leg/test_app_token.sh`, `tests/leg/test_factory_leg.sh`, `tests/leg/fixtures/factory/setup/{fa-session.sh,cache.sh,review-verdict.sh}`, `tests/leg/fixtures/kit/kit/{git-push-guard.sh,install-source-git-credentials.sh}`

**Interfaces:**
- `app-token.sh <owner/name>` — env `APP_ID`, `INSTALLATION_ID`, `APP_PRIVATE_KEY` (PEM), optional `GITHUB_API_URL` (default `https://api.github.com`). Prints the token; exit 1 with a message on failure. Helper `app_jwt` (sourced) prints the RS256 JWT.
- `factory-leg.sh <spec.json>` — exit 0 on ok/skip, non-zero on failure; exit code 3 means `infra_error` before any GitHub write (the shim reports it as `agent_failed` with that exit code; the state machine only retries `infra_error`, so this is deliberate: a broken secret is not retried).
- Env seams (tests only): `LEG_SECRETS=env` (use `APP_*` and provider keys from the environment instead of Secrets Manager), `LEG_TOKEN` (skip minting), `LEG_CLONE_URL`, `FACTORY_ROOT` (default `/opt/factory-setup`), `FACTORY_KIT` (default `/opt/factory-kit`), `LEG_WORK_ROOT`.
- Production env (set on the runtime, Task 9): `FA_AC_GITHUB_APP_SECRET=fa-ac-github-app`, `FA_AC_LLM_SECRET=fa-ac-llm-keys`, `DMTOOLS_PACK_REGISTRY`, `MAX_AUTO_REWORK_ROUNDS=2`.
- Secret JSON shapes: `fa-ac-github-app` = `{"app_id":"…","installation_id":"…","private_key":"-----BEGIN…"}`; `fa-ac-llm-keys` = `{"ZAI_CODE_KEY":"…","KIMI_REVIEW_KEY":"…"}` (any `^[A-Z][A-Z0-9_]*$` keys).

- [ ] **Step 1: Write fixture scripts** (stand-ins for IstiN's; behaviour minimal and documented)

`tests/leg/fixtures/factory/setup/fa-session.sh`:
```bash
#!/usr/bin/env bash
# Test stand-in for IstiN's fa-session.sh env: writes the two exports.
[[ "$1" == env ]] || exit 0
printf 'FA_SESSION_NAME=fa-test-%s\nFA_SESSION_ROOT=%s/.dmtools/fa-sessions\n' "$AI_TEAMMATE_CONCURRENCY_KEY" "$PWD" >> "$GITHUB_ENV"
```
`tests/leg/fixtures/factory/setup/cache.sh`:
```bash
#!/usr/bin/env bash
# Test stand-in: `cache.sh keys fa-session` sources fa-session.sh env.
[[ "$1 $2" == "keys fa-session" ]] && bash "$(dirname "$0")/fa-session.sh" env
exit 0
```
`tests/leg/fixtures/factory/setup/review-verdict.sh`:
```bash
#!/usr/bin/env bash
[[ "$1" == decide ]] && printf 'decision=rework\nescalate=false\nnext_round=1\nrounds_done=0\nround_labels=\ndiagnosis=test\n'
exit 0
```
`tests/leg/fixtures/kit/kit/git-push-guard.sh`:
```bash
#!/usr/bin/env bash
exec /usr/bin/git "$@"
```
`tests/leg/fixtures/kit/kit/install-source-git-credentials.sh`:
```bash
#!/usr/bin/env bash
echo "creds installed" >> "${CASE_DIR:-/tmp}/creds.log"
```

- [ ] **Step 2: Write failing tests**

`tests/leg/test_app_token.sh`:
```bash
#!/usr/bin/env bash
source "$(dirname "$0")/harness.sh"
source "$LEG/app-token.sh"

setup_case
openssl genrsa -out key.pem 2048 2>/dev/null
openssl rsa -in key.pem -pubout -out pub.pem 2>/dev/null
jwt="$(APP_ID=123 APP_PRIVATE_KEY="$(cat key.pem)" app_jwt)"
IFS=. read -r h p s <<<"$jwt"
b64d() { local x="${1//-/+}"; x="${x//_//}"; while (( ${#x} % 4 )); do x+="="; done; base64 -d <<<"$x"; }
assert_eq "$(b64d "$h" | jq -r .alg)" RS256 "header alg"
assert_eq "$(b64d "$p" | jq -r .iss)" 123 "issuer is the app id"
now=$(date +%s); exp=$(b64d "$p" | jq -r .exp)
assert_eq "$(( exp - now <= 600 && exp > now ))" 1 "expires within 10 minutes"
printf '%s' "$h.$p" > signed.txt; b64d "$s" > sig.bin
openssl dgst -sha256 -verify pub.pem -signature sig.bin signed.txt >/dev/null 2>&1 && r=valid || r=invalid
assert_eq "$r" valid "signature verifies with the public key"

out="$(APP_ID=123 INSTALLATION_ID=9 APP_PRIVATE_KEY="not a key" bash "$LEG/app-token.sh" o/r 2>&1)"; rc=$?
assert_eq "$rc" 1 "bad key fails"
finish
```

`tests/leg/test_factory_leg.sh`:
```bash
#!/usr/bin/env bash
source "$(dirname "$0")/harness.sh"

origin_with_issue_repo() {
  git init -q --bare "$CASE_DIR/origin.git"
  make_repo; mkdir -p .dmtools/runners; echo '{}' > .dmtools/runners/fa-story-dev.json
  git add -A; git commit -qm runners
  git remote add origin "$CASE_DIR/origin.git"; git push -q origin main; cd "$CASE_DIR" || exit 1
}
spec() { jq -n '{sessionId:("s"*36), leg:"dev", maxSeconds:3300, callback:{type:"s3"},
  repo:"o/r", issue:7, reason:"test", ref:"main", sender:"alice", runUrl:"https://example/run/1"}' > "$CASE_DIR/spec.json"; }
run_leg() {
  env LEG_SECRETS=env LEG_TOKEN=t0k LEG_CLONE_URL="$CASE_DIR/origin.git" LEG_WORK_ROOT="$CASE_DIR/work" \
    FACTORY_ROOT="$TESTS_DIR/fixtures/factory" FACTORY_KIT="$TESTS_DIR/fixtures/kit" \
    "$@" bash "$LEG/factory-leg.sh" "$CASE_DIR/spec.json"
}
issue_ok_rules() {
  gh_rule "api repos/o/r/actions/variables/AGENTCORE_ALLOWED_ACTORS --jq .value" "alice"
  gh_rule "issue view 7 --repo o/r --json labels*" "agent:dev"
  gh_rule "issue view 7 --repo o/r --json assignees*" ""
  gh_rule "issue view 7 --repo o/r --json state*" "OPEN"
  gh_rule "issue view 7 --repo o/r --json title*" "Add snake"
  gh_rule "api repos/o/r/issues/7 --jq .user.login" "alice"
  gh_rule "issue view 7 --repo o/r --json body*" "Build it."
  gh_rule "issue view 7 --repo o/r --json number,title,body,url,state,labels*" '{"key":"gh-7"}'
}

setup_case; origin_with_issue_repo; spec; issue_ok_rules
run_leg; rc=$?
assert_eq "$rc" 0 "dev leg succeeds"
assert_file_contains "$CASE_DIR/dmtools.log" "run .dmtools/runners/fa-story-dev.json" "runs the story runner"
assert_file_contains "$CASE_DIR/dmtools.log" '--metadata {"contextId":"gh-7"}' "passes the anchor as context"
assert_file_contains "$CASE_DIR/dmtools.log" "--ciRunUrl https://example/run/1" "passes the run url"
assert_file_contains "$FAKE_GH_LOG" "issue edit 7 --repo o/r --add-label agent:review" "hands over to review"
assert_file_contains "$FAKE_GH_LOG" "issue comment 7 --repo o/r --body 🤖 AI Teammate run success: https://example/run/1" "links the run"
assert_file_contains "$CASE_DIR/creds.log" "creds installed" "installs git credentials via IstiN's kit"

setup_case; origin_with_issue_repo; spec; issue_ok_rules
run_leg FAKE_DMTOOLS_EXIT=2; rc=$?
assert_eq "$rc" 2 "agent failure is the leg's exit code"
assert_file_contains "$FAKE_GH_LOG" "issue edit 7 --repo o/r --add-label agent:review" "handoff also runs after failure (FT 1607)"
assert_file_contains "$FAKE_GH_LOG" "AI Teammate run failure" "run link reports failure"

setup_case; origin_with_issue_repo; spec
gh_rule "api repos/o/r/actions/variables/AGENTCORE_ALLOWED_ACTORS --jq .value" "" 1
gh_rule "issue view 7 --repo o/r --json state*" "OPEN"
gh_rule "issue view 7 --repo o/r --json labels*" "agent:dev"
run_leg; rc=$?
assert_eq "$rc" 0 "missing allowlist is a clean skip"
[[ -f "$CASE_DIR/dmtools.log" ]] && r=ran || r=skipped
assert_eq "$r" skipped "agent never runs without an allowlist"

setup_case; origin_with_issue_repo; spec
out="$(env LEG_SECRETS=env APP_ID= APP_PRIVATE_KEY= INSTALLATION_ID= LEG_CLONE_URL="$CASE_DIR/origin.git" \
  LEG_WORK_ROOT="$CASE_DIR/work" bash "$LEG/factory-leg.sh" "$CASE_DIR/spec.json" 2>&1)"; rc=$?
assert_eq "$rc" 3 "missing App secret is an early infra failure"
assert_eq "$(wc -l < "$FAKE_GH_LOG" | tr -d ' ')" 0 "no GitHub call before the token exists"
finish
```

- [ ] **Step 3: Run to verify failure**

Run: `chmod +x tests/leg/fixtures/*/setup/*.sh tests/leg/fixtures/kit/kit/*.sh && bash tests/leg/test_app_token.sh; bash tests/leg/test_factory_leg.sh`
Expected: FAIL (scripts missing).

- [ ] **Step 4: Implement `runner/leg/app-token.sh`**

```bash
#!/usr/bin/env bash
# GitHub App installation token for one repo, valid one hour.
set -uo pipefail

b64url() { openssl base64 -A | tr '+/' '-_' | tr -d '='; }

app_jwt() {
  local now header payload key
  now="$(date +%s)"
  header="$(printf '{"alg":"RS256","typ":"JWT"}' | b64url)"
  payload="$(printf '{"iat":%d,"exp":%d,"iss":"%s"}' $((now - 60)) $((now + 540)) "$APP_ID" | b64url)"
  key="$(mktemp)"; printf '%s\n' "$APP_PRIVATE_KEY" > "$key"
  local sig
  sig="$(printf '%s' "$header.$payload" | openssl dgst -sha256 -sign "$key" 2>/dev/null | b64url)"
  rm -f "$key"
  [[ -n "$sig" ]] || return 1
  printf '%s.%s.%s' "$header" "$payload" "$sig"
}

if [[ "${BASH_SOURCE[0]}" == "$0" ]]; then
  repo="${1:?usage: app-token.sh owner/name}"
  jwt="$(app_jwt)" || { echo "app-token: cannot sign the JWT (bad private key?)" >&2; exit 1; }
  resp="$(curl -fsS -X POST -H "Authorization: Bearer $jwt" -H 'Accept: application/vnd.github+json' \
    "${GITHUB_API_URL:-https://api.github.com}/app/installations/${INSTALLATION_ID}/access_tokens" \
    -d "$(jq -nc --arg r "${repo#*/}" '{repositories:[$r]}')" 2>&1)" \
    || { echo "app-token: GitHub refused the token request: $resp" >&2; exit 1; }
  token="$(jq -r '.token // empty' <<<"$resp")"
  [[ -n "$token" ]] || { echo "app-token: no token in the response" >&2; exit 1; }
  printf '%s' "$token"
fi
```

- [ ] **Step 5: Implement `runner/leg/factory-leg.sh`**

```bash
#!/usr/bin/env bash
# One factory leg in AgentCore (spec 2026-10-09 §5.7). Ports the GitHub
# Actions glue of factory-teammate.yml; decisions stay in IstiN's scripts.
set -uo pipefail
here="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
source "$here/lib.sh"
spec="$1"
FACTORY_ROOT="${FACTORY_ROOT:-/opt/factory-setup}"
FACTORY_KIT="${FACTORY_KIT:-/opt/factory-kit}"
infra() { printf '[leg] INFRA: %s\n' "$*" >&2; exit 3; }

REPO="$(jq -r .repo "$spec")"
ISSUE="$(jq -r '.issue // empty' "$spec")"
PR="$(jq -r '.pr // empty' "$spec")"
INPUT_LEG="$(jq -r '.leg | if . == "auto" then "" else . end' "$spec")"
INPUT_REASON="$(jq -r '.reason // ""' "$spec")"
REF="$(jq -r '.ref // ""' "$spec")"
SENDER="$(jq -r '.sender // ""' "$spec")"
SESSION_ID="$(jq -r .sessionId "$spec")"
CI_RUN_URL="$(jq -r --arg d "local://leg/$SESSION_ID" '.runUrl // $d' "$spec")"
export REPO GITHUB_REPOSITORY="$REPO"

# 1. Secrets (Secrets Manager in AgentCore; the environment in tests).
if [[ "${LEG_SECRETS:-aws}" == aws ]]; then
  app="$(aws secretsmanager get-secret-value --secret-id "${FA_AC_GITHUB_APP_SECRET:-fa-ac-github-app}" --query SecretString --output text 2>/dev/null)" \
    || infra "cannot read the GitHub App secret"
  APP_ID="$(jq -r '.app_id // empty' <<<"$app")"; INSTALLATION_ID="$(jq -r '.installation_id // empty' <<<"$app")"
  APP_PRIVATE_KEY="$(jq -r '.private_key // empty' <<<"$app")"
  llm="$(aws secretsmanager get-secret-value --secret-id "${FA_AC_LLM_SECRET:-fa-ac-llm-keys}" --query SecretString --output text 2>/dev/null)" \
    || infra "cannot read the LLM keys secret"
  while IFS=$'\t' read -r k v; do
    [[ "$k" =~ ^[A-Z][A-Z0-9_]*$ ]] && export "$k=$v"
  done < <(jq -r 'to_entries[] | [.key, (.value|tostring)] | @tsv' <<<"$llm")
fi

# 2. Token.
if [[ -n "${LEG_TOKEN:-}" ]]; then
  GH_TOKEN="$LEG_TOKEN"
else
  [[ -n "${APP_ID:-}" && -n "${INSTALLATION_ID:-}" && -n "${APP_PRIVATE_KEY:-}" ]] || infra "GitHub App secret is incomplete"
  export APP_ID INSTALLATION_ID APP_PRIVATE_KEY
  GH_TOKEN="$(bash "$here/app-token.sh" "$REPO")" || infra "cannot mint a GitHub App token for $REPO"
fi
export GH_TOKEN SOURCE_GITHUB_TOKEN="$GH_TOKEN"

# 3. Clone (FT 412-422).
work="${LEG_WORK_ROOT:-$(mktemp -d)}"; mkdir -p "$work/tmp"
export RUNNER_TEMP="$work/tmp" GITHUB_ENV="$work/github_env" GITHUB_OUTPUT="$work/github_output"
: > "$GITHUB_ENV"; : > "$GITHUB_OUTPUT"
url="${LEG_CLONE_URL:-https://github.com/$REPO.git}"
auth="AUTHORIZATION: basic $(printf 'x-access-token:%s' "$GH_TOKEN" | base64 -w0)"
git -c "http.https://github.com/.extraheader=$auth" clone -q ${REF:+--branch "$REF"} "$url" "$work/repo" \
  || infra "cannot clone $REPO"
cd "$work/repo" || infra "no checkout"
export GITHUB_WORKSPACE="$PWD"

# 4. Guard (parsed line by line, never evaluated).
ALLOWED_ACTORS="$(gh api "repos/$REPO/actions/variables/AGENTCORE_ALLOWED_ACTORS" --jq .value 2>/dev/null || true)"
gout="$(env ISSUE="$ISSUE" PR="$PR" INPUT_LEG="$INPUT_LEG" INPUT_REASON="$INPUT_REASON" ACTOR=dispatch \
  SENDER="$SENDER" ALLOWED_ACTORS="$ALLOWED_ACTORS" bash "$here/guard.sh")" || exit 1
declare -A g=()
while IFS='=' read -r k v; do [[ -n "$k" ]] && g[$k]="$v"; done <<<"$gout"
if [[ "${g[run]:-false}" != true ]]; then log "guard: no run (${g[reason]:-})"; exit 0; fi
RUNNER="${g[runner]}"; KIND="${g[kind]}"; ANCHOR="${g[anchor]}"
CONFIG_NAME="$(basename "${g[config]}" .json)"
export RUNNER KIND ANCHOR CONFIG_NAME RUN_URL="$CI_RUN_URL" ISSUE PR

# 5. Setup.
printf '%s\n' .codegraph/ .dmtools/fa-trace.log .dmtools/run-output.txt .dmtools/stall-capture.log \
  .dmtools/fa-sessions/ .dmtools/copilot-sessions/ .fah/bash_jobs/ factory-agents/ >> .git/info/exclude   # FT 798-815
pack="$(ls -d "$HOME"/.dmtools/packs/"$CONFIG_NAME"-*/launch.json 2>/dev/null | sort -V | tail -1)"      # FT 540-577
for c in "$pack" "$FACTORY_ROOT/$CONFIG_NAME.json" "$FACTORY_ROOT/configs/$CONFIG_NAME.json"; do
  [[ -n "$c" && -f "$c" ]] && { export AI_TEAMMATE_CONFIG_FILE="$c"; break; }
done
export AI_TEAMMATE_DISPLAY_KEY="GH-$ANCHOR" AI_TEAMMATE_CONCURRENCY_KEY="$ANCHOR"                         # FT 579-585
bash "$FACTORY_ROOT/setup/cache.sh" keys fa-session >/dev/null 2>&1 || warn "session keys failed"
load_github_env "$GITHUB_ENV"
bash "$here/session.sh" restore                                                                         # FT 587-602
creds="$FACTORY_KIT/kit/install-source-git-credentials.sh"                                              # FT 719-726
[[ -f "$creds" ]] || creds="$FACTORY_ROOT/setup/install-source-git-credentials.sh"
bash "$creds" || warn "credential install failed"
if [[ -n "$ISSUE" ]]; then                                                                              # FT 817-834
  mkdir -p "input/gh-$ISSUE"
  gh issue view "$ISSUE" --repo "$REPO" --json number,title,body,url,state,labels \
    -q '{key:("gh-"+(.number|tostring)), number:.number, title:.title, body:.body, url:.url, state:.state, labels:[.labels[].name]}' \
    > "input/gh-$ISSUE/ticket.json"
  gh issue view "$ISSUE" --repo "$REPO" --json title,body -q '.title, .body' > input/ticket.md
  bash "$here/markers.sh" start                                                                         # FT 856-886
else                                                                                                    # FT 836-854
  mkdir -p "input/pr-$PR"
  gh pr view "$PR" --repo "$REPO" --json number,title,body,url,state,labels \
    -q '{key:("pr-"+(.number|tostring)), number:.number, title:.title, body:.body, url:.url, state:.state, labels:[.labels[].name]}' \
    > "input/pr-$PR/ticket.json"
  gh pr view "$PR" --repo "$REPO" --json title,body -q '.title, .body' > input/ticket.md
fi

# 6. Agent (FT 888-1059).
mkdir -p .dmtools
export FA_LOG_FILE="$PWD/.dmtools/fa-trace.log" RUN_OUTPUT="$PWD/.dmtools/run-output.txt" AI_AGENT_PROVIDER=fa
: > "$FA_LOG_FILE"
MACHINE_AUTHOR="$(sed -nE "s/.*machineAuthor[\"']?[[:space:]]*:[[:space:]]*[\"']([A-Za-z0-9._\[\]-]+)[\"'].*/\1/p" .dmtools/config.js | head -1)"
args=()
[[ -n "$MACHINE_AUTHOR" ]] && args+=("$(jq -nc --arg ma "$MACHINE_AUTHOR" '{params:{jobParams:{machineAuthor:$ma}}}')")
guard_bin="$RUNNER_TEMP/git-push-guard-bin"; mkdir -p "$guard_bin"
ln -sf "$FACTORY_KIT/kit/git-push-guard.sh" "$guard_bin/git"
stale="${FA_TRACE_STALE_MINUTES:-10}"
( while sleep "${WATCHDOG_INTERVAL_SECONDS:-60}"; do
    age=$(( $(date +%s) - $(stat -c %Y "$FA_LOG_FILE" 2>/dev/null || date +%s) ))
    if (( age >= stale * 60 )); then
      echo "FA-TRACE-STALE-WATCHDOG: no trace for ${age}s" >> "$RUN_OUTPUT"
      pkill -f "dmtools run" || true; pkill -f "fa --session" || true
    fi
  done ) & watchdog=$!
PATH="$guard_bin:$PATH" dmtools run "$RUNNER" "${args[@]}" --metadata "{\"contextId\":\"$ANCHOR\"}" \
  --ciRunUrl "$CI_RUN_URL" --debug 2>&1 | tee "$RUN_OUTPUT"
rc=${PIPESTATUS[0]}
kill "$watchdog" 2>/dev/null || true
outcome=success; (( rc == 0 )) || outcome=failure
log "agent exited $rc"

# 7. Post steps: each runs regardless of the others (FT always()).
post_rc=0
bash "$here/tokens.sh" || true                                                                          # FT 1061-1165
[[ -n "$ISSUE" ]] && OUTCOME="$outcome" bash "$here/markers.sh" done                                    # FT 1167-1204
if [[ "$KIND" == review && -n "$ISSUE" ]]; then                                                         # FT 1206-1420
  verdict_sh="$(ls -d "$HOME"/.dmtools/packs/pr_review-*/loop/verdict.sh 2>/dev/null | sort -V | tail -1)"
  MACHINE_AUTHOR="$MACHINE_AUTHOR" MAX_ROUNDS="${MAX_AUTO_REWORK_ROUNDS:-2}" \
    VERDICT_SH="${verdict_sh:-$FACTORY_ROOT/setup/review-verdict.sh}" bash "$here/verdict.sh" || post_rc=1
  gh issue edit "$ISSUE" --repo "$REPO" --remove-label agent:review || true                           # FT 1422-1432
fi
if [[ "$RUNNER" == *rework* ]]; then                                                                    # FT 1434-1452
  if [[ -n "$PR" ]]; then gh pr edit "$PR" --repo "$REPO" --remove-label agent:rework || true
  else gh issue edit "$ISSUE" --repo "$REPO" --remove-label agent:rework || true; fi
fi
if [[ -d .fah/memory && -n "$(git status --porcelain -- .fah/memory)" ]]; then                         # FT 1454-1480
  git add .fah/memory && git commit -qm "chore(fa): persist agent project memory [skip ci]" && git push -q || warn "memory push failed"
fi
(( rc == 0 )) || RUN_ID="$SESSION_ID" bash "$here/session.sh" quarantine                               # FT 1482-1538
if [[ "$KIND" == dev && -n "$ISSUE" ]]; then                                                            # FT 1607-1622
  gh issue edit "$ISSUE" --repo "$REPO" --remove-label agent:rework || true
  gh issue edit "$ISSUE" --repo "$REPO" --remove-label agent:review || true
  gh issue edit "$ISSUE" --repo "$REPO" --add-label agent:review || true
fi
[[ -n "$ISSUE" ]] && gh issue comment "$ISSUE" --repo "$REPO" --body "🤖 AI Teammate run ${outcome}: ${CI_RUN_URL}" || true   # FT 1624-1633
bash "$here/session.sh" persist                                                                         # FT 1571-1605 (last: leaves the orphan branch)
(( rc != 0 )) && exit "$rc"
exit "$post_rc"
```

- [ ] **Step 6: Run the leg tests**

Run: `chmod +x runner/leg/*.sh && bash tests/leg/test_app_token.sh && bash tests/leg/test_factory_leg.sh && bash tests/leg/run-all.sh`
Expected: every file reports `failed=0`; `run-all.sh` exits 0.

- [ ] **Step 7: Commit**

```bash
git add runner/leg tests/leg
git commit -m "feat(leg): App token and the full factory leg"
```

---

### Task 8: Image with pinned tools

**Files:**
- Modify: `Dockerfile`, `scripts/contract-test.sh`

**Interfaces:**
- Produces: `/app/leg_runner`, `/app/run-leg.sh`, `/app/hello.sh`, `/app/leg/*.sh`, `/opt/factory-setup` (IstiN factory-setup), `/opt/factory-kit/kit` (IstiN kit at `ca33362b06053c93e563c6d52d5a298fc2fb8d88`), `~runner/.dmtools/bin/dmtools`, `~runner/.local/bin/fa`, `/usr/local/bin/gh`.

- [ ] **Step 1: Resolve the pins** (record the printed values; they go into the Dockerfile ARGs)

```bash
# gh 2.102.0 arm64
curl -fsSL https://github.com/cli/cli/releases/download/v2.102.0/gh_2.102.0_checksums.txt | grep linux_arm64.tar.gz
# dmtools install.sh v0.1.44 (its tarball sha is e6af54ff9228d221fad92da46fd2782110aa8b3722befb0132a1deb963c8cf5e)
curl -fsSL https://github.com/epam/dmtools-dart/releases/download/v0.1.44/install.sh | sha256sum
curl -fsSL https://github.com/epam/dmtools-dart/releases/download/v0.1.44/install.sh | grep -nE 'DMTOOLS_HOME|\.dmtools|sha256|checksum' | head
# fa v1.0.535 arm64
curl -fsSL https://github.com/IstiN/flutter_agent_harness/releases/download/v1.0.535/fa-linux-arm64.tar.gz | sha256sum
# factory-setup asset of the pinned packs release
gh release view agents-rel-20261008-215056 --repo IstiN/dmtools-agents --json assets --jq '.assets[].name' | grep '^factory-setup-'
```
Then `curl -fsSL <factory-setup url> | sha256sum` for that asset name. If `install.sh` does not install under `$HOME/.dmtools/bin` or does not verify the tarball checksum, install the tarball directly instead (`curl` the `dmtools-linux-arm64.tar.gz`, `sha256sum -c` against `e6af54ff…cf5e`, extract to `/home/runner/.dmtools`) and note it in the commit message.

- [ ] **Step 2: Write the Dockerfile**

```dockerfile
# AgentCore Runtime requires linux/arm64 and port 8080.
FROM --platform=linux/arm64 dart:stable AS build
WORKDIR /src
COPY pubspec.yaml pubspec.lock ./
RUN dart pub get
COPY lib ./lib
COPY bin ./bin
RUN mkdir -p /out && dart compile exe bin/leg_runner.dart -o /out/leg_runner

FROM --platform=linux/arm64 debian:bookworm-slim
ARG GH_VERSION=2.102.0
ARG GH_SHA256=<from Step 1>
ARG DMTOOLS_VERSION=v0.1.44
ARG DMTOOLS_INSTALL_SHA256=<from Step 1>
ARG FA_VERSION=v1.0.535
ARG FA_SHA256=<from Step 1>
ARG AGENTS_TAG=agents-rel-20261008-215056
ARG FACTORY_SETUP_ASSET=<from Step 1>
ARG FACTORY_SETUP_SHA256=<from Step 1>
ARG AWF_SHA=ca33362b06053c93e563c6d52d5a298fc2fb8d88
RUN apt-get update \
 && apt-get install -y --no-install-recommends awscli jq ca-certificates curl git unzip openssl \
      procps util-linux python3 python3-venv python3-pip nodejs npm \
 && rm -rf /var/lib/apt/lists/* \
 && useradd --create-home --uid 10001 runner
RUN curl -fsSLo /tmp/gh.tgz "https://github.com/cli/cli/releases/download/v${GH_VERSION}/gh_${GH_VERSION}_linux_arm64.tar.gz" \
 && echo "${GH_SHA256}  /tmp/gh.tgz" | sha256sum -c - \
 && tar -xzf /tmp/gh.tgz -C /tmp && install -m 0755 "/tmp/gh_${GH_VERSION}_linux_arm64/bin/gh" /usr/local/bin/gh \
 && rm -rf /tmp/gh*
RUN curl -fsSLo /tmp/fs.zip "https://github.com/IstiN/dmtools-agents/releases/download/${AGENTS_TAG}/${FACTORY_SETUP_ASSET}" \
 && echo "${FACTORY_SETUP_SHA256}  /tmp/fs.zip" | sha256sum -c - \
 && mkdir -p /opt/factory-setup && unzip -q /tmp/fs.zip -d /opt/factory-setup && rm /tmp/fs.zip \
 && test -f /opt/factory-setup/setup/cache.sh \
 && git init -q /opt/factory-kit && git -C /opt/factory-kit fetch -q --depth 1 https://github.com/IstiN/dmtools-agentic-workflows.git "$AWF_SHA" \
 && git -C /opt/factory-kit checkout -q FETCH_HEAD -- kit && rm -rf /opt/factory-kit/.git \
 && head -1 /opt/factory-kit/kit/git-push-guard.sh | grep -q bash
USER runner
RUN curl -fsSLo /tmp/dmtools-install.sh "https://github.com/epam/dmtools-dart/releases/download/${DMTOOLS_VERSION}/install.sh" \
 && echo "${DMTOOLS_INSTALL_SHA256}  /tmp/dmtools-install.sh" | sha256sum -c - \
 && bash /tmp/dmtools-install.sh "${DMTOOLS_VERSION}" && rm /tmp/dmtools-install.sh \
 && curl -fsSLo /tmp/fa.tgz "https://github.com/IstiN/flutter_agent_harness/releases/download/${FA_VERSION}/fa-linux-arm64.tar.gz" \
 && echo "${FA_SHA256}  /tmp/fa.tgz" | sha256sum -c - \
 && mkdir -p /tmp/fa ~/.local/bin ~/.local/lib && tar -xzf /tmp/fa.tgz -C /tmp/fa \
 && cp /tmp/fa/bundle/bin/fa /tmp/fa/bundle/version.txt ~/.local/bin/ && cp -a /tmp/fa/bundle/lib/. ~/.local/lib/ \
 && rm -rf /tmp/fa /tmp/fa.tgz
ENV PATH=/home/runner/.dmtools/bin:/home/runner/.local/bin:$PATH
COPY --from=build /out/leg_runner /app/leg_runner
COPY --chmod=0755 runner/run-leg.sh runner/hello.sh /app/
COPY --chmod=0755 runner/leg /app/leg
EXPOSE 8080
CMD ["/app/leg_runner"]
```

Replace every `<from Step 1>` with the value recorded in Step 1 before building (the committed Dockerfile has no `<…>`).

- [ ] **Step 3: Extend the contract test** — in `scripts/contract-test.sh` replace the tool check line with:

```bash
docker run --rm --entrypoint sh "$image" -c \
  'command -v bash setsid aws jq git gh curl openssl node python3 pkill flock dmtools fa && test -f /opt/factory-setup/setup/review-verdict.sh && test -x /app/leg/factory-leg.sh && test -f /opt/factory-kit/kit/git-push-guard.sh' \
  >/dev/null || fail "image lacks a leg tool or IstiN's factory files"
docker run --rm --entrypoint sh "$image" -c 'dmtools --version && fa --version' >/dev/null \
  || fail "dmtools or fa does not start"
```

- [ ] **Step 4: Build and run the contract test** (native arm64 on the owner's Pi)

Run: `sg docker -c 'docker build -t fa-ac-leg-runner:local . && scripts/contract-test.sh fa-ac-leg-runner:local'`
Expected: `contract test: PASS`; image size printed by `docker image inspect` < 2 GB.

- [ ] **Step 5: Commit**

```bash
git add Dockerfile scripts/contract-test.sh
git commit -m "build(image): pin dmtools, fa, gh and IstiN's factory files"
```

---

### Task 9: Terraform — fence, locks, secrets, roles, leg state machine

**Files:**
- Modify: `terraform/bootstrap/fence.tf`, `terraform/bootstrap/tests/bootstrap.tftest.hcl`, `terraform/foundation/main.tf`, `terraform/foundation/iam.tf`, `terraform/foundation/tests/foundation.tftest.hcl`, `terraform/runtime/main.tf`, `terraform/runtime/tests/runtime.tftest.hcl`, `.github/workflows/deploy.yml`
- Create: `terraform/foundation/locks.tf`, `terraform/foundation/secrets.tf`, `terraform/runtime/leg_state_machine.tf`

**Interfaces:**
- Produces: DynamoDB table `fa-ac-locks` (hash key `pk` S, TTL `expiresAt`); secrets `fa-ac-github-app`, `fa-ac-llm-keys` (values set by CI); state machine `fa-ac-leg` taking input `{repo, issue|null, pr|null, leg, reason, ref, sender}`; foundation outputs `locks_table`, `invoker_role_arn`.
- The invoker role is reused as the state machine role and trusts only `states.amazonaws.com` from this account's `fa-ac-*` state machines. `var.invoker_subjects` and the `INVOKER_SUBJECTS_JSON` secret are removed.

- [ ] **Step 1: Fence** — in `terraform/bootstrap/fence.tf`:
  - In `OwnNamedResourcesInUsEast1`, add `"dynamodb:*"` to `Action` and `"arn:aws:dynamodb:us-east-1:${local.account}:table/fa-ac-*"` to `Resource`.
  - Add a statement after `EcrAuthToken`:

```hcl
      {
        Sid       = "StepFunctionsTaskCallbacks"
        Effect    = "Allow"
        Action    = ["states:SendTaskSuccess", "states:SendTaskFailure", "states:SendTaskHeartbeat"]
        Resource  = "*"
        Condition = { StringEquals = { "aws:RequestedRegion" = "us-east-1" } }
      },
```

  Add to `tests/bootstrap.tftest.hcl`:

```hcl
run "fence_allows_locks_and_task_callbacks" {
  command = plan
  assert {
    condition     = contains(one([for s in jsondecode(aws_iam_policy.boundary.policy).Statement : s.Resource if s.Sid == "OwnNamedResourcesInUsEast1"]), "arn:aws:dynamodb:us-east-1:111111111111:table/fa-ac-*")
    error_message = "The fence must allow fa-ac DynamoDB tables."
  }
  assert {
    condition     = one([for s in jsondecode(aws_iam_policy.boundary.policy).Statement : s.Effect if s.Sid == "StepFunctionsTaskCallbacks"]) == "Allow"
    error_message = "Leg sessions must be able to answer their task token."
  }
}
```

  Run `cd terraform/bootstrap && terraform test`. If `fence_fits_the_managed_policy_size_limit` fails, merge `StepFunctionsTaskCallbacks` into `EcrAuthToken` (same `Resource = "*"` and region condition) and re-run until 6144 holds.

- [ ] **Step 2: Locks and secrets** — `terraform/foundation/locks.tf`:

```hcl
# Repo and item locks for the AWS orchestrator (spec 2026-10-09 §5.3, §5.6).
resource "aws_dynamodb_table" "locks" {
  name         = "fa-ac-locks"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "pk"
  attribute {
    name = "pk"
    type = "S"
  }
  ttl {
    attribute_name = "expiresAt"
    enabled        = true
  }
  point_in_time_recovery { enabled = false }
}

output "locks_table" { value = aws_dynamodb_table.locks.name }
```

`terraform/foundation/secrets.tf`:

```hcl
# Containers only: CI writes the values from GitHub environment secrets
# (deploy.yml "Sync secrets"); no secret value lives in Terraform state.
resource "aws_secretsmanager_secret" "github_app" {
  name                    = "fa-ac-github-app"
  recovery_window_in_days = 7
}

resource "aws_secretsmanager_secret" "llm_keys" {
  name                    = "fa-ac-llm-keys"
  recovery_window_in_days = 7
}

locals {
  secret_arns = [aws_secretsmanager_secret.github_app.arn, aws_secretsmanager_secret.llm_keys.arn]
}
```

- [ ] **Step 3: Roles** — in `terraform/foundation/iam.tf`:
  - Append to the execution role policy statements:

```hcl
    { Effect = "Allow", Action = ["secretsmanager:GetSecretValue"], Resource = local.secret_arns },
    { Effect = "Allow", Action = ["states:SendTaskSuccess", "states:SendTaskFailure", "states:SendTaskHeartbeat"], Resource = "*" },
```

  - Replace the invoker role's comment, trust and policy:

```hcl
# Assumed by the fa-ac-leg state machine to lock items and invoke legs.
resource "aws_iam_role" "invoker" {
  name                 = "fa-ac-leg-invoker"
  path                 = "/fa-ac/"
  max_session_duration = 3600
  permissions_boundary = local.boundary_arn
  assume_role_policy = jsonencode({ Version = "2012-10-17", Statement = [{
    Effect    = "Allow"
    Principal = { Service = "states.amazonaws.com" }
    Action    = "sts:AssumeRole"
    Condition = {
      StringEquals = { "aws:SourceAccount" = local.account }
      ArnLike      = { "aws:SourceArn" = "arn:aws:states:${local.region}:${local.account}:stateMachine:fa-ac-*" }
    }
  }] })
}

resource "aws_iam_role_policy" "invoker" {
  name = "fa-ac-leg-invoker"
  role = aws_iam_role.invoker.id
  policy = jsonencode({ Version = "2012-10-17", Statement = [
    { Effect = "Allow", Action = ["bedrock-agentcore:InvokeAgentRuntime"], Resource = [local.runtime_arns] },
    { Effect = "Allow", Action = ["dynamodb:PutItem", "dynamodb:DeleteItem", "dynamodb:GetItem"], Resource = [aws_dynamodb_table.locks.arn] },
    { Effect = "Allow", Action = ["logs:CreateLogDelivery", "logs:GetLogDelivery", "logs:UpdateLogDelivery", "logs:DeleteLogDelivery", "logs:ListLogDeliveries", "logs:PutResourcePolicy", "logs:DescribeResourcePolicies", "logs:DescribeLogGroups"], Resource = "*" },
  ] })
}
```

  - In `main.tf` delete `variable "invoker_subjects"` and the `data "aws_iam_openid_connect_provider" "github"` block if nothing else uses it (check with `grep -rn openid_connect terraform/foundation`).
  - In `tests/foundation.tftest.hcl` delete the `invoker_subjects` variable line and the runs `invoker_trusts_only_listed_agentcore_environments`, `wildcard_invoker_subject_is_rejected`, `pull_request_subject_is_rejected`, `foreign_owner_subject_is_rejected`, `immutable_id_subject_is_accepted`, `execution_role_has_no_secret_access_in_m0`; update `invoker_can_only_invoke_our_runtime_and_read_results` to the new statements; add:

```hcl
run "invoker_is_the_leg_state_machine" {
  command = plan
  assert {
    condition     = jsondecode(aws_iam_role.invoker.assume_role_policy).Statement[0].Principal.Service == "states.amazonaws.com"
    error_message = "Only Step Functions may assume the invoker role."
  }
  assert {
    condition     = jsondecode(aws_iam_role.invoker.assume_role_policy).Statement[0].Condition.ArnLike["aws:SourceArn"] == "arn:aws:states:us-east-1:111111111111:stateMachine:fa-ac-*"
    error_message = "The invoker trust is pinned to fa-ac state machines."
  }
}

run "legs_read_only_their_two_secrets" {
  command = plan
  assert {
    condition     = length([for s in jsondecode(aws_iam_role_policy.execution.policy).Statement : s if contains(s.Action, "secretsmanager:GetSecretValue")]) == 1
    error_message = "Exactly one statement grants secret reads."
  }
}

run "locks_expire" {
  command = plan
  assert {
    condition     = aws_dynamodb_table.locks.name == "fa-ac-locks" && one(aws_dynamodb_table.locks.ttl).attribute_name == "expiresAt"
    error_message = "Locks must carry a TTL."
  }
}
```

  Run `cd terraform/foundation && terraform fmt && terraform init -backend=false -input=false && terraform test`. Expected: all pass. If the mock provider leaves a computed ARN unknown in a policy assertion, add an `override_resource` block with `override_during = plan` and a fixed `arn` (pattern already used for `aws_sns_topic.budget`).

- [ ] **Step 4: Leg state machine** — `terraform/runtime/leg_state_machine.tf`:

```hcl
# One factory leg (spec 2026-10-09 §5.6): lock the item, invoke the runtime
# with a task token, release the lock. M1 cap: 1 hour.
resource "aws_cloudwatch_log_group" "leg" {
  name              = "/aws/vendedlogs/states/fa-ac-leg"
  retention_in_days = 30
}

locals {
  runtime_arn = aws_bedrockagentcore_agent_runtime.leg_runner.agent_runtime_arn
  anchor_expr = "($states.input.issue ? 'gh-' & $string($states.input.issue) : 'pr-' & $string($states.input.pr))"
}

resource "aws_sfn_state_machine" "leg" {
  name     = "fa-ac-leg"
  type     = "STANDARD"
  role_arn = data.terraform_remote_state.foundation.outputs.invoker_role_arn
  logging_configuration {
    log_destination        = "${aws_cloudwatch_log_group.leg.arn}:*"
    include_execution_data = false
    level                  = "ERROR"
  }
  definition = jsonencode({
    Comment       = "fa-ac leg: lock, invoke with task token, unlock"
    QueryLanguage = "JSONata"
    StartAt       = "Prepare"
    States = {
      Prepare = {
        Type = "Pass"
        Assign = {
          sessionId = "{% $uuid() %}"
          lockKey   = "{% 'item#' & $states.input.repo & '#' & ${local.anchor_expr} %}"
          spec      = "{% $states.input %}"
        }
        Next = "AcquireLock"
      }
      AcquireLock = {
        Type     = "Task"
        Resource = "arn:aws:states:::dynamodb:putItem"
        Arguments = {
          TableName = data.terraform_remote_state.foundation.outputs.locks_table
          Item = {
            pk        = { S = "{% $lockKey %}" }
            owner     = { S = "{% $states.context.Execution.Id %}" }
            expiresAt = { N = "{% $string($floor($millis() / 1000) + 7200) %}" }
          }
          ConditionExpression       = "attribute_not_exists(pk) OR expiresAt < :now"
          ExpressionAttributeValues = { ":now" = { N = "{% $string($floor($millis() / 1000)) %}" } }
        }
        Catch = [{ ErrorEquals = ["DynamoDB.ConditionalCheckFailedException"], Next = "Busy" }]
        Next  = "Invoke"
      }
      Invoke = {
        Type             = "Task"
        Resource         = "arn:aws:states:::aws-sdk:bedrockagentcore:invokeAgentRuntime.waitForTaskToken"
        HeartbeatSeconds = 900
        TimeoutSeconds   = 3600
        Arguments = {
          AgentRuntimeArn  = local.runtime_arn
          RuntimeSessionId = "{% $sessionId %}"
          ContentType      = "application/json"
          Payload          = "{% $string({'sessionId': $sessionId, 'leg': ($spec.leg = '' or $not($exists($spec.leg))) ? 'auto' : $spec.leg, 'maxSeconds': 3300, 'callback': {'type': 's3'}, 'taskToken': $states.context.Task.Token, 'repo': $spec.repo, 'issue': $spec.issue, 'pr': $spec.pr, 'reason': $spec.reason, 'ref': $spec.ref, 'sender': $spec.sender, 'runUrl': 'https://us-east-1.console.aws.amazon.com/states/home?region=us-east-1#/v2/executions/details/' & $states.context.Execution.Id}) %}"
        }
        Retry  = [{ ErrorEquals = ["infra_error"], MaxAttempts = 1, IntervalSeconds = 30 }]
        Assign = { result = "{% $states.result %}" }
        Catch  = [{ ErrorEquals = ["States.ALL"], Assign = { failure = "{% $states.errorOutput %}" }, Next = "ReleaseLockAfterFailure" }]
        Next   = "ReleaseLock"
      }
      ReleaseLock = {
        Type     = "Task"
        Resource = "arn:aws:states:::dynamodb:deleteItem"
        Arguments = {
          TableName                 = data.terraform_remote_state.foundation.outputs.locks_table
          Key                       = { pk = { S = "{% $lockKey %}" } }
          ConditionExpression       = "#o = :me"
          ExpressionAttributeNames  = { "#o" = "owner" }
          ExpressionAttributeValues = { ":me" = { S = "{% $states.context.Execution.Id %}" } }
        }
        Catch  = [{ ErrorEquals = ["States.ALL"], Next = "Done" }]
        Output = "{% $result %}"
        Next   = "Done"
      }
      ReleaseLockAfterFailure = {
        Type     = "Task"
        Resource = "arn:aws:states:::dynamodb:deleteItem"
        Arguments = {
          TableName                 = data.terraform_remote_state.foundation.outputs.locks_table
          Key                       = { pk = { S = "{% $lockKey %}" } }
          ConditionExpression       = "#o = :me"
          ExpressionAttributeNames  = { "#o" = "owner" }
          ExpressionAttributeValues = { ":me" = { S = "{% $states.context.Execution.Id %}" } }
        }
        Catch = [{ ErrorEquals = ["States.ALL"], Next = "Failed" }]
        Next  = "Failed"
      }
      Busy   = { Type = "Succeed", Output = { status = "busy" } }
      Done   = { Type = "Succeed" }
      Failed = { Type = "Fail", Error = "{% $failure.Error %}", Cause = "{% $failure.Cause %}" }
    }
  })
}

output "leg_state_machine_arn" {
  value     = aws_sfn_state_machine.leg.arn
  sensitive = true
}
```

  In `terraform/runtime/main.tf` add to `environment_variables`:

```hcl
    FA_AC_GITHUB_APP_SECRET = "fa-ac-github-app"
    FA_AC_LLM_SECRET        = "fa-ac-llm-keys"
    DMTOOLS_PACK_REGISTRY   = "https://github.com/IstiN/dmtools-agents/releases/download/agents-rel-20261008-215056"
    MAX_AUTO_REWORK_ROUNDS  = "2"
```

  Add to `tests/runtime.tftest.hcl` (follow the file's existing mock/override pattern for `terraform_remote_state`; add `locks_table = "fa-ac-locks"` and `invoker_role_arn = "arn:aws:iam::111111111111:role/fa-ac/fa-ac-leg-invoker"` to its mocked outputs):

```hcl
run "leg_state_machine_contract" {
  command = plan
  assert {
    condition     = jsondecode(aws_sfn_state_machine.leg.definition).States.Invoke.TimeoutSeconds == 3600 && jsondecode(aws_sfn_state_machine.leg.definition).States.Invoke.HeartbeatSeconds == 900
    error_message = "M1 caps legs at 1 hour with a 15-minute heartbeat."
  }
  assert {
    condition     = jsondecode(aws_sfn_state_machine.leg.definition).States.Invoke.Retry[0].ErrorEquals == ["infra_error"]
    error_message = "Only infra errors are retried; the rules handle agent failures."
  }
  assert {
    condition     = jsondecode(aws_sfn_state_machine.leg.definition).States.AcquireLock.Catch[0].Next == "Busy"
    error_message = "A held item lock ends the execution as busy."
  }
  assert {
    condition     = jsondecode(aws_sfn_state_machine.leg.definition).States.Invoke.Catch[0].Next == "ReleaseLockAfterFailure"
    error_message = "Failures always release the lock."
  }
}
```

  Run each root: `terraform fmt -recursive && terraform init -backend=false -input=false && terraform validate && terraform test`. Expected: all pass.

- [ ] **Step 5: Validate the definition against AWS (read-only)**

```bash
cd terraform/runtime && terraform console <<<'aws_sfn_state_machine.leg.definition' >/dev/null  # syntax only
AWS_REGION=us-east-1 aws stepfunctions validate-state-machine-definition --type STANDARD \
  --definition "$(cd terraform/runtime && terraform console <<<'jsonencode(jsondecode(aws_sfn_state_machine.leg.definition))' 2>/dev/null | jq -r . )" \
  --query '{result:result,diagnostics:diagnostics[].message}'
```
Expected: `"result": "OK"` (if `terraform console` cannot render the definition without a backend, copy the rendered JSON from `terraform plan -out` / `terraform show -json` of a mocked test run instead).

- [ ] **Step 6: Deploy workflow** — in `.github/workflows/deploy.yml`:
  - Delete `TF_VAR_invoker_subjects: ${{ secrets.INVOKER_SUBJECTS_JSON }}` (and its `env:` key if empty).
  - Add after "Apply foundation" in the `foundation` job:

```yaml
      - name: Sync secrets
        env:
          GITHUB_APP_JSON: ${{ secrets.FA_AC_GITHUB_APP_JSON }}
          LLM_KEYS_JSON: ${{ secrets.FA_AC_LLM_KEYS_JSON }}
        run: |
          sync() {
            local id="$1" value="$2"
            [[ -n "$value" ]] || { echo "$id: GitHub secret not set yet, skipping"; return 0; }
            jq -e 'type == "object"' >/dev/null <<<"$value" || { echo "$id: value is not a JSON object" >&2; return 1; }
            current="$(aws secretsmanager get-secret-value --secret-id "$id" --query SecretString --output text 2>/dev/null || true)"
            if [[ "$current" == "$value" ]]; then echo "$id: unchanged"; return 0; fi
            aws secretsmanager put-secret-value --secret-id "$id" --secret-string "$value" >/dev/null
            echo "$id: updated"
          }
          sync fa-ac-github-app "$GITHUB_APP_JSON"
          sync fa-ac-llm-keys "$LLM_KEYS_JSON"
```

  - Delete the `INVOKER_SUBJECTS_JSON` environment secret after merge: `gh secret delete INVOKER_SUBJECTS_JSON --repo rustembuild/dark-factory-aws --env aws-deploy`.

- [ ] **Step 7: Commit, PR, deploy**

```bash
git add terraform .github/workflows/deploy.yml
git commit -m "feat(infra): leg state machine, locks, secrets and fence for M1"
git push -u origin feat/m1-leg-path
gh pr create --repo rustembuild/dark-factory-aws --base main --title "feat: M1 AWS leg path" --body-file docs/m1-pr.md
```

Write `docs/m1-pr.md` from the PR template (What / Why / How it was verified with the test counts / Risk and rollback: "Merging deploys bootstrap (fence), foundation (locks, secrets, roles), image and runtime (state machine). Rollback: revert the PR; the deploy re-applies the previous state."), and delete it before the final commit (it is PR text, not repo content).
The owner merges; then watch: `gh run watch --repo rustembuild/dark-factory-aws $(gh run list --repo rustembuild/dark-factory-aws --workflow deploy --limit 1 --json databaseId --jq '.[0].databaseId') --exit-status`. Expected: every job succeeds; "Sync secrets" prints "GitHub secret not set yet, skipping" until Task 10.

---

### Task 10: GitHub App, canary repo and the first manual legs (owner + agent)

**Files:**
- Create in new repo `rustembuild/fa-canary-snake`: `README.md`, `.dmtools/config.js`, `.dmtools/runners/{fa-bug-dev,fa-story-dev,fa-review,fa-rework}.json`
- Create in dark-factory-aws: `docs/m1-findings.md`

**Interfaces:**
- Consumes: deployed `fa-ac-leg`, secrets containers, Task 7 secret shapes.

- [ ] **Step 1: Owner creates the GitHub App** (GitHub UI, organization `rustembuild` → Settings → Developer settings → GitHub Apps → New)
  - Name `rustembuild-factory`; homepage `https://github.com/rustembuild`; webhook **inactive** (M2 adds it).
  - Repository permissions: Contents read/write, Issues read/write, Pull requests read/write, Checks read/write, Actions read/write, Variables read, Metadata read.
  - "Only on this account". Create, generate a private key (downloads a `.pem`), note the App ID.

- [ ] **Step 2: Agent creates the canary repo**

```bash
gh repo create rustembuild/fa-canary-snake --private --description "fa-agentcore canary: the factory builds a web snake game" --add-readme
gh api -X POST repos/rustembuild/fa-canary-snake/rulesets --input ~/projects/dark-factory-aws/github/fork-main-ruleset.json --jq .name
gh variable set AGENTCORE_ALLOWED_ACTORS --repo rustembuild/fa-canary-snake --body agzyamov
for l in "agent:dev:0E8A16" "agent:review:1D76DB" "agent:rework:D93F0B" "needs-human:B60205" "blocked:000000" "pr_approved:0E8A16"; do
  gh label create "${l%:*}" --repo rustembuild/fa-canary-snake --color "${l##*:}" --force
done
```
Then clone it, add `.dmtools/config.js`:

```js
// AWS-mode canary (spec 2026-10-09 §3).
module.exports = {
  machineAuthor: 'rustembuild-factory[bot]',
  orchestrator: 'aws',
  sm: { runners: {
    bug: '.dmtools/runners/fa-bug-dev.json',
    story: '.dmtools/runners/fa-story-dev.json',
    review: '.dmtools/runners/fa-review.json',
    rework: '.dmtools/runners/fa-rework.json'
  } }
};
```
and copy the four runner JSONs verbatim from the fa fork (`~/projects/flutter_agent_harness_agentcore/.dmtools/runners/`), changing only `customParams.targetRepository` to `rustembuild/fa-canary-snake` where it names a repository. Commit through a PR in the canary repo (`main` is PR-only); the owner merges.

- [ ] **Step 3: Owner installs the App and fills the secrets**
  - Install `rustembuild-factory` on `rustembuild/fa-canary-snake` only; note the installation ID (from the installation URL).
  - In `rustembuild/dark-factory-aws` → Settings → Environments → `aws-deploy`, add secrets (values typed by the owner, never pasted into chat):
    - `FA_AC_GITHUB_APP_JSON` = `{"app_id":"<id>","installation_id":"<id>","private_key":"<PEM with \n escapes>"}` (create with `jq -n --arg a <id> --arg i <id> --rawfile k app.pem '{app_id:$a,installation_id:$i,private_key:$k}' | gh secret set FA_AC_GITHUB_APP_JSON --repo rustembuild/dark-factory-aws --env aws-deploy`, then delete the `.pem`).
    - `FA_AC_LLM_KEYS_JSON` = the provider keys the runner JSONs need (`ZAI_CODE_KEY`, `KIMI_REVIEW_KEY`), e.g. via a script that reads them with `read -s`.
  - Re-run the deploy (`gh workflow run deploy.yml --repo rustembuild/dark-factory-aws --ref main`). Expected: "Sync secrets" prints `updated` for both.

- [ ] **Step 4: Verify the payload contract on a hello leg through the state machine**

```bash
export AWS_REGION=us-east-1
SM=$(aws stepfunctions list-state-machines --query "stateMachines[?name=='fa-ac-leg'].stateMachineArn|[0]" --output text)
gh issue create --repo rustembuild/fa-canary-snake --title "Canary: say hello" --body "M1 check." --label blocked
```
Start an execution against that issue: `aws stepfunctions start-execution --state-machine-arn "$SM" --name "m1-check-$(date +%s)" --input '{"repo":"rustembuild/fa-canary-snake","issue":1,"pr":null,"leg":"dev","reason":"M1 check","ref":"","sender":"agzyamov"}'`, then `aws stepfunctions describe-execution --execution-arn <arn> --query '{status:status,output:output,error:error,cause:cause}'` until it is not RUNNING.
Expected: `SUCCEEDED`, output with `"status":"ok"`; the leg's guard skipped because of `blocked` (CloudWatch log group `/aws/bedrock-agentcore/runtimes/fa_ac_leg_runner-*` shows `skip: blocked`). This proves invoke → payload → callback → unlock. If the execution fails with a payload/serialization error from `invokeAgentRuntime`, change `Payload` in Task 9 to pass the object without `$string(...)` and redeploy.

- [ ] **Step 5: First real dev leg**

```bash
gh issue create --repo rustembuild/fa-canary-snake --title "Build a playable web snake game" \
  --body "Python FastAPI backend serving a React (Vite) frontend. Arrow keys move the snake, food grows it, walls and self-collision end the game, score shown. Include tests (pytest for the API, vitest for the game logic) and a README with run instructions." \
  --label agent:dev
```
Start `fa-ac-leg` with `{"repo":"rustembuild/fa-canary-snake","issue":<n>,"pr":null,"leg":"dev","reason":"M1 first dev leg","ref":"","sender":"agzyamov"}`.
Expected within the hour: a PR `ai/gh-<n>` with `Closes #<n>`, a "Machine jobs" marker flipped to ✅ on the issue, labels `agent:review` added, a run-link comment, and a `fa-sess/gh-<n>` branch. Then start a `review` leg (`"leg":"review"`) and check the verdict labels (`pr_approved` or `agent:rework` + `rework-round-1`).

- [ ] **Step 6: Write `docs/m1-findings.md`** (dark-factory-aws; measured values only)

```markdown
# M1 findings (<date>)

| Measurement | Value |
|---|---|
| Hello-through-state-machine (start → SUCCEEDED) | <s> |
| Dev leg wall clock / outcome | <min> / <PR link> |
| Review leg wall clock / verdict | <min> / <labels> |
| Session cost estimate (vCPU-h + GB-h) | <$> |
| Issues found | <list or "none"> |
```

Commit via PR (`docs: M1 findings`).

---

## Self-review notes (planner)

- Spec coverage for this plan's scope (M0 finish, M1): §5.6 leg state machine (Task 9), §5.7 leg runner (Tasks 3-8), §5.1 App permissions and tokens without webhook (Tasks 7, 10), §5.8 fence/roles/locks/secrets (Task 9), §7 allowlist and short-lived tokens (Tasks 4, 7), §8 tests (each task), §9 M0/M1 rows (Tasks 1, 10). Deferred to the M2 plan: webhook, tick, engine adapter, the `orchestrator` switch in the stubs, comment filtering, the 4-hour cap with token refresh, the failure comment from the state machine (§5.6 step 5; the leg runner already comments on its own failures).
- Deviations from the spec, with reason: leg cap 1 h not 4 h (App tokens expire after 1 h; budget data lag); App creation moved into M1 because a leg cannot push without it; the item-lock TTL is 2 h to match the 1-hour cap.
