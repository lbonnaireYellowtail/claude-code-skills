---
name: review
description: Interactive, chunk-by-chunk review of code and doc changes. Steps through each changed file (or diff hunk) one at a time, gives a short analysis of each, and elaborates on demand. Works for code, docs, config, and mixed changesets.
triggers:
  - /review
  - review changes
  - review diff
  - review my changes
---

# Review

An interactive reviewer that walks through changes one chunk at a time. After each chunk it waits for the user to navigate — it never dumps everything at once.

## Invocation

| Command | Scope |
|---|---|
| `/review` | All uncommitted changes (staged + unstaged) |
| `/review --staged` | Staged changes only |
| `/review --branch` | All commits on current branch vs `master` |
| `/review --branch <base>` | All commits vs a custom base branch |
| `/review <file>` | A single file (uncommitted diff) |
| `/review <PR_ID>` | A Bitbucket PR (fetches diff via API — requires `JIRA_EMAIL` + `BITBUCKET_TOKEN`) |

---

## Step 1 — Collect the diff

Choose the right `git diff` command based on the invocation:

```bash
# default
git diff HEAD

# --staged
git diff --staged

# --branch
git diff origin/master...HEAD

# single file
git diff HEAD -- <file>
```

If the output is empty, report "No changes found for the given scope" and stop.

Parse the diff into **chunks** — one chunk per file. If a file has more than ~80 lines of diff, split it into logical hunks (at `@@` boundaries).

Count total chunks and announce the session:

```
Reviewing N chunks across M files.
Type a navigation command after each one.

  n / enter  → next chunk
  p          → previous chunk
  e          → elaborate on this chunk
  s          → skip remaining hunks in this file
  f <name>   → jump to a specific file
  q          → quit review

---
```

---

## Step 2 — Present chunks one at a time

For each chunk, output:

```
── Chunk X / N ──────────────────────────────────
File:   <relative file path>
Type:   <Code | Docs | Config | Test | Style>
Change: <Added | Modified | Deleted | Renamed>
─────────────────────────────────────────────────

<the diff hunk, formatted with +/- lines>

─────────────────────────────────────────────────
<Short analysis — 2–4 sentences max. See analysis rules below.>
─────────────────────────────────────────────────
[n] next  [p] prev  [e] elaborate  [s] skip file  [q] quit >
```

Wait for user input before continuing.

### Classifying chunk type

| Path pattern | Type |
|---|---|
| `*.ts`, `*.js`, `*.html`, `*.scss` (non-story, non-spec) | Code |
| `*.spec.ts`, `*.test.ts` | Test |
| `*.stories.ts`, `*.stories.js` | Style |
| `*.md`, `*.mdx` | Docs |
| `*.json`, `*.yaml`, `*.yml`, `*.toml` | Config |
| Mixed content (e.g. README with code examples) | Docs |

---

## Step 3 — Short analysis rules

Keep the analysis short and direct. Tailor it to the chunk type.

### Code chunks
Focus on:
- **What it does** — one sentence on the intent
- **Correctness** — any logic errors, edge cases, null-safety issues
- **Conventions** — naming, Angular signal patterns, component API consistency

Flag (but don't block) if you spot:
- A removed public API without a deprecation path
- A change that silently alters component behaviour
- Missing null guards on new optional inputs

### Doc chunks (README, .md files)
Focus on:
- **Accuracy** — does the new content match the actual code?
- **Code examples** — do the inline snippets use the current API?
- **Completeness** — is anything newly added to the code but missing from the docs?

Flag if:
- An example uses a removed type or input name
- An import path no longer exists in the public API
- A table row is missing a newly added input

### Test chunks
Focus on:
- **Coverage** — does the test cover the new/changed behaviour?
- **Assertions** — are they specific enough (not just `toBeTruthy()`)?
- **Setup** — does `beforeEach` still match the updated component API?

### Config / Story chunks
Focus on:
- **Correctness** — valid schema, no stale field references
- **Completeness** — do stories cover all new inputs/states?

---

## Step 4 — Elaboration (`e`)

When the user types `e`, go deeper on the current chunk. The elaboration should:

1. **Explain the full context** — why this pattern matters, what could go wrong
2. **Show a concrete example** of the issue (if any) — what would break and how
3. **Suggest a fix** — if there's a problem, show the corrected code/text inline
4. **Reference related code** — if the change touches something that ripples elsewhere, name the files

End elaboration with the same navigation prompt and wait for input.

---

## Step 5 — End of review

After the last chunk (or when the user types `q`):

```
── Review complete ──────────────────────────────

Chunks reviewed: X / N

Issues found:
  ⚠  <file> chunk 2 — <one-line summary>
  ⚠  <file> chunk 5 — <one-line summary>

Suggestions (non-blocking):
  💡 <file> chunk 1 — <one-line summary>

Everything else looked good.
─────────────────────────────────────────────────
```

If no issues were found, just say so — don't pad the output.

---

## Notes

- Never auto-advance. Always wait for the user's navigation input after each chunk.
- If the diff is entirely whitespace or formatting changes, label the chunk "Formatting only — nothing to review" and auto-advance.
- If a chunk is a file deletion, summarise what was removed and why it's safe (or flag if it looks like something still referenced elsewhere).
- If the user types anything other than a navigation command, treat it as a follow-up question about the current chunk and answer it, then re-show the navigation prompt.
