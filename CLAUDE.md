# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Claude Code plugin that integrates [FM-Agent](https://github.com/fmagent-project/FM-Agent) — a framework for fully automated reasoning on large-scale codebases using LLM-based Hoare-style verification for bug detection.

## Plugin Structure

```
.claude-plugin/
│   └── plugin.json          # Plugin manifest (name, version, description)
├── skills/
│   ├── install/SKILL.md      # /fm-agent:install - Clone FM-Agent to plugin data dir
│   ├── config/SKILL.md       # /fm-agent:config  - Show/modify FM-Agent config
│   ├── run/SKILL.md          # /fm-agent:run     - Execute analysis (background)
│   ├── export/SKILL.md       # /fm-agent:export  - Export conversation after git commit
│   ├── diagnose/SKILL.md      # /fm-agent:diagnose - View bug analysis results
│   ├── auto-fix/SKILL.md     # /fm-agent:auto-fix - Run verification-repair loop
│   └── help/SKILL.md         # /fm-agent:help    - Show help information
```

## Available Commands

| Command | Description |
|---------|-------------|
| `/fm-agent:install` | Clone FM-Agent to `${CLAUDE_PLUGIN_DATA}/FM-Agent/` |
| `/fm-agent:config` | Show/modify FM-Agent configuration |
| `/fm-agent:run` | Execute FM-Agent analysis on current project (runs in background) |
| `/fm-agent:diagnose` | View bug analysis results |
| `/fm-agent:auto-fix` | Run full-project verification-repair loop |
| `/fm-agent:help` | Show help information |

## Key Environment Variables

- `LLM_API_KEY` - Required for FM-Agent LLM calls. Stored in `${CLAUDE_PLUGIN_DATA}/.env` (format: `export LLM_API_KEY=<value>`) and sourced before `python3 main.py` runs. Set via `/fm-agent:config`.
- `CLAUDE_PLUGIN_DATA` - Plugin data directory where FM-Agent is installed

## FM-Agent Installation

FM-Agent is installed to `${CLAUDE_PLUGIN_DATA}/FM-Agent/` via `/fm-agent:install`. After cloning, run `${CLAUDE_PLUGIN_DATA}/FM-Agent/install.sh` to install dependencies.

## Analysis Output

When `/fm-agent:run` executes, it creates `./fm_agent/` in the project being analyzed:

| Path | Description |
|------|-------------|
| `fm_agent/bug_validation/summary.json` | Aggregated bug summary |
| `fm_agent/bug_validation/<file>--<fn>.md` | Per-function bug reports |
| `fm_agent/fm_agent.log` | Full execution log |

## Supported Languages

Rust, C, C++, Python, Java, Go, CUDA, JavaScript, TypeScript, ArkTS

## Configuration

FM-Agent settings live in two files:
- `${CLAUDE_PLUGIN_DATA}/.env` — `LLM_API_KEY` only
- `${CLAUDE_PLUGIN_DATA}/FM-Agent/config.py` — everything else

The `/fm-agent:config` skill exposes these 8 settings for modification: `LLM_MODEL` (default `anthropic/claude-sonnet-4.6`), `OPENCODE_SETUP_MODEL`, `OPENCODE_SPEC_MODEL`, `OPENCODE_BUG_VALIDATION_MODEL`, `REASONER_POST_CONDITION_MODEL`, `REASONER_SPEC_CHECK_MODEL`, `LLM_LLM_API_KEY`, `LLM_OPENROUTER_API_BASE_URL`. Other `config.py` constants (`MAX_SPC_ITER`, `GRANULARITY`, `MAX_WORKERS`, `OPENCODE_MAX_RETRIES`) must be edited directly.

## Adding New Skills

Add a new skill by creating a directory under `skills/` with a `SKILL.md` file containing:

```markdown
---
name: Skill Name
description: Trigger phrases for this skill  <-- these activate /fm-agent:name
version: 0.1.0
---

# Skill content...
```

The skill becomes available as `/fm-agent:<name>`.
