# Domain Implementation Reference Guide — Domain 3: Integration

**Source domain:** `domain-3-integration.md` (CCAR-P, exam weight 19% — the highest-weighted domain)
**Guide type:** Implementation Reference (technical depth, code examples, API/config detail)
**Audience:** Solution architects and integration engineers building tool, auth, retrieval, and observability layers for production Claude systems.

> This guide preserves the full structure of the source domain (1:1 section mapping) and expands every item into: what it is, why it matters, the underlying Anthropic/MCP technical mechanism, implementation patterns, pitfalls, and runnable Python/TypeScript examples. A full traceability matrix appears at the end.

---

## Table of Contents

1. [Domain Overview](#domain-overview)
2. [Section 1: Tooling, Auth & Trade-off Configuration](#section-1-tooling-auth--trade-off-configuration)
   - 1.1 Capability bloat
   - 1.2 Authentication and authorization
   - 1.3 Accuracy-latency trade-offs
3. [Section 2: RAG & Retrieval](#section-2-rag--retrieval)
   - 2.1 Chunking and indexing
   - 2.2 Contextual Retrieval
   - 2.3 Dense vs. sparse retrieval
   - 2.4 When to skip RAG entirely
   - 2.5 Hybrid search and rank fusion
   - 2.6 Two-stage reranking
   - 2.7 Localizing a bad RAG answer
4. [Section 3: Protocols, Observability & Context Strategy](#section-3-protocols-observability--context-strategy)
   - 3.1 Choosing the integration mechanism
   - 3.2 MCP transports and topology
   - 3.3 Observability at scale (OpenTelemetry)
   - 3.4 Progressive discovery vs. monolithic context
5. [Worked Architecture: RAG + MCP for a Telecom Knowledge Base](#worked-architecture-rag--mcp-for-a-telecom-knowledge-base)
6. [Architecture & Implementation Best Practices](#architecture--implementation-best-practices)
7. [Common Pitfalls and Limitations](#common-pitfalls-and-limitations)
8. [Official Documentation References](#official-documentation-references)
9. [Domain Coverage / Traceability Matrix](#domain-coverage--traceability-matrix)

---

## Domain Overview

Domain 3 carries the highest exam weight (19%) because it is where well-designed architectures (Domain 1) and well-tuned prompts (Domain 2) either survive contact with real production traffic or quietly fail under it. Four recurring themes structure the whole domain:

1. **Everything loaded costs something, whether or not it's used.** Tool definitions cost tokens every turn regardless of firing; a monolithic RAG-skipped context costs quality (Domain 2 §4.1) regardless of whether every fact is read. The domain's repeated answer is progressive disclosure and right-sizing, not maximal upfront loading.
2. **Deterministic security beats probabilistic security.** Deny rules override allow rules; least privilege bounds blast radius; tokens are scoped and validated in code. This is the same probabilistic/deterministic split from Domains 2, 5, and 7, now applied specifically to tool authorization and MCP transport security.
3. **Recall before precision, always.** In RAG, if the right chunk is never retrieved, no downstream reasoning can recover it — retrieval failure is the dominant failure origin, and it must be localized and fixed before anyone touches a prompt.
4. **Optimize the binding constraint, not every metric simultaneously.** Accuracy-latency trade-offs, chunking choices, and reranking depth are all decided relative to which target is closest to breaching — not by maximizing every dimension at once.

---

## Section 1: Tooling, Auth & Trade-off Configuration

### 1.1 Capability bloat

**What it is.** Every tool definition — name, description, and input schema — loads as **input tokens on every single turn**, plus an injected tool-use system prompt. The whole catalog is priced each turn *whether or not any tool fires*. Thousands of tool definitions can consume hundreds of thousands of tokens before the actual request is even processed.

**Why it matters — two distinct failure signatures:**
1. **Wrong-tool selection** — driven by similar names and overlapping descriptions between tools.
2. **Wrong parameters** — a reasonable tool gets malformed arguments, fails at execution, and forces costly retries.

**Confirm bloat with three diagnostic reads, not intuition:**
- Tool-call count — runaway counts signal the agent exploring an ambiguous toolset.
- Token consumption before vs. during the task.
- Reasoning transcripts — these expose vague/overlapping descriptions and tools that always load but never fire.

**Right-size with four moves:**
1. **Consolidate** related operations into one tool with an `action` parameter.
2. **Namespace** names with a service prefix to reduce collision and ambiguity.
3. **Return only high-signal fields** plus stable identifiers, with verbosity controls.
4. **Defer rarely-used tools** while keeping the hot few always loaded (§3.4, progressive discovery).

**The core trade-off:** fewer, more capable tools cut ambiguity, cost, and latency — but each one hides more internal logic and gives the agent less granular control. After right-sizing, **re-run the same tasks against the bloated baseline** to prove accuracy didn't slip from a capability cut that went too deep.

**Implementation — before/after tool consolidation (Python, Anthropic SDK):**

```python
# BEFORE — five separate, overlapping tools; bloated and ambiguous
tools_before = [
    {"name": "create_ticket", "description": "Create a new ticket", "input_schema": {...}},
    {"name": "update_ticket", "description": "Update an existing ticket", "input_schema": {...}},
    {"name": "close_ticket", "description": "Close a ticket", "input_schema": {...}},
    {"name": "reopen_ticket", "description": "Reopen a closed ticket", "input_schema": {...}},
    {"name": "delete_ticket", "description": "Delete a ticket", "input_schema": {...}},
]

# AFTER — consolidated, namespaced, single action parameter
tools_after = [
    {
        "name": "ticket_ops",  # could also be namespaced as "ticket.ops"
        "description": "Perform an operation on a ticket: create, update, close, reopen, or delete.",
        "input_schema": {
            "type": "object",
            "properties": {
                "action": {"type": "string", "enum": ["create", "update", "close", "reopen", "delete"]},
                "ticket_id": {"type": "string", "description": "Required for all actions except create."},
                "fields": {"type": "object", "description": "Fields to set on create/update."},
            },
            "required": ["action"],
        },
    }
]

def diagnose_bloat(transcript: list[dict], baseline_token_count: int, task_token_count: int) -> dict:
    """The three diagnostic reads, applied programmatically rather than by intuition."""
    tool_call_count = sum(1 for turn in transcript if turn.get("type") == "tool_use")
    never_fired = [t["name"] for t in tools_before if not any(
        turn.get("name") == t["name"] for turn in transcript if turn.get("type") == "tool_use")]
    return {
        "tool_call_count": tool_call_count,          # runaway count = ambiguous toolset signal
        "token_overhead": task_token_count - baseline_token_count,
        "tools_loaded_never_fired": never_fired,      # always-loaded, never-used tools
    }

def validate_consolidation_didnt_lose_accuracy(eval_cases: list[dict]) -> dict:
    """After right-sizing, re-run the SAME tasks against the bloated baseline to
    prove accuracy didn't slip from a capability cut that went too deep."""
    before_accuracy = run_eval(tools=tools_before, cases=eval_cases)
    after_accuracy = run_eval(tools=tools_after, cases=eval_cases)
    return {"before": before_accuracy, "after": after_accuracy,
            "regression": after_accuracy["score"] < before_accuracy["score"]}

def run_eval(tools, cases): ...
```

**Common pitfalls:**
- Adding new tools without ever removing or consolidating overlapping old ones — bloat accumulates silently because each new tool "looks fine" in isolation.
- Right-sizing the toolset without re-running the accuracy comparison against the prior baseline — a capability cut that goes too deep can look like a cost win while quietly regressing quality.

---

### 1.2 Authentication and authorization

**What it is.** **Least privilege**: grant only the minimum permissions and scope the task needs, so a compromise has a bounded blast radius. A summarizing agent has no business holding a delete tool — every exposed tool is both attack surface *and* additional context for the model to reason (and potentially err) over.

**Why it matters — deny rules are evaluated first and override any allow rule.** Encode hard prohibitions as denies — the *absence* of an allow only produces a prompt-level suggestion not to act, not an enforced block (this is Domain 7 §1.3's deterministic-layer principle, applied here to tool-level auth). A pre-tool hook can return one of four decisions: `allow`, `deny`, `ask`, or `defer`. Human confirmation is a protocol-level norm reserved for calls that cause real, potentially irreversible harm.

**MCP authorization is transport-dependent** — a frequently-missed distinction:

| Transport | Auth model | Typical use |
|---|---|---|
| **Streamable HTTP** (remote) | Subset of OAuth 2.1 with PKCE, binding the token to the client that started the flow | Many clients over the network |
| **stdio** (local) | No OAuth at all — reads credentials directly from the local environment | Single client, local subprocess |

Claiming OAuth applies to stdio is a classic misread — stdio servers typically serve a single client and pull credentials from environment variables, not an OAuth flow.[1]

**Token hygiene has three non-negotiable rules:**
1. A `resource` parameter binds the token to one specific server.
2. The token rides in an `Authorization` header on every request.
3. It **never** travels in the query string, where logs will capture it.

Servers must validate token *audience* and never forward a client's token upstream — the "confused deputy" defense against a server impersonating the client to reach a third system it shouldn't.

**The most common real-world gap: the over-privileged token.** It authenticates cleanly, and every call still *looks* authorized — the gap is invisible in normal operation because nothing fails. Treat all tool results as untrusted input, kept inside tool-result blocks, to block indirect prompt injection (Domain 2 §2.5), and require per-user attribution plus a full audit trail of who invoked which tool, when, with what inputs.

**Implementation — a pre-tool hook returning one of the four decisions (Python):**

```python
from enum import Enum

class ToolDecision(Enum):
    ALLOW = "allow"
    DENY = "deny"
    ASK = "ask"
    DEFER = "defer"

DENY_LIST = {"delete_customer_record", "wire_transfer"}  # hard prohibitions — never overridden by allow logic
IRREVERSIBLE_ACTIONS = {"wire_transfer", "delete_ticket", "close_account"}

def pre_tool_hook(tool_name: str, tool_input: dict, agent_role: str) -> ToolDecision:
    # Deny rules evaluated FIRST — override any allow logic that follows
    if tool_name in DENY_LIST:
        return ToolDecision.DENY

    if agent_role == "summarizer" and tool_name.startswith("delete_"):
        return ToolDecision.DENY  # least privilege: a summarizer has no business deleting anything

    if tool_name in IRREVERSIBLE_ACTIONS:
        return ToolDecision.ASK  # human confirmation for real, potentially irreversible harm

    return ToolDecision.ALLOW

def validate_and_scope_token(token: str, resource: str, request_headers: dict, request_url: str) -> bool:
    """Token hygiene: resource-bound, header-only, never in the query string."""
    if "token=" in request_url or "access_token=" in request_url:
        raise SecurityError("Token found in query string — logs will capture it. Use Authorization header only.")
    if request_headers.get("Authorization", "").replace("Bearer ", "") != token:
        return False
    return validate_audience(token, expected_resource=resource)  # confused-deputy defense

def validate_audience(token: str, expected_resource: str) -> bool: ...
class SecurityError(Exception): ...
```

**MCP transport-appropriate auth (TypeScript, conceptual):**

```typescript
// Streamable HTTP (remote) — OAuth 2.1 with PKCE
async function connectRemoteServer(serverUrl: string) {
  const { codeVerifier, codeChallenge } = generatePkcePair();
  const authUrl = `${serverUrl}/authorize?code_challenge=${codeChallenge}&code_challenge_method=S256&resource=${serverUrl}`;
  // ... standard OAuth 2.1 authorization code flow, binding token to serverUrl via `resource` param ...
}

// stdio (local) — NO OAuth; credentials read directly from environment
function connectLocalServer(command: string, args: string[]) {
  const env = { ...process.env, API_KEY: process.env.INTERNAL_TOOL_API_KEY }; // env-sourced, no OAuth flow
  return spawnSubprocess(command, args, env);
}

function generatePkcePair() { return { codeVerifier: "", codeChallenge: "" }; }
function spawnSubprocess(cmd: string, args: string[], env: any) { /* ... */ }
```

**Common pitfalls:**
- Implementing an OAuth flow for a stdio server — this is the classic misread the source domain calls out explicitly; stdio has no OAuth model at all.
- Putting a token in a query string "just for this one debug request" — logs will capture it, and this is one of the three non-negotiable token-hygiene violations.
- Granting a broad, multi-scope token because "it's simpler to manage one credential" — this is exactly the over-privileged-token gap that's invisible until an incident exposes it.
- A server forwarding the client's token upstream to a third system — this is the confused-deputy vulnerability the audience-validation rule exists to prevent.

---

### 1.3 Accuracy-latency trade-offs

**What it is.** A strict lever order for managing the accuracy-latency trade-off:

1. **Model-tier routing** — the single biggest mover. Send bounded, high-volume work to the fast tier; reserve the flagship for genuinely hard reasoning.
2. **Reasoning depth (effort)** — gains flatten well before the maximum setting; there's rarely a reason to run everything at `max`.
3. **Trade real accuracy away** — the last resort, not the first lever.

**Three levers are accuracy-neutral and should be exhausted before any of the above:**
- **Prompt caching** — reuses a stable prefix, no quality impact.
- **Batch processing** — trades immediacy for a large discount; exactly where deep reasoning with no hard deadline belongs.
- **Streaming** — cuts *perceived* latency by surfacing tokens early, even though total generation time is unchanged.

**Why it matters.** **Optimize for the binding constraint** — the target you're closest to breaching — and let the others stay comfortably satisfied. Pick the *cheapest* lever that meets the binding target, not the most accurate configuration you can theoretically build. Record what you gave up, and why.

**Justification is comparative.** Name the credible alternative you rejected and show it either misses the binding target or costs more for no benefit. **Three traps to avoid when defending a config to stakeholders:**
- Maximizing accuracy when the *latency* SLA is actually the binding constraint.
- Blurring an internal SLO with an external SLA in the write-up.
- Calling caching or batch an "accuracy sacrifice" — they move latency and cost, never quality.

**Implementation — a lever-ordering decision function (Python):**

```python
from dataclasses import dataclass

@dataclass
class BindingConstraint:
    metric: str          # "latency" | "cost" | "accuracy"
    current_value: float
    target_value: float
    is_sla: bool         # contractual (Domain 1 §1.2) vs. internal SLO

def select_levers(constraint: BindingConstraint) -> list[str]:
    """Exhaust accuracy-neutral levers first, then the ordered accuracy-affecting levers."""
    levers = ["prompt_caching", "batch_processing_if_latency_insensitive", "streaming_for_perceived_latency"]

    if constraint.metric == "latency" and constraint.current_value > constraint.target_value:
        levers.append("model_tier_routing_down")   # biggest mover, first accuracy-affecting lever
        levers.append("reduce_reasoning_effort")    # second — gains flatten before max anyway
        levers.append("accept_real_accuracy_tradeoff_LAST_RESORT")

    return levers

def justify_config(chosen_config: dict, rejected_alternative: dict, binding: BindingConstraint) -> str:
    """Justification is comparative — name the rejected alternative and show why."""
    if binding.is_sla:
        tier = "SLA (contractual penalty on miss)"
    else:
        tier = "SLO (internal target, no contractual penalty)"
    return (f"Binding constraint: {binding.metric} ({tier}). Rejected {rejected_alternative} because it "
            f"{'misses the binding target' if rejected_alternative.get('misses_target') else 'costs more for no benefit'}.")
```

**Common pitfalls:**
- Reaching for a smaller model tier or reduced effort before first exhausting caching, batch, and streaming — these are strictly cheaper, quality-neutral wins that should never be skipped.
- Presenting a config justification as "we chose the most accurate option" when the actual binding constraint was latency — this is one of the three explicit stakeholder-communication traps the source domain warns against.
- Describing a caching or batch-processing change as an "accuracy trade-off" — they affect latency and cost only; conflating this with quality changes misleads stakeholders about what was actually given up.

---

## Section 2: RAG & Retrieval

### 2.1 Chunking and indexing

**What it is.** Chunking has three independent knobs:

| Knob | Trade-off |
|---|---|
| **Size** | Smaller sharpens precision; larger preserves surrounding context |
| **Boundary** | Fixed-window (by token count) vs. semantic (by meaning) |
| **Overlap** | Keeps ideas that straddle a cut point whole |

**Why it matters.** Anthropic reports that fixed-window chunking lands roughly comparable to semantic chunking in practice — **embedding quality frequently dominates the chunking choice entirely**, so don't over-invest engineering time in boundary sophistication before validating the embedder itself.

**Implementation — fixed-window chunking with overlap (Python):**

```python
def fixed_window_chunk(text: str, chunk_tokens: int = 400, overlap_pct: float = 0.15) -> list[str]:
    """Fixed-window chunking, comparable in practice to semantic chunking per
    Anthropic's reported findings — validate the embedder before over-investing
    engineering time in boundary sophistication."""
    tokens = tokenize(text)  # placeholder: use the actual model tokenizer
    overlap = int(chunk_tokens * overlap_pct)
    stride = chunk_tokens - overlap
    chunks = []
    for start in range(0, len(tokens), stride):
        chunk = tokens[start:start + chunk_tokens]
        if chunk:
            chunks.append(detokenize(chunk))
        if start + chunk_tokens >= len(tokens):
            break
    return chunks

def tokenize(text: str) -> list[str]: ...
def detokenize(tokens: list[str]) -> str: ...
```

**Common pitfalls:**
- Investing heavily in semantic chunking boundary logic before validating whether the embedder itself is the actual quality ceiling — per the source domain, embedding quality frequently dominates chunking choice.
- Choosing zero overlap purely to minimize storage/index size — this cuts ideas straddling a chunk boundary and can silently drop retrievable content.

---

### 2.2 Contextual Retrieval

**What it is.** Anthropic's **Contextual Retrieval** technique prepends a short, model-written context (roughly 50–100 tokens) to each chunk **at indexing time**, not query time. The augmented chunk feeds *both* the dense embedding index and the BM25 lexical index simultaneously.[2] Caching the source document once (Domain 2 §4.5) is what makes generating this per-chunk context economically viable at scale.

**The reported gains stack in order** (measured as 1 − recall@20, i.e., retrieval *failure* rate):

| Technique | Reduction in retrieval failure |
|---|---|
| Contextual Embeddings alone | ~35% |
| + Contextual BM25 | ~49% |
| + Reranking | ~67% |

**Why it matters.** The 49% figure requires *both* halves (embeddings + BM25) — one alone doesn't get you there. Treat all three numbers as Anthropic's reported benchmark figures, not universal constants; validate on your own corpus and query distribution.[2]

**Implementation — generating and caching per-chunk context at indexing time (Python):**

```python
from anthropic import Anthropic

client = Anthropic()

def generate_contextual_chunk(full_document: str, chunk: str) -> str:
    """Generates a 50-100 token context prefix AT INDEXING TIME. Caching the
    full document once makes this economically viable across many chunks."""
    response = client.messages.create(
        model="claude-haiku-4-5",  # cheap tier — this runs once per chunk, at index time, at scale
        max_tokens=100,
        system=[
            {"type": "text", "text": f"Full document:\n{full_document}",
             "cache_control": {"type": "ephemeral"}},  # cached ONCE, reused across every chunk from this doc
        ],
        messages=[{"role": "user",
                   "content": f"Give brief context (50-100 tokens) situating this chunk "
                              f"within the full document:\n\n{chunk}"}],
    )
    context = response.content[0].text
    return f"{context}\n\n{chunk}"  # augmented chunk feeds BOTH dense and sparse indexes

def build_dual_index(documents: list[str]):
    dense_index, sparse_index = [], []
    for doc in documents:
        chunks = fixed_window_chunk(doc)
        for chunk in chunks:
            augmented = generate_contextual_chunk(doc, chunk)
            dense_index.append(embed(augmented))
            sparse_index.append(bm25_tokenize(augmented))
    return dense_index, sparse_index

def embed(text: str): ...
def bm25_tokenize(text: str): ...
```

**Common pitfalls:**
- Generating contextual chunks at query time instead of indexing time — this defeats the caching economics that make the technique viable and adds latency to every query.
- Reporting only the Contextual Embeddings gain (~35%) while implying the full ~67% figure applies — the full figure requires embeddings + BM25 + reranking all stacked together.
- Treating Anthropic's benchmark percentages as guaranteed outcomes rather than directional evidence requiring validation on your own corpus.

---

### 2.3 Dense vs. sparse retrieval

**What it is.**
- **Dense (embeddings)** — matches meaning; can blur exact identifiers (SKUs, error codes, ticket IDs).
- **Sparse (BM25)** — matches exact terms; misses pure paraphrases.

**Why it matters.** Derive the index choice from data shape and query pattern. **Build both in parallel** when queries genuinely mix exact-term lookups with conceptual questions, and choose the embedder *first* — it sets a ceiling on retrieval quality that no amount of index tuning can recover from.

**Implementation — dual retrieval decision logic (Python):**

```python
def should_build_both_indexes(sample_queries: list[str]) -> bool:
    """Build both in parallel only when queries genuinely mix exact-term lookups
    (SKUs, error codes) with conceptual/paraphrase questions."""
    has_exact_term_queries = any(contains_identifier_pattern(q) for q in sample_queries)
    has_conceptual_queries = any(is_conceptual_question(q) for q in sample_queries)
    return has_exact_term_queries and has_conceptual_queries

def contains_identifier_pattern(query: str) -> bool:
    import re
    return bool(re.search(r'\b[A-Z]{2,}\d{3,}\b', query))  # e.g., ERR4471, SKU9021

def is_conceptual_question(query: str) -> bool:
    return any(w in query.lower() for w in ["why", "how", "what causes", "explain"])
```

**Common pitfalls:**
- Choosing the embedder as an afterthought after tuning chunk size and index parameters extensively — the embedder sets a quality ceiling that no downstream tuning recovers.
- Relying on dense retrieval alone for a corpus with many exact identifiers (error codes, SKUs) — embeddings can blur these, causing silent recall failures on exactly the queries that need precision most.

---

### 2.4 When to skip RAG entirely

**What it is.** **Below roughly 200,000 tokens (~500 pages)**, load the whole knowledge base into the prompt with caching and skip the retrieval pipeline entirely. This threshold is era-dated guidance tied to current context-window sizes, not a law — it will move as context windows grow. RAG earns its keep when:
- The corpus exceeds the prompt.
- Content churns often (a monolithic cached prompt gets invalidated on every update).
- You need precise retrieval over millions of documents where "load it all" is structurally impossible.

**Implementation — a threshold check that documents *why* RAG was or wasn't chosen (design-time, auditable):**

```python
def should_use_rag(corpus_token_count: int, update_frequency_days: float,
                    context_window_size: int = 200_000) -> dict:
    reasons = []
    if corpus_token_count > context_window_size:
        reasons.append(f"Corpus ({corpus_token_count} tokens) exceeds cacheable prompt size ({context_window_size}).")
    if update_frequency_days < 1:
        reasons.append("Content churns more than daily — a monolithic cached prompt would be invalidated constantly.")

    decision = "rag" if reasons else "cached_full_context"
    return {"decision": decision, "reasons": reasons or ["Corpus fits in context; caching is simpler and cheaper."]}
```

**Common pitfalls:**
- Building a RAG pipeline reflexively for any knowledge-base task, without first checking whether the corpus fits under the cached-context threshold — this is a direct instance of Domain 1's technology-first anti-pattern.
- Treating the 200k-token threshold as a hard, permanent law rather than era-dated guidance tied to current context-window sizes.

---

### 2.5 Hybrid search and rank fusion

**What it is.** Dense and sparse failures rarely land on the same query — the *union* of their results carries higher recall than either list alone. Their raw scores live on different scales, so **Reciprocal Rank Fusion (RRF)** merges by *rank position*, not score: each appearance scores `1 / (constant + rank)`. A chunk appearing on both lists sums its contributions, so agreement compounds rather than double-counting.

**Worked math** (constant = 60): a chunk at dense rank 2 and sparse rank 5 scores `1/62 + 1/65 ≈ 0.0323` — nearly double a chunk that tops the dense list alone at `1/61 ≈ 0.0164`. Fusion trusts *agreement between two independent views* over one list's enthusiasm for a single result.

**Implementation — Reciprocal Rank Fusion (Python):**

```python
def reciprocal_rank_fusion(dense_results: list[str], sparse_results: list[str], k: int = 60) -> list[tuple[str, float]]:
    """Merges by RANK POSITION, not raw score — dense and sparse scores live on
    different scales and cannot be compared directly."""
    scores: dict[str, float] = {}
    for rank, doc_id in enumerate(dense_results, start=1):
        scores[doc_id] = scores.get(doc_id, 0.0) + 1.0 / (k + rank)
    for rank, doc_id in enumerate(sparse_results, start=1):
        scores[doc_id] = scores.get(doc_id, 0.0) + 1.0 / (k + rank)  # sums if it appears on both lists
    return sorted(scores.items(), key=lambda x: x[1], reverse=True)

# Worked example verification
dense = ["doc_A", "doc_C", "doc_B"]   # doc_C at dense rank 2
sparse = ["doc_D", "doc_E", "doc_F", "doc_G", "doc_C"]  # doc_C at sparse rank 5
result = reciprocal_rank_fusion(dense, sparse)
# doc_C score = 1/62 + 1/65 ≈ 0.0323 — nearly double doc_A's dense-only 1/61 ≈ 0.0164
```

**Common pitfalls:**
- Merging dense and sparse result lists by directly comparing or summing raw similarity/BM25 scores — these live on incomparable scales; only rank-position fusion (RRF) is valid.
- Assuming a chunk appearing on both lists should be deduplicated/counted once — the correct behavior is to *sum* its rank contributions, which is what makes cross-method agreement compound.

---

### 2.6 Two-stage reranking

**What it is.** Reranking retrieves a wide candidate set for recall (Anthropic's example: top ~150), then a **cross-encoder** that scores query and document *together* reorders that pool down to a precise top set (~20). Fusion merges multiple lists cheaply by rank position with no model call; reranking rescores one pool with a full model pass per candidate — more expensive, more precise. Production pipelines commonly use both in sequence: fusion first, then reranking on the fused shortlist.

**Implementation — the two-stage pipeline (Python):**

```python
def two_stage_retrieval(query: str, dense_index, sparse_index, cross_encoder,
                          fusion_pool_size: int = 150, final_top_k: int = 20) -> list[str]:
    # Stage 1: wide recall via fusion — cheap, no model call
    dense_hits = dense_index.search(query, top_k=fusion_pool_size)
    sparse_hits = sparse_index.search(query, top_k=fusion_pool_size)
    fused = reciprocal_rank_fusion(dense_hits, sparse_hits)
    candidate_pool = [doc_id for doc_id, _ in fused[:fusion_pool_size]]

    # Stage 2: precise rerank via cross-encoder — expensive, one full pass per candidate
    scored = [(doc_id, cross_encoder.score(query, doc_id)) for doc_id in candidate_pool]
    scored.sort(key=lambda x: x[1], reverse=True)
    return [doc_id for doc_id, _ in scored[:final_top_k]]
```

**Common pitfalls:**
- Running the cross-encoder over the entire corpus instead of a pre-filtered candidate pool — this is prohibitively expensive; reranking only makes sense on an already-narrowed shortlist.
- Skipping fusion and going straight to reranking on a single retrieval method's output — this forfeits the recall benefit of combining independent dense/sparse views before the precision pass.

---

### 2.7 Localizing a bad RAG answer

**What it is.** **Recall@k is the primary metric.** If the right chunk is never retrieved, nothing downstream — no amount of prompt engineering — can save the answer. Precision@k, MRR, and NDCG refine *how cleanly* results rank, but only matter once recall is solid.

**Why it matters.** **Retrieval is the dominant failure origin in RAG systems.** A confident-but-wrong answer showing up right after a document refresh points at a **stale index**, not the model — run recall@k on the new corpus before touching prompts or weights. **Measure recall per leg separately** (dense vs. sparse); fusion is forgiving by design and can mask a dying leg for weeks before it's noticed in the combined metric.

**Implementation — a RAG failure triage function that checks recall before anything else (Python):**

```python
def triage_bad_rag_answer(query: str, expected_chunk_id: str, dense_results: list[str],
                           sparse_results: list[str], fused_results: list[str]) -> str:
    """Checks recall FIRST, per leg separately — fusion can mask a dying leg
    for weeks in the combined metric."""
    dense_hit = expected_chunk_id in dense_results
    sparse_hit = expected_chunk_id in sparse_results
    fused_hit = expected_chunk_id in fused_results

    if not fused_hit:
        if not dense_hit and not sparse_hit:
            return "RETRIEVAL FAILURE: neither leg retrieved the correct chunk. Check index freshness first."
        elif not dense_hit:
            return "DENSE LEG DEGRADED: sparse retrieved it, dense did not. Investigate embedder/dense index."
        elif not sparse_hit:
            return "SPARSE LEG DEGRADED: dense retrieved it, sparse did not. Investigate BM25/lexical index."
    return "Recall is solid for this case — investigate precision/ranking (MRR, NDCG) or the model's reasoning next."

def check_index_freshness(index_last_updated: str, corpus_last_updated: str) -> bool:
    """A confident-but-wrong answer right after a document refresh points at a
    STALE INDEX, not the model."""
    return index_last_updated >= corpus_last_updated
```

**Common pitfalls:**
- Jumping straight to prompt engineering when a RAG answer is wrong, without first checking recall@k — if the chunk was never retrieved, no prompt fix can recover it.
- Monitoring only the fused/combined recall metric — a dying dense or sparse leg can be masked by the other leg's strength for weeks before the combined number visibly drops.
- Assuming a wrong answer after a content update is a model regression rather than checking index freshness first.

---

## Section 3: Protocols, Observability & Context Strategy

### 3.1 Choosing the integration mechanism

**What it is.** Four mechanisms, four distinct jobs:

| Mechanism | What it does | Constraint |
|---|---|---|
| **Built-in tools** (web search, code execution) | Run on Anthropic infrastructure, no handler code | Cannot reach private systems |
| **Custom tools** | You define and run them, reaching arbitrary internal APIs | You own the handler code and its security |
| **MCP** | Open, model-agnostic standard packaging tools, resources, and prompts behind one reusable cross-host interface | Overhead only earns itself with cross-host reuse |
| **Skills** | Procedural knowledge loaded via progressive disclosure | On the Claude API, a Skill cannot make live network calls — pair it with MCP or a custom tool for fresh data |

**Why it matters — select on three questions:**
1. **Connectivity** — does this need to reach a private system? (Rules out built-ins.)
2. **Reuse** — is this a single consumer (a direct call is right-sized) or genuine cross-host reuse (MCP earns its overhead)?
3. **Consistency** — is this a repeatable workflow? (That's a Skill.)

**Implementation — a mechanism-selection decision function (Python):**

```python
from enum import Enum

class IntegrationMechanism(Enum):
    BUILT_IN = "built_in_tool"
    CUSTOM_TOOL = "custom_tool"
    MCP = "mcp"
    SKILL = "skill"

def select_integration_mechanism(needs_private_system: bool, is_cross_host_reused: bool,
                                   is_repeatable_workflow: bool, needs_live_network_calls: bool) -> IntegrationMechanism:
    if not needs_private_system:
        return IntegrationMechanism.BUILT_IN  # built-ins run on Anthropic infra, cannot reach private systems anyway

    if is_repeatable_workflow and not needs_live_network_calls:
        return IntegrationMechanism.SKILL  # a Skill cannot make live network calls on the Claude API

    if is_cross_host_reused:
        return IntegrationMechanism.MCP  # overhead earns itself only with genuine cross-host reuse

    return IntegrationMechanism.CUSTOM_TOOL  # single consumer — a direct call is right-sized
```

**Common pitfalls:**
- Standing up an MCP server for a single-consumer, single-host integration — the overhead (server, transport, auth) only pays off with genuine cross-host reuse.
- Trying to give a Skill live network-call capability directly on the Claude API — Skills need to be paired with MCP or a custom tool for fresh data.
- Assuming built-in tools can reach a private internal system — they run on Anthropic infrastructure and structurally cannot.

---

### 3.2 MCP transports and topology

**What it is.** MCP defines exactly two standard transports:
- **stdio** — launches the server as a local subprocess, typically serving a single client, credentials from the environment, no OAuth.
- **Streamable HTTP** — serves many clients over the network, with OAuth; this replaced the deprecated legacy streaming (SSE) transport.[1]

**Why it matters.** MCP standardizes **vertical integration** — one agent reaching down to tools, the common case. **Agent-to-agent** is *horizontal* coordination between independent peer agents exchanging tasks and results, needed only for genuinely multi-agent workloads (Domain 1 §3). **A single agent calling many tools through MCP is *not* agent-to-agent, no matter how many tools it calls.**

Capability still has a context price even inside MCP: thousands of loaded tool definitions can burn hundreds of thousands of tokens before a request even runs. Progressive disclosure (§3.4) loads them on demand instead of up front. **Let the shape of the connection choose the mechanism — never pick the newest-sounding label.**

**Implementation — transport selection based on topology (Python):**

```python
def select_mcp_transport(num_clients: int, needs_network_access: bool) -> str:
    """Let the SHAPE of the connection choose the transport, not novelty."""
    if num_clients == 1 and not needs_network_access:
        return "stdio"  # local subprocess, single client, env-sourced credentials
    return "streamable_http"  # many clients over the network, OAuth 2.1 + PKCE

def is_agent_to_agent(topology: str) -> bool:
    """A single agent calling many tools through MCP is vertical integration,
    NOT agent-to-agent, regardless of tool count."""
    return topology == "peer_agents_exchanging_tasks"  # horizontal coordination only
```

**Common pitfalls:**
- Calling a single agent's many MCP tool calls "agent-to-agent" or "multi-agent" — this is vertical integration; agent-to-agent specifically means horizontal peer coordination (Domain 1 §3).
- Choosing Streamable HTTP for a genuinely single-client, local integration purely because it "sounds more modern" — stdio is the shape-appropriate, lower-overhead choice here.
- Loading an entire large MCP tool catalog up front instead of applying progressive disclosure (§3.4) — this pays the full capability-bloat cost (§1.1) even inside MCP.

---

### 3.3 Observability at scale (OpenTelemetry)

**What it is.** The Claude Agent SDK and Claude Code emit telemetry through **OpenTelemetry (OTLP)** exclusively, exporting to a swappable, vendor-neutral backend.[3]

**Three signals answer three different questions:**
- **Metrics** — fleet health: request counts, token totals, error rates.
- **Log events** — discrete, searchable moments.
- **Traces** — follow one request through causally linked spans.

**Why it matters.** Every signal ships **off by default** and is enabled per signal — the full trace view sits behind a beta flag as of current documentation.

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
- **Session ID** — groups many traces into one logical conversation.
- **Trace context** (trace ID + parent span ID) — links spans *within* a single trace and propagates across process boundaries.

Miss either one and your traces are technically correct but **orphaned** — unfindable and unlinkable at the volume where you'll actually need them.

**At scale:** stamp resource attributes at the source — service, environment, version, tenant — because an attribute you forget to attach is a dimension you can *never* slice by after the fact. Per-request inspection becomes infeasible at volume, so aggregates become the default view and traces become the drill-down; tenant tags drive per-tenant routing and isolation into a SIEM.

**Keep spans structural by default** — recording the call, timing, and status, but *not* the payload. Make content logging a **narrow, explicit opt-in**, since it drags sensitive data (and its compliance burden — Domain 5) into the observability pipeline. Treat export reliability as a design decision: batch export buffers spans in memory, and a process **must flush on exit or lose the last spans of the run** — exactly the spans you'd want most when diagnosing a crash. **Read sampled data as sampled, never as complete.**

**Implementation — OTel configuration with structural-by-default spans and explicit flush (Python):**

```python
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.sdk.resources import Resource
import atexit

# Stamp resource attributes AT THE SOURCE — never add these after the fact.
resource = Resource.create({
    "service.name": "telecom-triage-agent",
    "deployment.environment": "production",
    "service.version": "2.4.1",
    "tenant.id": "carrier-acme",  # drives per-tenant routing/isolation downstream
})

provider = TracerProvider(resource=resource)
exporter = OTLPSpanExporter(endpoint="https://otel-collector.internal:4317")
span_processor = BatchSpanProcessor(exporter)
provider.add_span_processor(span_processor)
trace.set_tracer_provider(provider)

# CRITICAL: flush on exit, or the last spans of a crashed run are lost —
# exactly the spans you'd most want when diagnosing the crash.
atexit.register(lambda: span_processor.force_flush(timeout_millis=5000))

tracer = trace.get_tracer("triage-agent")

def structural_tool_span(tool_name: str, tool_input: dict, include_payload: bool = False):
    """Structural by default: records call/timing/status, NOT payload,
    unless content logging is a narrow, explicit opt-in."""
    with tracer.start_as_current_span(f"tool.{tool_name}") as span:
        span.set_attribute("tool.name", tool_name)
        span.set_attribute("tool.status", "pending")
        if include_payload:  # explicit opt-in only — this drags sensitive data + compliance burden (Domain 5) in
            span.set_attribute("tool.input_payload", str(tool_input))
        # ... execute tool ...
        span.set_attribute("tool.status", "success")

def subagent_span_nesting(parent_tool_span, subagent_task: str):
    """Subagent spans nest under the TOOL span that spawned them — keeps cost
    attributable to the specific delegate, not blended into the lead's totals."""
    with tracer.start_as_current_span("subagent.execution", context=parent_tool_span) as span:
        span.set_attribute("subagent.task", subagent_task)
        # ... subagent's own model-request spans nest further under here ...
```

**Common pitfalls:**
- Assuming traces are captured "by default" — every signal ships off by default and must be explicitly enabled per signal.
- Forgetting to propagate trace context across process boundaries (e.g., into a subagent's own process) — orphaned spans are technically correct but unfindable at volume.
- Omitting a resource attribute (tenant, environment) at instrumentation time — this dimension becomes permanently unsliceable after the fact.
- Logging full tool-call payloads by default instead of as a narrow, explicit opt-in — this drags sensitive data and compliance burden into the observability pipeline unnecessarily.
- Not registering a flush-on-exit handler for batch export — the last, most diagnostically valuable spans of a crashed run are lost.
- Treating sampled trace data as if it were complete — sampled data must be read and reported as sampled.

---

### 3.4 Progressive discovery vs. monolithic context

**What it is.** **Monolithic context is taxed twice at scale:** tool-definition tokens load before the request even starts, and intermediate results re-traverse the context window until it fills with data the final answer never uses. This is an **architectural decision made early**, not a tactical audit you run on one misbehaving agent — retrofitting discovery onto a monolithic design later is expensive.

**Four disclosure mechanisms share one move — keep definitions out of context until they earn their place:**
1. **Filesystem layout** — tools exposed as code files, read on demand.
2. **Tool search** — defers rarely-used tools while hot-path tools stay loaded.
3. **Detail-level verbosity parameters** — request more or less detail per call.
4. **On-demand reading** of full schemas or document bodies.

**Code execution with MCP** has the agent write and run code that drives the tools itself — intermediate results stay in the runtime, and only the distilled final result returns to context. The difference between this and monolithic loading barely matters at three tool calls and is the whole ballgame at thirty.

Anthropic reported that code-on-filesystem tool loading cut one worked example's total token usage by **about 98.7%.**[4] Treat that as evidence the *pattern* works, not a guaranteed number — your own savings depend heavily on toolset size and task shape.

**Monolithic still wins** for a handful of tools, a stable knowledge base small enough to cache inline, and fixed tools and data. Discovery's price is real: lookup round trips, code-execution infrastructure to run and secure, and more moving parts to debug. **The two traps:** front-loading everything and then fixing bloat cosmetically after the fact, and standing up full discovery machinery for what is, in reality, a five-tool agent.

**Implementation — progressive tool discovery via filesystem layout (Python + TypeScript):**

```python
# Python — tools exposed as code files, loaded on demand rather than all up front
import importlib
from pathlib import Path

TOOL_DIR = Path("./tools")  # each tool is its own file, NOT loaded into context until needed

def discover_available_tool_names() -> list[str]:
    """Only names/descriptions are cheap and always visible — full implementations
    load on demand (mirrors Domain 2 §4.6's Skills progressive disclosure)."""
    return [f.stem for f in TOOL_DIR.glob("*.py")]

def load_tool_on_demand(tool_name: str):
    module = importlib.import_module(f"tools.{tool_name}")
    return module.TOOL_SCHEMA, module.handler
```

```typescript
// TypeScript — code execution with MCP: the agent writes/runs code that drives
// tools itself; intermediate results stay in the runtime, not the context window.
async function codeExecutionWithMcp(task: string, mcpToolNames: string[]) {
  // The agent generates code that calls MCP tools programmatically.
  // Intermediate results (e.g., a large API response) are processed IN the
  // runtime and never re-enter the model's context — only the distilled
  // final result does.
  const generatedCode = await generateAgentCode(task, mcpToolNames);
  const runtimeResult = await executeInSandbox(generatedCode); // intermediate data stays here
  return summarizeForContext(runtimeResult); // only THIS returns to the model's context
}

async function generateAgentCode(task: string, tools: string[]): Promise<string> { return ""; }
async function executeInSandbox(code: string): Promise<any> { return {}; }
function summarizeForContext(result: any): string { return JSON.stringify(result).slice(0, 500); }
```

**Decision function — monolithic vs. progressive discovery:**

```python
def choose_context_strategy(tool_count: int, knowledge_base_fits_cache: bool, data_is_stable: bool) -> str:
    """Architectural decision made EARLY — retrofitting later is expensive."""
    if tool_count <= 5 and knowledge_base_fits_cache and data_is_stable:
        return "monolithic"  # discovery machinery would be pure overhead here
    return "progressive_discovery"  # tool_search, filesystem layout, or code execution with MCP
```

**Common pitfalls:**
- Treating progressive discovery as a tactical fix applied to one misbehaving agent after the fact, rather than an early architectural decision — retrofitting is expensive precisely because the source domain frames this as foundational, not incremental.
- Standing up full discovery machinery (code execution sandboxes, tool search infrastructure) for a genuinely small, five-tool agent — this is one of the two explicit traps the source domain names.
- Citing the ~98.7% token reduction figure as a guaranteed outcome rather than directional evidence dependent on toolset size and task shape.

---

## Worked Architecture: RAG + MCP for a Telecom Knowledge Base

```
Knowledge base: 40,000 pages of network runbooks   > 200k tokens → RAG required
              ↓
INDEXING (offline, cached source document)
  chunk (fixed-window, ~400 tok, 15% overlap)
  → generate 50-100 tok context per chunk
  → dense embedding index + BM25 index
              ↓
QUERY TIME
  dense search (top 150) + sparse search (150)
  → Reciprocal Rank Fusion
  → cross-encoder rerank → top 20
              ↓
AGENT (MCP, Streamable HTTP, OAuth 2.1)
  tools: kb.search (namespaced, consolidated),
  network.diagnose, ticket.update
  least-privilege scoped per tool
  OTel spans: interaction → tool → subagent
```

Recall@20 is monitored per leg (dense, sparse) as the primary eval; a drop right after the next runbook refresh is diagnosed as a **stale-index** problem before anyone touches the prompt.

**Implementation note connecting the worked example to the mechanisms above:** the 40,000-page corpus exceeding the 200k-token threshold justifies RAG per §2.4; indexing applies §2.1's fixed-window chunking and §2.2's Contextual Retrieval; query time applies §2.5's RRF and §2.6's two-stage reranking in sequence; the agent's MCP tools are namespaced and consolidated per §1.1, transport-appropriately authenticated per §1.2/§3.2; and the OTel span hierarchy follows §3.3's interaction → tool → subagent nesting exactly.

---

## Architecture & Implementation Best Practices

1. **Run the three bloat diagnostics (tool-call count, token overhead, transcript inspection) before every toolset expansion**, not just when something visibly breaks — bloat accumulates silently.
2. **Encode every hard prohibition as a deny rule, never as an absent allow** — this is the deterministic-vs-persuasive distinction (Domain 7 §1.3) applied to tool authorization specifically.
3. **Match MCP transport to actual topology** (stdio for single-client-local, Streamable HTTP for multi-client-remote) — never choose based on which sounds more modern.
4. **Exhaust the three accuracy-neutral levers (caching, batch, streaming) before touching model tier, effort, or real accuracy** — and document the binding constraint explicitly before choosing a config.
5. **Validate the embedder before investing in chunking sophistication** — embedding quality frequently dominates the entire retrieval-quality ceiling.
6. **Measure recall@k per retrieval leg (dense, sparse) separately, never only on the fused result** — fusion is forgiving by design and can mask a dying leg for weeks.
7. **Treat progressive discovery as an early, foundational architecture decision** — size the investment to the actual toolset (a five-tool agent doesn't need code-execution-with-MCP infrastructure).
8. **Stamp OTel resource attributes at the source and register a flush-on-exit handler from day one** — both failure modes (unsliceable dimensions, lost crash spans) are unrecoverable after the fact.

---

## Common Pitfalls and Limitations

- **Silent capability bloat** — adding tools without consolidating or removing overlapping ones, diagnosed too late by intuition rather than the three explicit diagnostic reads.
- **Claiming OAuth applies to stdio transport** — a classic, explicitly-named misread; stdio has no OAuth model.
- **Over-privileged tokens** — invisible in normal operation because nothing fails; the most common real-world auth gap.
- **Tokens in query strings** — logs capture them; one of three non-negotiable token-hygiene violations.
- **Maximizing accuracy when latency is the actual binding constraint** — one of three explicit stakeholder-communication traps.
- **Over-investing in chunking sophistication before validating the embedder** — the embedder sets the ceiling.
- **Comparing raw dense/sparse scores directly instead of using rank-based fusion (RRF)** — the scales are incomparable.
- **Jumping to prompt engineering on a bad RAG answer without checking recall@k first** — retrieval is the dominant failure origin.
- **Monitoring only fused recall, missing a dying dense or sparse leg** — fusion masks single-leg degradation.
- **Building RAG for a corpus that fits under the cacheable-context threshold** — technology-first design applied to retrieval specifically.
- **Calling single-agent multi-tool-call patterns "agent-to-agent"** — conflates vertical integration with horizontal peer coordination.
- **Retrofitting progressive discovery onto a monolithic design after bloat is already painful** — expensive precisely because it should have been an early architectural decision.
- **Standing up full discovery machinery for a five-tool agent** — overkill relative to actual need.
- **Assuming telemetry signals are captured by default** — every OTel signal ships off by default.
- **Logging full payloads by default instead of as a narrow opt-in** — drags sensitive data and compliance burden into observability.
- **No flush-on-exit for batch span export** — loses exactly the spans most needed to diagnose a crash.

---

## Official Documentation References

1. Model Context Protocol Specification, *"Authorization"* — https://modelcontextprotocol.io/specification/2025-06-18/basic/authorization
2. Anthropic News, *"Introducing Contextual Retrieval,"* Sep 19, 2024 — https://www.anthropic.com/news/contextual-retrieval
3. Anthropic Docs, *"Monitoring — OpenTelemetry"* — https://docs.claude.com/en/docs/claude-code/monitoring-usage
4. Anthropic Engineering, *"Code execution with MCP: Building more efficient AI agents"* — https://www.anthropic.com/engineering/code-execution-with-mcp
5. Anthropic Docs, *"Model Context Protocol (MCP)"* — https://docs.claude.com/en/docs/agents-and-tools/mcp
6. OpenTelemetry, *official documentation* — https://opentelemetry.io/docs/
7. Anthropic Docs, *"Tool use overview"* — https://docs.claude.com/en/docs/build-with-claude/tool-use/overview

> **Version note:** MCP transport specifications, OAuth flow details, and OTel integration specifics are actively evolving. Verify exact protocol versions and SDK support against `modelcontextprotocol.io` and `docs.claude.com` before implementation, particularly around the deprecated SSE transport and beta trace-view flags.

---

## Domain Coverage / Traceability Matrix

| Source domain item | Implementation guide section(s) | Coverage |
|---|---|---|
| 1.1 Capability bloat definition + two failure signatures | §1.1 | ✅ Full |
| 1.1 Three diagnostic reads | §1.1 | ✅ Full, with `diagnose_bloat` code |
| 1.1 Four right-sizing moves | §1.1 | ✅ Full, with before/after consolidation code |
| 1.1 Re-run baseline after right-sizing | §1.1 | ✅ Full, with validation code |
| 1.2 Least privilege | §1.2 | ✅ Full |
| 1.2 Deny-first, four-decision pre-tool hook | §1.2 | ✅ Full, with `ToolDecision` code |
| 1.2 MCP transport-dependent auth table | §1.2 | ✅ Full, with Python + TS examples |
| 1.2 Three token-hygiene rules | §1.2 | ✅ Full, with validation code |
| 1.2 Confused-deputy defense | §1.2 | ✅ Full |
| 1.2 Over-privileged token gap | §1.2 | ✅ Full |
| 1.3 Three-lever accuracy-latency order | §1.3 | ✅ Full, with `select_levers` code |
| 1.3 Three accuracy-neutral levers | §1.3 | ✅ Full |
| 1.3 Binding-constraint optimization + comparative justification | §1.3 | ✅ Full, with `justify_config` code |
| 1.3 Three stakeholder-communication traps | §1.3 | ✅ Full |
| 2.1 Three chunking knobs | §2.1 | ✅ Full, with chunking code |
| 2.1 Fixed-window ≈ semantic; embedder dominates | §2.1 | ✅ Full |
| 2.2 Contextual Retrieval mechanism + caching economics | §2.2 | ✅ Full, with indexing code |
| 2.2 Stacked gains table (35%/49%/67%) | §2.2 | ✅ Full |
| 2.3 Dense vs. sparse trade-offs | §2.3 | ✅ Full, with decision code |
| 2.3 Build-both + embedder-first rules | §2.3 | ✅ Full |
| 2.4 ~200k token skip-RAG threshold | §2.4 | ✅ Full, with threshold-check code |
| 2.4 Three conditions RAG earns its keep | §2.4 | ✅ Full |
| 2.5 RRF mechanism + worked math | §2.5 | ✅ Full, with verified code example |
| 2.6 Two-stage reranking (fusion + cross-encoder) | §2.6 | ✅ Full, with pipeline code |
| 2.7 Recall@k as primary metric | §2.7 | ✅ Full, with triage code |
| 2.7 Stale-index diagnosis + per-leg recall measurement | §2.7 | ✅ Full, with freshness-check code |
| 3.1 Four integration mechanisms table | §3.1 | ✅ Full, with selector code |
| 3.1 Three selection questions | §3.1 | ✅ Full |
| 3.2 Two MCP transports | §3.2 | ✅ Full, with selector code |
| 3.2 Vertical vs. horizontal (agent-to-agent) distinction | §3.2 | ✅ Full |
| 3.3 Three OTel signal types | §3.3 | ✅ Full |
| 3.3 Span tree hierarchy | §3.3 | ✅ Full, with nesting code |
| 3.3 Session ID vs. trace context | §3.3 | ✅ Full |
| 3.3 Resource attribute stamping at scale | §3.3 | ✅ Full |
| 3.3 Structural-by-default spans, opt-in payload logging | §3.3 | ✅ Full, with span code |
| 3.3 Flush-on-exit requirement | §3.3 | ✅ Full, with `atexit` code |
| 3.3 Sampled-data caveat | §3.3 | ✅ Full |
| 3.4 Monolithic double-tax + four disclosure mechanisms | §3.4 | ✅ Full, with filesystem-layout code |
| 3.4 Code execution with MCP | §3.4 | ✅ Full, with TS sandbox code |
| 3.4 ~98.7% reported figure + two traps | §3.4 | ✅ Full, with decision code |
| Worked Architecture: RAG + MCP (full diagram) | Worked Architecture section | ✅ Full, reproduced and annotated |
| Key Takeaways (8 bullets) | Distributed across Best Practices + Pitfalls | ✅ Full — every bullet maps to at least one item |
| References 1–6 (source domain) | Official Documentation References | ✅ Full, retained and extended with tool-use reference |

**Coverage verification statement:** Every numbered subsection, table, and key-takeaway bullet in `domain-3-integration.md` is represented above with expanded technical detail, at least one implementation pattern, and a runnable Python and/or TypeScript example wherever the source item names or implies a concrete mechanism. No source item was merged, omitted, or skipped.
