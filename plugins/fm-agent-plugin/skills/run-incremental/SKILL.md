---
name: FM-Agent-Run-Incremental
description: Use when the user asks to run incremental FM-Agent analysis or when auto-fix needs incremental verification for a project with prior analysis state. Takes --incremental and an optional intent message.
version: 0.2.0
allowed-tools: Bash(*), Skill
---

Execute incremental FM-Agent analysis for the current project.

## Overview

This skill runs FM-Agent from the plugin data directory `$HOME/.fm-agent-plugin/FM-Agent` against the current project directory (`./`) in incremental mode.

## Arguments

This skill takes one required argument and one optional argument:

- `--incremental`: run incremental analysis
- `<intent-msg>`: optional user-provided intent message describing what changed or what should be analyzed

The skill always generates the intent file from exported summary files written by `fm-agent:export`. If `<intent-msg>` is provided, include it at the beginning of the generated intent file.

Rules:
- If `--incremental` is not supplied, stop and tell the user to run `/fm-agent:run-incremental --incremental`.
- If `<intent-msg>` is supplied, preserve it as user-provided intent and place it before the exported summaries in the generated intent file.
- If multiple extra tokens are supplied after `--incremental`, treat all of them together as `<intent-msg>`.

## Invocation Modes

This file defines two explicit procedures:

- **Default direct-user mode:** the normal `/fm-agent:run-incremental --incremental [<intent-msg>]` procedure for direct user invocations. It generates an intent file and launches FM-Agent in the background.
- **Orchestration mode:** a dedicated one-round verification procedure that `fm-agent:auto-fix` must follow when it needs deterministic incremental verification. It runs synchronously.

Mode selection is by entrypoint, not by implicit runtime caller detection:

- Normal `fm-agent:run-incremental` usage follows **default direct-user mode**.
- `fm-agent:auto-fix` must follow the **orchestration mode** section when it needs one incremental verification round.

## Prerequisites

- `/fm-agent:install` executed before to have FM-Agent installed in the plugin data directory `$HOME/.fm-agent-plugin/FM-Agent/`
- `/fm-agent:config` executed before to set up necessary configuration
- `fm_agent/version.log` exists from a prior FM-Agent analysis
- `fm_agent_plugin/export-<short-commit>-summary.md` files exist for every commit after the last analyzed commit through `HEAD`

## Shared Setup

Use these setup steps before either invocation mode runs FM-Agent.

### Step 1: Check for API Key

Check whether `$HOME/.fm-agent-plugin/.env` exists and contains the API key:

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

### Step 3: Generate Incremental Intent File

Generate the intent file from any optional user-provided intent message and the export summaries for commits between the last analyzed commit and the current commit.

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

5. Concatenate the optional `<intent-msg>` and all matching summary files into a generated intent file. Use the short form of the base and current commit ids in the file name:

