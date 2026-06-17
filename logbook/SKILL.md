---
name: logbook
description: Observability layer for Claude activity — write structured log entries and query them. Used by Loop, Dream, and Cultivate to record what happened. Use /logbook today, /logbook errors, /logbook ticket <ID>, etc. to inspect activity.
triggers:
  - /logbook
  - /logbook today
  - /logbook yesterday
  - /logbook errors
  - /logbook ticket
  - /logbook project
  - /logbook write
---

# Logbook

Records and queries Claude activity logs. One JSONL file per project per day.

## Storage layout

```
~/.claude/logbook/
  <project-name>/
    2026-06-16.jsonl    ← active logs, one per day
    archive/
      2026-03-01.jsonl  ← rotated after 90 days
```

`<project-name>` = the basename of the directory that contains `TICKETS.md`. If no TICKETS.md is found, use the CWD basename.

---

## Step 0 — Resolve project name

Run before every command:

```bash
# Walk up to find TICKETS.md, use its parent dir's basename
TICKETS=$(find . -maxdepth 1 -name "TICKETS.md" 2>/dev/null \
  || find .. -maxdepth 1 -name "TICKETS.md" 2>/dev/null \
  || find ../.. -maxdepth 1 -name "TICKETS.md" 2>/dev/null)

if [ -n "$TICKETS" ]; then
  PROJECT=$(basename "$(dirname "$TICKETS")")
else
  PROJECT=$(basename "$PWD")
fi

LOG_DIR="$HOME/.claude/logbook/$PROJECT"
TODAY=$(date -u +%Y-%m-%d)
LOG_FILE="$LOG_DIR/$TODAY.jsonl"
```

---

## `logbook.write()` — shared write helper

**This is the canonical way for any skill to write a log entry.**  
Loop, Dream, and Cultivate should copy this snippet and fill in their values.

```bash
# Resolve project name (Step 0 above)
LOG_DIR="$HOME/.claude/logbook/$PROJECT"
LOG_FILE="$LOG_DIR/$(date -u +%Y-%m-%d).jsonl"
mkdir -p "$LOG_DIR"

# Build and append the entry (replace values as needed)
python3 -c "
import json, sys
entry = {
  'ts':        '$(date -u +%Y-%m-%dT%H:%M:%SZ)',
  'source':    'loop',
  'project':   '$PROJECT',
  'event':     'ticket.done',
  'ticketId':  'SL-001',
  'details':   {'filesChanged': ['path/to/file.ts'], 'blastRadius': 'auto', 'branch': 'loop/SL-001-slug', 'commitHash': 'abc123'},
  'durationMs': 42000,
  'error':     None
}
print(json.dumps(entry))
" >> "$LOG_FILE"
```

**Caller fills in:** `source`, `event`, `ticketId` (if relevant), `details` (if relevant), `durationMs`, `error`.  
`ts` and `project` are always auto-filled.  
**Fail silently** — wrap in `|| true` so logging never crashes the parent skill.

### Event quick-reference for callers

| Caller | Event to use | Key details fields |
|---|---|---|
| Loop — ticket picked | `ticket.started` | `branch` |
| Loop — ticket committed | `ticket.done` | `filesChanged`, `commitHash`, `branch`, `blastRadius` |
| Loop — ticket staged | `ticket.staged` | `filesChanged`, `branch`, `blastRadius`, `reason` |
| Loop — ticket rejected | `ticket.rejected` | `reason` |
| Dream — cycle complete | `dream.run` | `memoriesConsolidated`, `skillsUpdated`, `digestPath` |
| Cultivate — skill promoted | `skill.created` | `skillName`, `skillPath` |
| Any — unhandled error | `error` | `message`, `ticketId` (if relevant) |

---

## `/logbook today`

Show a human-readable summary of today's Claude activity for the current project.

1. **Resolve** project name and `$LOG_FILE` (Step 0).
2. **Check:** if `$LOG_FILE` doesn't exist, print "No activity logged today for **[project]**." and stop.
3. **Run rotation** (see Rotation section below) — lazy, runs on every invocation.
4. **Parse** each line of `$LOG_FILE`:

```bash
python3 - <<'EOF'
import json, sys
from collections import defaultdict

entries = []
with open(sys.argv[1]) as f:
    for line in f:
        line = line.strip()
        if line:
            try:
                entries.append(json.loads(line))
            except json.JSONDecodeError:
                pass

# Group by event type
by_event = defaultdict(list)
for e in entries:
    by_event[e.get('event','unknown')].append(e)

print(f"── Logbook: {entries[0]['project']} — {sys.argv[2]} ──")
print()

# Tickets
ticket_events = ['ticket.started','ticket.done','ticket.staged','ticket.rejected']
ticket_entries = [e for e in entries if e.get('event') in ticket_events]
if ticket_entries:
    print("Tickets:")
    seen = set()
    for e in ticket_entries:
        tid = e.get('ticketId', '—')
        evt = e.get('event','').replace('ticket.','')
        if tid not in seen:
            seen.add(tid)
            details = e.get('details', {})
            files = details.get('filesChanged', [])
            commit = details.get('commitHash', '')
            extra = f"  {commit[:7]}" if commit else (f"  {len(files)} file(s)" if files else "")
            print(f"  [{tid}] {evt}{extra}")

# Files changed
all_files = []
for e in entries:
    all_files.extend(e.get('details', {}).get('filesChanged', []))
if all_files:
    print()
    print(f"Files changed: {len(set(all_files))}")

# Duration
total_ms = sum(e.get('durationMs', 0) for e in entries)
if total_ms:
    mins = total_ms // 60000
    secs = (total_ms % 60000) // 1000
    print(f"Time spent:    {mins}m {secs}s")

# Errors
errors = [e for e in entries if e.get('event') == 'error' or e.get('error')]
if errors:
    print()
    print(f"Errors: {len(errors)}")
    for e in errors:
        msg = e.get('error') or e.get('details', {}).get('message', '(no message)')
        print(f"  ✗ {msg}")

EOF
"$LOG_FILE" "$TODAY"
```

