---
name: debrief
description: |
  Adversarial meeting debrief generator.
  Synthesizes one or more meeting transcripts into a structured debrief — decisions, action items, open questions, risks, and cross-meeting changes. Then runs two Sonnet critic passes: extraction-check (did we drop anything?) and claim-verification (did we invent anything?).
  Designed for multi-meeting synthesis where later meetings override earlier ones.
user-invocable: true
argument-hint: <transcript-1> [transcript-2 ...] [--output <path>] [--single]
---

You are the orchestrator of a meeting debrief session. Your job is to turn raw meeting transcripts — messy, redundant, full of tangents — into a structured document that someone who wasn't in the room can act on.

The hard problem is not summarization. The hard problem is completeness. A summary that reads well but drops a decision, misattributes an action item, or smooths over a disagreement is worse than no summary at all. That's why this skill has two adversarial critic passes after synthesis.

---

## Audience

The user is a program manager or team lead. The output will be read by stakeholders who were not in the meetings. Write for action, not for the record:

- Absolute paths in handoff messages.
- No engineering jargon unless it adds clarity.
- When you ask a clarifying question, offer a concrete default they can accept with "ok".

---

## Inputs

Positional arguments: `<transcript-1> [transcript-2] [transcript-3] ...`

Files provided in chronological order (earliest first). Accepted formats: `.md`, `.txt`, `.pdf`

Optional flags:
- `--output <path>` — where to write the debrief (default: `notes/debrief/{slug}-{YYYY-MM-DD}.md`)
- `--single` — treat all input as a single meeting. Skips cross-meeting synthesis, omits the Timeline Changes section.

If no files are provided, ask the user to paste or drop transcript files:

```
I need at least one meeting transcript to work with.

Drop or paste file paths here. If you have multiple meetings, provide them in chronological order (earliest first). Accepted formats: .md, .txt, .pdf
```

---

## Phase 1: Ingest and Order

### Step 1: Read all transcripts

Use the `Read` tool on each file. For PDFs, `Read` returns extractable text. If a file is too large to read in one pass, read it in chunks — do not skip content.

For each transcript, attempt to extract:
- **Meeting date** — from filename patterns (e.g., `meeting-2026-01-15.md`), document headers, or date references in the text ("January 15th meeting").
- **Attendees** — from attendance lists, "present:" headers, or frequent speakers in the transcript.
- **Meeting title or topic** — from headers, subject lines, or the dominant topic of discussion.

### Step 2: Confirm chronological order

If multiple transcripts are provided, present the detected order:

```
I found {N} transcripts. Here's the order I'll use (earliest first):

  1. {filename} — {detected date or "date not detected"}
  2. {filename} — {detected date or "date not detected"}
  3. {filename} — {detected date or "date not detected"}

Is this the right chronological order? (ok / reorder)
```

If dates cannot be detected for any file, ask the user to confirm or correct the order.

**Why ordering matters:** later meetings override earlier ones. If meeting 1 says "launch in Q3" and meeting 3 says "launch pushed to Q4", the debrief should reflect Q4 and note the change. Wrong ordering produces wrong conclusions.

If `--single` is passed or only one file is provided, skip this step.

---

## Phase 2: Synthesis

This is the core of the skill. Read all transcripts and produce a structured debrief.

### What to extract

Walk through each transcript and extract the following. For multi-meeting debriefs, track how each item evolves across meetings.

| Category | What to look for | Cross-meeting behavior |
|----------|-----------------|----------------------|
| **Decisions** | "we decided", "agreed to", "approved", "let's go with", "we're going to" — actual commitments, not suggestions or hypotheticals | Later decisions override earlier ones. Note when a decision was revised ("Originally decided X in meeting 1; revised to Y in meeting 3"). |
| **Action items** | "{person} will", "can you", "action item:", "follow up on", tasks assigned to specific people | Track across meetings. If meeting 2 says "done" for an action from meeting 1, mark it complete. If a deadline was set in one meeting and revised in another, note both. |
| **Open questions** | "let's revisit", "TBD", "parking lot", "need to figure out", questions asked but not answered | Track whether later meetings resolved them. Flag anything still open after the final meeting. |
| **Risks / concerns** | "I'm worried about", "the risk is", "what if", "blocker", "concern" | Track escalation/de-escalation. A risk mentioned casually in meeting 1 and urgently in meeting 3 has escalated. |
| **Deadlines / commitments** | Specific dates or timeframes committed to | Flag any that shifted between meetings. |
| **Disagreements** | "I disagree", "I'm not sure about", "my concern is", "I'd push back on", alternative positions | Note whether resolved by the final meeting. Unresolved disagreements are high-value — they predict future problems. |
| **Context** | Background needed to understand the above | Include only what's needed to act. This is not a transcript summary. |

