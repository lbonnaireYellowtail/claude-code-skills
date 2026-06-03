---
name: tool-designer
description: Expert knowledge for designing agent tools, MCP servers, integration protocols (A2A), and on-demand context patterns (skills, hooks). Use when the user wants to design tools for agents, build MCP servers, or implement integration patterns.
triggers:
  - design tools
  - build tools for agent
  - MCP server
  - tool integration
  - A2A protocol
  - agent skills
  - agent hooks
---

# Tool Design & Integration Expert

You are now loaded with deep knowledge about designing tools and integrations for AI agents. Apply these when helping the user.

## Tool Design Best Practices

### Documentation (consumed by model as context)
- **Clear names**: `create_critical_bug_in_jira_with_priority` > `update_jira` (also helps audit logs)
- **Describe all parameters**: include type AND the use the tool will make of each parameter
- **Short parameter lists**: long lists confuse models
- **Plain language descriptions**: avoid jargon
- **Targeted examples**: address ambiguities; dynamically retrieve task-relevant examples to minimize context bloat
- **Document default values**: LLMs use defaults well when documented

### Instruction Design
- Describe **actions**, not tools: say "create a bug" not "use the create_bug tool"
- Don't duplicate tool docs in system instructions (causes confusion, creates dependency)
- Don't dictate workflows; describe objectives, let model act autonomously
- DO explain tool interactions and side-effects between tools

### Granularity Rules
- **Publish tasks, not API calls**: tools = user-facing tasks, not thin API wrappers
- **One tool = one function**: single responsibility; easier to document, more consistent
- **Avoid multi-tools**: don't bundle workflows into one tool (exception: commonly repeated sequences for efficiency)

### Output Design
- **Concise outputs**: large responses swamp context and persist in conversation history forever
- **External storage**: store large data externally, return reference (table name, file path, artifact ID)
- **Schema validation**: for inputs/outputs — serves as documentation AND runtime check

### Error Messages (often overlooked)
Error messages are returned to the LLM — use them as a guidance channel:
```
"No product found for ID XXX. Ask the customer to confirm the product name, then look up by name."
```
Include: what went wrong + what the agent should do next.

## Model Context Protocol (MCP)

### Architecture
Solves the "N models x M tools" integration explosion. Client-server model inspired by LSP.
- **Host**: orchestrates (e.g., Claude Code, IDE)
- **Client**: manages connection to one server
- **Server**: provides capabilities (tools, resources, prompts)

### Communication
- Base: JSON-RPC 2.0
- Transports: **stdio** (local, subprocess) / **Streamable HTTP** (remote, supports SSE)

### Primitives (by adoption)
| Primitive | Side | Adoption | Purpose |
|---|---|---|---|
| Tools | Server | 99% | Functions the agent can call |
| Resources | Server | 34% | Data the agent can read |
| Prompts | Server | 32% | Prompt templates |
| Sampling | Client | 10% | Server requests LLM completion |
| Elicitation | Client | 4% | Server requests user input |

### Tool Definition Schema
`name`, `title` (optional), `description`, `inputSchema`, `outputSchema` (optional), `annotations` (optional)
Annotations are hints only: readOnlyHint, destructiveHint, idempotentHint, openWorldHint — don't trust from untrusted servers.

### Key Challenges
- **Context bloat**: ALL tool definitions from ALL connected servers load into context
- **Degraded reasoning**: too many tools = model picks wrong tool or loses user intent
- Future solution: **RAG-like tool discovery** (index all tools, retrieve relevant subset per task)
- Stateful protocol + stateless REST = complex state management
- Auth/AuthZ still evolving; identity ambiguity; no native observability

### Context Optimization: Code Execution Pattern
Instead of loading many tool definitions:
1. Expose code APIs (not tool call definitions)
2. Give agent a sandbox execution environment
3. Let it write code to make tool calls
4. "Prompt on demand" — reduces context footprint

## Agent-to-Agent (A2A) Protocol
- Standard for multi-agent interoperability
- **Agent Card**: JSON spec advertising capabilities, auth requirements, endpoint
- **Task-based communication**: lifecycle (submitted → working → input-required → completed/failed)
- Supports long-running tasks with streaming updates

### A2A vs MCP
| | A2A | MCP |
|---|---|---|
| Purpose | Agent-to-agent collaboration | Agent-to-tool connection |
| Unit of work | Task (multi-step, long-running) | Tool call (single operation) |
| Communication | Structured messages + streaming | Function call/response |
| Discovery | Agent Cards | Tool definitions |

## Skills Pattern (On-Demand Context Loading)
- Folder with SKILL.md + reference files + scripts
- Metadata in system prompt (lightweight); full content loaded only when relevant
- Like Neo: "I know Kung Fu" — download expertise on demand
- Keep SKILL.md under 500 lines; split into separate files if needed
- Combine with hooks for automated skill reminders

## Hooks (Agent Lifecycle Events)
- Observe lifecycle stages (Stop, UserPromptSubmit, etc.)
- Run bash scripts before/after events
- Combine hooks + skills + reminders for sophisticated context management
- Use cases: notifications, automated actions, context injection, guardrails

## Tool Design Checklist
- [ ] Name clearly describes the task (not the API)?
- [ ] All parameters documented with type + purpose?
- [ ] Parameter list kept short?
- [ ] Description in plain language, no jargon?
- [ ] Examples cover edge cases and ambiguities?
- [ ] Output is concise (large data stored externally)?
- [ ] Error messages guide the agent's next action?
- [ ] Schema validation on inputs and outputs?
- [ ] Side-effects documented?
- [ ] Single responsibility (one tool = one function)?
