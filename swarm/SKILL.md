---
name: swarm
description: Multi-agent swarm — fans out one Loop agent per personal project that has ready tickets. Each agent runs in an isolated git worktree, picks one ticket, executes, and commits. Aggregates results into a swarm report.
triggers:
  - /swarm
  - /swarm status
  - /swarm dry
---

# Swarm

Orchestrates a fleet of Loop agents across all personal projects simultaneously. One agent per project, one ticket per agent, one worktree per agent. Context windows stay small. Results come back as a unified report.

**Inspired by:** Ronan Connolly's agent orchestration architecture — 100 small agents under 100k context beats 1 agent with 900k context.

**Context budget:** Every agent is hard-capped at 100k tokens total context. This is enforced by limiting what each agent is allowed to read and bailing early if the scope is too large. See the Context Budget section below.

---

## Commands

| Command | What it does |
|---|---|
| `/swarm` | Discover projects with ready tickets → fan out agents → report results |
| `/swarm dry` | Show which projects would run and which tickets they'd pick — no execution |
| `/swarm status` | Show the last swarm run summary from Logbook |

---

## `/swarm dry` — preview run

Run this before `/swarm` to see exactly what would happen.

### Step 1 — Discover projects

Scan for all `TICKETS.md` files across personal projects and AI26:

```bash
find "/Users/louisbonnaire/personal-projects" -maxdepth 2 -name "TICKETS.md" \
  -o -path "/Users/louisbonnaire/AI26/TICKETS.md" | sort
```

For each TICKETS.md found, extract:
- Project name (parent directory basename)
- Project path (parent directory)
- Prefix (from `<!-- prefix: XX -->`)
- Next ready ticket: first `### [` block under `## Ready`
- Count of ready tickets

### Step 2 — Filter eligible projects

A project is eligible if:
1. It has at least 1 ticket in `## Ready`
2. It has **no** ticket in `## In Progress` (already running — skip)
3. It is a git repository (`git -C $PROJECT_PATH rev-parse --git-dir`)

### Step 3 — Print dry-run table

```
── Swarm Dry Run (depth: 3) ────────────────────────────────────────
Project                 Next ticket                          Ready  Max
────────────────────────────────────────────────────────────────────
skill-hub               [SH-001] Wire catalogue to mock API   7     3
yellowtail-crosswords   [YC-001] Connect CMS to API route     5     3
performance-review      [PR-001] Update content for 2026      4     3
yellowtail-standards    [YS-001] Add npm run scripts          4     3
performance-review-form [PF-001] Scaffold feature-review      5     3
────────────────────────────────────────────────────────────────────
5 projects · up to 15 tickets · projects advance independently
Run /swarm to execute.  Run /swarm 5 to increase depth to 5.
```

If no eligible projects: "No projects have ready tickets. Add tickets with `/slate new`."

---

## `/swarm` — execute

### Step 1 — Discover and filter (same as dry-run)

Run the discovery logic above. Print the same preview table, then immediately proceed — do not wait for confirmation.

### Step 2 — Build and run the Workflow

Use the Workflow tool with `pipeline()` — NOT `parallel()`. This is the key design decision:

- `pipeline()` starts each project's next ticket **as soon as its previous one finishes** — no cross-project waiting
- `parallel()` would make every project wait for the slowest agent in the batch

**Max depth:** default 3 tickets per project per swarm run (configurable via `/swarm 5` etc). This caps runaway execution while letting fast projects get ahead of slow ones.

```
pipeline() behaviour:
  skill-hub      [SH-001 ──────] → [SH-002 ──────] → [SH-003 ──]  (fast project, gets 3 done)
  crosswords     [YC-001 ─────────────────] → [YC-002 ──────]      (slower, gets 2 done)
  perf-review    [PR-001 ──] → [PR-002 ──] → [PR-003 ──]           (fast, gets 3 done)
```

Each stage re-reads TICKETS.md to find the current next ready ticket — it does NOT assume which ticket comes next.

**Workflow script template:**

```javascript
export const meta = {
  name: 'swarm-run',
  description: 'Rolling pipeline — each project advances to next ticket as soon as current finishes',
  phases: [
    { title: 'Execute', detail: 'Rolling per-project pipelines, no cross-project waiting' },
    { title: 'Report', detail: 'Collect results into swarm summary' },
  ],
}

const MAX_DEPTH = args?.depth || 3  // max tickets per project per swarm run

// Build one agent prompt for a single ticket execution
function ticketAgentPrompt(projectPath, projectName) {
  return `You are a Loop agent working exclusively inside the project at ${projectPath}.
You must ONLY read and write files inside ${projectPath}. Never touch other projects.

## YOUR JOB
Pick the NEXT ready ticket from ${projectPath}/TICKETS.md and execute it.
Use the corrected awk to find the first ticket under ## Ready (stop scanning at next ## heading):
  awk '/^## Ready/{found=1; next} /^## /{found=0} found && /^### \\[/{print; exit}' TICKETS.md

