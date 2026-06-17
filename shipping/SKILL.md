---
name: shipping
description: Pre-push quality gate for mn-crp-frontend — runs lint + audit checks on affected libraries, scans for stale documentation, then pushes with explicit confirmation.
triggers:
  - /shipping
  - shipping
---

# Shipping

Pre-push checklist for `mn-crp-frontend`. Runs in three phases: quality gates, documentation
scan, then a guarded push. Never runs project-wide commands — only NX affected targets.

## Input

`/shipping` — no arguments needed. Derives everything from the current branch and git state.

---

## Phase 1 — Pre-flight

### 1.1 — Verify branch

```bash
BRANCH=$(git branch --show-current)
```

- If `BRANCH` is `master`, `development`, or empty → **stop**:
  ```
  ✗ You are on branch "$BRANCH". Shipping must run from a feature branch.
  ```
- Otherwise show:
  ```
  Branch: $BRANCH
  Base:   origin/development
  ```

### 1.2 — Uncommitted changes

```bash
git status --short
```

If there are uncommitted changes, show them and ask:
```
⚠ You have uncommitted changes. Commit them before shipping, or they will not be pushed.
Stage and commit now? (yes / no / list)
```

- **yes** → run `git diff --stat`, ask for a commit message, then:
  ```bash
  git add -p   # interactive staging — do NOT use git add .
  git commit -m "<message>"
  ```
- **no** → continue (uncommitted changes stay local)
- **list** → show `git diff --name-only`, then re-ask

### 1.3 — Commits to push

```bash
git log origin/development..HEAD --oneline
```

If empty → stop:
```
✗ No commits ahead of origin/development. Nothing to push.
```

Otherwise show the commit list.

---

## Phase 2 — Quality gates

Run all checks. **Stop and report at the first failure** — do not continue to push.

### 2.1 — Lint (affected only)

```bash
npx nx affected --target=lint --base=origin/development --head=HEAD
```

- Pass → `✓ Lint`
- Fail → show the errors, then:
  ```
  ✗ Lint failed. Fix the errors above before pushing.
  ```
  Stop.

### 2.2 — Prettier (check only, never auto-fix here)

```bash
npm run prettier:check 2>&1
```

- Pass → `✓ Prettier`
- Fail → show which files need formatting, then:
  ```
  ✗ Formatting issues found. Run `npm run prettier:write` to fix, then commit.
  ```
  Stop.

### 2.3 — Security audit

```bash
npm audit 2>&1
```

Always passes — runs for visibility only, never blocks the push. Show the summary line
(e.g. `found 0 vulnerabilities`) and mark `✓ Audit` regardless of findings.

---

## Phase 3 — Documentation scan

Get the full list of files changed on this branch:

```bash
git diff origin/development...HEAD --name-only
```

Work through each category below. Report findings — do **not** auto-edit files; flag what needs
human attention.

### 3.1 — New Angular components

Grep changed files for new `@Component` declarations:

```bash
git diff origin/development...HEAD --name-only | grep '\.component\.ts$' | while read f; do
  git show HEAD:"$f" 2>/dev/null | grep -q '@Component' && echo "$f"
done
```

For each new component file found, check `~/.claude/projects/-Users-louisbonnaire-MN-mn-crp-frontend/memory/visual-components.md`:
- If the component selector is **not** listed → flag:
  ```
  📄 New component not in visual-components.md: <selector> (<file>)
     → Update memory/visual-components.md if this component is reusable.
  ```

### 3.2 — Refinement doc

Extract the ticket ID from the branch name (pattern: `feature/MC-\d+`):

```bash
TICKET=$(git branch --show-current | grep -oE 'MC-[0-9]+')
```

If a ticket ID was found, check:

```bash
ls docs/refinement/${TICKET}-*.md 2>/dev/null
```

- Doc exists → `✓ Refinement doc present (docs/refinement/${TICKET}-*.md)`
- Doc missing → flag:
  ```
  📄 No refinement doc found for $TICKET.
     → Run /refine $TICKET to create one, or this is expected if the ticket was trivial.
  ```

### 3.3 — Wiki changes

```bash
git diff origin/development...HEAD --name-only | grep '^wiki/'
```

For each modified wiki file, check if it is referenced in the wiki index or any other wiki page:

```bash
WIKI_FILE=<filename without path>
grep -rl "$WIKI_FILE" wiki/ 2>/dev/null
```

If a wiki file was added but is not referenced anywhere → flag:
```
📄 New wiki file not referenced anywhere: <path>
   → Link it from an index page or remove it if it was added by mistake.
```

### 3.4 — Component library version bump

When component code changed, the owning package's `package.json` **must** have its version bumped.
Bump level follows the design-systems semver rule:

- **New component added** → bump the **minor** (`X.minor.X` in MAJOR.MINOR.PATCH).
- **Fix to an existing component** → bump the **patch** (`X.X.patch`).

Find component changes and the package they belong to, then check whether that `package.json`
version changed on this branch:

```bash
# Did any component source change?
git diff origin/development...HEAD --name-only | grep -E 'src/.*\.component\.(ts|html|scss)$'

# For each affected package.json, did its version line change?
git diff origin/development...HEAD --name-only | grep 'package\.json$' | while read pkg; do
  git diff origin/development...HEAD -- "$pkg" | grep -q '^[-+]  "version"' \
    && echo "✓ version bumped: $pkg" \
    || echo "✗ version NOT bumped: $pkg"
done
```

- Component changed **and** version bumped → `✓ Version bumped`.
- Component changed but version **not** bumped → flag:
  ```
  📄 Component code changed but <package.json> version was not bumped.
     → New component? bump minor. Fix to existing component? bump patch.
  ```
- If a new component was flagged in 3.1 but only the patch digit moved (or vice-versa), flag the
  mismatch so the bump level matches the change type.

### 3.5 — Documentation summary

After scanning, show a summary:

```
## Documentation scan

✓ No new unregistered components          (or list of flagged ones)
✓ Refinement doc present                  (or flag)
✓ No orphaned wiki files                  (or list of flagged ones)
✓ Version bumped for component changes    (or flag)
```

If flags were raised, ask:
```
Documentation issues flagged above. Address them now or continue anyway?
(fix / continue)
```

- **fix** → stop so the user can address them, then re-run `/shipping`
- **continue** → proceed to Phase 4

---

## Phase 4 — Push

### 4.1 — Pre-push summary

Show:
```
## Ready to push

Branch:  <branch> → origin/development
Commits: <n> commit(s)

<git log origin/development..HEAD --oneline>

Changed files:
<git diff origin/development...HEAD --stat | tail -5>

Quality gates: ✓ Lint  ✓ Prettier  ✓ Audit (informational)
```

### 4.2 — Guarded push

Ask explicitly:
```
Push <branch> to origin? (yes / no)
```

- **yes** →
  ```bash
  git push -u origin <branch>
  ```
  On success:
  ```
  ✓ Pushed. Create a PR with /bitbucket-pr create when ready.
  ```
- **no** → stop:
  ```
  Push cancelled. Branch stays local.
  ```

Do **not** push without an explicit "yes". Do not interpret silence or ambiguity as approval.

---

## Constraints (mn-crp-frontend specific)

- **Never** run `npm run qc`, `npm run test`, `npm run lint`, or any project-wide test command.
- **Never** run `git add .` or `git add -A` — stage files explicitly or use `git add -p`.
- **Never** force-push.
- Tests are not run here — they should already be passing from `/implement`. If the user wants
  to run tests first, they can run `npx nx test <library>` manually before invoking `/shipping`.
