# Claude Code Plugin of FM-Agent

This plugin integrates [FM-Agent](https://github.com/haoran-ding/FM-Agent) into Claude Code for automated code reasoning and bug repair.

## Prerequisites

- Ubuntu 22.04+ or compatible Linux distribution
- Python 3.12+

## How to install

```
/plugin marketplace add haoran-ding/FM-Agent-Plugin
/plugin install fm-agent-plugin@fm-agent-plugin
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

Clone FM-Agent from official repository to the plugin data directory (`${CLAUDE_PLUGIN_DATA}/FM-Agent/`):
```bash
git clone https://github.com/haoran-ding/FM-Agent.git
```
Then run `${CLAUDE_PLUGIN_DATA}/FM-Agent/install.sh` to install dependencies.

### config

Read and modify FM-Agent settings split across two files:
- API key → `${CLAUDE_PLUGIN_DATA}/.env` (sourced at runtime via `source .env`)
- All other settings → `${CLAUDE_PLUGIN_DATA}/FM-Agent/config.py`

Workflow:
- If `.env` already exists, show its contents and ask whether to change the stored API key
- If updating the key (or `.env` doesn't exist), use `$OPENROUTER_API_KEY` from the environment if set, otherwise prompt the user
- List the current `config.py` values, then use AskUserQuestion (multi-select) to let the user pick which settings to modify

**Configurable settings**:
- `LLM_MODEL` - Default model used as fallback for all task-specific model settings (default: `anthropic/claude-sonnet-4.6`)
- `OPENCODE_SETUP_MODEL` - Codebase understanding, phase planning, domain context
- `OPENCODE_SPEC_MODEL` - Batch behavioral spec generation
- `OPENCODE_BUG_VALIDATION_MODEL` - Validation of `MISMATCH` results with probe scripts
- `REASONER_POST_CONDITION_MODEL` - Block post-condition generation
- `REASONER_SPEC_CHECK_MODEL` - Spec violation checking
- `LLM_OPENROUTER_API_KEY` - OpenRouter API key (stored in `.env`)
- `LLM_OPENROUTER_API_BASE_URL` - OpenRouter API endpoint

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
- Verify `${CLAUDE_PLUGIN_DATA}/.env` exists and contains the API key (otherwise direct the user to `/fm-agent:config`)
- If `./fm_agent/` already exists, ask the user whether to **resume** (use `--resume`) or **start fresh** (delete and rerun)
- Launch as a background task so the session is not blocked
- Schedule periodic polling via the `loop` skill to detect completion, then notify the user with success or failure.

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
