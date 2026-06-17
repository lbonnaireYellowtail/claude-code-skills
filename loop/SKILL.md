---
name: loop
description: Agent execution engine — picks the next Slate ticket, plans the work, executes it, and auto-commits or stages for human approval based on blast radius. One ticket per context window. Use /work to start, /work approve or /work reject to handle staged changes.
triggers:
  - /work
  - /work status
  - /work approve
  - /work reject
---

# Loop

Turns a Slate ticket board into a working developer. Picks one ticket, plans, executes, then commits or stages — all within one bounded context window.

**One ticket per run. One branch per ticket. One context window per branch.**

---

## Pre-flight checks

Before every `/work` run:

```bash
# 1. Confirm we're in a git repo
git rev-parse --git-dir > /dev/null 2>&1 || { echo "Not a git repo. Loop requires git."; exit 1; }

# 2. Confirm TICKETS.md exists
find . -maxdepth 2 -name "TICKETS.md" | head -1 || { echo "No TICKETS.md found. Run /slate init first."; exit 1; }

# 3. Check for an existing staged branch (Loop state)
STAGED_BRANCH=$(git stash list --format="%s" | grep "^loop-staged:" | head -1 | sed 's/loop-staged: //')
```

If a staged branch exists, say:
> A change is already staged for review: **[branch name]**  
> Run `/work status` to see it, `/work approve` or `/work reject` to handle it before starting a new ticket.

---

## `/work` — pick, plan, execute

### Step 0.5 — Load project standards

Before picking a ticket, load the project's active rules so they inform the plan and execution.

```bash
STANDARDS_FILE=$(find . -maxdepth 2 -name "STANDARDS.md" | head -1)
```

If `STANDARDS.md` exists:

1. Read the file line by line.
2. For each line starting with `@`, resolve the referenced file:
   ```bash
   rule_file="${line:1}"   # strip leading @
   [ -f "$rule_file" ] && cat "$rule_file"
   ```
3. Collect all resolved content (inline rules + referenced file content) into a single **Active Standards** block.
4. Print a one-line acknowledgement:
   ```
   Standards loaded: 7 rule files + 3 project-specific rules
   ```

If no `STANDARDS.md` found: print "No STANDARDS.md — proceeding without project standards." and continue (non-blocking).

**These standards are active for the entire ticket execution.** Reference them when:
- Choosing between implementation approaches
- Deciding file placement and naming
- Writing tests (coverage thresholds, file naming)
- Evaluating whether a change respects architectural boundaries (e.g. NX lib dependency rules)

If the ticket's acceptance criteria would require violating a standard, stop and ask the user before proceeding.

---

### Step 1 — Pick a ticket

Call Slate's pick logic (follow the `/slate pick` instructions from the Slate skill):
- Read `## Ready` in TICKETS.md, take the first ticket
- If no ready tickets: "No ready tickets. Add one with `/slate new`." Stop.
- Move ticket to `## In Progress`

Extract: `TICKET_ID`, `TICKET_TITLE`, `PRIORITY`, `PARENT`, acceptance criteria lines.

Log to Logbook:
```bash
# event: ticket.started, ticketId: $TICKET_ID, details: { branch: (not yet created) }
```

### Step 2 — Create a branch

```bash
SLUG=$(echo "$TICKET_TITLE" | tr '[:upper:]' '[:lower:]' | sed 's/[^a-z0-9]/-/g' | sed 's/--*/-/g' | cut -c1-40)
BRANCH="loop/${TICKET_ID}-${SLUG}"
CURRENT_BRANCH=$(git branch --show-current)
git checkout -b "$BRANCH"
```

### Step 3 — Plan

Before writing any code, produce a brief implementation plan:

```
── Plan: [TICKET_ID] ───────────────────────────
Files to read:   (list the files needed for context)
Files to change: (list the files that will be modified)
Approach:        (2–3 sentences on what will change and why)
Blast radius:    (auto | approval-required — reason)
────────────────────────────────────────────────
Proceed? (auto-proceeds if blast radius = auto)
```

**Read only the files listed.** Do not read the whole repo.

If the plan would touch more than **10 files**: pause and ask the user to confirm before proceeding.

### Step 4 — Classify blast radius

Apply these rules in order. First match wins.

| Condition | Decision |
|---|---|
| Only test files (`*.spec.*`, `*.test.*`) | Auto |
| Only docs / comments / markdown | Auto |
| Only files in a single isolated component or module | Auto |
| Multiple files across modules, no shared API change | Auto |
| Any change to a public API surface | Approval |
| Any change to `package.json`, lock files | Approval |
| Any change to config files (`.env`, `*.config.*`, `*.yaml`, `*.json` at root) | Approval |
| Any change to shared library code (used by >1 project) | Approval |
| Any DB schema or migration file | Approval |
| Any shell command with `rm`, `drop`, `delete`, `truncate` | Approval |
| More than 10 files changed | Approval |

**Auto path → proceed to Step 5.**  
**Approval path → go to Step 6 (stage).**

### Step 5 — Execute and auto-commit

Execute the plan. For each file change, use the Edit or Write tool.

After all changes:

