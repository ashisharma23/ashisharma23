# Domain 2: Claude Models, Prompting & Context Engineering
**Exam weight: 13% | Claude Certified Architect — Professional**

> Model selection, prompt structure, and context management are the three levers an architect pulls *before* touching architecture. Get these wrong and even the right pattern from Domain 1 underperforms. Get them right and you often don't need the fancier pattern at all.

---

## 1. Model Selection

### 1.1 Think in tiers, not names

| Tier | Model family | Use case |
|---|---|---|
| Fast, economical | Haiku 4.5 | High-volume, bounded work: classification, extraction, routing |
| Scale default | Sonnet 5 | Sized to carry most production traffic |
| Flagship | Opus 4.8 | Complex, multi-step agentic work |
| Frontier | Fable 5.1 / Mythos 5.1 | Highest per-token price; reached only when the flagship visibly falls short |

A dateless model name like `claude-opus-4-8` is a **fixed snapshot with stable weights**, not an auto-upgrading pointer — though the serving layer around it (routing, safety classifiers) can still drift over time. **Pin explicit model names in production** for reproducible evaluations, and build migration windows into your roadmap ahead of any deprecation date, since calls to a retired model return hard errors, not silent fallbacks.

### 1.2 Two starting strategies attack opposite risks

- **Start cheap, then upgrade:** for high-volume, bounded, latency-sensitive work. Upgrade only the *slices* that miss the quality bar, not the whole pipeline — a lightweight classifier can route the hard 10% up to a stronger tier while the easy 90% stays cheap.
- **Start capable, then optimize:** for ambiguous, multi-step, or high-cost-of-error tasks. Prove the task is solvable at all on the flagship first; only then look for a cheaper tier that clears the same bar.

Architects who default to "start cheap everywhere" often burn more engineering time chasing prompt fixes for a model that was never going to clear a genuinely hard reasoning bar than they would have spent validating on the flagship first.

### 1.3 Tune effort before switching models

Before reaching for a pricier tier, tune the **effort** parameter — five levels from `low` through `xhigh` to `max`, with `high` as the default when omitted. This governs how much the model reasons before responding, and is frequently a better lever than a full model swap because it's cheaper to test and instantly reversible.

**Fast mode** is a separate, orthogonal lever — up to 2.5× output tokens/second on Opus-tier models at premium pricing (a research preview at time of writing). It's justified only when a hard latency target is the actual design constraint, not as a default "make it feel snappier" toggle.

### 1.4 Routing at scale

At production scale, don't route every request through your most capable model by hand — build the routing itself:
- A **lightweight classifier** sends simple work down-tier and genuinely hard requests up-tier.
- **Narrow subagent workers** run on a fast model while the **orchestrator** stays on a capable one (see Domain 1, §3).

**Stack the cost levers**, don't pick one:
1. **Batch API** — roughly half price for latency-insensitive work (see Domain 3/4 for details)
2. **Prompt caching** — for stable prefixes reused across calls
3. **Tokenizer re-baseline on migration** — a newer model generation can count the *same text* as roughly 30% more tokens than the prior generation, which silently changes your cost model if you don't re-measure after a migration

---

## 2. Prompt & Guardrail Design

### 2.1 Placement is the core rule

| Lives in system prompt | Lives in user turn |
|---|---|
| Persistent behavior, role, standing constraints | Ephemeral, per-request data |
| The output contract | Customer-specific fields, timestamps, session data |

Baking request-specific data into the static scaffold **silently kills your cache** (see §4 below) and breaks regression testing, because every "identical" test case now has a different byte-for-byte prompt.

### 2.2 Layer the system prompt deliberately

Anthropic's own prompt-engineering guidance recommends this order, and it maps directly to how attention and instruction-following behave in practice:<sup>[1]</sup>

1. **Role and task** first — tell the model what it is before what to do
2. **Context and motivation** in the middle — Claude follows instructions measurably better when it understands *why*, not just *what*
3. **Numbered, sequential steps** for genuinely multi-part work
4. **Output format + XML delimiters** at the end

