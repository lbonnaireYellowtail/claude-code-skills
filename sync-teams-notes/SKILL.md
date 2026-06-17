---
name: sync-teams-notes
description: Fetches new messages from the user's Microsoft Teams self-chat (48:notes) and appends them to ~/notes/teams-memos.md. Tracks last sync via ~/.config/teams-notes/state.json.
triggers:
  - sync teams notes
  - sync my teams notes
  - fetch teams notes
  - sync teams memos
---

# Sync Teams Notes

When this skill is invoked, fetch new self-chat messages from Microsoft Teams and append them to the local notes file.

## Setup

- **Output file:** `~/notes/teams-memos.md`
- **State file:** `~/.config/teams-notes/state.json` (contains `last_sync` ISO timestamp)
- **MCP tool:** `mcp__claude_ai_Microsoft_365__chat_message_search`
- **Self-chat ID:** `48:notes`
- **User email:** `lbonnaire@yellowtail.nl`

## Steps

1. **Read the state file** at `~/.config/teams-notes/state.json` to get the `last_sync` timestamp. If the file doesn't exist, fetch all messages (no date filter).

2. **Load the MCP tool schema** via ToolSearch if not already loaded:
   ```
   ToolSearch: select:mcp__claude_ai_Microsoft_365__chat_message_search
   ```

3. **Fetch messages** using:
   - `query`: `*`
   - `sender`: `lbonnaire@yellowtail.nl`
   - `recipient`: `lbonnaire@yellowtail.nl`
   - `afterDateTime`: the `last_sync` value from state (omit if no prior sync)
   - `limit`: `100`, `offset`: `0`

4. **Paginate** if the last result has `"moreResults": true` — repeat with `offset` incremented by the batch size until no more results.

5. **Filter** results to only those with `"chatId": "48:notes"`. Discard messages from other chats (meetings, group chats).

6. **Skip** messages where `summary` is empty or whitespace only.

7. **Sort** filtered messages oldest-first by `createdDateTime`.

8. **Append** each message to `~/notes/teams-memos.md` in this format:
   ```markdown
   ## YYYY-MM-DD HH:MM

   <summary text>
   ```
   Dates should be displayed in UTC. Strip any leading "Louis Bonnaire" prefix from summaries (this is an artifact of the search API).

9. **Update state file** `~/.config/teams-notes/state.json` with the `createdDateTime` of the most recent message appended:
   ```json
   {
     "last_sync": "<ISO timestamp of newest message>",
     "total_messages": <previous total + new count>
   }
   ```

10. **Report** how many new messages were appended, or "No new messages since last sync" if none.

## Notes

- The Microsoft 365 MCP must be authenticated. If tools are missing, ask the user to run `/mcp` and connect "claude.ai Microsoft 365".
- The `summary` field is a truncated preview — sufficient for logging purposes.
- Do not re-append messages already in the file. The `afterDateTime` filter handles this via the state timestamp.
