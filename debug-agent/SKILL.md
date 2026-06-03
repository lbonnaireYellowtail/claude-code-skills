---
name: debug-agent
description: Expert knowledge for debugging AI agents — common failure modes, tool call debugging, prompt debugging, loop detection, hallucination diagnosis, and trace analysis. Use when an agent is misbehaving, getting stuck in loops, calling wrong tools, hallucinating, or producing bad outputs.
triggers:
  - agent not working
  - debug agent
  - agent stuck
  - wrong tool
  - agent loop
  - hallucination
  - agent failure
  - bad output
  - agent debugging
  - tool call error
---

# Agent Debugger Expert

You are now loaded with deep knowledge about debugging AI agents. Apply these diagnostic patterns systematically when an agent is misbehaving.

## The Debugging Mindset

Agent failures almost always come from one of:
1. **Context problem** — wrong/missing information in context
2. **Prompt problem** — unclear/contradictory instructions
3. **Tool problem** — tool errors, bad schema, confusing output
4. **Logic problem** — incorrect workflow, missing branches
5. **Model problem** — wrong model for the task, temp too high/low

Start from the symptom and trace backwards through these layers.

## Common Failure Modes

### 1. Infinite Loops
**Symptom**: Agent keeps calling the same tool or repeating the same step.
**Causes**:
- No termination condition in loop
- Tool returns same result every time
- Agent can't detect "done" state

**Diagnosis**:
```python
# Add loop detection
call_counts = defaultdict(int)

def track_tool_call(tool_name: str):
    call_counts[tool_name] += 1
    if call_counts[tool_name] > 5:
        raise LoopDetectedError(f"Tool {tool_name} called {call_counts[tool_name]} times")
```

**Fix**: Add max iteration count, explicit termination condition, or detect when output stops changing.

### 2. Wrong Tool Selection
**Symptom**: Agent calls tool A when tool B is clearly correct.
**Causes**:
- Tool descriptions too similar or ambiguous
- Too many tools (model overwhelmed)
- Tool names not self-explanatory

**Diagnosis**:
```python
# Print tool selection rationale
# Ask model to explain choice before calling
system_prompt += "\nBefore calling any tool, explain in one sentence why that tool is the right choice."
```

**Fix**:
- Make tool names more descriptive: `search_product_catalog` not `search`
- Add explicit when-to-use guidance in tool description
- Remove tools not needed for current task

### 3. Hallucinated Tool Arguments
**Symptom**: Agent calls tool with plausible but incorrect arguments (e.g., made-up IDs).
**Causes**:
- Agent doesn't have the required data (was never retrieved)
- Schema too permissive (should have required fields)

**Diagnosis**: Check if the argument value appears anywhere in the conversation history. If not, the agent made it up.

**Fix**:
- Make required arguments truly `required` in schema
- Add validation: tool returns error with guidance when argument is invalid
- Ensure retrieval step happens before the tool that needs the data

### 4. Ignoring Tool Results
**Symptom**: Agent calls a tool, gets a result, then produces an answer that doesn't use it.
**Causes**:
- Tool result buried too deep in long conversation
- Tool result format hard to parse
- Agent "anchored" on its prior belief

**Diagnosis**: Look at what the tool returned. Is the final answer consistent with it?

**Fix**:
- Keep tool results concise and structured (JSON > prose)
- Add explicit instruction: "Always base your answer on tool results, not prior knowledge"
- Summarize tool results before passing to next reasoning step

### 5. Context Confusion (Cross-Contamination)
**Symptom**: Agent confuses data from multiple tool calls (e.g., applies user A's data to user B's result).
**Causes**:
- Ambiguous variable names in tool results
- Long conversation with many tool results

**Fix**:
- Include entity identifiers in every tool result: `{"user_id": "123", "balance": 500}`
- Summarize and clear old tool results before new task

### 6. Over-Refusal / Under-Action
**Symptom**: Agent says "I can't do that" when it has the tools to do it, or does too much without asking.
**Causes**:
- System prompt constraints too broad
- Model being overly cautious (safety vs helpfulness tension)

