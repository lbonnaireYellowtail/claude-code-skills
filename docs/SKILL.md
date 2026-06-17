---
name: docs
description: Manage all Anchor DS documentation — create ADRs, process considerations, generate component specs, sync Jira statuses, convert todos, and run a full doc update sweep. Routes to the right sub-flow based on the subcommand.
triggers:
  - /docs
  - /doc-adr
  - new adr
  - create decision record
  - write adr
  - /doc-consider
  - resolve consideration
  - close consideration
  - move consideration
  - /doc-spec
  - create component spec
  - new component spec
  - /doc-sync
  - sync jira
  - sync refinements
  - /doc-todo
  - process todos
  - refine docs todo
  - /doc-update
  - update docs
  - sync docs
---

# Docs

Unified documentation skill for the Anchor DS (`component-library`). Routes to the correct sub-flow based on the subcommand.

## Shared Paths

```
DOCS_ROOT=/Users/louisbonnaire/design-systems/component-library/docs
REPO=/Users/louisbonnaire/design-systems/yellowtail-libraries
DECISIONS=$DOCS_ROOT/decisions/
REFINEMENTS=$DOCS_ROOT/refinements/
ROADMAP=$DOCS_ROOT/components-roadmap/
TODOS=$DOCS_ROOT/todos.md
CONSIDERATIONS=$DOCS_ROOT/considerations.md
SETTLED=$DOCS_ROOT/settled.md
TEMPLATE=$ROADMAP/_template.md
```

## Routing

| Command | What it does |
|---|---|
| `/docs adr` or `/doc-adr` | Create a new Architecture Decision Record |
| `/docs consider` or `/doc-consider` | Move a resolved consideration to settled.md or an ADR |
| `/docs spec` or `/doc-spec` | Generate a new component spec from the template |
| `/docs sync` or `/doc-sync` | Pull live Jira statuses and sync refinement docs |
| `/docs todo` or `/doc-todo` | Convert a todos.md item into a refinement doc or ADR |
| `/docs update` or `/doc-update` | Full sweep — update all docs from git + Jira |
| `/docs` (no subcommand) | Show this menu and ask which sub-flow to run |

---

## `/docs adr` — New Architecture Decision Record

Creates a well-formed ADR in `$DECISIONS/` with the next available D-XX number.

### Input
- `/docs adr` — interactive: ask for topic, then guide through each section
- `/docs adr <topic>` — start with topic pre-seeded
- `--from-consideration` — called internally by `/docs consider`; skip intro prompt

### Step 1 — Assign next D-XX number
```bash
ls $DECISIONS/D-*.md | grep -oE 'D-[0-9]+' | sort -t- -k2 -n | tail -1
```
Increment by 1. If no files exist, start at D-01.

### Step 2 — Gather topic context
1. `grep -rl "<topic>" $DECISIONS/` — find related existing ADRs (read up to 3)
2. `grep -rl "<topic>" $DOCS_ROOT/` — check considerations or settled for related references

Surface related ADRs before writing so the new record can reference them.

### Step 3 — Prompt for each section (one at a time)

**Title** — short noun-first phrase (e.g. "ViewEncapsulation strategy for components")  
**Context** — problem, constraints, alternatives on the table  
**Decision** — the choice made, one paragraph max  
**Alternatives considered** — at least 2 options with reason for rejection  
**Consequences** — positive outcomes + negative trade-offs  
**Code example** (optional) — before/after or canonical snippet  
**Status** — default `Accepted`. Other: `Rejected`, `Deferred`, `Superseded by D-XX`  
**Applies to** — component name, layer, or `all components`

### Step 4 — Write the ADR

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

### Step 5 — Confirm and suggest follow-ups
```
Created: decisions/D-XX-<name>.md
Related ADRs referenced: D-XX, D-XX

Suggested next steps:
  - Link this ADR from any affected component spec
  - If this supersedes an older decision, update that file's Status field
  - If this resolves an open consideration, run /docs consider to close it
```

---

## `/docs consider` — Process a Consideration

Closes out an open item in `considerations.md`, routing it to `settled.md` or a new ADR.

### Input
- `/docs consider` — list open items and let the user pick
- `/docs consider <keyword>` — filter by keyword and process the match

### Step 1 — Read open items
Read `$CONSIDERATIONS`. Extract every item without `Status: Decided` or `Status: Closed`. Display as a numbered list.

### Step 2 — Classify the resolution
```
How was this resolved?
  1. Simple resolution → settled.md (practical choice, no deep rationale needed)
  2. Architectural decision → new ADR in decisions/
  3. Deferred → update the status note in considerations.md
  4. Dropped → mark as closed with a note
```

### Step 3a — If settled
Append to `$SETTLED`:
```markdown
---

### [Item title]

**Resolution:** [One sentence summary]
**Date:** YYYY-MM-DD
**Status:** Settled

[Original context, condensed to 2–3 sentences]
```
Mark original item in `$CONSIDERATIONS`: `**Status:** Settled — see settled.md`

