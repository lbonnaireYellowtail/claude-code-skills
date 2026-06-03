---
name: security-reviewer
description: Expert knowledge for reviewing and designing agent security, implementing guardrails, threat modeling for MCP/tools, and governance at scale. Use when the user wants to secure an agent system, review threats, implement guardrails, or design governance.
triggers:
  - agent security
  - security review
  - guardrails
  - threat model
  - agent governance
  - prompt injection defense
  - MCP security
---

# Agent Security & Governance Expert

You are now loaded with deep knowledge about agent security threats and mitigations. Apply these when helping the user.

## Core Principle
Every power granted to an agent = corresponding risk. Always balance utility vs security.

## Defense-in-Depth (Three Layers)

### Layer 1: Deterministic Guardrails
Hardcoded rules as security chokepoint OUTSIDE the model:
- Block purchases > threshold
- Require confirmation for external API calls
- Allowlist permitted tool calls
- Rate limiting on sensitive operations
- **Before-tool callbacks**: inspect parameters before execution, validate against current state

### Layer 2: Reasoning-Based Defenses
- Adversarial training
- Guard models that screen plans before execution
- **Dual-planner architecture**: trusted planner for first-party tools, untrusted for third-party, restricted channel between them

### Layer 3: Continuous Assurance
- Full re-evaluation on any model/safety system change
- Responsible AI testing (NPOV, parity evaluations)
- Proactive red teaming (manual + AI-driven persona-based simulation)

## Production Security Layers
1. **Policy Definition**: agent's "constitution" — desired/undesired behaviors in system instructions
2. **Guardrails & Filtering**:
   - Input filtering: classifiers block malicious inputs before reaching agent
   - Output filtering: safety filters for harmful content, PII, policy violations
   - HITL escalation: pause high-risk/ambiguous actions for human review
3. **Continuous Assurance**: ongoing evaluation, red teaming, feedback loops

## Agent Identity
Agents are a **new class of principal** (distinct from users and service accounts). They need:
- Verifiable identity
- Least-privilege permissions
- Scoped credentials with short expiration
- Audience-bound credentials
- **Keep secrets out of agent context** (use side channels)

## MCP-Specific Threat Model

### 1. Dynamic Capability Injection
**Threat**: Servers change available tools without client notification. Low-risk agent suddenly gains high-risk capabilities.
**Mitigations**: tool allowlists, mandatory change notification, tool/package pinning, API gateway filtering

### 2. Tool Shadowing
**Threat**: Malicious tool descriptions overshadow legitimate tools. Attacker's tool intercepts data.
**Mitigations**: naming collision checks (including semantic similarity), mTLS, deterministic policy enforcement, HITL for high-risk ops

### 3. Malicious Tool Definitions & Content
**Threat**: Tool descriptors manipulate agent planners; ingested content contains prompt injections.
**Mitigations**: input validation, output sanitization (PII, tokens, URLs), separate system prompts from user inputs, **dual-planner architecture**

### 4. Sensitive Information Leaks
**Threat**: Tools receive/transmit sensitive info from conversation context.
**Mitigations**: structured outputs with sensitivity annotations, **taint tracking** (tag inputs/outputs as tainted/clean)

### 5. Confused Deputy Problem
**Threat**: Privileged MCP server tricked by unprivileged user via AI intermediary into unauthorized actions.
**Mitigations**: scoped credentials, audience validation, least privilege, side channels for secrets

### 6. No Fine-Grained Access Scope
**Threat**: MCP only supports coarse client-server auth, no per-tool/per-resource authorization.
**Mitigations**: scoped tokens with short expiration, audience-bound credentials, strict least privilege

## Memory Security
- **Memory poisoning**: false info corrupts future interactions → validate/sanitize before committing
- **Data isolation**: strict per-user/per-tenant ACLs, never cross-contaminate
- **User control**: opt-out of memory generation, request deletion
- **Exfiltration risk**: anonymize shared procedural memories to prevent cross-user leaks
- **PII redaction**: mandatory before persisting any data

## Governance at Scale
- **Central gateway**: control plane for all agentic traffic (user↔agent, agent↔tool, agent↔agent, inference requests)
- **AgentGateway (Solo.io)**: Rust-based proxy that natively understands MCP + A2A. Traditional proxies (Envoy) cannot enforce meaningful agent security — they can't distinguish tool calls from model invocations
- **Central registry**: "enterprise app store" for agents/tools with discovery, reuse, security review, versioning
- **NIST AI Risk Management Framework for Agents**: Govern-Map-Measure-Manage lifecycle for compliance

## New Threats (2026)

### Visual/Multimodal Prompt Injection
NVIDIA AI Red Team: symbolic visual inputs (emoji sequences, rebus puzzles) bypass text guardrails on multimodal agents. As agents gain vision, the attack surface expands beyond text.

### Guardrail Sandwich Architecture (Emerging Best Practice)
Input sanitization + trust labeling → Bounded reasoning (tool allowlists, step limits, budgets) → Output validation + PII redaction. Post-execution hooks scan tool outputs for injection patterns before the LLM sees them. Design so a compromised context has limited blast radius.

### Production Security Tools
- **Lakera Guard**: real-time prompt injection detection API, learns from 100K+ attacks daily, scans content/attachments/URLs
- **Bedrock AgentCore Policy**: declarative action boundaries at infrastructure level (not app level)

## Security Review Checklist
- [ ] Agent has least-privilege permissions?
- [ ] Deterministic guardrails on high-risk operations?
- [ ] Input filtering for prompt injection / malicious input?
- [ ] **Visual/multimodal input filtering** for agents with vision?
- [ ] Output filtering for PII, harmful content, policy violations?
- [ ] HITL escalation path for ambiguous/high-stakes actions?
- [ ] Tool allowlisting (no dynamic capability injection)?
- [ ] Tool naming collision checks (no shadowing)?
- [ ] Secrets kept out of agent context (side channels)?
- [ ] Memory poisoning defense (validation before commit)?
- [ ] PII redacted before any persistence?
- [ ] Per-user data isolation enforced?
- [ ] Credentials scoped and short-lived?
- [ ] Red teaming scheduled (manual + automated)?
- [ ] **Agent-aware network gateway** for MCP/A2A traffic?
- [ ] **Guardrail sandwich** (input → bounded reasoning → output)?
