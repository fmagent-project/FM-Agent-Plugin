---
name: Export Conversation
description: Use this skill immediately after any git commit to export the Claude Code conversation. The skill runs automatically after each `git commit` Bash command — do NOT wait to be asked. Just run it right after the commit completes. It creates a transcript and summary of all work done since the previous commit, scoped to the files that changed. Essential for tracking incremental progress commit by commit. If multiple commits have been made in the session, use this skill after EACH commit to export that commit's conversation.
version: 0.4.0
allowed-tools: Write,Bash,AskUserQuestion,Skill
---

Export the Claude Code conversation to `./fm_agent_plugin/` after a git commit is executed as a Bash command. Two output files are created: a full transcript and a summary.

## Overview

This skill runs automatically after any `git commit` Bash command completes. It captures:
- The commit metadata (hash, message, author, changed files)
- The conversation turns between the previous commit and the new commit
- A concise summary of the user's goal, decisions made, and code changes

Output files use the commit hash as identifier and are written to the **current project directory where Claude Code is running** (not the plugin directory): `export-<COMMIT_ID>.md` and `export-<COMMIT_ID>-summary.md` in the `./fm_agent_plugin/` subdirectory.

If no git commit has been executed in the current session, do NOT run this skill — return a message saying there's no commit to export to.

**IMPORTANT — Multiple commits in session:** If multiple git commit Bash commands have been executed in the current session, run this skill AFTER EACH COMMIT. For each commit, export the conversation between that commit and the one before it. The most recent commit is always `git rev-parse --short HEAD`, the previous one is `HEAD~1`.

## Steps

### 1. Get the commit id

This skill triggers after a git commit Bash command. Extract the commit hash using:
```bash
git rev-parse --short HEAD
```

Determine whether this is the first `git commit` Bash command in the current session by checking the current session context for any earlier successful `git commit` Bash command before the one that just finished.

- If no earlier successful `git commit` Bash command appears in the current session context, this is the first commit in the current session. There is no earlier in-session commit boundary. Use the session start as the previous reference, meaning the beginning of the current Claude Code conversation in this session. Scope the export to all conversation turns from session start up to this commit.
- If this is not the first commit in the current session, get the previous commit hash for scoping the conversation:
```bash
git rev-parse --short HEAD~1
```
Use that previous in-session commit as the earlier boundary for the export.

The commit id will be used in the filename to uniquely identify the conversation associated with that commit.

### 2. Reconstruct and Write the conversation transcript

Walk through the conversation context between the current new commit and its previous commit, and write it to `./fm_agent_plugin/export-<COMMIT_ID>.md` in a readable plain-text format.

```md
# Claude Code Conversation

## Date: <current date>

## Commit: <commit id>

---

**User:** <user message>

**Claude:** <claude response>

---

**Claude:** (runs `some command`)

**Tool output:** [output of the command]

---

**User:** <user message>

**Claude:** <claude response>
```

- Write ALL conversation turns that occurred between the new commit and its previous commit
- Include tool calls (Bash, Read, Edit, etc.) and their outputs where relevant to the work
- Separate each user/claude exchange with `---`
- Keep tool output concise — omit very long outputs, summarize if needed
- Do not fabricate content — only write what appears in the conversation context

### 3. Write the summary file

The summary file should be written to `./fm_agent_plugin/export-<COMMIT_ID>-summary.md`. Conclude the conversation with a summary, including what the user wanted to achieve and what was done.

**Filtering:** Before writing the summary, scan the conversation and identify only the parts directly related to the code changes in this commit. Skip any off-topic discussions, side conversations, debugging tangents that led nowhere, or general Q&A that didn't result in code changes. The summary should reflect only what contributed to the final diff.

```md
# Conversation Summary

## Date: <current date>

## Commit: <commit id>

## User's intent and goal

<summary of user's intent and goal based on the conversation>

## What was done

<summary of what was done based on the conversation>

## Code changes

- `<function 1>` — what the user wanted to implement
- `<function 2>` — what the user wanted to implement

---

Exported by fm-agent-plugin at <ISO timestamp>
```

When writing `What was done` and `Code changes`:
- Describe WHAT the changed function or unit guarantees, not HOW it achieves it.
- Goal-level decription of what the user wanted to achieve
- Intent-revealing bullets describing the code changes, not implementation details or line-by-line changes
- Concise and capture the essence of the conversation
- Prefer data invariants, state invariants, result contracts, output format contracts, error contracts and resource/ownership contracts.
- DO NOT go into technical details of the implementation
- DO NOT think in terms of code diffs or line changes — focus on the user's intent and the high-level design decisions
- DO NOT include any information that is irrelevant to the user's intent and the overall goal of this commit, such as building process, test outputs, error messages, off-topic discussions

### 4. Offer incremental FM-Agent analysis

The auto-fix loop is manually triggered and no longer branches from commit messages. Preserve the current incremental FM-Agent behavior for all commits:

- Ask the user whether they want to run FM-Agent on just this commit's changes. The export already pinned the commit id (`<COMMIT_ID>`) — reusing it as the scope for incremental analysis gives the user verification of exactly what they just committed without re-analyzing the whole codebase.

Use AskUserQuestion:

- Question: "Run FM-Agent incremental analysis on commit `<COMMIT_ID>`?"
- Options:
  - Yes (Run `/fm-agent:run` scoped to this commit)
  - No (Skip — exports are saved, the user can run analysis later)

If the user selects "Yes", invoke the `fm-agent:run` skill with the commit id as its argument. The commit id matches the export filenames, making the analysis output easy to correlate with the export:

```
Skill(skill="fm-agent:run", args="<COMMIT_ID>")
```

If the user selects "No", end the skill without further action — the export files are already saved.