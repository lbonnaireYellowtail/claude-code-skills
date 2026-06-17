---
name: agent-quality
description: Expert knowledge for agent evaluation, observability, and production quality — eval datasets, LLM-as-judge, CI/CD pipelines, traces, metrics, and safe rollout. Use when building eval systems, adding observability, measuring agent quality, or deploying agents to production.
triggers:
  - eval
  - evaluation
  - LLM judge
  - agent quality
  - test dataset
  - regression testing
  - CI/CD agent
  - benchmark
  - agent metrics
  - evaluation pipeline
  - evaluate agent
  - build eval
  - observability
  - CI/CD for agents
  - agent monitoring
  - deploy agent
---

# Agent Quality & Evaluation Expert

You are now loaded with deep knowledge about agent evaluation, observability, and production operations. Apply these when helping the user design or implement quality systems.

## Four Pillars of Agent Quality

1. **Effectiveness** (Goal Achievement): did agent achieve user's actual intent? Connect to business KPIs (conversion rate, completion rate)
2. **Efficiency** (Operational Cost): tokens consumed, wall-clock time, trajectory complexity. 25 steps for a simple task = low quality even if it succeeds
3. **Robustness** (Reliability): graceful handling of API timeouts, missing data, ambiguous prompts. Retry, clarify, or report inability — never crash/hallucinate
4. **Safety & Alignment** (Trustworthiness): non-negotiable gate. Fairness, bias, prompt injection resistance, data leakage prevention

## Why Evaluation Matters

Agents degrade silently — a prompt change, model update, or new tool can break behavior without obvious errors. Evaluation is how you detect regressions before users do.

**Without evals**: you're flying blind. With evals: you can ship with confidence.

## Evaluation Hierarchy (Outside-In)

### Stage 1: End-to-End (Black Box)
"Did the agent achieve the user's goal?"
- Task Success Rate
- User Satisfaction (CSAT, thumbs up/down)
- Overall Quality (accuracy, completeness)

### Stage 2: Trajectory Evaluation (Glass Box)
When black box reveals failure, open it to find **why**:
1. **LLM Planning** ("Thought"): hallucinations, off-topic, context pollution, repetitive loops
2. **Tool Selection & Params**: wrong tool, missing call, hallucinated names, wrong types, malformed JSON
3. **Tool Response Interpretation**: misreading numbers, missing entities, treating 404 as success
4. **RAG Performance**: irrelevant retrieval, outdated info, ignoring retrieved context
5. **Trajectory Efficiency**: excessive API calls, high latency, redundant steps, unhandled exceptions
6. **Multi-Agent Dynamics**: communication failures, role conflicts, loops

### Stage 3: Component-Level (Unit Evals)
Test individual components in isolation:
- Prompt quality (given input X, does the model produce good output Y?)
- Tool behavior (given state A, does the agent call the right tool with right args?)
- Retrieval quality (is the retrieved context relevant?)

### Adversarial Evals
- Prompt injection attempts
- Edge cases, ambiguous inputs
- Conflicting instructions
- Out-of-domain queries

## Dataset Design

### Minimum Viable Eval Set
Start with 50–100 examples:
```
30% — Typical / core use cases (must get these right)
30% — Edge cases (tricky inputs, ambiguous requests)
20% — Negative cases (queries agent should refuse or handle gracefully)
20% — Regression cases (known past failures now fixed)
```

### Dataset Structure
```python
@dataclass
class EvalExample:
    id: str
    input: str | dict            # user query or structured input
    expected_output: str         # ideal reference answer (optional)
    criteria: list[str]          # what "good" looks like for this example
    tags: list[str]              # for filtering (e.g., "edge-case", "tool-use")
    reference_context: str = ""  # ground truth context (for RAG evals)
```

### Creating Good Examples
1. **Start from production logs** — real user queries, not imagined ones
2. **Include golden answers** — write what the ideal response looks like
3. **Tag by category** — enables drill-down on regressions
4. **Store in version control** — eval dataset is code

## Evaluator Types (Use Hybrid Approach)

### 1. Automated Metrics
- String similarity (ROUGE, BLEU), embedding similarity (BERTScore, cosine)
- Efficient but shallow; use as trend indicators / first CI/CD gate
- Track changes: drop from 0.8 to 0.6 BERTScore = detected regression

### 2. LLM-as-Judge
**Prefer pairwise comparison** over single-scoring (mitigates biases). Run eval prompts against two versions → "which is better: A or B?" with reasoning. Calculate win/loss/tie rate — more reliable than noisy 1-5 scores.

```python
JUDGE_PROMPT = """
You are an expert evaluator for an AI assistant.

## Task
Evaluate the AI's response against the provided criteria.

## Input
{input}

## AI Response
{response}

## Reference Answer (if available)
{reference}

## Evaluation Criteria
{criteria}

## Instructions
For each criterion, rate: PASS / FAIL / PARTIAL
Provide a brief justification for each rating.

Respond in JSON:
{{
  "ratings": [
    {{"criterion": "...", "rating": "PASS|FAIL|PARTIAL", "reason": "..."}}
  ],
  "overall": "PASS|FAIL",
  "score": 0.0-1.0,
  "feedback": "One sentence summary"
}}
"""
```

**Judge calibration:**
- Test your judge on known good/bad examples — does it agree with human judgment?
- Use a **stronger model** as judge than the model being evaluated
- Avoid positional bias: run judge twice with swapped response order

**Specialized judges:**
```python
# Faithfulness — is answer grounded in context?
FAITHFULNESS_JUDGE = """Does this answer use ONLY information from the provided context?
Answer YES, PARTIALLY, or NO. Quote the parts that are/aren't grounded.
Context: {context}  Answer: {answer}"""

# Completeness — did agent finish the task?
COMPLETENESS_JUDGE = """Did the agent complete the task?
Task: {task}  Output: {output}
Rate: COMPLETE / PARTIAL / INCOMPLETE. Explain what's missing if not complete."""
```

