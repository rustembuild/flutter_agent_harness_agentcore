# M0 — AgentCore Foundation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Deploy a hello-world leg runner to Amazon Bedrock AgentCore Runtime in the shared AWS account — fenced away from df-agentcore, budget-capped, deployed by Terraform through GitHub Actions — and prove it end to end from a GitHub workflow in the fa fork.

**Architecture:** A new repo `dark-factory-aws` holds a small Dart HTTP shim (same toolchain as fa) (the AgentCore `/ping` + `/invocations` contract) that runs `run-leg.sh` in the background and writes a `LegResult` to S3, plus three Terraform roots: `bootstrap` (state bucket + fenced CI role, applied once locally by the owner), `foundation` (ECR, results bucket, runtime/invoker roles, budget + hard stop) and `runtime` (the AgentCore runtime pointing at an image digest). The fa fork gets one smoke workflow that assumes the invoker role via GitHub OIDC, invokes a hello leg, and waits for its result.

**Tech Stack:** Dart ≥ 3.12 (`dart:io` HttpServer, AOT `dart compile exe`, `package:test`, `package:lints`), aws CLI for S3 (Dart has no official AWS SDK), Docker (linux/arm64), Terraform 1.16 with `hashicorp/aws` 6.65.0 and `terraform test` mock providers, GitHub Actions with OIDC, AWS CLI v2.

**Spec:** `docs/superpowers/specs/2026-10-08-agentcore-executor-design.md` (this repo, branch `docs/agentcore-executor-spec`). This plan covers milestone **M0** only; M1–M5 get their own plans after M0's findings.

## Global Constraints

