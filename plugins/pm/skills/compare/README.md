# compare

Compare two versions of a program timeline to identify what changed between them.

Takes two files (Excel, YAML, or JSON), matches tasks by name and workstream, and produces a structured diff report covering date shifts, status changes, added/removed tasks, and per-workstream impact analysis.

Optionally outputs an updated YAML with changed tasks flagged as at-risk for visual review in the timeline editor.

## Usage

```bash
# Compare two Excel snapshots
/pm:compare timeline-v1.xlsx timeline-v2.xlsx

# Compare and output a flagged YAML for visual review
/pm:compare timeline-jan.xlsx timeline-mar.xlsx --yaml

# Mix formats — Excel before, YAML after
/pm:compare old-timeline.xlsx current-timeline.yaml
```
