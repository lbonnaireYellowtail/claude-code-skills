# Shared Claude Code Skills

[![version](https://img.shields.io/badge/version-1.1.0-blue)](https://github.com/lbonnaireYellowtail/claude-code-skills/releases/latest) [![changelog](https://img.shields.io/badge/changelog-md-lightgrey)](CHANGELOG.md)

A collection of reusable Claude Code skills for frontend development, AI/agent building, documentation, and workflow automation. These are project-agnostic and can be dropped into any Claude Code setup.

## Installation

Copy any skill folder into your `~/.claude/skills/` directory:

```bash
# Clone the repo
git clone <repo-url> shared-skills

# Copy all skills at once
cp -r shared-skills/*/  ~/.claude/skills/

# Or copy a single skill
cp -r shared-skills/angular-signals ~/.claude/skills/
```

Then invoke with `/skill-name` in any Claude Code session.

---

## Skills

### Frontend Development
| Skill | Description |
|---|---|
| `/a11y-reviewer` | WCAG 2.2 compliance, ARIA patterns, keyboard navigation, screen reader support |
| `/angular-architect` | Component architecture, services, state, routing patterns |
| `/angular-signals` | Reactive primitives, computed, effects — Angular Signals patterns |
| `/bff-designer` | Backend for Frontend pattern — API shaping and aggregation layer design |
| `/design-tokens` | CSS custom properties, 3-layer token hierarchy, theming, dark mode |
| `/nx-workspace` | NX monorepo management — generators, executors, task pipelines |
| `/perf-auditor` | Change detection, lazy loading, bundle size optimization |
| `/test-writer` | Angular unit and integration testing — Jest, Testing Library patterns |

### AI / Agent Building
| Skill | Description |
|---|---|
| `/agent-architect` | Design and build AI agent systems and multi-agent orchestration |
| `/context-optimizer` | Token budgeting, context window management, compression strategies |
| `/debug-agent` | AI agent failure modes, tool errors, reasoning trace debugging |
| `/eval-designer` | Agent evaluation systems and eval dataset design |
| `/memory-designer` | Context engineering, session memory, persistent knowledge patterns |
| `/prompt-engineer` | System prompt design, few-shot examples, prompt optimization |
| `/quality-reviewer` | Agent quality evaluation and evaluation harness design |
| `/rag-designer` | RAG pipelines — chunking, embeddings, retrieval, reranking |
| `/security-reviewer` | Agent security, guardrails, threat modelling, governance at scale |
| `/tool-designer` | Agent tools, MCP servers, A2A integration protocol design |
| `/workflow-designer` | Multi-step agent workflows — DAG design, orchestration patterns |
| `/learn` | Structured lesson + quiz sessions for learning any technical topic |

### Workflow & Git
| Skill | Description |
|---|---|
| `/bitbucket-pr` | Manage Bitbucket PRs — view, create, and update from the CLI |
| `/implement` | Implement a Jira story end-to-end following its refinement document |
| `/refine` | Refine a user story, clarify scope, generate a refinement document |
| `/review` | Chunk-by-chunk interactive code and doc review |
| `/view-story` | Fetch and display a Jira story with acceptance criteria and comments |

### Documentation
| Skill | Description |
|---|---|
| `/doc-adr` | Author a new Architecture Decision Record with auto-numbering |
| `/doc-update` | Scan recent commits and update docs that have drifted from code |

---

## Scripts

Standalone helper scripts (not skills) that plug into Claude Code via `settings.json`. Each has its own README with install steps.

| Script | Description |
|---|---|
| [`scripts/statusline`](scripts/statusline/) | Usage-gauge statusline — context fill, 5h/7d rate limits, model — with cross-session rate-limit sync across all open terminals |

---

## Contributing

Skills are markdown files with a YAML frontmatter block. See any existing `SKILL.md` for the format.