- AWS region for every regional resource: `us-east-1`. Never create or change anything in `eu-central-1`.
- Every resource name starts with `fa-ac-` (`fa_ac_` where AWS forbids hyphens); every resource is tagged `Project=fa-agentcore` via provider `default_tags`; every IAM role/policy lives under path `/fa-ac/`; every IAM role carries the `fa-ac-boundary` permissions boundary.
- Never create, modify, read secrets of, or delete anything belonging to df-agentcore: tag `Project=df-agentcore`, names `darkfactory-*`, `df-agentcore-*`, role `GitHubActionsRole`, the GitHub OIDC provider (read-only via `data` source).
- Terraform state only in `fa-ac-state-<account>` (us-east-1, `use_lockfile=true`). Never df-agentcore's state bucket.
- Terraform is applied only by GitHub Actions — except `terraform/bootstrap`, which the owner applies locally exactly once.
- The AWS account ID, budget email, role/runtime ARNs and bucket names never appear in a public repo or public log: pass them as GitHub **secrets** (masked), or derive them at run time.
- Budget: alert at **$20/month**, hard stop (deny-all on this project's roles) at **$50/month**, filtered to Region `us-east-1`.
- AgentCore contract: ARM64 image, `0.0.0.0:8080`, `GET /ping` → `{"status": "Healthy"|"HealthyBusy", "time_of_last_update": <unix seconds>}`, `POST /invocations`. Runtime session id ≥ 33 characters. Image ≤ 2 GB.
- Dart SDK `>=3.12.0 <4.0.0`, `test: ^1.31.2`, `lints: ^6.1.0` — the same floors as fa's `pubspec.yaml`; `dart analyze --fatal-infos` must be clean.
- `hashicorp/aws` provider pinned to `6.65.0` (same as df-agentcore, proven with `aws_bedrockagentcore_agent_runtime`); Terraform `>= 1.10.0`.
- GitHub Actions: no `pull_request_target`, no `issue_comment` triggers; third-party actions pinned by commit SHA.
- Commit identity in new repos: whatever `git config user.*` the owner has; end commit messages with `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`.

## Review Focus

1. **A leg spawns background children and then times out** — the whole process group must die; no orphan keeps burning a paid session. (Test added in Task 3.)
2. **A malformed `Content-Length` header** on `/invocations` — expect a 400 or a closed connection, never a started leg, a crashed server, or a stuck `HealthyBusy`. (Test added in Task 4.)
3. **AWS budget data lags by hours, so the $50 hard stop is not instant** — the per-session `max_lifetime` (≤ 3600 s in M0) and the smoke workflow's single-run concurrency are the real-time caps; expect them asserted, not assumed. (Test added in Task 8; concurrency in Task 10.)
4. **The fenced CI role can't step outside the fence** — creating a role without the boundary, touching `df-agentcore-*` buckets, or acting in eu-central-1 must be explicitly denied by IAM itself, not just by our Terraform. (IAM policy simulator checks added in Task 6.)
5. **df-agentcore is byte-for-byte unchanged after M0** — expect a before/after snapshot diff of its tagged resources, CI role, and OIDC provider to be empty. (Snapshot in Task 0, diff in Task 10.)

---

## File Structure

New repo `~/projects/dark-factory-aws` (GitHub: `rustembuild/dark-factory-aws`, **private**). The leg runner is a Dart package at the repo root, the same toolchain as fa:

```
dark-factory-aws/
├── pubspec.yaml / pubspec.lock     # leg_runner package (no runtime deps; test + lints for dev)
├── analysis_options.yaml
├── lib/
│   ├── leg_runner.dart             # exports
│   └── src/
│       ├── busy.dart               # BusyTracker: Healthy/HealthyBusy + one leg per session
│       ├── spec.dart               # parseLegSpec: validate the LegSpec payload
│       ├── results.dart            # resultKey, FileResultWriter, AwsCliResultWriter
│       ├── job.dart                # runLeg: run the leg command, deliver LegResult
│       ├── server.dart             # serve: /ping + /invocations
│       └── config.dart             # buildWriter from the environment
├── bin/leg_runner.dart             # container entrypoint (compiled AOT)
├── test/                           # package:test, one file per lib/src module
├── runner/run-leg.sh               # M0 hello leg (replaced by the real leg in M2)
├── Dockerfile                      # dart:stable build → debian slim + aws CLI
├── .dockerignore
├── scripts/contract-test.sh        # runs the image locally, checks the contract
├── terraform/
│   ├── bootstrap/                  # state bucket + fa-ac-ci role + fa-ac-boundary (local, once)
│   ├── foundation/                 # ECR, results bucket, runtime/invoker roles, budget
│   └── runtime/                    # aws_bedrockagentcore_agent_runtime
├── docs/m0-findings.md             # measured results (Task 10)
└── .github/workflows/
    ├── test.yml                    # dart analyze/test + contract test + terraform fmt/validate/test
    ├── deploy.yml                  # foundation → image → runtime
    └── whoami.yml                  # prints this repo's OIDC subject
```

This repo (`rustembuild/flutter_agent_harness_agentcore`):

```
.github/workflows/agentcore-smoke.yml   # assume invoker role, invoke hello leg, wait for LegResult
```

---

### Task 0: Preflight and prerequisites (owner + read-only checks)

**Files:** none in any repo. Outputs go to `~/.cache/fa-ac/` (never committed).

**Interfaces:**
- Produces: `~/.cache/fa-ac/df-snapshot-before.json` (consumed by Task 10), the owner's AWS CLI profile name (used as `AWS_PROFILE` in Task 6), confirmation that us-east-1 spend is ~0.

- [ ] **Step 1: Install the Dart SDK (ARM64) locally**

Dart is missing on the owner's machine (Docker, AWS CLI, Terraform 1.16.3 and gh are present).

```bash
curl -fsSLo /tmp/dartsdk.zip https://storage.googleapis.com/dart-archive/channels/stable/release/latest/sdk/dartsdk-linux-arm64-release.zip
rm -rf ~/.local/dart-sdk && unzip -q /tmp/dartsdk.zip -d ~/.local && rm /tmp/dartsdk.zip
echo 'export PATH="$HOME/.local/dart-sdk/bin:$PATH"' >> ~/.bashrc
export PATH="$HOME/.local/dart-sdk/bin:$PATH"
dart --version
```
Expected: `Dart SDK version: 3.x.y (stable)` with x ≥ 12 (fa's floor).

- [ ] **Step 2: Confirm GitHub access for both repos**

`gh` is logged in as `agzyamov`; the repos live under `rustembuild`.

```bash
gh api repos/rustembuild/flutter_agent_harness_agentcore --jq '.permissions.admin'
gh api users/rustembuild --jq '.type'
```
Expected: `true` (admin is needed to create environments/secrets), and `Organization` or `User`. If admin is `false`, STOP and ask the owner to grant `agzyamov` admin or to log in as `rustembuild` (`gh auth login`).

- [ ] **Step 3: Confirm AWS identity (owner's admin credentials)**

```bash
export AWS_PROFILE=<owner's profile for the shared account>
export AWS_REGION=us-east-1
aws sts get-caller-identity --query '{Account:Account,Arn:Arn}'
```
Expected: the shared account. Keep the account ID out of any repo.

- [ ] **Step 4: Read-only inventory — must find no collisions**

```bash
ACC=$(aws sts get-caller-identity --query Account --output text)
aws iam list-open-id-connect-providers --query 'OpenIDConnectProviderList[].Arn'
aws s3api head-bucket --bucket "fa-ac-state-$ACC" 2>&1 | tail -1
aws iam list-roles --path-prefix /fa-ac/ --query 'Roles[].RoleName'
aws budgets describe-budgets --account-id "$ACC" --query 'Budgets[].BudgetName' 2>/dev/null
aws bedrock-agentcore-control list-agent-runtimes --region us-east-1 --query 'agentRuntimes[].agentRuntimeName'
aws resourcegroupstaggingapi get-resources --region us-east-1 --query 'ResourceTagMappingList[].ResourceARN'
```
Expected: exactly one OIDC provider ending `oidc-provider/token.actions.githubusercontent.com`; head-bucket says `Not Found`/404; no `/fa-ac/` roles; no budget named `fa-ac-*`; no runtimes in us-east-1 (or none named `fa_ac_*`); no tagged us-east-1 resources. Anything named `fa-ac`/`fa_ac` already present → STOP and report.

- [ ] **Step 5: Confirm nothing else spends in us-east-1 (budget filter assumption)**

Cost Explorer charges $0.01 per request.

```bash
aws ce get-cost-and-usage --region us-east-1 \
  --time-period Start=2026-07-01,End=2026-10-01 --granularity MONTHLY --metrics UnblendedCost \
  --filter '{"Dimensions":{"Key":"REGION","Values":["us-east-1"]}}' \
  --query 'ResultsByTime[].Total.UnblendedCost.Amount'
```
Expected: every month `< 1.00`. If any month ≥ 1.00, STOP: the region-filtered budget would false-alarm; report to the owner (fallback: tag-filtered budget, spec §6).

- [ ] **Step 6: Snapshot df-agentcore for the unchanged check**

```bash
mkdir -p ~/.cache/fa-ac
{
  echo '{"tagged":'; aws resourcegroupstaggingapi get-resources --region eu-central-1 \
     --tag-filters Key=Project,Values=df-agentcore --query 'sort_by(ResourceTagMappingList,&ResourceARN)' --output json
  echo ',"ci_role":'; aws iam get-role --role-name GitHubActionsRole --query 'Role.{Arn:Arn,Trust:AssumeRolePolicyDocument,Max:MaxSessionDuration}' --output json
  echo ',"ci_role_policies":'; aws iam list-attached-role-policies --role-name GitHubActionsRole --output json
  echo ',"oidc":'; aws iam get-open-id-connect-provider --open-id-connect-provider-arn \
     "arn:aws:iam::$ACC:oidc-provider/token.actions.githubusercontent.com" --query '{Clients:ClientIDList,Thumbprints:ThumbprintList}' --output json
  echo '}'
} > ~/.cache/fa-ac/df-snapshot-before.json
python3 -m json.tool ~/.cache/fa-ac/df-snapshot-before.json > /dev/null && echo snapshot ok
```
Expected: `snapshot ok`.

---

### Task 1: Repo scaffold + BusyTracker

**Files:**
- Create: `~/projects/dark-factory-aws/pubspec.yaml`, `analysis_options.yaml`, `.gitignore`, `lib/leg_runner.dart`, `lib/src/busy.dart`
- Test: `test/busy_test.dart`

**Interfaces:**
- Produces: `BusyTracker({int Function()? clock})` (clock returns unix seconds) with `bool tryStart()`, `void finish()`, `Map<String, Object> ping()` → `{'status': 'Healthy'|'HealthyBusy', 'time_of_last_update': int}`. Dart runs the server and legs on one event loop, so no locking is needed.

- [ ] **Step 1: Create the repo and project files**

```bash
mkdir -p ~/projects/dark-factory-aws/{lib/src,bin,test,runner,scripts,docs}
cd ~/projects/dark-factory-aws && git init -b main
```

`pubspec.yaml`:
```yaml
name: leg_runner
description: AgentCore Runtime service that runs one factory leg per session.
publish_to: none

environment:
  sdk: '>=3.12.0 <4.0.0'

dev_dependencies:
  lints: ^6.1.0
  test: ^1.31.2
```

`analysis_options.yaml`:
```yaml
include: package:lints/recommended.yaml
```

`.gitignore`:
```
.dart_tool/
build/
.terraform/
*.tfstate
*.tfstate.*
.terraform.tfstate.lock.info
```

`lib/leg_runner.dart`:
```dart
/// AgentCore Runtime leg runner: the /ping + /invocations contract around
/// one factory leg per session.
library;

export 'src/busy.dart';
export 'src/config.dart';
export 'src/job.dart';
export 'src/results.dart';
export 'src/server.dart';
export 'src/spec.dart';
```

```bash
dart pub get
```
Expected: `Got dependencies!` and a `pubspec.lock`. (`dart analyze` reports the missing `src/` files until Task 4 — expected.)

- [ ] **Step 2: Write the failing test** — `test/busy_test.dart`

```dart
import 'package:leg_runner/src/busy.dart';
import 'package:test/test.dart';

void main() {
  late int now;
  BusyTracker tracker() => BusyTracker(clock: () => now);

  setUp(() => now = 1000);

  test('idle tracker reports Healthy', () {
    expect(tracker().ping(), {'status': 'Healthy', 'time_of_last_update': 1000});
  });

  test('started tracker reports HealthyBusy with the change time', () {
    final busy = tracker();
    now = 1005;
    expect(busy.tryStart(), isTrue);
    expect(busy.ping(), {'status': 'HealthyBusy', 'time_of_last_update': 1005});
  });

  test('second start is refused while busy', () {
    final busy = tracker();
    expect(busy.tryStart(), isTrue);
    expect(busy.tryStart(), isFalse);
  });

  test('finish returns to Healthy and allows the next leg', () {
    final busy = tracker()..tryStart();
    now = 1010;
    busy.finish();
    expect(busy.ping(), {'status': 'Healthy', 'time_of_last_update': 1010});
    expect(busy.tryStart(), isTrue);
  });

  test('ping does not move time_of_last_update', () {
    final busy = tracker();
    now = 2000;
    expect(busy.ping()['time_of_last_update'], 1000);
  });
}
```

- [ ] **Step 3: Run it to verify it fails**

Run: `dart test test/busy_test.dart`
Expected: FAIL — `Error: Error when reading 'lib/src/busy.dart': No such file or directory`.

- [ ] **Step 4: Implement** — `lib/src/busy.dart`

```dart
/// Session health for the AgentCore /ping contract.
///
/// AgentCore keeps a session alive while /ping says HealthyBusy and reclaims
/// it after the idle timeout once it says Healthy. One session runs one leg.
class BusyTracker {
  BusyTracker({int Function()? clock}) : _clock = clock ?? _unixSeconds {
    _changedAt = _clock();
  }

  final int Function() _clock;
  bool _busy = false;
  late int _changedAt;

  static int _unixSeconds() => DateTime.now().millisecondsSinceEpoch ~/ 1000;

  /// Claims the session for a leg; false if a leg is already running.
  bool tryStart() {
    if (_busy) return false;
    _busy = true;
    _changedAt = _clock();
    return true;
  }

  void finish() {
    if (!_busy) return;
    _busy = false;
    _changedAt = _clock();
  }

  /// time_of_last_update moves only when the status changes (contract).
  Map<String, Object> ping() => {
        'status': _busy ? 'HealthyBusy' : 'Healthy',
        'time_of_last_update': _changedAt,
      };
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `dart test test/busy_test.dart`
Expected: `+5: All tests passed!`

- [ ] **Step 6: Commit**

```bash
git add pubspec.yaml pubspec.lock analysis_options.yaml .gitignore lib test
git commit -m "feat: leg_runner scaffold and BusyTracker for the AgentCore ping contract

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 2: LegSpec validation

**Files:**
- Create: `lib/src/spec.dart`
- Test: `test/spec_test.dart`

**Interfaces:**
- Produces: `Map<String, Object?> parseLegSpec(List<int> raw)` (throws `SpecException` with `.message`), `const maxSecondsCap = 28800`. A valid spec has `sessionId` matching `^[A-Za-z0-9_-]{1,100}$`, non-empty string `leg`, int `maxSeconds` in 1..28800, `callback` map with `type == 's3'`. Extra keys (e.g. `hello`) pass through untouched.

- [ ] **Step 1: Write the failing test** — `test/spec_test.dart`

```dart
import 'dart:convert';

import 'package:leg_runner/src/spec.dart';
import 'package:test/test.dart';

/// A valid spec with [overrides] applied; a null value removes the key.
List<int> raw([Map<String, Object?> overrides = const {}]) {
  final spec = <String, Object?>{
    'sessionId': 's' * 40,
    'leg': 'hello',
    'maxSeconds': 60,
    'callback': {'type': 's3'},
    ...overrides,
  }..removeWhere((_, value) => value == null);
  return utf8.encode(jsonEncode(spec));
}

void main() {
  test('accepts a minimal valid spec', () {
    expect(parseLegSpec(raw())['sessionId'], 's' * 40);
  });

  test('passes extra keys through', () {
    expect(parseLegSpec(raw({'hello': {'sleepSeconds': 5}}))['hello'], {'sleepSeconds': 5});
  });

  final nonObjects = <String, List<int>>{
    'empty': [],
    'not json': utf8.encode('not json'),
    'invalid utf-8': [0xff, 0xfe, 0x00],
    'array': utf8.encode('[1, 2]'),
    'string': utf8.encode('"text"'),
  };
  nonObjects.forEach((name, payload) {
    test('rejects a non-object payload: $name', () {
      expect(() => parseLegSpec(payload), throwsA(isA<SpecException>()));
    });
  });

  final invalid = <Map<String, Object?>>[
    {'sessionId': null},
    {'sessionId': ''},
    {'sessionId': 42},
    {'sessionId': '../../etc/passwd'},
    {'sessionId': 'x' * 101},
    {'leg': null},
    {'leg': ''},
    {'maxSeconds': null},
    {'maxSeconds': 0},
    {'maxSeconds': maxSecondsCap + 1},
    {'maxSeconds': '60'},
    {'maxSeconds': true},
    {'maxSeconds': 1.5},
    {'callback': null},
    {'callback': 's3'},
    {'callback': {'type': 'sfn'}},
  ];
  for (final overrides in invalid) {
    test('rejects invalid field $overrides', () {
      expect(() => parseLegSpec(raw(overrides)), throwsA(isA<SpecException>()));
    });
  }
}
```

- [ ] **Step 2: Run it to verify it fails**

Run: `dart test test/spec_test.dart`
Expected: FAIL — `Error when reading 'lib/src/spec.dart'`.

- [ ] **Step 3: Implement** — `lib/src/spec.dart`

```dart
import 'dart:convert';

/// AgentCore's 8-hour session ceiling.
const maxSecondsCap = 28800;

/// sessionId ends up in an S3 key and a file path, so it is restricted to a
/// safe alphabet.
final _sessionId = RegExp(r'^[A-Za-z0-9_-]{1,100}$');

/// The payload is not a usable LegSpec; the server answers 400.
class SpecException implements Exception {
  SpecException(this.message);

  final String message;

  @override
  String toString() => message;
}

/// Validates the LegSpec payload POSTed to /invocations (spec §2.2).
Map<String, Object?> parseLegSpec(List<int> raw) {
  final Object? decoded;
  try {
    decoded = jsonDecode(utf8.decode(raw));
  } on FormatException catch (error) {
    throw SpecException('payload is not JSON: ${error.message}');
  }
  if (decoded is! Map<String, Object?>) {
    throw SpecException('payload must be a JSON object');
  }

  final sessionId = decoded['sessionId'];
  if (sessionId is! String || !_sessionId.hasMatch(sessionId)) {
    throw SpecException('sessionId must be 1-100 characters of A-Z a-z 0-9 _ -');
  }
  final leg = decoded['leg'];
  if (leg is! String || leg.isEmpty) {
    throw SpecException('leg is required');
  }
  final maxSeconds = decoded['maxSeconds'];
  if (maxSeconds is! int || maxSeconds < 1 || maxSeconds > maxSecondsCap) {
    throw SpecException('maxSeconds must be an integer from 1 to $maxSecondsCap');
  }
  final callback = decoded['callback'];
  if (callback is! Map || callback['type'] != 's3') {
    throw SpecException("callback.type must be 's3'");
  }
  return decoded;
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `dart test test/spec_test.dart`
Expected: `+23: All tests passed!`

- [ ] **Step 5: Commit**

```bash
git add lib/src/spec.dart test/spec_test.dart
git commit -m "feat: validate LegSpec payloads

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 3: Result writers + runLeg

**Files:**
- Create: `lib/src/results.dart`, `lib/src/job.dart`
- Test: `test/results_test.dart`, `test/job_test.dart`

**Interfaces:**
- Consumes: `BusyTracker` (Task 1).
- Produces:
  - `String resultKey(String sessionId)` → `'legs/<sessionId>/result.json'`.
  - `abstract interface class ResultWriter { Future<void> write(String sessionId, Map<String, Object?> result); }`; `FileResultWriter(Directory root)`; `AwsCliResultWriter(String bucket, {ProcessRunner? run})` where `typedef ProcessRunner = Future<ProcessResult> Function(String executable, List<String> arguments)` — runs `aws s3api put-object --bucket <b> --key <resultKey> --body <tmp json> --content-type application/json`, throws `StateError` on non-zero exit.
  - `Future<void> runLeg(Map<String, Object?> spec, ResultWriter writer, BusyTracker busy, List<String> command)`: runs `setsid <command...> <spec-json-path>` (own process group), kills the group at `maxSeconds`, writes `LegResult` = `{'status': 'ok'|'agent_failed'|'infra_error', 'exitCode': int?, 'sessionId', 'durationSeconds': double, 'reason'?: String}`, always calls `busy.finish()`. Exit codes 126/127 (command not executable / not found) are `infra_error`.

- [ ] **Step 1: Write the failing writer test** — `test/results_test.dart`

```dart
import 'dart:convert';
import 'dart:io';

import 'package:leg_runner/src/results.dart';
import 'package:test/test.dart';

void main() {
  test('resultKey layout', () {
    expect(resultKey('abc'), 'legs/abc/result.json');
  });

  test('AwsCliResultWriter puts the JSON under the result key', () async {
    late List<String> args;
    late String body;
    final writer = AwsCliResultWriter('bucket-x', run: (exe, arguments) async {
      expect(exe, 'aws');
      args = arguments;
      body = File(arguments[arguments.indexOf('--body') + 1]).readAsStringSync();
      return ProcessResult(0, 0, '', '');
    });
    await writer.write('sess-1', {'status': 'ok'});
    expect(args, [
      's3api', 'put-object', '--bucket', 'bucket-x', '--key', 'legs/sess-1/result.json',
      '--body', args[7], '--content-type', 'application/json',
    ]);
    expect(jsonDecode(body), {'status': 'ok'});
  });

  test('AwsCliResultWriter surfaces a failed upload', () async {
    final writer = AwsCliResultWriter('bucket-x', run: (_, _) async => ProcessResult(0, 255, '', 'AccessDenied'));
    expect(() => writer.write('sess-1', {'status': 'ok'}), throwsA(isA<StateError>()));
  });
}
```

- [ ] **Step 2: Write the failing job test** — `test/job_test.dart`

```dart
import 'dart:convert';
import 'dart:io';

import 'package:leg_runner/src/busy.dart';
import 'package:leg_runner/src/job.dart';
import 'package:leg_runner/src/results.dart';
import 'package:test/test.dart';

Map<String, Object?> spec({int maxSeconds = 30}) =>
    {'sessionId': 'sess-1', 'leg': 'hello', 'maxSeconds': maxSeconds, 'callback': {'type': 's3'}};

BusyTracker started() => BusyTracker()..tryStart();

class BrokenWriter implements ResultWriter {
  @override
  Future<void> write(String sessionId, Map<String, Object?> result) async => throw StateError('s3 is down');
}

void main() {
  late Directory root;

  setUp(() async => root = await Directory.systemTemp.createTemp('legtest-'));
  tearDown(() => root.delete(recursive: true));

  Map<String, Object?> result() =>
      jsonDecode(File('${root.path}/${resultKey('sess-1')}').readAsStringSync()) as Map<String, Object?>;

  test('successful command reports ok and frees the session', () async {
    final busy = started();
    await runLeg(spec(), FileResultWriter(root), busy, ['bash', '-c', 'exit 0']);
    expect(result(), containsPair('status', 'ok'));
    expect(result(), containsPair('exitCode', 0));
    expect(result(), containsPair('sessionId', 'sess-1'));
    expect(busy.ping()['status'], 'Healthy');
  });

  test('the leg runs to completion, not just until setsid returns', () async {
    await runLeg(spec(), FileResultWriter(root), started(), ['bash', '-c', 'sleep 1']);
    expect(result()['durationSeconds'] as num, greaterThanOrEqualTo(0.9));
  });

  test('failing command reports agent_failed', () async {
    await runLeg(spec(), FileResultWriter(root), started(), ['bash', '-c', 'exit 3']);
    expect(result(), containsPair('status', 'agent_failed'));
    expect(result(), containsPair('exitCode', 3));
  });

  test('command over maxSeconds is killed and reported', () async {
    await runLeg(spec(maxSeconds: 1), FileResultWriter(root), started(), ['bash', '-c', 'sleep 30']);
    expect(result(), containsPair('status', 'infra_error'));
    expect(result(), containsPair('reason', 'timeout'));
    expect(result()['durationSeconds'] as num, lessThan(10));
  });

  test('timeout kills background children too', () async {
    final pidFile = '${root.path}/child.pid';
    await runLeg(spec(maxSeconds: 1), FileResultWriter(root), started(),
        ['bash', '-c', 'sleep 60 & echo \$! > $pidFile; wait']);
    final child = File(pidFile).readAsStringSync().trim();
    final deadline = DateTime.now().add(const Duration(seconds: 5));
    while (Directory('/proc/$child').existsSync()) {
      if (DateTime.now().isAfter(deadline)) fail('background child $child survived the leg timeout');
      await Future<void>.delayed(const Duration(milliseconds: 50));
    }
  });

  test('missing command reports infra_error', () async {
    await runLeg(spec(), FileResultWriter(root), started(), ['/nonexistent/run-leg.sh']);
    expect(result(), containsPair('status', 'infra_error'));
    expect(result()['reason'] as String, startsWith('could not start'));
  });

  test('the command receives the spec as a file', () async {
    final seen = '${root.path}/seen.json';
    await runLeg(spec(), FileResultWriter(root), started(), ['bash', '-c', 'cp "\$1" $seen', '_']);
    expect(jsonDecode(File(seen).readAsStringSync()), containsPair('sessionId', 'sess-1'));
  });

  test('writer failure still frees the session', () async {
    final busy = started();
    await runLeg(spec(), BrokenWriter(), busy, ['bash', '-c', 'exit 0']);
    expect(busy.ping()['status'], 'Healthy');
  });
}
```

- [ ] **Step 3: Run them to verify they fail**

Run: `dart test test/results_test.dart test/job_test.dart`
Expected: FAIL — `Error when reading 'lib/src/results.dart'`.

- [ ] **Step 4: Implement** — `lib/src/results.dart`

```dart
import 'dart:convert';
import 'dart:io';

String resultKey(String sessionId) => 'legs/$sessionId/result.json';

/// Where a LegResult goes: S3 in AgentCore, a directory in local tests.
abstract interface class ResultWriter {
  Future<void> write(String sessionId, Map<String, Object?> result);
}

class FileResultWriter implements ResultWriter {
  FileResultWriter(this.root);

  final Directory root;

  @override
  Future<void> write(String sessionId, Map<String, Object?> result) async {
    final file = File('${root.path}/${resultKey(sessionId)}');
    await file.parent.create(recursive: true);
    await file.writeAsString(jsonEncode(result));
  }
}

typedef ProcessRunner = Future<ProcessResult> Function(String executable, List<String> arguments);

/// Uploads through the aws CLI baked into the image: Dart has no official AWS
/// SDK, and the CLI picks up the runtime role's credentials by itself.
class AwsCliResultWriter implements ResultWriter {
  AwsCliResultWriter(this.bucket, {ProcessRunner? run}) : _run = run ?? Process.run;

  final String bucket;
  final ProcessRunner _run;

  @override
  Future<void> write(String sessionId, Map<String, Object?> result) async {
    final dir = await Directory.systemTemp.createTemp('legresult-');
    try {
      final body = File('${dir.path}/result.json');
      await body.writeAsString(jsonEncode(result));
      final outcome = await _run('aws', [
        's3api', 'put-object', '--bucket', bucket, '--key', resultKey(sessionId),
        '--body', body.path, '--content-type', 'application/json',
      ]);
      if (outcome.exitCode != 0) {
        throw StateError('aws s3api put-object failed (${outcome.exitCode}): ${outcome.stderr}');
      }
    } finally {
      await dir.delete(recursive: true);
    }
  }
}
```

- [ ] **Step 5: Implement** — `lib/src/job.dart`

```dart
import 'dart:async';
import 'dart:convert';
import 'dart:io';

import 'busy.dart';
import 'results.dart';

/// Runs one leg: hands the LegSpec to the leg command, then delivers a LegResult.
///
/// `setsid` puts the command in its own process group, so a timeout kills
/// everything the leg started — an orphan would keep a paid session busy.
Future<void> runLeg(
  Map<String, Object?> spec,
  ResultWriter writer,
  BusyTracker busy,
  List<String> command,
) async {
  final stopwatch = Stopwatch()..start();
  final sessionId = spec['sessionId'] as String;
  try {
    final result = await _execute(spec, command)
      ..['sessionId'] = sessionId
      ..['durationSeconds'] = stopwatch.elapsedMilliseconds / 1000;
    try {
      await writer.write(sessionId, result);
    } catch (error) {
      stderr.writeln('could not deliver LegResult for session $sessionId: $error');
    }
  } finally {
    busy.finish();
  }
}

Future<Map<String, Object?>> _execute(Map<String, Object?> spec, List<String> command) async {
  final dir = await Directory.systemTemp.createTemp('legspec-');
  try {
    final specFile = File('${dir.path}/spec.json');
    await specFile.writeAsString(jsonEncode(spec));
    final Process process;
    try {
      process = await Process.start('setsid', [...command, specFile.path], mode: ProcessStartMode.inheritStdio);
    } on ProcessException catch (error) {
      return {'status': 'infra_error', 'exitCode': null, 'reason': 'could not start leg command: ${error.message}'};
    }
    final int code;
    try {
      code = await process.exitCode.timeout(Duration(seconds: spec['maxSeconds'] as int));
    } on TimeoutException {
      // setsid execs the command in place, so its pid is the process-group id.
      await Process.run('bash', ['-c', 'kill -KILL -- -${process.pid}']);
      await process.exitCode;
      return {'status': 'infra_error', 'exitCode': null, 'reason': 'timeout'};
    }
    if (code == 126 || code == 127) {
      return {'status': 'infra_error', 'exitCode': code, 'reason': 'could not start leg command (exit $code)'};
    }
    return {'status': code == 0 ? 'ok' : 'agent_failed', 'exitCode': code};
  } finally {
    await dir.delete(recursive: true);
  }
}
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `dart test test/results_test.dart test/job_test.dart`
Expected: `+11: All tests passed!` If "runs to completion" fails, `setsid` forked instead of exec'ing (the Dart child was a process-group leader): switch to `setsid --wait` and record the pid via `bash -c 'echo $$ > "$PIDFILE"; exec "$@"'` — do not drop the test.

- [ ] **Step 7: Commit**

```bash
git add lib/src/results.dart lib/src/job.dart test/results_test.dart test/job_test.dart
git commit -m "feat: run a leg in its own process group and deliver its LegResult

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 4: HTTP server + configuration + entrypoint

**Files:**
- Create: `lib/src/server.dart`, `lib/src/config.dart`, `bin/leg_runner.dart`
- Test: `test/server_test.dart`, `test/config_test.dart`

**Interfaces:**
- Consumes: `BusyTracker` (Task 1), `parseLegSpec`/`SpecException` (Task 2), `runLeg`, `FileResultWriter`, `AwsCliResultWriter` (Task 3).
- Produces: `typedef LegStarter = Future<void> Function(Map<String, Object?> spec)`; `Future<HttpServer> serve(Object address, int port, BusyTracker busy, LegStarter startLeg)`; `const maxBodyBytes = 1048576`; responses: `GET /ping` 200 ping JSON; `POST /invocations` 200 `{"accepted":true,"sessionId":…}`, 400 bad spec, 409 busy, 413 too large, 404 other paths. `ResultWriter buildWriter(Map<String, String> env)` (`LEG_RESULT_DIR` → file, else `LEG_RESULT_BUCKET` → aws CLI, else `StateError`). `bin/leg_runner.dart` serves on `0.0.0.0:8080`, leg command from `LEG_COMMAND` (default `/app/run-leg.sh`).

- [ ] **Step 1: Write the failing server test** — `test/server_test.dart`

```dart
import 'dart:convert';
import 'dart:io';

import 'package:leg_runner/src/busy.dart';
import 'package:leg_runner/src/server.dart';
import 'package:test/test.dart';

const valid = {'sessionId': 'sess-1', 'leg': 'hello', 'maxSeconds': 60, 'callback': {'type': 's3'}};

void main() {
  late HttpServer server;
  late BusyTracker busy;
  late List<Map<String, Object?>> started;

  setUp(() async {
    busy = BusyTracker();
    started = [];
    server = await serve(InternetAddress.loopbackIPv4, 0, busy, (spec) async => started.add(spec));
  });
  tearDown(() => server.close(force: true));

  Future<(int, Map<String, Object?>)> call(String path, {Object? body}) async {
    final client = HttpClient();
    try {
      final request = body == null
          ? await client.get('127.0.0.1', server.port, path)
          : await client.post('127.0.0.1', server.port, path);
      if (body != null) request.add(body is List<int> ? body : utf8.encode(jsonEncode(body)));
      final response = await request.close();
      final text = await utf8.decodeStream(response);
      return (response.statusCode, jsonDecode(text) as Map<String, Object?>);
    } finally {
      client.close(force: true);
    }
  }

  /// Sends raw request bytes; returns everything the server sends back.
  Future<String> raw(String head) async {
    final socket = await Socket.connect('127.0.0.1', server.port);
    socket.write(head);
    await socket.flush();
    // Read until the response head arrives or the server closes; the server
    // may keep an unread body's connection open, so never wait for close.
    final reply = StringBuffer();
    await for (final chunk in socket.timeout(const Duration(seconds: 5), onTimeout: (sink) => sink.close())) {
      reply.write(utf8.decode(chunk, allowMalformed: true));
      if (reply.toString().contains('\r\n\r\n')) break;
    }
    socket.destroy();
    return reply.toString();
  }

  test('ping reports Healthy when idle', () async {
    final (status, body) = await call('/ping');
    expect(status, 200);
    expect(body['status'], 'Healthy');
  });

  test('valid invocation is accepted, started, and marks the session busy', () async {
    final (status, body) = await call('/invocations', body: valid);
    expect(status, 200);
    expect(body, {'accepted': true, 'sessionId': 'sess-1'});
    expect(started.single['sessionId'], 'sess-1');
    expect((await call('/ping')).$2['status'], 'HealthyBusy');
  });

  test('second invocation while busy is refused', () async {
    await call('/invocations', body: valid);
    final (status, body) = await call('/invocations', body: valid);
    expect(status, 409);
    expect(body['error'], contains('already running'));
    expect(started, hasLength(1));
  });

  test('invalid spec is rejected and the session stays Healthy', () async {
    final (status, body) = await call('/invocations', body: {'leg': 'hello'});
    expect(status, 400);
    expect(body['error'], contains('sessionId'));
    expect(started, isEmpty);
    expect((await call('/ping')).$2['status'], 'Healthy');
  });

  test('non-numeric Content-Length never starts a leg', () async {
    final reply = await raw('POST /invocations HTTP/1.1\r\nHost: x\r\nContent-Length: abc\r\nConnection: close\r\n\r\n');
    expect(reply, anyOf(isEmpty, startsWith('HTTP/1.1 400')));
    expect(started, isEmpty);
    expect((await call('/ping')).$2['status'], 'Healthy');
  });

  test('oversized body is refused before reading', () async {
    final reply = await raw('POST /invocations HTTP/1.1\r\nHost: x\r\n'
        'Content-Length: ${maxBodyBytes + 1}\r\nConnection: close\r\n\r\n');
    expect(reply, startsWith('HTTP/1.1 413'));
    expect(started, isEmpty);
  });

  for (final path in ['/', '/invoke', '/ping/extra']) {
    test('unknown path $path is 404', () async {
      expect((await call(path)).$1, 404);
      expect((await call(path, body: valid)).$1, 404);
    });
  }
}
```

- [ ] **Step 2: Write the failing config test** — `test/config_test.dart`

```dart
import 'package:leg_runner/src/config.dart';
import 'package:leg_runner/src/results.dart';
import 'package:test/test.dart';

void main() {
  test('LEG_RESULT_DIR selects the file writer', () {
    expect(buildWriter({'LEG_RESULT_DIR': '/tmp/results'}), isA<FileResultWriter>());
  });

  test('LEG_RESULT_BUCKET selects the aws CLI writer', () {
    final writer = buildWriter({'LEG_RESULT_BUCKET': 'fa-ac-legs-test'});
    expect(writer, isA<AwsCliResultWriter>());
    expect((writer as AwsCliResultWriter).bucket, 'fa-ac-legs-test');
  });

  test('a missing destination fails fast', () {
    expect(() => buildWriter({}), throwsA(isA<StateError>()));
  });
}
```

- [ ] **Step 3: Run them to verify they fail**

Run: `dart test test/server_test.dart test/config_test.dart`
Expected: FAIL — `Error when reading 'lib/src/server.dart'`.

- [ ] **Step 4: Implement** — `lib/src/server.dart`

```dart
import 'dart:async';
import 'dart:convert';
import 'dart:io';

import 'busy.dart';
import 'spec.dart';

const maxBodyBytes = 1024 * 1024;

typedef LegStarter = Future<void> Function(Map<String, Object?> spec);

/// HTTP surface of the AgentCore Runtime contract: GET /ping, POST /invocations.
///
/// /invocations answers at once and runs the leg in the background; progress
/// is visible through /ping and the LegResult the leg writes.
Future<HttpServer> serve(Object address, int port, BusyTracker busy, LegStarter startLeg) async {
  final server = await HttpServer.bind(address, port);
  server.listen(
    (request) {
      _handle(request, busy, startLeg).catchError((Object error) {
        stderr.writeln('request failed: $error');
      });
    },
    // Malformed requests (e.g. a non-numeric Content-Length) surface here.
    onError: (Object error) => stderr.writeln('connection error: $error'),
  );
  return server;
}

Future<void> _handle(HttpRequest request, BusyTracker busy, LegStarter startLeg) async {
  final path = request.uri.path;
  if (request.method == 'GET' && path == '/ping') {
    return _reply(request, 200, busy.ping());
  }
  if (request.method != 'POST' || path != '/invocations') {
    return _reply(request, 404, {'error': 'not found'});
  }
  if (request.contentLength > maxBodyBytes) {
    return _reply(request, 413, {'error': 'payload too large'});
  }
  final body = <int>[];
  await for (final chunk in request) {
    body.addAll(chunk);
    if (body.length > maxBodyBytes) {
      return _reply(request, 413, {'error': 'payload too large'});
    }
  }
  final Map<String, Object?> spec;
  try {
    spec = parseLegSpec(body);
  } on SpecException catch (error) {
    return _reply(request, 400, {'error': error.message});
  }
  if (!busy.tryStart()) {
    return _reply(request, 409, {'error': 'a leg is already running in this session'});
  }
  unawaited(startLeg(spec));
  return _reply(request, 200, {'accepted': true, 'sessionId': spec['sessionId']});
}

Future<void> _reply(HttpRequest request, int status, Map<String, Object?> body) async {
  request.response
    ..statusCode = status
    ..headers.contentType = ContentType.json
    ..write(jsonEncode(body));
  await request.response.close();
}
```

- [ ] **Step 5: Implement** — `lib/src/config.dart`

```dart
import 'dart:io';

import 'results.dart';

/// Picks where LegResults go: a local directory (tests, contract test) or
/// S3 through the aws CLI (AgentCore).
ResultWriter buildWriter(Map<String, String> env) {
  final dir = env['LEG_RESULT_DIR'];
  if (dir != null && dir.isNotEmpty) return FileResultWriter(Directory(dir));
  final bucket = env['LEG_RESULT_BUCKET'];
  if (bucket != null && bucket.isNotEmpty) return AwsCliResultWriter(bucket);
  throw StateError('set LEG_RESULT_BUCKET (AgentCore) or LEG_RESULT_DIR (local)');
}
```

- [ ] **Step 6: Implement** — `bin/leg_runner.dart`

```dart
import 'dart:io';

import 'package:leg_runner/leg_runner.dart';

/// Container entrypoint: serves the AgentCore contract on 0.0.0.0:8080.
Future<void> main() async {
  final env = Platform.environment;
  final writer = buildWriter(env);
  final busy = BusyTracker();
  final command = [env['LEG_COMMAND'] ?? '/app/run-leg.sh'];
  await serve(InternetAddress.anyIPv4, 8080, busy, (spec) => runLeg(spec, writer, busy, command));
  stdout.writeln('leg runner listening on 0.0.0.0:8080');
}
```

- [ ] **Step 7: Run the whole suite and the analyzer**

```bash
dart analyze --fatal-infos
dart test
```
Expected: `No issues found!` and `+51: All tests passed!` (5 + 23 + 3 + 8 + 9 + 3).

- [ ] **Step 8: Commit**

```bash
git add lib bin test
git commit -m "feat: AgentCore /ping and /invocations server and entrypoint

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 5: Container image + hello leg + local contract test

**Files:**
- Create: `runner/run-leg.sh`, `Dockerfile`, `.dockerignore`, `scripts/contract-test.sh`

**Interfaces:**
- Consumes: `bin/leg_runner.dart` (Task 4).
- Produces: image `fa-ac-leg-runner` (linux/arm64, AOT-compiled Dart + aws CLI) that the deploy workflow (Task 9) builds; hello-leg payload extension `"hello": {"sleepSeconds": <int>}` used by Tasks 5 and 10.

- [ ] **Step 1: Write the hello leg** — `runner/run-leg.sh`

```bash
#!/usr/bin/env bash
# M0 hello leg: proves the AgentCore contract end to end. M2 replaces this
# file with the real factory leg (checkout, fa, push).
set -euo pipefail
spec="$1"
session="$(jq -r .sessionId "$spec")"
sleep_seconds="$(jq -r '(.hello.sleepSeconds // 0) | floor' "$spec")"
echo "hello leg: session=${session} sleeping ${sleep_seconds}s"
sleep "${sleep_seconds}"
echo "hello leg: done"
```

```bash
chmod +x runner/run-leg.sh
```

- [ ] **Step 2: Write the image** — `Dockerfile` and `.dockerignore`

`Dockerfile`:
```dockerfile
# AgentCore Runtime requires linux/arm64 and port 8080.
FROM --platform=linux/arm64 dart:stable AS build
WORKDIR /src
COPY pubspec.yaml pubspec.lock ./
RUN dart pub get
COPY lib ./lib
COPY bin ./bin
RUN dart compile exe bin/leg_runner.dart -o /out/leg_runner

FROM --platform=linux/arm64 debian:bookworm-slim
# aws CLI: S3 uploads (Dart has no official AWS SDK). jq: the hello leg.
RUN apt-get update \
 && apt-get install -y --no-install-recommends awscli jq ca-certificates \
 && rm -rf /var/lib/apt/lists/* \
 && useradd --create-home --uid 10001 runner
COPY --from=build /out/leg_runner /app/leg_runner
COPY --chmod=0755 runner/run-leg.sh /app/run-leg.sh
USER runner
EXPOSE 8080
CMD ["/app/leg_runner"]
```

`.dockerignore`:
```
.git
.github
.dart_tool
build
test
terraform
docs
scripts
```

- [ ] **Step 3: Write the contract test** — `scripts/contract-test.sh`

Dart's `jsonEncode` emits compact JSON (`"status":"Healthy"`, no spaces); the greps match that.

```bash
#!/usr/bin/env bash
# Runs the leg-runner image locally (native on ARM64) and checks the AgentCore
# contract: Healthy -> accept -> HealthyBusy -> LegResult ok -> Healthy.
set -euo pipefail
image="${1:-fa-ac-leg-runner:local}"
port=18080
results="$(mktemp -d)"
chmod 0777 "$results"
cid="$(docker run -d --rm -p "${port}:8080" -e LEG_RESULT_DIR=/results -v "${results}:/results" "$image")"
trap 'docker rm -f "$cid" >/dev/null 2>&1 || true; rm -rf "$results"' EXIT

wait_for() {  # wait_for <seconds> <command...>
  local deadline=$((SECONDS + $1)); shift
  until "$@" >/dev/null 2>&1; do
    if (( SECONDS >= deadline )); then echo "timed out: $*"; return 1; fi
    sleep 0.2
  done
}
ping_is() { curl -fsS "localhost:${port}/ping" | grep -q "\"status\":\"$1\""; }

wait_for 10 ping_is Healthy
session="contract-test-$(date +%s)-padding-to-33-chars"
curl -fsS -X POST "localhost:${port}/invocations" -H 'Content-Type: application/json' \
  -d "{\"sessionId\":\"${session}\",\"leg\":\"hello\",\"maxSeconds\":60,\"callback\":{\"type\":\"s3\"},\"hello\":{\"sleepSeconds\":3}}" \
  | grep -q '"accepted":true'
ping_is HealthyBusy
wait_for 15 test -f "${results}/legs/${session}/result.json"
grep -q '"status":"ok"' "${results}/legs/${session}/result.json"
wait_for 5 ping_is Healthy
echo "contract test: PASS"
```

```bash
chmod +x scripts/contract-test.sh
```

- [ ] **Step 4: Build and run the contract test**

```bash
docker build -t fa-ac-leg-runner:local .
docker image inspect fa-ac-leg-runner:local --format '{{.Architecture}} {{.Size}}'
docker run --rm --entrypoint aws fa-ac-leg-runner:local --version
scripts/contract-test.sh fa-ac-leg-runner:local
```
Expected: `arm64 <size well under 2000000000>`, an `aws-cli/…` version line, and `contract test: PASS`. If the container exits with `GLIBC_… not found`, the `dart:stable` build stage is on a newer Debian than the runtime stage: change the runtime `FROM` to the same Debian release as `dart:stable` (`docker run --rm dart:stable cat /etc/debian_version`).

- [ ] **Step 5: Commit**

```bash
git add runner/run-leg.sh Dockerfile .dockerignore scripts/contract-test.sh
git commit -m "feat: ARM64 leg-runner image (AOT Dart + aws CLI) with hello leg and contract test

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 6: Bootstrap Terraform — state bucket + fenced CI role (owner applies once)

**Files:**
- Create: `terraform/bootstrap/main.tf`, `terraform/bootstrap/fence.tf`, `terraform/bootstrap/tests/bootstrap.tftest.hcl`

**Interfaces:**
- Produces (live, for Tasks 7–9): S3 bucket `fa-ac-state-<account>`; managed policy `arn:aws:iam::<account>:policy/fa-ac/fa-ac-boundary`; role `arn:aws:iam::<account>:role/fa-ac/fa-ac-ci` trusted only by the OIDC subject in `var.deploy_subject`. Outputs `state_bucket`, `ci_role_arn`, `boundary_arn`.

- [ ] **Step 1: Write the configuration** — `terraform/bootstrap/main.tf`

```hcl
# One-time bootstrap, applied LOCALLY by the owner with admin credentials:
# CI cannot create the role it runs as. Touches nothing of df-agentcore.
terraform {
  required_version = ">= 1.10.0"
  required_providers {
    aws = { source = "hashicorp/aws", version = "6.65.0" }
  }
}

provider "aws" {
  region = "us-east-1"
  default_tags { tags = { Project = "fa-agentcore", ManagedBy = "Terraform" } }
}

variable "deploy_subject" {
  type        = string
  description = "Exact GitHub OIDC sub of dark-factory-aws deploy jobs (from whoami.yml)."
  validation {
    condition     = !strcontains(var.deploy_subject, "*") && endswith(var.deploy_subject, ":environment:aws-deploy")
    error_message = "deploy_subject must be an exact subject (no wildcards) ending in :environment:aws-deploy."
  }
}

data "aws_caller_identity" "current" {}

# Shared with df-agentcore: read, never manage.
data "aws_iam_openid_connect_provider" "github" {
  url = "https://token.actions.githubusercontent.com"
}

locals {
  account      = data.aws_caller_identity.current.account_id
  boundary_arn = "arn:aws:iam::${local.account}:policy/fa-ac/fa-ac-boundary"
}

resource "aws_s3_bucket" "state" {
  bucket = "fa-ac-state-${local.account}"
  lifecycle { prevent_destroy = true }
}

resource "aws_s3_bucket_versioning" "state" {
  bucket = aws_s3_bucket.state.id
  versioning_configuration { status = "Enabled" }
}

resource "aws_s3_bucket_public_access_block" "state" {
  bucket                  = aws_s3_bucket.state.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_s3_bucket_server_side_encryption_configuration" "state" {
  bucket = aws_s3_bucket.state.id
  rule {
    apply_server_side_encryption_by_default { sse_algorithm = "AES256" }
  }
}

resource "aws_s3_bucket_policy" "state" {
  bucket = aws_s3_bucket.state.id
  policy = jsonencode({ Version = "2012-10-17", Statement = [{
    Effect    = "Deny", Principal = "*", Action = "s3:*",
    Resource  = [aws_s3_bucket.state.arn, "${aws_s3_bucket.state.arn}/*"],
    Condition = { Bool = { "aws:SecureTransport" = "false" } }
  }] })
}

resource "aws_iam_policy" "boundary" {
  name   = "fa-ac-boundary"
  path   = "/fa-ac/"
  policy = local.fence_policy
}

resource "aws_iam_role" "ci" {
  name                 = "fa-ac-ci"
  path                 = "/fa-ac/"
  max_session_duration = 3600
  permissions_boundary = local.boundary_arn
  assume_role_policy = jsonencode({ Version = "2012-10-17", Statement = [{
    Effect    = "Allow"
    Principal = { Federated = data.aws_iam_openid_connect_provider.github.arn }
    Action    = "sts:AssumeRoleWithWebIdentity"
    Condition = { StringEquals = {
      "token.actions.githubusercontent.com:aud" = "sts.amazonaws.com"
      "token.actions.githubusercontent.com:sub" = var.deploy_subject
    } }
  }] })
  depends_on = [aws_iam_policy.boundary]
}

resource "aws_iam_role_policy" "ci" {
  name   = "fa-ac-ci-fence"
  role   = aws_iam_role.ci.id
  policy = local.fence_policy
}

output "state_bucket" { value = aws_s3_bucket.state.id }
output "ci_role_arn" { value = aws_iam_role.ci.arn }
output "boundary_arn" { value = local.boundary_arn }
```

- [ ] **Step 2: Write the fence** — `terraform/bootstrap/fence.tf`

```hcl
# The fence: the most any fa-ac role (CI included) can ever do. Used both as
# the permissions boundary and as the CI role's own policy (spec §6).
locals {
  df_names = [
    "arn:aws:iam::${local.account}:role/GitHubActionsRole",
    "arn:aws:iam::${local.account}:role/darkfactory-*",
    "arn:aws:iam::${local.account}:policy/darkfactory-*",
    "arn:aws:s3:::df-agentcore-*",
    "arn:aws:s3:::df-agentcore-*/*",
    "arn:aws:secretsmanager:*:${local.account}:secret:darkfactory-*",
    "arn:aws:ecr:*:${local.account}:repository/darkfactory-*",
  ]

  fence_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "RegionalServicesInUsEast1Only"
        Effect = "Allow"
        Action = [
          "s3:*", "ecr:*", "bedrock-agentcore:*", "logs:*", "xray:*", "cloudwatch:*",
          "secretsmanager:*", "states:*", "lambda:*", "apigateway:*", "wafv2:*", "sts:GetCallerIdentity",
        ]
        Resource  = "*"
        Condition = { StringEquals = { "aws:RequestedRegion" = "us-east-1" } }
      },
      {
        Sid      = "OwnBudgetsOnly"
        Effect   = "Allow"
        Action   = ["budgets:*"]
        Resource = "arn:aws:budgets::${local.account}:budget/fa-ac-*"
      },
      {
        Sid      = "ReadSharedGitHubOidcProvider"
        Effect   = "Allow"
        Action   = ["iam:GetOpenIDConnectProvider", "iam:ListOpenIDConnectProviders"]
        Resource = "*"
      },
      {
        Sid    = "ManageOwnIamPath"
        Effect = "Allow"
        Action = ["iam:*"]
        Resource = [
          "arn:aws:iam::${local.account}:role/fa-ac/*",
          "arn:aws:iam::${local.account}:policy/fa-ac/*",
        ]
      },
      {
        Sid       = "RolesMustCarryBoundary"
        Effect    = "Deny"
        Action    = ["iam:CreateRole", "iam:PutRolePermissionsBoundary"]
        Resource  = "*"
        Condition = { StringNotEquals = { "iam:PermissionsBoundary" = local.boundary_arn } }
      },
      {
        Sid      = "BoundaryIsPermanent"
        Effect   = "Deny"
        Action   = ["iam:DeleteRolePermissionsBoundary"]
        Resource = "*"
      },
      {
        Sid    = "FenceIsBootstrapOnly"
        Effect = "Deny"
        Action = [
          "iam:CreatePolicyVersion", "iam:DeletePolicy", "iam:DeletePolicyVersion",
          "iam:SetDefaultPolicyVersion", "iam:TagPolicy", "iam:UntagPolicy",
        ]
        Resource = local.boundary_arn
      },
      {
        Sid      = "CiRoleIsBootstrapOnly"
        Effect   = "Deny"
        Action   = ["iam:*"]
        Resource = "arn:aws:iam::${local.account}:role/fa-ac/fa-ac-ci"
      },
      {
        Sid    = "NeverChangeTheSharedOidcProvider"
        Effect = "Deny"
        Action = [
          "iam:CreateOpenIDConnectProvider", "iam:DeleteOpenIDConnectProvider",
          "iam:UpdateOpenIDConnectProviderThumbprint", "iam:AddClientIDToOpenIDConnectProvider",
          "iam:RemoveClientIDFromOpenIDConnectProvider", "iam:TagOpenIDConnectProvider",
          "iam:UntagOpenIDConnectProvider",
        ]
        Resource = "*"
      },
      {
        Sid       = "NeverTouchDfAgentcoreByTag"
        Effect    = "Deny"
        Action    = "*"
        Resource  = "*"
        Condition = { StringEquals = { "aws:ResourceTag/Project" = "df-agentcore" } }
      },
      {
        Sid      = "NeverTouchDfAgentcoreByName"
        Effect   = "Deny"
        Action   = "*"
        Resource = local.df_names
      },
      {
        Sid       = "NeverActInDfAgentcoreRegion"
        Effect    = "Deny"
        Action    = "*"
        Resource  = "*"
        Condition = { StringEquals = { "aws:RequestedRegion" = "eu-central-1" } }
      },
    ]
  })
}
```

- [ ] **Step 3: Write the tests** — `terraform/bootstrap/tests/bootstrap.tftest.hcl`

```hcl
mock_provider "aws" {
  mock_data "aws_caller_identity" {
    defaults = { account_id = "111111111111" }
  }
  mock_data "aws_iam_openid_connect_provider" {
    defaults = { arn = "arn:aws:iam::111111111111:oidc-provider/token.actions.githubusercontent.com" }
  }
}

