---
name: prompt-engineer
description: Expert knowledge for prompt engineering — system prompt design, few-shot examples, chain-of-thought, output formatting, Claude-specific best practices, and common anti-patterns. Use when writing or improving system prompts, designing few-shot examples, structuring agent instructions, or debugging prompt failures.
triggers:
  - prompt engineering
  - system prompt
  - few-shot
  - chain of thought
  - prompt design
  - improve prompt
  - prompt optimization
  - instruction design
  - prompt template
  - output format
---

# Prompt Engineering Expert

You are now loaded with deep knowledge about prompt engineering. Apply these principles when helping the user write or improve prompts for AI systems.

## System Prompt Architecture

### Core Sections (in order)
```
1. ROLE / PERSONA       — who the model is
2. CONTEXT              — background knowledge, relevant facts
3. TASK / OBJECTIVE     — what it must accomplish
4. CONSTRAINTS          — what it must NOT do
5. OUTPUT FORMAT        — structure, length, style of response
6. EXAMPLES             — few-shot demonstrations (most impactful section)
7. EDGE CASES           — how to handle ambiguous or tricky situations
```

### Template
```xml
<system>
You are [ROLE] at [CONTEXT].

Your job is to [PRIMARY TASK].

## Constraints
- [What NOT to do]
- [Boundaries]

## Output Format
[Describe structure, length, tone, format exactly]

## Examples
[Few-shot examples here — most important section]
</system>
```

## Claude-Specific Best Practices

### XML Tags for Structure
Claude responds well to XML-structured prompts:
```xml
<context>
  The user is a software engineer asking about their codebase.
</context>
<task>
  Answer the question below using only the provided code context.
  If the answer isn't in the context, say "I don't know."
</task>
<code_context>
  {retrieved_code}
</code_context>
<question>
  {user_question}
</question>
```

### Be Direct and Specific
```
# BAD — vague
"Be helpful and answer questions."

# GOOD — specific
"Answer questions about Angular component architecture.
Always include a code example. Limit responses to 300 words."
```

### Use Positive Instructions
```
# BAD — negative
"Don't be verbose. Don't use jargon. Don't skip examples."

# GOOD — positive
"Be concise. Use plain language. Include one code example per answer."
```

### Extended Thinking (Claude 3.7+)
For complex reasoning tasks, enable extended thinking:
```python
response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=16000,
    thinking={"type": "enabled", "budget_tokens": 10000},
    messages=[{"role": "user", "content": prompt}]
)
```
Use for: math, logic puzzles, multi-step planning, complex code generation.

## Few-Shot Examples

### Why They Work
Examples show the model the **pattern** of correct behavior. The more specific the task, the more examples matter.

**Rule of thumb**: Add examples when accuracy needs to be > 90%.

### Example Structure
```
Input: [realistic input]
Output: [ideal output]

Input: [edge case input]
Output: [ideal edge case handling]

Input: [another variant]
Output: [another ideal output]
```

### Example Quality Rules
- **Diverse**: cover different input types, edge cases, tones
- **Representative**: match real production inputs, not toy examples
- **Consistent**: output format identical across all examples
- **Labeled**: mark edge cases explicitly in examples

### Dynamic Few-Shot Selection
Don't hardcode examples — retrieve relevant ones:
```python
def get_few_shot_examples(user_input: str, example_bank: list) -> list:
    # Embed user_input and example_bank
    # Return top-K most similar examples
    similarities = cosine_similarity(embed(user_input), embed_bank)
    return [example_bank[i] for i in similarities.argsort()[-3:][::-1]]
```

## Chain of Thought (CoT)

### Zero-Shot CoT
```
"Think step by step before giving your final answer."
"Before responding, reason through the problem carefully."
```

### Few-Shot CoT
```
Q: If a product costs $12.50 and we apply a 15% discount, what's the final price?
A: Let me think step by step.
   1. Discount amount = 12.50 × 0.15 = $1.875
   2. Final price = 12.50 - 1.875 = $10.625
   3. Rounded to cents: $10.63
   Final answer: $10.63

Q: {user_question}
A: Let me think step by step.
```

### When to Use CoT
| Use CoT | Skip CoT |
|---|---|
| Math / logic | Simple lookup |
| Multi-step reasoning | Classification |
| Planning | Extraction |
| Code debugging | Formatting |

## Output Format Control

### Structured Output (JSON)
```
"Respond in valid JSON with this exact schema:
{
  \"intent\": string,       // user's primary intent
  \"entities\": string[],  // key entities mentioned
  \"confidence\": number   // 0.0 to 1.0
}
No explanation, just the JSON object."
```

### Length Control
```
"Respond in exactly 3 bullet points."
"Keep your response under 100 words."
"Respond in one sentence."
"Give a detailed response with sections for: Overview, Steps, Example, Gotchas."
```

### Tone Control
```
"Respond as a senior engineer reviewing a junior's PR — direct, constructive, respectful."
"Respond as a Socratic tutor — ask questions rather than giving answers."
```

## Temperature Guide

| Task | Temperature |
|---|---|
| Code generation | 0.0–0.2 |
| Factual Q&A | 0.0–0.3 |
| Summarization | 0.3–0.5 |
| Creative writing | 0.7–1.0 |
| Brainstorming | 0.8–1.0 |

## Prompt Anti-Patterns

| Anti-Pattern | Problem | Fix |
|---|---|---|
| "Be helpful" | Too vague | Specify the task precisely |
| Negative-only constraints | Model focuses on what NOT to do | Add positive instructions |
| No output format | Unpredictable structure | Define exact output schema |
| No examples | High variance | Add 2–5 examples |
| Too many instructions | Model misses some | Prioritize, use numbered lists |
| Contradictory instructions | Model guesses which to follow | Audit for conflicts |
| Jargon-heavy | Model interprets differently | Use plain language |

## Prompt Iteration Process

1. **Baseline**: Write a simple prompt, test against 10 inputs
2. **Analyze failures**: cluster failure modes
3. **Hypothesize**: which part of the prompt caused each failure?
4. **Fix one thing at a time**: isolate changes
5. **Regression test**: new fix doesn't break previous working cases
6. **Repeat**

**Track versions**: keep a `prompts/` directory with versioned prompt files.

## Checklist
- [ ] Role and context clearly defined?
- [ ] Task stated specifically (not vaguely)?
- [ ] Output format described exactly?
- [ ] At least 2–3 few-shot examples included?
- [ ] Examples cover at least one edge case?
- [ ] No contradictory instructions?
- [ ] Temperature appropriate for task?
- [ ] Prompt tested against at least 10 real inputs?
- [ ] Failure modes analyzed and addressed?
