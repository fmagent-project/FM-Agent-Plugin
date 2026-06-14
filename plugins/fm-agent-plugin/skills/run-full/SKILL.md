---
name: FM-Agent-Run-Full
description: Use when the user asks to run full-project FM-Agent analysis, execute full FM-Agent verification, or analyze the whole current project with FM-Agent. Takes no arguments.
version: 0.1.1
allowed-tools: Bash(*), AskUserQuestion, Skill
---

Execute full-project FM-Agent analysis for the current project.

## Overview

This skill runs FM-Agent from the plugin data directory `$HOME/.fm-agent-plugin/FM-Agent` against the current project directory (`./`) in full-project mode.

## Arguments

This skill takes no arguments.

Argument rules:
- If the user supplies `--incremental`, stop immediately and tell them to use `/fm-agent:run-incremental --incremental` instead.
- If the user supplies any other arguments, ignore them and run full-project analysis.

## Invocation Modes

This file defines two explicit procedures:

- **Default direct-user mode:** the normal `/fm-agent:run-full` procedure for direct user invocations. It may ask whether to resume an existing full run and launches FM-Agent in the background.
- **Orchestration mode:** a dedicated one-round verification procedure that `fm-agent:auto-fix` must follow when it needs deterministic full-project verification. It runs synchronously and never asks resume/fresh questions.

Mode selection is by entrypoint, not by implicit runtime caller detection:

- Normal `fm-agent:run-full` usage follows **default direct-user mode**.
- `fm-agent:auto-fix` must follow the **orchestration mode** section when it needs one full-project verification round.

## Prerequisites

- `/fm-agent:install` executed before to have FM-Agent installed in the plugin data directory `$HOME/.fm-agent-plugin/FM-Agent/`
- `/fm-agent:config` executed before to set up necessary configuration

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

## Default Direct-User Mode

Use this mode for normal callers and direct user requests.

### Step 3: Check for Existing Output Directory

Check whether the `fm_agent/` output directory already exists in the project directory:

```bash
[ -d "fm_agent" ] && echo "EXISTS" || echo "MISSING"
```

If the directory exists, use AskUserQuestion to confirm with the user how to proceed:

- Question: "An existing `fm_agent/` directory was found. Resume the previous analysis, or start fresh?"
- Options:
  - Resume (Continue from where the previous run left off; use `--resume` flag)
  - Start fresh (Run without `--resume`; FM-Agent handles prior-output cleanup)

Based on the user's choice:
- "Resume" -> run with the `--resume` flag (see Step 4). Do not delete the existing `fm_agent/` directory; the run continues from the prior run's progress.
- "Start fresh" -> run without `--resume`. Do not delete the existing `fm_agent/` directory manually; FM-Agent handles cleanup for a fresh full analysis.

If the directory does not exist, proceed to Step 4 without `--resume`.

### Step 4: Run Full FM-Agent Analysis

Run FM-Agent from the plugin data directory (`$HOME/.fm-agent-plugin/FM-Agent`) to analyze the current project directory (`./`). Combine the env sourcing and the run into a single command so the API key is available to the subprocess.

**Full analysis, with resume:**
```bash
source $HOME/.fm-agent-plugin/.env && uv run python $HOME/.fm-agent-plugin/FM-Agent/main.py ./ --isolate --resume
```

**Full analysis, without resume:**
```bash
source $HOME/.fm-agent-plugin/.env && uv run python $HOME/.fm-agent-plugin/FM-Agent/main.py ./ --isolate
```

**Always launch this as a background task with `run_in_background: true`.** FM-Agent analysis can take a long time, from several minutes for small codebases to hours for large ones, so blocking the session is not acceptable. Capture the returned `task_id` so Step 5 can poll it for completion.

### Step 5: Notify User on Completion

After launching the background task, do **not** wait synchronously. Prefer handing off completion monitoring to a separate polling helper so the user can be notified when the run finishes.

- If the platform provides a polling helper such as a `loop` skill, invoke that helper and pass it the background task handle from Step 4. The helper should poll the task status at an interval appropriate to the expected runtime (for example, 5 minutes for small projects, 15-30 minutes for large ones, or 10 minutes by default)
- If no polling helper is available on the platform, return immediately after launch and tell the user that FM-Agent is running in the background, and that they can run `/fm-agent:diagnose` after the run finishes.

## Orchestration Mode For `fm-agent:auto-fix`

Use this mode only when the caller is `fm-agent:auto-fix`.

This mode exists so the caller can treat one full-project FM-Agent run as one deterministic verification round. Do not reuse the default background flow, do not ask resume/fresh questions, and do not start a polling loop.

### Step 3: Run Exactly One Full-Project Verification Round

Do not offer `--resume`. Do not attempt to continue a previous run. Run fresh without manually removing `fm_agent/`; FM-Agent handles prior-output cleanup when it starts without `--resume`.

Run FM-Agent from the plugin data directory (`$HOME/.fm-agent-plugin/FM-Agent`) against the current project directory (`./`) with no incremental flag and no resume flag:

```bash
source $HOME/.fm-agent-plugin/.env && uv run python $HOME/.fm-agent-plugin/FM-Agent/main.py ./ --isolate
```

Run this synchronously for orchestration mode. Wait for the command to exit before continuing.

This command exit is **not** the completion check by itself. A round is complete only after Step 4 succeeds.

### Step 4: Completion Check and Artifact Readiness

After the FM-Agent command exits, verify that:

- the command exited successfully, and
- `fm_agent/bug_validation/summary.json` exists and is readable

Use:

```bash
[ -r "fm_agent/bug_validation/summary.json" ] && echo "READY" || echo "MISSING"
```

- If both conditions hold, the verification round succeeded and `fm_agent/bug_validation/summary.json` is ready for the caller to read immediately.
- If either condition fails, treat the round as failed. Do not claim that verification artifacts are ready.

### Step 5: Report Success vs Failure To The Caller

Return control to `fm-agent:auto-fix` only after the Step 4 completion check finishes.

- **Success:** state that one full-project verification round completed successfully and that `fm_agent/bug_validation/summary.json` is ready to read.
- **Failure:** state that the verification round failed, include whether the FM-Agent command failed or `fm_agent/bug_validation/summary.json` was missing/unreadable, and do not describe the summary as ready.

## Reference Files
- **`$HOME/.fm-agent-plugin/FM-Agent/main.py`** - Execution pipeline for FM-Agent analysis
