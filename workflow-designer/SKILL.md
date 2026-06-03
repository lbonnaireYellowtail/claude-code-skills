---
name: workflow-designer
description: Expert knowledge for designing multi-step agent workflows — DAG design, conditional branching, parallel execution, error handling, state management, and workflow orchestration frameworks. Use when designing complex agent pipelines, multi-step automation, or orchestration logic.
triggers:
  - workflow design
  - agent workflow
  - multi-step pipeline
  - DAG
  - workflow orchestration
  - parallel agents
  - conditional branching
  - agent pipeline
  - LangGraph
  - workflow state
---

# Workflow Designer Expert

You are now loaded with deep knowledge about agent workflow design. Apply these patterns when helping the user design or implement multi-step agent workflows.

## Workflow Fundamentals

### When to Use Workflows vs Single-Agent
| Use Workflow | Use Single Agent |
|---|---|
| Task requires > 3 distinct steps | Simple request-response |
| Steps have different skill requirements | One domain of knowledge |
| Steps can run in parallel | Sequential by nature |
| Steps need human approval | Fully automated |
| Output of step A feeds step B | Single transformation |

### Core Workflow Primitives
- **Node**: a unit of work (LLM call, tool call, human step)
- **Edge**: dependency between nodes
- **State**: data passed between nodes
- **Branch**: conditional routing based on state
- **Loop**: repeat a node until condition met

## DAG Design Principles

### Design from the Goal Backwards
1. What is the final output?
2. What inputs does the final step need?
3. What produces those inputs?
4. Repeat until you reach inputs you have

### Minimize Sequential Depth
```
BAD (sequential):  A → B → C → D → E    (5 steps in series)
GOOD (parallel):   A → [B, C, D] → E     (only 3 steps deep)
```

Parallelism = speed. Identify which steps have no dependencies on each other.

### State Schema Design
Define state upfront — the "blood" of the workflow:
```python
from dataclasses import dataclass, field
from typing import Annotated

@dataclass
class WorkflowState:
    # Inputs
    user_query: str
    user_id: str

    # Accumulated during workflow
    retrieved_docs: list[str] = field(default_factory=list)
    draft_answer: str = ""
    critique: str = ""
    final_answer: str = ""

    # Control
    iteration_count: int = 0
    errors: list[str] = field(default_factory=list)
    status: str = "running"  # running | completed | failed | waiting_human
```

## LangGraph (Recommended for Complex Workflows)

```python
from langgraph.graph import StateGraph, END

builder = StateGraph(WorkflowState)

# Add nodes
builder.add_node("retrieve", retrieve_documents)
builder.add_node("generate", generate_answer)
builder.add_node("critique", critique_answer)
builder.add_node("refine", refine_answer)

# Add edges
builder.set_entry_point("retrieve")
builder.add_edge("retrieve", "generate")
builder.add_edge("generate", "critique")

# Conditional branching
def should_refine(state: WorkflowState) -> str:
    if state.iteration_count >= 3:
        return "done"
    if "significant issues" in state.critique.lower():
        return "refine"
    return "done"

builder.add_conditional_edges("critique", should_refine, {
    "refine": "refine",
    "done": END,
})
builder.add_edge("refine", "critique")  # loop back

graph = builder.compile()
```

## Parallel Execution

### LangGraph Parallel Fan-Out
```python
# Nodes that can run in parallel — add them all after the same source node
builder.add_edge("start", "search_web")
builder.add_edge("start", "search_db")
builder.add_edge("start", "search_docs")

# Merge results
builder.add_node("merge", merge_search_results)
builder.add_edge("search_web", "merge")
builder.add_edge("search_db", "merge")
builder.add_edge("search_docs", "merge")
```

### Python asyncio for Parallel Agent Calls
```python
import asyncio

async def run_parallel_agents(inputs: list) -> list:
    tasks = [agent.ainvoke(input) for input in inputs]
    results = await asyncio.gather(*tasks, return_exceptions=True)
    # Filter out exceptions, log failures
    return [r for r in results if not isinstance(r, Exception)]
```