**Fix**:
- Be explicit about what IS allowed: "You CAN call the database tool to look up customer data"
- For under-action: add HITL checkpoints or require confirmation before high-stakes actions

### 7. Prompt Injection via Tool Results
**Symptom**: Agent suddenly changes behavior, ignores prior instructions, or starts following instructions embedded in retrieved content.

**Diagnosis**:
```python
# Log and inspect all tool outputs before they enter context
for tool_result in tool_results:
    if contains_instruction_pattern(tool_result.content):
        logger.warning(f"Potential prompt injection in tool result: {tool_result}")
```

**Fix**: Wrap tool results in XML tags, instruct model to treat tool output as data not instructions.

## Debugging Process

### Step 1: Enable Verbose Logging
```python
import logging
logging.basicConfig(level=logging.DEBUG)

# Log every LLM call
def log_llm_call(messages, response):
    logger.debug(f"INPUT:\n{format_messages(messages)}")
    logger.debug(f"OUTPUT:\n{response.content}")
    logger.debug(f"TOKENS: {response.usage}")
```

### Step 2: Inspect Full Context
The most common debugging mistake is not looking at the **full context** the model received. Print it:
```python
print("=== FULL CONTEXT SENT TO MODEL ===")
print(json.dumps(messages, indent=2))
print("=== END CONTEXT ===")
```
Read it as if you were the model. Does it make sense? Is the instruction clear?

### Step 3: Bisect the Failure
```python
# Test each component in isolation
# 1. Test the tool alone
result = my_tool.invoke(test_input)
print(result)

# 2. Test the prompt without tools
response = llm.invoke(test_prompt_without_tools)
print(response)

# 3. Test with one tool at a time
response = agent_with_one_tool.invoke(test_query)
print(response)
```

### Step 4: Temperature and Sampling
```python
# If output is inconsistent/random → lower temperature
model = ChatAnthropic(model="claude-sonnet-4-6", temperature=0)

# Run same input 5 times — if results are very different, temperature is too high
results = [agent.invoke(test_input) for _ in range(5)]
```

### Step 5: Ablation Test
Change one thing at a time:
- Remove a tool → does it still fail?
- Add an example → does quality improve?
- Simplify the task → does it work?
- Change the model → is it a capability issue?

## Tool Debugging

### Test Tool Schema
```python
# Validate your tool schema directly
from jsonschema import validate

validate(instance=test_input, schema=tool_schema)
```

### Test Error Messages
Error messages returned to the LLM are critical for recovery. Test them:
```python
# Simulate tool failure
def my_tool(args):
    try:
        return actual_implementation(args)
    except ItemNotFoundError:
        return {
            "error": f"No item found with ID {args['item_id']}.",
            "guidance": "Ask the user to confirm the item ID or search by name first."
        }
```

Ask: does the error message tell the agent exactly what to do next?

## Trace Analysis

```python
# LangSmith / LangFuse tracing
from langfuse import Langfuse

langfuse = Langfuse()
trace = langfuse.trace(name="my-agent-run")

# Add spans for each step
with trace.span(name="retrieval") as span:
    results = retriever.search(query)
    span.end(output={"chunks": len(results)})
```

Look for in traces:
- Which step took the most time?
- Where did token usage spike?
- Which tool call returned unexpected data?
- What was the model's last reasoning before failure?

## Debugging Checklist
- [ ] Full context printed and read as if you were the model?
- [ ] Each tool tested in isolation?
- [ ] Tool descriptions disambiguated (no two tools sound similar)?
- [ ] Loop detection added with max iterations?
- [ ] Tool error messages guide the agent's next action?
- [ ] Temperature set to 0 for debugging?
- [ ] Traces collected and inspected (LangSmith/LangFuse)?
- [ ] Failure reproduced consistently with same input?
- [ ] One variable changed at a time in fixes?
- [ ] Prompt injection in tool results checked?
