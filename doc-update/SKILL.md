---
name: doc-update
description: Scan recent git commits and update all docs that have drifted — refinement statuses, component roadmap build status, todos.md, and stale ADRs. Run after a merge or sprint end to keep docs in sync with code.
triggers:
  - /doc-update
  - update docs
  - sync docs
---

# Doc-Update

Keeps the component-library docs in sync with the actual state of the codebase by scanning git history and cross-checking each doc type.

## Paths

```
DOCS_ROOT=/Users/louisbonnaire/design-systems/component-library/docs
REPO=/Users/louisbonnaire/design-systems/yellowtail-libraries
REFINEMENTS=$DOCS_ROOT/refinements/
DECISIONS=$DOCS_ROOT/decisions/
ROADMAP=$DOCS_ROOT/components-roadmap/
TODOS=$DOCS_ROOT/todos.md
```

## Input

- `/doc-update` — default: scan last 14 days
- `/doc-update --since <date>` — e.g. `/doc-update --since 2026-05-01`
- `/doc-update --full` — scan entire git history (slower, use after long absence)

---

## Step 1 — Collect recent changes

```bash
cd $REPO
git log --oneline --name-only --since="14 days ago" --format="%h %s" 2>/dev/null | head -200
```

Also fetch current branch list to detect merged branches:
```bash
git branch -r --merged origin/master 2>/dev/null
```

From the output, build a list of:
- Changed library files (path → component name)
- Merged branches (branch name → ticket ID if pattern matches `fix/yfe-XXX-*` or `feature/yfe-XXX-*`)

---

## Step 2 — Update refinement doc statuses

For each file in `$REFINEMENTS/YFE-*.md`:

1. Read the file, extract `**Status:**` and `**Branch:**` fields
2. If status is `Draft` or `Ready`:
   - Check if the branch is in the merged list from Step 1
   - Check Jira status (batch fetch if multiple):
     ```bash
     curl -s -u "$JIRA_EMAIL:$JIRA_TOKEN" -H "Accept: application/json" \
       "$JIRA_BASE_URL/rest/api/3/issue/YFE-XXX?fields=status"
     ```
   - If Jira status = Done/Closed OR branch is merged → update `**Status:**` to `Implemented`
3. Track all changes made

---

## Step 3 — Update component roadmap specs

For each component spec in `$ROADMAP/atoms/`, `$ROADMAP/molecules/`, `$ROADMAP/organisms/`:

1. Extract component name from filename (e.g. `avatar.md` → `avatar`)
2. Check if implementation exists in the repo:
   ```bash
   find $REPO/libs/component-library-angular/src -name "<component>.component.ts" 2>/dev/null
   ```
3. If implementation found and spec still shows `Status: Spec` or no status → note it as "built, spec not updated"
4. Report these gaps — do NOT auto-edit spec files (they may have deliberate open questions still to resolve)

---

## Step 4 — Sync todos.md

Read `$TODOS`. For each unchecked item with a Jira ID:

1. Fetch Jira status (batch where possible)
2. If Jira = Done/Closed → mark checkbox as `[x]` and append `— closed in Jira`
3. If a completed item has no Completed section → move it there

---

## Step 5 — Flag potentially stale ADRs

Scan changed files from Step 1. For each changed file, check whether:
- Its component prefix appears in any ADR title (e.g. `icon` → D-11, D-23, D-24)
- The change contradicts or extends a decision (heuristic: look for token overrides, new injection tokens, new component flags)

Do NOT auto-edit ADRs. Surface as a list:
```
⚠ Potentially stale ADRs — review manually:
  D-22 (contract-based theming) — button size override CSS vars added in YFE-137
  D-23 (icon registration) — icon revert behaviour changed in YFE-136
```

---

## Step 6 — Report

Show a structured summary:

```
## Doc-Update Summary — <date>

### Refinements updated (N)
  YFE-134 — status: Draft → Implemented (branch merged)
  YFE-135 — status: Ready → Implemented (Jira: Done)

### Component specs with built implementations (not yet marked Built)
  atoms/spinner.md — implementation found at libs/.../spinner.component.ts

### Todos synced (N)
  T1 (YFE-130) — still open in Jira

### ADRs to review (N)
  D-22, D-23 — see details above

### Nothing changed
  [any doc types with no drift]
```

Ask the user: "Apply refinement and todo changes? (yes / review each / no)"

- **yes** → apply all changes noted above
- **review each** → show each diff and ask per-file
- **no** → leave docs as-is, report only