### 3. Agent-as-Judge
One agent evaluates the full execution trace of another. Build a "Critic Agent" with a rubric asking specific process questions about the trace. Evaluates: plan quality, tool selection, context handling.

### 4. Human-in-the-Loop (HITL)
Essential for subjectivity, domain expertise, nuanced judgment. Creates the "Golden Set" of reference evaluations.
- **Interruption workflow**: agent pauses before high-stakes calls → Reviewer UI for approval
- Low-friction feedback: thumbs up/down, sliders, short comments
- **Event-driven**: thumbs-down auto-captures trace → review queue

## Rule-Based Checks (Complement to LLM Judge)

Fast, deterministic checks that should always run:
```python
def run_rule_checks(response: str, tool_calls: list) -> dict:
    return {
        "no_hallucinated_ids": not contains_fabricated_ids(response),
        "correct_tool_order": verify_tool_call_order(tool_calls),
        "response_length": 50 < len(response.split()) < 500,
        "no_pii_leakage": not contains_pii(response),
        "json_valid": is_valid_json(response) if expects_json else True,
    }
```

## Metrics

| Metric | Formula | Target |
|---|---|---|
| Pass Rate | passing / total | > 90% |
| Regression Rate | new failures / total | 0% |
| Latency P50/P95 | percentile latency | < 3s / < 10s |
| Token Efficiency | output quality / tokens used | maximize |
| Tool Accuracy | correct_tool_calls / total_tool_calls | > 95% |
| Faithfulness | grounded claims / total claims | > 95% for RAG |

```python
def compute_eval_score(results: list[EvalResult]) -> dict:
    return {
        "overall_pass_rate": sum(r.passed for r in results) / len(results),
        "by_tag": {
            tag: sum(r.passed for r in results if tag in r.tags) /
                 sum(1 for r in results if tag in r.tags)
            for tag in get_all_tags(results)
        },
        "avg_latency_ms": mean(r.latency_ms for r in results),
        "avg_tokens": mean(r.total_tokens for r in results),
    }
```

## Three Pillars of Observability

### Logs (The "What")
- Timestamped, structured (JSON), immutable
- **Log intent before action, outcome after**
- Capture: prompt/response, reasoning traces, tool calls (inputs/outputs/errors), state changes
- DEBUG in dev, INFO in prod

### Traces (The "Why")
- Follow single task end-to-end, stitching spans into causal narrative
- Built on OpenTelemetry — **Spans** + **Attributes** (prompt_id, latency_ms, token_count) + **Context Propagation** (trace_id)
- Transforms "error happened" → "error happened because X failed → caused Y → led to Z"

### Metrics (The "How Well")
- **System Metrics**: P50/P99 latency, error rate, tokens/task, cost/run, completion rate, tool usage frequency
- **Quality Metrics** (require judgment): correctness, trajectory adherence, safety, helpfulness
- **Separate dashboards**: Operational (SREs) vs Quality (PMs)
- PII scrubbing mandatory before long-term storage

## CI/CD Pipeline for Agents

### Phase 1: Pre-Merge CI (on pull request)
```yaml
name: Agent Evals
on:
  pull_request:
    paths: ['prompts/**', 'tools/**', 'agents/**']
jobs:
  eval:
    steps:
      - name: Run eval suite
        run: python -m pytest evals/ --eval-mode --output=eval-results.json
      - name: Check pass rate
        run: python scripts/check_eval_results.py eval-results.json --min-pass-rate 0.90
      - name: Post results to PR
        uses: actions/github-script@v6
```

### Phase 2: Post-Merge Staging
Deploy to staging (high-fidelity production replica), load testing, integration tests, and **dogfooding** — humans interact and provide qualitative feedback.

### Phase 3: Gated Production
Human sign-off (product owner). Exact artifact from staging promoted — no rebuilding.

### Safe Rollout Strategies
| Strategy | Approach | Best for |
|---|---|---|
| Canary | 1% users → monitor → scale/rollback | Gradual risk reduction |
| Blue-Green | Two environments, instant switch | Zero-downtime recovery |
| A/B Testing | Compare versions on business metrics | Data-driven decisions |
| Feature Flags | Deploy code, control release dynamically | Granular control |

## Eval Frameworks

| Framework | Strength |
|---|---|
| RAGAS | RAG-specific metrics (faithfulness, relevancy) |
| LangSmith | Eval datasets + LLM judge + tracing |
| Langfuse | Open source, self-hosted, tracing + evals |
| Braintrust | Dataset management + eval scoring |
| PromptFoo | CLI-based, fast iteration on prompts |
| DeepEval | Metric library with pytest integration |

## The Quality Flywheel

1. **Define Quality** → Four Pillars as concrete targets
2. **Instrument for Visibility** → Logs + Traces
3. **Evaluate the Process** → Outside-in + hybrid evaluators
4. **Architect Feedback Loop** → every production failure → permanent regression test in Golden Set

## Eval Checklist

- [ ] Eval dataset in version control?
- [ ] At least 50 examples covering core, edge, negative cases?
- [ ] LLM judge calibrated against human judgments?
- [ ] Rule-based checks implemented for objective criteria?
- [ ] Pass rate and regression rate tracked per eval run?
- [ ] CI pipeline runs evals on every prompt/tool change?
- [ ] Quality gate blocks merges below threshold?
- [ ] Failed evals converted to new dataset examples?
- [ ] Separate operational and quality dashboards in place?
- [ ] PII scrubbing before long-term trace storage?
