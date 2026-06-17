---
name: design-audit
description: Check local branch changes against Figma and token JSONs — surfaces CSS var fallback drift, missing component tokens, and design-spec mismatches before review.
triggers:
  - /design-audit
  - audit design
  - check design tokens
  - token drift
  - check tokens
---

# Design Audit

Checks local component changes against the design token JSON files and (optionally) Figma, to surface drift before it reaches PR review. Run it on any branch that adds or modifies a component atom.

## Input

`/design-audit` — detects scope from the current branch diff automatically.
`/design-audit <component>` — runs against a named component (e.g. `/design-audit container`).

---

## Paths

```
TOKEN_DIR=/Users/louisbonnaire/design-systems/component-library/packages/tokens/figma-source
ATOMS_DIR=libs/component-library-angular/src/lib/atoms
FIGMA_FILE_KEY=SFR4tLDUM1xkav9pMQJx6F   # Anchor DS V0.8
```

---

## Step 1 — Detect scope

If a component name was passed, use it. Otherwise:

```bash
git diff origin/master...HEAD --name-only
```

Look for files under `libs/component-library-angular/src/lib/atoms/<name>/`. Component = `<name>`. If multiple components are changed, run Steps 2–4 for each and combine the report.

If no component atoms changed, report:
> No component atom SCSS changes detected on this branch. `/design-audit` checks files under `libs/component-library-angular/src/lib/atoms/`.

Stop.

---

## Step 2 — Token cross-check

Run the following script. Pass each changed `.scss` file path as a command-line argument. The script extracts every `var(--name, fallback)` usage from the SCSS, maps the CSS variable name to the corresponding token JSON key, looks up the value in the token JSON files, and reports any fallback drift.

