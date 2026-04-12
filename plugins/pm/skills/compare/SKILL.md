---
name: compare
description: |
  Compare two versions of a program timeline to identify what changed.
  Parses two Excel or YAML files, matches tasks by name and workstream, and produces a structured diff report showing date shifts, status changes, added/removed tasks, and per-workstream impact analysis.
  Optionally outputs an updated YAML with changed tasks flagged for visual review in the timeline editor.
user-invocable: true
argument-hint: <before-file> <after-file> [--output <path>] [--yaml]
---

You are the orchestrator of a timeline comparison session. Your job is to take two versions of a program timeline — a "before" and an "after" — and produce a clear, structured report of what changed, what the impact is, and what the user should pay attention to.

The user is a biotech program manager comparing timeline snapshots. They want to know: what moved, by how much, and does it matter?

---

## Audience

Same as `/pm:timeline` — a domain expert, not a software engineer:

- Absolute paths in handoff messages.
- No engineering jargon.
- Never ask them to run terminal commands.
- When you ask a clarifying question, offer a concrete default they can accept with "ok".

---

## Inputs

Two positional arguments: `<before-file> <after-file>`

Accepted formats: `.xlsx`, `.xls`, `.yaml`, `.yml`, `.json`

Optional flags:
- `--output <path>` — where to write the diff report (default: `notes/timeline/compare-{slug}-{YYYY-MM-DD}.md`)
- `--yaml` — also output an updated YAML file with changed tasks flagged for visual review

If only one file is provided, stop and ask for the second:

```
I need two timeline files to compare — a "before" and an "after."

You gave me one: {path}

Drop or paste the path to the second file. Which one is this — the older version or the newer version?
```

If no files are provided, stop and ask:

```
I need two timeline files to compare — a "before" and an "after."

Drop or paste both file paths. Accepted formats: Excel (.xlsx), YAML (.yaml), or JSON (.json).
```

---

## Phase 1: Parse Both Files

### Excel files (`.xlsx`, `.xls`)

Parse the first worksheet. Extract headers, normalize to lowercase, and match columns using these synonyms:

| Field | Accepted column names |
|-------|----------------------|
| **name** | task, task name, name, activity, project, project name |
| **workstream** | workstream, group, category, swimlane, team, function |
| **start** | start, start date, begin, begin date |
| **end** | end, end date, finish, finish date |
| **duration** | duration, days, length |
| **status** | status, state |

If `end` is missing but `duration` is present, compute `end = start + duration days`.

Required columns: **name**, **workstream**, **start**, and either **end** or **duration**. If any are missing, stop and tell the user which columns are missing and what names are accepted.

### Status normalization

After lowercasing and stripping spaces and hyphens, map status values:

| Aliases | Canonical value |
|---------|----------------|
| notstarted, todo, pending, new, open | `not_started` |
| inprogress, wip, active, ongoing, started | `in_progress` |
| complete, completed, done, finished, closed | `complete` |
| atrisk, risk, warning | `at_risk` |
| blocked, block, stuck | `blocked` |
| delayed, late, slipping, slipped | `delayed` |
| estimate, estimated, forecast, tbd, tentative | `estimate` |

If a status value doesn't match any alias, default to `not_started` and note the unrecognized value in the terminal output.

### YAML / JSON files

If the file has `workstreams` and `tasks` keys matching the `/pm:timeline` canonical schema, parse directly. Each task has: `id`, `name`, `workstream`, `start`, `end`, `status`, `milestone`.

For YAML/JSON, tasks can be matched by `id` first (if both files have IDs), then fall back to name + workstream matching.

### Normalize

After parsing, each file should produce a flat list of tasks, each with:
- `name` (string)
- `workstream` (string, lowercased)
- `start` (ISO date)
- `end` (ISO date)
- `status` (canonical enum value)
- `milestone` (boolean — true if start == end)

---

## Phase 2: Match and Diff

### Task matching

Match tasks between before and after by **task name + workstream** (case-insensitive, trimmed). For YAML files with task IDs, match by ID first, then fall back to name + workstream for unmatched tasks.

Categorize every task:

| Category | Definition |
|----------|-----------|
| **Unchanged** | Same name+workstream match, all fields identical |
| **Modified** | Same name+workstream match, at least one field differs |
| **Added** | Present in after, no match in before |
| **Removed** | Present in before, no match in after |

### Per-task deltas (modified tasks only)

For each modified task, compute:

- **Start delta** — difference in days between before and after start dates. Positive = pushed out, negative = pulled in.
- **End delta** — same for end dates.
- **Duration change** — did the task get longer or shorter? By how many days?
- **Status change** — before status → after status. Flag regressions specifically.
- **Workstream change** — if the task name matched but the workstream differs, note the reassignment.

### Status regressions

These transitions are regressions and should be flagged:

- `complete` → anything other than `complete`
- `in_progress` → `blocked`, `delayed`, `at_risk`
- `not_started` → `blocked`, `delayed`
- Any status → `blocked` or `delayed`

---

## Phase 3: Impact Analysis

### Overall timeline

- **Earliest start**: did the program start date move? Before vs. after.
- **Latest end**: did the program end date move? Before vs. after.
- **Net timeline shift**: difference in the latest end date between versions.

### Per-workstream

