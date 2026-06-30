---
name: ping-pong
description: >-
  Research-document-experiment loop for architectural decisions. Alternates
  between focused investigation, live documentation, and experimental validation
  before any implementation. Use when the user wants to research a technical
  topic, evaluate libraries or tools, investigate a tradeoff, or understand
  a system before deciding how to build it. Triggered by /ping-pong, "let's
  investigate", "before we implement let's understand", or any request to
  evaluate options without committing to one yet.
triggers:
  - /ping-pong
  - let's investigate
  - before we implement
  - research first
---

# Ping-Pong: Research ↔ Document Loop

A structured pattern for making good architectural decisions. The loop:

```
Refine → Research → Refine (elaborate) → Check in → Document → Experiment → Document → (implement when ready)
```

Never implement during a ping-pong session unless the user explicitly asks.

---

## Phase 0 — Short-form refinement (before research)

**Do this first, before any research or code inspection.**

Read the ticket or topic supplied by the user. Produce a short refinement covering:
1. **Scope** — what is and is not in scope for this investigation
2. **Key questions** — the 2–4 specific unknowns that research needs to answer
3. **Constraints already known** — anything from the codebase, ADRs, or prior decisions that narrows the option space before we even start

Keep it tight: bullet points, no prose padding. This is a shared compass for the research phase, not a deliverable.

Do not wait for user approval before moving to Phase 1 — proceed immediately unless a constraint or scope question is genuinely ambiguous.

---

## Phase 1 — Investigation only

**Do not implement anything.**

**Do NOT invoke the deep-research skill or spawn agent swarms.** Use inline tools only:
- A few targeted `WebSearch` calls for current best practices, library comparisons, known issues
- Code inspection of what is already in the project (existing choices, ADRs, parsers)
- Hands-on testing if tools are already installed (quick benchmarks, output samples)

Keep research lightweight: 3–5 web searches max, read the most relevant results directly.

Produce a **research brief** covering:
1. What the problem actually is (root cause, not symptoms)
2. The tool/library landscape with honest trade-offs
3. What the current setup does well and where it falls short
4. 2–3 concrete options with their real implementation cost

---

## Phase 1.5 — Elaborate refinement (after research, before writing the doc)

After research, produce a fuller refinement that incorporates what was found:
1. **Revised scope** — did research change what's in or out of scope?
2. **Decision axes** — the specific dimensions each option differs on (performance, complexity, migration cost, etc.)
3. **Recommended approach** — one clear recommendation with the primary reason
4. **Open questions** — anything still uncertain that an experiment or user input would resolve
5. **Implementation sketch** — rough steps if the recommendation is adopted (no code yet)

Then **stop and check in with the user**. Present the elaborate refinement as a summary and ask:
- Is the recommended approach the right direction?
- Are there constraints or priorities that change the recommendation?
- Should we proceed to write the ADR and design the experiment, or pivot?

Do not write the ADR until the user confirms direction.

---

## Phase 2 — Write it down before it leaves context

After Phase 1, write findings into a doc immediately — do not wait for the
user to ask. Use the project's existing doc conventions:

| Doc type | Where |
|----------|-------|
| Architectural decision | `docs/decisions/` as a new ADR |
| Future investigation | `future/<topic>.md` (update if file exists) |
| Experiment results | `<codebase>/experiments/<topic>/results/report.md` |

**Every doc must include:**
- Root cause of the problem (not just "tool X is better")
- Reason each option was accepted or rejected
- **"Do not use when" conditions** for every library considered
- **Switch triggers** — the specific observable conditions that should prompt a revisit
- Date of the investigation

---

## Phase 3 — Experiment to validate

When the research produces competing hypotheses, build a minimal experiment:

1. Create fixtures that stress the specific dimensions that differ (not general ones)
2. Write a scorer that measures what the downstream system actually cares about
3. Run all candidates against all fixtures
4. Capture raw outputs — future you needs to read them, not just scores

The experiment lives in `<codebase>/experiments/<topic>/` with:
- `fixtures/` — synthetic test inputs
- `converters.py` / `adapters.py` — one function per candidate
- `scorer.py` — scoring logic tied to real downstream requirements
- `run.py` — single entry point
- `results/report.md` — auto-generated, committed to git

---

## Phase 4 — Update the doc with experiment findings

After the experiment, go back to the doc and add:
- What the experiment confirmed or contradicted
- Specific scores / outputs that drove the conclusion
- The recommended approach and why
- What was deferred and under what conditions it should be revisited

---

## Phase 5 — Implementation (only when user asks)

After Phases 1–4, the implementation brief is already written. At that point:
- The architecture is validated
- The "why" is documented
- Edge cases are known
- Switch triggers are defined

Implement against the experiment's winning approach, not a fresh hypothesis.

---

## What makes ping-pong different from just researching

| Regular research | Ping-pong |
|-----------------|-----------|
| Answers "what is best?" | Answers "what is best *for this specific system*?" |
| Produces a recommendation | Produces a decision guide with switch triggers |
| Documents results | Documents reasoning + "do not use when" conditions |
| Research then implement | Refine → research → refine → check in → doc → experiment → implement |
| One pass | Iterated until confident |

The key constraint: **document the reasoning, not just the conclusion**. Future
maintainers (including the agent in a new context window) need to reconstruct
why a choice was made, not just what it was.

The check-in after Phase 1.5 is a hard gate — do not write the ADR until the
user confirms the direction. This prevents spending effort documenting an
approach the user would have redirected in 30 seconds.

---

## Example trigger phrases

```
/ping-pong chunking strategies
/ping-pong should we switch to Docling for PDF ingestion?
/ping-pong embedding model options
let's investigate hybrid search before implementing
deep research on reranker options — no implementation yet
```
