# Leg port reference — factory-teammate.yml → AgentCore leg runner

Source of truth for porting IstiN's GitHub Actions leg (`factory-teammate.yml`)
to the AWS-mode leg runner. Line numbers refer to
`IstiN/dmtools-agentic-workflows` `.github/workflows/factory-teammate.yml` at
`ca33362b06053c93e563c6d52d5a298fc2fb8d88` (FT), the consumer stub
`.github/workflows/ai-teammate.yml` in the fa fork (AT),
`IstiN/dmtools-agents` `scripts/run-teammate-local.sh` (RTL) and
`setup/review-verdict.sh` (RV) at release `agents-rel-20261008-215056`.

Reused unchanged (shipped in the `factory-setup-*.zip` asset of the
dmtools-agents release, or in `kit/` of dmtools-agentic-workflows):
`setup/fa-session.sh`, `setup/cache.sh`, `setup/review-verdict.sh` (verdict
decisions), `setup/install-source-git-credentials.sh`,
`kit/git-push-guard.sh`, `kit/install-source-git-credentials.sh`, the agent
packs. Only the YAML glue below is ported.

RTL is Jira-oriented (`--inputJql "key = KEY"`), has no PR anchor, and checks
out the base branch (breaks review/rework legs). Reuse only its ideas: an
`flock` per repo dir and a dirty-tree refusal.

## 1. Inputs

- Exactly one of `issue` / `pr` (FT 154-162), plus `leg` (dev|review|rework
  or empty), `reason`, `machine-author`.
- Consumer gate (AT 95-143): on `issues:assigned` with only `ai-teammate`
  assigned → leg=dev; a human assignee → leg empty ("machine-standby").
  In AWS mode the SM dispatches with explicit `leg`, so the gate is not
  ported.
- Per-anchor serialization (AT concurrency group `ai-teammate-issue-<n|pr-n>`)
  → the DynamoDB item lock of `fa-ac-leg` plus an `flock` in the runner.

## 2. Guard (FT 109-390)

Runner resolution (FT 164-181): `.dmtools/config.js` must exist (else error,
exit 1);
`node -e "c=require('./.dmtools/config.js');r=(c.sm&&c.sm.runners)||{};console.log([r.bug||r.dev||'',r.story||r.dev||'',r.review||'',r.rework||''].join('\t'))"`;
all four empty → error, exit 1.

Decision table, evaluated in order (all exits 0 unless stated):

| # | Condition | Result |
|---|---|---|
| 1 | issue and pr both set | error, exit 1 |
| 2 | neither set | error, exit 1 |
| 3 | PR mode, `gh pr view $PR --json state -q .state` empty | run=false |
| 4 | PR mode, state != OPEN | run=false |
| 5 | PR mode, PR has label `blocked` | run=false |
| 6 | PR mode: leg=INPUT_LEG; if empty and INPUT_REASON matches `rework` (case-insensitive) → leg=rework | continue |
| 7 | PR mode, leg=rework, `rework_override_ok` fails | run=false |
| 8 | PR mode, leg=rework | run=true, runner=REWORK, config=pr_rework.json, kind=rework, anchor=pr-N |
| 9 | PR mode, other leg | run=true, runner=REVIEW, config=pr_review.json, kind=review, anchor=pr-N |
| 10 | Issue mode: anchor=gh-N | |
| 11 | refresh labels and assignees from `gh issue view N --json labels/assignees` (keep payload on error) | |
| 12 | issue state empty | run=false |
| 13 | state != OPEN | run=false |
| 14 | labels contain `agent:skip` (word match) | run=false |
| 15 | labels contain `blocked` | run=false |
| 16 | INPUT_LEG set: rework → REWORK/slot rework; review → REVIEW/slot review; dev → bug/story (§2.1); other → run=false. No override check here | |
| 17 | else labels contain `agent:rework` and not `status:In Rework` | override ok → runner=REWORK, slot EMPTY; not ok → run=false. No fall-through |
| 18 | else labels contain `agent:review` | REVIEW, slot review |
| 19 | else labels match `status:In Development\|status:In Review\|status:In Rework\|in progress\|ai_developed\|ai_pr_reviewed\|_wip\|rework-round-` | run=false |
| 20 | else assignees contain `ai-teammate` or labels contain `agent:dev` | bug/story (§2.1) |
| 21 | runner non-empty | run=true with slot map below |
| 22 | runner empty | run=false |