For each workstream that has at least one changed task:
- Number of tasks changed, added, removed
- Average shift (days) across modified tasks
- Maximum shift (the single task that moved the most)
- Any status regressions

### Significant shifts

Flag any task whose start or end moved by more than **14 days (2 weeks)** as a significant shift. These get called out prominently in the report.

---

## Phase 4: Output

### Diff report

Write to `--output` path (default: `notes/timeline/compare-{slug}-{YYYY-MM-DD}.md`).

The slug is derived from the title if available (from YAML), or from the "after" filename if Excel.

If the output directory doesn't exist, create it.

```markdown
---
before: {absolute path to before file}
after: {absolute path to after file}
created: {YYYY-MM-DD}
---

# Timeline Comparison: {title}

## Summary

- Tasks compared: {total unique tasks across both files}
- Changed: {N} | Added: {N} | Removed: {N} | Unchanged: {N}
- Overall timeline shift: {e.g., "+42 days (latest end moved from 2026-09-15 to 2026-10-27)" or "no change"}
- Workstreams affected: {comma-separated list}

## Significant shifts (>2 weeks)

| Task | Workstream | Field | Before | After | Delta |
|------|-----------|-------|--------|-------|-------|
| {name} | {ws} | start | 2026-03-01 | 2026-04-15 | +45 days |
| {name} | {ws} | end | 2026-06-30 | 2026-08-15 | +46 days |

{Omit this section if no shifts exceed 2 weeks.}

## All date changes

| Task | Workstream | Before (start → end) | After (start → end) | Start delta | End delta |
|------|-----------|----------------------|---------------------|-------------|-----------|
| ... | ... | 2026-01-15 → 2026-04-30 | 2026-02-01 → 2026-05-15 | +17 days | +15 days |

{Omit if no date changes.}

## Status changes

| Task | Workstream | Before | After | Regression? |
|------|-----------|--------|-------|-------------|
| ... | ... | in_progress | blocked | Yes |

{Omit if no status changes.}

## Added tasks

| Task | Workstream | Start | End | Status |
|------|-----------|-------|-----|--------|
| ... | ... | ... | ... | ... |

{Omit if none added.}

## Removed tasks

| Task | Workstream | Start | End | Status |
|------|-----------|-------|-----|--------|
| ... | ... | ... | ... | ... |

{Omit if none removed.}

## Impact by workstream

### {Workstream name}

- Tasks changed: {N} | Added: {N} | Removed: {N}
- Largest shift: {task name}, {field} moved {delta}
- Status regressions: {count, or "none"}

{Repeat for each affected workstream. Omit workstreams with no changes.}
```

### Updated YAML (when `--yaml` is passed)

Write a YAML file using the canonical `/pm:timeline` schema, based on the "after" version, with changes flagged:

- Tasks that had **date shifts** or **status regressions**: set `status: at_risk`, unless their current status is already `blocked` or `delayed` (those are worse — don't overwrite).
- Add a `change_note` field to every modified task with a short human-readable description:
  - `"start shifted +6w (was 2026-01-15), end shifted +6w (was 2026-04-30)"`
  - `"status changed: in_progress → blocked"`
  - `"new task (not in previous version)"`
- Tasks that are unchanged: preserve as-is, no `change_note`.
- Tasks that were removed: do **not** include them in the YAML (it represents the current state).

Output path: `notes/timeline/compare-{slug}-{YYYY-MM-DD}.yaml`

---

## Phase 5: Terminal Summary

After writing the artifacts, print:

```
## Timeline Comparison Complete

Before:  {absolute path}
After:   {absolute path}
Report:  {absolute path to diff report}
{YAML:    {absolute path to YAML}  — only if --yaml was used}

Tasks compared: {N}
Changed: {N} | Added: {N} | Removed: {N} | Unchanged: {N}
Overall shift: {one-line summary}
Significant shifts (>2 weeks): {count}

{If --yaml was used:}
Open this file in your browser to review the flagged changes visually:
  file:///{absolute path to repo}/tools/timeline-editor/index.html

Then drop this YAML onto the page:
  {absolute path to output YAML}
```

---

## Edge Cases

- **Identical files**: If all tasks match and no fields differ, report "No changes detected between the two versions." Write no output files.
- **Completely different files**: If no tasks match by name + workstream, warn the user: "No matching tasks found between the two files. Are these timelines for the same program? Check that task names and workstream names are consistent across versions." Stop and ask whether to proceed with a full added/removed report.
- **Mixed formats**: One file can be Excel and the other YAML. Both get normalized to the same internal format before comparison.
- **Duplicate task names**: If the same name + workstream appears more than once within a single file, warn the user and ask them to disambiguate. Do not silently pick one.
- **Missing status column**: If an Excel file has no status column, default all tasks to `not_started` and note this in the terminal output.

---

## Notes

- This skill does not modify the input files. It reads both and produces new artifacts.
- The `change_note` field in the YAML output is not part of the canonical timeline schema. The timeline editor will ignore it, but it's useful for anyone reading the YAML directly.
- Task matching is by name + workstream, not by row order. Row order may differ between exports.
- When computing deltas, always show the sign: `+17 days` or `-5 days`. Never just `17 days`.
- "Significant shift" threshold is 14 days. This is not configurable — it's a reasonable default for program-level timelines where tasks span weeks or months.
