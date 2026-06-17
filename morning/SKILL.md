---
name: morning
description: Pull overnight Dream digests from all personal project repos. Pulls each repo, then displays any digest written by last night's Dream cloud run.
triggers:
  - /morning
---

# Morning

Pull and display Dream's overnight work across your personal projects. Run this first thing to see what happened while you were asleep.

---

## Personal project repos

```
/Users/louisbonnaire/personal-projects/CarteDeVisite
/Users/louisbonnaire/personal-projects/performance-review
/Users/louisbonnaire/personal-projects/performance-review-form
/Users/louisbonnaire/personal-projects/skill-hub
/Users/louisbonnaire/personal-projects/yellowtail-crosswords
```

---

## Run sequence

### Step 1 — Pull all repos

For each repo, run:

```bash
git -C <repo> pull --ff-only 2>&1
```

Collect results: pulled / already up to date / failed. Never abort on a failed pull — log it and continue.

### Step 2 — Find today's digests

```bash
TODAY=$(date +%Y-%m-%d)
DIGEST="<repo>/digests/$TODAY.md"
[ -f "$DIGEST" ] && echo "found" || echo "missing"
```

### Step 3 — Display

Print a header:

```
# Morning — YYYY-MM-DD
```

**For each repo with a digest today:** print the full digest content, separated by `---`.

**For each repo without a digest today:** list it under a "Quiet last night" section (no commits, Dream skipped writing one, or Dream hasn't run yet).

**For each repo that failed to pull:** list it under a "Pull failed" section with the error output.

**If no digests exist at all:**

> No Dream digests found for today. Dream runs at 00:00 UTC (02:00 Amsterdam). If it's before that, check back later — or run `/dream` manually in any project.

---

## Output format

```
# Morning — 2026-06-18

## skill-hub
<digest content>

---

## performance-review-form
<digest content>

---

## Quiet last night
- CarteDeVisite
- yellowtail-crosswords

## Pull failed
- performance-review — already on a detached HEAD, skipped
```

---

## Notes

- Uses `--ff-only` to avoid accidental merges on work branches
- If a repo has uncommitted local changes, the pull will fail safely — listed under "Pull failed", nothing lost
- Dream writes digests only for projects with a TICKETS.md — all five repos qualify
