---
name: slate
description: Local issue tracker — read and manage TICKETS.md boards. Pick tickets, mark done, create new ones, slice into children, and check status. Used by Loop (/work) to drive autonomous ticket execution.
triggers:
  - /slate
  - /slate pick
  - /slate done
  - /slate new
  - /slate slice
  - /slate status
  - /slate init
  - /slate standards
---

# Slate

Manages `TICKETS.md` boards for any project. Every command begins by locating the board, and ends by writing the updated board back.

## Board format

```
# Tickets — Project Name
<!-- prefix: XX -->

## In Progress   ← max 1 ticket
## Refined       ← refined (ADR/refinement doc exists, scoped); ready to build. ordered: high → medium → low
## Ready         ← triaged but not yet refined. ordered: high → medium → low
## Blocked
## Done          ← newest at top
```

`## Refined` is optional — boards created before it may not have it. When present, it sits between `## In Progress` and `## Ready`. A ticket moves Ready → Refined once a refinement doc or ADR exists for it (via `/refine`, `/ping-pong`, or manual triage).

Each ticket block:
```
### [PREFIX-NNN] Ticket title
**Priority:** high | medium | low
**Parent:** PREFIX-NNN          ← omit line if no parent
**Blocked by:** reason or ID   ← omit line unless in Blocked section

Acceptance criteria:
- [ ] Criterion

Notes:
Free text.                      ← omit section if empty

---
```

The `---` ends every ticket block. A ticket's status = the H2 section it lives in.

---

## Step 0 — Find TICKETS.md

Run before every command:

```bash
# Walk up from current directory
find . -maxdepth 1 -name "TICKETS.md" 2>/dev/null \
  || find .. -maxdepth 1 -name "TICKETS.md" 2>/dev/null \
  || find ../.. -maxdepth 1 -name "TICKETS.md" 2>/dev/null
```

If not found: say "No TICKETS.md found in this project. Run `/slate init` to set up Slate." and stop.

Read the file. Extract the prefix from the `<!-- prefix: XX -->` comment.

---

## `/slate pick`

Pick the next ready ticket and move it to In Progress.

1. **Guard:** Check `## In Progress`. If it contains a ticket block (a line starting with `### [`), stop:
   > Already in progress: **[ID] Title**
   > Complete it with `/slate done` before picking another.

2. **Find next ticket:** Prefer `## Refined` (if the board has that section) — find the first ticket block there (the text from `### [` to the closing `---` inclusive). If `## Refined` is absent or empty, fall back to `## Ready`; when falling back, warn: "No refined tickets — picking an unrefined ticket from Ready. Consider `/refine` or `/ping-pong` first."

3. **If both Refined and Ready are empty:** say "No ready tickets. Add one with `/slate new`." and stop.

4. **Move:** Remove the ticket block from its source section. Paste it under `## In Progress` (after the header line, before the next H2).

5. **Write** the updated TICKETS.md.

6. **Print:**
   ```
   Picked up [SL-NNN]: Ticket title

   Priority: high
   Parent: SL-001 (if present)

   Acceptance criteria:
   - [ ] ...

   Notes: (if present)
   ...
   ```

---

## `/slate done`

Mark the in-progress ticket as done and move it to Done.

1. **Find:** Read `## In Progress`. If no ticket block, say "Nothing in progress. Use `/slate pick` to start a ticket." and stop.

2. **Check:** Read all checkboxes. If any `- [ ]` remain, ask: "Not all acceptance criteria are checked. Mark as done anyway? (yes/no)". If no, stop.

3. **Check all boxes:** Replace every `- [ ]` with `- [x]` in the ticket block.

4. **Move:** Remove the block from `## In Progress`. Prepend it to `## Done` (insert immediately after the `## Done` header line, before any existing tickets).

5. **Write** the updated TICKETS.md.

6. **Print:**
   ```
   Done: [SL-NNN] Ticket title
   ```

