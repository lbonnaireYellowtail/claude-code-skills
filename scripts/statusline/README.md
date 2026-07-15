# Statusline — usage gauges with cross-session sync

A Claude Code statusline showing context fill, real 5h / 7d rate-limit usage, and the active model — with the rate-limit numbers **synchronized across all open terminals**.

```
🧠 ctx 42.0k (21%) | 🕐 5h 55% →2h00m | 📅 7d 13% →111h06m ⇄ | 🤖 Fable 5
```

## Why sync?

Rate limits are account-level, but each Claude Code session only refreshes its own statusline when *it* is active. Without sync, an idle terminal shows stale percentages while another session burns tokens.

Claude Code has no built-in cross-session push, so this script does it pull-based:

- Whichever session receives fresher `rate_limits` in its stdin payload publishes them to `~/.cache/claude-statusline/shared-rate-limits.json`.
- Sessions holding staler data render from that file instead, marked with a dim `⇄`.
- Freshness is derived from the data itself — `(resets_at, used_percentage)` per window never decreases for an account — so concurrent writers can never regress the cache. No locks, no per-session state.

Paired with `refreshInterval`, idle terminals converge within a couple of seconds of any active session's update. An idle tick costs ~44 ms (mostly Python startup); writes only happen when the numbers actually advanced.

## Install

```bash
cp scripts/statusline/statusline.py ~/.claude/scripts/statusline.py
chmod +x ~/.claude/scripts/statusline.py
```

Then in `~/.claude/settings.json`:

```json
{
  "statusLine": {
    "type": "command",
    "command": "~/.claude/scripts/statusline.py",
    "padding": 0,
    "refreshInterval": 2
  }
}
```

`refreshInterval` is what lets idle sessions pick up the shared cache; drop it to `1` for snappier propagation at roughly double the (small) idle cost. Already-open sessions need a restart to pick it up.

## Segments

| Segment | Source | Scope |
|---|---|---|
| 🧠 ctx | `context_window` tokens, colored against a soft target | per session |
| 🕐 5h / 📅 7d | `rate_limits` used % + time until reset | account-wide, synced |
| ⇄ | shown when rate limits came from another session's publish | — |
| 🤖 | active model | per session |

## Config (env vars)

| Var | Default | Meaning |
|---|---|---|
| `STATUSLINE_CTX_TARGET` | `100000` | soft context-token target for coloring |
| `STATUSLINE_CAUTION_PCT` | `60` | yellow at/above this % |
| `STATUSLINE_WARN_PCT` | `85` | red + ⚠️ prefix at/above this % |

On Claude Code versions too old to pass `rate_limits`, the script falls back to `ccusage` dollar estimates (cached 60s so the refresh timer can't hammer it), configurable via `STATUSLINE_5H_LIMIT` / `STATUSLINE_WEEK_LIMIT`.
