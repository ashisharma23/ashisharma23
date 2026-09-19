# Context Engineering: Navigating Token Limits, Tools, and MCP Servers

## Summary

Context engineering is the discipline of deciding which tokens occupy an LLM's working memory at each inference step. The core argument: context is a finite, degrading resource, not a bucket. The most reliable gains in agentic systems come from curating what enters the window (system prompt, tool definitions, memory, tool results, history), not from making the window larger. The Model Context Protocol (MCP, the open standard for connecting agents to tools and data) standardizes *how* context is supplied, but by itself it does not manage *how much*. That remains the harness's job.

To make this concrete, this article uses one running example: **a 1,000,000-token context window** (the default size on current Claude Opus 5, Sonnet 5 and Fable/Mythos models, per [Anthropic's context window docs](https://platform.claude.com/docs/en/build-with-claude/context-windows)). Each section explains what the concept means at that scale and gives numbers a harness can be configured with.

Two caveats up front. There is no industry-standard split of a context window; published defaults are absolute token counts (100K, 150K, 25K) that were calibrated for smaller windows and did not scale up when windows grew to 1M. The percentages below are therefore starting points derived from those defaults, and should be tuned against your own evals. Each table marks whether a number is a **documented default**, a **documented limit**, or a **suggested starting point**.

A note on naming: there is no product called "MCP 2.0". The protocol uses dated releases. The most recent, `2026-07-28`, is the largest architectural revision to date and is what this article treats as the "2.0-generation" spec.

## 1. The 1M window as an attention budget

Anthropic frames the context window as an "attention budget" that the model draws on when parsing context, and reports that recall of information decreases as token count grows ([Anthropic, Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)). The post attributes this to the transformer's n² pairwise token relationships and to training data dominated by shorter sequences.

Chroma's "Context Rot" study tested 18 models across five controlled experiments that varied only input length. It found performance "grows increasingly unreliable as input length grows," even on deliberately simple tasks, with faster decay when the question and answer are semantically dissimilar and when topically related distractors are present. The authors note their tasks are intentionally simple and that they cannot explain the mechanism ([Chroma Research](https://www.trychroma.com/research/context-rot)).

**What this means at 1M.** A 1M window is a capacity limit, not a quality guarantee. Billing is not the constraint either: 1M-token models use standard long-context pricing with no special tier ([Anthropic](https://platform.claude.com/docs/en/build-with-claude/context-windows)). Attention quality is. No vendor publishes a "safe fill level", so the zones below are suggested starting points.

| Zone | Tokens | % of 1M | Basis |
|---|---|---|---|
| Output + thinking reserve | 64K typical, up to 128K | 6.4% to 12.8% | 128K is the documented max output on 1M models |
| Safety buffer | 15K | 1.5% | Claude Code reserves about 13K below the limit, per a [GitHub issue](https://github.com/anthropics/claude-code/issues/31806) |
| Target working set | 300K to 350K | 30% to 35% | Suggested starting point (see section 2) |
| Alarm: force compaction or handoff | 500K | 50% | Suggested starting point |
| Hard stop for planned work | 700K | 70% | Suggested starting point; leaves room for 128K output plus buffer |

**Cheatsheet: attention budget**

- Set a target working set (about 30% of the window), an alarm (about 50%), and a hard stop (about 70%); never plan to fill 100%.
- Reserve output and thinking tokens before allocating input; thinking tokens count against the window.
- Test recall on your own tasks at 100K, 300K, 500K and 700K before trusting any zone.
- Prefer lowering the fill level over adding more context when accuracy drops.
- Log window fill per step, not just per request.

## 2. Budgeting the 1M window by component

Published token figures show where budget goes in agentic systems: agents use about 4x the tokens of chat and multi-agent systems about 15x ([Anthropic](https://www.anthropic.com/engineering/multi-agent-research-system)); a five-server MCP setup with 58 tools costs about 55K tokens in definitions before any work starts, and Anthropic saw 134K in one case ([Anthropic](https://www.anthropic.com/engineering/advanced-tool-use)); copying a 2-hour meeting transcript between tools adds about 50K tokens ([Anthropic](https://www.anthropic.com/engineering/code-execution-with-mcp)).

At 1M, 55K of tool definitions is 5.5% of the window (it would be 27.5% of a 200K window). That looks affordable, but it is 55K tokens of low-relevance text competing for attention on every request, and Anthropic reports MCP-eval accuracy rose from 49% to 74% (Opus 4) once definitions were deferred ([source](https://www.anthropic.com/engineering/advanced-tool-use)).

**Reference budget for a 1M window** (planned working set of about 315K; the rest is headroom):

| Component | Cap (tokens) | % of 1M | Basis |
|---|---|---|---|
| System prompt and policies | 5K (range 3K to 8K) | 0.5% | Suggested; Claude Code advises under 200 lines per CLAUDE.md file |
| Persistent memory loaded at start | 8K | 0.8% | Documented limit: first 200 lines or 25KB of MEMORY.md ([Claude Code docs](https://code.claude.com/docs/en/memory)); token conversion is approximate |
| Tool definitions, non-deferred | 15K (hard cap 20K) | 1.5% | Tool search recommended above 10K ([docs](https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-search-tool)) |
| Retrieved documents (RAG) | 50K | 5% | Suggested starting point |
| Live tool results | 75K (3 results x 25K) | 7.5% | Documented defaults: keep last 3 results; 25K max MCP output |
| Conversation history and summary | 100K | 10% | Suggested starting point |
| Retained thinking blocks | 50K | 5% | Suggested; Opus 4.5+ and Sonnet 4.6+ keep all thinking blocks by default |
| Sub-agent returns | 10K (5 x 2K) | 1% | Documented range of 1,000 to 2,000 tokens per sub-agent |
| **Planned working set** | **about 313K** | **about 31%** | |

**Cheatsheet: token accounting**

- Emit a per-request breakdown: system, tools, memory, retrieval, tool results, history, thinking, output.
- Alert when any component exceeds its cap; the usual offenders are tool results and tool definitions.
- Use the token counting API before sending large requests.
- Track cost per completed task, not per call; budget for the 4x (agent) and 15x (multi-agent) multipliers.
- Re-derive the table for your own workload after two weeks of production traces.

## 3. Strategies against context rot, with harness settings

### Pruning (tool-result clearing and context editing)

Removing raw tool outputs from deep history once consumed is described as one of the lightest-touch forms of compaction. Anthropic's context editing did this on a 100-turn web-search eval with an 84% token reduction, and with the memory tool showed a 39% improvement over baseline (29% for context editing alone) ([Anthropic](https://www.anthropic.com/news/context-management)). These are vendor-run results on a task set where stale tool output is unusually disposable.

**What it means at 1M.** The documented default trigger is 100,000 input tokens, which is only 10% of a 1M window. Anthropic did not raise it when windows grew, which suggests the default is about quality, not capacity.

| Setting | Value | Basis |
|---|---|---|
| `trigger` | 100,000 input tokens | Documented default ([context editing docs](https://platform.claude.com/docs/en/build-with-claude/context-editing)) |
| `keep` | 3 most recent tool uses | Documented default |
| `clear_at_least` | 5,000 tokens | Example value in docs; prevents clearing that would not be worth a cache break |
| `exclude_tools` | evidence and audit tools | Suggested |
| `clear_tool_inputs` | false | Documented default |
| Thinking blocks `keep` | last 2 to 4 turns if you need to cap them | Model-specific default is "all" on Opus 4.5+ and Sonnet 4.6+ |

**Cheatsheet: pruning**

- Start with the documented defaults (trigger 100K, keep 3) and change one value at a time against evals.
- Set `clear_at_least` so a clearing pass frees enough tokens to justify itself.
- Exempt tools whose output is evidence you must cite later, or persist that output externally first.
- Cap each tool result at the tool boundary rather than relying on clearing to fix it later.
- Compare task success with and without pruning before enabling it broadly.

### Compaction

Compaction summarizes a conversation nearing a threshold and continues from the summary. Claude Code preserves architectural decisions and unresolved bugs while dropping redundant output ([Anthropic](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)). Risk: lossy summaries can drop a detail that matters later, and summary errors are inherited by every later step.

**What it means at 1M.** The API's server-side compaction defaults to a 150,000-token trigger (minimum 50,000), which is 15% of a 1M window; the docs suggest lowering it (for example to 100,000) for long tasks. Claude Code's default auto-compact fires near the top of the window (window minus about 13K, roughly 83.5% on a 200K window, per the [GitHub issue](https://github.com/anthropics/claude-code/issues/31806)). Those two defaults sit at very different fill levels, so choose deliberately.

| Setting | Value | Basis |
|---|---|---|
| API trigger | 150,000 tokens | Documented default ([compaction docs](https://platform.claude.com/docs/en/build-with-claude/compaction)) |
| API trigger minimum | 50,000 tokens | Documented limit |
| Trigger for long, precise tasks | 100,000 tokens | Suggested in docs |
| Trigger if evals show no quality loss | 200,000 to 300,000 tokens | Suggested starting point |
| Alarm / forced handoff | 500,000 tokens | Suggested starting point |
| Summary size | 3K to 5K tokens | Suggested; the docs example shows about 3,500 output tokens |
| `pause_after_compaction` | true for budgeted runs | Documented option; lets you count compactions |
| Cost | each compaction is a separate billed sampling step | Documented; sum `usage.iterations` |

**Cheatsheet: compaction**

- Trigger well before the alarm level; do not wait for the window to fill.
- Write custom `instructions` that name what to preserve: decisions, open issues, constraints, current goal, identifiers.
- Count compactions and set a total token budget; the docs show a loop that stops at a cap.
- Include a test that crosses a compaction boundary in every eval suite; re-run on model or prompt changes.
- Bill from `usage.iterations`, not the top-level usage fields.

### Structured external memory

Agents write notes outside the window (a NOTES.md, a memory directory) and reload them later. Anthropic's memory tool exposes create, read, update and delete over a client-side directory ([source](https://www.anthropic.com/news/context-management)). Persisted memory is also persisted error: a wrong note keeps steering later sessions.

**What it means at 1M.** Memory competes for the budget only when loaded. Claude Code loads the first 200 lines or 25KB of MEMORY.md at session start and moves detail into topic files ([docs](https://code.claude.com/docs/en/memory)). Anthropic's research agent saved its plan to memory before the context reached 200,000 tokens ([source](https://www.anthropic.com/engineering/multi-agent-research-system)).

| Setting | Value | Basis |
|---|---|---|
| Memory index loaded at start | 200 lines or 25KB (about 8K tokens) | Documented limit |
| Instruction file size | under 200 lines each | Documented guidance |
| Topic files | loaded on demand, 2K to 5K tokens each | Suggested |
| Checkpoint plan and state to memory | at each compaction, and before 200K | Anthropic's research agent uses 200K |
| Entry review or expiry | 30 to 90 days | Suggested |

**Cheatsheet: external memory**

- Keep the always-loaded index small (under 8K tokens) and put detail in on-demand files.
- Store decisions, progress and stable facts; not raw transcripts.
- Date every entry and give the store an owner and an expiry review.
- Write a checkpoint before every compaction.
- Keep storage under your control for access control and data residency.

### Just-in-time retrieval

Keep lightweight identifiers (file paths, stored queries, URLs) in context and load content at runtime ([Anthropic](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)). The trade-off is latency and extra tool calls, and an agent can misjudge what to fetch.

| Setting | Value | Basis |
|---|---|---|
| Retrieved content per turn | 10K to 25K tokens | Suggested; aligns with the 25K MCP output default |
| Total retrieved content live | 50K (5% of 1M) | Suggested |
| Fetch granularity | section, line range or filtered query | Suggested |

**Cheatsheet: just-in-time retrieval**

- Hold references, not content, and fetch the narrowest slice that answers the question.
- Use self-describing names and paths so the agent chooses well.
- Keep a small pre-loaded core for content needed on every run.
- Budget for extra round trips in latency targets.

### Sub-agent isolation

Sub-agents work in clean windows and return condensed summaries of typically 1,000 to 2,000 tokens ([Anthropic](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)). The cost is the roughly 15x token multiplier, plus coordination failures such as duplicated work.

**What it means at 1M.** A sub-agent's window does not add to the lead's window. Five sub-agents each spending 150K tokens cost 750K in total but add only about 10K to the lead's context. Anthropic's research system embeds these scaling rules: one agent with 3 to 10 tool calls for simple fact-finding, 2 to 4 sub-agents with 10 to 15 calls each for comparisons, and more than 10 sub-agents for complex research ([source](https://www.anthropic.com/engineering/multi-agent-research-system)).

| Setting | Value | Basis |
|---|---|---|
| Parallel sub-agents | 3 to 5 | Documented in Anthropic's research system |
| Parallel tool calls per sub-agent | 3 or more | Documented |
| Sub-agent return size | 1,000 to 2,000 tokens | Documented |
| Sub-agent window cap | 100K to 150K tokens | Suggested starting point |
| Lead agent budget for returns | about 10K (5 x 2K) | Derived |
| Simple task | 1 agent, 3 to 10 tool calls | Documented scaling rule |
| Comparison task | 2 to 4 sub-agents, 10 to 15 calls each | Documented scaling rule |

**Cheatsheet: sub-agents**

- Use for parallelizable, breadth-heavy work; avoid for tightly coupled steps that share state.
- Give each sub-agent a narrow task, output format and stopping rule.
- Hard-cap the returned summary at 2,000 tokens.
- Have the lead reconcile results instead of trusting them.
- Track the token multiplier and duplicated effort as metrics.

### Tool-definition and tool-result offloading

Anthropic's Tool Search Tool with `defer_loading` keeps most schemas out of the window until needed, with an 85% reduction in tool-definition tokens; Programmatic Tool Calling cut average tokens on complex research tasks by 37% (43,588 to 27,297) ([source](https://www.anthropic.com/engineering/advanced-tool-use)). Presenting MCP servers as code APIs cut one workflow from 150,000 to 2,000 tokens (98.7%), at the cost of a secure sandbox ([source](https://www.anthropic.com/engineering/code-execution-with-mcp)). These are vendor-reported best cases.

| Setting | Value | Basis |
|---|---|---|
| Enable tool search when | 10 or more tools, or definitions above 10K tokens | Documented guidance ([tool search docs](https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-search-tool)) |
| Tools kept non-deferred | 3 to 5 most-used | Documented guidance |
| Search results per query | 5 (default) | Documented default |
| Max deferred tools | 10,000 | Documented limit |
| Non-deferred definition budget | 15K tokens (cap 20K) | Suggested |
| Per-tool result cap | 25,000 tokens | Claude Code default (`MAX_MCP_OUTPUT_TOKENS`) |
| Result warning | 10,000 tokens | Claude Code default |
| Oversized results | spill to a file, return the path | Claude Code behavior ([MCP docs](https://code.claude.com/docs/en/mcp)) |

**Cheatsheet: tool offloading**

- Defer everything except the 3 to 5 tools used on most requests.
- Namespace tools (`github_`, `slack_`) so one search matches the group.
- Cap results at 10K to 25K tokens; write larger outputs to a file and return the path.
- Process bulk data in a sandbox (filter, aggregate, join) and return only the answer.
- Measure tool-selection accuracy before and after deferring.

## 4. How MCP standardizes context management

MCP defines a common wire contract for tools, resources and prompts, so any compliant client can consume any compliant server. Its context-relevant value is standardization of supply: schemas, structured results, discovery and auth. It does not decide what belongs in the window. The 55K-token example above is what happens when a client loads every server's tool list.

### The `2026-07-28` specification (the "2.0-generation" release)

Per the project's release-candidate post ([MCP blog](https://blog.modelcontextprotocol.io/posts/2026-07-28-release-candidate/)):

- **Stateless core.** The `initialize` handshake and `Mcp-Session-Id` header are removed, so any request can land on any server instance without sticky routing or shared session stores.
- **Cache metadata.** List responses carry `ttlMs` and `cacheScope`, letting clients avoid re-fetching tool lists.
- **Routing and observability.** Mandatory `Mcp-Method` and `Mcp-Name` headers allow routing without body inspection; W3C Trace Context (`traceparent`, `tracestate`, `baggage`) is standardized.
- **Extensions.** Capabilities are negotiated through an `extensions` map using reverse-DNS IDs. Two are official: MCP Apps and Tasks.
- **Tasks.** Long-running work returns a handle; clients use `tasks/get`, `tasks/update` and `tasks/cancel`, so an agent can hold a reference instead of streaming large intermediate output into the window.
- **Schemas.** Input and output schemas support full JSON Schema 2020-12, which makes it feasible to validate and trim results before they reach the model.
- **Auth.** Six SEPs (spec change proposals) harden OAuth/OIDC alignment, including RFC 9207 `iss` validation.
- **Deprecations.** Roots, Sampling and Logging enter a formal deprecation lifecycle with 12-month removal windows.

The release candidate was locked May 21, 2026 and the final specification dated July 28, 2026. Confirm details against the final spec text before implementing.

The 2026 roadmap prioritizes transport scalability, agent communication (Tasks), governance and enterprise readiness, and does not specifically address tool-definition bloat ([MCP roadmap](https://blog.modelcontextprotocol.io/posts/2026-mcp-roadmap/)). Token-budget control therefore lives in clients and gateways.

**What it means at 1M.** The protocol gives you the hooks; the numbers below are yours to set.

| Setting | Value | Basis |
|---|---|---|
| Tool-list cache TTL (`ttlMs`) | 300,000 ms (5 minutes) for stable servers | Suggested; the spec defines the field, not a value |
| Per-tool result cap | 25,000 tokens | Claude Code default |
| Per-tool override ceiling | 500,000 characters (`anthropic/maxResultSizeChars`) | Documented limit; use sparingly |
| Definitions per server | under 5K tokens | Suggested; 58 tools across 5 servers averages about 11K per server |
| Servers connected per agent | as few as the task needs | Suggested; scope per workflow |
| Task handle polling | poll status, fetch result only when complete | Design pattern from Tasks lifecycle |

**Cheatsheet: implementing MCP**

- Target a specific dated spec version and pin it.
- Honor `ttlMs` and `cacheScope` when caching tool lists.
- Keep servers stateless so they scale horizontally.
- Define output schemas; return handles or references for large data.
- Use Tasks for long-running work instead of blocking or streaming payloads.
- Propagate W3C trace headers end to end.
- Use least-privilege OAuth/OIDC scopes and validate `iss`.
- Plan migrations off Roots, Sampling and Logging within 12 months.
- Enforce token budgets in a gateway or client policy layer.

## 5. Harness configuration: putting the numbers together

A directional starting configuration for a 1M-window agent harness. Values marked (default) are documented vendor defaults; the rest are suggested starting points to tune against evals.

```yaml
window:
  size: 1_000_000
  output_reserve: 64_000          # up to 128_000 for heavy thinking
  safety_buffer: 15_000
  working_target: 315_000         # ~31%
  alarm: 500_000                  # force compaction or handoff
  hard_stop: 700_000

components:
  system_prompt: 5_000
  memory_index: 8_000             # 200 lines / 25KB cap (default)
  tool_definitions_nondeferred: 15_000
  retrieved_context: 50_000
  history_and_summary: 100_000
  thinking_retained: 50_000
  subagent_returns: 10_000

tools:
  tool_search_when: ">=10 tools or >10K tokens"   # default guidance
  nondeferred_tools: 5                              # 3-5 (default guidance)
  search_results_per_query: 5                       # default
  result_cap_tokens: 25_000                         # default
  result_warn_tokens: 10_000                        # default
  mcp_list_ttl_ms: 300_000

context_editing:
  trigger_input_tokens: 100_000   # default
  keep_tool_uses: 3               # default
  clear_at_least: 5_000
  exclude_tools: [evidence_lookup, audit_log]

compaction:
  trigger_input_tokens: 150_000   # default; 100_000 for long precise tasks
  max_trigger_if_evals_pass: 300_000
  summary_target_tokens: 4_000
  pause_after_compaction: true
  max_compactions_per_task: 10    # suggested budget guard

subagents:
  parallel: 4                     # 3-5 (default guidance)
  tool_calls_parallel: 3
  return_tokens_max: 2_000        # default range 1-2K
  window_cap: 150_000
```

## 6. Future work and open problems

The MCP roadmap points to refined Tasks semantics (retry behavior on transient failure, result-expiry policies); standard server metadata for pre-connection discovery; enterprise features (audit trails, SSO-integrated auth, gateway behavior, configuration portability) intended to land as extensions; and a Working Group model for faster SEP acceptance.

Open problems that remain:

1. **No standard budget split.** Published defaults are absolute counts that predate 1M windows; there is no benchmark that tells you the best fill level for your task.
2. **Discovery at scale.** Search-based tool discovery works in vendor evals, but no cross-vendor standard defines relevance ranking or description quality.
3. **Compaction quality.** No shared benchmark measures what a summary loses.
4. **Context rot mechanics.** Chroma says it cannot explain why degradation occurs, so mitigation is empirical.
5. **Independent replication.** Most headline token figures come from the vendor selling the feature.
6. **Security.** Code-execution offloading and richer extensions widen the sandbox and authorization surface.

## 7. Best practices for enterprise AI workflows

1. **Budget context explicitly.** Adopt a component table like section 2 and alert on breaches. Owner: the platform team running the agent runtime.
2. **Plan for about 30% fill, alarm at 50%, stop at 70%.** Treat the rest as headroom, then validate against your own evals.
3. **Load tools lazily.** Keep 3 to 5 tools non-deferred and defer the rest behind search.
4. **Treat tool results as disposable.** Clear at 100K, keep the last 3, cap each result at 10K to 25K tokens.
5. **Make compaction testable.** Trigger at 100K to 150K by default, and build evals that cross compaction boundaries.
6. **Govern memory.** Keep the loaded index under about 8K tokens; version, date and expire entries.
7. **Choose isolation by task shape.** 3 to 5 sub-agents for parallel breadth work; a single agent with compaction for tightly coupled work.
8. **Pin and test against a spec version.** Plan for the 12-month deprecation windows and use trace propagation.
9. **Assign failure ownership.** When an agent fails because of bad context, the owner is whoever controls the context assembly layer. Trace-level observability is how they find out.

## Bottom line

A 1M window raises the ceiling on what an agent can hold; it does not raise the quality of what it attends to. Vendor defaults still trigger clearing at 100K and compaction at 150K, so plan to use about a third of the window and treat the rest as headroom. MCP is the plumbing; the budget is policy, and policy is yours to set.

## Sources

- [Effective context engineering for AI agents, Anthropic](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
- [Context windows, Claude Platform docs](https://platform.claude.com/docs/en/build-with-claude/context-windows)
- [Context editing, Claude Platform docs](https://platform.claude.com/docs/en/build-with-claude/context-editing)
- [Compaction, Claude Platform docs](https://platform.claude.com/docs/en/build-with-claude/compaction)
- [Tool search tool, Claude Platform docs](https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-search-tool)
- [Managing context on the Claude Developer Platform, Anthropic](https://www.anthropic.com/news/context-management)
- [Introducing advanced tool use, Anthropic](https://www.anthropic.com/engineering/advanced-tool-use)
- [Code execution with MCP, Anthropic](https://www.anthropic.com/engineering/code-execution-with-mcp)
- [How we built our multi-agent research system, Anthropic](https://www.anthropic.com/engineering/multi-agent-research-system)
- [How Claude remembers your project, Claude Code docs](https://code.claude.com/docs/en/memory)
- [Connect Claude Code to tools via MCP, Claude Code docs](https://code.claude.com/docs/en/mcp)
- [Auto-compact threshold issue #31806, anthropics/claude-code](https://github.com/anthropics/claude-code/issues/31806)
- [Context Rot, Chroma Research](https://www.trychroma.com/research/context-rot)
- [The 2026-07-28 MCP Specification Release Candidate, MCP Blog](https://blog.modelcontextprotocol.io/posts/2026-07-28-release-candidate/)
- [The 2026 MCP Roadmap, MCP Blog](https://blog.modelcontextprotocol.io/posts/2026-mcp-roadmap/)