If ## Ready is empty (no tickets), return immediately with status "no-tickets".

## CONTEXT BUDGET — READ THIS FIRST
Hard cap: 100k tokens total. Stay well under it.
- Max 8 files read, max 200k characters of file content (~50k tokens)
- No node_modules, dist, .git, lock files, generated files, files over 500 lines
- No recursive directory listings

## EXECUTION STEPS
Step 0.5: Read ${projectPath}/STANDARDS.md, resolve @ references.
Step 1: Find next ready ticket in TICKETS.md. Move to ## In Progress. Save file.
Step 2: Create branch: loop/<ticket-id-lowercase>-<slug> from current HEAD.
         git -C ${projectPath} checkout -b BRANCH
Step 3: Plan — list files to read, estimate sizes (wc -c / 4), confirm fits in budget. State approach + blast radius.
Step 4: Classify blast radius (first match wins):
  auto: test-only, docs-only, single isolated module, multiple files no shared API change
  approval: public API change, package.json/lockfile, root config, shared lib, >10 files
Step 5 (auto): Execute. Stage specific files by name only. Verify branch not protected. Commit.
  git commit -m "[TICKET_ID] Ticket title"
  Mark ticket done in TICKETS.md (check all boxes, move to ## Done).
Step 6 (approval): Stash ONLY the changed source files by name — never --include-untracked (that would swallow TICKETS.md):
  git -C PROJECT_PATH stash push -m "loop-staged: BRANCH" -- file1 file2 file3
  Move ticket to ## Blocked: "Blocked by: awaiting approval — blast radius: REASON"

## RETURN VALUE (JSON only, no placeholders)
{
  "project": "${projectName}",
  "ticketId": "<actual ID>",
  "ticketTitle": "<actual title>",
  "status": "done" | "staged" | "budget-exceeded" | "no-tickets" | "error",
  "branch": "<actual branch or null>",
  "commit": "<actual short hash or null>",
  "filesChanged": ["<actual paths>"],
  "blastRadius": "auto" | "approval-required",
  "reason": "<explanation if not done, else empty string>"
}`
}

const RESULT_SCHEMA = {
  type: 'object',
  properties: {
    project: { type: 'string' },
    ticketId: { type: 'string' },
    ticketTitle: { type: 'string' },
    status: { type: 'string', enum: ['done', 'staged', 'budget-exceeded', 'no-tickets', 'error'] },
    branch: { type: ['string', 'null'] },
    commit: { type: ['string', 'null'] },
    filesChanged: { type: 'array', items: { type: 'string' } },
    blastRadius: { type: 'string', enum: ['auto', 'approval-required'] },
    reason: { type: 'string' },
  },
  required: ['project', 'ticketId', 'status', 'blastRadius', 'filesChanged', 'reason'],
}

// Build pipeline stages — one per depth level
// Each stage: if previous result was no-tickets/error/budget-exceeded, skip (pass through)
// Otherwise run the next ticket for this project
const stages = Array.from({ length: MAX_DEPTH }, (_, i) => async (prevResult, project) => {
  // Skip if previous stage found nothing or errored
  if (prevResult && ['no-tickets', 'budget-exceeded', 'error'].includes(prevResult.status)) {
    return prevResult
  }
  const result = await agent(
    ticketAgentPrompt(project.path, project.name),
    { label: `loop:${project.name}:t${i + 1}`, phase: 'Execute', effort: 'low', schema: RESULT_SCHEMA }
  )
  return result
})

// pipeline() — each project advances independently, no cross-project barrier
const allResults = await pipeline(projects, ...stages)

// Flatten: each item is the last result for that project; collect all non-null terminal states
// But we want ALL results (each ticket), not just the last — so agents should accumulate
// For now collect the final result per project and log it
const results = allResults.filter(Boolean)
log(`All pipelines complete. Compiling report...`)
return results
```

### Step 3 — Write the Logbook entry

After the workflow completes, write a swarm run event to the Logbook:

```bash
python3 -c "
import json, sys
from datetime import datetime
entry = {
  'ts': datetime.utcnow().isoformat() + 'Z',
  'event': 'swarm.run',
  'results': $RESULTS_JSON
}
print(json.dumps(entry))
" >> ~/.claude/logbook/swarm/$(date +%Y-%m-%d).jsonl
```

### Step 4 — Print the swarm report

```
── Swarm Report ────────────────────────────────────────────────
Project                Ticket    Status    Files   Branch
──────────────────────────────────────────────────────────────
skill-hub              SH-001    ✓ done    4       loop/sh-001-...
yellowtail-crosswords  YC-001    ✓ done    2       loop/yc-001-...
performance-review     PR-001    ⚠ staged  6       loop/pr-001-...
yellowtail-standards   YS-001    ✓ done    1       loop/ys-001-...
performance-review-form PF-001   ✓ done    3       loop/pf-001-...
──────────────────────────────────────────────────────────────
4 done · 1 staged for approval · 0 errors · 0 budget-exceeded

Staged changes need approval:
  performance-review / PR-001 — blast radius: config file changed
  Run: cd ~/personal-projects/performance-review && /work approve

Tickets needing slicing:
  (none)
```

---

## `/swarm status`

Read the most recent swarm Logbook entry:

```bash
LATEST=$(ls ~/.claude/logbook/swarm/*.jsonl 2>/dev/null | sort | tail -1)
[ -z "$LATEST" ] && echo "No swarm runs recorded yet." && exit 0
tail -1 "$LATEST" | python3 -c "import sys, json; d=json.load(sys.stdin); print(json.dumps(d, indent=2))"
```

Format and print as a readable summary (same table format as the swarm report).

---

## Blast radius in swarm context

The same rules as Loop apply per agent. In addition:

- **Never merge across projects** — each agent works only in its own project's worktree
- If an agent's blast radius is `approval-required`, it stashes and moves the ticket to Blocked — the swarm does NOT block on it; other agents continue
- Staged changes are listed in the final report with the exact command to approve them

---

## Error handling

If an agent errors or returns `status: "error"`:
- Log the error in the swarm Logbook entry
- Mark it in the report with `✗ error`
- The ticket stays in `## Ready` (the agent should not move it on error)
- Other agents are unaffected

---

## Context budget enforcement

Each agent is bounded at **100k tokens total context**. The breakdown:

| Budget slice | Tokens | What fits |
|---|---|---|
| System prompt + skill files | ~15k | Loop SKILL.md, STANDARDS.md |
| Ticket + project context | ~5k | TICKETS.md, project README |
| File reads (capped) | ~50k | Max 8 files, 200k chars total |
| Model reasoning + output | ~30k | Plan, code edits, structured output |
| **Total** | **~100k** | |

### How agents self-enforce

**Step 3 (Plan)** is the enforcement gate. Before reading any source files, the agent must:

1. List every file it intends to read
2. Estimate token count: `wc -c <file> / 4` gives a rough token estimate
3. If estimated total exceeds 50k tokens → scope down the ticket (do the smallest useful subset)
4. If it cannot be scoped down → return `status: "budget-exceeded"` immediately

**Files always excluded** (even if the ticket mentions them):
- `node_modules/`, `dist/`, `.git/`, `coverage/`, `*.lock`, `*.log`
- Any generated file (`.d.ts` output, compiled JS in dist)
- Any file over 500 lines unless it is the direct subject of the ticket

### What happens on `budget-exceeded`

- Ticket stays in `## Ready` — not moved
- Report shows `⚡ budget` status
- Swarm report prints: "Ticket [ID] needs slicing — run `/slate slice [ID]` to break it up"
- Other agents unaffected

---

## Guard rails

- **Default depth: 3** — each project runs at most 3 tickets per swarm. Override with `/swarm <N>` e.g. `/swarm 5`. Max allowed depth: 10.
- **Stops early per project** — if a project hits `staged`, `budget-exceeded`, or `no-tickets`, its pipeline stops there. Other projects continue unaffected.
- Maximum **10 projects per swarm run** — if more eligible, pick the 10 with the highest-priority next ticket (high > medium > low)
- Never run `/swarm` if a `.loop-lock` file exists at `~/.claude/.loop-lock` (Dream is running)
- Never touch projects outside `~/personal-projects/`, `~/design-systems/`, and `~/AI26/` — hardcoded safety boundary

---

## Example flow

```
You: /swarm

── Swarm Dry Run ────────────────────────────────
skill-hub             [SH-001] Wire catalogue    7 ready
yellowtail-crosswords [YC-001] Connect CMS       5 ready
performance-review    [PR-001] Update content    4 ready
─────────────────────────────────────────────────
3 agents queued. Running...

[skill-hub]             ▶ Picked SH-001 · Planning...
[yellowtail-crosswords] ▶ Picked YC-001 · Planning...
[performance-review]    ▶ Picked PR-001 · Planning...

[skill-hub]             ✓ Done · 4 files · abc123
[yellowtail-crosswords] ✓ Done · 2 files · def456
[performance-review]    ⚠ Staged · blast radius: vite.config.js changed

── Swarm Report ────────────────────────────────
skill-hub              SH-001  ✓ done    4 files  loop/sh-001-wire-catalogue
yellowtail-crosswords  YC-001  ✓ done    2 files  loop/yc-001-connect-cms
performance-review     PR-001  ⚠ staged  6 files  loop/pr-001-update-content
────────────────────────────────────────────────
2 done · 1 staged · 0 errors
```