## Branching Patterns

### Content Router
```python
def route_by_intent(state: WorkflowState) -> str:
    intent = state.detected_intent
    routes = {
        "question": "answer_agent",
        "task": "task_agent",
        "complaint": "support_agent",
        "feedback": "feedback_agent",
    }
    return routes.get(intent, "fallback_agent")

builder.add_conditional_edges("classify_intent", route_by_intent, {
    "answer_agent": "answer_agent",
    "task_agent": "task_agent",
    "support_agent": "support_agent",
    "feedback_agent": "feedback_agent",
    "fallback_agent": "fallback_agent",
})
```

### Quality Gate
```python
def quality_gate(state: WorkflowState) -> str:
    score = evaluate_output(state.draft_answer, state.user_query)
    state.quality_score = score

    if score > 0.8:
        return "approved"
    elif state.revision_count >= 3:
        return "max_revisions"  # escape hatch!
    else:
        return "needs_revision"
```

**Always add escape hatches** to loops to prevent infinite iteration.

## Human-in-the-Loop (HITL)

### Approval Gate Pattern
```python
# Node that pauses workflow for human approval
async def human_approval_node(state: WorkflowState) -> WorkflowState:
    # Persist state (workflow is paused here)
    await db.save_pending_approval({
        "workflow_id": state.workflow_id,
        "action": state.proposed_action,
        "state": state.to_dict(),
    })
    # Send notification (Slack, email, etc.)
    await notify_human(state.proposed_action)
    # Signal workflow to pause (implementation depends on framework)
    state.status = "waiting_human"
    return state

# Human approves via API → workflow resumes from this point
```

### HITL Triggers (When to Pause)
- High-cost or irreversible actions (delete, payment, email to customer)
- Low-confidence decisions (below threshold)
- Novel situations (no historical precedent in logs)
- Regulatory requirements (financial, medical)

## Error Handling

### Try-Retry Pattern
```python
import tenacity

@tenacity.retry(
    stop=tenacity.stop_after_attempt(3),
    wait=tenacity.wait_exponential(multiplier=1, min=1, max=10),
    retry=tenacity.retry_if_exception_type(RateLimitError),
)
async def call_llm_with_retry(prompt: str) -> str:
    return await llm.ainvoke(prompt)
```

### Fallback Pattern
```python
async def retrieve_with_fallback(query: str) -> list[str]:
    try:
        return await primary_retriever.search(query)
    except Exception as e:
        logger.warning(f"Primary retriever failed: {e}")
        return await fallback_retriever.search(query)  # simpler/more reliable
```

### Dead Letter / Error State
```python
def error_handler(state: WorkflowState, error: Exception) -> WorkflowState:
    state.errors.append(str(error))
    state.status = "failed"
    # Persist for debugging
    save_failed_workflow(state)
    # Alert
    send_alert(f"Workflow {state.workflow_id} failed: {error}")
    return state
```

## Workflow State Persistence

For long-running or resumable workflows:
```python
# LangGraph checkpointing
from langgraph.checkpoint.sqlite import SqliteSaver

memory = SqliteSaver.from_conn_string("workflows.db")
graph = builder.compile(checkpointer=memory)

# Run with thread_id for resumability
config = {"configurable": {"thread_id": "workflow-123"}}
result = graph.invoke(initial_state, config=config)

# Resume after human approval
graph.invoke(None, config=config)  # resumes from last checkpoint
```

## Workflow Design Checklist
- [ ] State schema defined upfront with all fields?
- [ ] Independent steps run in parallel?
- [ ] All loops have escape hatches (max iterations)?
- [ ] Error handling with retry + fallback for each node?
- [ ] HITL gates defined for high-risk actions?
- [ ] State persisted for long-running workflows?
- [ ] Workflow observable (logs, traces per node)?
- [ ] Idempotent nodes where possible (safe to retry)?
- [ ] Tested with failure scenarios (node failures, timeouts)?
