---
name: FM-Agent-Run
description: Use when the user asks to "run fm-agent", "execute fm-agent", "analyze code with fm-agent", "start reasoning", or wants to run code analysis on the current project. Optionally runs in incremental mode; the intent file may be supplied or generated from exported commit summaries.
version: 0.4.5
allowed-tools: Bash(*), AskUserQuestion, Skill
---

Execute FM-Agent to analyze the project codebase for bugs.

## Overview

This skill runs FM-Agent from the plugin data directory `$HOME/.fm-agent-plugin/FM-Agent` to analyze the current project directory. FM-Agent performs fully automated reasoning using LLM-based Hoare-style verification.

## Arguments (optional, incremental mode)

Incremental mode is selected by the `--incremental` flag. The flag may be used with or without a value:
- `<intent-file>` (`--incremental`): path to a text file describing the goal of the change being analyzed.

Rules:
- **No incremental flag supplied:** run full-project analysis.
- **Incremental flag with intent file supplied:** run incremental analysis with that intent file.
- **Incremental flag supplied without an intent file:** generate the intent file from exported commit summaries before running incremental analysis.
- Incremental mode maps to `--incremental <intent-file>` only.
- If the user supplies extra incremental arguments beyond the intent file, ignore them and run with only the intent file.


## Invocation Modes

This file defines two explicit procedures:

- **Default direct-user mode:** the normal `/fm-agent:run` procedure for direct user invocations, including optional incremental analysis with a supplied or generated intent file.
- **Orchestration mode:** a dedicated one-round verification procedure that `fm-agent:auto-fix` must follow when it needs deterministic verification.

Mode selection is by entrypoint, not by implicit runtime caller detection:

- Normal `fm-agent:run` usage follows **default direct-user mode**.
- `fm-agent:auto-fix` must follow the **orchestration mode** section when it needs one verification round.
- Do **not** infer orchestration mode from the presence or absence of incremental arguments alone.

In orchestration mode, use the optional incremental arguments only when the caller explicitly requests incremental verification. `fm-agent:auto-fix` chooses full-project verification for first analysis and incremental verification when prior analysis state exists.

## Prerequisites

- `/fm-agent:install` executed before to have FM-Agent installed in the plugin data directory `$HOME/.fm-agent-plugin/FM-Agent/`
- `/fm-agent:config` executed before to set up necessary configuration

## Default Direct-User Mode

Use this mode for normal callers and direct user requests.

## Execution Steps

### Step 1: Check for API Key

Check whether `$HOME/.fm-agent-plugin/.env` file exists and contains the API key:

```bash
cat $HOME/.fm-agent-plugin/.env
```

If the file or the API key is missing, stop execution and ask the user to run `/fm-agent:config` to set up configuration.

### Step 2: Commit Pending Code Changes

Before running FM-Agent analysis, make sure all current code changes are committed so the analysis has a stable git baseline.

Check the working tree:

```bash
git status --short
```

If there are uncommitted project changes, commit them before continuing:

```bash
git add -A -- . ':!fm_agent/**' && { git diff --cached --quiet || git commit -m "chore: save changes before fm-agent analysis"; }
```

Rules:
- Include tracked, modified, deleted, and untracked project files in the commit.
- Do not commit the `fm_agent/` output directory created by prior analysis runs.
- If `git status --short` only reports files under `fm_agent/`, skip the commit and continue.
- If the commit fails, stop execution and report the failure; do not run FM-Agent analysis on an uncommitted working tree.
- If there are no uncommitted project changes, continue without creating a commit.

### Step 3: Prepare Incremental Intent File

Skip this step entirely unless incremental mode was requested.

If the user supplied an intent file, use that file as `<intent-file>` and continue to Step 4.

If incremental mode was requested without an intent file, generate one from the export summaries for commits between the last analyzed commit and the current commit.

1. Read the base commit id from the last line of `fm_agent/version.log`:

```bash
tail -n 1 fm_agent/version.log
```

2. Read the current commit id from `HEAD`:

```bash
git rev-parse --short HEAD
```

3. Enumerate commits after the base commit through the current `HEAD`, in chronological order:

