---
name: notes
description: Capture, refine, and file notes per project. Paste raw notes; the skill runs a short refinement Q&A, weaves in insights from earlier notes in the same project, writes a clean markdown file, and offers to turn action points into tickets. Standalone command — works in any mode.
triggers:
  - /notes
  - notes
  - take notes
  - note this
  - capture notes
---

# Notes

Turn a raw brain-dump into a clean, filed note — refined through a short Q&A, connected to earlier notes, and optionally converted into tickets.

This is a **standalone command** that works in any mode. **Do not change the active mode header** (`**[FE] Frontend Dev**` / `**[AG] Agent Building**`) when running a notes session.

## Core rules

- **Notes are per project.** They live inside the relevant project repo at `<project>/notes/`, never in a global folder.
- **Refine before writing.** Never dump the raw paste to disk. Run the Q&A, then write the cleaned-up version.
- **Distinguish action points from brainstorming.** Only action points become ticket candidates.
- **Insights come from the same project's earlier notes** — surface real connections, don't invent them.

## Input

The user invokes `/notes` and usually pastes the raw notes after it. If no notes were pasted, ask them to paste the brain-dump now.

## Flow

### 1. Resolve the project

Notes are stored per project, so first decide **which project** these notes belong to.

1. Determine the working project from the current directory — the nearest git root, or the enclosing project folder.
2. If the cwd is clearly inside one project, use it (state which one).
3. If the cwd is ambiguous (e.g. `~`) or not a project, **ask which project** these notes are for. Offer the user's known active projects (check `MEMORY.md` "Active Projects") plus "somewhere else (give a path)".

Then ensure the notes location exists:

```bash
mkdir -p "<project>/notes"
```

The store layout per project:

```
<project>/notes/
  INDEX.md                       ← one line per note, newest first
  YYYY-MM-DD-<slug>.md           ← one file per notes session
```

### 2. Scan earlier notes for context

Before refining, read `<project>/notes/INDEX.md` (if it exists) and skim any notes that look related to the current paste. Use this to:

- Spot **recurring themes** (the same idea coming up again).
- Find **notes that connect** to the new material.
- Notice **earlier action points** that this note advances, contradicts, or completes.

Keep this lightweight — you'll weave the relevant links into the note and mention them during refinement. Don't dump the old notes back at the user.

### 3. Refinement Q&A (adaptive, batched)

Analyze the raw paste and, before writing, resolve what's unclear:

- Identify **gaps, ambiguities, and vague references**.
- Provisionally tag each item as an **action point** (concrete, ownable, has a next step) or **brainstorming** (exploratory, no owner or next step yet).
- Note anything that connects to earlier notes (from step 2).

Then ask **one batched round of 3–6 clarifying questions** covering the most important gaps — grouped in a single message, not one at a time. Typical questions:

- What's the goal / why behind item X?
- Is Y an action to take, or just an idea to hold?
- Any deadline, owner, or dependency on Z?
- Does this relate to `<earlier note>`? (when a connection is likely)

Only run **additional rounds** if the notes are very rough after the first pass. If everything is already clear, skip straight to writing (light touch — don't manufacture questions).

After the round, produce the refined structure and **briefly confirm** with the user before writing.

### 4. Write the note file

Compute the date at runtime:

```bash
date +%Y-%m-%d
```

Write `<project>/notes/YYYY-MM-DD-<slug>.md` (slug = short kebab-case topic) using this template:

```markdown
---
title: <Note title>
date: YYYY-MM-DD
project: <project name>
tags: [<topic>, <topic>]
status: captured   # captured | ticketed | archived
---

# <Note title>

## Summary
One or two lines on what this note is about.

## Context
Background, motivation, and any relevant detail from the refinement Q&A.

## Refined notes
The cleaned-up, structured content (headings / bullets as needed).

## Action points
- [ ] Concrete, ownable next steps only. Empty if none.

## Brainstorming / open questions
- Exploratory ideas and unresolved questions that are NOT yet actions.

## Insights & connections
- Links to related earlier notes: [<title>](./YYYY-MM-DD-<slug>.md) — how they connect.
- Recurring themes noticed across notes.

## Tickets
(Added in step 5 if tickets are created — id + link per action point.)
```

Then update `<project>/notes/INDEX.md` — create it if missing — prepending one line (newest first):

```markdown
- [YYYY-MM-DD — <Note title>](./YYYY-MM-DD-<slug>.md) — <one-line hook> · tags: <topic>, <topic>
```

If `INDEX.md` is new, start it with a `# Notes Index` heading, then the list.

### 5. Offer to create tickets

Only relevant if the note has **action points**. Brainstorming items are never auto-ticketed — if the user wants to promote one, treat it as an action point first.

1. List the action points found, numbered.
2. Ask which to turn into tickets (all / a subset / none) and **where** — ask each time:

   ```
   Action points found: <n>
   Create tickets in?
     → Slate (local TICKETS.md)
     → Jira
     → Trello
     → Skip
   ```

3. Route to the matching skill/tool:
   - **Slate** → use the `slate` skill (`/slate new`) to add tickets to the project's `TICKETS.md`.
   - **Jira** → use the `ronan-workflow-create-jira-ticket` skill (restate ticket target/scope and confirm before creating, per the Jira workflow rules).
   - **Trello** → use the `ronan-trello` skill.
4. After creation, write the ticket ids/links back into the note's **## Tickets** section and set the note's `status:` to `ticketed`.

If the user picks Skip, leave the note as `captured` and finish.

## Finishing

End with a short recap: where the note was saved, how many action points, which became tickets, and any notable connection to earlier notes.