Slot map (FT 375-381): bug → bug_development.json/dev; story →
story_development.json/dev; review → pr_review.json/review; rework →
pr_rework.json/dev; empty/other → story_development.json/dev (quirk: the
label-derived rework of row 17 records kind dev).

### 2.1 Bug vs story (FT 319-323, 358-362)

Title `gh issue view N --json title -q .title` matches `^\[?bug\]`
(case-insensitive), or labels contain word `bug` → RUNNER_BUG/slot bug; else
RUNNER_STORY/slot story.

### 2.2 rework_override_ok (FT 192-204)

OK when ACTOR != `labeled` or SENDER empty. Else
`gh api repos/R/collaborators/$SENDER/permission -q .permission`; accepted
`admin|maintain|write`; otherwise warn and return 1.

### 2.3 machineAuthor

Input `machine-author` (consumer `vars.MACHINE_AUTHOR`), injected as
`dmtools run <runner> '{"params":{"jobParams":{"machineAuthor":"<MA>"}}}'`.
Verdict-step fallback: `sed -nE` on `.dmtools/config.js` `machineAuthor`.

## 3. Teammate steps (FT 392-1633), ported vs dropped

| FT lines | Step | Port |
|---|---|---|
| 412-422 | checkout | clone the repo (App token) |
| 433-504 | fetch factory-setup zip | baked into the image at `/opt/factory-setup` (pinned tag) → `FACTORY_ROOT` |
| 506-538 | fetch kit | baked at `/opt/factory-kit/kit` (pinned SHA) |
| 540-577 | resolve parent config | newest `~/.dmtools/packs/<agent>-*/launch.json`, else `$FACTORY_ROOT/<agent>.json`, else `$FACTORY_ROOT/configs/<agent>.json`; export `AI_TEAMMATE_CONFIG_FILE` |
| 579-585 | session identity | `AI_TEAMMATE_DISPLAY_KEY=GH-<anchor>`, `AI_TEAMMATE_CONCURRENCY_KEY=<anchor>`, source `$FACTORY_ROOT/setup/fa-session.sh env` (exports `FA_SESSION_NAME`, `FA_SESSION_ROOT`) |
| 587-602 | restore session | `git fetch --depth 1 origin refs/heads/fa-sess/<anchor>` → `git checkout refs/remotes/origin/fa-sess/<anchor> -- .dmtools/fa-sessions; git restore --staged .dmtools/fa-sessions` |
| 604-689 | install fa / dmtools | baked into the image |
| 719-726 | git creds | `SOURCE_GITHUB_TOKEN=<App token> bash /opt/factory-kit/kit/install-source-git-credentials.sh` |
| 798-815 | `.git/info/exclude` | append `.codegraph/ .dmtools/fa-trace.log .dmtools/run-output.txt .dmtools/stall-capture.log .dmtools/fa-sessions/ .dmtools/copilot-sessions/ .fah/bash_jobs/ factory-agents/` |
| 817-834 | issue input | `mkdir -p input/gh-N`; `gh issue view N --json number,title,body,url,state,labels -q '{key:("gh-"+(.number|tostring)),number:.number,title:.title,body:.body,url:.url,state:.state,labels:[.labels[].name]}' > input/gh-N/ticket.json`; `gh issue view N --json title,body -q '.title, .body' > input/ticket.md` |
| 836-854 | PR input | same with `gh pr view`, key `pr-N`, dir `input/pr-N/` |
| 856-886 | start marker (issue only) | §4.1 |
| 888-1059 | run agent | §3.1 |
| 1061-1165 | token publish | §4.3 (always) |
| 1167-1204 | done marker (issue only, always) | §4.1 |
| 1206-1420 | verdict (kind review, issue only) | §5 |
| 1422-1432 | close review cycle (always, review, issue) | `gh issue edit N --remove-label agent:review \|\| true` |
| 1434-1452 | close rework cycle (always, runner path contains `rework`) | PR: `gh pr edit PR --remove-label agent:rework \|\| true`; else issue |
| 1454-1480 | persist memory (always) | if `.fah/memory` changed: add, commit `chore(fa): persist agent project memory [skip ci]`, push (warn on failure) |
| 1482-1538 | quarantine (agent failed/cancelled) | §4.4 |
| 1555-1569 | upload trace | dropped; result.json carries the S3 log key |
| 1571-1605 | persist session (always) | §4.4 |
| 1607-1622 | reviewer handoff (kind dev, issue, also after failure) | `--remove-label agent:rework \|\| true`; `--remove-label agent:review \|\| true`; `--add-label agent:review` |
| 1624-1633 | run link (always, issue) | `gh issue comment N --body "🤖 AI Teammate run ${OUTCOME}: ${CI_RUN_URL}"` |