---

## `/slate new`

Create a new ticket and add it to Ready.

1. **Determine next ID:** Scan the entire TICKETS.md for all `[PREFIX-NNN]` patterns. Take the highest NNN and add 1. If none exist, start at `001`.

2. **Gather fields** (ask the user if not provided as args):
   - **Title** (required)
   - **Priority** — high / medium / low (default: medium)
   - **Parent** ticket ID (optional)
   - **Acceptance criteria** — ask for a list, one per line. Keep prompting until the user sends an empty line.

3. **Format the ticket block:**
   ```
   ### [PREFIX-NNN] Title
   **Priority:** medium
   **Parent:** PREFIX-NNN     ← omit if no parent

   Acceptance criteria:
   - [ ] ...

   ---
   ```

4. **Insert into Ready** in priority order: after the last `high` ticket if priority is `medium`, after the last `medium` ticket if priority is `low`, at the top of Ready if priority is `high`.

5. **Write** the updated TICKETS.md.

6. **Print:**
   ```
   Created [SL-NNN]: Title
   Added to Ready (priority: medium)
   ```

---

## `/slate slice <ticket-id>`

Break a ticket into smaller child tickets.

1. **Find the parent ticket** anywhere in TICKETS.md by searching for `### [TICKET-ID]`.

2. **If not found:** say "Ticket [ID] not found in TICKETS.md." and stop.

3. **Gather children:** Ask: "Enter child ticket titles (one per line, empty line to finish):"
   - For each title, ask its priority (default: same as parent).

4. **Assign IDs** sequentially from the current highest ID + 1.

5. **Create each child block** with `**Parent:** PARENT-ID`.

6. **Append children** to `## Ready` in priority order.

7. **Write** the updated TICKETS.md.

8. **Print:**
   ```
   Sliced [SL-NNN] into 3 children:
   - [SL-004] Child one
   - [SL-005] Child two
   - [SL-006] Child three
   All added to Ready.
   ```

---

## `/slate refine <ticket-id>`

Move a ticket from Ready to Refined once it has a refinement doc or ADR.

1. **Find** the ticket by `### [TICKET-ID]`. If not found, say "Ticket [ID] not found." and stop.

2. **If the board has no `## Refined` section:** add one immediately after `## In Progress` (blank line before `## Ready`).

3. **Move:** Remove the ticket block from `## Ready` (or wherever it lives) and insert into `## Refined` in priority order (high → medium → low), same ordering rule as `/slate new`.

4. **Optionally** append a `⏳ Refined YYYY-MM-DD` line with the ADR/doc path to the ticket's Notes (ask if the doc path isn't obvious).

5. **Write** and print: `Refined: [SL-NNN] Title → moved to Refined (ADR/doc: ...)`.

`/refine` and `/ping-pong` should call this transition when they finish producing a refinement doc.

---

## `/slate status`

Print a compact board overview.

1. Count ticket blocks in each section (count occurrences of `### [`).

2. Find the current in-progress ticket title (if any).

3. Find the next buildable ticket title (first `### [` in `## Refined`, else first in `## Ready`).

4. Print (omit the `Refined` row if the board has no such section):
   ```
   ── Slate Board ─────────────────────
   In Progress : 1  →  [SL-003] Write /slate pick and /slate done
   Refined     : 3  →  next: [SL-007] Pluggable DB adapter
   Ready       : 4  →  [SL-004] Write /slate new and /slate slice
   Blocked     : 0
   Done        : 2
   ────────────────────────────────────
   ```

If no ticket is in progress, show `In Progress : 0  →  (none — run /slate pick)`.

---

## `/slate init`

Set up Slate in the current project. Creates `TICKETS.md` and `STANDARDS.md`.

1. **Check:** If `TICKETS.md` already exists in the current directory, say "TICKETS.md already exists. Nothing to do." and stop.

