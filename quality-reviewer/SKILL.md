---
name: quality-reviewer
description: Expert knowledge for evaluating agent quality, building evaluation systems, implementing observability, and setting up CI/CD for agents. Use when the user wants to assess agent quality, build eval pipelines, add observability, or deploy agents to production.
triggers:
  - evaluate agent
  - agent quality
  - build eval
  - observability
  - CI/CD for agents
  - agent metrics
  - agent monitoring
  - deploy agent
---

# Agent Quality & Operations Expert

You are now loaded with deep knowledge about agent evaluation, observability, and production operations. Apply these when helping the user.

## Four Pillars of Agent Quality
1. **Effectiveness** (Goal Achievement): did agent achieve user's actual intent? Connect to business KPIs (conversion rate, completion rate)
2. **Efficiency** (Operational Cost): tokens consumed, wall-clock time, trajectory complexity. 25 steps for a simple task = low quality even if it succeeds
3. **Robustness** (Reliability): graceful handling of API timeouts, missing data, ambiguous prompts. Retry, clarify, or report inability — never crash/hallucinate
4. **Safety & Alignment** (Trustworthiness): non-negotiable gate. Fairness, bias, prompt injection resistance, data leakage prevention

## Evaluation: The Outside-In Hierarchy

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

## Evaluator Types (use hybrid approach)

### 1. Automated Metrics
- String similarity (ROUGE, BLEU), embedding similarity (BERTScore, cosine)
- Efficient but shallow; use as trend indicators / first CI/CD gate
- Track changes: drop from 0.8 to 0.6 BERTScore = detected regression

### 2. LLM-as-a-Judge
- Powerful model evaluates against rubric
- **Prefer pairwise comparison** over single-scoring (mitigates biases)
- Run eval prompts against two versions → "which is better: A or B?" with reasoning
- Calculate win/loss/tie rate — more reliable than noisy 1-5 scores

### 3. Agent-as-a-Judge
- One agent evaluates the full execution trace of another
- Build "Critic Agent" with rubric asking specific process questions about the trace
- Evaluates: plan quality, tool selection, context handling

### 4. Human-in-the-Loop (HITL)
- Essential for subjectivity, domain expertise, nuanced judgment
- Creates the "Golden Set" of reference evaluations
- **Interruption workflow**: agent pauses before high-stakes calls → Reviewer UI for approval

### User Feedback System
- Low-friction: thumbs up/down, sliders, short comments
- Context-rich: pair with full conversation + reasoning trace
- **Event-driven**: thumbs-down auto-captures trace → review queue

## Three Pillars of Observability

### Logs (The "What")
- Timestamped, structured (JSON), immutable
- **Log intent before action, outcome after** (clarifies failed attempt vs deliberate inaction)
- Capture: prompt/response, reasoning traces, tool calls (inputs/outputs/errors), state changes
- DEBUG in dev, INFO in prod

### Traces (The "Why")
- Follow single task end-to-end, stitching spans into causal narrative
- Built on OpenTelemetry
- **Spans** (named operations) + **Attributes** (prompt_id, latency_ms, token_count) + **Context Propagation** (trace_id)
- Transforms "error happened" → "error happened because X failed → caused Y → led to Z"

### Metrics (The "How Well")
- **System Metrics**: P50/P99 latency, error rate, tokens/task, cost/run, completion rate, tool usage frequency
- **Quality Metrics** (require judgment): correctness, trajectory adherence, safety, helpfulness

### Operational Practices
- **Separate dashboards**: Operational (SREs: latency, errors, cost) vs Quality (PMs: correctness, helpfulness)
- PII scrubbing mandatory before long-term storage
- **Dynamic sampling**: 100% errors traced, X% successes; high granularity dev, strategic sampling prod

## CI/CD Pipeline for Agents

### Phase 1: Pre-Merge CI (on pull request)
- Unit tests, linting, dependency scanning
- **Agent quality evaluation suite** — immediate performance feedback
- Catches issues before main branch pollution

### Phase 2: Post-Merge Staging
- Deploy to staging (high-fidelity production replica)
- Load testing, integration tests against remote services
- **Dogfooding**: humans interact and provide qualitative feedback

### Phase 3: Gated Production
- Human sign-off (product owner)
- Exact artifact from staging promoted (no rebuilding)

### Safe Rollout Strategies
| Strategy | Approach | Best for |
|---|---|---|
| Canary | 1% users → monitor → scale/rollback | Gradual risk reduction |
| Blue-Green | Two environments, instant switch | Zero-downtime recovery |
| A/B Testing | Compare versions on business metrics | Data-driven decisions |
| Feature Flags | Deploy code, control release dynamically | Granular control |

All require **rigorous versioning**: code, prompts, model endpoints, tool schemas, memory structures, eval datasets.

## The Quality Flywheel
1. **Define Quality** → Four Pillars as concrete targets
2. **Instrument for Visibility** → Logs + Traces
3. **Evaluate the Process** → Outside-in + hybrid evaluators
4. **Architect Feedback Loop** → every production failure → permanent regression test in Golden Set
