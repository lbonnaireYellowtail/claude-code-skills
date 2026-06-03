---
name: doc-adr
description: Author a new Architecture Decision Record (ADR). Auto-assigns the next D-XX number, prompts for context/decision/alternatives/consequences, and links to related existing ADRs. Use when formalising any architectural or design-system choice.
triggers:
  - /doc-adr
  - new adr
  - create decision record
  - write adr
---

# Doc-ADR

Creates a well-formed ADR in `docs/decisions/` with the next available D-XX number.

## Paths

Set these to match your project structure:

```
DOCS_ROOT=<path-to-your-docs-root>        # e.g. docs/ or design-system/docs
DECISIONS=$DOCS_ROOT/decisions/
```

## Input

- `/doc-adr` — interactive: ask for topic, then guide through each section
- `/doc-adr <topic>` — start with context pre-seeded from the topic keyword
- `/doc-adr --from-consideration` — called internally by `/doc-consider`; skip the intro prompt

---

## Step 1 — Assign next D-XX number

```bash
ls $DECISIONS/D-*.md 2>/dev/null \
  | grep -oE 'D-[0-9]+' | sort -t- -k2 -n | tail -1
```

Increment by 1. If no files exist, start at D-01.

---

## Step 2 — Gather topic context

If a topic was provided (or `--from-consideration` with a pre-filled topic):

1. `grep -rl "<topic>" $DECISIONS/` — find related existing ADRs
2. Read any matches (max 3) to understand what's already decided
3. `grep -rl "<topic>" $DOCS_ROOT/` — check if the consideration or settled doc references this topic

Surface related ADRs before writing so the new record can reference them.

---

## Step 3 — Prompt for each section

Ask in sequence (one at a time, or gather all upfront if the topic is clear):

**Title** — short phrase, noun-first (e.g. "ViewEncapsulation strategy for components")

**Context** — what problem prompted this decision? What constraints exist? What alternatives were on the table?

**Decision** — the choice made, stated plainly. One paragraph max.

**Alternatives considered** — at least 2 options with a brief reason why each was rejected (or deferred).

**Consequences** — positive outcomes + negative trade-offs or constraints introduced.

**Code example** (optional) — before/after or canonical usage snippet.

**Status** — default `Accepted`. Other values: `Rejected`, `Deferred`, `Superseded by D-XX`.

**Applies to** — component name, layer, or `all components`.

---

## Step 4 — Write the ADR

File: `$DECISIONS/D-XX-<kebab-title>.md`

```markdown
# D-XX — [Title]

**Status:** Accepted
**Date:** YYYY-MM-DD
**Applies to:** [scope]

---

## Context

[Context paragraph]

## Decision

[Decision paragraph]

## Alternatives considered

| Alternative | Why rejected |
|---|---|
| [option] | [reason] |

## Consequences

### Positive
- [benefit]

### Negative / trade-offs
- [cost or constraint]

## Code example

```typescript
// canonical usage or before/after
```

## Related decisions

- [[D-XX]] — [short description of relation]
```

---

## Step 5 — Confirm and suggest follow-ups

Show the created file path and:

```
Created: decisions/D-XX-<name>.md

Related ADRs referenced: D-XX, D-XX

Suggested next steps:
  - Link this ADR from any component spec that is affected
  - If this supersedes an older decision, update that file's Status field
  - If this resolves an open item in considerations.md, run /doc-consider to close it
```
