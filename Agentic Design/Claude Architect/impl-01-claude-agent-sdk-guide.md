# Implementation Guide 1: Claude Agent SDK

> The Agent SDK is the same engine that powers Claude Code, packaged as a library you embed in your own application and run in your own infrastructure — not Anthropic-hosted. Use it when you're building a custom product (a support agent, an ops assistant, a coding tool) rather than using Claude Code interactively.<sup>[1]</sup>

---

## 1. Install and mental model

```bash
# Python
pip install claude-agent-sdk

# TypeScript / Node
npm install @anthropic-ai/claude-agent-sdk
```

The SDK gives you, out of the box, everything the raw API forces you to hand-roll:
- The tool-execution loop (Implementation Guide 0, §1)
- Built-in tools (file read/write/edit, bash, web search, glob/grep)
- Context management (compaction, tool-result clearing — Domain 2, §4.2)
- A permission system (allow/deny/ask/defer — Domain 3, §1.2)
- Hooks on lifecycle events
- Subagent spawning
- MCP client support (in-process, stdio, and remote servers)

**The TypeScript SDK bundles a native Claude Code binary** — it drives the same underlying agent harness Claude Code uses at the terminal, just embedded in your process instead of run interactively.

## 2. Minimal working agent

```python
import asyncio
from claude_agent_sdk import ClaudeAgentOptions, query

async def main():
    options = ClaudeAgentOptions(
        model="claude-sonnet-4-6",
        system_prompt="You are a network-triage assistant for a telecom carrier.",
        allowed_tools=["Read", "Bash", "WebSearch"],
        permission_mode="acceptEdits",   # or "default" / "plan" / "bypassPermissions"
        max_turns=10,
    )
    async for message in query(prompt="Diagnose ticket #48213", options=options):
        print(message)

asyncio.run(main())
```

```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

for await (const message of query({
  prompt: "Diagnose ticket #48213",
  options: {
    model: "claude-sonnet-4-6",
    systemPrompt: "You are a network-triage assistant for a telecom carrier.",
    allowedTools: ["Read", "Bash", "WebSearch"],
    permissionMode: "acceptEdits",
    maxTurns: 10,
  },
})) {
  console.log(message);
}
```

**`max_turns` is your step-cap guardrail against compounding errors (Domain 5, §2.3)** — always set one explicitly in production; the default is generous and an unbounded agentic loop on a live system is a cost and safety risk, not just an efficiency concern.

## 3. Custom tools