variables {
  deploy_subject = "repo:rustembuild/dark-factory-aws:environment:aws-deploy"
}

run "ci_role_trusts_only_the_deploy_environment" {
  command = plan
  assert {
    condition     = jsondecode(aws_iam_role.ci.assume_role_policy).Statement[0].Condition.StringEquals["token.actions.githubusercontent.com:sub"] == "repo:rustembuild/dark-factory-aws:environment:aws-deploy"
    error_message = "CI role must trust exactly the deploy subject."
  }
  assert {
    condition     = jsondecode(aws_iam_role.ci.assume_role_policy).Statement[0].Principal.Federated == "arn:aws:iam::111111111111:oidc-provider/token.actions.githubusercontent.com"
    error_message = "CI role must use the shared GitHub OIDC provider."
  }
  assert {
    condition     = aws_iam_role.ci.permissions_boundary == "arn:aws:iam::111111111111:policy/fa-ac/fa-ac-boundary" && aws_iam_role.ci.path == "/fa-ac/"
    error_message = "CI role must sit under /fa-ac/ with the fa-ac boundary."
  }
}

run "fence_is_both_boundary_and_ci_policy" {
  command = plan
  assert {
    condition     = aws_iam_policy.boundary.policy == aws_iam_role_policy.ci.policy
    error_message = "Boundary and CI policy must be the same fence."
  }
}

