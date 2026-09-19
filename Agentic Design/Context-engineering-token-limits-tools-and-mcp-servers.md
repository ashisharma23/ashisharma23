# Context Engineering: Navigating Token Limits, Tools, and MCP Servers

## Summary

Context engineering is the discipline of deciding which tokens occupy an LLM's working memory at each inference step. The core argument: context is a finite, degrading resource, not a bucket. The most reliable gains in agentic systems now come from curating what enters the window (tool definitions, tool results, history), not from making the window larger. The Model Context Protocol (MCP, the open standard for connecting agents to tools and data) standardizes *how* context is supplied, but by itself it does not manage *how much*. That remains the client's and architect's job.

A note on naming: there is no product called "MCP 2.0". The protocol uses dated releases. The most recent, `2026-07-28`, is the largest architectural revision to date and is what this article treats as the "2.0-generation" spec.

## 1. Context as an attention budget

Anthropic frames the context window as an "attention budget" that the model draws on when parsing context, and reports that as token count grows, recall of information in that context decreases ([Anthropic, Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)). The post attributes this to the transformer's n² pairwise token relationships and to training data dominated by shorter sequences.

Chroma's "Context Rot" study tested 18 models (including Claude, GPT-4.1, Gemini 2.5, and Qwen3 variants) across five controlled experiments that varied only input length. It found performance "grows increasingly unreliable as input length grows," even on deliberately simple tasks, with faster decay when the question and answer are semantically dissimilar and when topically related distractors are present. The authors note their tasks are intentionally simple, say real-world degradation may be worse, and do not explain the mechanism ([Chroma Research](https://www.trychroma.com/research/context-rot)).

Two consequences follow. A 1M-token window is a capacity limit, not a quality guarantee; effective capacity is task-dependent and lower than advertised. And the right optimization target is the smallest set of high-signal tokens that makes the desired outcome likely.

**Cheatsheet: attention budget**

- Treat the window as a budget with a fixed cost (system prompt, tool schemas) and a variable cost (history, tool results).
- Never assume the advertised window size is your usable size; test recall on your own tasks at your real context lengths.
- Put the highest-signal instructions and facts where the model reliably attends to them; early positions were recognized more reliably in Chroma's tests.
- Remove near-duplicate and topically similar filler; distractors accelerate degradation.
- Measure context size per step in production, not just per request.

## 2. Where the tokens actually go

Token consumption in agentic systems is dominated by things the user never typed. Published figures:

