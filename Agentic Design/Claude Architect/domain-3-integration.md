# Domain 3: Integration
**Exam weight: 19% (highest-weighted domain) | Claude Certified Architect — Professional**

> Integration is where architecture meets reality: tool bloat, auth boundaries, retrieval quality, and observability at production volume. This is the heaviest-weighted domain on the exam for a reason — it's where well-designed systems (Domain 1) either survive contact with real traffic or don't.

---

## 1. Tooling, Auth & Trade-off Configuration

### 1.1 Capability bloat

Every tool definition — name, description, and input schema — loads as **input tokens on every single turn**, plus an injected tool-use system prompt. The whole catalog is priced each turn *whether or not any tool fires*. Thousands of tool definitions can consume hundreds of thousands of tokens before the actual request is even processed.

**Bloat has two distinct failure signatures:**
1. **Wrong-tool selection** — driven by similar names and overlapping descriptions between tools
2. **Wrong parameters** — a reasonable tool gets malformed arguments, fails at execution, and forces costly retries

**Confirm bloat with three diagnostic reads, not intuition:**
- Tool-call count — runaway counts signal the agent exploring an ambiguous toolset
- Token consumption before vs. during the task
- Reasoning transcripts — these expose vague/overlapping descriptions and tools that always load but never fire

**Right-size with four moves:**
1. **Consolidate** related operations into one tool with an `action` parameter (e.g., one `ticket_ops(action, ...)` instead of five separate `create_ticket`, `update_ticket`, `close_ticket`...)
2. **Namespace** names with a service prefix (`crm.lookup`, `network.diagnose`) to reduce collision and ambiguity
3. **Return only high-signal fields** plus stable identifiers, with verbosity controls, instead of dumping full API payloads back into context
4. **Defer rarely-used tools** while keeping the hot few always loaded (see §3.3, progressive discovery)

**The core trade-off:** fewer, more capable tools cut ambiguity, cost, and latency — but each one hides more internal logic and gives the agent less granular control. After right-sizing, **re-run the same tasks against the bloated baseline** to prove accuracy didn't slip from a capability cut that went too deep.

### 1.2 Authentication and authorization

**Least privilege**: grant only the minimum permissions and scope the task needs, so a compromise has a bounded blast radius. A summarizing agent has no business holding a delete tool — every exposed tool is both attack surface *and* additional context for the model to reason (and potentially err) over.

**Deny rules are evaluated first and override any allow rule.** Encode hard prohibitions as denies — the *absence* of an allow only produces a prompt-level suggestion not to act, not an enforced block. A pre-tool hook can return one of four decisions: `allow`, `deny`, `ask`, or `defer`. Human confirmation is a protocol-level norm reserved for calls that cause real, potentially irreversible harm.

**MCP authorization is transport-dependent** — this is a frequently-missed distinction:

| Transport | Auth model | Typical use |
|---|---|---|
| **Streamable HTTP** (remote) | Subset of OAuth 2.1 with PKCE, binding the token to the client that started the flow | Many clients over the network |
| **stdio** (local) | No OAuth at all — reads credentials directly from the local environment | Single client, local subprocess |

Claiming OAuth applies to stdio is a classic misread — stdio servers typically serve a single client and pull credentials from environment variables, not an OAuth flow.<sup>[1]</sup>

**Token hygiene has three non-negotiable rules:**
1. A `resource` parameter binds the token to one specific server
2. The token rides in an `Authorization` header on every request
3. It **never** travels in the query string, where logs will capture it

Servers must validate token *audience* and never forward a client's token upstream — the "confused deputy" defense against a server impersonating the client to reach a third system it shouldn't.

**The most common real-world gap: the over-privileged token.** It authenticates cleanly, and every call still *looks* authorized — the gap is invisible in normal operation because nothing fails. Treat all tool results as untrusted input, kept inside tool-result blocks, to block indirect prompt injection (Domain 2, §2.5), and require per-user attribution plus a full audit trail of who invoked which tool, when, with what inputs.

### 1.3 Accuracy-latency trade-offs

**Pull the levers in this order:**

1. **Model-tier routing** — the single biggest mover. Send bounded, high-volume work (classification, extraction) to the fast tier; reserve the flagship for genuinely hard reasoning.
2. **Reasoning depth (effort)** — gains flatten well before the maximum setting; there's rarely a reason to run everything at `max`.
3. **Trade real accuracy away** — the last resort, not the first lever.