```bash
# Stage specific files only — never git add -A
git add path/to/file1.ts path/to/file2.ts   # list each file explicitly

# Confirm branch is not protected
BRANCH=$(git branch --show-current)
if [[ "$BRANCH" == "main" || "$BRANCH" == "master" || "$BRANCH" == "development" ]]; then
  echo "ERROR: cannot commit to protected branch $BRANCH"
  exit 1
fi

git commit -m "[${TICKET_ID}] ${TICKET_TITLE}"
```

Mark ticket done via Slate's `done` logic (check all AC boxes, move to Done section).

Log to Logbook:
```bash
# event: ticket.done
# details: { filesChanged: [...], commitHash: $(git rev-parse --short HEAD), branch: $BRANCH, blastRadius: "auto" }
# durationMs: (elapsed since Step 1)
```

Print summary:
```
✓ Done: [SL-NNN] Ticket title
  Branch:  loop/SL-NNN-slug
  Commit:  abc123f
  Files:   3 changed
```

### Step 6 — Stage for approval (high blast radius)

Do NOT commit. Instead:

```bash
# Stash the changes with a loop-staged marker
git stash push -m "loop-staged: $BRANCH" --include-untracked
```

Move the ticket to `## Blocked` in TICKETS.md with:
```
**Blocked by:** awaiting human approval — blast radius: [reason]
```

Log to Logbook:
```bash
# event: ticket.staged
# details: { filesChanged: [...], branch: $BRANCH, blastRadius: "approval-required", reason: "..." }
```

Print:
```
⚠ Staged for review: [SL-NNN] Ticket title
  Blast radius: [reason]
  Run /work approve or /work reject to continue.
```

---

## `/work status`

Show what Loop is currently doing / what needs attention.

```bash
# Check for staged stash
STAGED=$(git stash list --format="%s" 2>/dev/null | grep "^loop-staged:" | head -1)
# Check in-progress ticket
IN_PROGRESS=$(awk '/^## In Progress/{s=1} /^## Ready/{s=0} s && /^### \[/{print; exit}' TICKETS.md 2>/dev/null)
```

Print:
```
── Loop Status ──────────────────────────
Staged for review : [SL-NNN] Ticket title  ← or "(none)"
In progress       : [SL-NNN] Ticket title  ← or "(none)"
Ready             : N tickets
─────────────────────────────────────────
```

---

## `/work approve`

Commit a staged change.

1. Check for a staged stash: `git stash list | grep "loop-staged:"`. If none: "Nothing staged for approval." Stop.
2. Extract the branch name from the stash message.
3. Confirm with the user: "Approve and commit **[branch]**? (yes/no)"
4. On yes:
```bash
git stash pop
git add <files from stash>   # re-stage the specific files
git commit -m "[TICKET_ID] TICKET_TITLE (approved)"
```
5. Move ticket from Blocked → Done in TICKETS.md.
6. Log: `event: ticket.done`, `details: { blastRadius: "approval-required" }`.
7. Print: "✓ Approved and committed: [branch]"

---

## `/work reject`

Discard a staged change and put the ticket back in Ready.

1. Check for a staged stash. If none: "Nothing staged to reject." Stop.
2. Confirm: "Reject and discard **[branch]**? This cannot be undone. (yes/no)"
3. On yes:
```bash
git stash drop
git checkout "$CURRENT_BRANCH"
git branch -D "$BRANCH"
```
4. Move ticket from Blocked → Ready in TICKETS.md (remove the "awaiting approval" blocked-by line).
5. Log: `event: ticket.rejected`, `details: { reason: "human rejected" }`.
6. Print: "✗ Rejected: [branch] — ticket [ID] returned to Ready."

---

## Context discipline

- **Read only what the plan says.** Before execution, list the files to read. Read those. Nothing else.
- **One ticket per invocation.** `/work` exits after one ticket is committed or staged.
- **Never open `/work` for a second ticket** in the same session if one is staged.
- If you find yourself needing more than 10 files to implement a ticket, the ticket is too big — say so, move it back to Ready, and suggest using `/slate slice` to break it up.

---

## Error handling

On any unhandled error during execution:

1. Attempt to restore the branch state: `git checkout "$CURRENT_BRANCH" && git branch -D "$BRANCH" 2>/dev/null || true`
2. Move the ticket back to `## Ready` in TICKETS.md.
3. Log: `event: error`, `details: { message: "...", ticketId: "$TICKET_ID" }`.
4. Print: "✗ Error during [TICKET_ID]: [message]. Ticket returned to Ready."

| Situation | Action |
|---|---|
| No TICKETS.md | "Run /slate init first." Stop. |
| No ready tickets | "No ready tickets. Add one with /slate new." Stop. |
| Ticket AC would violate a standard | Stop and ask user before executing. Do not auto-proceed. |
| `@` reference in STANDARDS.md not found | Log warning, continue with remaining rules. |
| Staged change exists | "Resolve staged change first with /work approve or /work reject." Stop. |
| On protected branch | Switch to a safe branch before starting. |
| Plan requires >10 files | Ask for confirmation. Suggest /slate slice if user declines. |
| Commit fails | Log error, restore branch, return ticket to Ready. |
