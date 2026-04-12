---
name: debrief
description: |
  Adversarial meeting debrief generator.
  Synthesizes one or more meeting transcripts into a structured debrief — decisions, action items, open questions, risks, and cross-meeting changes. Then runs two Sonnet critic passes to verify completeness and accuracy.
user-invocable: true
argument-hint: <transcript-1> [transcript-2 ...] [--output <path>] [--single]
---

Follow the process defined in `skills/debrief/SKILL.md`.
