---
name: cultivate
description: Learnings-to-skills pipeline — extracts recurring patterns from memory and Logbook, surfaces skill candidates for human review, and promotes approved ones to ~/.claude/skills/. Run /cultivate after a productive session to grow your skill library organically.
triggers:
  - /cultivate
  - /cultivate review
  - /cultivate promote
  - /cultivate purge
---

# Cultivate

Turns session learnings into reusable skills. Extracts patterns from memory files and Logbook, writes draft candidate skills, and lets you promote approved ones into `~/.claude/skills/`.

---

## Storage layout

```
~/.claude/skills/
  _candidates/
    REVIEW.md            ← human-facing queue of pending candidates
    slate-pick-order.md  ← a draft candidate skill file
    ...
  loop/
    SKILL.md             ← promoted skills live here
```

---

## `/cultivate`

Run pattern extraction, write candidates, print the review queue.

### Step 1 — Scan feedback memories

```bash
MEMORY_DIR="$HOME/.claude/projects/$(echo "$PWD" | sed 's|/|-|g')/memory"
# Also scan global memory
GLOBAL_MEMORY="$HOME/.claude/projects/-Users-$(whoami)/memory"
```

Read all `feedback_*.md` files in both directories. For each file, extract the core rule (the leading sentence before the `**Why:**` line).

**Detect duplicates:** group files by semantic similarity — same instruction phrased differently. If ≥ 2 feedback memories contain the same rule, flag as a candidate for a new or updated skill.

### Step 2 — Scan Logbook signals

```bash
LOG_DIR="$HOME/.claude/logbook"
# Scan all projects' last 14 days of logs
find "$LOG_DIR" -name "*.jsonl" -newer "$(date -d '14 days ago' +%Y-%m-%d 2>/dev/null || date -v-14d +%Y-%m-%d).jsonl" 2>/dev/null
```

For each log file, find:
- **Repeated errors:** same `error.message` pattern appearing ≥ 3 times → candidate for a guard skill
- **Repeated context loads:** same file path appearing in `details.filesChanged` every session → candidate for a context-preloading skill
- **Ticket patterns:** tickets that consistently go to `ticket.staged` (blast radius triggered) → candidate for a blast radius rule update

### Step 3 — Write candidates

For each detected pattern, create a draft skill file in `~/.claude/skills/_candidates/`:

```markdown
---
name: candidate-slug
description: One-line description of what this skill does
metadata:
  type: candidate
  surfaced: 2026-06-16
  reason: "Feedback memory correction appeared 3 times: 'never use git add -A'"
  status: pending
---

# [Candidate] Slug

## What this skill would do

[Description of the behaviour to encode]

## Draft instructions

[The actual skill content — what Claude should do]

## Source evidence

- feedback_stale_server_and_git_add.md — "Never use git add -A"
- feedback_git_branch_conventions.md — "Always stage specific files"
```

### Step 4 — Update REVIEW.md

```markdown
# Candidate Skills — Review Queue

Last updated: 2026-06-16

## Pending

| # | Candidate | Why surfaced | Date |
|---|---|---|---|
| 1 | [always-stage-specific-files](always-stage-specific-files.md) | Feedback memory rule seen 3×: stage specific files, never -A | 2026-06-16 |
| 2 | [preload-tickets-context](preload-tickets-context.md) | TICKETS.md loaded in every Loop session | 2026-06-16 |

## Approved
(moved to ~/.claude/skills/ — no longer listed here)

## Rejected
| Candidate | Reason rejected | Date |
|---|---|---|
```

### Step 5 — Print summary

```
── Cultivate ───────────────────────────────────
Scanned:  12 feedback memories, 7 days of logs
New candidates: 2
  → always-stage-specific-files (from feedback memories)
  → preload-tickets-context (from Logbook patterns)

Run /cultivate review to step through them.
────────────────────────────────────────────────
```