### Step 3b — If ADR
Call the `/docs adr` flow with `--from-consideration`, passing the item title, body, and stated resolution as seeds. Then update `$CONSIDERATIONS`: `**Status:** Decided — see decisions/D-XX-<name>.md`

### Step 3c — If deferred
```
**Status:** Deferred — [reason / waiting on what]
**Updated:** YYYY-MM-DD
```

### Step 3d — If dropped
```
**Status:** Dropped — [brief reason]
**Date:** YYYY-MM-DD
```

### Step 4 — Confirm
```
Updated: considerations.md — item [N] marked as [Settled/Decided/Deferred/Dropped]
[Created: decisions/D-XX-<name>.md]
[Updated: settled.md]
Remaining open items: N
```

---

## `/docs spec` — New Component Spec

Generates a fully pre-filled component spec in `$ROADMAP/<tier>/<component>.md`.

### Input
- `/docs spec <component>` — e.g. `/docs spec avatar`
- `/docs spec` — show existing specs, ask which component to create

### Step 1 — Check for existing spec
Look for `$ROADMAP/**/<component>.md`. If found, ask: open for review, overwrite, or cancel.

### Step 2 — Determine atomic tier
```
1. atom — simple, single-responsibility presentational (no internal state)
2. molecule — composed of 2+ atoms, may have local interaction state
3. organism — complex, composed of molecules, may emit domain events
```
Reference D-12 (atomic design decision).

### Step 3 — Gather context (run in parallel)
- **3a.** Find existing implementation: `find $REPO/libs -name "<component>.component.ts"`
- **3b.** Read baseline ADRs: D-06, D-07, D-08, D-12, D-16, D-22 (plus any matching the component domain)
- **3c.** Read 1–2 similar existing specs of the same tier to match conventions
- **3d.** Note any known Figma URL if available

### Step 4 — Draft the spec (fill every section)
| Section | Source |
|---|---|
| Purpose | Component name + tier context |
| API — inputs | From implementation if exists; infer from similar components |
| API — outputs | EventEmitter pattern (e.g. `clicked`, `changed`) |
| API — slots | `ng-content` slots if applicable |
| Visual variants | Default + domain-specific variants |
| Sizes | `sm`, `md`, `lg` unless size-agnostic |
| States | disabled, loading, error — mark N/A where not applicable |
| Accessibility | WCAG 2.2 role, aria-* attributes, keyboard pattern |
| Tokens consumed | Map D-08 layers: primitive → semantic → component |
| Dependencies | Other components this one composes |
| Infrastructure required | Wave classification (C1–C6) |
| Stories | One per variant/state combination |
| Open questions | Surface genuine ambiguities |

### Step 5 — Write the file
`$ROADMAP/<tier>/<component>.md`. Set wave based on infrastructure required (no deps → Wave 0/1; form infrastructure → Wave 2; overlay/portal → Wave 3+).

### Step 6 — Confirm
```
Created: components-roadmap/<tier>/<component>.md
Sections pre-filled from: [sources used]
Open questions: [any genuine ambiguities]
Next: Review the spec, resolve OQs, then run /docs todo to create a refinement doc when ready to implement.
```

---

## `/docs sync` — Sync Jira Statuses

Fetches current Jira statuses for all refinement docs and updates `**Status:**` fields where they've drifted.

### Input
- `/docs sync` — sync all refinement docs
- `/docs sync <YFE-XXX>` — sync one specific ticket

### Step 1 — Collect tickets
Scan `$REFINEMENTS/YFE-*.md`. Extract ticket ID, current `**Status:**`, and `**Branch:**` field.

### Step 2 — Fetch Jira statuses (in parallel)
```bash
curl -s -u "$JIRA_EMAIL:$JIRA_TOKEN" -H "Accept: application/json" \
  "$JIRA_BASE_URL/rest/api/3/issue/YFE-XXX?fields=status,resolution,assignee,updated"
```

### Step 3 — Determine doc status from Jira
| Jira status | Doc status |
|---|---|
| To Do / Backlog | `Draft` |
| In Progress / Executing | `Ready` (if impl plan exists) or `Draft` |
| Done / Closed / Resolved | `Implemented` |
| Won't Do / Rejected | `Cancelled` |

Only update if the doc status differs.

### Step 4 — Check for merged branches
```bash
cd $REPO && git fetch origin --quiet && git branch -r --merged origin/master
```
If a doc's `**Branch:**` is in the merged list and status is not `Implemented` → flag for update.

### Step 5 — Show diff and confirm
```
## Doc-Sync Preview
  YFE-131  Ready → Implemented  (Jira: Done, branch: merged)
  YFE-133  Draft → Implemented  (Jira: Done)
  No change needed: YFE-130 (Jira: In Progress, doc: Draft ✓)
```
Ask: "Apply these updates? (yes / no)"

