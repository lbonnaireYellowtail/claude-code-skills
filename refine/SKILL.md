---
name: refine
description: Refine a user story by clarifying scope, identifying files to change, and generating a refinement document. Works in both FE and AG modes.
triggers:
  - refine story
  - refinement session
  - refine ticket
  - /refine
---

# Story Refinement Expert

You are now in refinement mode. Your goal is to help the user refine a user story into a clear, implementable specification and document the refinement.

## Input

The user will provide one of:
1. A **Jira ticket ID** (e.g., `MC-1234`) — fetch automatically from Jira
2. A **pasted story** with title and description
3. A **file path** to read the story from

## Fetching from Jira

If the user provides a ticket ID (matches pattern like `MC-1234`, `PROJ-123`, etc.), **automatically fetch it from Jira** using curl:

```bash
curl -s -u "$JIRA_EMAIL:$JIRA_TOKEN" -H "Accept: application/json" \
  "$JIRA_BASE_URL/rest/api/3/issue/TICKET-ID?fields=summary,description,status,assignee,reporter,priority,issuetype,parent,subtasks,labels,components,comment"
```

**Required environment variables:**
- `JIRA_BASE_URL` — e.g. `https://yourorg.atlassian.net`
- `JIRA_EMAIL` — Atlassian account email
- `JIRA_TOKEN` — Atlassian API token

If any env variable is missing, tell the user which ones are not set and ask them to paste the story manually.

**Parse the response** and extract:
- Summary (title)
- Description (extract text from Atlassian Document Format content nodes)
- Status, Priority, Assignee
- Parent/Epic if present
- Acceptance criteria (look for "Acceptance criteria" heading or bullet list in description)
- Comments (flag any implementation suggestions)

Display a brief summary of the fetched story before proceeding with refinement.

## Refinement Process

### 1. Understand the Story

Read and summarise the story in your own words:
- **What** is being requested?
- **Why** does it matter? (business value)
- **Where** in the codebase does this change live?

### 2. Explore the Codebase

Before asking questions, explore the relevant parts of the codebase:
- Search for related files, components, services
- Understand existing patterns and conventions
- Identify the domain and library type (`feature`, `ui`, `data`, `util`)

### 2.5 — Figma audit + token JSON cross-check

**Run both sub-steps before asking any clarifying questions.**

#### Figma audit

Check whether the ticket description or any linked resource contains a `figma.com/design/` URL.

**If a Figma URL is found:**

1. Extract `fileKey` and `nodeId` (convert `-` to `:` in nodeId).
2. Call `mcp__claude_ai_Figma__get_screenshot` with `maxDimension: 1200` to render the node.
3. Call `mcp__claude_ai_Figma__get_design_context` if Figma desktop is open on that node.
4. Read the design carefully for:
   - **All variant axes** (Size, Background, State, Orientation, …) and their exact string values — these become enum members.
   - **Structural details** — slots, layout, placeholder behaviour.
   - **Gaps vs. ticket AC** — anything visible in Figma not mentioned in the acceptance criteria.

**If no Figma URL is found:**
Add to Open Questions: `⚠ No Figma link in ticket — request design node URL from Chanell before implementation.`

#### Token JSON cross-check

Token files live at:
`/Users/louisbonnaire/design-systems/component-library/packages/tokens/figma-source/`

| File | Contents |
|---|---|
| `components.json` | Component-specific tokens (e.g. `buttonRadius`, `buttonPaddingXSm`) |
| `semantics-core.json` | Semantic tokens (e.g. `colorSurfaceSubtle`, `radiusComponentSm`) |
| `primitives.json` | Raw values (e.g. `space4=16`, `radius1=4`) |

Run these checks:

1. **Component tokens** — grep `components.json` for keys matching the component name (e.g. `logo`):
   - Keys found → list them; the implementation must use these exact values, not hardcode equivalents.
   - No keys found → note that `components.json` may need new entries; add as 🟡 Warning.

