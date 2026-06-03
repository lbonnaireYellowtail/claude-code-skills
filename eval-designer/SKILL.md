---
name: eval-designer
description: Expert knowledge for designing agent evaluation systems — eval datasets, LLM-as-judge patterns, CI/CD for agents, evaluation metrics, and regression testing. Use when building eval pipelines for agents, setting up quality gates in CI, designing test datasets, or measuring agent performance.
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
---

# Eval Designer Expert

You are now loaded with deep knowledge about AI agent evaluation. Apply these patterns when helping design or implement evaluation systems.

## Why Evaluation Matters

Agents degrade silently — a prompt change, model update, or new tool can break behavior without obvious errors. Evaluation is how you detect regressions before users do.

**Without evals**: you're flying blind. With evals: you can ship with confidence.

## The Eval Mindset

**Evals are tests for LLM-based systems.** Just like unit tests, they should:
- Run automatically on every change
- Be deterministic (or statistically stable)
- Cover the most important behaviors
- Catch regressions early

## Types of Evaluations

### 1. Unit Evals (Component-Level)
Test individual components in isolation:
- Prompt quality (given input X, does the model produce good output Y?)
- Tool behavior (given state A, does the agent call the right tool with right args?)
- Retrieval quality (is the retrieved context relevant?)

### 2. Integration Evals (End-to-End)
Test the full agent workflow:
- Given user query X, does the final answer meet quality bar?
- Does the agent complete the task in acceptable steps?

### 3. Adversarial Evals
- Prompt injection attempts
- Edge cases, ambiguous inputs
- Conflicting instructions
- Out-of-domain queries

## Dataset Design

### Minimum Viable Eval Set
Start with 50–100 examples. Cover:
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

```yaml
# evals/dataset/user-lookup.yaml
- id: user-lookup-001
  input: "What's John Smith's current account balance?"
  criteria:
    - Uses get_user_by_name tool before get_balance
    - Returns balance in dollars (not cents)
    - Does not hallucinate account number
  tags: [tool-use, data-lookup]

- id: user-lookup-002
  input: "Check balance for user john.smith@example.com"
  criteria:
    - Uses email to look up user correctly
    - Handles case where user not found gracefully
  tags: [tool-use, edge-case]
```

## LLM-as-Judge

### When to Use LLM-as-Judge
- Output quality is subjective (helpfulness, tone, clarity)
- No single correct answer
- Checking criteria that are hard to code (e.g., "is this response factual?")

### Judge Prompt Template
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

### Judge Calibration
- Test your judge on known good/bad examples — does it agree with human judgment?
- Use a **stronger model** as judge than the model being evaluated
- Avoid positional bias: run judge twice with swapped response order

```python
async def judge_with_consistency_check(input, response_a, reference):
    judge_1 = await llm_judge(input, response_a, reference)
    # Slight rephrasing to check consistency
    judge_2 = await llm_judge(input, response_a, reference, variant=True)
    if judge_1["overall"] != judge_2["overall"]:
        # Inconsistent — flag for human review
        return {"overall": "UNCERTAIN", "judges": [judge_1, judge_2]}
    return judge_1
```

### Specialized Judges
```python
# Faithfulness judge — is answer grounded in context?
FAITHFULNESS_JUDGE = """
Does this answer use ONLY information from the provided context?
Answer YES, PARTIALLY, or NO. Quote the parts that are/aren't grounded.
Context: {context}
Answer: {answer}
"""

# Completeness judge — did agent complete the task?
COMPLETENESS_JUDGE = """
Given the task description and the agent's final output, did the agent complete the task?
Task: {task}
Output: {output}
Rate: COMPLETE / PARTIAL / INCOMPLETE. Explain what's missing if not complete.
"""
```

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

### Core Metrics to Track
| Metric | Formula | Target |
|---|---|---|
| Pass Rate | passing / total | > 90% |
| Regression Rate | new failures / total | 0% |
| Latency P50/P95 | percentile latency | < 3s / < 10s |
| Token Efficiency | output quality / tokens used | maximize |
| Tool Accuracy | correct_tool_calls / total_tool_calls | > 95% |
| Faithfulness | grounded claims / total claims | > 95% for RAG |

### Aggregate Scores
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

## CI/CD Integration

### GitHub Actions Eval Pipeline
```yaml
name: Agent Evals

on:
  pull_request:
    paths:
      - 'prompts/**'
      - 'tools/**'
      - 'agents/**'

jobs:
  eval:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Run eval suite
        run: python -m pytest evals/ --eval-mode --output=eval-results.json
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}

      - name: Check pass rate
        run: python scripts/check_eval_results.py eval-results.json --min-pass-rate 0.90

      - name: Post results to PR
        uses: actions/github-script@v6
        with:
          script: |
            const results = require('./eval-results.json');
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              body: formatEvalResults(results)
            });
```

### Quality Gate
```python
def quality_gate(results: EvalResults, thresholds: dict) -> bool:
    if results.overall_pass_rate < thresholds["min_pass_rate"]:
        print(f"FAIL: Pass rate {results.overall_pass_rate:.0%} < {thresholds['min_pass_rate']:.0%}")
        return False
    if results.regression_count > 0:
        print(f"FAIL: {results.regression_count} regressions detected")
        return False
    return True
```

## Eval Frameworks

| Framework | Strength |
|---|---|
| RAGAS | RAG-specific metrics (faithfulness, relevancy) |
| LangSmith | Eval datasets + LLM judge + tracing |
| Langfuse | Open source, self-hosted, tracing + evals |
| Braintrust | Dataset management + eval scoring |
| PromptFoo | CLI-based, fast iteration on prompts |
| DeepEval | Metric library with pytest integration |

## Eval Checklist
- [ ] Eval dataset in version control?
- [ ] At least 50 examples covering core, edge, negative cases?
- [ ] LLM judge calibrated against human judgments?
- [ ] Rule-based checks implemented for objective criteria?
- [ ] Pass rate and regression rate tracked per eval run?
- [ ] CI pipeline runs evals on every prompt/tool change?
- [ ] Quality gate blocks merges below threshold?
- [ ] Eval results posted to PRs for visibility?
- [ ] Failed evals converted to new dataset examples?
- [ ] Eval dataset grows with each discovered bug?