2. **Ask:**
   - Project name (default: current directory name)
   - Ticket prefix (default: first 2 uppercase letters of project name, e.g. `MY` for `my-project`)

3. **Write** a blank `TICKETS.md`:
   ```markdown
   # Tickets — [Project Name]
   <!-- prefix: XX -->

   ## In Progress

   ## Refined

   ## Ready

   ## Blocked

   ## Done
   ```

4. **Auto-detect project type and create `STANDARDS.md`:**

   ```bash
   # Check for yellowtail-standards agent rules
   HAS_YT_RULES=$(find . -maxdepth 3 -path "*/.agent/rules/core.md" | head -1)
   # Check for any existing CLAUDE.md or AGENTS.md with rules
   HAS_CLAUDE_MD=$(find . -maxdepth 1 -name "CLAUDE.md" -o -name "AGENTS.md" | head -1)
   ```

   **If `.agent/rules/` found (YT project):**
   ```markdown
   # Standards — [Project Name]

   ## Rules

   @.agent/rules/core.md
   @.agent/rules/angular.md
   @.agent/rules/angular-casing.md
   @.agent/rules/api.md
   @.agent/rules/styling.md
   @.agent/rules/nx.md
   @.agent/rules/testing.md

   ## Project-specific rules

   <!-- Add rules unique to this project below -->
   ```

   **If no YT rules found (personal project):**
   ```markdown
   # Standards — [Project Name]

   ## Rules

   <!-- Reference shared rule files with @ syntax, e.g.:           -->
   <!-- @/Users/louisbonnaire/personal-projects/yellowtail-standards/.agent/rules/core.md -->

   ## Code style
   - Use TypeScript strict mode
   - Prefer explicit types over inference on public APIs

   ## Testing
   - Every exported function has at least one test
   - Test filenames match source: `foo.ts` → `foo.test.ts`

   ## Git
   - Commit messages: `[TICKET-ID] description`
   - One branch per ticket: `loop/TICKET-ID-slug`
   - Never commit directly to main / master / development
   ```

   If `STANDARDS.md` already exists, skip creation and print a note.

5. **Print:**
   ```
   Slate initialised for [Project Name] (prefix: XX)
   Created: TICKETS.md
   Created: STANDARDS.md  (yellowtail rules detected)   ← or "(personal template)"
   Add your first ticket with /slate new.
   ```

---

## `/slate standards`

Show the active standards for the current project.

1. Find `STANDARDS.md` by walking up from CWD (same as TICKETS.md lookup).
2. If not found: "No STANDARDS.md in this project. Run `/slate init` to create one." Stop.
3. Read the file. Resolve any `@path/to/rules.md` references:

   ```bash
   # For each line starting with @, read that file and inline its content
   while IFS= read -r line; do
     if [[ "$line" == @* ]]; then
       rule_file="${line:1}"   # strip leading @
       if [ -f "$rule_file" ]; then
         echo "── $rule_file ──"
         cat "$rule_file"
       else
         echo "⚠ Rule file not found: $rule_file"
       fi
     else
       echo "$line"
     fi
   done < STANDARDS.md
   ```

4. Print the resolved content so all active rules are visible in one view.

---

## Error handling

| Situation | Response |
|---|---|
| TICKETS.md not found | "No TICKETS.md found. Run `/slate init`." |
| Ticket already in progress when picking | Show current ticket, say to use `/slate done` first |
| No ready tickets when picking | "No ready tickets. Use `/slate new`." |
| Nothing in progress when done | "Nothing in progress. Use `/slate pick`." |
| Ticket ID not found for slice | "Ticket [ID] not found." |
| TICKETS.md exists on init | "Already initialised." |
| STANDARDS.md not found on `/slate standards` | "No STANDARDS.md. Run `/slate init`." |
| `@` reference file not found | Print warning inline, continue with remaining rules |

Always write a clean TICKETS.md — preserve all whitespace, section order, and `---` separators exactly as specified.