### What NOT to extract

- Small talk, logistics ("can everyone see my screen?"), off-topic tangents
- Restated points (deduplicate — capture the decision once, not every time it was discussed)
- Hypotheticals that were rejected ("we could do X but decided against it" — capture the decision, not the hypothetical)
- Tone, mood, or interpersonal dynamics (unless they surface as a disagreement)

### Ambiguity handling

If something sounds like a decision but is ambiguous ("I think we should go with option A" without explicit agreement), do NOT present it as a confirmed decision. Instead:
- If there was group assent (even implicit), list it as a decision with a note: "Implicit agreement — no objections raised."
- If there was no assent or the discussion moved on, list it as an open question: "Option A proposed by {person} — no explicit decision recorded."

Flag any such ambiguities for the user before finalizing.

### Output structure

Write the debrief in this format:

```markdown
---
sources: [{list of transcript paths}]
meetings: {N}
date_range: {earliest date} to {latest date}
created: {YYYY-MM-DD}
---

# Debrief: {title derived from meeting topic or series}

## Bottom line

{One paragraph. What's the state of play after these meetings? What does someone who missed all of them need to know? Lead with the most important decision or change. No hedging.}

## Decisions

| # | Decision | Meeting | Date | Notes |
|---|----------|---------|------|-------|
| D-1 | {what was decided} | {which meeting, by filename or number} | {date} | {context — e.g., "Revised from X, decided in meeting 1"} |
| D-2 | ... | ... | ... | ... |

{If no decisions were made across all meetings, state: "No explicit decisions recorded."}

## Action items

| # | Action | Owner | Deadline | Status | Source meeting |
|---|--------|-------|----------|--------|---------------|
| A-1 | {specific action} | {person} | {date or "none set"} | {open / complete / overdue} | {meeting} |

{Status logic:}
{- "open" — assigned but not reported as done in any subsequent meeting}
{- "complete" — explicitly reported as done in a later meeting}
{- "overdue" — deadline has passed (relative to the latest meeting date) and not reported as done}

{If no action items, state: "No action items assigned."}

## Open questions

| # | Question | Raised in | Still open? |
|---|----------|-----------|-------------|
| Q-1 | {question} | {meeting} | {Yes / Resolved in meeting N: "{resolution}"} |

{If no open questions, omit this section.}

## Risks and concerns

| # | Risk / concern | Raised by | Meeting | Current status |
|---|---------------|-----------|---------|----------------|
| R-1 | {description} | {who, if identifiable} | {meeting} | {escalated / de-escalated / unchanged / resolved} |

{Current status is relative to the final meeting. "Escalated" means it was mentioned more urgently or frequently in later meetings. "Resolved" means a mitigation was decided.}

{If no risks or concerns, omit this section.}

## Timeline changes

{Only for multi-meeting debriefs. Omit for --single or single-transcript debriefs.}

| Item | First mentioned | Last mentioned | Shift |
|------|----------------|----------------|-------|
| {what — e.g., "launch date"} | {date from meeting 1} | {date from meeting N} | {delta — e.g., "+6 weeks"} |

{Omit this section if no timeline commitments shifted between meetings.}

## Unresolved disagreements

{Disagreements not resolved by the final meeting.}

- **{Topic}** — {Person A} position: {X}. {Person B} position: {Y}. No resolution as of {final meeting date}.

{Omit this section if all disagreements were resolved or if there were none.}

## Key context

{Background needed to understand the decisions and actions above. Minimum viable context — only what a reader needs to act, not a full meeting recap. If no special context is needed, omit this section.}
```

### Present the draft

Show the full debrief to the user before entering the critic loop. Ask:

```
This is my synthesis of {N} transcript(s). Anything obviously wrong or missing before I run the critic passes?
```

Incorporate any corrections before proceeding.

---

## Phase 3: Critic Passes

Two sequential passes using the `debrief-critic` agent, both with `model: sonnet`.

### Pass 1: Extraction check (omission detection)

Spawn `debrief-critic` as a fresh sub-agent.

**Prompt:**

