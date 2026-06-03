---
name: view-story
description: Fetch and display a Jira story with its description, status, acceptance criteria, and comments. Always includes comments and critically flags implementation suggestions.
triggers:
  - view story
  - show story
  - view ticket
  - show ticket
  - /view-story
---

# View Story

You are fetching and presenting a Jira story in a clear, structured format.

## Input

The user provides a **ticket ID** (e.g. `MC-1234`). If not provided, ask for it.

## Fetching the Story

Use `curl` with the Jira REST API. Required environment variables:
- `JIRA_BASE_URL` — e.g. `https://yourorg.atlassian.net`
- `JIRA_EMAIL` — the user's Atlassian account email
- `JIRA_TOKEN` — Atlassian API token

```bash
curl -s \
  -u "$JIRA_EMAIL:$JIRA_TOKEN" \
  -H "Accept: application/json" \
  "$JIRA_BASE_URL/rest/api/3/issue/TICKET-ID?fields=summary,description,status,assignee,reporter,priority,issuetype,parent,subtasks,labels,components,comment,customfield_10014,customfield_10016"
```

If any env variable is missing, tell the user which ones are not set and stop.

## Parsing & Display

Parse the JSON response and display the story in this structure:

```
# [TICKET-ID] — [Summary]

**Type:** [Issue type]   **Status:** [Status name]   **Priority:** [Priority]
**Assignee:** [Name or Unassigned]   **Reporter:** [Name]
**Epic:** [Parent issue key + summary, if present]

---

## Description

[Rendered description text — Atlassian Document Format: extract `text` nodes from `content` array, preserve paragraph breaks]

---

## Acceptance Criteria

[Extract from description if present as a list or heading — label them clearly]
If none found in description, note: "No explicit acceptance criteria in description."

---

## Subtasks

[List subtasks with key, summary, and status — if any]
If none: omit this section.

---

## Comments ([count])

[For each comment, newest first:]
**[Author]** — [date]:
> [comment body text]

[Flag any comment that contains implementation suggestions with ⚠️ **Implementation suggestion — review critically**]
```

## Critical Comment Review

After displaying comments, if any contain implementation suggestions (code snippets, "you should", "I suggest", "we could"), add a section:

```
---

## ⚠️ Implementation Suggestions in Comments

[List each suggestion with the author and a brief summary]

> These suggestions are **not authoritative**. Evaluate them against existing codebase patterns before accepting. A better alternative may exist — propose it during refinement.
```

## After Display

Offer the user next steps:
- `Run /refine to start a refinement session for this ticket`
- `Run /implement MC-XXXX if a refinement doc already exists`
