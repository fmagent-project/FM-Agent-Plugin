---
name: FM-Agent-Help
description: Use when the user asks to "fm-agent help", "how to use fm-agent", "fm-agent usage", "fm-agent commands", or needs information about FM-Agent plugin capabilities.
version: 0.3.3
---

Provide help and usage information for the FM-Agent plugin.

## Overview

This plugin integrates FM-Agent into Claude Code or codex for automated code reasoning and bug detection. FM-Agent uses LLM-based Hoare-style verification to analyze codebases.

## Available Commands

| Command | Description |
|---------|-------------|
| `/fm-agent:install` | Clone FM-Agent to the plugin data directory |
| `/fm-agent:config` | Show/modify FM-Agent configuration |
| `/fm-agent:run-full` | Execute full-project FM-Agent analysis on current project (background) |
| `/fm-agent:run-incremental` | Execute incremental FM-Agent analysis on current project (background) |
| `/fm-agent:diagnose` | View bug analysis results |
| `/fm-agent:auto-fix` | Run verification-repair loop for the whole codebase |
| `/fm-agent:export` | Export current conversation after a git commit |
| `/fm-agent:help` | Show help information |

## install

Clone FM-Agent from official repository to the plugin data directory (`$HOME/.fm-agent-plugin/FM-Agent/`):
```bash
git clone https://github.com/fmagent-project/FM-Agent.git
```
Then run `$HOME/.fm-agent-plugin/FM-Agent/install.sh` to install dependencies.

## config

Read and modify FM-Agent settings stored in `$HOME/.fm-agent-plugin/FM-Agent/.env` (loaded by FM-Agent via dotenv at runtime).

Workflow:
- If `.env` does not exist, create it with default values
- List the current `.env` values as a table
- Ask whether to modify any settings; if yes, use AskUserQuestion (multi-select) to let the user pick which settings to modify and prompt for each new value
- After writing changes, verify `LLM_API_KEY` is set; if not, loop back to the modification step

**Configurable settings** (all stored in `.env`):
- `LLM_API_KEY` - API key (required for FM-Agent LLM calls)
- `LLM_API_BASE_URL` - API endpoint (default: `https://openrouter.ai/api/v1`)
- `LLM_MODEL` - Default model used by FM-Agent (default: `anthropic/claude-sonnet-4.6`)
- `OPENCODE_MODEL_PROVIDER` - Model provider used by OpenCode (default: `openrouter`)

## run-full

Execute full-project FM-Agent analysis from the plugin data directory against the current project directory (`./`):
- Verify `$HOME/.fm-agent-plugin/FM-Agent/.env` exists and contains the API key (otherwise direct the user to `/fm-agent:config`)
- If `./fm_agent/` already exists, ask the user whether to **resume** (continue the previous run with `--resume`) or **start fresh** (run without `--resume`; FM-Agent handles prior-output cleanup).
- Launch as a background task so the session is not blocked
- Schedule periodic polling via the `loop` skill to detect completion, then notify the user with success or failure.

The skill also exposes an **orchestration mode** used by `/fm-agent:auto-fix` to run a single deterministic full-project verification round synchronously.

## run-incremental

Execute incremental FM-Agent analysis from the plugin data directory against the current project directory (`./`):
- Takes one required argument, `--incremental`, and an optional `<intent-msg>`
- Verifies `$HOME/.fm-agent-plugin/FM-Agent/.env` exists and contains the API key (otherwise direct the user to `/fm-agent:config`)
- Generates the intent file from optional user-provided intent plus export summaries for commits after the last analyzed commit recorded in `fm_agent/version.log` through `HEAD`
- Runs FM-Agent with `--incremental <generated-intent-file>`
- Launches as a background task so the session is not blocked

The skill also exposes an **orchestration mode** used by `/fm-agent:auto-fix` to run a single deterministic incremental verification round synchronously.

## diagnose

Read FM-Agent output from `./fm_agent/`:
- **Summary first**: Show `bug_validation/summary.json` with totals
- **Bug list**: Paginated table (10 per page), sorted with confirmed bugs first
- **Details on request**: Show individual bug reports (`<id>.md`) including specification claim, actual behavior, code evidence, trigger condition, probe script, and probe output

## auto-fix

Run the FM-Agent verification-repair-review loop with `/fm-agent:auto-fix [<intent-msg>]`.

Usage and constraints:
- The loop starts from the current project working tree
- The loop runs one FM-Agent verification round first: `run-full` if `fm_agent/` is missing, or `run-incremental --incremental` if `fm_agent/` and `fm_agent/version.log` exist
- If `<intent-msg>` is provided and incremental verification is selected, auto-fix passes it to `run-incremental --incremental <intent-msg>`
- The skill asks for the maximum number of coding-agent repair rounds before starting session state changes
- If FM-Agent reports bugs, the plugin launches one dedicated coding-agent repair sub-session for the full bug batch
- After each repair round, the plugin launches one dedicated reviewer sub-session to decide whether the reported bugs are fixed
- If the reviewer passes the repair, the loop exits; if the reviewer returns feedback, the next coding-agent round receives that feedback and continues repairing
- Coding-agent repairs stay in the working tree only; the auto-fix loop does not auto-commit repair changes
- Only one auto-fix session may be active per repository at a time

Session artifacts live under `./fm_agent_plugin/`:
- `./fm_agent_plugin/auto-fix-<session-id>.json` - Machine-readable session state
- `./fm_agent_plugin/auto-fix-<session-id>.md` - Human-readable session log
- `./fm_agent_plugin/auto-fix-active.json` - Active-session lock

## export

Export current conversation after a git commit completes. Runs automatically after each `git commit` Bash command — no manual invocation required.

For each commit, the skill writes two files to `./fm_agent_plugin/` in the current project directory:
- `export-<COMMIT_ID>.md` — Full transcript of conversation turns between this commit and the previous one (or session start, if this is the first in-session commit)
- `export-<COMMIT_ID>-summary.md` — Concise summary of the user's intent, decisions, and goal-level description of code changes

After writing the export files, the export skill ends. It does not offer or launch incremental FM-Agent analysis.

## Prerequisites

- Ubuntu 22.04+ or compatible Linux distribution
- Python 3.10+

## Output Directory

Analysis results in `./fm_agent/`:
- `bug_validation/summary.json` - Aggregated summary
- `bug_validation/<id>.md` - Bug detail reports
- `bug_validation/<id>.result.json` - Machine-readable results

## Supported Languages

Rust, C, C++, Python, Java, Go, CUDA, JavaScript, TypeScript, ArkTS