run "fence_denies_df_agentcore" {
  command = plan
  assert {
    condition = alltrue([for sid in ["NeverTouchDfAgentcoreByTag", "NeverTouchDfAgentcoreByName", "NeverActInDfAgentcoreRegion", "NeverChangeTheSharedOidcProvider", "RolesMustCarryBoundary", "CiRoleIsBootstrapOnly"] :
      one([for s in jsondecode(aws_iam_policy.boundary.policy).Statement : s.Effect if s.Sid == sid]) == "Deny"])
    error_message = "Every df-agentcore and escalation guard must be an explicit Deny."
  }
  assert {
    condition     = contains(one([for s in jsondecode(aws_iam_policy.boundary.policy).Statement : s.Resource if s.Sid == "NeverTouchDfAgentcoreByName"]), "arn:aws:iam::111111111111:role/GitHubActionsRole")
    error_message = "df-agentcore's CI role must be explicitly protected."
  }
}

run "state_bucket_is_project_named" {
  command = plan
  assert {
    condition     = aws_s3_bucket.state.bucket == "fa-ac-state-111111111111"
    error_message = "State bucket must be fa-ac-state-<account>."
  }
}

run "wildcard_subject_is_rejected" {
  command         = plan
  variables { deploy_subject = "repo:rustembuild/dark-factory-aws:*" }
  expect_failures = [var.deploy_subject]
}

