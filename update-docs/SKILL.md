---
name: update-docs
description: Update README files in yellowtail-libraries (branch / pre-merge) and matching Confluence pages (master-only / post-merge) to reflect code changes. README and Confluence are two separate passes — they are never run together. Detects scope from the git diff, or accepts a component name. Always confirms before writing.
triggers:
  - /update-docs
  - update docs
  - update readme
  - update confluence
---

# Update Docs

Keeps documentation in sync with code in `yellowtail-libraries`. There are **two separate passes** that are **never run together**, because they target different points in the lifecycle:

| Pass | Source of truth | When | Default |
|---|---|---|---|
| **README** | the current branch (`HEAD`) | during PR work, **before merge** | ✅ default |
| **Confluence** | `origin/master` only | **after the PR merges** | opt-in (`--confluence`) |

**Why separate:** READMEs ship inside the PR, so they're written against the branch. Confluence is the *published* reference — it must only ever reflect what's actually released on `master`. Updating Confluence from an unmerged branch would publish docs for code that doesn't exist yet.

## Invocation

- `/update-docs` — **README pass** (default). Scope from the current branch diff.
- `/update-docs <component>` — README pass for a named component (e.g. `/update-docs button`).
- `/update-docs --confluence` — **Confluence pass**. Reads `origin/master` only; refuses if the relevant changes aren't merged yet (see Step C0).
- `/update-docs --confluence <component>` — Confluence pass for a named component.

Pick exactly one pass per invocation. If the user asks for "both", do the README pass now and tell them to run `--confluence` after the PR merges.

---

# README pass (default)

## Step R1 — Detect scope

If a component name was passed, use it. Otherwise:

```bash
git diff HEAD --name-only
git branch --show-current
git log origin/master..HEAD --oneline
```

Files under `libs/component-library-angular/src/lib/atoms/<name>/` → component = `<name>`. Use the commit messages to understand *what* changed (API shape, new inputs, removed types, behaviour).

## Step R2 — Find relevant READMEs

For each changed component `<name>`, check **both**:

1. **Component-level README** — `libs/component-library-angular/src/lib/atoms/<name>/README.md` (primary API doc).
2. **Library-level README** — `libs/component-library-angular/README.md` (only if the change is library-wide: new component in the Components table, icon renderer change, theming contract change, etc.).

```bash
find /Users/louisbonnaire/design-systems/yellowtail-libraries/libs -name "README.md"
```

A README is relevant if it references the changed component's selector (`yt-<name>`), input names, type names, or import paths in the diff.

## Step R3 — Identify stale content

Compare README content against the **branch** source:

- **Input table** — cross-check against `<component>.component.ts` signal inputs.
- **Type definitions** — any removed types still documented?
- **Code examples** — do inline templates use the current API?
- **Import statements** — reference removed exports?
- **Components table** (library README only) — is the new component listed?

## Step R4 — Propose & apply

Show before/after for each stale section. Never apply silently:

```
atoms/button/README.md — 2 stale sections found:
1. Input table: missing `spinnerIcon` row   [before/after]
2. Enums block: ButtonVariant missing Ghost [before/after]
Apply all? (yes / review each / no)
```

Process component-level READMEs first, then the library README. These are normal repo edits — they get committed as part of the branch/PR (follow the repo's commit conventions).

---

# Confluence pass (`--confluence`)

## Step C0 — Merge gate (MANDATORY — do this first)

Confluence reflects **`origin/master` only**. Before reading or writing anything:

```bash
git fetch origin master --quiet
git log origin/master..HEAD --name-only   # commits/files on this branch NOT yet on master
```

- If the targeted component's changes are **already in `origin/master`** → proceed; treat `origin/master` as the source of truth (read source via `git show origin/master:<path>`).
- If the branch has **unmerged** changes touching the targeted component (they appear in the `origin/master..HEAD` output) → **STOP**. Do not update Confluence. Tell the user:

  > Confluence is only updated after a PR merges. The changes for `<component>` aren't in `origin/master` yet — re-run `/update-docs --confluence <component>` once the PR lands.

Never read the working tree or `HEAD` for the Confluence pass. Always `git show origin/master:<path>`. When the source code's own README on master is itself stale, trust the **source code**, not the README, and note the README nit separately.

## Step C1 — Discover relevant pages

```
searchConfluenceUsingCql:
  cql: 'type = page AND text ~ "<component-name>"'
  limit: 10
```

If no results, try broader terms (e.g. "button API", "component library"). Show matching pages with titles + space names, then ask:

```
Found N Confluence pages that may reference this component:
  1. [Page title] — Space: <space> (updated: <date>)
Check all for stale content? (yes / select / no)
```

## Step C2 — Read & audit each selected page

`getConfluencePage` (full body). Look for: selector usage (`yt-<name>`), changed input/prop names, removed types, dead import paths, old-API snippets, version strings, component tables, roadmap statuses.

## Step C3 — Propose & apply

Show what changes per page; confirm before writing:

```
[Page title] — 2 stale sections:
1. [content]="{ label: 'Submit' }" → label="Submit"
2. ButtonContent no longer exported
Update this page? (yes / skip)
```

If confirmed, `updateConfluencePage` with a clear `versionMessage`.

**Format tip:** for pages with status lozenges / panels / layouts, fetch and write in `html` (round-trip safe). Plain content can use `markdown` — but markdown table cells escape `|` to `\|`, which renders a literal backslash; if a type cell needs a pipe (`IconData | null`), write that page in `html` instead.

---

## Summary (either pass)

```
## Update Docs — <README | Confluence> pass — Summary

### Updated (N)
  <file or page> — <what changed>
### Unchanged (N)
  ...
### Skipped / gated (N)
  <page> — user skipped  |  <component> — not yet on master
### Nothing to update
  [if no drift found]
```

---

## Notes

- Never update docs without showing a before/after diff first.
- Never run the README and Confluence passes in the same invocation.
- The Confluence merge gate (Step C0) is non-negotiable — if unsure whether changes are merged, treat them as unmerged and stop.
- If Confluence/Atlassian access is unavailable, report it and stop the Confluence pass.
- When the diff is ambiguous (refactor with no API change), say so and confirm scope first.
- README improvements that aren't strictly stale → mention as suggestions, not in the "stale" list.