```bash
git rev-list --reverse <base-commit>..HEAD
```

4. For each commit from Step 3, resolve its short id and require the matching exported summary file:

```bash
git rev-parse --short <commit>
test -r "fm_agent_plugin/export-<short-commit>-summary.md"
```

5. Concatenate all matching summary files into a generated intent file. Use the short form of the base and current commit ids in the file name:

```bash
mkdir -p fm_agent_plugin
{
  echo "# Incremental FM-Agent Intent"
  echo
  echo "Base analyzed commit: <base-commit>"
  echo "Current commit: <current-commit>"
  echo
  echo "Generated from exported summaries in chronological order."
  echo
  for commit in $(git rev-list --reverse <base-commit>..HEAD); do
    short_commit=$(git rev-parse --short "$commit")
    summary_file="fm_agent_plugin/export-${short_commit}-summary.md"
    echo "## Commit ${short_commit}"
    echo
    cat "$summary_file"
    echo
    echo
  done
} > "fm_agent_plugin/incremental-intent-<base-short-commit>-to-<current-short-commit>.md"
```

Use the generated file as `<intent-file>` for Step 5.

Rules:
- If `fm_agent/version.log` is missing, empty, or unreadable, stop and report that incremental intent generation cannot determine the last analyzed commit.
- If the base commit from `fm_agent/version.log` is not a valid ancestor reference for the current repository, stop and report the invalid base commit.
- If there are no commits between the base commit and `HEAD`, stop and report that there are no exported summaries to combine.
- If any required `fm_agent_plugin/export-<short-commit>-summary.md` file is missing or unreadable, stop and report the missing summary files; do not run FM-Agent with a partial generated intent file.

### Step 4: Check for Existing Output Directory

Skip this step entirely if incremental mode was requested — incremental runs do not interact with the prior full-run state.

Otherwise, check whether the `fm_agent/` output directory already exists in the project directory:

```bash
[ -d "fm_agent" ] && echo "EXISTS" || echo "MISSING"
```

If the directory exists, use AskUserQuestion to confirm with the user how to proceed:

- Question: "An existing `fm_agent/` directory was found. Resume the previous analysis, or start fresh?"
- Options:
  - Resume (Continue from where the previous run left off — use `--resume` flag)
  - Start fresh (Discard existing results and start a new analysis)

Based on the user's choice:
- "Resume" → run with the `--resume` flag (see Step 5). Do not delete the existing `fm_agent/` directory; the run continues from the prior run's progress.
- "Start fresh" → run without `--resume`. Do not delete the existing `fm_agent/` directory manually; FM-Agent handles cleanup for a fresh full analysis.

If the directory does not exist, proceed to Step 5 without `--resume`.

### Step 5: Run FM-Agent Analysis

Run FM-Agent from the plugin data directory (`$HOME/.fm-agent-plugin/FM-Agent`) to analyze the current project directory (`./`). Combine the env sourcing and the run into a single command so the API key is available to the subprocess.

Pick the command based on the arguments:

**Full analysis, with resume:**
```bash
source $HOME/.fm-agent-plugin/.env && uv run python $HOME/.fm-agent-plugin/FM-Agent/main.py ./ --resume
```

**Full analysis, without resume:**
```bash
source $HOME/.fm-agent-plugin/.env && uv run python $HOME/.fm-agent-plugin/FM-Agent/main.py ./
```

**Incremental analysis (intent file supplied):**
```bash
source $HOME/.fm-agent-plugin/.env && uv run python $HOME/.fm-agent-plugin/FM-Agent/main.py ./ --incremental <intent-file>
```

Incremental mode is mutually exclusive with `--resume`: when incremental mode was requested, do not also pass `--resume`, even if a previous `fm_agent/` directory exists.

**Always launch this as a background task with `run_in_background: true`.** FM-Agent analysis can take a long time — from several minutes for small codebases to hours for large ones — so blocking the session is not acceptable. Capture the returned `task_id` so Step 6 can poll it for completion.

### Step 6: Notify User on Completion

