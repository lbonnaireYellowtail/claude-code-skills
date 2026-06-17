---
name: ng-migrate
description: Migrate Angular to the next major version using NX — researches breaking changes, checks package compatibility, executes the migration with commits per step, and produces a Confluence-ready report.
triggers:
  - migrate angular
  - angular migration
  - upgrade angular
  - /ng-migrate
---

# Angular Migration Expert

You are now in Angular migration mode. Your goal is to safely migrate this NX Angular project to the next major Angular version — one major version at a time — following NX best practices.

> **One major version at a time.** If the project is on Angular 20 and the user asks to migrate to Angular 22, run this skill twice — once targeting v21, once targeting v22. Never skip a major.

---

## Step 0 — Detect Versions & Verify Target Exists

Read `package.json` and extract:
- Current Angular version (`@angular/core`)
- Current NX version (`nx`)
- All dependencies and devDependencies (the full list — you will need it for Steps 2 and 3)

Determine the **target version**: the next major Angular release. For example, if current is `20.x`, target is `21`.

**Before doing any research, verify the target version actually exists as a stable release:**

```bash
npm view @angular/core@[target] version 2>/dev/null
```

- If this returns a version → the release exists, continue.
- If this returns nothing or an error → the release does not yet exist. Stop and notify the user:
  ```
  Angular [target] has not been released yet as a stable version.
  Current latest: [result of: npm view @angular/core version]
  Nothing to do until the release lands. Re-run /ng-migrate when it does.
  ```

Display a version summary before proceeding:

```
## Migration Target

| Package           | Current  | Target   |
|-------------------|----------|----------|
| @angular/core     | 20.x.x   | 21.x.x   |
| nx                | 21.x.x   | 22.x.x   |
| TypeScript        | 5.8.x    | 5.9.x    |
| jest-preset-ang.. | 15.x.x   | 16.x.x   |
```

---

## Step 1 — Check for Existing Migration Doc

Before doing any research, check whether a migration assessment document already exists:

```
docs/migration/angular-[current]-to-[target]-assessment.md
```

### If the doc EXISTS

Read it and extract:
- **Blockers** (packages listed as ❌)
- **Warnings** (packages listed as ⚠️)
- **Date the assessment was run**

For each ❌ blocker, run a **two-phase re-check**:

**Phase 1 — Check if the package was removed from the project:**

Read the current `package.json`. If the blocking package is no longer present in `dependencies` or `devDependencies`, mark it ✅ resolved (by removal) immediately — no web search needed.

```
✅ @ngrx/signals — resolved: package removed from project (not in package.json)
```

**Phase 2 — For blockers still present in package.json:**

Use WebSearch to look up the latest release and check whether a version compatible with Angular [target] has been published. Update the status accordingly (✅ resolved / ❌ still blocked / ⚠️ conditional).

Show the user a **diff of blocker statuses** vs the previous assessment:

```
## Blocker Status Update (vs assessment from YYYY-MM-DD)

| Package       | Previous | Now     | Resolution                         |
|---------------|-----------|---------|------------------------------------|
| @ngrx/signals | ❌        | ✅      | Package removed from project       |
| nx            | ❌        | ❌      | NX 23 still in beta as of today    |
| some-lib      | ❌        | ✅      | v22.0.0 released 2026-07-01        |
```

If all blockers are resolved → proceed to Step 3 (skip full research, use the existing doc as the base).
If blockers remain → notify the user, update the doc with new statuses, and stop.

### If the doc does NOT exist

Proceed to Step 2.

---

## Step 2 — Research Breaking Changes

Use **WebSearch** and **WebFetch** to gather information from official sources.

### 2a — Angular changelog

Search for the official Angular changelog and release notes for the target version:

- Search: `Angular [target version] release notes breaking changes`
- Fetch the Angular update guide: `https://angular.dev/update-guide` (filter for current→target)

Extract and categorise:
- **Breaking changes** (API removals, signature changes, default behaviour changes)
- **Deprecations that become errors** in this version
- **New mandatory patterns** (e.g. standalone required, new template syntax, host expressions)

### 2b — NX migration guide