| Consumer | Measured figure | Source |
|---|---|---|
| Agent vs. chat | ~4x tokens | [Anthropic multi-agent research system](https://www.anthropic.com/engineering/multi-agent-research-system) |
| Multi-agent vs. chat | ~15x tokens | same |
| Tool definitions, 5 MCP servers | 58 tools, ~55K tokens before the conversation starts | [Anthropic, Advanced tool use](https://www.anthropic.com/engineering/advanced-tool-use) |
| Tool definitions, worst case seen internally | 134K tokens | same |
| Intermediate result passthrough | e.g., a 2-hour meeting transcript copied between tools adds ~50K tokens | [Anthropic, Code execution with MCP](https://www.anthropic.com/engineering/code-execution-with-mcp) |

Assuming a 200K window (the arithmetic in the tool-search figures implies this), 55K of tool definitions is roughly 27% of the budget spent before any work begins, and 134K is roughly two-thirds. These ratios are derived here, not published metrics. Fixed overhead (tool schemas, system prompt) and variable overhead (accumulated tool results) compete with the actual task for the same attention budget.

Anthropic also reports that token usage alone explained 80% of performance variance on the BrowseComp evaluation, and that a Claude Opus 4 lead with Sonnet 4 sub-agents outperformed single-agent Opus 4 by 90.2% on an internal research eval ([source](https://www.anthropic.com/engineering/multi-agent-research-system)). This is the tension architects must manage: more tokens often buy better results, yet each additional token in a single window degrades attention. Multi-agent designs partly resolve this by buying tokens in separate windows. The same post cautions that tasks needing shared context or tight interdependencies are a poor fit.

**Cheatsheet: token accounting**

- Log tokens by category: system prompt, tool definitions, history, tool results, model output.
- Alert when tool definitions exceed a set share of the window (for example 20%).
- Track cost per completed task, not cost per call; agents and multi-agent systems multiply spend (about 4x and 15x versus chat).
- Watch for intermediate data flowing through the model that never needed to (large files, transcripts, query results).
- Budget for the multiplier before choosing multi-agent; it needs high-value tasks to pay off.

## 3. Strategies against context rot

### Compaction

Summarize a conversation nearing the limit and start a new window seeded with the summary. Claude Code preserves architectural decisions and unresolved bugs while discarding redundant tool output ([Anthropic](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)). The risk is lossy summarization: a dropped detail may prove critical later, and errors in a summary are inherited by every subsequent step.

**Cheatsheet: compaction**

- Trigger before the limit, not at it; leave headroom for the next step.
- Tune the summarizer for recall first, then trim for precision.
- Always carry forward: decisions made, open issues, constraints, and current goal.
- Drop: raw tool output, superseded plans, repeated attempts.
- Test with evals that cross a compaction boundary, and re-run them when the model or summary prompt changes.

### Pruning (tool-result clearing and context editing)

Removing raw tool outputs from deep history once consumed is described as one of the safest, lightest-touch forms of compaction. Anthropic's context editing feature does this automatically; on a 100-turn web-search evaluation it reported an 84% token reduction while enabling completion of tasks that otherwise failed. Combined with the memory tool, it reported a 39% improvement over baseline (29% for context editing alone) ([Anthropic, Managing context](https://www.anthropic.com/news/context-management)). These are vendor-run benchmarks on a web-search-heavy task set, where stale tool output is unusually disposable, so expect smaller gains elsewhere.

**Cheatsheet: pruning**

- Clear tool results once the agent has acted on them; keep the call record, drop the payload.
- Keep the most recent N results intact so the agent retains working detail.
- Cap tool output size at the tool boundary; return a reference or a summary instead of a blob.
- Exclude anything the agent may need to re-cite (for example evidence for a decision) or store it externally first.
- Verify task success with and without pruning before enabling it broadly.

### Structured external memory

Agents write notes (for example a NOTES.md or a memory directory) outside the window and read them back later. Anthropic's memory tool exposes create, read, update and delete over a client-side directory, so the developer controls storage ([source](https://www.anthropic.com/news/context-management)). Persisted memory is also persisted error: a wrong note keeps steering later sessions.

**Cheatsheet: external memory**

- Store decisions, progress, and stable facts; not raw transcripts.
- Keep notes short, dated, and structured so they are cheap to reload.
- Give memory an owner, an expiry or review policy, and an audit trail.
- Keep the storage backend under your control (access control, data residency).
- Load memory selectively at session start rather than dumping all of it.

### Just-in-time retrieval

Keep lightweight identifiers (file paths, stored queries, URLs) in context and load content at runtime via tools, rather than pre-loading everything ([Anthropic](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)). The trade-off is added latency and extra tool calls, and an agent can misjudge what to fetch.

**Cheatsheet: just-in-time retrieval**

- Hold references, not content; fetch on demand.
- Make identifiers self-describing (meaningful names and paths) so the agent can choose well.
- Fetch the narrowest slice that answers the question (a section, a range, a filtered query).
- Combine with a small pre-loaded core for content the agent will always need.
- Budget for the extra round trips in your latency targets.

### Sub-agent isolation

Sub-agents work in clean windows and return condensed summaries, typically 1,000 to 2,000 tokens, to the coordinator ([Anthropic](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)). The cost is the roughly 15x token multiplier noted above, plus coordination failures such as duplicated work and conflicting conclusions.

**Cheatsheet: sub-agents**

- Use for parallelizable, breadth-heavy work; avoid for tightly coupled steps that share state.
- Give each sub-agent a narrow task, clear output format, and explicit stopping criteria.
- Cap the returned summary size (for example 1,000 to 2,000 tokens).
- Have the coordinator verify or reconcile results rather than trusting them blindly.
- Track token multiplier and duplicated effort as first-class metrics.

### Tool-definition and tool-result offloading

Anthropic's Tool Search Tool with `defer_loading` keeps most tool schemas out of the window until needed. On its figures, context preserved rose from 122,800 to 191,300 tokens (an 85% reduction in tool-definition tokens), and MCP-eval accuracy rose from 49% to 74% (Opus 4) and 79.5% to 88.1% (Opus 4.5). Programmatic Tool Calling cut average tokens on complex research tasks from 43,588 to 27,297 (37%) ([source](https://www.anthropic.com/engineering/advanced-tool-use)).

Presenting MCP servers as code APIs that an agent calls from a sandbox cut a Google Drive to Salesforce workflow from 150,000 to 2,000 tokens (98.7%) and kept raw data from passing through the model. The post names the cost: a secure sandbox with resource limits and monitoring ([source](https://www.anthropic.com/engineering/code-execution-with-mcp)). These are best-case, vendor-reported figures on specific workloads; expect smaller gains when few tools are connected.

**Cheatsheet: tool offloading**

- Expose a small core toolset; defer the rest behind search.
- Write tool names and descriptions precisely; ambiguous descriptions cause silent misuse.
- Process bulk data in code (filter, aggregate, join) and return only the result to the model.
- Run generated code in a sandbox with resource limits, network policy, and monitoring.
- Measure tool-selection accuracy before and after deferring tools.

## 4. How MCP standardizes context management

MCP defines a common wire contract for tools, resources, and prompts, so any compliant client can consume any compliant server. Its context-relevant value is standardization of supply: schemas, structured results, discovery, and auth. It does not decide what belongs in the window. The 55K-token example above is what happens when a client naively loads every server's tool list.

### The `2026-07-28` specification (the "2.0-generation" release)

Per the project's release-candidate post ([MCP blog](https://blog.modelcontextprotocol.io/posts/2026-07-28-release-candidate/)):

- **Stateless core.** The `initialize` handshake and `Mcp-Session-Id` header are removed, so any request can land on any server instance without sticky routing or shared session stores at the protocol layer.
- **Cache metadata.** List responses carry `ttlMs` and `cacheScope`, letting clients avoid re-fetching tool lists and re-paying for them.
- **Routing and observability.** Mandatory `Mcp-Method` and `Mcp-Name` headers allow routing without body inspection; W3C Trace Context (`traceparent`, `tracestate`, `baggage`) is standardized for distributed tracing.
- **Extensions.** Capabilities are negotiated through an `extensions` map using reverse-DNS IDs. Two are official: MCP Apps and Tasks.
- **Tasks.** Long-running work returns a task handle; clients use `tasks/get`, `tasks/update`, and `tasks/cancel`. For context management, this lets an agent hold a reference instead of streaming or polling large intermediate output into the window.
- **Schemas.** Input and output schemas support full JSON Schema 2020-12. Typed output schemas make it feasible to validate and trim results before they reach the model.
- **Auth.** Six SEPs (spec change proposals) harden OAuth/OIDC alignment, including RFC 9207 `iss` validation.
- **Deprecations.** Roots, Sampling, and Logging enter a formal deprecation lifecycle with 12-month removal windows.

The post lists the release candidate as locked May 21, 2026 and the final specification as July 28, 2026. Confirm details against the final spec text before implementing.

### What the spec does not solve

The 2026 roadmap prioritizes transport scalability, agent communication (Tasks), governance, and enterprise readiness, and does not specifically address tool-definition bloat; server metadata discoverable via `.well-known` without a live connection is the closest item ([MCP roadmap](https://blog.modelcontextprotocol.io/posts/2026-mcp-roadmap/)). Token-budget control therefore lives in clients and gateways: deferred tool loading, result truncation, and code-execution wrappers are implementation choices, not protocol guarantees.

**Cheatsheet: implementing MCP**

- Target a specific dated spec version and pin it in your client and server configuration.
- Honor `ttlMs` and `cacheScope` when caching tool lists.
- Design servers to be stateless so they scale horizontally behind a load balancer.
- Define output schemas and keep results compact; return handles or references for large data.
- Use Tasks for long-running operations instead of blocking or polling large payloads.
- Propagate W3C trace headers end to end so failures can be traced across servers.
- Apply least-privilege scopes with OAuth/OIDC; validate `iss` per RFC 9207.
- Plan migrations off Roots, Sampling, and Logging within the 12-month windows.
- Put a gateway or client-side policy layer in front of servers to enforce token budgets.

## 5. Future work and open problems

The roadmap points to refined Tasks semantics (retry behavior on transient failure, result-expiry policies); standard server metadata for pre-connection discovery; enterprise features (audit trails, SSO-integrated auth, gateway behavior, configuration portability) intended to land as extensions rather than core changes; and a Working Group model for faster SEP acceptance.

Open problems that remain:

1. **Discovery at scale.** Search-based tool discovery works in vendor evals, but no cross-vendor standard defines relevance ranking or tool-description quality.
2. **Compaction quality.** No shared benchmark measures what a summary loses; evaluation is mostly per-team.
3. **Context rot mechanics.** Chroma says it cannot explain why degradation occurs, so mitigation remains empirical.
4. **Independent replication.** Most headline token figures come from the vendor selling the feature.
5. **Security.** Code-execution offloading and richer extensions widen the sandbox and authorization surface.

One plausible direction is clients converging on lazy tool loading plus cached list metadata via `ttlMs`, though the sources reviewed do not commit to that.

## 6. Best practices for enterprise AI workflows

1. **Budget context explicitly.** Set a per-workflow ceiling (fixed overhead plus working room) and alert on breaches. Owner: the platform team running the agent runtime.
2. **Load tools lazily.** Expose a small core set and defer the rest behind search.
3. **Treat tool results as disposable.** Clear consumed results, cap size at the tool boundary, return references instead of blobs.
4. **Choose isolation by task shape.** Sub-agents for parallel, breadth-first work; a single agent with compaction for tightly coupled work.
5. **Make compaction testable.** Build evals that cross compaction boundaries and re-run them on every model or prompt change.
6. **Govern memory.** Version notes, expire them, and log which entries influenced a decision.
7. **Pin and test against a spec version.** Plan for deprecation windows and use trace propagation.
8. **Sandbox before you offload.** Code-execution patterns save tokens but need isolation, limits, and monitoring.
9. **Assign failure ownership.** When an agent fails because of bad context, the owner is whoever controls the context assembly layer. Trace-level observability is how they find out.

## Bottom line

A larger window raises the ceiling on what an agent can hold; it does not raise the quality of what it attends to. Curate the window as deliberately as you would a production database's working set, and treat MCP as the plumbing, not the policy.

## Sources

- [Effective context engineering for AI agents, Anthropic](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
- [Context Rot: How Increasing Input Tokens Impacts LLM Performance, Chroma Research](https://www.trychroma.com/research/context-rot)
- [How we built our multi-agent research system, Anthropic](https://www.anthropic.com/engineering/multi-agent-research-system)
- [Introducing advanced tool use, Anthropic](https://www.anthropic.com/engineering/advanced-tool-use)
- [Code execution with MCP, Anthropic](https://www.anthropic.com/engineering/code-execution-with-mcp)
- [Managing context on the Claude Developer Platform, Anthropic](https://www.anthropic.com/news/context-management)
- [The 2026-07-28 MCP Specification Release Candidate, MCP Blog](https://blog.modelcontextprotocol.io/posts/2026-07-28-release-candidate/)
- [The 2026 MCP Roadmap, MCP Blog](https://blog.modelcontextprotocol.io/posts/2026-mcp-roadmap/)