**Three levers are accuracy-neutral** and should be exhausted before any of the above:
- **Prompt caching** — reuses a stable prefix, no quality impact
- **Batch processing** — trades immediacy for a large discount; this is exactly where deep reasoning with no hard deadline belongs
- **Streaming** — cuts *perceived* latency by surfacing tokens early, even though total generation time is unchanged

**Optimize for the binding constraint** — the target you're closest to breaching — and let the others stay comfortably satisfied. Pick the *cheapest* lever that meets the binding target, not the most accurate configuration you can theoretically build. Record what you gave up, and why.

**Justification is comparative.** Name the credible alternative you rejected and show it either misses the binding target or costs more for no benefit. Three traps to avoid when defending a config to stakeholders:
- Maximizing accuracy when the *latency* SLA is actually the binding constraint
- Blurring an internal SLO with an external SLA in the write-up
- Calling caching or batch an "accuracy sacrifice" — they move latency and cost, never quality

---

## 2. RAG & Retrieval

### 2.1 Chunking and indexing

Chunking has three independent knobs:

| Knob | Trade-off |
|---|---|
| **Size** | Smaller sharpens precision; larger preserves surrounding context |
| **Boundary** | Fixed-window (by token count) vs. semantic (by meaning) |
| **Overlap** | Keeps ideas that straddle a cut point whole |

Anthropic reports that fixed-window chunking lands roughly comparable to semantic chunking in practice — **embedding quality frequently dominates the chunking choice entirely**, so don't over-invest engineering time in boundary sophistication before validating the embedder itself.

### 2.2 Contextual Retrieval

Anthropic's **Contextual Retrieval** technique prepends a short, model-written context (roughly 50–100 tokens) to each chunk **at indexing time**, not query time. The augmented chunk feeds *both* the dense embedding index and the BM25 lexical index simultaneously.<sup>[2]</sup> Caching the source document once (Domain 2, §4.5) is what makes generating this per-chunk context economically viable at scale.

**The reported gains stack in order** (measured as 1 − recall@20, i.e., retrieval *failure* rate):

| Technique | Reduction in retrieval failure |
|---|---|
| Contextual Embeddings alone | ~35% |
| + Contextual BM25 | ~49% |
| + Reranking | ~67% |

The 49% figure requires *both* halves (embeddings + BM25) — one alone doesn't get you there. Treat all three numbers as Anthropic's reported benchmark figures, not universal constants; validate on your own corpus and query distribution.<sup>[2]</sup>

### 2.3 Dense vs. sparse retrieval

- **Dense (embeddings)** — matches meaning; can blur exact identifiers (SKUs, error codes, ticket IDs)
- **Sparse (BM25)** — matches exact terms; misses pure paraphrases

Derive the index choice from data shape and query pattern. **Build both in parallel** when queries genuinely mix exact-term lookups with conceptual questions, and choose the embedder *first* — it sets a ceiling on retrieval quality that no amount of index tuning can recover from.

### 2.4 When to skip RAG entirely

**Below roughly 200,000 tokens (~500 pages)**, load the whole knowledge base into the prompt with caching and skip the retrieval pipeline entirely. This threshold is era-dated guidance tied to current context-window sizes, not a law — it will move as context windows grow. RAG earns its keep when:
- The corpus exceeds the prompt
- Content churns often (a monolithic cached prompt gets invalidated on every update)
- You need precise retrieval over millions of documents where "load it all" is structurally impossible

### 2.5 Hybrid search and rank fusion

Dense and sparse failures rarely land on the same query — the *union* of their results carries higher recall than either list alone. Their raw scores live on different scales, so **Reciprocal Rank Fusion (RRF)** merges by *rank position*, not score: each appearance scores `1 / (constant + rank)`. A chunk appearing on both lists sums its contributions, so agreement compounds rather than double-counting.

**Worked math** (constant = 60): a chunk at dense rank 2 and sparse rank 5 scores `1/62 + 1/65 ≈ 0.0323` — nearly double a chunk that tops the dense list alone at `1/61 ≈ 0.0164`. Fusion trusts *agreement between two independent views* over one list's enthusiasm for a single result.

### 2.6 Two-stage reranking

