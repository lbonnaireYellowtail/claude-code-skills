---
name: context-optimizer
description: Expert knowledge for context window management — token budgeting, context compression, summarization strategies, prompt caching, and managing long conversations. Use when optimizing agent context usage, reducing costs, managing long conversations, or designing context engineering strategies.
triggers:
  - context window
  - token budget
  - context optimization
  - context compression
  - prompt caching
  - token limit
  - long conversation
  - context management
  - summarization
  - context engineering
---

# Context Optimizer Expert

You are now loaded with deep knowledge about context window management and optimization. Apply these strategies when helping design or debug context engineering for agents.

## Context Window Fundamentals

### What Lives in the Context Window
```
System Prompt
├── Agent identity & instructions
├── Tool definitions (ALL tools loaded)
├── Few-shot examples
└── Background knowledge

Conversation History
├── User messages
├── Assistant responses
└── Tool call results (can be huge)

Retrieved Context (RAG)
└── Relevant document chunks

Current Task
└── User's current request
```

**Key insight**: Everything in the context window costs tokens on every request and influences the model's behavior. Context = cost + quality.

### Token Budget Allocation (Example for 200K window)
```
System prompt:          5,000 tokens  (2.5%)
Tool definitions:       8,000 tokens  (4%)
Few-shot examples:      4,000 tokens  (2%)
Conversation history:  40,000 tokens  (20%)
Retrieved context:     20,000 tokens  (10%)
Current turn:           3,000 tokens  (1.5%)
Buffer for output:    120,000 tokens  (60%)
─────────────────────────────────────────
Total:                200,000 tokens
```

## Prompt Caching (Cost Optimization)

### Anthropic Prompt Caching
Cache frequently used, static portions of context:
```python
client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    system=[
        {
            "type": "text",
            "text": "You are a helpful assistant...",
            "cache_control": {"type": "ephemeral"}  # cache this
        },
        {
            "type": "text",
            "text": large_document_content,           # cache this too
            "cache_control": {"type": "ephemeral"}
        }
    ],
    messages=conversation_history  # not cached (changes each turn)
)
```

**Savings**: Cached tokens cost 90% less. Cache hits valid for 5 minutes (ephemeral) or longer (persistent).

### What to Cache
- System prompt (changes rarely)
- Tool definitions (static)
- Background documents (reference material)
- Few-shot examples (static)
- **Don't cache**: conversation history (changes every turn), user messages

### Cache Structure Rule
Cacheable content must be at the **beginning** of the prompt — put dynamic content last:
```
[CACHED] System prompt
[CACHED] Tool definitions
[CACHED] Reference documents
[CACHED] Few-shot examples
[DYNAMIC] Conversation history
[DYNAMIC] Current user message
```

## Conversation History Management

### Strategy 1: Rolling Window
Keep only the last N messages:
```python
MAX_HISTORY_TOKENS = 40_000

def trim_history(messages: list, max_tokens: int) -> list:
    total = 0
    trimmed = []
    for msg in reversed(messages):
        tokens = count_tokens(msg)
        if total + tokens > max_tokens:
            break
        trimmed.insert(0, msg)
        total += tokens
    return trimmed
```
**Problem**: Loses early context (original task, key decisions).

### Strategy 2: Progressive Summarization
Summarize older turns, keep recent ones verbatim:
```python
async def compress_history(messages: list, keep_recent: int = 10) -> list:
    if len(messages) <= keep_recent:
        return messages

    old_messages = messages[:-keep_recent]
    recent_messages = messages[-keep_recent:]

    summary = await llm.ainvoke(f"""
    Summarize these conversation turns concisely.
    Preserve: decisions made, key facts, user preferences, current task status.

    Turns to summarize:
    {format_messages(old_messages)}
    """)

    return [
        {"role": "system", "content": f"[Previous conversation summary]\n{summary}"},
        *recent_messages
    ]
```