run "branch_subject_is_rejected" {
  command         = plan
  variables { deploy_subject = "repo:rustembuild/dark-factory-aws:ref:refs/heads/main" }
  expect_failures = [var.deploy_subject]
}
```

- [ ] **Step 4: Run tests**

```bash
cd terraform/bootstrap
terraform fmt -check -recursive
terraform init -backend=false -input=false
terraform validate
terraform test
```
Expected: `Success! 6 passed, 0 failed.` If `fmt -check` fails, run `terraform fmt -recursive` and re-run.

- [ ] **Step 5: Commit (before any apply)**

```bash
cd ~/projects/dark-factory-aws
git add terraform/bootstrap
git commit -m "feat: bootstrap state bucket and fenced fa-ac-ci role

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

- [ ] **Step 6: Create the GitHub repo and discover the deploy subject**

Ask the owner before running (creates a GitHub repository). Then:

```bash
gh repo create rustembuild/dark-factory-aws --private --source ~/projects/dark-factory-aws --push
gh api -X PUT repos/rustembuild/dark-factory-aws/environments/aws-deploy \
  -F 'deployment_branch_policy[protected_branches]=false' -F 'deployment_branch_policy[custom_branch_policies]=true'
gh api -X POST repos/rustembuild/dark-factory-aws/environments/aws-deploy/deployment-branch-policies -f name=main
```

Create `.github/workflows/whoami.yml`:
```yaml
name: whoami
# Prints this repo's OIDC subject for the aws-deploy environment (not a secret).
on: workflow_dispatch
permissions:
  contents: read
jobs:
  whoami:
    runs-on: ubuntu-24.04
    environment: aws-deploy
    permissions:
      id-token: write
    steps:
      - name: Print OIDC subject
        run: |
          token=$(curl -sSf -H "Authorization: bearer ${ACTIONS_ID_TOKEN_REQUEST_TOKEN}" \
            "${ACTIONS_ID_TOKEN_REQUEST_URL}&audience=sts.amazonaws.com" | jq -r .value)
          python3 -c 'import base64,json,sys; p=sys.argv[1].split(".")[1]; print("sub:", json.loads(base64.urlsafe_b64decode(p+"="*(-len(p)%4)))["sub"])' "$token"
```

```bash
git add .github/workflows/whoami.yml
git commit -m "ci: whoami workflow prints the deploy OIDC subject

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
git push
gh workflow run whoami.yml -R rustembuild/dark-factory-aws
sleep 20; gh run watch -R rustembuild/dark-factory-aws $(gh run list -R rustembuild/dark-factory-aws -w whoami.yml -L1 --json databaseId --jq '.[0].databaseId')
gh run view -R rustembuild/dark-factory-aws --log $(gh run list -R rustembuild/dark-factory-aws -w whoami.yml -L1 --json databaseId --jq '.[0].databaseId') | grep 'sub:'
```
Expected: one line `sub: repo:rustembuild…dark-factory-aws…:environment:aws-deploy`. The owner may have GitHub's immutable-ID subject format enabled (df-agentcore's trust uses `repo:rustembuild@<id>/df-agentcore@<id>:…`); use the printed value **exactly**.

- [ ] **Step 7: Owner applies the bootstrap locally (the only local apply)**

Ask the owner to run (admin credentials, reviewed plan):

```bash
cd ~/projects/dark-factory-aws/terraform/bootstrap
export AWS_PROFILE=<owner's profile> TF_VAR_deploy_subject='<exact sub from Step 6>'
terraform init -input=false
terraform plan -out=bootstrap.plan
```
Expected plan: **only creates**, all under `fa-ac-` names: 1 bucket + 4 bucket sub-resources, 1 policy, 1 role, 1 role policy (8 to add, 0 to change, 0 to destroy). Anything else → STOP.

```bash
terraform apply bootstrap.plan
terraform output
```

Migrate bootstrap state into its own bucket (so it is not only on the Pi):

```bash
cat > backend.tf <<'EOF'
terraform {
  backend "s3" {}
}
EOF
ACC=$(aws sts get-caller-identity --query Account --output text)
terraform init -migrate-state -force-copy -input=false \
  -backend-config=bucket=fa-ac-state-$ACC -backend-config=key=bootstrap.tfstate \
  -backend-config=region=us-east-1 -backend-config=encrypt=true -backend-config=use_lockfile=true
rm -f terraform.tfstate terraform.tfstate.backup bootstrap.plan
```

- [ ] **Step 8: Prove the fence with the IAM policy simulator**

```bash
CI=arn:aws:iam::$ACC:role/fa-ac/fa-ac-ci
sim() { aws iam simulate-principal-policy --policy-source-arn "$CI" --action-names "$1" --resource-arns "$2" \
  ${3:+--context-entries "$3"} --query 'EvaluationResults[0].EvalDecision' --output text; }
sim s3:DeleteBucket "arn:aws:s3:::df-agentcore-state-$ACC" 'ContextKeyName=aws:RequestedRegion,ContextKeyValues=us-east-1,ContextKeyType=string'
sim iam:UpdateAssumeRolePolicy "arn:aws:iam::$ACC:role/GitHubActionsRole"
sim iam:CreateRole "arn:aws:iam::$ACC:role/fa-ac/escape"
sim bedrock-agentcore:CreateAgentRuntime "*" 'ContextKeyName=aws:RequestedRegion,ContextKeyValues=eu-central-1,ContextKeyType=string'
sim iam:DeleteOpenIDConnectProvider "arn:aws:iam::$ACC:oidc-provider/token.actions.githubusercontent.com"
sim s3:PutObject "arn:aws:s3:::fa-ac-state-$ACC/x" 'ContextKeyName=aws:RequestedRegion,ContextKeyValues=us-east-1,ContextKeyType=string'
```
Expected, in order: `explicitDeny`, `explicitDeny`, `explicitDeny` (no boundary in the request), `explicitDeny`, `explicitDeny`, `allowed`. Any other result → STOP and fix the fence (via a reviewed local bootstrap re-apply).

- [ ] **Step 9: Commit the backend file**

```bash
cd ~/projects/dark-factory-aws
git add terraform/bootstrap/backend.tf
git commit -m "chore: bootstrap state lives in fa-ac-state

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
git push
```

---

### Task 7: Foundation Terraform — ECR, results bucket, runtime + invoker roles, budget

**Files:**
- Create: `terraform/foundation/main.tf`, `terraform/foundation/iam.tf`, `terraform/foundation/budget.tf`, `terraform/foundation/tests/foundation.tftest.hcl`

**Interfaces:**
- Consumes: boundary `arn:aws:iam::<account>:policy/fa-ac/fa-ac-boundary`, shared OIDC provider (data).
- Produces (outputs, read by Task 8 via `terraform_remote_state` key `foundation.tfstate`): `ecr_repository_url` (string), `results_bucket` (`fa-ac-legs-<account>`), `execution_role_arn` (`…:role/fa-ac/fa-ac-leg-runtime`), `invoker_role_arn` (`…:role/fa-ac/fa-ac-leg-invoker`). Runtime name contract: `fa_ac_leg_runner` (the invoker policy matches `runtime/fa_ac_leg_runner-*`).

- [ ] **Step 1: Write the tests first** — `terraform/foundation/tests/foundation.tftest.hcl`

