---
name: check-stories
description: List all unfinished stories for the current repo — reads docs/STORIES.md if present, falls back to Jira. Only shows stories not in done status.
triggers:
  - check stories
  - unfinished stories
  - /check-stories
---

# Check Stories

Display only the stories that are **not done** for the current repository.

## Source Detection

```bash
find . -maxdepth 3 -name "STORIES.md" | head -5
```

- If found → **Local Mode**
- If not found → **Jira Mode**

---

## Local Mode

Read the file and output only stories whose status is NOT `done`.

Display format — one line per story, grouped by epic:

```
## Epic N — [Title]

- [ID] — [Summary]  `[status]`
- [ID] — [Summary]  `[status]`

---

[N] stories remaining
```

---

## Jira Mode

Requires: `JIRA_BASE_URL`, `JIRA_EMAIL`, `JIRA_TOKEN`, `JIRA_PROJECT_KEY`.
If any are missing, say which and stop.

```bash
curl -s \
  -u "$JIRA_EMAIL:$JIRA_TOKEN" \
  -H "Accept: application/json" \
  "$JIRA_BASE_URL/rest/api/3/search?jql=project=$JIRA_PROJECT_KEY+AND+status+NOT+IN+(Done)+ORDER+BY+created+ASC&maxResults=50&fields=summary,status,parent"
```

Display the same compact list format.

---

## Error handling

- STORIES.md empty or unparseable → say so, offer to fall back to Jira.
- Jira non-200 → show status code and error body.
- Zero results → "All stories are done."
