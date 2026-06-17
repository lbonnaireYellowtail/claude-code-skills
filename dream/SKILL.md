---
name: dream
description: Nightly maintenance system — consolidates memory files, runs Cultivate's pattern extractor, rotates logs, and writes a morning digest. Runs at 02:00 via cron. Can also be invoked manually with /dream. Never deletes anything outright — always archives.
triggers:
  - /dream
  - /dream status
  - /dream digest
---

# Dream

Runs while you sleep. Keeps memory clean, compounds learnings into skills, and leaves a digest for the morning. One cron job, zero maintenance required.

**Dream never deletes. It moves things to `_archive/` and logs everything it does.**

---

## Run sequence

```
01. Acquire lock (check Loop is not running)
02. Run Cultivate extractor (generate skill candidates from today's activity)
03. Consolidate memory files (per project + global)
04. Clean _candidates/ (purge 30+ day stale rejecteds)
05. Run Logbook rotation (archive logs older than 90 days)
06. Write nightly digest
07. Log Dream run to Logbook
08. Release lock
```

If any step fails: log the error, skip that step, continue with the rest. Dream never halts early.

---

## Step 01 — Acquire lock

```bash
LOCK_FILE="$HOME/.claude/.dream-lock"

# Check Loop's lock first
LOOP_LOCK=$(find . -maxdepth 3 -name ".loop-lock" 2>/dev/null | head -1)
if [ -n "$LOOP_LOCK" ]; then
  echo "Loop is running. Waiting up to 5 minutes..."
  for i in $(seq 1 30); do
    sleep 10
    LOOP_LOCK=$(find . -maxdepth 3 -name ".loop-lock" 2>/dev/null | head -1)
    [ -z "$LOOP_LOCK" ] && break
    [ "$i" -eq 30 ] && { echo "Loop still running after 5 min. Skipping Dream run."; exit 0; }
  done
fi

# Acquire Dream's own lock
if [ -f "$LOCK_FILE" ]; then
  echo "Dream already running (stale lock?). Exiting."
  exit 0
fi
touch "$LOCK_FILE"
trap "rm -f $LOCK_FILE" EXIT
```

---

## Step 02 — Run Cultivate extractor

Follow the `/cultivate` instructions (Steps 1–4 only — extract patterns and write candidates, do NOT open the interactive review). Dream surfaces candidates; the human reviews them in the morning.

Log: how many candidates were surfaced.

---

## Step 03 — Consolidate memory files

For each project active in the last 24 hours (has a Logbook entry today):

```bash
MEMORY_DIR="$HOME/.claude/projects/$(echo "$PROJECT_PATH" | sed 's|/|-|g')/memory"
ARCHIVE_DIR="$MEMORY_DIR/_archive/$(date -u +%Y-%m-%d)"
mkdir -p "$ARCHIVE_DIR"
```

**Consolidation rules (apply in order):**

1. **Duplicate detection:** read all `feedback_*.md` files. If two or more contain the same core rule (same intent, different wording), merge them — keep the clearest wording, preserve `**Why:**` and `**How to apply:**` lines, move superseded files to `_archive/`.

2. **Superseded detection:** if a memory's rule is directly contradicted by a newer memory (by file modification date), move the older one to `_archive/`. Note it in the digest.

3. **Verbosity flag:** if a memory body exceeds 300 words, flag it in the digest for human review. Do NOT auto-trim.

4. **MEMORY.md sync:** verify every remaining memory file has an entry in `MEMORY.md`. Add missing entries. Remove entries pointing to files that no longer exist.

Log: `event: memory.updated`, `details: { memoriesConsolidated: N, memoriesArchived: N }`.

---

## Step 04 — Clean _candidates/

Move rejected candidates older than 30 days to `_candidates/_archive/`:

```bash
CANDIDATES_DIR="$HOME/.claude/skills/_candidates"
CUTOFF=$(date -u -v-30d +%Y-%m-%d 2>/dev/null || date -u -d "30 days ago" +%Y-%m-%d)

find "$CANDIDATES_DIR" -maxdepth 1 -name "*.md" | while read f; do
  surfaced=$(grep "surfaced:" "$f" | head -1 | sed 's/.*surfaced: //' | tr -d '"')
  status=$(grep "status:" "$f" | head -1 | sed 's/.*status: //' | tr -d '"')
  if [[ "$status" == "rejected" && "$surfaced" < "$CUTOFF" ]]; then
    mkdir -p "$CANDIDATES_DIR/_archive"
    mv "$f" "$CANDIDATES_DIR/_archive/"
  fi
done
```

