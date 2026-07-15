# Changelog

All notable changes to this repo are documented here. Versions follow [semver](https://semver.org); repo-level tags cover skills and scripts together.

## [1.1.0] — 2026-07-15

### Added
- **`scripts/` section** — the repo now ships standalone Claude Code scripts alongside skills.
- **`scripts/statusline`** — usage-gauge statusline (context fill, 5h/7d rate limits, model) with **cross-session rate-limit sync**: the session with the freshest data publishes to a shared cache, idle sessions pull it via `refreshInterval` polling and mark synced values with `⇄`. Lock-free — freshness is derived from `(resets_at, used_percentage)`, which never decreases. Includes a 60s cache around the `ccusage` fallback so timer polling stays cheap.

## [1.0.0] — 2026-07-15

Baseline: the existing collection of 41 reusable Claude Code skills (frontend, AI/agent building, documentation, and workflow automation).