If no new candidates: "No new patterns detected. Your skill library looks current."

---

## `/cultivate review`

Step through pending candidates one at a time.

For each candidate in REVIEW.md with `status: pending`:

1. Print the candidate file content (name, reason surfaced, draft instructions).

2. Ask:
   ```
   [1/N] always-stage-specific-files
   Reason: Feedback memory rule seen 3 times.

   (a) Approve — promote to ~/.claude/skills/
   (b) Reject — remove candidate
   (e) Edit — open draft for editing before approving
   (s) Skip — leave as pending
   >
   ```

3. On **approve:** call `/cultivate promote <name>` (see below).

4. On **reject:**
   - Remove the candidate file from `_candidates/`
   - Move entry to the `## Rejected` table in REVIEW.md with today's date and prompt for a reason

5. On **edit:**
   - Open the candidate file for editing (Read + ask user to provide edits)
   - After editing, ask again: approve or reject?

6. On **skip:** leave unchanged, move to next candidate.

7. After all candidates: print a completion summary.

---

## `/cultivate promote <name>`

Promote a candidate skill to active use.

1. Find `~/.claude/skills/_candidates/<name>.md`. If not found: "Candidate [name] not found." Stop.

2. Check if a skill with the same name already exists in `~/.claude/skills/`:
   - If yes: ask "A skill named [name] already exists. Overwrite or merge? (overwrite/merge/cancel)"
   - On **overwrite:** replace the existing skill file
   - On **merge:** open both files side by side, ask the user to describe the merge, apply it
   - On **cancel:** stop

3. Copy the candidate file to `~/.claude/skills/<name>/SKILL.md` (create the directory).

4. Remove the candidate file from `_candidates/`.

5. Update REVIEW.md: move entry from Pending to Approved, add promotion date.

6. Log to Logbook: `event: skill.created`, `details: { skillName: name, skillPath: "~/.claude/skills/name/SKILL.md" }`.

7. Print: "✓ Promoted: [name] → ~/.claude/skills/[name]/SKILL.md"

---

## `/cultivate purge`

Remove stale candidates that have sat in the queue for more than 30 days without action.

```bash
CUTOFF=$(date -u -v-30d +%Y-%m-%d 2>/dev/null || date -u -d "30 days ago" +%Y-%m-%d)
```

1. Scan REVIEW.md for candidates with `surfaced` date older than the cutoff and `status: pending`.
2. List them and ask: "These candidates are older than 30 days. Remove them? (yes/no)"
3. On yes: delete the files, move entries to Rejected in REVIEW.md with reason "purged (30+ days stale)".
4. Print: "Purged N stale candidates."

---

## Skill health check (passive — runs during `/cultivate`)

During Step 2, also check:

```bash
# Find skills not referenced in any Logbook entry in the last 30 days
SKILLS_DIR="$HOME/.claude/skills"
for skill in "$SKILLS_DIR"/*/; do
  name=$(basename "$skill")
  # Skip _candidates and known meta-skills
  [[ "$name" == "_candidates" ]] && continue
  # Search logs for this skill name
  found=$(grep -r "\"skillName\":\"$name\"" "$HOME/.claude/logbook/" 2>/dev/null | wc -l)
  # Also check if it's referenced in recent Logbook
done
```

If a skill has zero log references in the last 30 days, add a note to the console output:
```
Note: These skills haven't been used in 30+ days — consider archiving:
  - old-skill-name (last seen: never in logs)
```

These are surfaced as observations only. Cultivate never auto-archives skills.

---

## Error handling

| Situation | Response |
|---|---|
| No memory files found | "No memory files found. Run a session first." |
| No Logbook entries found | "No Logbook entries. Start using Loop to build up signals." |
| _candidates/ doesn't exist | Create it silently |
| Candidate file malformed | Skip it, note in output |
| Skill already exists on promote | Ask: overwrite / merge / cancel |
