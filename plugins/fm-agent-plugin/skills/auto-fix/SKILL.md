---
name: FM-Agent Auto-Fix
description: Use when the user asks to run the FM-Agent full-project verification-repair loop for the current project.
version: 0.1.0
allowed-tools: Write,Bash,AskUserQuestion,Skill,Task
---

Run the FM-Agent auto-fix orchestration loop for the current project.

## Overview

This skill owns the verification-repair-review loop for FM-Agent. It enforces that only one auto-fix session is active in the repository, runs FM-Agent in **full-project mode only**, collects bug artifacts from `fm_agent/bug_validation/`, dispatches one dedicated coding-agent sub-session per repair round, dispatches one dedicated reviewer sub-session after each repair round, and records machine-readable and human-readable session state under `./fm_agent_plugin/`.

The reviewer agent's structured return envelope is the control signal for the loop after the initial FM-Agent full-project verification. Do **not** infer the repair outcome from `git diff` or the coding agent's self-report.

## Argument: `<max-iterations>`

This skill requires one argument:

- `<max-iterations>`: the maximum number of coding-agent repair rounds allowed for this session

Manual invocation must provide `<max-iterations>` explicitly. If it is missing, ask the user for the maximum number of repair rounds before starting the loop.

Validate `<max-iterations>` before doing anything else:

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

## Step 2: Run One Full-Project Verification Round

This loop always uses FM-Agent full-project verification. Incremental verification is not allowed in auto-fix mode.

Before verification:

- run this step once at the start of the session
- set `status=verifying`
- write `last_verification_started_at`
- append a verification header to the human-readable log

Invoke `fm-agent:run` as the execution primitive for this step and require it to follow its documented **orchestration mode** for one full-project verification round. Do not use the default direct-user background flow from `fm-agent:run`; the auto-fix orchestrator depends on the run skill's synchronous orchestration-mode completion and readiness contract before continuing.

If the FM-Agent run fails, or if the round completes without producing readable verification artifacts, update the session to:

- `status=failed`
- `stop_reason=verification_failed`

Then clear `./fm_agent_plugin/auto-fix-active.json` if it still points to the current session, append the failure to the human-readable log, and stop.

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

If `summary.json` is missing, unreadable, or malformed, treat that as `verification_failed`.

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

- the agent may modify the current working tree
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
- `bug_count`
- `bugs`
- `coding_agent_summary`
- `changed_files`
- `previous_review_feedback`
- `workspace_rules`

Each item in `bugs` must include the same bug artifact metadata and contents passed to the coding agent.

`workspace_rules` must explicitly state:

- the reviewer must inspect whether the current working tree fully fixes the reported bug batch
- the reviewer may use the original bug artifacts, the coding-agent summary, and the current working tree
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
  - update `status=completed`
  - set `stop_reason=review_passed`
  - append the round summary to `./fm_agent_plugin/auto-fix-<session-id>.md`
  - clear `./fm_agent_plugin/auto-fix-active.json` if it still points to the current session
  - stop

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