### 3.1 Run agent (FT 888-1059)

- Env: `AI_AGENT_PROVIDER=fa`, `AI_TEAMMATE_*`, `ZAI_CODE_KEY`,
  `KIMI_REVIEW_KEY`, `SOURCE_GITHUB_TOKEN`, `GH_TOKEN`,
  `DMTOOLS_PACK_REGISTRY=https://github.com/IstiN/dmtools-agents/releases/download/<tag>`,
  `CI_RUN_URL`.
- Push guard: `ln -sf /opt/factory-kit/kit/git-push-guard.sh $TMP/guard-bin/git`,
  `PATH=$TMP/guard-bin:$PATH` for the agent only.
- `FA_LOG_FILE=$WORK/.dmtools/fa-trace.log` (touched).
- Watchdog: every 60 s, if `FA_LOG_FILE` mtime age ≥ `FA_TRACE_STALE_MINUTES`
  (10) min → append `FA-TRACE-STALE-WATCHDOG` to run-output, `pkill -f "dmtools run"; pkill -f "fa --session"`.
- Command:
  `dmtools run "$RUNNER" [jobParams JSON] --metadata '{"contextId":"<anchor>"}' --ciRunUrl "$CI_RUN_URL" --debug 2>&1 | tee .dmtools/run-output.txt`; exit = `PIPESTATUS[0]`.

## 4. Markers, tokens, sessions

### 4.1 Job markers (issue body)

Start: `leg="${KIND:-dev} ($(basename "${RUNNER:-agent}" .json))"`;
`line="- [ ] ▶ ${leg} · [run](${RUN_URL}) · started $(date -u '+%H:%M') ⏳"`.
Body via `gh issue view N --json body -q .body`, normalised with
`printf '%s\n' "$(cat f)" > f`; append under an existing `^## Machine jobs`
line or add `\n## Machine jobs\n<line>\n`; `gh issue edit N --body-file f`.

Done: mark success ✅ / failure ❌ / cancelled 🚫 / other `⚠️ <status>`.
The line containing `(RUN_URL)` and matching `^- \[ \]` becomes `- [x] ` and
`started HH:MM ...` becomes `started HH:MM · done HH:MM <mark>`. Edit only
if changed. RUN_URL must be unique per run (AWS mode: the execution console
URL).

### 4.3 Token publish (never fails the leg)

