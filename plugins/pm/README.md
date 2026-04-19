# pm

Program management tools. Ships six skills and a browser-based timeline editor.

| Skill | What it does |
|-------|-------------|
| [`/pm:timeline`](skills/timeline/SKILL.md) | Extract, draft, and edit program timelines from prose, conversation, or structured shifts. Pairs with the [browser editor](../../tools/timeline-editor/README.md). |
| [`/pm:compare`](skills/compare/SKILL.md) | Compare two versions of a program timeline — matches tasks by name and workstream, reports date shifts, status changes, added/removed tasks, and per-workstream impact. Optionally outputs a flagged YAML for visual review. |
| [`/pm:debrief`](skills/debrief/SKILL.md) | Adversarial meeting debrief generator — synthesizes one or more transcripts into decisions, action items, open questions, risks, and cross-meeting changes, then runs two [critic passes](agents/debrief-critic.md) to verify completeness and accuracy. |
| [`/pm:assess`](skills/assess/SKILL.md) | Adversarial risk assessment for project critical path — interviews for context, drafts a structured register, then iterates with a [critic sub-agent](agents/risk-critic.md) until stress-tested. |
| [`/pm:exec-summary`](skills/exec-summary/SKILL.md) | Concision editor for executive communications — takes a draft, surfaces analyst questions, produces a structured exec-ready rewrite, then runs adversarial critique via [concision-critic](agents/concision-critic.md). Supports persistent [personas](skills/persona/SKILL.md) via `--audience <slug>`. |
| [`/pm:persona`](skills/persona/SKILL.md) | Create and manage executive personas — persistent models of specific readers (domain expertise, priorities, reading style, predictable questions, sensitivities). Used by `/pm:exec-summary` to calibrate output for a known audience. |

## Supported inputs

All five skills accept files as positional arguments. The file format table below applies to all of them.

### File formats

| Extension | How it's read |
|-----------|--------------|
| `.xlsx`, `.xls` | Parsed as spreadsheet — columns matched by synonym (Task/Activity, Start Date/Begin, Status/State, etc.) |
| `.yaml`, `.yml` | Parsed as structured data — if it matches the `/pm:timeline` canonical schema (`workstreams` + `tasks` keys), fields are extracted directly |
| `.json` | Same as YAML — parsed as structured data |
| `.md`, `.txt` | Read as unstructured prose — programs, dates, milestones, dependencies, and risk language extracted by analysis |
| `.pdf` | Read as unstructured prose (extractable text only) |

### Content types

The file format tells the skill *how* to parse; the content tells it *what* to extract. Common content types:

| Content | What the skills extract from it |
|---------|-------------------------------|
| **Program timeline** (Excel or YAML from `/pm:timeline`) | Programs, workstreams, tasks, dates, statuses, milestone structure. `/pm:assess` also infers stage and dependencies from task sequencing. |
| **Strategy / war doc** | Program names, approval pathways, regulatory authority, competitive context, strategic rationale. Rich source of critical-path "why." |
| **Steering committee deck / meeting notes** | Status updates, recently surfaced risks, decision log, action items. Often contains implicit critical-path signals ("we need X before Y"). |
| **Regulatory submission plan** | Approval pathway, submission timeline, agency interactions, required studies. Directly maps to regulatory risk category. |
| **Prior risk register / assessment** | Previously identified risks, severity scores, mitigations, critical-path descriptions. Baseline for incremental updates. |
| **Protocol or study report** | Endpoints, enrollment targets, comparator design, safety monitoring. Maps to clinical data risk category. |

Multiple files can be provided at once — the skills cross-reference them (e.g., a timeline gives dates while a strategy doc gives rationale).

## Files

```
plugins/pm/
├── .claude-plugin/plugin.json
├── commands/
│   ├── timeline.md
│   ├── compare.md
│   ├── debrief.md
│   ├── assess.md
│   ├── exec-summary.md
│   └── persona.md
├── skills/
│   ├── timeline/SKILL.md
│   ├── compare/SKILL.md
│   ├── debrief/SKILL.md
│   ├── assess/SKILL.md
│   ├── exec-summary/SKILL.md
│   └── persona/SKILL.md
├── agents/
│   ├── risk-critic.md
│   ├── concision-critic.md
│   └── debrief-critic.md
└── README.md

tools/timeline-editor/          # paired browser tool for /pm:timeline
├── index.html
├── sample-timeline.yaml
├── examples/
│   └── sample-program.xlsx
└── README.md

notes/risk/                     # output directory for /pm:assess artifacts
├── portfolio-*.md              # portfolio context files
├── register-*.md               # risk registers
└── exec-summary-*.md           # executive summaries

notes/timeline/                 # output directory for /pm:compare artifacts
├── compare-*.md                # diff reports
└── compare-*.yaml              # flagged YAML for visual review

notes/debrief/                  # output directory for /pm:debrief artifacts
└── {slug}-{date}.md            # meeting debriefs

notes/exec/                     # output directory for /pm:exec-summary artifacts
├── {slug}-{date}.md            # exec summaries
└── personas/                   # persistent reader profiles (/pm:persona)
    └── {slug}.md               # individual persona files
```
