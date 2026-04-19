---
name: persona
description: |
  Create and manage executive personas for audience-aware communication skills.
  Personas capture what a reader knows, cares about, and how they read — used by exec-summary and other pm skills to calibrate output.
user-invocable: true
argument-hint: "[name] [--list] [--edit <slug>] [--delete <slug>]"
---

You are the orchestrator of an executive persona session. Your job is to build a concrete model of a specific reader — not a demographic profile, but a practical description of what this person knows, what they care about, and how they consume information. This model is used downstream by skills like `/pm:exec-summary` to calibrate what gets cut, what gets elevated, and how things are framed.

A good persona answers: "If I hand this document to this person, what will they understand without help, what will they skip, what will they immediately ask about, and what will make them stop reading?"

### Artifact Locations

All personas go under `notes/exec/personas/`:

- Persona files: `notes/exec/personas/{slug}.md`

---

## Setup

### Parse Arguments

- **No arguments** → enter **Create** mode. Start the interview.
- **Positional argument** (bare name, no flags) → enter **Create** mode with the name pre-filled. Start the interview at Step 2.
- `--list` → list all existing personas, then **stop**. Do not enter the interview.
- `--edit <slug>` → load the existing persona and enter **Edit** mode.
- `--delete <slug>` → confirm deletion with the user, delete the file if confirmed, then **stop**.

---

## List Mode

Read all `.md` files in `notes/exec/personas/`. For each file, extract the frontmatter fields and present:

```
## Existing Personas

| Slug | Name | Role | Created | Updated |
|------|------|------|---------|---------|
| {slug} | {name} | {role} | {created} | {updated} |
```

If no personas exist:

```
No personas found in notes/exec/personas/.

Run /pm:persona to create one.
```

---

## Create Mode

### The Interview

The goal is to build a persona through conversation, not a form. Ask questions that people can actually answer — "what makes them lean forward in a meeting?" not "list their priority vector." Batch related questions where natural, but don't dump everything at once.

#### Step 1: Who is this person?

```
Who is this persona for?

Give me their name (or a label like "CEO") and their role.
If you want to use a specific slug for referencing them later (e.g., --audience ceo), tell me — otherwise I'll derive one from the name.
```

Extract: **name**, **role/title**, **slug** (user-provided or derived — lowercase, hyphenated, short).

#### Step 2: What do they already know?

```
What does {name} already understand well — domains where you don't need to translate or explain?

And where do they need translation? What topics require plain language or context they don't have?
```

Extract: **domain expertise** (strong areas, needs-translation areas).

#### Step 3: What do they care about?

```
When {name} reads a document or sits in a meeting, what makes them pay attention? What are they always tracking — cost, speed, risk, competitive position, compliance, something else?

And what do they tend to skip or dismiss?
```

Extract: **priorities** (what they track, what they skip).

#### Step 4: How do they read?

```
How does {name} actually consume a document?

Some people read everything. Some read the first paragraph and the recommendation and skip the middle. Some jump to the numbers. Some won't read at all unless someone flags it as urgent.

What's their pattern?
```

Extract: **reading style**.

#### Step 5: What do they always ask?

```
When you present something to {name}, what questions do you know are coming? The ones you've learned to preemptively address?
```

Extract: **predictable questions**.

#### Step 6: What should we avoid?

```
Last one — anything that annoys {name} or that you've learned to avoid? Pet peeves, sensitivities, things that make them tune out or push back?

If nothing comes to mind, that's fine — we can skip this.
```

Extract: **sensitivities** (optional — only include if the user provides them).

#### Step 7: Decision authority

```
What can {name} actually approve or decide? This helps me frame recommendations correctly — "approve this" vs. "endorse this for escalation" are different asks.
```

Extract: **decision authority**.

### After the interview

Present a summary of what you captured:

```
## Persona Summary: {name}

**Slug:** {slug}
**Role:** {role}

**Knows well:** {strong domains}
**Needs translation:** {translation domains}

**Cares about:** {priorities}
**Skips:** {what they dismiss}

**Reading style:** {pattern}

**Will ask:** {predictable questions}

**Avoid:** {sensitivities, or "Nothing flagged"}

**Can decide:** {decision authority}

Anything to change or add before I save this?
```

Incorporate feedback. Then proceed to **Write**.

---

## Edit Mode

### Step 1: Load the persona

Read `notes/exec/personas/{slug}.md`. If the file doesn't exist, tell the user and offer to create it instead.

Present the current persona content to the user.

### Step 2: What changed?

```
What do you want to update? You can:
- Tell me what changed and I'll update the relevant sections
- Ask me to re-interview for a specific section (e.g., "re-do priorities")
- Paste new content to replace a section
```

### Step 3: Apply changes

Update the persona file in place. Update the `updated` field in frontmatter. Do not overwrite sections the user didn't change.

Present the updated persona and confirm:

```
Updated. Anything else to change?
```

---

## Write

### Step 1: Write the persona file

Create `notes/exec/personas/` if it doesn't exist. Write to `notes/exec/personas/{slug}.md`:

```markdown
---
name: {full name or label}
slug: {slug}
role: {role/title}
created: {YYYY-MM-DD}
updated: {YYYY-MM-DD}
---

## Domain expertise

**Understands well:**
- {strong domain 1}
- {strong domain 2}

**Needs translation:**
- {domain 1 — brief note on what kind of translation}
- {domain 2}

## Priorities

What they track:
- {priority 1}
- {priority 2}

What they skip or dismiss:
- {thing 1}

## Decision authority

{What they can approve, decide, or authorize. Be specific — dollar thresholds, scope of authority, what requires escalation above them.}

## Reading style

{How they consume documents. One short paragraph.}

## Predictable questions

- "{question 1}"
- "{question 2}"
- "{question 3}"

## Sensitivities

{What annoys them, what to avoid. If nothing was flagged, omit this section entirely.}
```

### Step 2: Terminal summary

```
## Persona Created

Output:   {absolute path}
Slug:     {slug}
Name:     {name}
Role:     {role}

Use with: /pm:exec-summary --audience {slug}
```

---

## Examples

```bash
# Create a new persona interactively
/pm:persona

# Create with a name pre-filled
/pm:persona "Sarah Chen"

# List existing personas
/pm:persona --list

# Edit an existing persona
/pm:persona --edit ceo

# Delete a persona
/pm:persona --delete ceo
```

---

## Notes

- The interview is the value. Most people can describe their CEO's reading habits conversationally but can't fill in a "reading style" field cold. The questions are designed to draw out concrete behaviors, not abstract categories.
- Personas are organizational knowledge. They accumulate over time and get refined as the user learns more about how someone reads and reacts. The `--edit` flow is as important as creation.
- Slugs should be short and memorable — `ceo`, `vp-eng`, `sarah`, `board`. They're typed in `--audience` flags.
- The "sensitivities" section is optional. Don't push for it. Some users won't want to write down that their CEO hates hedging — but many will, and it's high-value calibration.
- Personas are not personas in the marketing sense. They're not demographic archetypes. They're models of specific readers built from the user's direct experience with those people.
- "What they skip" is as valuable as "what they care about." If the VP of Engineering never reads the financial section, the exec summary shouldn't lead with cost.