Prefer **positive instructions** ("Respond only in valid JSON matching this schema") over lists of prohibitions ("Don't add commentary, don't use markdown, don't..."). A long "don't" list is harder for the model to satisfy jointly and harder for you to test.

### 2.3 Guardrails: prompt-based vs. programmatic

This is one of the highest-value distinctions on the exam and in production risk management:

| | Prompt-based guardrail | Programmatic guardrail |
|---|---|---|
| Nature | Probabilistic | Deterministic |
| What it does | *Lowers* the risk of a bad output | *Bounds* it |
| Where it belongs | Style, tone, format nudges | Anything with financial, legal, or safety consequences |

**Any constraint that carries real-world consequences belongs in code**, not in the system prompt, with high-stakes actions gated behind human review — because risk compounds across enough calls even at a low per-call failure rate (see Domain 5, §1 for guardrail-layering in depth). A cheap, fast model running as a harmlessness pre-screen catches clearly harmful inputs early and should **fail closed** when uncertain.

### 2.4 Reading the stop reason before touching the response body

- `end_turn` — healthy completion
- `max_tokens` — **truncation**, not a wrong answer; don't score it as a quality failure
- `refusal` — a policy decline. `stop_reason` returns `refusal`, and stop *details* (category plus a short explanation) populate only on refusal — they're empty on every other stop reason.

Route refusals to a dedicated handler with a fixed, friendly message and a safe next step. **Never echo the model's internal refusal explanation directly to the end user** — it's diagnostic detail, not customer-facing copy.

### 2.5 Prompt injection: direct vs. indirect

- **Direct injection** — the adversary types it straight into user input.
- **Indirect injection** — the payload is hidden in a document, web page, or tool result the agent retrieves. This is the **dominant risk for anything that browses, reads files, or calls external tools**, because the untrusted content arrives disguised as legitimate data.

**Defend structurally, with four stacked layers** (not one clever prompt):
1. Isolate untrusted content inside tool-result blocks
2. JSON-encode or XML-fence payloads so they cannot escape their data context
3. Grant every tool least privilege to shrink the blast radius of a successful injection
4. Verify high-stakes actions in code, not by trusting the model's self-report

---

## 3. Prompting Techniques: Zero-Shot, Few-Shot, Chain-of-Thought

### 3.1 Zero-shot is the default on current models

Strong instruction-following on current Claude models clears most well-specified tasks with **no examples at all.** Every example you add costs input tokens on *every* call forever. Only add few-shot when plain instructions keep demonstrably missing on your eval set — don't add examples pre-emptively "to be safe."

### 3.2 Few-shot buys three specific things

Format lock, call-to-call consistency, and edge-case coverage — nothing else. Use **three to five diverse, consistently structured examples**, wrapped in example tags.

**Classic failure modes:**
- **All-happy-path examples** — the model never sees what a hard case looks like, so it doesn't generalize to one
- **Inconsistent shapes** — examples that don't share a structural pattern teach the model the wrong invariant
- **Near-duplicates** — three examples that are really one example three times, buying no additional coverage for the token cost

Quality and coverage beat raw count every time.

### 3.3 Extended thinking has replaced hand-written CoT scripts

Hand-written "think step by step" scripts are largely obsolete on current models. Extended thinking is an API feature that produces **separate reasoning blocks** natively — and a rigid, hand-authored script can actually *constrain* a better internal reasoning path the model would otherwise take.

On the current Opus-tier model, **adaptive thinking is the only supported mode** — it's off unless explicitly enabled, and legacy `budget_tokens`-style parameters are rejected outright by the API.

**Billing note:** reasoning tokens are billed whether or not you display them. Setting the display parameter to `summarized` changes *visibility only* — it never changes billing. Surface the full trace when you need it for debugging, auditing, or building user trust; otherwise summarize to keep the response clean. Match reasoning depth to the cost of a wrong answer — a customer-facing FAQ bot rarely needs `xhigh` effort; a contract-risk classifier might.

