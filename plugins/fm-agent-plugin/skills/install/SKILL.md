---
name: FM-Agent-Install
description: Use when the user asks to "install fm-agent", "setup fm-agent", "clone fm-agent", or needs to set up FM-Agent in the plugin data directory.
version: 0.2.0
allowed-tools: Bash(*)
---

Install FM-Agent framework to the plugin data directory for automated code reasoning and bug detection.

## Overview

This skill clones FM-Agent from the official repository to the plugin data directory `$HOME/.fm-agent-plugin/`. FM-Agent is a framework that realizes fully automated reasoning for large-scale systems using LLM-based Hoare-style verification.

## Installation Steps

For the following steps, exactly execute the provide bash commands. Do not run other commands or modify the provided commands.

If any step fails, stop the installation and inform the user to set up FM-Agent manually.

### Step 1: Ensure Plugin Data Directory Exists

Create `$HOME/.fm-agent-plugin/` if it does not already exist (`mkdir -p` is a no-op when the directory is present):

```bash
mkdir -p $HOME/.fm-agent-plugin/
```

If `mkdir` fails (for example, due to permission errors), stop the installation and inform the user to set up FM-Agent manually.

### Step 2: Clone or Update FM-Agent

Clone FM-Agent to `$HOME/.fm-agent-plugin/FM-Agent`, or update if it already exists:

```bash
if [ -d "$HOME/.fm-agent-plugin/FM-Agent" ]; then
  echo "FM-Agent already exists, updating..."
  cd "$HOME/.fm-agent-plugin/FM-Agent" && git pull
else
  echo "Cloning FM-Agent..."
  cd "$HOME/.fm-agent-plugin/" && git clone https://github.com/fmagent-project/FM-Agent.git
fi
```

### Step 3: Install Dependencies

Run FM-Agent's install script to set up required dependencies:

```bash
"$HOME/.fm-agent-plugin/FM-Agent/install.sh"
```

If the install script fails, stop the installation and inform the user to install manually.

### Step 4: Verify Installation

After cloning and installing dependencies, confirm FM-Agent is ready:

```bash
if [ -f "$HOME/.fm-agent-plugin/FM-Agent/main.py" ]; then
  echo "FM-Agent installed successfully"
else
  echo "FM-Agent installation incomplete — main.py not found"
fi
```

### Step 5: Next Steps

After successful installation, inform the user:

```
FM-Agent installed successfully.

Next steps:
/fm-agent:config - Configure FM-Agent settings
/fm-agent:run - Run FM-Agent for current codebase
/fm-agent:diagnose - Diagnose bugs found by FM-Agent
```

## Reference Files

For detailed information about FM-Agent (after cloning):
- **`$HOME/.fm-agent-plugin/FM-Agent/README.md`** - Brief introduction to FM-Agent
- **`$HOME/.fm-agent-plugin/FM-Agent/install.sh`** - Installation script for dependencies