Reranking retrieves a wide candidate set for recall (Anthropic's example: top ~150), then a **cross-encoder** that scores query and document *together* reorders that pool down to a precise top set (~20). Fusion merges multiple lists cheaply by rank position with no model call; reranking rescores one pool with a full model pass per candidate — more expensive, more precise. Production pipelines commonly use both in sequence: fusion first, then reranking on the fused shortlist.

### 2.7 Localizing a bad RAG answer

**Recall@k is the primary metric.** If the right chunk is never retrieved, nothing downstream — no amount of prompt engineering — can save the answer. Precision@k, MRR, and NDCG refine *how cleanly* results rank, but only matter once recall is solid.

**Retrieval is the dominant failure origin in RAG systems.** A confident-but-wrong answer showing up right after a document refresh points at a **stale index**, not the model — run recall@k on the new corpus before touching prompts or weights. Measure recall **per leg separately** (dense vs. sparse); fusion is forgiving by design and can mask a dying leg for weeks before it's noticed in the combined metric.

---

## 3. Protocols, Observability & Context Strategy

### 3.1 Choosing the integration mechanism

Four mechanisms, four distinct jobs:

| Mechanism | What it does | Constraint |
|---|---|---|
| **Built-in tools** (web search, code execution) | Run on Anthropic infrastructure, no handler code | Cannot reach private systems |
| **Custom tools** | You define and run them, reaching arbitrary internal APIs | You own the handler code and its security |
| **MCP** | Open, model-agnostic standard packaging tools, resources, and prompts behind one reusable cross-host interface | Overhead only earns itself with cross-host reuse |
| **Skills** | Procedural knowledge loaded via progressive disclosure | On the Claude API, a Skill cannot make live network calls — pair it with MCP or a custom tool for fresh data |

**Select on three questions:**
1. **Connectivity** — does this need to reach a private system? (Rules out built-ins.)
2. **Reuse** — is this a single consumer (a direct call is right-sized) or genuine cross-host reuse (MCP earns its overhead)?
3. **Consistency** — is this a repeatable workflow? (That's a Skill.)

### 3.2 MCP transports and topology

MCP defines exactly two standard transports:
- **stdio** — launches the server as a local subprocess, typically serving a single client, credentials from the environment, no OAuth
- **Streamable HTTP** — serves many clients over the network, with OAuth; this replaced the deprecated legacy streaming (SSE) transport<sup>[1]</sup>

MCP standardizes **vertical integration** — one agent reaching down to tools, the common case. **Agent-to-agent** is *horizontal* coordination between independent peer agents exchanging tasks and results, needed only for genuinely multi-agent workloads (Domain 1, §3). A single agent calling many tools through MCP is *not* agent-to-agent, no matter how many tools it calls.

Capability still has a context price even inside MCP: thousands of loaded tool definitions can burn hundreds of thousands of tokens before a request even runs. Progressive disclosure (§3.3 below) loads them on demand instead of up front. Let the shape of the connection choose the mechanism — never pick the newest-sounding label.

### 3.3 Observability at scale (OpenTelemetry)

The Claude Agent SDK and Claude Code emit telemetry through **OpenTelemetry (OTLP)** exclusively, exporting to a swappable, vendor-neutral backend.<sup>[3]</sup>

**Three signals answer three different questions:**
- **Metrics** — fleet health: request counts, token totals, error rates
- **Log events** — discrete, searchable moments
- **Traces** — follow one request through causally linked spans

Every signal ships **off by default** and is enabled per signal — the full trace view sits behind a beta flag as of current documentation.

**The span tree reads top-down:**
```
interaction span (whole turn: total duration, session attributes)
 ├── model-request span (tokens, inference timing)
 ├── tool span
 │    └── subagent spans nest HERE, under the tool span that spawned them
 └── hook span
```

A subagent's own spans nest under the tool span that spawned it, keeping cost and latency **attributable to the specific delegate**, not blended into the lead's totals.

**Two identifiers, two different jobs:**
- **Session ID** — groups many traces into one logical conversation
- **Trace context** (trace ID + parent span ID) — links spans *within* a single trace and propagates across process boundaries

Miss either one and your traces are technically correct but **orphaned** — unfindable and unlinkable at the volume where you'll actually need them.

**At scale:** stamp resource attributes at the source — service, environment, version, tenant — because an attribute you forget to attach is a dimension you can *never* slice by after the fact. Per-request inspection becomes infeasible at volume, so aggregates become the default view and traces become the drill-down; tenant tags drive per-tenant routing and isolation into a SIEM.

**Keep spans structural by default** — recording the call, timing, and status, but *not* the payload. Make content logging a **narrow, explicit opt-in**, since it drags sensitive data (and its compliance burden — see Domain 5) into the observability pipeline. Treat export reliability as a design decision: batch export buffers spans in memory, and a process **must flush on exit or lose the last spans of the run** — exactly the spans you'd want most when diagnosing a crash. Read sampled data as sampled, never as complete.

### 3.4 Progressive discovery vs. monolithic context

**Monolithic context is taxed twice at scale:** tool-definition tokens load before the request even starts, and intermediate results re-traverse the context window until it fills with data the final answer never uses. This is an **architectural decision made early**, not a tactical audit you run on one misbehaving agent — retrofitting discovery onto a monolithic design later is expensive.

**Four disclosure mechanisms share one move — keep definitions out of context until they earn their place:**
1. **Filesystem layout** — tools exposed as code files, read on demand
2. **Tool search** — defers rarely-used tools while hot-path tools stay loaded
3. **Detail-level verbosity parameters** — request more or less detail per call
4. **On-demand reading** of full schemas or document bodies

**Code execution with MCP** has the agent write and run code that drives the tools itself — intermediate results stay in the runtime, and only the distilled final result returns to context. The difference between this and monolithic loading barely matters at three tool calls and is the whole ballgame at thirty.

Anthropic reported that code-on-filesystem tool loading cut one worked example's total token usage by **about 98.7%.**<sup>[4]</sup> Treat that as evidence the *pattern* works, not a guaranteed number — your own savings depend heavily on toolset size and task shape.

**Monolithic still wins** for a handful of tools, a stable knowledge base small enough to cache inline, and fixed tools and data. Discovery's price is real: lookup round trips, code-execution infrastructure to run and secure, and more moving parts to debug. **The two traps:** front-loading everything and then fixing bloat cosmetically after the fact, and standing up full discovery machinery for what is, in reality, a five-tool agent.

---

## Worked Architecture: RAG + MCP for a Telecom Knowledge Base

```
┌───────────────────────────┐
│ Knowledge base: 40,000     │   > 200k tokens → RAG required
│ pages of network runbooks  │
└─────────────┬───────────────┘
              ▼
┌────────────────────────────────────────────┐
│ INDEXING (offline, cached source document)   │
│  chunk (fixed-window, ~400 tok, 15% overlap) │
│  → generate 50-100 tok context per chunk     │
│  → dense embedding index + BM25 index         │
└─────────────┬────────────────────────────────┘
              ▼
┌────────────────────────────────────────────┐
│ QUERY TIME                                    │
│  dense search (top 150) + sparse search (150) │
│  → Reciprocal Rank Fusion                     │
│  → cross-encoder rerank → top 20              │
└─────────────┬────────────────────────────────┘
              ▼
┌────────────────────────────────────────────┐
│ AGENT (MCP, Streamable HTTP, OAuth 2.1)       │
│  tools: kb.search (namespaced, consolidated), │
│  network.diagnose, ticket.update              │
│  least-privilege scoped per tool               │
│  OTel spans: interaction → tool → subagent     │
└────────────────────────────────────────────┘
```

Recall@20 is monitored per leg (dense, sparse) as the primary eval; a drop right after the next runbook refresh is diagnosed as a **stale-index** problem before anyone touches the prompt.

---

## Key Takeaways

- Every loaded tool definition costs tokens on every turn, whether or not it fires. Consolidate, namespace, defer.
- Deny rules always win over allow rules. Least privilege bounds blast radius; the over-privileged token is the gap that hides in plain sight.
- MCP auth is transport-dependent: OAuth 2.1+PKCE over Streamable HTTP, no OAuth at all over stdio.
- Pull accuracy-neutral levers (caching, batch, streaming) before trading real accuracy away, and optimize for whichever SLO is the binding constraint.
- Contextual Retrieval's gains stack: embeddings (~35%) → +BM25 (~49%) → +rerank (~67%), measured as reduction in retrieval failure — all benchmark figures, not constants.
- Retrieval is the dominant failure origin in RAG. Recall@k first; precision/MRR/NDCG only matter once recall is solid.
- OTel spans nest hierarchically (interaction → tool → subagent); keep them structural by default and opt in to payload logging narrowly.
- Progressive discovery is an early architecture decision, not a later bloat-fix — but it's real overhead that a five-tool agent doesn't need.

---

## References

1. Model Context Protocol Specification, *"Authorization"* — https://modelcontextprotocol.io/specification/2025-06-18/basic/authorization
2. Anthropic News, *"Introducing Contextual Retrieval,"* Sep 19, 2024 — https://www.anthropic.com/news/contextual-retrieval
3. Anthropic Docs, *"Monitoring — OpenTelemetry"* — https://docs.claude.com/en/docs/claude-code/monitoring-usage
4. Anthropic Engineering, *"Code execution with MCP: Building more efficient AI agents"* — https://www.anthropic.com/engineering/code-execution-with-mcp
5. Anthropic Docs, *"Model Context Protocol (MCP)"* — https://docs.claude.com/en/docs/agents-and-tools/mcp
6. OpenTelemetry, *official documentation* — https://opentelemetry.io/docs/