Log: how many candidates were archived.

---

## Step 05 — Run Logbook rotation

Follow the Logbook rotation instructions: move files older than 90 days to `archive/`. Apply across all projects in `~/.claude/logbook/`.

Log: how many log files were rotated.

---

## Step 06 — Write nightly digest

Write to `~/.claude/logbook/<project>/digests/YYYY-MM-DD.md` for each active project.

**Digest format:**

```markdown
# Dream Digest — [Project] — 2026-06-16

## Loop activity today
- ✓ Committed: [SL-003] Write /slate pick and /slate done  (commit: abc123f)
- ✓ Committed: [SL-004] Write /slate new and /slate slice  (commit: def456a)
- ⚠ Staged for review: [LP-002] Blast radius classifier  (reason: API surface change)

## Staged — needs your attention
- [LP-002] Blast radius classifier — run /work approve or /work reject

## Cultivate candidates (2 new)
- always-stage-specific-files — from 3 feedback memories
- preload-tickets-context — from Logbook patterns
  Run /cultivate review to process them.

## Memory changes
- Merged 2 duplicate feedback memories → feedback_git_conventions.md
- Archived: feedback_old_pattern.md (superseded)

## Ticket board
- Ready: 1  |  In Progress: 0  |  Blocked: 1  |  Done: 4

## Errors
(none)

---
*Dream run: 2026-06-16T02:00:03Z — 8 steps completed, 0 skipped*
```

Skip projects with no activity today (no Logbook entry).

---

## Step 07 — Log Dream run

Append to each active project's Logbook:

```
event: dream.run
details: {
  memoriesConsolidated: N,
  skillsUpdated: N,
  logsRotated: N,
  digestPath: "~/.claude/logbook/<project>/digests/YYYY-MM-DD.md"
}
```

---

## Step 08 — Release lock

```bash
rm -f "$LOCK_FILE"
```

The `trap` from Step 01 handles this automatically even on error.

---

## `/dream status`

Show the most recent digest for the current project.

```bash
PROJECT=$(basename "$PWD")
DIGEST_DIR="$HOME/.claude/logbook/$PROJECT/digests"
LATEST=$(ls "$DIGEST_DIR"/*.md 2>/dev/null | sort | tail -1)
```

If no digest exists: "No Dream digest found for [project]. Dream may not have run yet."  
Otherwise: print the digest content.

---

## `/dream digest`

Alias for `/dream status`. Shows today's digest if present, otherwise the most recent one.

---

## Cron setup

Dream runs at 02:00 local time every night via Claude Code's `CronCreate` tool.

**Schedule:** `0 2 * * *`  
**Command:** `/dream`  
**Scope:** all projects explicitly listed in the cron config (not auto-discovered)

To add a project: add its path to the cron config. To remove: remove it. Dream only runs where you tell it to.

---

## Safety rules

| Rule | Reason |
|---|---|
| Never `rm` — always `mv` to `_archive/` | Accidental loss prevention |
| Never overwrite a memory file without archiving the old version | Preserve history |
| Never commit to project repos | Dream is read-only relative to code |
| Never run if Loop lock is present | Prevents TICKETS.md race conditions |
| Every step is individually error-wrapped | One bad step never kills the run |

---

## Error handling

Every step is wrapped — failure logs and skips, never halts:

```bash
run_step() {
  local name="$1"; shift
  "$@" 2>&1 || {
    echo "  ✗ Step failed: $name (continuing)"
    # logbook.write event:error details:{ message: "Dream step $name failed" }
  }
}

run_step "cultivate-extract"  step_cultivate_extract
run_step "memory-consolidate" step_memory_consolidate
run_step "candidates-clean"   step_candidates_clean
run_step "log-rotation"       step_log_rotation
run_step "write-digest"       step_write_digest
run_step "log-dream-run"      step_log_dream_run
```
