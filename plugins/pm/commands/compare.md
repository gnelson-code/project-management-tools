---
name: compare
description: |
  Compare two versions of a program timeline to identify what changed.
  Parses two Excel or YAML files, matches tasks by name and workstream, and produces a structured diff report showing date shifts, status changes, added/removed tasks, and per-workstream impact analysis.
user-invocable: true
argument-hint: <before-file> <after-file> [--output <path>] [--yaml]
---

Follow the process defined in `skills/compare/SKILL.md`.