```bash
mkdir -p fm_agent_plugin
{
  echo "# Incremental FM-Agent Intent"
  echo
  echo "Base analyzed commit: <base-commit>"
  echo "Current commit: <current-commit>"
  echo
  if [ -n "<intent-msg>" ]; then
    echo "## User-provided intent"
    echo
    printf '%s\n' "<intent-msg>"
    echo
  fi
  echo "## Exported commit summaries"
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

Use the generated file as `<intent-file>` when running FM-Agent.

Rules:
- If `fm_agent/version.log` is missing, empty, or unreadable, stop and report that incremental intent generation cannot determine the last analyzed commit.
- If the base commit from `fm_agent/version.log` is not a valid ancestor reference for the current repository, stop and report the invalid base commit.
- If there are no commits between the base commit and `HEAD`, stop and report that there are no exported summaries to combine.
- If any required `fm_agent_plugin/export-<short-commit>-summary.md` file is missing or unreadable, stop and report the missing summary files; do not run FM-Agent with a partial generated intent file.

## Default Direct-User Mode

Use this mode for normal callers and direct user requests.

### Step 4: Run Incremental FM-Agent Analysis

Run FM-Agent from the plugin data directory (`$HOME/.fm-agent-plugin/FM-Agent`) to analyze the current project directory (`./`). Combine the env sourcing and the run into a single command so the API key is available to the subprocess.

```bash
source $HOME/.fm-agent-plugin/.env && uv run python $HOME/.fm-agent-plugin/FM-Agent/main.py ./ --incremental <intent-file>
```

Do not pass `--resume` with incremental analysis.

**Always launch this as a background task with `run_in_background: true`.** FM-Agent analysis can take a long time, from several minutes for small codebases to hours for large ones, so blocking the session is not acceptable. Capture the returned `task_id` so Step 5 can poll it for completion.

### Step 5: Notify User on Completion

After launching the background task, do **not** wait synchronously. Prefer handing off completion monitoring to a separate polling helper so the user can be notified when the run finishes.

- If the platform provides a polling helper such as a `loop` skill, invoke that helper and pass it the background task handle from Step 4. The helper should poll the task status at an interval appropriate to the expected runtime (for example, 5 minutes for small projects, 15-30 minutes for large ones, or 10 minutes by default)
- If no polling helper is available on the platform, return immediately after launch and tell the user that FM-Agent incremental analysis is running in the background, and that they can run `/fm-agent:diagnose` after the run finishes.

## Orchestration Mode For `fm-agent:auto-fix`

Use this mode only when the caller is `fm-agent:auto-fix`.

This mode exists so the caller can treat one incremental FM-Agent run as one deterministic verification round. Do not reuse the default background flow and do not start a polling loop.

### Step 4: Run Exactly One Incremental Verification Round

Run FM-Agent from the plugin data directory (`$HOME/.fm-agent-plugin/FM-Agent`) against the current project directory (`./`) with the generated intent file and no resume flag:

```bash
source $HOME/.fm-agent-plugin/.env && uv run python $HOME/.fm-agent-plugin/FM-Agent/main.py ./ --incremental <intent-file>
```

Run this synchronously for orchestration mode. Wait for the command to exit before continuing.

This command exit is **not** the completion check by itself. A round is complete only after Step 5 succeeds.

### Step 5: Completion Check and Artifact Readiness

After the FM-Agent command exits, verify that:

- the command exited successfully, and
- either `fm_agent/bug_validation/summary.json` exists and is readable, or `fm_agent/bug_validation/` contains no per-bug artifacts because FM-Agent found no bugs

Use:

```bash
if [ -r "fm_agent/bug_validation/summary.json" ]; then
  echo "READY"
elif find fm_agent/bug_validation -maxdepth 1 \( -name "*.result.json" -o -name "*.md" \) -print -quit 2>/dev/null | grep -q .; then
  echo "MISSING"
else
  echo "NO_BUGS_NO_SUMMARY"
fi
```

- If the command succeeded and the check prints `READY`, the incremental verification round succeeded and `fm_agent/bug_validation/summary.json` is ready for the caller to read immediately.
- If the command succeeded and the check prints `NO_BUGS_NO_SUMMARY`, the incremental verification round succeeded with no reported bugs. Tell the caller that no `summary.json` was produced because FM-Agent found no bugs.
- If either condition fails, treat the round as failed. Do not claim that verification artifacts are ready.

### Step 6: Report Success vs Failure To The Caller

Return control to `fm-agent:auto-fix` only after the Step 5 completion check finishes.

- **Success:** state that one incremental verification round completed successfully. If the readiness check printed `READY`, say that `fm_agent/bug_validation/summary.json` is ready to read. If it printed `NO_BUGS_NO_SUMMARY`, say that no bugs were found and no `summary.json` was produced.
- **Failure:** state that the incremental verification round failed, include whether the FM-Agent command failed or `fm_agent/bug_validation/summary.json` was missing/unreadable, and do not describe the summary as ready.

## Reference Files
- **`$HOME/.fm-agent-plugin/FM-Agent/main.py`** - Execution pipeline for FM-Agent analysis
