# Implementation Guide 4: MCP Servers, Connectors & Plugins

> MCP standardizes how a model reaches tools; connectors are pre-built MCP integrations to specific products; plugins package tools, Skills, and commands for distribution across a team or org. This guide builds each, and covers when each one is (and isn't) the right mechanism, per Domain 3, §3.1's selection framework.

---

## 1. When to build vs. when to reach for what already exists

```
Need to reach a private system?
 ├── No  → use a built-in tool (web search, code execution)
 └── Yes → Is it reused across multiple clients/hosts?
            ├── No  → write a custom tool (Guide 1, §3) — done, no MCP needed
            └── Yes → build an MCP server (this guide, §2)
                       Does the product already have a hosted connector?
                        └── Yes → use the connector instead of building (§5)
```

Building an MCP server has real overhead (a process to run, secure, and version). Don't reach for it when a custom tool in the Agent SDK, scoped to one consumer, would do the job (Domain 3, §3.1).

---

## 2. Building an MCP server

### 2.1 stdio transport (local, single client, no OAuth)

```typescript
// mcp-servers/network/src/index.ts
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { z } from "zod";

const server = new McpServer({ name: "network-tools", version: "1.0.0" });

server.registerTool(
  "check_signal_strength",
  {
    title: "Check Signal Strength",
    description: "Get current WiFi signal strength and channel for a customer's modem.",
    inputSchema: { customer_id: z.string() },
  },
  async ({ customer_id }) => {
    const reading = await networkApi.getSignal(customer_id);
    return {
      content: [{ type: "text", text: `${reading.dbm} dBm, channel ${reading.channel}` }],
    };
  }
);

const transport = new StdioServerTransport();
await server.connect(transport);
```

**Credentials for a stdio server come from the local environment** (`process.env.NETWORK_API_KEY`) — there is no OAuth flow for this transport, and claiming there is is the classic misread from Domain 3, §1.2.

### 2.2 Streamable HTTP transport (remote, multi-client, OAuth 2.1 + PKCE)

```typescript
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StreamableHTTPServerTransport } from "@modelcontextprotocol/sdk/server/streamableHttp.js";
import express from "express";

const app = express();
const server = new McpServer({ name: "network-tools-remote", version: "1.0.0" });
// ... registerTool calls as above ...

app.post("/mcp", async (req, res) => {
  // Validate the OAuth 2.1 bearer token BEFORE dispatching to the transport.
  const token = req.headers.authorization?.replace("Bearer ", "");
  const claims = await validateToken(token, { expectedAudience: "network-tools-remote" });
  if (!claims) return res.status(401).send("Unauthorized");

  const transport = new StreamableHTTPServerTransport({ sessionIdGenerator: undefined });
  await server.connect(transport);
  await transport.handleRequest(req, res, req.body);
});

app.listen(3000);
```

**Token hygiene checklist (Domain 3, §1.2), all three enforced here or upstream of it:**
1. `resource`/audience validated against this specific server (`expectedAudience`) — never trust a token just because it's valid *somewhere*
2. Token rides in the `Authorization` header — never accept it from a query string
3. **Never forward the caller's token further upstream** to another internal system — that's the confused-deputy pattern; if this server needs to call another service, it authenticates with its *own* service credential, not the client's token

### 2.3 Least privilege, applied to tool design

```typescript
// BAD — one tool, unscoped, mirrors the whole internal API
server.registerTool("network_api", { inputSchema: { endpoint: z.string(), method: z.string(), body: z.any() } }, ...);

// GOOD — narrow, named, least-privilege tools
server.registerTool("check_signal_strength", { inputSchema: { customer_id: z.string() } }, ...);
server.registerTool("query_firmware_version", { inputSchema: { modem_id: z.string() } }, ...);
// no generic "call any endpoint" tool exists at all
```

A generic passthrough tool defeats least privilege at the design level — no amount of downstream permissioning fixes a tool whose blast radius is "anything the underlying API can do."

### 2.4 Right-sizing the catalog (Domain 3, §1.1, as server design)

- **Consolidate**: `ticket_ops(action: "create"|"update"|"close", ...)` instead of three separate tools, when the actions share most of their schema
- **Namespace**: server name (`network-tools`) becomes the tool's namespace automatically once a client connects — pick server names deliberately, they show up in every tool call
- **High-signal returns only**: return `"{dbm} dBm, channel {channel}"`, not the full raw API payload with forty fields the model will never use
- **Defer rarely-used tools**: split a large catalog across multiple MCP servers so a client only connects to (and pays context tokens for) the ones it actually needs for a given task

---

## 3. Connecting an MCP server to Claude

**Agent SDK (in-process, no separate binary needed for simple cases — Guide 1, §3):**
```python
options = ClaudeAgentOptions(mcp_servers={"network": network_server})
```

**Claude Code / external process (`.mcp.json`, Guide 2, §5):**
```json
{ "mcpServers": { "network-tools": { "command": "node", "args": ["./mcp-servers/network/dist/index.js"] } } }
```

**Remote HTTP server:**
```json
{ "mcpServers": { "network-tools-remote": { "type": "http", "url": "https://mcp.internal.example.com/network" } } }
```

**Anthropic API directly (server-side, in a Messages call):**
```python
response = client.messages.create(
    model="claude-sonnet-4-6",
    messages=[...],
    mcp_servers=[{"type": "url", "url": "https://mcp.internal.example.com/network", "name": "network-tools"}],
)
```

---

## 4. Plugins: packaging Skills, tools, and commands as one unit

A plugin bundles what would otherwise be several separately-installed pieces — MCP server config, Skills, and slash commands — into one distributable artifact a team can install with a single action.

```
network-triage-plugin/
├── plugin.json
├── .mcp.json               # bundled MCP server definitions
├── skills/
│   └── triage-checklist/
│       └── SKILL.md          # see Implementation Guide 5
└── commands/
    └── triage.md
```

```json
// plugin.json
{
  "name": "network-triage",
  "version": "1.2.0",
  "description": "Network fault-triage tools, Skills, and commands for telecom support engineering.",
  "author": "platform-team",
  "components": {
    "mcpServers": [".mcp.json"],
    "skills": ["skills/triage-checklist"],
    "commands": ["commands/triage.md"]
  }
}
```

**Review a plugin before installing it, the same way you'd review a shared Skill (Domain 2, §4.6) — a plugin's Skills invoke autonomously and its MCP servers may request real credentials.** Installing a plugin is a trust decision, not just a convenience click; this matters more, not less, for org-wide plugin catalogs where the barrier to install is deliberately low.

---

## 5. Connectors: pre-built, hosted MCP integrations

A connector is a maintained, hosted MCP server for a specific product (Slack, Gmail, Google Drive, and similar) that you enable rather than build. Use one whenever it exists for the product you need, instead of writing a custom MCP server against that product's raw API — you inherit the maintainer's handling of auth refresh, rate limits, and API version changes for free.

```python
response = client.messages.create(
    model="claude-sonnet-4-6",
    messages=[{"role": "user", "content": "Summarize unread messages in #network-ops"}],
    mcp_servers=[{"type": "url", "url": "https://mcp.slack.com/mcp", "name": "slack"}],
)
```

**Connectors still need least-privilege scoping at the workspace/account level** — the OAuth grant you approve when enabling a connector determines its blast radius exactly as if you'd written the permission scope yourself (Domain 3, §1.2). A connector that requests broader scope than the task needs is an over-privileged token by another name.

---

## Worked Example: Full Integration Layer for the Telecom Triage System

```
┌───────────────────────────────────────────────────────────┐
│  network-triage-plugin (installed org-wide via catalog)      │
│   ├── .mcp.json → network-tools (stdio, internal API)         │
│   ├── skills/triage-checklist (Domain 2 style rubric)          │
│   └── commands/triage.md                                       │
├───────────────────────────────────────────────────────────┤
│  Custom tool (Agent SDK, single-consumer, no MCP overhead)     │
│   └── ticket_ops(action) — only this service calls it           │
├───────────────────────────────────────────────────────────┤
│  Slack connector (hosted, org-approved scope: #network-ops     │
│   channel read + post only)                                    │
│   └── posts escalation summaries; no write access elsewhere     │
└───────────────────────────────────────────────────────────┘
```

Notice the mix: a plugin for the reusable, team-distributed piece; a raw custom tool for the single-consumer internal call that doesn't earn MCP's overhead; and a hosted connector instead of a hand-built Slack integration — each construct chosen by the decision tree in §1, not by which one sounded most sophisticated.

---

## Key Takeaways

- MCP earns its overhead only with genuine cross-host reuse; a single-consumer integration is a custom tool, full stop.
- stdio = local, single client, credentials from environment, no OAuth. Streamable HTTP = remote, multi-client, OAuth 2.1 + PKCE. Don't mix up the auth models.
- Validate token audience server-side and never forward a client's token upstream — the confused-deputy defense is implemented in your server code, not assumed from the transport.
- Design tools narrow and named, never as a generic API passthrough — that defeats least privilege regardless of downstream permissioning.
- Plugins package Skills + tools + commands for team-wide distribution; review them like any shared Skill before installing, since they run with real access.
- Prefer a hosted connector over a hand-built integration whenever one exists for the product you need — but still scope its OAuth grant to least privilege.

---

## References

1. Model Context Protocol, *official specification* — https://modelcontextprotocol.io/
2. Model Context Protocol, *"Authorization"* — https://modelcontextprotocol.io/specification/2025-06-18/basic/authorization
3. Anthropic Docs, *"Model Context Protocol (MCP)"* — https://docs.claude.com/en/docs/agents-and-tools/mcp
4. Anthropic Docs, *"Remote MCP servers"* — https://docs.claude.com/en/docs/agents-and-tools/remote-mcp-servers
5. GitHub, *modelcontextprotocol/typescript-sdk* — https://github.com/modelcontextprotocol/typescript-sdk
6. Anthropic Docs, *"Claude Plugins"* — https://docs.claude.com/en/docs/claude-code/plugins