- Search: `NX migrate angular [target version] guide [current year]`
- Fetch: `https://nx.dev/recipes/tips-n-tricks/advanced-update`

Note the recommended NX version that pairs with the target Angular version.

### 2c — Key library compatibility

Derive the list of packages to check directly from `package.json` — do not use a hardcoded list. For every package in `dependencies` and `devDependencies` that is Angular-aware (any package with `@angular/` in its name, or that declares `@angular/core` in its peerDependencies), check compatibility with Angular [target].

For each such package:
```bash
npm view [package]@latest peerDependencies 2>/dev/null
```

This tells you whether the latest version supports Angular [target] without a web search. For any package where the peerDeps are restrictive or ambiguous, follow up with a web search.

Also explicitly check:
- **TypeScript** — what version does Angular [target] require?
- **NX** — what NX version pairs with Angular [target]?
- **jest-preset-angular** — what peer dep range does the latest version declare?

---

## Step 3 — Package Compatibility Checklist

Produce a full compatibility checklist from the packages identified in Step 2c:

```markdown
## Package Compatibility — Angular [current] → [target]
Assessment date: YYYY-MM-DD

| Package                  | Current version | Target-compatible version | Status | Notes                          |
|--------------------------|-----------------|---------------------------|--------|--------------------------------|
| @angular/core            | 20.x.x          | 21.x.x                    | ✅     | Official package               |
| @angular/material        | 20.x.x          | 21.x.x                    | ✅     | Released alongside core        |
| jest-preset-angular      | 14.x.x          | 16.x.x                    | ✅     | Requires jest@30               |
| some-third-party         | 3.x.x           | unknown                    | ❌     | No Angular [target] release yet|
| another-lib              | 2.x.x           | requires investigation     | ⚠️     | Pre-release only               |

Legend: ✅ Ready  ⚠️ Conditional/pre-release  ❌ Not ready (blocker)
```

**Status definitions:**
- ✅ **Ready** — a stable release is available that explicitly supports Angular [target]
- ⚠️ **Conditional** — pre-release, workaround required (e.g. `overrides` in package.json), or support stated but not yet released
- ❌ **Not ready** — no Angular [target] compatible release; migration is blocked by this package

---

## Step 4 — Present Findings & Investigate Blockers

### 4a — Show the checklist and summary

Show the user:
1. The full checklist from Step 3
2. A summary:

```
## Migration Readiness Summary

✅ Ready:        [N] packages
⚠️ Conditional:  [N] packages (workarounds documented below)
❌ Blockers:     [N] packages
```

### 4b — For each ❌ blocker: investigate before recommending patience

Before telling the user to wait for a new release, investigate each blocker:

1. **Check usage depth** — how many files import from this package? Which symbols are used?
   ```bash
   grep -r "from '[package-name]'" [project-root] --include="*.ts" -l
   grep -r "from '[package-name]'" [project-root] --include="*.ts" -h | grep "^import"
   ```

2. **Assess replaceability** — based on which symbols are imported, determine if native Angular APIs or a simpler alternative could replace them. Ask:
   - Are only a few symbols used?
   - Do those symbols have direct Angular equivalents (e.g. `signalState` → `signal`, `patchState` → `.set()`)?
   - Is the package used as a thin wrapper or for deep functionality?

3. **Present options** — for each blocker, present the user with choices:

   ```
   ## Blocker: [package-name]

   Usage: [N] files, symbols: [list]
   Assessment: [brief — e.g. "only wraps 2 native Angular primitives"]

   Options:
     A) Remove / replace — [description of what would change, estimated effort]
     B) Wait — [package]@[target] not yet released. Track: [link]
     C) Override (if pre-release exists) — [exact overrides entry]
   ```

4. **Wait for user decision** before proceeding if blockers exist.

---

## Step 5 — Save Assessment Document

Write the assessment to:

```
docs/migration/angular-[current]-to-[target]-assessment.md
```

Structure:

```markdown
# Angular [current] → [target] Migration Assessment

**Date:** YYYY-MM-DD
**Status:** READY | BLOCKED | CONDITIONAL
**Target Angular version:** [target].x
**Target NX version:** [NX version]

---

## Package Compatibility Checklist

[full table from Step 3]

---

## Breaking Changes Summary

### Angular [target]

- [breaking change 1 — source: link]
- [breaking change 2]

### NX [NX target version]

- [change 1]

---

## Blockers

[list each ❌ package with: issue tracker link, usage depth, options presented to user, decision taken]

---

## Workarounds for Conditional Packages

[for each ⚠️ package: the exact workaround, e.g. package.json overrides]

---

## Recommended Migration Steps

1. [step — omit steps for packages that were resolved by removal]
2. [step]
3. ...

---

## References

- [Angular update guide link]
- [NX migration guide link]
- [Relevant GitHub issues]
```

---

## Step 6 — Create Migration Branch

If the user confirmed to continue, create a dedicated branch:

```bash
# First verify we're not on a protected branch
git branch --show-current

git checkout development
git pull origin development
git checkout -b feature/angular-[current]-to-[target]-migration
```

Echo the branch name after creation to confirm it is not `master`, `main`, or `development`.

---

## Step 7 — Execute Migration with NX

### 7a — Check for interrupted migration

Before running anything, check whether `migrations.json` already exists at the project root:

```bash
ls migrations.json 2>/dev/null && echo "EXISTS" || echo "NONE"
```

- If it **EXISTS** → a previous migration was interrupted. Show the user a summary of pending migrations in the file and ask:
  ```
  migrations.json already exists — this migration was started previously.

  Options:
    A) Resume — skip nx migrate calls and go straight to npm install + --run-migrations
    B) Start fresh — delete migrations.json and re-run all nx migrate commands
  ```
  Wait for the user's choice before continuing.

- If it does **not exist** → proceed normally from Step 7b.

### 7b — Run nx migrate for Angular packages

```bash
npx nx migrate @angular/core@[target]
```

This creates `migrations.json`. Inspect the file — show the user how many migration steps are queued.

### 7c — Migrate Angular Material and CDK

```bash
npx nx migrate @angular/material@[target]
npx nx migrate @angular/cdk@[target]
```

### 7d — Migrate NX

```bash
npx nx migrate nx@[NX target version]
```

### 7e — Migrate other packages

Based on the compatibility checklist, run `nx migrate` for each additional package that needs upgrading (skip any that were resolved by removal in Step 4). Only include packages that are still in `package.json`.

After all `nx migrate` calls, show the final `migrations.json` so the user can review the queued changes.

### 7f — Install updated dependencies

```bash
npm install
```

Check the output for:
1. **Peer dependency warnings** — resolve any conflicts before continuing.
2. **Audit vulnerabilities** — run `npm audit --production` immediately after install:

```bash
npm audit --production --audit-level=moderate
```

If new vulnerabilities appear that weren't present before the migration:
- Check what package introduced them (`npm explain [package]`)
- Pin the vulnerable transitive dependency in `package.json` `overrides` using the patched version
- Re-run `npm install` and confirm `npm audit --production` is clean before continuing

### 7g — Run migrations with per-step commits

```bash
npx nx migrate --run-migrations --createCommits
```

This runs each migration step and creates a git commit for each one. Watch the output and report any errors immediately.

If a migration step fails:
1. Show the full error output
2. Investigate the cause before retrying
3. Do not skip or force-continue without understanding the failure

### 7h — Handle known patterns from past migrations

After automated migrations run, apply known manual fixes based on this project's history and the breaking changes identified in Step 2. Common patterns to check for:

- `@HostBinding` → `host: {}` property migration (see Angular 21 migration)
- `CommonModule` removal from component imports (keep only when `ngClass` is used)
- Angular Material design token renames
- `inject()` vs constructor injection for `InjectionToken` types
- TypeScript `moduleResolution` settings in `tsconfig.spec.json` files
- Jest `setupFilesAfterEnv` and zoneless test environment setup
- `zone.js` in dependencies (should be removed for zoneless projects)

For each manual fix needed, grep the entire codebase for the pattern before fixing, then fix all occurrences in one pass. Never fix only the file you're currently looking at.

### 7i — Run targeted tests