```bash
python3 << 'PYEOF'
import json, re, sys

TOKEN_BASE = "/Users/louisbonnaire/design-systems/component-library/packages/tokens/figma-source"

# ── Token file loader ──────────────────────────────────────────────────────────
# Each file is a JSON-encoded string (double-encoded). Parse the outer string
# first, then parse the inner JSON. Sections: pick desktop for responsive,
# default for brand, core/components for others.

PREFER_SECTION = {
    "semantics-responsive.json": "desktop",
    "brand.json":                "default",
    "semantics-core.json":       "core",
    "components.json":           "components",
    "primitives.json":           "core",
}

def load_tokens(fname):
    with open(f"{TOKEN_BASE}/{fname}") as f:
        raw = f.read().strip()
    inner = json.loads(raw)           # decode outer JSON string
    data  = json.loads(inner)         # decode inner JSON object
    preferred = PREFER_SECTION.get(fname)
    result = {}
    for section, vals in data.items():
        if isinstance(vals, dict):
            if preferred is None or section == preferred:
                result.update(vals)
    return result

all_tokens   = {}  # key → value
token_source = {}  # key → filename

for fname in ["components.json", "semantics-core.json", "semantics-responsive.json",
              "primitives.json", "brand.json"]:
    try:
        for k, v in load_tokens(fname).items():
            if k not in all_tokens:
                all_tokens[k]   = v
                token_source[k] = fname
    except Exception as e:
        print(f"# Warning: could not load {fname}: {e}")

# ── CSS variable → JSON key conversion ────────────────────────────────────────
# Rules (in order):
#   1. Strip leading --
#   2. Split on -
#   3. If first part is "spacing" and second is inset/stack/layout → rename first to "space"
#   4. Merge consecutive digit-only parts into the preceding part (e.g. palette-2-200 → palette2200)
#   5. camelCase the result

def css_var_to_key(var_name):
    name  = var_name.lstrip("-")
    parts = name.split("-")
    if len(parts) > 1 and parts[0] == "spacing" and parts[1] in ("inset", "stack", "layout"):
        parts[0] = "space"
    merged = []
    for p in parts:
        if merged and p.isdigit():
            merged[-1] += p
        else:
            merged.append(p)
    return merged[0] + "".join(p.capitalize() for p in merged[1:])

# ── Value normaliser ───────────────────────────────────────────────────────────
# For comparison: collapse whitespace, normalise comma spacing, strip px unit
# from plain numeric values (token JSON stores them as bare numbers).

def norm(v, strip_px=False):
    s = str(int(v)) if isinstance(v, (int, float)) and float(v) == int(float(v)) else str(v)
    s = re.sub(r"\s+", " ", s).strip()
    s = re.sub(r",\s*", ", ", s)
    if strip_px and re.fullmatch(r"\d+px", s):
        s = s[:-2]
    return s.lower()

# ── Extract vars from SCSS ─────────────────────────────────────────────────────
# Match var(--name) and var(--name, fallback) — handles multi-line and nested ()

SCSS_FILES = sys.argv[1:]
if not SCSS_FILES:
    # Default: find changed scss under atoms
    import subprocess
    result = subprocess.run(
        ["git", "diff", "origin/master...HEAD", "--name-only"],
        capture_output=True, text=True
    )
    SCSS_FILES = [
        p for p in result.stdout.splitlines()
        if "atoms/" in p and p.endswith(".scss")
    ]

VAR_RE = re.compile(
    r"var\(\s*(--[a-z0-9-]+)\s*(?:,\s*((?:[^)(]|\([^)]*\))*))?\)",
    re.DOTALL,
)
# Match custom property declarations (component-own vars defined in this file)
# e.g.: --container-bg: transparent;
DECL_RE = re.compile(r"(--[a-z0-9-]+)\s*:")

rows = []
seen_vars = set()
component_own_vars = set()  # vars declared in the SCSS file itself (not design-system tokens)

for scss_path in SCSS_FILES:
    try:
        scss = open(scss_path).read()
    except FileNotFoundError:
        print(f"# Skipping {scss_path}: not found")
        continue

    # Collect component-own custom properties (declared in this file)
    for dm in DECL_RE.finditer(scss):
        component_own_vars.add(dm.group(1))

    for m in VAR_RE.finditer(scss):
        var_name     = m.group(1).strip()
        fallback_raw = m.group(2)
        fallback     = " ".join(fallback_raw.split()) if fallback_raw else None

        if var_name in seen_vars:
            continue
        seen_vars.add(var_name)

        # Skip component-own custom properties — they're internal to the component,
        # not design system token references
        if var_name in component_own_vars:
            continue

        key    = css_var_to_key(var_name)
        found  = key in all_tokens
        source = token_source.get(key, "—")

        if not found:
            status = "🔴 NOT IN TOKENS"
            note   = f"tried key: {key}"
        elif fallback is None:
            status = "⚪ no fallback"
            note   = f"token = {norm(all_tokens[key])}"
        else:
            tok_norm = norm(all_tokens[key])
            fb_norm  = norm(fallback, strip_px=True)
            status   = "✓" if fb_norm == tok_norm else "🟡 MISMATCH"
            note     = "" if status == "✓" else f"token = {tok_norm}"

        rows.append((var_name, key, fallback or "—", status, source, note))

# ── Print report ──────────────────────────────────────────────────────────────
W = (36, 34, 42, 22)
header = f"{'CSS Variable':<{W[0]}} {'JSON Key':<{W[1]}} {'Fallback':<{W[2]}} {'Status':<{W[2]}} Source  Note"
print(header)
print("─" * 140)
for var_name, key, fallback, status, source, note in rows:
    fb = (fallback[:39] + "…") if len(fallback) > 40 else fallback
    print(f"{var_name:<{W[0]}} {key:<{W[1]}} {fb:<{W[2]}} {status:<{W[2]}} {source}  {note}")

counts = {"✓": 0, "🟡": 0, "🔴": 0, "⚪": 0}
for r in rows:
    for k in counts:
        if r[3].startswith(k):
            counts[k] += 1
print()
print(f"Summary: ✓ {counts['✓']} matched  🟡 {counts['🟡']} mismatch  🔴 {counts['🔴']} missing  ⚪ {counts['⚪']} no-fallback")
PYEOF
```

If no SCSS files are detected automatically, pass the path explicitly:

```bash
python3 … << 'PYEOF' … PYEOF  libs/component-library-angular/src/lib/atoms/<name>/<name>.component.scss
```

### Interpreting the output

| Status | Meaning | Action |
|---|---|---|
| ✓ | Fallback matches the token JSON value | None |
| 🟡 MISMATCH | Fallback differs from the token JSON value | Update the SCSS fallback to match the token JSON |
| 🔴 NOT IN TOKENS | CSS var name maps to a key that doesn't exist in any token JSON | Either the var name is wrong OR a new token is needed in components.json |
| ⚪ no fallback | Token exists but the SCSS has no fallback value | Low risk; consider adding a fallback for robustness |

### Common mismatch causes

