---
name: FM-Agent-Auto-Fix
description: Use when the user asks to run the FM-Agent verification-repair loop for the current project.
version: 0.2.0
allowed-tools: Write,Bash,AskUserQuestion,Skill,Task
---

Run the FM-Agent auto-fix orchestration loop for the current project.

## Overview

This skill owns the verification-repair-review loop for FM-Agent. It enforces that only one auto-fix session is active in the repository, chooses full-project or incremental FM-Agent verification based on prior analysis state, collects bug artifacts from `fm_agent/bug_validation/`, dispatches one dedicated coding-agent sub-session per repair round, dispatches one dedicated reviewer sub-session after each repair round, and records machine-readable and human-readable session state under `./fm_agent_plugin/`.

The reviewer agent's structured return envelope is the control signal for the loop after the initial FM-Agent verification. Do **not** infer the repair outcome from `git diff` or the coding agent's self-report.

All repair and review work happens inside the isolated git worktree snapshot that FM-Agent's `--isolate` run created during verification, never in the real project working tree. Only after the reviewer passes does the loop merge the snapshot's repairs back into the real current branch, auto-resolving any conflicts. The real working tree stays untouched until that final merge.

## Argument: `<intent-msg>` (optional)

This skill takes one optional argument:

- `<intent-msg>`: optional user-provided intent message describing what changed or what should be analyzed

If `<intent-msg>` is provided and the project uses incremental verification, pass it to `fm-agent:run-incremental --incremental <intent-msg>`.

## Repair Round Limit

Before starting the loop, ask the user for the maximum number of coding-agent repair rounds allowed for this session.

Validate the answer before doing anything else:

- It must be a positive integer.
- If it is missing, zero, negative, or not an integer, ask the user for a positive integer before starting any session state changes.
- Record the validated value as `max_iterations`.

## Session Files

All session artifacts live under `./fm_agent_plugin/` in the current project directory. Generate a fresh `session_id` for each new run using the current timestamp, formatted as `YYYYMMDD-HHMMSS`.

- Machine-readable state: `./fm_agent_plugin/auto-fix-<session-id>.json`
- Human-readable log: `./fm_agent_plugin/auto-fix-<session-id>.md`
- Active-session lock: `./fm_agent_plugin/auto-fix-active.json`

The state file must contain at least:

- `session_id`
- `status`
- `iteration`
- `max_iterations`
- `isolated_worktree`
- `last_verification_started_at`
- `last_verification_finished_at`
- `last_bug_count`
- `last_bug_ids`
- `last_repair_outcome`
- `last_review_outcome`
- `last_review_feedback`
- `stop_reason`

Allowed status values:

- `triggered`
- `verifying`
- `repairing`
- `reviewing`
- `completed`
- `failed`

Allowed stop reasons:

- `no_bugs`
- `review_passed`
- `max_iterations`
- `verification_failed`
- `repair_failed`
- `review_failed`
- `merge_conflict`
- `session_conflict`

## Step 1: Enforce a Single Active Session

Only one auto-fix session may be active per repository at a time.

Check `./fm_agent_plugin/auto-fix-active.json` before starting.

If the active-session lock points to a session whose state is still `triggered`, `verifying`, `repairing`, or `reviewing`, refuse to start a second concurrent loop. When refusing to start, report the active `session_id` and the path to `./fm_agent_plugin/auto-fix-active.json`, then stop. Do not create a second state file for the refused run.

If no active session exists:

- generate a fresh `session_id`
- create or overwrite `./fm_agent_plugin/auto-fix-active.json` so it points at the current session
- initialize `./fm_agent_plugin/auto-fix-<session-id>.json` with:
  - `session_id=<session-id>`
  - `status=triggered`
  - `iteration=0`
  - `max_iterations=<validated max>`
  - `last_verification_started_at=null`
  - `last_verification_finished_at=null`
  - `last_bug_count=null`
  - `last_bug_ids=[]`
  - `last_repair_outcome=null`
  - `last_review_outcome=null`
  - `last_review_feedback=null`
  - `stop_reason=null`
- create `./fm_agent_plugin/auto-fix-<session-id>.md` and write the session start record

Do not require a commit id. Do not read or validate a commit message. The loop always starts from the current project working tree.

## Step 2: Run One Verification Round

Before running FM-Agent, check whether the project has been analyzed before:

```bash
[ -d "fm_agent" ] && echo "FM_AGENT_EXISTS" || echo "FM_AGENT_MISSING"
[ -r "fm_agent/version.log" ] && echo "VERSION_LOG_EXISTS" || echo "VERSION_LOG_MISSING"
```

Verification mode selection:

- If `fm_agent/` does not exist, run full-project analysis.
- If `fm_agent/` exists and `fm_agent/version.log` exists and is readable, run incremental analysis.
- If `fm_agent/` exists but `fm_agent/version.log` is missing or unreadable, treat the prior analysis state as incomplete and run full-project analysis.

Before verification:

- run this step once at the start of the session
- set `status=verifying`
- write `last_verification_started_at`
- append a verification header to the human-readable log
- record whether the selected verification mode is `full-project` or `incremental`
- capture the current set of `fm_agent_wt_*` git worktrees as a baseline (`git worktree list --porcelain`), so this round's new snapshot can be identified afterwards

Invoke the split run skills as the execution primitive for this step.

- For full-project analysis, invoke `fm-agent:run-full` and require it to follow its documented **orchestration mode** for one full-project verification round.
- For incremental analysis, invoke `fm-agent:run-incremental --incremental <intent-msg>` when `<intent-msg>` was provided; otherwise invoke `fm-agent:run-incremental --incremental`. The run-incremental skill generates the intent file from any provided intent message plus exported summaries between the last analyzed commit in `fm_agent/version.log` and the current `HEAD`.

Do not use the default direct-user background flow from either run skill; the auto-fix orchestrator depends on the selected run skill's synchronous completion and artifact-readiness contract before continuing.

If the FM-Agent run fails, or if the round completes without producing readable verification artifacts, update the session to:

- `status=failed`
- `stop_reason=verification_failed`

Then clear `./fm_agent_plugin/auto-fix-active.json` if it still points to the current session, append the failure to the human-readable log, and stop.

### Locate the Isolated Worktree

The verification round ran FM-Agent with `--isolate`, which froze the project into a throwaway git worktree snapshot under a `fm_agent_wt_*` temp directory and left it on disk. All repair and review work for this session happens in that snapshot, so identify it now by diffing against the baseline captured before verification:

```bash
git worktree list --porcelain | awk '/^worktree /{print $2}' | grep "/fm_agent_wt_"
```

The `fm_agent_wt_*` worktree present now but absent from the pre-verification baseline is this session's snapshot. If more than one new entry appears, pick the most recently modified. Record its path as `isolated_worktree` in the session state and use it as the workspace for every repair and review round.

If no new `fm_agent_wt_*` worktree can be identified, the isolated run did not produce a reusable snapshot; treat the round as failed:

- `status=failed`
- `stop_reason=verification_failed`

Append the failure to the human-readable log, clear the active-session lock if it still points to the current session, and stop.

## Step 3: Parse `summary.json`

After a successful verification round, read:

```text
fm_agent/bug_validation/summary.json
```

Use `summary.json` only to discover the bug ids present in the current round and the aggregate bug count.

Record:

- `last_verification_finished_at`
- `last_bug_count`
- `last_bug_ids`

If `summary.json` is missing after a successful verification round and `fm_agent/bug_validation/` contains no per-bug artifacts, treat the round as clean:

- record `last_bug_count=0`
- record `last_bug_ids=[]`
- update `status=completed`
- set `stop_reason=no_bugs`
- append the clean result to `./fm_agent_plugin/auto-fix-<session-id>.md`, noting that FM-Agent produced no `summary.json` because no bugs were found
- clear `./fm_agent_plugin/auto-fix-active.json` if it still points to the current session
- stop

If `summary.json` exists but is unreadable or malformed, or if `summary.json` is missing while per-bug artifacts are present, treat that as `verification_failed`.

If `summary.json` reports zero bugs:

- update `status=completed`
- set `stop_reason=no_bugs`
- append the clean result to `./fm_agent_plugin/auto-fix-<session-id>.md`
- clear `./fm_agent_plugin/auto-fix-active.json` if it still points to the current session
- stop

## Step 4: Pair Bug Artifacts by Bug Id

For every bug id listed in `summary.json`, require both files:

- `fm_agent/bug_validation/<id>.md`
- `fm_agent/bug_validation/<id>.result.json`

Pair artifacts strictly by the shared `<id>` basename.

The orchestrator must treat the pair as the per-bug payload:

- `<id>.md` is the human-readable bug report
- `<id>.result.json` is the machine-readable result payload

If any bug id is missing either file, fail the round:

- `status=failed`
- `stop_reason=verification_failed`

Append the missing artifact details to the human-readable log, clear the active-session lock if it still points to the current session, and stop.

## Step 5: Launch the Repair Sub-Session

If at least one bug remains, enforce the repair-round limit before starting a coding-agent sub-session:

- If `iteration >= max_iterations`, do not launch another coding-agent sub-session.
- Update `status=completed`.
- Set `stop_reason=max_iterations`.
- Append the stop decision to `./fm_agent_plugin/auto-fix-<session-id>.md`, including the current `iteration`, `max_iterations`, bug count, and bug ids that remain.
- Clear `./fm_agent_plugin/auto-fix-active.json` if it still points to the current session.
- Stop.

If the limit has not been reached, set `status=repairing` and launch exactly one dedicated coding-agent sub-session for the current round by using the platform's subagent or agent-dispatch capability. This skill therefore requires the task-dispatch tool declared in `allowed-tools`.

Dispatch one coding-agent sub-session per repair round, not one sub-session per bug.

The repair task passed to the coding agent must include:

- `iteration`
- `max_iterations`
- `project_root`
- `isolated_worktree`
- `bug_count`
- `bugs`
- `review_feedback`
- `workspace_rules`

Each item in `bugs` must include:

- `id`
- `source_file`
- `function_name`
- `confirmation_status`
- `trigger_summary`
- `detail_markdown_path`
- `result_json_path`
- the contents of `fm_agent/bug_validation/<id>.md`
- the contents of `fm_agent/bug_validation/<id>.result.json`

`workspace_rules` must explicitly state:

- the agent must perform all edits inside the isolated worktree at `isolated_worktree`, never in the real project working tree
- the agent must address the bug artifacts and, on later rounds, the reviewer feedback
- the agent may decide that a reported bug does not require a code fix, but must explain that decision in `rationale` for reviewer evaluation
- the agent must not create commits
- the agent must not start FM-Agent verification or reviewer evaluation itself
- the agent must return exactly one structured outcome envelope

## Step 6: Require the Structured Coding-Agent Return Envelope

The sub-session must return exactly one envelope in this shape:

```json
{
  "outcome": "completed" | "failed",
  "...": "payload fields depend on outcome"
}
```

Accepted outcome values are:

- `completed`
- `failed`

Required payloads:

- `completed`
  - `summary`
  - `changed_files`
  - `rationale`
- `failed`
  - `summary`
  - `error`

If the sub-session crashes, does not return an envelope, or returns an invalid schema, treat the repair round as failed:

- `status=failed`
- `stop_reason=repair_failed`

Append the contract failure to the human-readable log, clear the active-session lock if it still points to the current session, and stop.

On `failed`:

- set `last_repair_outcome=failed`
- update `status=failed`
- set `stop_reason=repair_failed`
- append the failure summary to the human-readable log
- clear `./fm_agent_plugin/auto-fix-active.json` if it still points to the current session
- stop

On `completed`:

- set `last_repair_outcome=completed`
- increment `iteration` by 1
- record `changed_files`
- append the coding-agent summary to `./fm_agent_plugin/auto-fix-<session-id>.md`
- continue to **Step 7**

## Step 7: Launch the Reviewer Sub-Session

Set `status=reviewing` and launch exactly one dedicated reviewer sub-session for the current repair round by using the platform's subagent or agent-dispatch capability.

The reviewer task must include:

- `iteration`
- `max_iterations`
- `project_root`
- `isolated_worktree`
- `bug_count`
- `bugs`
- `coding_agent_summary`
- `changed_files`
- `previous_review_feedback`
- `workspace_rules`

Each item in `bugs` must include the same bug artifact metadata and contents passed to the coding agent.

`workspace_rules` must explicitly state:

- the reviewer must inspect whether the isolated worktree at `isolated_worktree` fully fixes the reported bug batch
- the reviewer may use the original bug artifacts, the coding-agent summary, and the isolated worktree
- the reviewer must not modify files
- the reviewer must not create commits
- the reviewer must not start FM-Agent verification or another repair round
- the reviewer must return exactly one structured outcome envelope

## Step 8: Require the Structured Reviewer Return Envelope

The reviewer sub-session must return exactly one envelope in this shape:

```json
{
  "outcome": "passed" | "needs_work" | "failed",
  "...": "payload fields depend on outcome"
}
```

Accepted outcome values are:

- `passed`
- `needs_work`
- `failed`

Required payloads:

- `passed`
  - `summary`
  - `evidence`
- `needs_work`
  - `summary`
  - `feedback`
  - `remaining_issues`
- `failed`
  - `summary`
  - `error`

If the reviewer sub-session crashes, does not return an envelope, or returns an invalid schema, treat the review round as failed:

- `status=failed`
- `stop_reason=review_failed`

Append the contract failure to the human-readable log, clear the active-session lock if it still points to the current session, and stop.

## Step 9: Use the Reviewer Envelope as the Only Loop Control Signal

Interpret the post-repair result from the reviewer `outcome` only.

