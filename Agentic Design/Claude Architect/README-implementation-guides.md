# Implementation Guides: Building Agentic Solutions with Claude

A companion series to the [7-domain exam study set](./README.md), shifting from *what to know* to *how to build*. Each guide is runnable-code-level detail on one construct, cross-referenced back to the domain that governs its design decisions.

## Contents

| # | Guide | Covers |
|---|---|---|
| 0 | [`impl-00-agentic-loop-and-core-concepts.md`](./impl-00-agentic-loop-and-core-concepts.md) | The tools-in-a-loop primitive every other construct is built from; a router for which construct fits which job |
| 1 | [`impl-01-claude-agent-sdk-guide.md`](./impl-01-claude-agent-sdk-guide.md) | Install, custom tools, subagents, hooks, permission modes, structured output |
| 2 | [`impl-02-claude-code-implementation-guide.md`](./impl-02-claude-code-implementation-guide.md) | Team config, `CLAUDE.md`, settings hierarchy, hooks, MCP wiring, plan mode, CI/headless |
| 3 | [`impl-03-workflows-and-orchestration-patterns.md`](./impl-03-workflows-and-orchestration-patterns.md) | All five Anthropic workflow patterns and orchestrator-workers, as runnable code |
| 4 | [`impl-04-mcp-connectors-plugins.md`](./impl-04-mcp-connectors-plugins.md) | Building MCP servers (stdio & HTTP), plugins, and when to use a hosted connector instead |
| 5 | [`impl-05-skills-implementation-guide.md`](./impl-05-skills-implementation-guide.md) | `SKILL.md` structure, progressive disclosure, Skill vs. subagent vs. plugin vs. slash command |
| 6 | [`impl-06-end-to-end-reference-build.md`](./impl-06-end-to-end-reference-build.md) | Full repository scaffold tying every construct together, with a build-order sequencing guide |

## How the two series relate

| Domain guide (theory / exam) | Implementation guide (practice / code) |
|---|---|
| Domain 1 — Solution Design & Architecture | Guides 0, 3 |
| Domain 2 — Models, Prompting & Context | Guides 1, 2, 5 |
| Domain 3 — Integration | Guides 1, 4 |
| Domain 4 — Evaluation, Testing & Optimization | Guide 6, §3 |
| Domain 5 — Governance, Safety & Risk | Guides 1 (hooks), 6, §5 |
| Domain 6 — Stakeholder Communication & Lifecycle | Guide 6, §5–6 |
| Domain 7 — Developer Productivity & Operational Enablement | Guides 2, 6 |

## Suggested reading order

- **New to the constructs?** Read 0 → 1 → 2 → 3 → 4 → 5 → 6 in order.
- **Already building something specific?** Jump straight to the matching guide via the router in Guide 0, §2.
- **Standing up a new project from scratch?** Start at Guide 6 — it's the scaffold; follow its links backward into the specific guide for each piece as you build it.

## A note on code samples

Examples are illustrative and trimmed for clarity — error handling, retries, and full type annotations are abbreviated. Treat them as a correct skeleton to extend, not copy-paste-ready production code. Package names and API shapes referenced (`claude-agent-sdk`, `@anthropic-ai/claude-agent-sdk`, `@modelcontextprotocol/sdk`) were verified current at time of writing; check `docs.claude.com` for the latest signatures before shipping, since agentic tooling in this space moves quickly.