### Step 6 — Apply changes
Edit `**Status:**` in-place. Also update `**Date:**` to today's date.

---

## `/docs todo` — Convert a Todo Item

Converts open items from `todos.md` into refinement docs or decision records.

### Input
- `/docs todo` — show open items, let user pick
- `/docs todo <search>` — filter by keyword or Jira ID

### Step 1 — Read the todo list
Extract every unchecked line (`- [ ]`). Group into:
- **Refinements** — items with a Jira ticket ID (`YFE-\d+`)
- **Decisions** — items describing an architectural or design choice

### Step 2 — Determine output type
| Signal | Output |
|---|---|
| Jira ticket ID present | Refinement doc → `$REFINEMENTS/YFE-XXX-name.md` |
| Words like "decide", "pattern", "convention", "approach" | ADR → `$DECISIONS/D-XX-name.md` |
| Both apply | Create both; link them from each other |

Check whether the doc already exists before creating. If a partial doc exists, update it rather than overwrite.

### Step 3 — Gather context (in parallel)
1. **Fetch Jira ticket** (if applicable) — title, description, acceptance criteria, comments
2. **Read the relevant code** — search for files related to the todo subject
3. **Check existing docs** — scan `$REFINEMENTS/` and `$DECISIONS/` for related prior art

### Step 4 — Analyse before writing
- What is the current state in the code?
- What does the todo want to achieve?
- What decisions have already been made?
- Any blockers or unanswered questions? → surface before writing

### Step 5 — Write the document

**Refinement doc:**
```markdown
# YFE-XXX — [Story title]

**Status:** Draft | Ready | Implemented
**Branch:** feature/yfe-xxx-name (if known)
**Jira:** [link]
**Date:** YYYY-MM-DD

## Scope
[2–4 sentences]

### In scope
- [concrete deliverable]

### Out of scope
- [explicitly excluded]

## Implementation Plan
### 1. [First step]
### 2. [Second step]

## Files to Change
| File | Change |
|---|---|

## Open Questions
- [ ] [Question]

## Acceptance Criteria
- [ ] [from Jira]

## Testing
[How to verify: unit tests, Storybook stories, playground sections]
```

**Decision record format:** follow the ADR format from `/docs adr`.

### Step 6 — Update todos.md
Change `- [ ]` to `- [x]` and append the created doc path: `— see refinements/YFE-XXX-name.md`

### Step 7 — Confirm
Show what was created, any remaining open questions, and whether the next step is `/docs sync` or ready to run `/implement`.

---

## `/docs update` — Full Doc Sweep

Keeps all component-library docs in sync by scanning git history and cross-checking each doc type.

### Input
- `/docs update` — scan last 14 days
- `/docs update --since <date>` — e.g. `/docs update --since 2026-05-01`
- `/docs update --full` — scan entire git history

### Step 1 — Collect recent changes
```bash
cd $REPO
git log --oneline --name-only --since="14 days ago" --format="%h %s" | head -200
git branch -r --merged origin/master
```
Build a list of changed library files and merged branches (matched to ticket IDs).

### Step 2 — Update refinement doc statuses
For each `$REFINEMENTS/YFE-*.md` with status `Draft` or `Ready`:
- Check if branch is in merged list
- Batch-fetch Jira status
- If Done/Closed in Jira OR branch merged → update `**Status:**` to `Implemented`

### Step 3 — Update component roadmap specs
For each spec in `$ROADMAP/atoms/`, `$ROADMAP/molecules/`, `$ROADMAP/organisms/`:
- Check if implementation exists: `find $REPO/libs/.../src -name "<component>.component.ts"`
- If built and spec still shows `Status: Spec` → note as "built, spec not updated" (do NOT auto-edit)

### Step 4 — Sync todos.md
For each unchecked item with a Jira ID, fetch status. If Done/Closed → mark `[x]` and append `— closed in Jira`.

### Step 5 — Flag potentially stale ADRs
For each changed file, check if its component prefix appears in any ADR title. Heuristic: look for token overrides, new injection tokens, new component flags that may contradict a decision.

Surface as:
```
⚠ Potentially stale ADRs — review manually:
  D-22 (contract-based theming) — button size override added in YFE-137
```
Do NOT auto-edit ADRs.

### Step 6 — Report and confirm
```
## Doc-Update Summary — <date>

### Refinements updated (N)
  YFE-134 — Draft → Implemented (branch merged)

### Component specs with built implementations not yet marked Built
  atoms/spinner.md

### Todos synced (N)
  T1 (YFE-130) — still open in Jira

### ADRs to review (N)
  D-22, D-23 — see details above
```

Ask: "Apply refinement and todo changes? (yes / review each / no)"
