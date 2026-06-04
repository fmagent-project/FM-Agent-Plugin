---
name: FM-Agent Config
description: Use when the user asks to "configure fm-agent", "config fm-agent", "show fm-agent configuration", "set fm-agent model", "set fm-agent api key", or needs to modify FM-Agent configuration.
version: 0.3.0
allowed-tools: Read, Write, AskUserQuestion
---

Configure FM-Agent settings in the plugin data directory.

## Overview

This skill reads the FM-Agent configuration file from the plugin data directory `${CLAUDE_PLUGIN_DATA}/FM-Agent/config.py` and allows users to modify configuration.

## Prerequisites

- `fm-agent:install` executed before to have FM-Agent installed in the plugin data directory `${CLAUDE_PLUGIN_DATA}/FM-Agent/`

## Configuration Steps

For the following steps, exactly execute the provide bash commands. Do not run other commands or modify the provided commands.

### Step 1: Check if `${CLAUDE_PLUGIN_DATA}/.env` exists

```bash
cat ${CLAUDE_PLUGIN_DATA}/.env 2>/dev/null || echo "NO_ENV_FILE"
```

**If `.env` does not exist:** Create an `.env` file in `${CLAUDE_PLUGIN_DATA}/` with default content:

```
export LLM_API_KEY=""
export LLM_API_BASE_URL="https://openrouter.ai/api/v1"
export LLM_MODEL="anthropic/claude-sonnet-4.6"
export OPENCODE_MODEL_PROVIDER="openrouter"
```

**If `.env` exists:** Go to next step.

### Step 2: List Configuration

Read file `${CLAUDE_PLUGIN_DATA}/.env` and list configuration as table:

| Parameter                       | Value                          |
| ------------------------------- | ------------------------------ | 
| `LLM_API_KEY`                   | <current_value>                |
| `LLM_API_BASE_URL`              | <current_value>                |
| `LLM_MODEL`                     | <current_value>                |
| `OPENCODE_MODEL_PROVIDER`       | <current_value>                |

Then ask user to select: "Do you want to modify any configuration? (yes/no)"

If the user selects "no", Goto next step.

If the user selects "yes", then use AskUserQuestion to ask which configuration they want to modify (multi-select):
- Question: "Which configuration do you want to modify? (you can select multiple)"
- multiSelect: true
- Options:
  - LLM_API_KEY
  - LLM_API_BASE_URL
  - LLM_MODEL
  - OPENCODE_MODEL_PROVIDER

For each selected configuration, ask for the new value:
- Question: "Please enter the new value for {config_name}:"

Then update the `${CLAUDE_PLUGIN_DATA}/.env` file with the new value.

### Step 3: Next Steps

Check whether the user has set `LLM_API_KEY`. If not, inform the user to set it and go back to Step 2.

After successful configuration, inform the user:

```
FM-Agent configured successfully.

Next steps:
/fm-agent:run - Run FM-Agent for current codebase
/fm-agent:diagnose - Diagnose bugs found by FM-Agent
```

## Reference Files

- **`${CLAUDE_PLUGIN_DATA}/FM-Agent/config.py`** - Configuration file of FM-Agent.