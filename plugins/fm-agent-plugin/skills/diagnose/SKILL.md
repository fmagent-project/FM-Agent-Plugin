---
name: FM-Agent Diagnose
description: Use when the user asks to "diagnose fm-agent", "show bugs", "view fm-agent results", "check bug reports", or wants to analyze FM-Agent output.
version: 0.1.0
allowed-tools: Read, Glob
---

Diagnose and analyze FM-Agent output in the project directory.

## Overview

This skill reads FM-Agent output (`./fm_agent/`) from the project directory. It presents a summary, lists bugs with pagination, and shows detailed bug reports on request. The primary data source is `fm_agent/bug_validation/summary.json`, with per-bug detail in corresponding `.md` and `.result.json` files.

## Prerequisites

- `./fm_agent/` directory must exist.
- If files or directories are missing, FM-Agent analysis has not completed. DO NOT sleep and retry — finish diagnosis immediately with whatever is available.

## Output Directory Structure

```
fm_agent/
├── phases.json                 # Analysis phase definitions and module groupings
├── fm_agent_file_list.json     # All source files processed
├── extracted_functions/        # Per-function extracted source snippets
├── spec_prompts/               # Generated spec prompts and topdown layer JSONs
├── logic_verification_results/ # Formal verification output
├── bug_validation/             # Bug reports (the directory you care about)
│   ├── summary.json            # Aggregated results — always start here
│   ├── <bug_id>.md             # Human-readable bug detail report
│   ├── <bug_id>.result.json    # Machine-readable bug summary (same fields as summary entry)
│   └── _probe_<bug_id>.*       # Compiled probe binary and code source
```

The `<bug_id>` uses `--` as path separators, e.g. `src--observer--net--server-cpp--accept`.

## Step 1: Report Summary

Always start here. Read `fm_agent/bug_validation/summary.json`.

The JSON contains these top-level fields:
- `total_reported` — total bugs found
- `total_confirmed` — bugs validated by probe
- `total_not_confirmed` — bugs where probe did not reproduce the issue
- `total_error` — bugs where probe execution failed
- `bugs` — array of bug objects (see Step 2)

Present the summary as:

```
FM-Agent Analysis Summary

Total Reported: X
  Confirmed:     Y
  Not Confirmed: Z
  Errors:        W

Bug reports: fm_agent/bug_validation/
```

If `total_reported` is 0, say so and stop — there are no bugs to list.

## Step 2: Show Bug List (Paginated)

Extract the `bugs` array from `summary.json`. Each bug object has these fields:

| Field | Description |
|-------|-------------|
| `id` | Unique bug identifier (also the detail file basename) |
| `function_name` | The function analyzed |
| `confirmation_status` | `"confirmed"` or `"not_confirmed"` |
| `attempts` | Number of probe attempts |
| `trigger_summary` | One-line description of the trigger condition |
| `detail_file` | Path to the `.md` detail report |
| `source_file` | Path to the extracted function source |

Sort the bugs: **confirmed first, then not_confirmed**. Within each group, keep the original order from the JSON.

Display bugs in pages of 10. For each bug show:
- **Index** — 1-based, global across all pages
- **Function** — `function_name`
- **Status** — "Confirmed" or "Not Confirmed" (human-readable, not the raw key)
- **Trigger** — `trigger_summary`

Format as a table:

```
| # | Function | Status | Trigger |
|---|----------|--------|---------|
| 1 | quit_signal_handle | Confirmed | pthread_create return value not checked, shutdown thread silently not created |
| 2 | flush_internal | Confirmed | Buffer advanced on partial write before IOERR_WRITE returned |
...
```

**Pagination rules:**

1. Show the first 10 bugs.
2. After each page, ask:
   - "Show next 10 bugs?" — next page
   - "Show details for bug N" — jump to Step 3 for that bug
   - "Show all remaining" — dump all remaining bugs without further pagination
3. If 10 or fewer bugs remain, list them all and don't prompt again.
4. If there are no more pages, say "End of bug list."

## Step 3: Show Bug Detail

When the user asks for details on bug N, find the corresponding entry in the `bugs` array by index (1-based). Read the markdown file at `fm_agent/bug_validation/<bug.id>.md`.

The markdown file contains these sections — present each one:

### Report Header
The file opens with `# Bug Report: <function_name>`, followed by **Source file**, **Verdict** (always "MISMATCH" for reported bugs), and **Confirmation status**.

### Specification Claim
What the function's formal specification requires as a post-condition. This is what correct code should guarantee.

### Actual Behavior
What the code actually does, expressed in both natural language and formal logic. The gap between this and the Specification Claim is the bug.

### Code Evidence
The specific code lines (with line numbers) that cause the violation.

### Trigger Condition
A description of the concrete scenario that triggers the bug — what must be true for the bug to manifest.

### How to Trigger
Concrete reproduction steps: input parameters, expected vs. actual output, and compilation/run instructions.

### Probe Script
The full standalone C++ test program used to validate the bug.

### Probe Output
Raw stdout/stderr from executing the probe script. Look for `CONFIRMED` (bug reproduced) or `NOT CONFIRMED` (bug not reproduced) lines.

---

**Tip:** For a quick machine-readable view, you can also read `fm_agent/bug_validation/<bug.id>.result.json` which contains the same fields as the summary entry (`confirmation_status`, `probe_stdout`, `trigger_summary`, etc.).