---

## `/logbook yesterday`

Identical to `/logbook today` but for yesterday's log file.

```bash
YESTERDAY=$(date -u -v-1d +%Y-%m-%d 2>/dev/null || date -u -d "yesterday" +%Y-%m-%d)
YESTERDAY_FILE="$LOG_DIR/$YESTERDAY.jsonl"
```

If the file doesn't exist: "No activity logged yesterday for **[project]**."  
Then run the same parse-and-print logic as `today`.

---

## `/logbook errors`

Scan the last 7 days of logs for error events.

```bash
python3 - <<'EOF'
import json, os, sys
from datetime import datetime, timedelta

log_dir = sys.argv[1]
project = sys.argv[2]
errors = []

for i in range(7):
    day = (datetime.utcnow() - timedelta(days=i)).strftime('%Y-%m-%d')
    path = os.path.join(log_dir, f"{day}.jsonl")
    if not os.path.exists(path):
        continue
    with open(path) as f:
        for line in f:
            line = line.strip()
            if not line:
                continue
            try:
                e = json.loads(line)
                if e.get('event') == 'error' or e.get('error'):
                    errors.append((day, e))
            except json.JSONDecodeError:
                pass

if not errors:
    print(f"No errors in the last 7 days for {project}.")
else:
    print(f"── Errors ({project}, last 7 days) ──")
    for day, e in errors:
        tid = e.get('ticketId', '—')
        msg = e.get('error') or e.get('details', {}).get('message', '(no message)')
        ts  = e.get('ts', day)[:16].replace('T',' ')
        print(f"  {ts}  [{tid}]  {msg}")

EOF
"$LOG_DIR" "$PROJECT"
```

---

## `/logbook ticket <ID>`

Trace the full lifecycle of a single ticket across all log files.

1. Prompt for ticket ID if not provided as an argument.
2. Scan all JSONL files in `$LOG_DIR` (not archive) for entries where `ticketId == ID`.
3. Print each matching entry in chronological order:

```
── Ticket SL-003 lifecycle ──
  2026-06-16 09:26  ticket.started   branch: loop/SL-003-slate-pick-done
  2026-06-16 09:32  ticket.done      files: 1  commit: abc123f  duration: 6m
```

If no entries found: "No log entries for ticket [ID]."

---

## `/logbook project <name>`

Show aggregate activity for a named project (across all dates).

1. If `<name>` not provided, list all available project names:

```bash
ls "$HOME/.claude/logbook/"
```

2. Set `LOG_DIR="$HOME/.claude/logbook/<name>"`. If directory doesn't exist: "No logs found for project [name]."

3. Scan all JSONL files in the directory (not archive). Aggregate:
   - Total tickets worked (unique ticket IDs with `ticket.started`)
   - Total tickets done / staged / rejected
   - Total files changed
   - Total time spent
   - Any errors

4. Print a compact summary:

```
── Logbook: slate (all time) ──
Tickets:    5 started  /  4 done  /  1 staged  /  0 rejected
Files:      12 unique files changed
Time:       1h 23m
Errors:     0
Log range:  2026-06-16 → 2026-06-16 (1 day)
```

---

## Log rotation

Run automatically at the start of every `/logbook` command invocation.

```bash
ARCHIVE_DIR="$LOG_DIR/archive"
mkdir -p "$ARCHIVE_DIR"

# Move files older than 90 days
ROTATED=0
find "$LOG_DIR" -maxdepth 1 -name "*.jsonl" | while read f; do
  fname=$(basename "$f" .jsonl)
  # Compare date: if file date < (today - 90 days)
  cutoff=$(date -u -v-90d +%Y-%m-%d 2>/dev/null || date -u -d "90 days ago" +%Y-%m-%d)
  if [[ "$fname" < "$cutoff" ]]; then
    mv "$f" "$ARCHIVE_DIR/"
    ROTATED=$((ROTATED + 1))
  fi
done

if [ "$ROTATED" -gt 0 ]; then
  echo "Note: $ROTATED log file(s) archived (>90 days old)."
fi
```

---

## `/logbook write` — manual entry

For writing an entry by hand (testing, manual milestones).

Ask the user:
- Source (default: `manual`)
- Event type (show the list)
- Ticket ID (optional)
- Details as free text (optional)

Format and append using `logbook.write()`. Print: "Entry written to [LOG_FILE]."

---

## Error handling

| Situation | Response |
|---|---|
| No log file for the requested date | "No activity logged [today/yesterday] for [project]." |
| Malformed JSONL line | Skip it silently |
| Log directory doesn't exist | Create it; "No logs yet for [project]." |
| Project not found via TICKETS.md | Fall back to CWD basename, note it |