Skip if no `fa-tokens:` line. Sum `.input`/`.output` of every
`fa-tokens:\s*{json}`. LEG = `rework` if the config basename contains
`rework`, else KIND. KEY: `pr-N` stays, `gh-N` → `issue-N`. Row
`{"leg","at","prompt","completion","total"}`. Ensure branch `factory-data`
from the default branch; up to 3 attempts: GET
`repos/R/contents/data/fa-tokens.json?ref=factory-data`, update with
`.[$key] = ((.[$key] // []) + [$row]) | with_entries(.value |= (map(select((.at // "") >= $cutoff)) | .[-40:])) | with_entries(select(.value | length > 0))`
(cutoff 14 days), PUT with `branch=factory-data`,
`message="leg tokens — KEY LEG [skip ci]"`.

### 4.4 Sessions

Persist (always): skip if `.dmtools/fa-sessions` empty;
`git config user.name "dm.ai"; git config user.email "dm.ai@epam.com"`;
snapshot to a temp dir; `git switch --orphan fa-sess/<anchor> || git switch -C fa-sess/<anchor>`;
replace `.dmtools/fa-sessions` with the snapshot; `git add -f`; commit
`fa sessions: GH-<anchor> (<config>) [skip ci]` unless nothing staged;
`git push origin HEAD:fa-sess/<anchor> --force` (warn on failure). Runs last
(it leaves the tree on the orphan branch).

Quarantine (agent failed): if the trace file exists and its age ≥
`FA_TRACE_QUARANTINE_SILENCE_MINUTES` (5) min → move
`.dmtools/fa-sessions` to `.dmtools/fa-sessions-quarantine/run-<id>`, append
`FA-SESSION-QUARANTINE` to run-output, `git push origin --delete fa-sess/<anchor>`.

## 5. Review verdict (FT 1206-1420, issue-anchored review only)

Inputs: fresh labels; linked open PR = first of `gh pr list --state open
--json number,body,headRefName` whose body matches
`(?i)\b(closes|fixes|resolves)\s+#N\b` or branch matches `^N[-_.]` or
`(^|[-_./])gh-0*N$`; PR comments+reviews and issue comments as
`[{author,body}]` files.

Decide: `eval "$(MAX_ROUNDS=$MAX PR_REVIEW_JSON=outputs/pr_review.json PR_REVIEW_JSON_ALT=outputs/gh-N/pr_review.json RUN_OUTPUT=.dmtools/run-output.txt PR_COMMENTS_JSON=f1 ISSUE_COMMENTS_JSON=f2 MACHINE_AUTHOR=$MA ISSUE_LABELS="$labels" bash $VERDICT_SH decide)"`
with `VERDICT_SH` = newest `~/.dmtools/packs/pr_review-*/loop/verdict.sh`,
else `$FACTORY_ROOT/setup/review-verdict.sh`. Emits `decision`
(approve|rework|unknown), `escalate`, `next_round`, `rounds_done`,
`round_labels`, `diagnosis`.

| Case | Actions |
|---|---|
| rework, escalate | strip round labels; ensure+add `needs-human` (B60205); comment "🛑 **Auto-rework loop capped** …" with unresolved review threads (GraphQL reviewThreads → `verdict.sh threads`); exit 0 |
| rework | `--remove-label needs-human`; strip round labels; `--add-label agent:rework`; ensure `rework-round-<next>` (BFD4F2); add it; exit 0 |
| unknown | ensure+add `needs-human`; comment "🛑 **Review verdict missing** …" with diagnosis; exit 1 |
| approve, no PR | warn; exit 0 |
| approve, PR | poll `statusCheckRollup` every 30 s up to 600 s; any of FAILURE/ERROR/CANCELLED/TIMED_OUT/ACTION_REQUIRED/STARTUP_FAILURE/STALE → warn, exit 0; all SUCCESS/SKIPPED/NEUTRAL → strip round labels, `--remove-label needs-human`, `--add-label pr_approved`; still pending → warn, exit 0 |

## 6. Tools

bash, git, gh, jq, node (config read), python3 (token sums), awk, sed, curl,
unzip, tar, base64, stat, pkill (procps), flock (util-linux), openssl (App
JWT), aws CLI, dmtools, fa.