```hcl
mock_provider "aws" {
  mock_data "aws_caller_identity" {
    defaults = { account_id = "111111111111" }
  }
  mock_data "aws_iam_openid_connect_provider" {
    defaults = { arn = "arn:aws:iam::111111111111:oidc-provider/token.actions.githubusercontent.com" }
  }
}

variables {
  invoker_subjects = ["repo:rustembuild/flutter_agent_harness_agentcore:environment:agentcore"]
  budget_email     = "owner@example.com"
}

run "invoker_trusts_only_listed_agentcore_environments" {
  command = plan
  assert {
    condition     = jsondecode(aws_iam_role.invoker.assume_role_policy).Statement[0].Condition.StringEquals["token.actions.githubusercontent.com:sub"] == ["repo:rustembuild/flutter_agent_harness_agentcore:environment:agentcore"]
    error_message = "Invoker must trust exactly the listed subjects."
  }
}

run "invoker_can_only_invoke_our_runtime_and_read_results" {
  command = plan
  assert {
    condition = sort(flatten([for s in jsondecode(aws_iam_role_policy.invoker.policy).Statement : s.Action])) == sort(["bedrock-agentcore:InvokeAgentRuntime", "s3:GetObject"])
    error_message = "Invoker may only invoke the runtime and read results."
  }
  assert {
    condition     = contains(flatten([for s in jsondecode(aws_iam_role_policy.invoker.policy).Statement : s.Resource]), "arn:aws:bedrock-agentcore:us-east-1:111111111111:runtime/fa_ac_leg_runner-*")
    error_message = "Invoker must be scoped to the fa_ac_leg_runner runtime."
  }
}

run "execution_role_has_no_secret_access_in_m0" {
  command = plan
  assert {
    condition     = length([for a in flatten([for s in jsondecode(aws_iam_role_policy.execution.policy).Statement : s.Action]) : a if startswith(a, "secretsmanager:")]) == 0
    error_message = "M0 runtime must not read any secret."
  }
  assert {
    condition     = jsondecode(aws_iam_role.execution.assume_role_policy).Statement[0].Condition.StringEquals["aws:SourceAccount"] == "111111111111"
    error_message = "Only AgentCore in this account may assume the execution role."
  }
}

run "every_role_is_fenced" {
  command = plan
  assert {
    condition = alltrue([for r in [aws_iam_role.execution, aws_iam_role.invoker, aws_iam_role.budget_action] :
    r.path == "/fa-ac/" && r.permissions_boundary == "arn:aws:iam::111111111111:policy/fa-ac/fa-ac-boundary"])
    error_message = "Every role must sit under /fa-ac/ with the fa-ac boundary."
  }
}

run "budget_is_20_alert_50_stop_in_us_east_1" {
  command = plan
  assert {
    condition     = aws_budgets_budget.monthly.limit_amount == "50.0" && one([for n in aws_budgets_budget.monthly.notification : n.threshold]) == 20
    error_message = "Budget must alert at 20 and cap at 50."
  }
  assert {
    condition     = one([for f in aws_budgets_budget.monthly.cost_filter : f.values if f.name == "Region"]) == ["us-east-1"]
    error_message = "Budget must count only us-east-1 spend (df-agentcore is eu-central-1)."
  }
  assert {
    condition     = aws_budgets_budget_action.hard_stop.action_threshold[0].action_threshold_value == 50 && aws_budgets_budget_action.hard_stop.approval_model == "AUTOMATIC"
    error_message = "Hard stop must fire automatically at 50."
  }
  assert {
    condition     = sort(aws_budgets_budget_action.hard_stop.definition[0].iam_action_definition[0].roles) == sort(["fa-ac-leg-invoker", "fa-ac-leg-runtime"])
    error_message = "Hard stop must target only this project's roles."
  }
}

run "results_expire_and_are_private" {
  command = plan
  assert {
    condition     = aws_s3_bucket.results.bucket == "fa-ac-legs-111111111111" && aws_s3_bucket_public_access_block.results.restrict_public_buckets
    error_message = "Results bucket must be project-named and private."
  }
  assert {
    condition     = aws_s3_bucket_lifecycle_configuration.results.rule[0].expiration[0].days == 30
    error_message = "LegResults expire after 30 days."
  }
}

run "wildcard_invoker_subject_is_rejected" {
  command         = plan
  variables { invoker_subjects = ["repo:rustembuild/flutter_agent_harness_agentcore:*"] }
  expect_failures = [var.invoker_subjects]
}

run "pull_request_subject_is_rejected" {
  command         = plan
  variables { invoker_subjects = ["repo:rustembuild/flutter_agent_harness_agentcore:pull_request"] }
  expect_failures = [var.invoker_subjects]
}

run "stop_must_exceed_alert" {
  command         = plan
  variables { alert_usd = 50, stop_usd = 20 }
  expect_failures = [var.stop_usd]
}
```

- [ ] **Step 2: Run them to verify they fail**

```bash
cd ~/projects/dark-factory-aws/terraform/foundation
terraform init -backend=false -input=false && terraform test
```
Expected: FAIL — errors about missing resources/variables (no configuration yet).

- [ ] **Step 3: Write** — `terraform/foundation/main.tf`

```hcl
terraform {
  required_version = ">= 1.10.0"
  required_providers {
    aws = { source = "hashicorp/aws", version = "6.65.0" }
  }
  backend "s3" {}
}

provider "aws" {
  region = "us-east-1"
  default_tags { tags = { Project = "fa-agentcore", ManagedBy = "Terraform" } }
}

variable "invoker_subjects" {
  type        = list(string)
  description = "Exact GitHub OIDC subjects allowed to invoke legs (agentcore environments only)."
  validation {
    condition = length(var.invoker_subjects) > 0 && alltrue([
      for s in var.invoker_subjects : !strcontains(s, "*") && endswith(s, ":environment:agentcore")
    ])
    error_message = "invoker_subjects must be exact subjects ending in :environment:agentcore."
  }
}

variable "budget_email" {
  type      = string
  sensitive = true
  validation {
    condition     = can(regex("^[^@\\s]+@[^@\\s]+$", var.budget_email))
    error_message = "budget_email must be an email address."
  }
}

variable "alert_usd" {
  type    = number
  default = 20
}

variable "stop_usd" {
  type    = number
  default = 50
  validation {
    condition     = var.stop_usd > var.alert_usd
    error_message = "stop_usd must be greater than alert_usd."
  }
}

data "aws_caller_identity" "current" {}

data "aws_iam_openid_connect_provider" "github" {
  url = "https://token.actions.githubusercontent.com"
}

locals {
  account        = data.aws_caller_identity.current.account_id
  region         = "us-east-1"
  boundary_arn   = "arn:aws:iam::${local.account}:policy/fa-ac/fa-ac-boundary"
  results_bucket = "fa-ac-legs-${local.account}"
  results_arn    = "arn:aws:s3:::${local.results_bucket}"
  ecr_arn        = "arn:aws:ecr:${local.region}:${local.account}:repository/fa-ac-leg-runner"
  runtime_arns   = "arn:aws:bedrock-agentcore:${local.region}:${local.account}:runtime/fa_ac_leg_runner-*"
  deny_all_arn   = "arn:aws:iam::${local.account}:policy/fa-ac/fa-ac-deny-all"
}

resource "aws_ecr_repository" "leg_runner" {
  name                 = "fa-ac-leg-runner"
  image_tag_mutability = "IMMUTABLE"
  image_scanning_configuration { scan_on_push = true }
}

resource "aws_ecr_lifecycle_policy" "leg_runner" {
  repository = aws_ecr_repository.leg_runner.name
  policy = jsonencode({ rules = [{
    rulePriority = 1, description = "keep the last 10 images",
    selection    = { tagStatus = "any", countType = "imageCountMoreThan", countNumber = 10 },
    action       = { type = "expire" }
  }] })
}

resource "aws_s3_bucket" "results" {
  bucket = local.results_bucket
}

resource "aws_s3_bucket_public_access_block" "results" {
  bucket                  = aws_s3_bucket.results.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_s3_bucket_server_side_encryption_configuration" "results" {
  bucket = aws_s3_bucket.results.id
  rule {
    apply_server_side_encryption_by_default { sse_algorithm = "AES256" }
  }
}

resource "aws_s3_bucket_lifecycle_configuration" "results" {
  bucket = aws_s3_bucket.results.id
  rule {
    id     = "expire-leg-results"
    status = "Enabled"
    filter {}
    expiration { days = 30 }
  }
}

resource "aws_s3_bucket_policy" "results" {
  bucket = aws_s3_bucket.results.id
  policy = jsonencode({ Version = "2012-10-17", Statement = [{
    Effect    = "Deny", Principal = "*", Action = "s3:*",
    Resource  = [local.results_arn, "${local.results_arn}/*"],
    Condition = { Bool = { "aws:SecureTransport" = "false" } }
  }] })
}

output "ecr_repository_url" { value = aws_ecr_repository.leg_runner.repository_url }
output "results_bucket" { value = aws_s3_bucket.results.id }
output "execution_role_arn" { value = aws_iam_role.execution.arn }
output "invoker_role_arn" { value = aws_iam_role.invoker.arn }
```

- [ ] **Step 4: Write** — `terraform/foundation/iam.tf`

```hcl
# Assumed by AgentCore to run the leg-runner image.
resource "aws_iam_role" "execution" {
  name                 = "fa-ac-leg-runtime"
  path                 = "/fa-ac/"
  permissions_boundary = local.boundary_arn
  assume_role_policy = jsonencode({ Version = "2012-10-17", Statement = [{
    Effect    = "Allow", Action = "sts:AssumeRole",
    Principal = { Service = "bedrock-agentcore.amazonaws.com" },
    Condition = { StringEquals = { "aws:SourceAccount" = local.account } }
  }] })
}

# Same runtime permissions df-agentcore proved in production, scoped to fa-ac.
resource "aws_iam_role_policy" "execution" {
  name = "fa-ac-leg-runtime"
  role = aws_iam_role.execution.id
  policy = jsonencode({ Version = "2012-10-17", Statement = [
    { Effect = "Allow", Action = ["ecr:GetAuthorizationToken", "logs:DescribeLogGroups", "xray:PutTraceSegments", "xray:PutTelemetryRecords"], Resource = "*" },
    { Effect = "Allow", Action = ["ecr:BatchGetImage", "ecr:GetDownloadUrlForLayer", "ecr:BatchCheckLayerAvailability"], Resource = local.ecr_arn },
    { Effect = "Allow", Action = ["logs:CreateLogGroup", "logs:CreateLogStream", "logs:DescribeLogStreams", "logs:PutLogEvents"], Resource = "arn:aws:logs:${local.region}:${local.account}:log-group:/aws/bedrock-agentcore/runtimes/*" },
    { Effect = "Allow", Action = ["bedrock-agentcore:AllowVendedLogDeliveryForResource"], Resource = "*" },
    { Effect = "Allow", Action = ["s3:PutObject"], Resource = "${local.results_arn}/legs/*" },
  ] })
}

# Assumed by GitHub workflows in the agentcore environment to start legs.
resource "aws_iam_role" "invoker" {
  name                 = "fa-ac-leg-invoker"
  path                 = "/fa-ac/"
  max_session_duration = 3600
  permissions_boundary = local.boundary_arn
  assume_role_policy = jsonencode({ Version = "2012-10-17", Statement = [{
    Effect    = "Allow"
    Principal = { Federated = data.aws_iam_openid_connect_provider.github.arn }
    Action    = "sts:AssumeRoleWithWebIdentity"
    Condition = { StringEquals = {
      "token.actions.githubusercontent.com:aud" = "sts.amazonaws.com"
      "token.actions.githubusercontent.com:sub" = var.invoker_subjects
    } }
  }] })
}

resource "aws_iam_role_policy" "invoker" {
  name = "fa-ac-leg-invoker"
  role = aws_iam_role.invoker.id
  policy = jsonencode({ Version = "2012-10-17", Statement = [
    { Effect = "Allow", Action = ["bedrock-agentcore:InvokeAgentRuntime"], Resource = [local.runtime_arns] },
    { Effect = "Allow", Action = ["s3:GetObject"], Resource = ["${local.results_arn}/legs/*"] },
  ] })
}
```

- [ ] **Step 5: Write** — `terraform/foundation/budget.tf`

```hcl
# $20 alert / $50 hard stop on us-east-1 spend only (spec §6). Budget data lags
# by hours; the runtime's max_lifetime is the real-time per-session cap.
resource "aws_budgets_budget" "monthly" {
  name         = "fa-ac-monthly"
  budget_type  = "COST"
  limit_amount = format("%.1f", var.stop_usd)
  limit_unit   = "USD"
  time_unit    = "MONTHLY"

  cost_filter {
    name   = "Region"
    values = [local.region]
  }

  notification {
    comparison_operator        = "GREATER_THAN"
    threshold                  = var.alert_usd
    threshold_type             = "ABSOLUTE_VALUE"
    notification_type          = "ACTUAL"
    subscriber_email_addresses = [var.budget_email]
  }
}

resource "aws_iam_policy" "deny_all" {
  name   = "fa-ac-deny-all"
  path   = "/fa-ac/"
  policy = jsonencode({ Version = "2012-10-17", Statement = [{ Effect = "Deny", Action = "*", Resource = "*" }] })
}

resource "aws_iam_role" "budget_action" {
  name                 = "fa-ac-budget-action"
  path                 = "/fa-ac/"
  permissions_boundary = local.boundary_arn
  assume_role_policy = jsonencode({ Version = "2012-10-17", Statement = [{
    Effect = "Allow", Action = "sts:AssumeRole", Principal = { Service = "budgets.amazonaws.com" }
  }] })
}

resource "aws_iam_role_policy" "budget_action" {
  name = "fa-ac-budget-action"
  role = aws_iam_role.budget_action.id
  policy = jsonencode({ Version = "2012-10-17", Statement = [{
    Effect    = "Allow", Action = ["iam:AttachRolePolicy", "iam:DetachRolePolicy"],
    Resource  = [aws_iam_role.execution.arn, aws_iam_role.invoker.arn],
    Condition = { ArnEquals = { "iam:PolicyARN" = local.deny_all_arn } }
  }] })
}

resource "aws_budgets_budget_action" "hard_stop" {
  budget_name        = aws_budgets_budget.monthly.name
  action_type        = "APPLY_IAM_POLICY"
  approval_model     = "AUTOMATIC"
  notification_type  = "ACTUAL"
  execution_role_arn = aws_iam_role.budget_action.arn

  action_threshold {
    action_threshold_type  = "ABSOLUTE_VALUE"
    action_threshold_value = var.stop_usd
  }

  definition {
    iam_action_definition {
      policy_arn = aws_iam_policy.deny_all.arn
      roles      = [aws_iam_role.execution.name, aws_iam_role.invoker.name]
    }
  }

  subscriber {
    address           = var.budget_email
    subscription_type = "EMAIL"
  }

  depends_on = [aws_iam_role_policy.budget_action]
}
```

- [ ] **Step 6: Run tests to verify they pass**

```bash
terraform fmt -recursive && terraform validate && terraform test
```
Expected: `Success! 9 passed, 0 failed.` If an assertion cannot be evaluated at plan because a value is unknown, replace the resource reference in the asserted attribute with the matching deterministic `local.*` (as done for ARNs above) — never weaken the assertion.

- [ ] **Step 7: Commit**

