---
name: agent-architect
description: Expert knowledge for designing and building AI agent systems. Use when the user wants to architect an agent, design multi-agent systems, choose orchestration patterns, or plan agent components.
triggers:
  - build an agent
  - design agent
  - agent architecture
  - multi-agent system
  - orchestration pattern
  - agent loop
---

# Agent Architecture Expert

You are now loaded with deep knowledge about agent architecture. Apply these patterns when helping the user design or build agent systems.

## Core Definition
Agent = Model + Tools + Orchestration + Deployment, using an LLM in a loop. An agent is "a system dedicated to context window curation."

## The Agent Loop
1. **Get Mission** — receive goal from user or trigger
2. **Scan Scene** — gather context (request, memory, tools)
3. **Think** — reason and plan (chain of reasoning, not single thought)
4. **Act** — execute via tools
5. **Observe & Iterate** — assess outcome, update context, loop back

## Orchestration Design Choices
- **Degree of autonomy**: spectrum from deterministic workflow (LLM as a step) to fully LLM-driven dynamic execution
- Use prompting frameworks: Chain-of-Thought, ReAct
- **System prompt = agent's constitution**: persona, constraints, output schema, rules, tool guidance, few-shot examples
- Build for observability from day one

## Model Selection Strategy
- Don't pick by benchmark; test against YOUR business problem
- **Model routing**: frontier model for complex reasoning/planning, cheaper models for classification/summarization
- Models become obsolete fast; build CI/CD pipeline to evaluate new models continuously

## Taxonomy of Agent Complexity (5 Levels)
| Level | Description | Example |
|---|---|---|
| 0 | Core reasoning only (no tools/memory) | Simple chatbot |
| 1 | Connected problem-solver (LLM + tools) | RAG assistant |
| 2 | Strategic problem-solver (multi-step planning + context engineering per step) | Code agent |
| 3 | Collaborative multi-agent (team of specialists) | Research pipeline |
| 4 | Self-evolving (creates new tools/agents dynamically) | Autonomous systems |

## Multi-Agent Design Patterns

### Coordinator Pattern
Manager agent segments tasks → routes to specialist agents → aggregates responses.
Best for: heterogeneous tasks requiring different expertise.

### Sequential Pattern
Assembly line: output of one agent feeds into the next.
Best for: pipeline processing (extract → transform → validate → output).

### Iterative Refinement
Generator agent + critic agent in a feedback loop.
Best for: quality-sensitive outputs (writing, code, plans).

### Human-in-the-Loop
Deliberate pause for human approval on high-stakes actions.
Implementation: SMS, UI, database task queue.

## Multi-Agent Session Strategies
1. **Shared unified history**: all agents read/write same log → tightly coupled, collaborative tasks
2. **Separate histories**: each agent maintains own perspective → loosely coupled specialists

Key: use **framework-agnostic memory layer** (strings, dicts) as universal shared knowledge between heterogeneous agents.

## Sub-Agent Architecture
- Sub-agents are separate instances spawned by main agent
- **Context inheritance varies by type**:
  - General-purpose/plan: inherit full context
  - Search/explore: start fresh (search tasks are independent)
- Background execution for long-running or monitoring tasks
- After sub-agent returns summary → main agent should read relevant files itself (attention cross-referencing extracts more pairwise relationships)

## Agent Evolution & Learning
- Agents degrade without adaptation ("aging")
- Learning sources: runtime experience (logs, traces, HITL feedback) + external signals
- Two optimization paths:
  1. **Enhanced context engineering**: refine prompts, few-shot examples, retrieved memories
  2. **Tool optimization/creation**: identify capability gaps, create/modify tools
- Consider "Agent Gym": offline simulation for trial-and-error with synthetic data and red-teaming

## New Patterns (2026)

### Agent Teams (Peer-to-Peer)
Multiple agent instances coordinate as a team — team lead assigns tasks, teammates communicate directly (not through central coordinator). Removes hub-and-spoke bottleneck. Best for: research, debugging with competing hypotheses, cross-layer coordination.

### Agent Harness Pattern (from Anthropic engineering)
For long-running agents across multiple sessions:
1. **Initializer agent**: sets up environment on first run
2. **Coding agent**: makes incremental progress per session
3. Progress file (e.g., `progress.txt`) + version control as cross-session state
4. Different prompts for first vs subsequent context windows

### Durable Execution
Agent preserves progress across API failures and restarts. Critical for production reliability. (Pydantic AI, Strands)

### Blackboard Architecture
Shared semantic workspace — agents read/write to blackboard rather than passing messages through manager. Alternative to coordinator pattern for loosely coupled collaboration.

## Framework Landscape (Feb 2026)
| Framework | Key Strength |
|---|---|
| Google ADK | TypeScript + Python, A2A native, conversation rewind |
| LangGraph 1.0 | Stable API, Agent Builder, Insights Agent |
| Strands (AWS) | A2A + MCP, native OTel, serverless deployment |
| CrewAI | Cross-framework agent integration |
| Pydantic AI | Durable execution, type-safe |

## Design Checklist
When helping design an agent, address these:
- [ ] What level of autonomy? (L0-L4)
- [ ] Single agent or multi-agent? Which pattern? (coordinator/sequential/iterative/peer-to-peer/blackboard)
- [ ] Model selection and routing strategy?
- [ ] What tools does it need? (retrieval + action)
- [ ] Session strategy (shared vs separate)?
- [ ] Memory architecture (short-term + long-term + episodic)?
- [ ] Security model (guardrails, HITL, identity)?
- [ ] Observability (logs, traces, metrics — use OTel GenAI conventions)?
- [ ] How will it be evaluated?
- [ ] How will it evolve/learn?
- [ ] Durable execution for failure recovery?
- [ ] Cross-session state (progress files, git history)?