Custom tools are how you connect the loop to your private systems (Domain 3, §1.1's least-privilege principle applies here directly — only expose what the task needs).

```python
from claude_agent_sdk import tool, create_sdk_mcp_server

@tool("check_signal_strength", "Query current WiFi signal strength for a customer's modem",
      {"customer_id": str})
async def check_signal_strength(args):
    reading = await network_api.get_signal(args["customer_id"])
    return {"content": [{"type": "text", "text": f"Signal: {reading.dbm} dBm, channel {reading.channel}"}]}

server = create_sdk_mcp_server(
    name="network-tools",
    version="1.0.0",
    tools=[check_signal_strength],
)

options = ClaudeAgentOptions(
    mcp_servers={"network": server},
    allowed_tools=["mcp__network__check_signal_strength"],
)
```

**Naming convention:** in-process SDK MCP tools are addressed as `mcp__<server_name>__<tool_name>` in `allowed_tools` — this is also where the namespacing best practice from Domain 3, §1.1 (consolidate + namespace) becomes concrete: one server per logical service, one tool per action, not one tool per API endpoint.

## 4. Subagents

Subagents implement Domain 1, §3's orchestrator-worker pattern in code. Each subagent gets its own fresh context and a scoped tool list — nothing is inherited automatically (this is the SDK enforcing the "context isolation is engineered, never free" principle).

```python
options = ClaudeAgentOptions(
    agents={
        "diagnostic-specialist": {
            "description": "Investigates WiFi and connectivity issues in depth",
            "prompt": "You diagnose connectivity issues using signal, firmware, and driver data.",
            "tools": ["mcp__network__check_signal_strength", "mcp__network__query_firmware"],
            "model": "claude-sonnet-4-6",
        },
        "billing-specialist": {
            "description": "Investigates billing and account discrepancies",
            "prompt": "You reconcile billing records against charges.",
            "tools": ["mcp__billing__lookup_charges"],
            "model": "claude-haiku-4-5",   # cheaper tier for bounded lookup work
        },
    },
)
```

The lead agent decides, at runtime, which named subagent to dispatch to for a given subtask — this is the "planning" job from Domain 1, §3.1. Only the subagent's **final** message returns to the lead's context; the exploratory back-and-forth stays contained (Domain 2, §4.3).

**Factory pattern for dynamic subagents:** when the set of specialists isn't known until runtime (e.g., one subagent per detected device category), generate the `agents` dict programmatically before constructing `ClaudeAgentOptions`, rather than trying to spawn agents mid-loop from inside a tool call.

## 5. Hooks: the deterministic enforcement layer

Hooks are the SDK's implementation of Domain 5's "deterministic vs. probabilistic" guardrail split (§1.1) and Domain 7's "only deny rules, hooks, and managed policy are truly enforced" principle (§1.3). A hook fires on a lifecycle event and can **block** an action outright, regardless of what the model decided.

```python
async def pre_tool_use_hook(input_data, tool_use_id, context):
    if input_data["tool_name"] == "Bash" and "rm -rf" in input_data["tool_input"].get("command", ""):
        return {"permissionDecision": "deny", "reason": "Destructive command blocked by policy."}
    return {"permissionDecision": "allow"}

options = ClaudeAgentOptions(
    hooks={"PreToolUse": [pre_tool_use_hook]},
)
```

**The four `permissionDecision` values — `allow`, `deny`, `ask`, `defer`** — map directly to Domain 5, §3's four HITL gate patterns:

| Decision | Domain 5 pattern equivalent |
|---|---|
| `allow` | No gate — flows through |
| `deny` | Hard block — a deterministic guardrail, not HITL at all |
| `ask` | Approval-before-action gate |
| `defer` | Escalation-threshold gate (routes to a reviewer/queue) |

Common hook events beyond `PreToolUse`: `PostToolUse` (inspect/log a completed call — Domain 7, §2.4), `Notification`, `Stop` (fires when the agent finishes a turn — useful for final output validation before it reaches a user).

## 6. Permission modes

| Mode | Behavior |
|---|---|
| `default` | Prompts for anything not explicitly allowed |
| `acceptEdits` | Auto-approves file edits, still gates other risky actions |
| `plan` | Read-only — research and propose, cannot execute (Domain 7, §2.2) |
| `bypassPermissions` | No gating — reserve for fully sandboxed, low-stakes environments only |

**Never ship `bypassPermissions` against a production system with write access.** It exists for sandboxed CI or throwaway environments where the blast radius of a mistake is already contained by infrastructure, not by the permission system itself.

## 7. Structured output and error handling

```python
options = ClaudeAgentOptions(
    output_schema={  # forces a final structured response matching this schema
        "type": "object",
        "properties": {
            "category": {"type": "string"},
            "confidence": {"type": "number"},
            "queue": {"type": "string"},
        },
        "required": ["category", "confidence", "queue"],
    },
)
```

Always wrap `query()` calls with explicit exception handling for the SDK's typed exceptions (rate limits, tool execution failures, max-turns-exceeded) — this is the code-level implementation of Domain 4, §2.5's "diagnose the layer first" discipline: an SDK exception is an *integration* failure (Domain 4's first layer), while a structurally valid but wrong response is a *model-output* problem (the second layer), and they need different handling paths.

## 8. Batch processing for async workloads

For latency-insensitive, high-volume work (Domain 3, §1.3 and Domain 4, §3.2's batch-economics), don't run agents one at a time in a loop — use `asyncio.gather()` (Python) or `Promise.all` (TypeScript) to fan out concurrent, independent `query()` calls, or use the underlying Messages Batches API directly for a flat 50% discount when the individual calls don't need the full agentic loop.

---

## Worked Example: Telecom Diagnostic Agent, End to End

```python
import asyncio
from claude_agent_sdk import ClaudeAgentOptions, query, tool, create_sdk_mcp_server

@tool("check_signal_strength", "Get WiFi signal strength and channel", {"customer_id": str})
async def check_signal_strength(args):
    data = await network_api.get_signal(args["customer_id"])
    return {"content": [{"type": "text", "text": f"{data.dbm} dBm, channel {data.channel}"}]}

@tool("query_firmware_version", "Get modem firmware version and release date", {"modem_id": str})
async def query_firmware_version(args):
    fw = await network_api.get_firmware(args["modem_id"])
    return {"content": [{"type": "text", "text": f"v{fw.version}, released {fw.release_date}"}]}

@tool("escalate_to_engineer", "Escalate an unresolved case with full context",
      {"reason": str, "confidence": float, "summary": str})
async def escalate_to_engineer(args):
    await ticketing_api.escalate(**args)
    return {"content": [{"type": "text", "text": "Escalated."}]}

network_server = create_sdk_mcp_server(
    name="network", version="1.0.0",
    tools=[check_signal_strength, query_firmware_version, escalate_to_engineer],
)

async def pre_tool_hook(input_data, tool_use_id, context):
    if input_data["tool_name"] == "mcp__network__escalate_to_engineer":
        conf = input_data["tool_input"].get("confidence", 1.0)
        if conf < 0.6:
            return {"permissionDecision": "ask", "reason": "Low-confidence escalation needs review."}
    return {"permissionDecision": "allow"}

options = ClaudeAgentOptions(
    model="claude-sonnet-4-6",
    system_prompt=open("prompts/triage_system_prompt.txt").read(),
    mcp_servers={"network": network_server},
    allowed_tools=[
        "mcp__network__check_signal_strength",
        "mcp__network__query_firmware_version",
        "mcp__network__escalate_to_engineer",
    ],
    hooks={"PreToolUse": [pre_tool_hook]},
    max_turns=8,
    permission_mode="default",
)

async def diagnose(ticket_text: str):
    async for message in query(prompt=ticket_text, options=options):
        yield message

asyncio.run(diagnose("WiFi drops in my home office, only on my laptop..."))
```

This is the Domain 1 "agentic diagnostic branch" from the running telecom example, now as buildable code: the `PreToolUse` hook implements the Domain 5 escalation-threshold gate, `max_turns=8` bounds compounding errors, and the namespaced, consolidated tool set follows Domain 3, §1.1's right-sizing moves.

---

## Key Takeaways

- The Agent SDK gives you the loop, context management, permissions, and hooks pre-built — don't hand-roll a tool-execution loop against the raw Messages API for anything beyond a single fixed-sequence workflow.
- Always set `max_turns` explicitly; it's your primary defense against compounding-error runaway.
- Subagents get fresh context and a scoped tool list by construction — nothing is inherited, matching Domain 1's context-isolation principle.
- Hooks are the deterministic enforcement layer; the four permission decisions map directly onto Domain 5's HITL gate patterns.
- Never ship `bypassPermissions` against anything with real write access outside a sandboxed environment.

---

## References

1. Anthropic Docs, *"Agent SDK overview"* — https://docs.claude.com/en/api/agent-sdk/overview
2. Anthropic Docs, *"Agent SDK — Python reference"* — https://docs.claude.com/en/api/agent-sdk/python
3. Anthropic Docs, *"Agent SDK — TypeScript reference"* — https://docs.claude.com/en/api/agent-sdk/typescript
4. Anthropic Docs, *"Agent SDK — Subagents"* — https://docs.claude.com/en/api/agent-sdk/subagents
5. Anthropic Docs, *"Agent SDK — Hooks"* — https://docs.claude.com/en/api/agent-sdk/hooks
6. GitHub, *anthropics/claude-agent-sdk-python* — https://github.com/anthropics/claude-agent-sdk-python