After migration, run tests library by library (never project-wide):

```bash
npx nx test [library-name]
```

Run one at a time sequentially. For each failure, diagnose and fix before moving to the next library. Document each failure and fix in the findings log.

---

## Step 8 — Produce Confluence-Ready Migration Report

After migration is complete, generate a markdown document ready to paste into Confluence:

Write to: `docs/migration/angular-[current]-to-[target]-report.md`

Use this exact template, styled after the existing migration pages in the Confluence MN CRP space:

```markdown
# Angular [current] to Angular [target] Migration

This page serves as a summary of the steps taken and roadblocks encountered during the migration from Angular [current] to Angular [target].

---

## Process

The migration was done using the NX migrate tool:

```
npx nx migrate package-name@version
```

This tool creates a `migrations.json` file which automatically adds a list of required changes for each package updated. Running `npx nx migrate --run-migrations` executes these changes. The `--createCommits` flag creates a git commit for each migration step.

**Packages migrated:**

- Angular dependencies to version **[target].x**
  - `@angular/core`, `@angular/cli`, `@angular/devkit`, `@angular/forms`, `@angular/router`, `@angular/cdk`, `@angular/material`, etc.
- NX dependencies to version **[NX version]**
  - `@nx/angular`, `@nx/playwright`, `@nx/eslint`, `@nx/cli`, etc.
- [other packages with versions]
- TypeScript to version **[version]**

---

## Pre-Migration Dependency Changes

[List any packages removed or replaced before the migration to unblock it, with rationale and PR links.]

---

## Main Changes

### Angular [target]

[For each notable change: description, before/after code example if relevant]

### Angular Material [target]

[Material-specific changes]

---

## Roadblocks

### [Roadblock title]

[Detailed description of the problem encountered, why it occurred, and how it was resolved. Include code snippets for before/after where relevant.]

---

## New Features in Angular [target]

[Highlight the most relevant new features introduced in this Angular version, with links to official docs. Focus on features relevant to this codebase.]

- **[Feature name]** — [brief description] ([official docs link])
- **[Feature name]** — [brief description] ([official docs link])

---

## Relevant PRs

- Pre-migration cleanup: [link if applicable]
- Migration PR: [Bitbucket PR link or "pending"]

---

## References

- [Angular [target] Release Notes — link]
- [Angular Update Guide — https://angular.dev/update-guide]
- [NX Advanced Update Process — https://nx.dev/recipes/tips-n-tricks/advanced-update]
- [Any GitHub issues referenced during migration]
```

After writing the file, display the full document in the conversation so the user can review it before pasting into Confluence.

---

## Step 9 — Final Summary

End with a concise summary:

```
## Migration Complete — Angular [current] → [target]

**Branch:** feature/angular-[current]-to-[target]-migration
**Commits:** [N] (one per migration step)
**Pre-migration removals:** [list packages removed to unblock]
**Blockers encountered during migration:** [N]
**Manual fixes applied:** [list]
**Libraries with test failures fixed:** [list]
**Audit status:** clean / [N vulnerabilities pinned via overrides]

**Files produced:**
- docs/migration/angular-[current]-to-[target]-assessment.md
- docs/migration/angular-[current]-to-[target]-report.md

**Next steps:**
- Review the Confluence-ready report and paste to the MN CRP space
- Run /shipping to push the migration branch
- Open a PR against development
```

---

## Behaviour Notes

- **Never run `npm run qc`, `npm run test`, or `npm run lint` project-wide.** Test with `npx nx test [library-name]` only, one at a time.
- **Never commit to `master`, `main`, or `development`.** Always work on the migration branch.
- **Never use `git add .` or `git add -A`.** Stage files explicitly by path.
- **Echo the current branch name before every commit** to confirm it is not a protected branch.
- **One major version at a time.** If the user asks to skip a version, explain why this is not recommended and proceed one version at a time.
- **WebSearch and WebFetch are mandatory for Steps 2 and 3.** Never rely on training data for package compatibility — always check current releases.
- **Derive the package list from `package.json`**, not from memory or hardcoded tables. The project's dependencies change between runs.
