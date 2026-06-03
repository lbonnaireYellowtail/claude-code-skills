---
name: bitbucket-pr
description: Manage Bitbucket pull requests — view, create, and update PR title/description. Workspace and repo are derived from the current git remote automatically.
triggers:
  - /bitbucket-pr
  - /pr
---

# Bitbucket PR

Manage pull requests on Bitbucket Cloud via the REST API.

## Auth & environment

Authentication is HTTP Basic: Atlassian email as username, API token as password.

Required environment variables:
- `JIRA_EMAIL` — your Atlassian account email
- `BITBUCKET_TOKEN` — API token from bitbucket.org → Personal settings → API tokens
  - Required scopes: **Pull requests: Read + Write**

If either is missing, tell the user and stop.

## Derive workspace and repo from git remote

Run this at the start of every operation:

```bash
REMOTE_URL=$(git remote get-url origin 2>/dev/null)
if [[ $REMOTE_URL =~ bitbucket\.org[:/]([^/]+)/([^/.]+)(\.git)?$ ]]; then
  BB_WORKSPACE="${BASH_REMATCH[1]}"
  BB_REPO="${BASH_REMATCH[2]}"
else
  echo "Not a Bitbucket repo — cannot continue."
  exit 1
fi
BB_API="https://api.bitbucket.org/2.0/repositories/$BB_WORKSPACE/$BB_REPO"
```

## Commands

### list — show open PRs

```bash
curl -s -u "$JIRA_EMAIL:$BITBUCKET_TOKEN" \
  "$BB_API/pullrequests?state=OPEN&pagelen=25" \
  | jq -r '.values[] | "#\(.id) [\(.state)] \(.title) — \(.source.branch.name) → \(.destination.branch.name)"'
```

Display as a numbered list with PR ID, title, source → destination branch.

---

### view <PR_ID> — show PR details

```bash
curl -s -u "$JIRA_EMAIL:$BITBUCKET_TOKEN" \
  "$BB_API/pullrequests/<PR_ID>" \
  | jq '{id,title,description,state,source:.source.branch.name,destination:.destination.branch.name,author:.author.display_name}'
```

Display:
```
# PR #<id> — <title>
**State:** <state>   **Author:** <author>
**Branch:** <source> → <destination>

---

<description>
```

---

### update <PR_ID> — update title and/or description

Ask the user for the new title and description if not already provided. Then:

```bash
curl -s -X PUT \
  -u "$JIRA_EMAIL:$BITBUCKET_TOKEN" \
  -H "Content-Type: application/json" \
  "$BB_API/pullrequests/<PR_ID>" \
  -d "{\"title\": \"<TITLE>\", \"description\": \"<DESCRIPTION>\"}"
```

Confirm success by printing the returned `title` and first 100 chars of `description`.

**Escaping:** pipe the JSON body through `jq -n` to handle special characters safely:

```bash
BODY=$(jq -n \
  --arg title "$TITLE" \
  --arg desc "$DESCRIPTION" \
  '{title: $title, description: $desc}')

curl -s -X PUT \
  -u "$JIRA_EMAIL:$BITBUCKET_TOKEN" \
  -H "Content-Type: application/json" \
  "$BB_API/pullrequests/<PR_ID>" \
  -d "$BODY"
```

---

### create — open a new PR from the current branch

Derive defaults:
- **source branch:** `git branch --show-current`
- **destination branch:** `master` (or ask the user to confirm)
- **title / description:** ask the user, or generate from `git log origin/master..HEAD --oneline`

```bash
BODY=$(jq -n \
  --arg title "$TITLE" \
  --arg desc "$DESCRIPTION" \
  --arg src "$SOURCE_BRANCH" \
  --arg dst "$DEST_BRANCH" \
  '{
    title: $title,
    description: $desc,
    source: { branch: { name: $src } },
    destination: { branch: { name: $dst } },
    close_source_branch: true
  }')

curl -s -X POST \
  -u "$JIRA_EMAIL:$BITBUCKET_TOKEN" \
  -H "Content-Type: application/json" \
  "$BB_API/pullrequests" \
  -d "$BODY" \
  | jq -r '"Created PR #\(.id): \(.links.html.href)"'
```

---

## Error handling

- HTTP 401 → bad credentials, ask user to check `JIRA_TOKEN`
- HTTP 404 → PR not found or wrong repo, print `$BB_WORKSPACE/$BB_REPO` so the user can verify
- Non-zero `jq` exit → malformed response, print raw response for inspection
