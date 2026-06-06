# FM-Agent Plugin

This plugin integrates [FM-Agent](https://github.com/fmagent-project/FM-Agent) into Claude Code or codex for automated code reasoning and bug repair.

## Prerequisites

- Ubuntu 22.04+ or compatible Linux distribution
- Python 3.10+

## How to install

For Claude Code:

```
claude
/plugin marketplace add fmagent-project/FM-Agent-Plugin
/plugin install fm-agent-plugin@fm-agent-plugin
```

For codex:
```
codex plugin marketplace add fmagent-project/FM-Agent-Plugin
codex plugin add fm-agent-plugin@fm-agent-plugin
```

**Important Note**:  
FM-Agent analysis needs to write opencode db file `~/.local/share/opencode/opencode.db`. However, by default, codex can only read and edit files in the current workspace. To use FM-Agent properly, set codex permissions to Full Access.
```
/permissions
[Choose: Full Access]
```

## Available Commands

| Command | Description |
|---------|-------------|
| `/fm-agent:install` | Clone FM-Agent to the plugin data directory |
| `/fm-agent:config` | Show/modify FM-Agent configuration |
| `/fm-agent:auto-fix` | Run FM-Agent verification-repair loop for the whole codebase |
| `/fm-agent:run` | Execute FM-Agent analysis on current project without repairing bugs (background) |
| `/fm-agent:diagnose` | View bug analysis results |
| `/fm-agent:help` | Show help information |

### install

Clone FM-Agent from official repository to the plugin data directory (`$HOME/.fm-agent-plugin/FM-Agent/`):
```bash
git clone https://github.com/fmagent-project/FM-Agent.git
```
Then run `$HOME/.fm-agent-plugin/FM-Agent/install.sh` to install dependencies.

### config

Read and modify FM-Agent settings stored in `$HOME/.fm-agent-plugin/.env` (sourced at runtime via `source .env`).

Workflow:
- If `.env` does not exist, create it with default values
- List the current `.env` values as a table
- Ask whether to modify any settings; if yes, use AskUserQuestion (multi-select) to let the user pick which settings to modify and prompt for each new value
- After writing changes, verify `LLM_API_KEY` is set; if not, loop back to the modification step

**Configurable settings** (all stored in `.env`):
- `LLM_API_KEY` - OpenRouter API key (required for FM-Agent LLM calls)
- `LLM_API_BASE_URL` - OpenRouter API endpoint (default: `https://openrouter.ai/api/v1`)
- `LLM_MODEL` - Default model used by FM-Agent (default: `anthropic/claude-sonnet-4.6`)
- `OPENCODE_MODEL_PROVIDER` - Model provider used by OpenCode (default: `openrouter`)

### auto-fix

Run the FM-Agent verification-repair-review loop with `/fm-agent:auto-fix <max-iterations>`.

Usage and constraints:
- The loop starts from the current project working tree
- The loop runs one full-project FM-Agent verification round first; incremental verification is not allowed in auto-fix mode
- If FM-Agent reports bugs, the plugin launches one dedicated coding-agent repair sub-session for the full bug batch
- After each repair round, the plugin launches one dedicated reviewer sub-session to decide whether the reported bugs are fixed
- If the reviewer passes the repair, the loop exits; if the reviewer returns feedback, the next coding-agent round receives that feedback and continues repairing
- Coding-agent repairs stay in the working tree only; the auto-fix loop does not auto-commit repair changes
- Only one auto-fix session may be active per repository at a time

Session artifacts live under `./fm_agent_plugin/`:
- `./fm_agent_plugin/auto-fix-<session-id>.json` - Machine-readable session state
- `./fm_agent_plugin/auto-fix-<session-id>.md` - Human-readable session log
- `./fm_agent_plugin/auto-fix-active.json` - Active-session lock

### run

Execute FM-Agent from the plugin data directory to analyze the current project directory (`./`):
- Verify `$HOME/.fm-agent-plugin/.env` exists and contains the API key (otherwise direct the user to `/fm-agent:config`)
- If `./fm_agent/` already exists, ask the user whether to **resume** or **start fresh**. **Resume (`--resume`) is not yet implemented** — when chosen, the skill stops and asks the user whether to start fresh instead.
- Launch as a background task so the session is not blocked
- Schedule periodic polling via the `loop` skill to detect completion, then notify the user with success or failure.

The skill also exposes an **orchestration mode** used exclusively by `/fm-agent:auto-fix` to run a single deterministic full-project verification round synchronously, with no resume/fresh prompts and no background polling.

### diagnose

Read FM-Agent output from `./fm_agent/`:
- **Summary first**: Show `bug_validation/summary.json` with totals
- **Details on request**: Show individual bug reports (`<source>--<function>.md`)
- Bug reports include: specification claim, actual behavior, code evidence, trigger condition, probe script, probe output

### Output Directory

Analysis results in `./fm_agent/`:
- `bug_validation/summary.json` - Aggregated summary
- `bug_validation/<source>--<function>.md` - Bug detail reports
- `bug_validation/<source>--<function>.result.json` - Machine-readable results

## Supported Languages

Rust, C, C++, Python, Java, Go, CUDA, JavaScript, TypeScript, ArkTS