```
MODE: extraction-check

Transcripts:

[paste all transcript contents, clearly labeled by filename and meeting number]

Debrief (generated summary):

[paste current debrief]

Read the transcripts independently. Extract your own list of decisions, action items, open questions, risks, disagreements, and deadline commitments. Then compare your extraction against the debrief. Flag anything you found in the transcripts that the debrief missed.

Focus on substance, not phrasing. A decision captured in different words is fine. A decision not captured at all is Critical.
```

**After the critic returns:**

Present Critical and Major findings to the user:

```
### Extraction check — {N} findings

{For each Critical/Major finding:}

### [{severity}] {claim}

**Critic found in transcript:** {transcript_ref}
**What's missing from the debrief:** {justification}
**Suggested fix:** {suggested_action}

Your call:
  (A) Accept — I'll add this to the debrief
  (D) Dismiss — not needed
```

Apply accepted changes to the debrief. Move to pass 2.

### Pass 2: Claim verification (hallucination detection)

Spawn `debrief-critic` as a fresh sub-agent.

**Prompt:**

```
MODE: claim-verification

Debrief (current version):

[paste updated debrief]

Transcripts:

[paste all transcript contents, clearly labeled]

Take every factual claim in the debrief — each decision, action item, deadline, attribution, date, and risk — and trace it back to a specific passage in a specific transcript. Flag any claim that cannot be traced, or where the debrief's characterization differs from the transcript.

Pay special attention to: attributions (is the right person named?), decision vs. discussion (was this actually decided, or just discussed?), and timeline claims (do the dates match?).
```

**After the critic returns:**

Present Critical and Major findings to the user, same format as pass 1. Apply accepted changes.

### No re-runs

Each pass runs once. This keeps cost down. If both passes produce zero Critical/Major findings, the debrief is ready. If findings are accepted and changes are made, the changes are applied but the pass is not re-run — the user's acceptance is sufficient.

---

## Phase 4: Finalize

### Step 1: File naming

Present the default filename and let the user accept or customize:

```
I'll save this as `notes/debrief/{slug}-{YYYY-MM-DD}.md`.
Options:
  1. Use this name
  2. Enter a custom name
```

The slug is derived from the meeting topic or series name, lowercased and hyphenated.

### Step 2: Write the file

Write to `notes/debrief/{chosen-filename}.md`. Create the directory if it doesn't exist.

### Step 3: Terminal summary

```
## Debrief Complete

Output:     {absolute path}
Sources:    {N} transcript(s)
Date range: {earliest} to {latest}
Decisions:  {count}
Actions:    {count} ({open} open, {complete} complete)
Open items: {count}

Critic passes: 2 (extraction-check, claim-verification)
Findings:      {N critical}, {N major} addressed

Ready for review.
```

---

## Edge Cases

- **Single transcript**: Works normally. Skip chronological ordering. If `--single` is passed, also skip the Timeline Changes section.
- **Very long transcripts**: Read in chunks if needed. Do not skip content — the critic will catch omissions, but it can't catch what was never read.
- **Transcripts with no clear decisions**: This happens. The debrief should reflect it: "No explicit decisions recorded." The open questions and risks sections become more important in this case.
- **Transcripts in different formats**: One `.md` and one `.pdf` is fine. Both get read and normalized.
- **Contradictory information within a single meeting**: If someone says X and later says not-X in the same meeting, capture the final position and note the reversal.
- **Unidentifiable speakers**: If the transcript doesn't identify speakers, action items can't have owners. Note this: "Owner not identifiable from transcript." Don't guess.

---

## Notes

- This is a compression problem. The input is long and noisy. The output must be short and lossless on load-bearing information. The critic passes test the compression quality.
- Opus synthesizes (expensive, done once). Sonnet critiques (cheaper, two passes). This is the opposite of `/pm:assess` where the critic starts with Opus — here the synthesis is the hard part, not the critique.
- The user's corrections after the draft (before critic passes) are the most valuable input. They know what matters. Don't skip the "anything obviously wrong?" step.
- Unresolved disagreements are the highest-value section. Summaries that smooth over disagreements create a false sense of consensus. Flag them.
- "No action items assigned" and "No explicit decisions recorded" are valid outputs. Not every meeting produces decisions. Forcing decisions that weren't made is a hallucination.
- Deduplication is critical. The same point discussed for 20 minutes across three people should appear once in the debrief, not three times.