- `--spacing-inset-*` uses the desktop value from `semantics-responsive.json`. The mobile value may differ — this is expected; only flag desktop mismatches.
- Shadow strings: whitespace around commas in `rgba(...)` may vary. Normalise by collapsing spaces before comparing.
- `border.json` `default` section only covers the Yellowtail default brand. Mismatches against `brand1` / `brand2` are expected and not flagged.

---

## Step 3 — Component token check

Check whether `components.json` contains any tokens for this component.

```bash
python3 -c "
import json
with open('/Users/louisbonnaire/design-systems/component-library/packages/tokens/figma-source/components.json') as f:
    raw = f.read()
inner = json.loads(raw)
data = json.loads(inner)
tokens = data.get('components', {})
name = '<component-name>'
matches = {k: v for k, v in tokens.items() if name.lower() in k.lower()}
if matches:
    print(f'Found {len(matches)} component tokens:')
    for k, v in matches.items():
        print(f'  {k}: {v}')
else:
    print(f'No component tokens found for \"{name}\" in components.json')
"
```

- **Tokens found** → list them. Verify the implementation uses these exact values (or the semantic vars that resolve to them) — not hardcoded equivalents.
- **No tokens found** → add a 🟡 warning: `components.json has no entries for <name> — check if dedicated component tokens are expected per the design spec.`

---

## Step 4 — Figma audit

### 4a — Find the Figma URL

Check in this order:

1. **Refinement doc** — `ls /Users/louisbonnaire/design-systems/component-library/docs/refinements/` — look for `<TICKET-ID>-*` matching the current branch. If found, read it and extract the Figma URL from the `## Figma Audit` section.
2. **Anchor DS file** — if no refinement doc or no URL: use file key `SFR4tLDUM1xkav9pMQJx6F` (Anchor DS V0.8). Search for the component frame by name using `mcp__claude_ai_Figma__search_design_system` with a query like `<component-name> component`.

If neither source produces a node URL, note: `⚠ No Figma node found — skip visual audit or provide a Figma URL manually.`

### 4b — Fetch design context

```
mcp__claude_ai_Figma__get_screenshot(fileKey, nodeId, maxDimension=1200)
mcp__claude_ai_Figma__get_design_context(fileKey, nodeId)   # if Figma desktop has it open
```

Extract from the design:

- **Variant axes** — what axes are defined (e.g. `Variant`, `Size`, `State`) and their string values. Cross-check against the TypeScript types in `<component>.types.ts`.
- **Spacing values** — padding, gap, border-radius visible in the design. Cross-check against token fallback values in the SCSS.
- **Color values** — fill colors for each variant state. Cross-check against the CSS vars used for `--container-bg`, `--container-border-color`, etc.
- **Missing states** — any Figma state (e.g. disabled, hover, selected) that has no corresponding SCSS rule.

### 4c — Build Figma drift table

```
| Property               | Figma                     | SCSS / Token JSON         | Severity |
|------------------------|---------------------------|---------------------------|----------|
| border-radius          | 8px                       | --radius-component-md ✓  | ✓        |
| padding (md)           | 16px                      | --spacing-inset-md ✓     | ✓        |
| filled bg color        | #eef3fa                   | --color-brand-surface ✓  | ✓        |
| hover border (filled)  | #3a6daf                   | --color-brand-secondary-hover ✓ | ✓  |
| focus outline          | 2px solid #3a6daf         | outline: 2px solid --color-border-focus ✓ | ✓ |
| size axis in Figma     | none                      | ContainerPadding xs/sm/md/lg/xl | 🟡 no size axis — padding only |
```

Severity guide:
- ✓ — matches
- 🟡 — diverges but workable (label/naming difference, no functional impact)
- 🔴 — blocking mismatch (wrong value, missing variant, wrong token used)

---

## Step 5 — Report

Present the combined findings in this structure:

```markdown
## Design Audit — <component-name>

**Branch:** `<branch>`  **Date:** <today>

### Token cross-check

<output of Step 2 script, reformatted as a markdown table>

**Component tokens in components.json:** <found N tokens | no tokens found>

### Figma audit

> Node: <figma URL or "not found">

<Figma drift table from Step 4c, or "Skipped — no Figma node available.">

### Summary

| Category        | Count |
|-----------------|-------|
| ✓ matched       | N     |
| 🟡 mismatch     | N     |
| 🔴 missing      | N     |
| ⚪ no fallback  | N     |

**Action required:** <list each 🔴 item as a blocking fix; list each 🟡 item as a recommended fix>
```

If all statuses are ✓ or ⚪, close with:
> All token fallback values are aligned with the token JSON. No design drift detected.