### Strategy 3: Selective Retention
Keep only "important" messages:
```python
# Tag messages with importance at creation time
def is_important(message: dict) -> bool:
    # Keep: task definition, key decisions, errors, tool results with data
    # Drop: pleasantries, confirmations, intermediate steps
    return any([
        message.get("important"),
        "error" in message["content"].lower(),
        message["role"] == "tool" and len(message["content"]) < 500,
    ])
```

### Strategy 4: External Memory (Best for Long Agents)
Don't keep everything in context — store in external memory:
```python
# After each conversation turn, extract and store key facts
async def extract_and_store(user_msg: str, assistant_msg: str, memory_db):
    facts = await llm.ainvoke(f"""
    Extract key facts from this exchange worth remembering.
    Return as JSON list: [{{"fact": "...", "type": "preference|decision|data"}}]

    User: {user_msg}
    Assistant: {assistant_msg}
    """)
    await memory_db.upsert(facts)

# At start of each turn, retrieve relevant memories
async def get_relevant_context(query: str, memory_db) -> str:
    memories = await memory_db.search(query, top_k=10)
    return format_memories(memories)
```

## Tool Definition Optimization

Tool schemas are loaded into every request and can bloat context:

### Problem: Too Many Tools
```
50 tools × 200 tokens each = 10,000 tokens per request
```

### Solution: Dynamic Tool Loading
```python
# Retrieve only relevant tools based on the current task
def get_relevant_tools(user_query: str, all_tools: list) -> list:
    query_embedding = embed(user_query)
    tool_embeddings = [embed(t["description"]) for t in all_tools]
    similarities = cosine_similarity(query_embedding, tool_embeddings)
    top_k_indices = similarities.argsort()[-10:][::-1]
    return [all_tools[i] for i in top_k_indices]
```

### Solution: Tool Groups / Namespacing
```python
# Load tool groups based on detected intent
TOOL_GROUPS = {
    "search": [web_search, doc_search, db_query],
    "write": [create_file, edit_file, append_file],
    "communicate": [send_email, send_slack, create_ticket],
}

active_tools = TOOL_GROUPS.get(detected_intent, TOOL_GROUPS["search"])
```

## Context Injection Patterns

### System Prompt Injection
Inject dynamic context at agent initialization:
```python
def build_system_prompt(user_context: UserContext) -> str:
    return f"""
You are a {user_context.role} assistant.

Current user: {user_context.name}
Organization: {user_context.org}
Permissions: {', '.join(user_context.permissions)}
Current date: {datetime.now().strftime('%Y-%m-%d')}

Relevant background:
{retrieve_relevant_background(user_context)}
"""
```

### Per-Turn Context Injection
Inject retrieved context into user turns (not system):
```python
def build_user_message(query: str, retrieved_context: list) -> str:
    context_text = "\n\n".join(retrieved_context)
    return f"""
<context>
{context_text}
</context>

<question>
{query}
</question>
"""
```

## Token Counting

```python
import anthropic

client = anthropic.Anthropic()

# Count tokens before sending
response = client.messages.count_tokens(
    model="claude-sonnet-4-6",
    system=system_prompt,
    messages=messages,
    tools=tools,
)
print(f"Input tokens: {response.input_tokens}")
```

## Context Budget Monitoring

```python
@dataclass
class ContextBudget:
    total: int = 200_000
    system_used: int = 0
    history_used: int = 0
    retrieved_used: int = 0

    @property
    def remaining(self) -> int:
        return self.total - self.system_used - self.history_used - self.retrieved_used

    def warn_if_low(self, threshold: float = 0.8):
        if (self.system_used + self.history_used + self.retrieved_used) / self.total > threshold:
            logger.warning(f"Context at {self.used_pct:.0%} — consider compression")
```

## Checklist
- [ ] Token budget defined and allocated per section?
- [ ] Prompt caching applied to static sections?
- [ ] Conversation history trimmed/summarized after N turns?
- [ ] Tool definitions dynamically loaded (not all at once)?
- [ ] Retrieved context capped at a max token budget?
- [ ] Context usage monitored and logged per request?
- [ ] Long-running agents use external memory (not just context)?
- [ ] Token counting tested before deploying to production?