```bash
cd ~/projects/dark-factory-aws
git add terraform/foundation
git commit -m "feat: foundation — ECR, results bucket, fenced runtime/invoker roles, budget hard stop

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 8: Runtime Terraform — the AgentCore runtime

**Files:**
- Create: `terraform/runtime/main.tf`, `terraform/runtime/tests/runtime.tftest.hcl`

**Interfaces:**
- Consumes: foundation outputs `execution_role_arn`, `results_bucket` via `terraform_remote_state` (bucket `var.state_bucket`, key `foundation.tfstate`); `var.image_uri` = `<account>.dkr.ecr.us-east-1.amazonaws.com/fa-ac-leg-runner@sha256:<64 hex>` from Task 9.
- Produces: output `runtime_arn` (`arn:aws:bedrock-agentcore:us-east-1:<account>:runtime/fa_ac_leg_runner-<suffix>`), consumed by Task 10 as a GitHub secret.

- [ ] **Step 1: Write the tests first** — `terraform/runtime/tests/runtime.tftest.hcl`

```hcl
mock_provider "aws" {}

override_data {
  target = data.terraform_remote_state.foundation
  values = {
    outputs = {
      execution_role_arn = "arn:aws:iam::111111111111:role/fa-ac/fa-ac-leg-runtime"
      results_bucket     = "fa-ac-legs-111111111111"
    }
  }
}

variables {
  state_bucket = "fa-ac-state-111111111111"
  image_uri    = "111111111111.dkr.ecr.us-east-1.amazonaws.com/fa-ac-leg-runner@sha256:0000000000000000000000000000000000000000000000000000000000000000"
}

run "runtime_is_public_capped_and_wired" {
  command = plan
  assert {
    condition     = aws_bedrockagentcore_agent_runtime.leg_runner.agent_runtime_name == "fa_ac_leg_runner"
    error_message = "Runtime name must match the invoker policy (fa_ac_leg_runner)."
  }
  assert {
    condition     = aws_bedrockagentcore_agent_runtime.leg_runner.network_configuration[0].network_mode == "PUBLIC"
    error_message = "M0 runtime uses public networking."
  }
  assert {
    condition     = aws_bedrockagentcore_agent_runtime.leg_runner.lifecycle_configuration[0].max_lifetime <= 3600 && aws_bedrockagentcore_agent_runtime.leg_runner.lifecycle_configuration[0].idle_runtime_session_timeout == 900
    error_message = "M0 sessions are capped at 1 hour (budget data lags; this is the real-time cap)."
  }
  assert {
    condition     = aws_bedrockagentcore_agent_runtime.leg_runner.environment_variables["LEG_RESULT_BUCKET"] == "fa-ac-legs-111111111111"
    error_message = "Runtime must write LegResults to the foundation bucket."
  }
  assert {
    condition     = aws_bedrockagentcore_agent_runtime.leg_runner.role_arn == "arn:aws:iam::111111111111:role/fa-ac/fa-ac-leg-runtime"
    error_message = "Runtime must run as the fenced execution role."
  }
}

run "tag_based_image_is_rejected" {
  command         = plan
  variables { image_uri = "111111111111.dkr.ecr.us-east-1.amazonaws.com/fa-ac-leg-runner:latest" }
  expect_failures = [var.image_uri]
}

run "foreign_repository_is_rejected" {
  command         = plan
  variables { image_uri = "111111111111.dkr.ecr.us-east-1.amazonaws.com/darkfactory-dev-codex@sha256:0000000000000000000000000000000000000000000000000000000000000000" }
  expect_failures = [var.image_uri]
}

run "lifetime_above_eight_hours_is_rejected" {
  command         = plan
  variables { max_lifetime_seconds = 28801 }
  expect_failures = [var.max_lifetime_seconds]
}
```

- [ ] **Step 2: Run them to verify they fail**

```bash
cd ~/projects/dark-factory-aws/terraform/runtime
terraform init -backend=false -input=false && terraform test
```
Expected: FAIL — no configuration yet.

- [ ] **Step 3: Write** — `terraform/runtime/main.tf`

```hcl
terraform {
  required_version = ">= 1.10.0"
  required_providers {
    aws = { source = "hashicorp/aws", version = "6.65.0" }
  }
  backend "s3" {}
}

provider "aws" {
  region = "us-east-1"
  default_tags { tags = { Project = "fa-agentcore", ManagedBy = "Terraform" } }
}

variable "state_bucket" {
  type = string
  validation {
    condition     = can(regex("^fa-ac-state-[0-9]{12}$", var.state_bucket))
    error_message = "state_bucket must be fa-ac-state-<account>."
  }
}

variable "image_uri" {
  type        = string
  description = "Leg-runner image pinned by digest."
  validation {
    condition     = can(regex("^[0-9]{12}\\.dkr\\.ecr\\.us-east-1\\.amazonaws\\.com/fa-ac-leg-runner@sha256:[0-9a-f]{64}$", var.image_uri))
    error_message = "image_uri must be the fa-ac-leg-runner repository pinned by sha256 digest."
  }
}

variable "max_lifetime_seconds" {
  type    = number
  default = 3600
  validation {
    condition     = var.max_lifetime_seconds >= 60 && var.max_lifetime_seconds <= 28800
    error_message = "max_lifetime_seconds must be 60..28800 (AgentCore's 8-hour ceiling)."
  }
}

data "terraform_remote_state" "foundation" {
  backend = "s3"
  config = {
    bucket = var.state_bucket
    key    = "foundation.tfstate"
    region = "us-east-1"
  }
}

resource "aws_bedrockagentcore_agent_runtime" "leg_runner" {
  agent_runtime_name = "fa_ac_leg_runner"
  role_arn           = data.terraform_remote_state.foundation.outputs.execution_role_arn
  agent_runtime_artifact {
    container_configuration { container_uri = var.image_uri }
  }
  network_configuration {
    network_mode = "PUBLIC"
  }
  lifecycle_configuration {
    idle_runtime_session_timeout = 900
    max_lifetime                 = var.max_lifetime_seconds
  }
  environment_variables = {
    LEG_RESULT_BUCKET = data.terraform_remote_state.foundation.outputs.results_bucket
    AWS_REGION        = "us-east-1"
  }
}

output "runtime_arn" { value = aws_bedrockagentcore_agent_runtime.leg_runner.agent_runtime_arn }
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
terraform fmt -recursive && terraform validate && terraform test
```
Expected: `Success! 4 passed, 0 failed.`

- [ ] **Step 5: Commit**

```bash
cd ~/projects/dark-factory-aws
git add terraform/runtime
git commit -m "feat: AgentCore runtime for the leg runner, pinned by image digest

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 9: CI — test and deploy workflows, first deploy

**Files:**
- Create: `.github/workflows/test.yml`, `.github/workflows/deploy.yml`

**Interfaces:**
- Consumes: GitHub environment `aws-deploy` (Task 6) with secrets `FA_AC_CI_ROLE_ARN` (= `ci_role_arn`), `BUDGET_EMAIL`, `INVOKER_SUBJECTS_JSON` (JSON list of exact subjects); Terraform roots from Tasks 6–8.
- Produces: deployed foundation + image + runtime; `runtime_arn`, `invoker_role_arn`, `results_bucket` values for Task 10.

- [ ] **Step 1: Resolve action SHAs (pin by commit, record the tag in a comment)**

```bash
for a in actions/checkout@v4 hashicorp/setup-terraform@v3 aws-actions/configure-aws-credentials@v4; do
  echo "$a $(gh api repos/${a%@*}/commits/${a#*@} --jq .sha)"
done
```
Expected: three lines `<action>@<tag> <40-hex sha>`. Substitute each SHA for `<SHA:…>` below.

- [ ] **Step 2: Write** — `.github/workflows/test.yml`

```yaml
name: test
on:
  pull_request:
  push:
    branches: [main]
  workflow_call:
permissions:
  contents: read
jobs:
  dart:
    runs-on: ubuntu-24.04-arm
    steps:
      - uses: actions/checkout@<SHA:actions/checkout@v4> # v4
      - uses: dart-lang/setup-dart@6afc89df92d6eb3834022f73cd65adc8cdfcb92d # v1 (same pin as fa)
        with:
          sdk: stable
      - run: dart pub get
      - run: dart analyze --fatal-infos
      - run: dart test
      - name: Contract test (native arm64)
        run: |
          docker build -t fa-ac-leg-runner:ci .
          scripts/contract-test.sh fa-ac-leg-runner:ci
  terraform:
    runs-on: ubuntu-24.04
    strategy:
      matrix:
        root: [bootstrap, foundation, runtime]
    defaults:
      run:
        working-directory: terraform/${{ matrix.root }}
    steps:
      - uses: actions/checkout@<SHA:actions/checkout@v4> # v4
      - uses: hashicorp/setup-terraform@<SHA:hashicorp/setup-terraform@v3> # v3
        with:
          terraform_version: 1.16.3
          terraform_wrapper: false
      - run: terraform fmt -check -recursive
      - run: terraform init -backend=false -input=false
      - run: terraform validate
      - run: terraform test
```

- [ ] **Step 3: Write** — `.github/workflows/deploy.yml`

```yaml
name: deploy
# The only path that applies foundation/runtime Terraform (spec §6).
on:
  push:
    branches: [main]
  workflow_dispatch:
permissions:
  contents: read
concurrency:
  group: deploy
  cancel-in-progress: false
env:
  AWS_REGION: us-east-1
  TF_IN_AUTOMATION: "true"
jobs:
  test:
    uses: ./.github/workflows/test.yml

  foundation:
    needs: test
    runs-on: ubuntu-24.04
    environment: aws-deploy
    permissions:
      id-token: write
      contents: read
    steps:
      - uses: actions/checkout@<SHA:actions/checkout@v4> # v4
      - uses: hashicorp/setup-terraform@<SHA:hashicorp/setup-terraform@v3> # v3
        with:
          terraform_version: 1.16.3
          terraform_wrapper: false
      - uses: aws-actions/configure-aws-credentials@<SHA:aws-actions/configure-aws-credentials@v4> # v4
        with:
          role-to-assume: ${{ secrets.FA_AC_CI_ROLE_ARN }}
          aws-region: us-east-1
      - name: Apply foundation
        working-directory: terraform/foundation
        env:
          TF_VAR_budget_email: ${{ secrets.BUDGET_EMAIL }}
          TF_VAR_invoker_subjects: ${{ secrets.INVOKER_SUBJECTS_JSON }}
        run: |
          account=$(aws sts get-caller-identity --query Account --output text)
          terraform init -input=false -backend-config=bucket=fa-ac-state-$account \
            -backend-config=key=foundation.tfstate -backend-config=region=us-east-1 \
            -backend-config=encrypt=true -backend-config=use_lockfile=true
          terraform apply -input=false -auto-approve

  image:
    needs: foundation
    runs-on: ubuntu-24.04-arm
    environment: aws-deploy
    permissions:
      id-token: write
      contents: read
    outputs:
      image_uri: ${{ steps.push.outputs.image_uri }}
    steps:
      - uses: actions/checkout@<SHA:actions/checkout@v4> # v4
      - uses: aws-actions/configure-aws-credentials@<SHA:aws-actions/configure-aws-credentials@v4> # v4
        with:
          role-to-assume: ${{ secrets.FA_AC_CI_ROLE_ARN }}
          aws-region: us-east-1
      - name: Build and push (immutable tag = commit)
        id: push
        run: |
          account=$(aws sts get-caller-identity --query Account --output text)
          registry="$account.dkr.ecr.us-east-1.amazonaws.com"
          repo="$registry/fa-ac-leg-runner"
          aws ecr get-login-password | docker login --username AWS --password-stdin "$registry"
          if ! aws ecr describe-images --repository-name fa-ac-leg-runner --image-ids imageTag="$GITHUB_SHA" >/dev/null 2>&1; then
            docker build -t "$repo:$GITHUB_SHA" .
            docker push "$repo:$GITHUB_SHA"
          fi
          digest=$(aws ecr describe-images --repository-name fa-ac-leg-runner --image-ids imageTag="$GITHUB_SHA" \
            --query 'imageDetails[0].imageDigest' --output text)
          echo "image_uri=$repo@$digest" >> "$GITHUB_OUTPUT"

  runtime:
    needs: image
    runs-on: ubuntu-24.04
    environment: aws-deploy
    permissions:
      id-token: write
      contents: read
    steps:
      - uses: actions/checkout@<SHA:actions/checkout@v4> # v4
      - uses: hashicorp/setup-terraform@<SHA:hashicorp/setup-terraform@v3> # v3
        with:
          terraform_version: 1.16.3
          terraform_wrapper: false
      - uses: aws-actions/configure-aws-credentials@<SHA:aws-actions/configure-aws-credentials@v4> # v4
        with:
          role-to-assume: ${{ secrets.FA_AC_CI_ROLE_ARN }}
          aws-region: us-east-1
      - name: Apply runtime
        working-directory: terraform/runtime
        env:
          TF_VAR_image_uri: ${{ needs.image.outputs.image_uri }}
        run: |
          account=$(aws sts get-caller-identity --query Account --output text)
          export TF_VAR_state_bucket=fa-ac-state-$account
          terraform init -input=false -backend-config=bucket=fa-ac-state-$account \
            -backend-config=key=runtime.tfstate -backend-config=region=us-east-1 \
            -backend-config=encrypt=true -backend-config=use_lockfile=true
          terraform apply -input=false -auto-approve
```

Note: `image_uri` contains the account ID; `configure-aws-credentials` masks the account ID in logs by default, and the repo is private.

- [ ] **Step 4: Discover the fa fork's invoker subject**

Create the `agentcore` environment in the fa fork (owner approves; outward-facing), restricted to `main`:

```bash
R=rustembuild/flutter_agent_harness_agentcore
gh api -X PUT repos/$R/environments/agentcore \
  -F 'deployment_branch_policy[protected_branches]=false' -F 'deployment_branch_policy[custom_branch_policies]=true'
gh api -X POST repos/$R/environments/agentcore/deployment-branch-policies -f name=main
```
The smoke workflow cannot run before it is merged, so derive the fork's subject from the format `whoami` printed in Task 6:

```bash
gh api repos/$R --jq '"owner id: \(.owner.id)  repo id: \(.id)"'
```
- default format (`repo:rustembuild/dark-factory-aws:environment:aws-deploy`) → `repo:rustembuild/flutter_agent_harness_agentcore:environment:agentcore`
- immutable-ID format (`repo:rustembuild@<ownerId>/dark-factory-aws@<repoId>:…`) → `repo:rustembuild@<ownerId>/flutter_agent_harness_agentcore@<forkRepoId>:environment:agentcore`

The smoke workflow's first step prints the real subject (Task 10, Step 4); if it differs, fix `INVOKER_SUBJECTS_JSON` and re-deploy.

- [ ] **Step 5: Set deploy secrets on `aws-deploy`** (owner runs; values never echoed)

```bash
D=rustembuild/dark-factory-aws
gh secret set FA_AC_CI_ROLE_ARN -R $D --env aws-deploy --body "$(cd terraform/bootstrap && terraform output -raw ci_role_arn)"
gh secret set BUDGET_EMAIL -R $D --env aws-deploy            # paste the alert email at the prompt
gh secret set INVOKER_SUBJECTS_JSON -R $D --env aws-deploy --body '["<exact fa-fork agentcore subject>"]'
```

- [ ] **Step 6: Commit, push, and watch the first deploy**

```bash
git add .github/workflows/test.yml .github/workflows/deploy.yml
git commit -m "ci: test and deploy workflows (Terraform applied only from CI)

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
git push
gh run watch -R rustembuild/dark-factory-aws $(gh run list -R rustembuild/dark-factory-aws -w deploy.yml -L1 --json databaseId --jq '.[0].databaseId') --exit-status
```
Expected: `test`, `foundation`, `image`, `runtime` all green. On failure, read the job log; an AccessDenied names the missing action — add it to the narrowest place (role policy in foundation, or the fence via a reviewed local bootstrap re-apply) and record it for `docs/m0-findings.md`. Never widen to `*` on `*`.

- [ ] **Step 7: Record runtime outputs locally** (owner's admin profile; read-only)

```bash
ACC=$(aws sts get-caller-identity --query Account --output text)
aws bedrock-agentcore-control list-agent-runtimes --region us-east-1 \
  --query "agentRuntimes[?agentRuntimeName=='fa_ac_leg_runner'].[agentRuntimeArn,status]" --output text
echo "arn:aws:iam::$ACC:role/fa-ac/fa-ac-leg-invoker"
echo "fa-ac-legs-$ACC"
```
Expected: runtime ARN with status `READY`. Keep these three values for Task 10 (not in any repo).

---

### Task 10: Smoke workflow in the fa fork + M0 findings

**Files:**
- Create (fa fork, on a new branch `feat/agentcore-smoke`): `.github/workflows/agentcore-smoke.yml`
- Create (dark-factory-aws): `docs/m0-findings.md`

**Interfaces:**
- Consumes: invoker role, runtime ARN, results bucket (Task 9, Step 7); hello payload `"hello": {"sleepSeconds": n}` (Task 5).
- Produces: measured cold start and idle-survival evidence for the M1/M2 plans.

- [ ] **Step 1: Write the workflow** — `.github/workflows/agentcore-smoke.yml` (fa fork)

No third-party actions: OIDC exchange via curl + AWS CLI, so the fork's action-pin checks have nothing to flag.

```yaml
name: 'AgentCore smoke (M0)'
# Invokes the hello leg in AgentCore and waits for its LegResult.
# Only owner-dispatched, only from main (agentcore environment), one at a time.
on:
  workflow_dispatch:
    inputs:
      sleep_seconds:
        description: 'Seconds the hello leg sleeps (0-1500; >900 proves HealthyBusy keeps the session alive)'
        required: false
        default: '30'
        type: string
permissions:
  contents: read
concurrency:
  group: agentcore-smoke
  cancel-in-progress: false
jobs:
  smoke:
    runs-on: ubuntu-24.04
    environment: agentcore
    timeout-minutes: 40
    permissions:
      id-token: write
      contents: read
    env:
      SLEEP_SECONDS: ${{ inputs.sleep_seconds }}
    steps:
      - name: Print OIDC subject (not a secret)
        run: |
          token=$(curl -sSf -H "Authorization: bearer ${ACTIONS_ID_TOKEN_REQUEST_TOKEN}" \
            "${ACTIONS_ID_TOKEN_REQUEST_URL}&audience=sts.amazonaws.com" | jq -r .value)
          python3 -c 'import base64,json,sys; p=sys.argv[1].split(".")[1]; print("sub:", json.loads(base64.urlsafe_b64decode(p+"="*(-len(p)%4)))["sub"])' "$token"

      - name: Gate — kill switch, actor allowlist, input
        env:
          ENABLED: ${{ vars.AGENTCORE_ENABLED }}
          ALLOWED: ${{ vars.AGENTCORE_ALLOWED_ACTORS }}
          ACTOR: ${{ github.actor }}
        run: |
          if [ "$ENABLED" != "true" ]; then
            echo "::error::AGENTCORE_ENABLED is not 'true' — refusing to start a leg."; exit 1
          fi
          case ",${ALLOWED}," in *",${ACTOR},"*) ;; *)
            echo "::error::${ACTOR} is not in AGENTCORE_ALLOWED_ACTORS."; exit 1;;
          esac
          if ! [[ "$SLEEP_SECONDS" =~ ^[0-9]+$ ]] || (( SLEEP_SECONDS > 1500 )); then
            echo "::error::sleep_seconds must be an integer 0-1500."; exit 1
          fi

      - name: Assume the invoker role (GitHub OIDC)
        env:
          ROLE_ARN: ${{ secrets.AGENTCORE_ROLE_ARN }}
        run: |
          token=$(curl -sSf -H "Authorization: bearer ${ACTIONS_ID_TOKEN_REQUEST_TOKEN}" \
            "${ACTIONS_ID_TOKEN_REQUEST_URL}&audience=sts.amazonaws.com" | jq -r .value)
          creds=$(aws sts assume-role-with-web-identity --role-arn "$ROLE_ARN" \
            --role-session-name "agentcore-smoke-${GITHUB_RUN_ID}" --web-identity-token "$token" \
            --duration-seconds 3600 --query Credentials --output json)
          for k in AccessKeyId SecretAccessKey SessionToken; do echo "::add-mask::$(jq -r .$k <<<"$creds")"; done
          {
            echo "AWS_ACCESS_KEY_ID=$(jq -r .AccessKeyId <<<"$creds")"
            echo "AWS_SECRET_ACCESS_KEY=$(jq -r .SecretAccessKey <<<"$creds")"
            echo "AWS_SESSION_TOKEN=$(jq -r .SessionToken <<<"$creds")"
            echo "AWS_REGION=us-east-1"
          } >> "$GITHUB_ENV"

      - name: Invoke hello leg and wait for its LegResult
        env:
          RUNTIME_ARN: ${{ secrets.AGENTCORE_RUNTIME_ARN }}
          BUCKET: ${{ secrets.AGENTCORE_RESULT_BUCKET }}
        run: |
          sid="github-smoke-run-${GITHUB_RUN_ID}-attempt-${GITHUB_RUN_ATTEMPT}"
          (( ${#sid} >= 33 )) || { echo "::error::session id shorter than 33 chars"; exit 1; }
          max=$(( SLEEP_SECONDS + 300 ))
          jq -n --arg sid "$sid" --argjson sleep "$SLEEP_SECONDS" --argjson max "$max" \
            '{sessionId:$sid, leg:"hello", maxSeconds:$max, callback:{type:"s3"}, hello:{sleepSeconds:$sleep}}' > spec.json
          t0=$(date +%s.%N)
          aws bedrock-agentcore invoke-agent-runtime --agent-runtime-arn "$RUNTIME_ARN" \
            --runtime-session-id "$sid" --content-type application/json \
            --payload fileb://spec.json accept.json > /dev/null
          t1=$(date +%s.%N)
          jq -e '.accepted == true' accept.json > /dev/null || { echo "::error::leg not accepted"; exit 1; }
          echo "accepted after $(python3 -c "print(round($t1-$t0,2))")s (cold start + accept)"
          deadline=$(( $(date +%s) + max + 120 ))
          until aws s3api get-object --bucket "$BUCKET" --key "legs/$sid/result.json" result.json > /dev/null 2>&1; do
            (( $(date +%s) < deadline )) || { echo "::error::no LegResult before deadline"; exit 1; }
            sleep 5
          done
          t2=$(date +%s.%N)
          jq '{status, exitCode, durationSeconds}' result.json
          echo "total $(python3 -c "print(round($t2-$t0,2))")s for sleep ${SLEEP_SECONDS}s"
          [ "$(jq -r .status result.json)" = ok ] || { echo "::error::leg status is not ok"; exit 1; }
```

- [ ] **Step 2: Commit on a branch and open for the owner** (no merge without the owner)

```bash
cd /home/rustem/projects/flutter_agent_harness_agentcore
git switch main && git switch -c feat/agentcore-smoke
git add .github/workflows/agentcore-smoke.yml
git commit -m "ci: AgentCore smoke workflow (M0 hello leg)

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```
Ask the owner before pushing (the fork is public). The workflow must reach `main` to run in the `agentcore` environment; the owner merges it. Note: a push to the fork's `main` also triggers the fork's existing CI workflows (`ci.yml` and others) — expected, unrelated to AgentCore, and they may fail for lack of upstream secrets.

- [ ] **Step 3: Configure the fork** (owner runs; values never echoed)

```bash
R=rustembuild/flutter_agent_harness_agentcore
gh variable set AGENTCORE_ENABLED -R $R --body true
gh variable set AGENTCORE_ALLOWED_ACTORS -R $R --body "agzyamov,rustembuild"
gh secret set AGENTCORE_ROLE_ARN -R $R --env agentcore          # invoker role ARN from Task 9 Step 7
gh secret set AGENTCORE_RUNTIME_ARN -R $R --env agentcore       # runtime ARN
gh secret set AGENTCORE_RESULT_BUCKET -R $R --env agentcore     # fa-ac-legs-<account>
```
Expected: the allowlist contains only the owner's logins — confirm with the owner.

- [ ] **Step 4: Positive run — short leg**

```bash
gh workflow run agentcore-smoke.yml -R $R -f sleep_seconds=30
gh run watch -R $R $(gh run list -R $R -w agentcore-smoke.yml -L1 --json databaseId --jq '.[0].databaseId') --exit-status
```
Expected: green; log shows `accepted after N s`, `"status": "ok"`, `total M s`. If the first step's `sub:` differs from `INVOKER_SUBJECTS_JSON`, update that secret in dark-factory-aws, re-run deploy, then retry.

- [ ] **Step 5: Positive run — idle survival (> 15 min)**

```bash
gh workflow run agentcore-smoke.yml -R $R -f sleep_seconds=1200
```
Expected: green after ~21 minutes, `"status": "ok"` — proves `HealthyBusy` keeps a session alive past the 900 s idle timeout.

- [ ] **Step 6: Negative runs**

```bash
gh variable set AGENTCORE_ENABLED -R $R --body false
gh workflow run agentcore-smoke.yml -R $R -f sleep_seconds=0      # expect: fails at Gate, "refusing to start"
gh variable set AGENTCORE_ENABLED -R $R --body true
git push -u origin feat/agentcore-smoke 2>/dev/null || true       # only if the owner approved pushing
gh workflow run agentcore-smoke.yml -R $R --ref feat/agentcore-smoke -f sleep_seconds=0  # expect: blocked by environment branch policy
```
Expected: run 1 fails at the gate with no AWS call; run 2 never starts the job (`Branch "feat/agentcore-smoke" is not allowed to deploy to agentcore`).

- [ ] **Step 7: df-agentcore unchanged**

Re-run the snapshot command from Task 0 Step 6 into `~/.cache/fa-ac/df-snapshot-after.json`, then:

```bash
diff <(python3 -m json.tool --sort-keys ~/.cache/fa-ac/df-snapshot-before.json) \
     <(python3 -m json.tool --sort-keys ~/.cache/fa-ac/df-snapshot-after.json) && echo "df-agentcore unchanged"
```
Expected: `df-agentcore unchanged`. Any diff → STOP and report to the owner immediately.

- [ ] **Step 8: Write the findings** — `docs/m0-findings.md` (dark-factory-aws)

Fill each line from the runs above (numbers, not adjectives):

```markdown
# M0 findings — AgentCore foundation (YYYY-MM-DD)

| Measure | Result | Source |
|---|---|---|
| Cold start + accept (invoke → `accepted`) | N s (run A), N s (run B) | smoke runs 30 s / 1200 s |
| Total for 30 s leg | N s | smoke run A |
| Idle survival > 900 s with HealthyBusy | pass / fail | smoke run B |
| Image size | N MB | `docker image inspect` |
| OIDC subject format | default / immutable-ID | whoami + smoke first step |
| IAM actions added beyond the plan | none / list | deploy logs |
| Kill switch | blocked at gate, no AWS call | negative run 1 |
| Branch policy | job never started | negative run 2 |
| df-agentcore snapshot diff | empty | Task 10 Step 7 |
| us-east-1 cost after 48 h | $N.NN | Cost Explorer, Region filter |

## Consequences for M1/M2
- (one line per finding that changes the next plan)
```

```bash
cd ~/projects/dark-factory-aws
git add docs/m0-findings.md
git commit -m "docs: M0 findings

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
git push
```