### 3.4 Refine prompts like an engineer, not an artist

1. Test against representative cases (not the three examples you wrote the prompt around)
2. Diagnose *before* touching anything — read the actual failure, don't guess
3. Change exactly one thing
4. Re-run the **full** test set to catch regressions the fix introduced elsewhere

Externalize self-correction into a separate call when you need logging, gating, or a cheaper checker model to grade the primary model's output. Every self-critique loop needs an **explicit stopping condition** — "keep revising until good" is not a stopping condition, it's an infinite loop waiting to happen.

---

## 4. Context Window Optimization & Prompt Reuse

### 4.1 Context rot is real and directional

As token count climbs, recall of any single fact *drops* — because *n* tokens create roughly *n²* pairwise attention links competing for a fixed attention budget. Padding a prompt with marginal, "might be useful" context doesn't just cost money; it **actively lowers quality** on the facts that matter.

**The guiding principle: informative yet tight** — the smallest set of high-signal tokens that does the job. The model does not prune itself. Curation is an architecture decision you own, not something that happens automatically as context grows.

### 4.2 Compaction vs. tool-result clearing

| | Compaction | Tool-result clearing |
|---|---|---|
| Mechanism | A model call summarizes older turns | Free, mechanical, server-side edit |
| Cost | Real inference cost | Free |
| What survives | Key facts and decisions | Placeholders swap in for old results |
| Default trigger | 150,000 input tokens (documented floor: 50,000) | 100,000 tokens |
| Preserves | — | The 3 most recent tool-use pairs |
| Best for | Dialogue and reasoning growth | Large, re-fetchable tool-result bloat |

**Default to free clearing** for big re-fetchable tool output; pay for compaction only when the growth is genuinely dialogue and reasoning, not restated data you can pull again cheaply.

### 4.3 Memory and subagent isolation

The **memory tool** persists state in files outside the context window, across sessions — structured note-taking the model writes to deliberately. Write key state to memory **before** clearing or compaction erases it; once it's gone from context, it's gone unless it was externalized. When the valuable state already lives on disk (a git log, a progress file), starting a fresh window often beats compacting an old one in place.

**Subagents isolate context by design:** messy exploration burns tokens in the subagent's own window, and only a distilled 1,000–2,000 token summary returns to the lead. This caps cost, contains failures, and limits leakage — but delegating genuinely simple, single-step work to a subagent just adds coordination overhead for nothing (see Domain 1, §3).

### 4.4 Token accounting you actually need to track

`input_tokens` reported by the API counts **only the uncached remainder.** True prompt size is:

```
true_prompt_tokens = input_tokens + cache_creation_input_tokens + cache_read_input_tokens
```

Context awareness — a model tracking its own remaining budget — is scoped to **named models only** (e.g., Sonnet 5, Sonnet 4.6, Sonnet 4.5, Haiku 4.5). Never assume every model in your fleet has this capability; verify per model before relying on it in an agentic loop's stopping logic.

### 4.5 Prompt caching mechanics

Caching is a **strict byte-identical prefix match** over content rendered in a fixed order: **tools, then system, then messages.** A single changed character anywhere invalidates everything *after* it in the render order — so stable content leads, volatile per-request data trails, and a cache breakpoint marks the seam between them.

| | Reads | 5-min write | 1-hour write |
|---|---|---|---|
| Relative cost | ~0.1× base input | ~1.25× base | ~2× base |
| Pays off after | — | 1 read | 2+ reads |

Caching is a bet that cheap reads repay the write premium — match cache lifetime to how far apart reuses actually land in your traffic pattern. A prefix called only once always loses the bet and should not be cached.

**Debugging a silent cache miss:** verify with usage fields, never assumptions. High `cache_read_input_tokens` = hit. Zero reads across expected matches = silent miss, usually caused by:
- A timestamp or session ID baked into the "static" system prompt
- Shifting tool order between calls
- Unsorted JSON keys in a serialized object
- A prefix below the model's minimum cacheable size (512–4,096 tokens depending on model; Haiku sits at the highest floor, and this fails **silently, with no error**)

