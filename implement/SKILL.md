---
name: implement
description: Implement a Jira story by following its refinement document. Checks for a refinement doc first and offers to refine inline if one doesn't exist.
triggers:
  - implement story
  - implement ticket
  - start implementation
  - /implement
---

# Implement Story

You are implementing a Jira story by following a previously created refinement document.

## Input

The user provides a **ticket ID** (e.g. `MC-1234`). If not provided, ask for it.

## Step 1 — Create Feature Branch

Before writing any code, create a new feature branch from the latest `development`:

```bash
git stash  # if there are uncommitted changes
git checkout development
git pull origin development
git checkout -b feature/[TICKET-ID]-[short-description]
git stash pop  # if changes were stashed
```

Branch naming convention: `feature/MC-1234-brief-description` (kebab-case, max 50 chars)

If the user is already on a feature branch for this ticket, skip this step.

## Step 2 — Refinement Check

Before writing any code, check whether a refinement document exists:

```
docs/refinement/[TICKET-ID]-refinement.md
```

### If the refinement doc EXISTS

Proceed to Step 2.

### If the refinement doc does NOT exist

Show this warning:

```
⚠️ No refinement document found for [TICKET-ID].

Implementing without refinement risks scope drift, missed edge cases, and rework.

Options:
  1. Run /refine [TICKET-ID] now to create a refinement doc, then come back to implement.
  2. Type "continue anyway" to proceed without refinement (not recommended).
```

Wait for the user's response:
- If they choose to refine: invoke the `refine` skill flow for this ticket.
- If they type "continue anyway": warn once more that this is not recommended, then proceed using only the Jira story details (fetch via the Jira API using the same pattern as `/view-story`).

## Step 3 — Read the Refinement Document

Read the full refinement document and extract:
- **Implementation Plan** — ordered steps
- **Files to Change** — the list of files and their specific changes
- **Scope Clarifications** — decisions already made (do not re-ask these)
- **Open Questions** — flag any unresolved ones before starting

If there are open questions in the refinement doc, surface them:

```
⚠️ Open questions remain in the refinement doc:
- [question 1]
- [question 2]

These should ideally be resolved before implementing. Continue anyway? (yes/no)
```

## Step 4 — Explore the Codebase

Before touching any files, explore to build context:
- Read the files listed in the refinement doc
- Understand existing patterns and conventions in those files
- Check the domain and library type for any new files to create

## Step 5 — Implement

Follow the implementation plan from the refinement doc **step by step**. For each step:
1. State which step you are on
2. Make the change
3. Confirm it's done before moving to the next

### Mandatory conventions (from CLAUDE.md — never skip)

- `#` prefix for ALL private members — never the `private` keyword
- `protected` for all template-bound properties/methods
- Class member order: `inject()` → constructor → variables (signals, computed, inputs, outputs) → methods (public → protected → `#private`)
- `computed`/`effect`: no braces for single-line expressions
- Strict TypeScript: no `any`, explicit return types, `readonly` for immutable properties
- New control flow in templates: `@if`, `@for`, `@switch`
- `ChangeDetectionStrategy.OnPush` on all components
- BEM naming for SCSS, no `::ng-deep`
- Full words only — no abbreviations (`column` not `col`, `index` not `idx`, etc.)
- `[ngClass]` for conditional classes — never `[class.foo]`

### For new files

- Use NX generators: `npx nx g @nx/angular:component --project=[lib]` or `npx nx g @nx/angular:service --project=[lib]`
- kebab-case filenames
- Mock data > 1 line → `test-util/[file].mock.ts`

## Step 6 — Tests

After implementation, add or update tests per the refinement doc's testing notes:

- Minimum 75% coverage
- AAA pattern
- One `describe` per public method: `#methodName()` for public, `private: #methodName()` for private
- Parameterized tests: `for...of` with inline arrays, no separate `const testCases`
- Run targeted test: `npx nx test [library-name]` — NEVER `npm run test` or `npm run qc`

## Step 7 — Requirements Traceability Table

**MANDATORY — show this before every commit, without exception.**

After implementation is complete, produce a requirements traceability table:

```
## Requirements Traceability — [TICKET-ID]

| Acceptance Criterion | Implementing File | Method / Signal |
|---|---|---|
| [criterion from ticket] | `path/to/file.ts:line` | `methodName()` |
| [criterion 2] | `path/to/file.ts:line` | `computedSignal` |
```

Do not substitute this with code coverage percentages. Show the actual mapping from requirement to code.

## Step 8 — Summary

After all steps are done, provide:

```
## Implementation Complete — [TICKET-ID]

**Steps completed:** [list]
**Files changed:** [list with brief description]
**Tests added/updated:** [list]
**Next:** Run `npx nx test [lib]` to verify, then create a PR.
```
