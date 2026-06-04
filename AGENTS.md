# Repository Guidelines

## Project Structure & Module Organization

This repository contains the FM-Agent plugin metadata and skill instructions.

- `README.md` documents installation, available commands, and user-facing behavior.
- `.claude-plugin/plugin.json` is the Claude plugin manifest.
- `skills/.codex-plugin/plugin.json` is the Codex plugin manifest and points to `./skills/`.
- `skills/<command>/SKILL.md` defines each plugin command, such as `install`, `config`, `run`, `diagnose`, `auto-fix`, and `help`.
- `.claude-plugin/marketplace.json` and `.agents/plugins/marketplace.json` hold marketplace registration metadata.
- Runtime outputs such as `fm_agent/`, `fm_agent_plugin/`, `fm_agent_run.log`, and `fm_agent_run.pid` are generated artifacts and should not be treated as source.

## Build, Test, and Development Commands

There is no package build step in this repo. Validate changes with lightweight checks:

```bash
python3 -m json.tool .claude-plugin/plugin.json
python3 -m json.tool skills/.codex-plugin/plugin.json
python3 -m json.tool .claude-plugin/marketplace.json
```

Use the installed plugin commands for manual verification:

```text
/fm-agent:help
/fm-agent:config
/fm-agent:run
/fm-agent:diagnose
```

Run `/fm-agent:install` before commands that require `${CLAUDE_PLUGIN_DATA}/FM-Agent/`.

## Coding Style & Naming Conventions

Write Markdown in short, imperative instructions. Skill files must keep their YAML front matter (`name`, `description`, `version`, `allowed-tools`) valid and specific. Use fenced code blocks with language tags for commands. Keep command names lowercase and hyphenated, matching directory names under `skills/`, for example `auto-fix/SKILL.md`.

For JSON, use two-space indentation, stable key ordering where practical, and valid URLs/licenses matching the manifests.

## Testing Guidelines

No automated test suite is currently defined. For skill changes, test the affected command manually in a plugin-enabled workspace and verify both normal and failure paths. For example, a `run` change should cover missing `.env`, existing `fm_agent/`, and successful artifact creation at `fm_agent/bug_validation/summary.json`.

## Commit & Pull Request Guidelines

Recent history uses short, lowercase commit subjects such as `update` and concise change descriptions like `update OPENROUTER_API_KEY to LLM_API_KEY`. Prefer more specific subjects when possible, for example `update config env defaults`.

Pull requests should include a brief summary, affected commands or manifests, manual verification steps, and screenshots or terminal excerpts only when they clarify user-visible behavior. Link related issues when available.

## Security & Configuration Tips

Never commit real `LLM_API_KEY` values or generated `.env` files from plugin data. Keep secrets in `${CLAUDE_PLUGIN_DATA}/.env`, and document new configuration keys in both `README.md` and the relevant skill file.