After launching the background task, do **not** wait synchronously. Prefer handing off completion monitoring to a separate polling helper so the user can be notified when the run finishes.

- If the platform provides a polling helper such as a `loop` skill, invoke that helper and pass it the background task handle from Step 5. The helper should poll the task status at an interval appropriate to the expected runtime (for example, 5 minutes for small projects, 15-30 minutes for large ones, or 10 minutes by default)
- If no polling helper is available on the platform, return immediately after launch and tell the user that FM-Agent is running in the background, and that they can run `/fm-agent:diagnose` after the run finishes.

This keeps the direct-user mode actionable even on platforms that do not expose a task-status facility inside this skill itself.

## Orchestration Mode For `fm-agent:auto-fix`

Use this mode only when the caller is `fm-agent:auto-fix`.

This mode exists so the caller can treat one FM-Agent run as one deterministic verification round. Do not reuse the default background flow, do not ask resume/fresh questions, and do not start a polling loop.

### Step 1: Check for API Key

Use the same prerequisite check as in direct-user mode:

```bash
cat $HOME/.fm-agent-plugin/.env
```

If the file or API key is missing, stop immediately and report failure to the caller.

### Step 2: Commit Pending Code Changes

Use the same pending-change commit procedure as in direct-user mode before resetting prior output or launching verification.

If committing fails, report failure to the caller and do not run FM-Agent verification.

### Step 3: Prepare Verification Mode

If orchestration mode was invoked with incremental mode, use the same incremental intent-file preparation procedure as direct-user mode:

- If an intent file was supplied, use it.
- If no intent file was supplied, generate it from exported summaries using `fm_agent/version.log` and `HEAD`.

If incremental intent preparation fails, report failure to the caller and do not run FM-Agent verification.

Do not offer `--resume`. Do not attempt to continue a previous run. For full-project verification, run fresh without manually removing `fm_agent/`; FM-Agent handles prior-output cleanup when it starts without `--resume`.

### Step 4: Run Exactly One Verification Round

Run FM-Agent from the plugin data directory (`$HOME/.fm-agent-plugin/FM-Agent`) against the current project directory (`./`).

For full-project verification, run with no incremental flag and no resume flag:

```bash
source $HOME/.fm-agent-plugin/.env && uv run python $HOME/.fm-agent-plugin/FM-Agent/main.py ./
```

For incremental verification, run with the prepared intent file and no resume flag:

```bash
source $HOME/.fm-agent-plugin/.env && uv run python $HOME/.fm-agent-plugin/FM-Agent/main.py ./ --incremental <intent-file>
```

Run this synchronously for orchestration mode. Wait for the command to exit before continuing.

This command exit is **not** the completion check by itself. A round is complete only after Step 5 succeeds.

### Step 5: Completion Check and Artifact Readiness

After the FM-Agent command exits, verify that:

- the command exited successfully, and
- `fm_agent/bug_validation/summary.json` exists and is readable

Use:

```bash
[ -r "fm_agent/bug_validation/summary.json" ] && echo "READY" || echo "MISSING"
```

This is the round completion check for orchestration mode.

- If both conditions hold, the verification round succeeded and `fm_agent/bug_validation/summary.json` is ready for the caller to read immediately.
- If either condition fails, treat the round as failed. Do not claim that verification artifacts are ready.

### Step 6: Report Success vs Failure To The Caller

Return control to `fm-agent:auto-fix` only after the Step 5 completion check finishes.

The caller learns the result from this skill's final report for the invocation:

- **Success:** state that one verification round completed successfully, include whether it was full-project or incremental, and state that `fm_agent/bug_validation/summary.json` is ready to read.
- **Failure:** state that the verification round failed, include whether the FM-Agent command failed or `fm_agent/bug_validation/summary.json` was missing/unreadable, and do not describe the summary as ready.

For orchestration mode, "success" means:

1. the FM-Agent command finished without error, and
2. `fm_agent/bug_validation/summary.json` passed the readiness check

Anything else is "failure".

## Reference Files
- **`$HOME/.fm-agent-plugin/FM-Agent/main.py`** - Execution pipeline for FM-Agent analysis