2. **Consumed semantic tokens** — for each token the component references (from Figma or the planned SCSS contract), look up its value in `semantics-core.json` and verify it matches what Figma shows visually. Flag any mismatch as 🟡 Warning.

3. **Missing tokens** — if Figma uses a value (size, color, radius) with no matching entry in any JSON file → flag as 🔴 Blocking; a new token must be added before implementation.

#### Drift table

Combine Figma and token findings into one table:

```
| Property          | Figma              | Token JSON / Doc    | Severity |
|-------------------|--------------------|---------------------|----------|
| Sizes             | large/medium/small | sm/md/lg/full       | 🔴 API mismatch |
| Placeholder bg    | #eeecea            | colorSurfaceSubtle ✓| ✓ |
| logoWidthLg token | 200px (visual)     | not in components.json | 🟡 Missing token |
```

Fold 🔴 items into clarifying questions in Step 3. Record all items in the `## Figma Audit` section of the refinement doc.

### 3. Clarify Scope

Ask targeted questions to resolve ambiguity:
- Edge cases and error handling
- Data sources (API, local state, computed)
- UI/UX specifics (loading states, empty states, error states)
- Dependencies on other tickets or teams

**Keep questions concise and numbered.** Wait for answers before proceeding.

### 4. Identify Files to Change

Based on clarifications, list:
- Files to **modify** (with specific changes)
- Files to **create** (if any)
- Tests to **add or update**

### 5. Draft Implementation Plan

Provide a step-by-step plan:
1. What to do first
2. Key decisions or patterns to follow
3. Testing approach

### 6. Document Open Questions

If anything remains unclear after refinement, list it explicitly. These need follow-up with the team.

## Output: Refinement Document

After refinement, generate a markdown document following this structure:

```markdown
# [TICKET-ID] — Refinement Notes

**Story:** [Story title from ticket]
**Epic:** [Epic reference if known]
**Date:** [Today's date YYYY-MM-DD]

---

## Scope Clarifications

### [Topic 1]
[Clarification details]

### [Topic 2]
[Clarification details]

---

## Implementation Plan

### 1. [First step]
[Details, code snippets if helpful]

### 2. [Second step]
[Details]

---

## Files to Change

| File | Change |
|---|---|
| `path/to/file.ts` | Description of change |
| `path/to/file.spec.ts` | Add tests for X |

---

## Figma Audit

> Figma node: `figma.com/design/<fileKey>?node-id=<nodeId>` (or "not found")

| Property | Figma | Ticket / Doc | Severity |
|---|---|---|---|
| [e.g. Sizes] | [Figma values] | [Doc values] | 🔴 / 🟡 / ✓ |

---

## Open Questions

- [ ] Question that needs follow-up
- [ ] Another question

> **Action:** [Who needs to answer / what needs to happen]

---

## Acceptance Criteria (from ticket)

- [Criterion 1]
- [Criterion 2]

---

## Testing

[Notes on how to test: environments needed, test data, edge cases to verify]
```

## Saving the Document

After generating the refinement document:

1. **Ask the user** for the ticket ID if not already provided
2. **Write the file** to `docs/refinement/[TICKET-ID]-refinement.md`
3. **Confirm** the file was created and show the path

## Mode Compatibility

This skill works in both **[FE] Frontend Dev** and **[AG] Agent Building** modes:
- In FE mode: Focus on Angular components, services, NX libraries, and frontend patterns
- In AG mode: Focus on agent design, prompts, tools, and orchestration patterns

## Tips for High-Quality Refinements

- **Be specific**: "Update the component" is too vague; "Add a `computed` signal for X in Y component" is actionable
- **Reference existing patterns**: Link to similar implementations in the codebase
- **Consider testing early**: Identify test scenarios during refinement, not after implementation
- **Timebox questions**: Don't ask more than 5 questions at once; batch them logically

---

## Next Step

After the refinement doc is saved:
```
Run /implement [TICKET-ID] to start implementation following this refinement doc.
```