Debug by diffing consecutive requests byte-for-byte to find the first differing character.

### 4.6 Modular prompts and Skills — two different kinds of reuse

**Modular prompts** compose role, policy, format, and task blocks. The first three stay stable up front — giving isolated testability, fix-once consistency, and a naturally cache-friendly prefix — while only the task block and user data vary at the tail. Modularity reuses *authored content*; caching reuses *computation*. Never inject volatile data into the fixed skeleton, or you defeat both.

**Skills** package a whole capability — instructions plus optional bundled resources — that Claude invokes autonomously when a task matches its description. This differs from a subagent, which sends work *out* to an isolated context; a Skill loads *into* the current context. Skills use **progressive disclosure** in three levels:
1. Description — always in context (cheap)
2. Full instructions — loaded on invocation
3. Bundled resources — loaded only on demand

Idle Skills cost almost nothing. Scope (personal, project, plugin) sets reach, and because invocation is autonomous, **review a shared Skill's instructions and permissions before trusting it** — it will run with whatever access it's granted, without asking first.

---

## Worked Example: Prompt & Context Design for the Telecom Triage Agent

Building on Domain 1's architecture:

```
SYSTEM PROMPT (stable, cached prefix)
├── Role: "You are a network-triage assistant for [Carrier]..."
├── Context: why accurate routing matters (SLA consequences, § motivation)
├── Tool definitions (namespaced: ticket.*, network.*, customer.*)
└── Output contract: strict JSON schema, XML-delimited

USER TURN (volatile, appended per request)
├── Ticket text (customer-specific — never cached)
├── Account enrichment (from input-handling stage)
└── Cache breakpoint marks the seam here
```

- Fast-tier model (Haiku-class) handles the routing classifier — bounded, high-volume, cheap.
- Flagship-tier model (Opus-class) only engages on the agentic diagnostic branch, where multi-step reasoning earns its cost.
- Tool-result clearing (free) handles the repeated `network.signal_check()` calls that would otherwise bloat context during a long diagnostic session; compaction is reserved for genuinely long, reasoning-heavy escalation write-ups.
- Memory tool records the diagnostic timeline before any compaction, so an escalated ticket handed to a human engineer never loses the "what we already tried" history.

---

## Key Takeaways

- Pin explicit model versions in production; tune `effort` before you swap tiers.
- System prompt = stable and cacheable. User turn = volatile. Mixing the two kills your cache and your regression tests.
- Prompt-based guardrails lower risk; programmatic guardrails bound it. High-stakes actions belong in code with human review, never in prompt language alone.
- Zero-shot is the default; add few-shot only when the eval proves it's needed, with diverse, non-redundant examples.
- Context rot is real: bigger context ≠ better recall. Curate deliberately; the model won't do it for you.
- Cache matching is byte-identical and order-sensitive (tools → system → messages). Verify hits with usage fields, never assume.
- Skills load capability into context; subagents send work out to isolated context. They solve different problems.

---

## References

1. Anthropic Docs, *"Prompt engineering overview"* — https://docs.claude.com/en/docs/build-with-claude/prompt-engineering/overview
2. Anthropic Docs, *"Prompt caching"* — https://docs.claude.com/en/docs/build-with-claude/prompt-caching
3. Anthropic Docs, *"Extended thinking"* — https://docs.claude.com/en/docs/build-with-claude/extended-thinking
4. Anthropic Docs, *"Context editing and the memory tool"* — https://docs.claude.com/en/docs/build-with-claude/context-editing
5. Anthropic Docs, *"Agent Skills overview"* — https://docs.claude.com/en/docs/agents-and-tools/agent-skills/overview
6. Anthropic Docs, *"Models overview"* — https://docs.claude.com/en/docs/about-claude/models/overview
