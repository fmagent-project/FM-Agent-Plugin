---
name: FM-Agent-Config
description: Use when the user asks to "configure fm-agent", "config fm-agent", "show fm-agent configuration", "set fm-agent model", "set fm-agent api key", or needs to modify FM-Agent configuration.
version: 0.3.1
allowed-tools: Read, Write, AskUserQuestion, Bash(*)
---

Configure FM-Agent settings in the plugin data directory.

## Overview

This skill reads the FM-Agent dotenv configuration file from `$HOME/.fm-agent-plugin/FM-Agent/.env` and allows users to modify configuration.

## Prerequisites

- `fm-agent:install` executed before to have FM-Agent installed in the plugin data directory `$HOME/.fm-agent-plugin/FM-Agent/`

## Configuration Steps

For the following steps, exactly execute the provide bash commands. Do not run other commands or modify the provided commands.

### Step 1: Check if `$HOME/.fm-agent-plugin/FM-Agent/.env` exists

```bash
cat $HOME/.fm-agent-plugin/FM-Agent/.env 2>/dev/null || echo "NO_ENV_FILE"
```

**If `.env` does not exist:** Create it from FM-Agent's example dotenv file:

```bash
cd $HOME/.fm-agent-plugin/FM-Agent && cp .env.example .env
```

**If `.env` exists:** Go to next step.

### Step 2: List Configuration

Read file `$HOME/.fm-agent-plugin/FM-Agent/.env` and list configuration as table:

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

Then update the `$HOME/.fm-agent-plugin/FM-Agent/.env` file with the new value.

### Step 3: Next Steps

Check whether the user has set `LLM_API_KEY`. If not, inform the user to set it and go back to Step 2.

After successful configuration, inform the user:

```
FM-Agent configured successfully.

Next steps:
/fm-agent:run-full - Run full-project FM-Agent analysis for current codebase
/fm-agent:run-incremental --incremental [<intent-msg>] - Run incremental FM-Agent analysis for current codebase
/fm-agent:diagnose - Diagnose bugs found by FM-Agent
```

## Reference Files

- **`$HOME/.fm-agent-plugin/FM-Agent/.env.example`** - Example dotenv configuration copied to create `.env`.