Do **not** decide whether the bug batch is fixed by inspecting `git diff`, changed file timestamps, working tree state alone, or the coding agent's self-report. Those may be logged for observability, but they are not the loop control signal.

After reading the envelope:

- On `passed`:
  - set `last_review_outcome=passed`
  - append the round summary to `./fm_agent_plugin/auto-fix-<session-id>.md`
  - continue to **Step 10** to merge the approved repairs from the isolated worktree back into the real working tree

- On `needs_work`:
  - set `last_review_outcome=needs_work`
  - set `last_review_feedback` to the reviewer's `feedback` and `remaining_issues`
  - append the reviewer feedback to `./fm_agent_plugin/auto-fix-<session-id>.md`
  - loop back to **Step 5** so the next coding-agent sub-session repairs using the reviewer feedback

- On `failed`:
  - set `last_review_outcome=failed`
  - update `status=failed`
  - set `stop_reason=review_failed`
  - append the failure summary to the human-readable log
  - clear `./fm_agent_plugin/auto-fix-active.json` if it still points to the current session
  - stop

## Step 10: Merge the Repaired Snapshot Back

Reached only after the reviewer returns `passed`. Until this point every repair lives in the isolated worktree recorded as `isolated_worktree`, and the real project working tree has not been touched.

Bring the approved repairs into the real current branch. Stage everything in the snapshot except FM-Agent's own output, then apply it onto the real working tree as a three-way merge so any overlap with the current branch surfaces as standard conflict markers instead of a silent overwrite. Repairs are applied to the working tree only; this step does not create commits.

```bash
git -C "<isolated_worktree>" add -A -- . ':!fm_agent/**'
git -C "<isolated_worktree>" diff --cached --binary > "<patch-file>"
git -C "<project_root>" apply --3way --whitespace=nowarn "<patch-file>"
```

- If the apply succeeds with no conflict:
  - update `status=completed`
  - set `stop_reason=review_passed`
  - append the merge result to the human-readable log
  - remove the snapshot: `git -C "<project_root>" worktree remove --force "<isolated_worktree>"`
  - clear `./fm_agent_plugin/auto-fix-active.json` if it still points to the current session
  - stop

- If the apply reports conflicts, auto-resolve them by dispatching exactly one dedicated conflict-resolver sub-session using the platform's agent-dispatch capability. The resolver task must include:
  - `project_root`
  - `isolated_worktree`
  - the list of conflicted files
  - `workspace_rules` stating: the resolver may edit only the conflicted files in the real working tree to reconcile the snapshot repairs with the current branch; it must remove every conflict marker; it must not create commits; it must not start FM-Agent or another repair round; it must return exactly one structured envelope `{ "outcome": "resolved" | "failed", "summary": ..., "...": ... }`.

  Interpret the resolver envelope:

  - On `resolved`:
    - update `status=completed`
    - set `stop_reason=review_passed`
    - append the resolution summary to the human-readable log
    - remove the snapshot worktree
    - clear the active-session lock if it still points to the current session
    - stop
  - On `failed`, or if the resolver crashes or returns an invalid envelope:
    - update `status=failed`
    - set `stop_reason=merge_conflict`
    - leave the conflict markers in the real working tree and report the conflicted file paths to the user
    - keep the snapshot worktree so the user can inspect it, and report its path
    - append the failure to the human-readable log
    - clear the active-session lock if it still points to the current session
    - stop

## Observability Requirements

Each round should append enough detail to `./fm_agent_plugin/auto-fix-<session-id>.md` to explain why the loop continued or stopped. Record:

- `iteration`
- `max_iterations`
- verification start and finish timestamps
- bug count
- bug ids
- artifact paths sent to the coding agent
- coding-agent outcome
- artifact paths and repair summary sent to the reviewer
- reviewer outcome
- reviewer feedback when additional work is required
- working tree change presence after repair, if checked

For the terminal session record, also record the final `stop_reason`.

The machine-readable state file should be updated at each transition so a future run can inspect whether a previous session already completed, failed terminally, or was interrupted.

## Snapshot Lifecycle

The isolated worktree recorded as `isolated_worktree` is reused across all repair rounds of the session. Remove it with `git -C "<project_root>" worktree remove --force "<isolated_worktree>"` whenever the session stops — including the early exits for `no_bugs`, `max_iterations`, `verification_failed`, `repair_failed`, and `review_failed` — except when `stop_reason=merge_conflict` left unresolved markers for the user; in that case keep the snapshot and report its path. Pruning the snapshot on every other terminal state prevents stale `fm_agent_wt_*` worktrees from accumulating and keeps the next session's worktree-baseline discovery unambiguous.