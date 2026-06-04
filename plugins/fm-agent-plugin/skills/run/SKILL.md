---
name: FM-Agent Run
description: Use when the user asks to "run fm-agent", "execute fm-agent", "analyze code with fm-agent", "start reasoning", or wants to run code analysis on the current project. Accepts an optional git commit id argument — when supplied, only the changes introduced by that commit are analyzed (incremental mode).
version: 0.3.0
allowed-tools: Bash(*), AskUserQuestion, Skill
---

Execute FM-Agent to analyze the project codebase for bugs.

## Overview

This skill runs FM-Agent from the plugin data directory `${CLAUDE_PLUGIN_DATA}/FM-Agent` to analyze the current project directory. FM-Agent performs fully automated reasoning using LLM-based Hoare-style verification.

## Argument: `<commit-id>` (optional)

The skill accepts a single optional argument: a git commit id (short or full SHA) intended to scope the analysis to that commit's changes.

- **Not supplied (null):** run full-project analysis.
- **Supplied:** **incremental analysis (`--incremental <commit-id>`) is not yet implemented.** Stop immediately and inform the user that incremental mode is not supported and the run cannot proceed with a commit id. Do not silently fall back to full-project analysis — switching modes without consent would surprise the user. Ask whether they want to run a full-project analysis instead.

## Invocation Modes

This file defines two explicit procedures:

- **Default direct-user mode:** the normal `/fm-agent:run` procedure for direct user invocations, including optional incremental analysis with `<commit-id>`.
- **Orchestration mode:** a dedicated one-round verification procedure that `fm-agent:auto-fix` must follow when it needs deterministic full-project verification.

Mode selection is by entrypoint, not by implicit runtime caller detection:

- Normal `/fm-agent:run` usage follows **default direct-user mode**.
- `fm-agent:auto-fix` must follow the **orchestration mode** section when it needs one full verification round.
- Do **not** infer orchestration mode from the presence or absence of `<commit-id>` alone.

In orchestration mode, ignore the optional `<commit-id>` argument entirely. `fm-agent:auto-fix` always requires full-project verification, never incremental verification.

## Prerequisites

- `fm-agent:install` executed before to have FM-Agent installed in the plugin data directory `${CLAUDE_PLUGIN_DATA}/FM-Agent/`
- `fm-agent:config` executed before to set up necessary configuration

## Default Direct-User Mode

Use this mode for normal callers and direct user requests.

## Execution Steps

### Step 1: Check for API Key

Check whether `${CLAUDE_PLUGIN_DATA}/.env` file exists and contains the API key:

```bash
cat ${CLAUDE_PLUGIN_DATA}/.env
```

If the file or the API key is missing, stop execution and ask the user to run `fm-agent:config` to set up configuration.

### Step 2: Check for Existing Output Directory

Skip this step entirely if a commit id was supplied — incremental runs do not interact with the prior full-run state.

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
- "Resume" → **`--resume` is not yet implemented.** Stop immediately and inform the user that resume is not supported and the previous run cannot be resumed. Ask them whether they want to start fresh instead; do not silently fall back to a fresh run.
- "Start fresh" → remove the existing directory (`rm -rf fm_agent`) and run without `--resume`

If the directory does not exist, proceed to Step 3 without `--resume`.

### Step 3: Run FM-Agent Analysis

Run FM-Agent from the plugin data directory (`${CLAUDE_PLUGIN_DATA}/FM-Agent`) to analyze the current project directory (`./`). Combine the env sourcing and the run into a single command so the API key is available to the subprocess.

Pick the command based on the arguments:

**Full analysis, with resume:**
```bash
source ${CLAUDE_PLUGIN_DATA}/.env && python3 ${CLAUDE_PLUGIN_DATA}/FM-Agent/main.py ./ --resume
```

**Full analysis, without resume:**
```bash
source ${CLAUDE_PLUGIN_DATA}/.env && python3 ${CLAUDE_PLUGIN_DATA}/FM-Agent/main.py ./
```

**Incremental analysis (commit id supplied):**
```bash
source ${CLAUDE_PLUGIN_DATA}/.env && python3 ${CLAUDE_PLUGIN_DATA}/FM-Agent/main.py ./ --incremental <commit-id>
```

Incremental mode is mutually exclusive with `--resume`: when a commit id is supplied, do not also pass `--resume`, even if a previous `fm_agent/` directory exists. Incremental runs are scoped by the commit, not by the prior run's progress.

**Always launch this as a background task with `run_in_background: true`.** FM-Agent analysis can take a long time — from several minutes for small codebases to hours for large ones — so blocking the session is not acceptable. Capture the returned `task_id` so Step 4 can poll it for completion.

### Step 4: Notify User on Completion

After launching the background task, do **not** wait synchronously. Prefer handing off completion monitoring to a separate polling helper so the user can be notified when the run finishes.

- If the platform provides a polling helper such as a `loop` skill, invoke that helper and pass it the background task handle from Step 3. The helper should poll the task status at an interval appropriate to the expected runtime (for example, 5 minutes for small projects, 15-30 minutes for large ones, or 10 minutes by default)
- If no polling helper is available on the platform, return immediately after launch and tell the user that FM-Agent is running in the background, and that they can run `/fm-agent:diagnose` after the run finishes.

This keeps the direct-user mode actionable even on platforms that do not expose a task-status facility inside this skill itself.

## Orchestration Mode For `fm-agent:auto-fix`

Use this mode only when the caller is `fm-agent:auto-fix`.

This mode exists so the caller can treat one FM-Agent run as one deterministic verification round. Do not reuse the default background flow, do not ask resume/fresh questions, and do not start a polling loop.

### Step 1: Check for API Key

Use the same prerequisite check as in direct-user mode:

```bash
cat ${CLAUDE_PLUGIN_DATA}/.env
```

If the file or API key is missing, stop immediately and report failure to the caller.

### Step 2: Reset Prior Full-Run Output

To make the round deterministic, remove any existing `fm_agent/` directory before starting the run.

If `fm_agent/` exists:

```bash
rm -rf fm_agent
```

Do not offer `--resume`. Do not attempt to continue a previous run. The orchestration caller needs one fresh full-project round with artifacts produced by this invocation.

### Step 3: Run Exactly One Full-Project Verification Round

Run FM-Agent from the plugin data directory (`${CLAUDE_PLUGIN_DATA}/FM-Agent`) against the current project directory (`./`) with no incremental flag and no resume flag:

```bash
source ${CLAUDE_PLUGIN_DATA}/.env && python3 ${CLAUDE_PLUGIN_DATA}/FM-Agent/main.py ./
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

This is the round completion check for orchestration mode.

- If both conditions hold, the verification round succeeded and `fm_agent/bug_validation/summary.json` is ready for the caller to read immediately.
- If either condition fails, treat the round as failed. Do not claim that verification artifacts are ready.

### Step 5: Report Success vs Failure To The Caller

Return control to `fm-agent:auto-fix` only after the Step 4 completion check finishes.

The caller learns the result from this skill's final report for the invocation:

- **Success:** state that one full-project verification round completed successfully and that `fm_agent/bug_validation/summary.json` is ready to read.
- **Failure:** state that the verification round failed, include whether the FM-Agent command failed or `fm_agent/bug_validation/summary.json` was missing/unreadable, and do not describe the summary as ready.

For orchestration mode, "success" means:

1. the full-project FM-Agent command finished without error, and
2. `fm_agent/bug_validation/summary.json` passed the readiness check

Anything else is "failure".

## Reference Files
- **`${CLAUDE_PLUGIN_DATA}/FM-Agent/main.py`** - Execution pipeline for FM-Agent analysis
