# Implementation Guide 0: The Agentic Loop & Core Constructs
**Companion to the 7-domain study set — this series covers *how to build*, not just what to know for the exam.**

> Every construct in this guide series — Agent SDK, Claude Code, Skills, subagents, MCP, connectors, plugins — is an implementation of the same underlying loop. Understand the loop once, and every other construct is a variation on where control lives and what's plugged into it.

---

## 1. The Agentic Loop (the atomic unit of everything)

```
┌─────────────────────────────────────────────────────────────────┐
│  1. Gather context   (system prompt, tools, conversation, files)  │
│  2. Send to model     (Claude decides: respond, or call a tool)   │
│  3. Execute action    (run the tool call; read-only concurrent,   │
│                         state-changing sequential)                 │
│  4. Verify result     (validate output, check stop_reason)        │
│  5. Append to context (tool_result block)                         │
│  6. Repeat from 2 until model returns no more tool calls          │
└─────────────────────────────────────────────────────────────────┘
```

This loop — "tools in a loop," in Anthropic's own phrasing<sup>[1]</sup> — is what every layer above it is built from:

| Layer | What it adds on top of the raw loop |
|---|---|
| **Raw API + tool use** | Nothing — you implement the loop yourself |
| **Claude Agent SDK** | The loop, context management, permissions, hooks — pre-built |
| **Claude Code** | The SDK, plus a terminal/IDE harness, file editing, CLAUDE.md, slash commands |
| **A workflow** (Domain 1) | Your code decides the *sequence* of loop invocations |
| **An agent** (Domain 1) | The model decides the next loop invocation itself |
| **A subagent** | A nested loop, spawned with its own fresh context and a scoped toolset |
| **A Skill** | Instructions + resources loaded *into* one loop's context, on demand |
| **MCP** | A standard interface for what tools the loop can call, and where they live |
| **A plugin** | A packaged bundle of tools + skills + commands, installed as a unit |

If you can point at where in this table a requirement sits, you already know which construct to reach for.

### 1.1 Who owns the next step is the only real axis

Restating Domain 1's core distinction in implementation terms:

```python
# WORKFLOW — your code owns the sequence
symptoms = extract_symptoms(call_transcript)          # step 1: your code calls this
domain = identify_domain(symptoms)                     # step 2: your code calls this
diagnosis = diagnose_root_cause(domain, symptoms)       # step 3: your code calls this
route_to_queue(diagnosis)                               # step 4: your code calls this

# AGENT — the model owns the sequence
result = client.messages.create(
    model="claude-sonnet-4-6",
    tools=[lookup_account, record_symptom, request_diagnostic,
           query_firmware, diagnose_root_cause, escalate, resolve_ticket],
    messages=[{"role": "user", "content": call_transcript}],
)
# the model decides which tool to call next, in what order, and when to stop
```

Nothing about the *API call itself* distinguishes a workflow from an agent — the SDK looks identical either way. The distinction is entirely about **whether your application code or the model's own reasoning selects the next tool call.**

---

## 2. Which construct, for which job

Use this as a first-pass router before opening the next guide in this series:

| You need to... | Reach for |
|---|---|
| One-off tool-augmented call with your code driving sequence | Raw API + tool use, or a workflow pattern (Guide 3) |
| A long-running, file-editing, self-directed coding or ops agent | **Claude Agent SDK** (Guide 1) or **Claude Code** (Guide 2) |
| A terminal/IDE-based developer tool your team runs interactively | **Claude Code** (Guide 2) |
| A custom application embedding agentic behavior (support bot, ops tool) | **Claude Agent SDK** (Guide 1) |
| Parallel, isolated exploration of independent subtasks | **Subagents** (Guide 3, §3) |
| A reusable, autonomously-triggered procedure (house style, checklist, domain workflow) | **A Skill** (Guide 5) |
| Reaching a private system, database, or SaaS product | **A custom tool**, or **MCP** if it needs to be reused across hosts/clients (Guide 4) |
| Distributing a bundle of tools + skills + commands as one unit across a team/org | **A plugin** (Guide 4, §4) |
| Wiring into an existing product like Slack, Gmail, or Drive with no code | **A connector** (Guide 4, §5) |

---

## 3. The five constructs, defined precisely (so the rest of the series doesn't have to re-derive this)

- **Workflow** — LLM calls orchestrated through code paths *you* wrote in advance.
- **Agent** — the model dynamically directs its own tool use and process, maintaining control over how it accomplishes a task.<sup>[1]</sup>
- **Subagent** — an agent spawned *by* another agent (the orchestrator), running in its own isolated context window, returning only a distilled result.
- **Skill** — a folder of instructions (`SKILL.md`) plus optional bundled resources that Claude loads autonomously when a task matches the Skill's description.
- **MCP server** — a process (local or remote) exposing tools, resources, and prompts through the Model Context Protocol's standard interface, so any MCP-compatible client can use it without custom integration code.

**Plugins and connectors are packaging concepts, not new execution primitives** — a plugin bundles Skills/tools/commands for distribution; a connector is a pre-built, hosted MCP integration to a specific external product. Both ultimately resolve down to tools and Skills feeding the same loop from §1.

---

## Key Takeaways

- Everything in this series is the same six-step loop with something different plugged into steps 2–3.
- The workflow/agent distinction is about who picks the next tool call, not what the API call looks like.
- Route by job, not by hype: reach for the SDK/Code before an agent, an agent before a multi-agent system, a tool before an MCP server, and a Skill before a plugin.

---

## References

1. Anthropic Engineering, *"Building Effective Agents"* — https://www.anthropic.com/engineering/building-effective-agents
2. Anthropic Docs, *"Agent SDK overview"* — https://docs.claude.com/en/api/agent-sdk/overview
3. Anthropic Docs, *"Claude Code overview"* — https://docs.claude.com/en/docs/claude-code/overview
