---
name: memory-designer
description: Expert knowledge for designing context engineering, session management, and memory systems for AI agents. Use when the user wants to build memory, implement context management, design RAG+memory hybrid systems, or handle session persistence.
triggers:
  - build memory system
  - context engineering
  - context management
  - session management
  - memory architecture
  - RAG and memory
  - compaction strategy
---

# Context Engineering & Memory Systems Expert

You are now loaded with deep knowledge about context engineering and memory systems. Apply these patterns when helping the user design or build these systems.

## Context Engineering
The practice of dynamically assembling and managing the ENTIRE payload in an LLM's context window. It answers: "what configuration of context is most likely to generate the model's desired behavior?"

### What Goes Into Context
- **Reasoning guidance**: system instructions, tool definitions, few-shot examples
- **Evidential data**: long-term memory, RAG results, tool outputs, sub-agent outputs
- **Immediate conversation**: history, state/scratchpad, user's prompt

### The Context Lifecycle (per turn)
1. **Fetch** — retrieve memories, RAG docs, recent events (dynamic retrieval using query + metadata)
2. **Prepare** — construct full prompt (blocking, hot-path — can't proceed until ready)
3. **Invoke LLM + Tools** — iteratively call until final response; append outputs to context
4. **Upload** — persist new info to storage (background/async — agent can finish while this runs)

### Context Rot
As context grows: cost increases, latency increases, attention to critical info diminishes.
- Effective context windows are ~50-60% of stated size
- Don't start complex tasks when half-way through conversation
- Performance drops correlate with length, not task difficulty

### Counter-Strategies
- Plug only the most relevant context; reduce bloat
- Few, non-conflicting instructions
- **Attention manipulation via recitation**: continuously rewrite todo/plan into recent context (Manus pattern — pushes objectives into model's recent attention span)
- Runtime injection of reminders into tool results and messages
- Use external storage for large data, return references only

## Session Design

### Definition
Container for a single conversation: chronological event history + working memory (state/scratchpad).
- **Events**: user input, agent response, tool call, tool output
- **State**: structured working memory (temporary, mutable)

### Compaction Strategies
| Strategy | Approach | Trade-off |
|---|---|---|
| Sliding window | Keep last N turns | Simple but loses old context |
| Token truncation | Count backward, cut at budget | Precise budget control |
| Recursive summarization | LLM summarizes old → prefix to recent verbatim | Preserves essence, costs inference |

Trigger mechanisms: count-based (tokens/turns), time-based (inactivity), event-based (task concludes).
Best practice: run compaction **async in background**, track which events are already summarized.

### Production Requirements
- Strict user-level isolation (ACLs), authenticate every request
- Redact PII before persisting; TTL policies for auto-deletion
- Session is on hot path — must be fast
- Stateless runtimes retrieve full history each turn

## Memory Architecture

### Core Concept
Memory = extracted, meaningful information persisted across sessions. NOT raw dialogue — processed insights.

### Memory vs RAG
| | RAG | Memory |
|---|---|---|
| Goal | Inject factual knowledge | Personalized experience |
| Data | Static, pre-indexed KB | User-agent dialogue |
| Isolation | Shared/global (read-only) | Per-user (isolated) |
| Write | Batch/offline | Event-based |
| Read | Agent decides (tool) | Always-on or agent decides |

**RAG = expert on facts. Memory = expert on the user.** Build both.

### Memory Classification

**By kind (cognitive science)**:
- **Declarative** ("knowing what"): facts, events → answers "what" questions
- **Procedural** ("knowing how"): workflows, playbooks → answers "how" questions, guides actions

**By organization**:
- **Collections**: pool of self-contained NL memories (searchable)
- **Structured profile**: contact-card core facts, continuously updated
- **Rolling summary**: single evolving document of relationship

**By storage**:
- **Vector DB**: semantic similarity search (unstructured, conceptual)
- **Knowledge graph**: entity-relationship networks (structured, relational)
- **Hybrid**: graph entities + vector embeddings (both)

**By scope**: User-level (most common) | Session-level | Application-level (global, sanitize sensitive content)

### Memory Generation Pipeline (LLM-driven ETL)
1. **Extraction** — targeted intelligent filtering, NOT summarization. "What is meaningful enough to become a memory?" Defined by topic definitions, few-shot prompting
2. **Consolidation** (most sophisticated) — compare new vs existing: UPDATE, CREATE, DELETE/INVALIDATE. Addresses: duplication, conflict, evolution, relevance decay. **Forgetting is important**
3. **Storage** — persist to vector DB or knowledge graph
4. **Retrieval** — blend scores: relevance (semantic) + recency (time) + importance (significance). Never rely solely on vector similarity. Advanced: query rewriting, reranking (add latency; cache)

### Retrieval Timing
- **Proactive** (always-on): load at every turn. Simple but adds latency for turns that don't need memory
- **Reactive** (memory-as-a-tool): agent decides when. Efficient but agent may not know what's available
- **Hybrid**: system prompt for stable/global, tool for transient/episodic

### Injecting into Context
- **System instructions**: high authority, best for stable info. Risk: over-influence
- **Conversation history**: flexible, good for episodic. Risk: confusion, tokens
- **Tool output**: natural for memory-as-a-tool
- **Hybrid recommended**: stable in system prompt, transient via tool

### Memory Provenance & Trust
Source hierarchy: bootstrapped data (high) > explicit user input > implicit extraction > tool output (low)
Confidence evolves: increases via corroboration, decreases via decay/contradiction.
Inject confidence scores so LLM can weigh reliability.

### Memory-as-a-Tool Pattern
Expose `create_memory(fact)` and `load_memory(query)` — agent autonomously decides what to remember and when to retrieve.

## New Developments (2026)

### Episodic Memory
Agents that learn from past experiences to improve future decisions (not just store facts). Bedrock AgentCore offers this as a managed service. Research (MemRL) explores self-evolving memory via reinforcement learning.

### Sparse RAG
Filter low-relevance content BEFORE self-attention, retaining only high-signal tokens. Models like RQ-RAG achieve 800%+ improvement on multi-hop QA. Key insight: RAG is evolving from retrieval pattern into autonomous agent capability (agents decide what to ask, which tools to use, and aggregate autonomously).

### Context Compaction Techniques (2026)
- **80% threshold**: compact at 80% capacity, add prior context as virtual file for later retrieval
- **Context folding**: continual summarization within the window itself
- **Model-native compaction**: GPT-5.2-Codex manages its own context (paradigm shift)
- **Progress file pattern**: `progress.txt` + version control as cross-session state (Anthropic harness pattern)

### Mem0 (Production Memory)
Scalable extraction + consolidation + retrieval: 26% accuracy boost, 91% lower p95 latency, 90% token savings vs full-context approaches.

### Production Checklist
- [ ] Memory generation is async/non-blocking (post-response)
- [ ] Concurrency handled (transactional ops or optimistic locking)
- [ ] Retry with exponential backoff + dead-letter queue
- [ ] Per-user ACLs, never cross-contaminate
- [ ] User opt-out/deletion supported
- [ ] PII redacted before persisting
- [ ] Memory poisoning defense (validate/sanitize input)
- [ ] Testing: precision, recall, F1 for generation; Recall@K + latency for retrieval
