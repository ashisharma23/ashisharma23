# Domain Implementation Reference Guide — Domain 2: Claude Models, Prompting & Context Engineering

**Source domain:** `domain-2-models-prompting-context-engineering.md` (CCAR-P, exam weight 13%)
**Guide type:** Implementation Reference (technical depth, code examples, API/config detail)
**Audience:** Solution architects and engineers making model-selection, prompting, and context-management decisions for production Claude systems.

> This guide preserves the full structure of the source domain (1:1 section mapping) and expands every item into: what it is, why it matters, the underlying Anthropic technical mechanism, implementation patterns, pitfalls, and runnable Python/TypeScript examples using the Claude API. A full traceability matrix appears at the end.

---

## Table of Contents

1. [Domain Overview](#domain-overview)
2. [Section 1: Model Selection](#section-1-model-selection)
   - 1.1 Think in tiers, not names
   - 1.2 Two starting strategies
   - 1.3 Tune effort before switching models
   - 1.4 Routing at scale
3. [Section 2: Prompt & Guardrail Design](#section-2-prompt--guardrail-design)
   - 2.1 Placement is the core rule
   - 2.2 Layer the system prompt deliberately
   - 2.3 Guardrails: prompt-based vs. programmatic
   - 2.4 Reading the stop reason
   - 2.5 Prompt injection: direct vs. indirect
4. [Section 3: Prompting Techniques](#section-3-prompting-techniques-zero-shot-few-shot-chain-of-thought)
   - 3.1 Zero-shot as default
   - 3.2 Few-shot's three specific benefits
   - 3.3 Extended thinking vs. hand-written CoT
   - 3.4 Refining prompts like an engineer
5. [Section 4: Context Window Optimization & Prompt Reuse](#section-4-context-window-optimization--prompt-reuse)
   - 4.1 Context rot
   - 4.2 Compaction vs. tool-result clearing
   - 4.3 Memory and subagent isolation
   - 4.4 Token accounting
   - 4.5 Prompt caching mechanics
   - 4.6 Modular prompts and Skills
6. [Worked Example: Prompt & Context Design for the Telecom Triage Agent](#worked-example-prompt--context-design-for-the-telecom-triage-agent)
7. [Architecture & Implementation Best Practices](#architecture--implementation-best-practices)
8. [Common Pitfalls and Limitations](#common-pitfalls-and-limitations)
9. [Official Documentation References](#official-documentation-references)
10. [Domain Coverage / Traceability Matrix](#domain-coverage--traceability-matrix)

---

## Domain Overview

Domain 2 governs the three levers an architect pulls **before** touching architecture at all: which model tier to use, how to structure the prompt, and how to manage the context window. The domain's central claim is that getting these three levers right often removes the need for a fancier Domain-1 pattern entirely — a well-tiered model with a well-layered prompt and a disciplined context budget frequently clears an eval bar that a poorly-tuned agent, however sophisticated its topology, does not.

Three threads recur throughout: **(1) reversibility as a design preference** — tuning `effort` or testing a cheaper tier is cheap and instantly reversible, while a full architecture change is not, so cheap/reversible levers should be exhausted first; **(2) the probabilistic/deterministic split** — prompt-based guardrails and instructions *persuade*, code-level checks and programmatic guardrails *enforce*, mirroring the same distinction Domain 7 makes about hooks vs. context; and **(3) context as a scarce, actively-managed resource** — tokens are not free real estate to pad "just in case," because context rot means more tokens can actively *lower* quality, not just cost more money.

---

## Section 1: Model Selection

### 1.1 Think in tiers, not names

**What it is.** A tier-based mental model for choosing among Claude models, rather than memorizing specific model names:

| Tier | Model family | Use case |
|---|---|---|
| Fast, economical | Haiku 4.5 | High-volume, bounded work: classification, extraction, routing |
| Scale default | Sonnet 5 | Sized to carry most production traffic |
| Flagship | Opus 4.8 | Complex, multi-step agentic work |
| Frontier | Fable 5.1 / Mythos 5.1 | Highest per-token price; reached only when the flagship visibly falls short |

**Why it matters.** A dateless model name like `claude-opus-4-8` is a **fixed snapshot with stable weights**, not an auto-upgrading pointer — though the serving layer around it (routing, safety classifiers) can still drift over time. **Pin explicit model names in production** for reproducible evaluations, and build migration windows into your roadmap ahead of any deprecation date, since calls to a retired model return hard errors, not silent fallbacks.

**Implementation — pinning model versions and handling deprecation gracefully:**

```python
from anthropic import Anthropic, NotFoundError

client = Anthropic()

# Pin explicit snapshots — never use a bare "latest" alias in production code
MODEL_TIERS = {
    "fast": "claude-haiku-4-5",
    "scale": "claude-sonnet-4-5",
    "flagship": "claude-opus-4-8",
}

def call_with_deprecation_guard(prompt: str, tier: str = "scale") -> str:
    model = MODEL_TIERS[tier]
    try:
        response = client.messages.create(
            model=model, max_tokens=1024,
            messages=[{"role": "user", "content": prompt}],
        )
        return response.content[0].text
    except NotFoundError:
        # A retired model returns a hard error, never a silent fallback.
        # A migration window (documented, tested ahead of deprecation) is the correct
        # response — NOT catching this and silently routing to a different model at runtime,
        # which would break the reproducibility this pinning discipline exists to protect.
        raise RuntimeError(
            f"Model {model} has been retired. Migrate MODEL_TIERS['{tier}'] per your "
            "documented deprecation runbook before resuming traffic."
        )
```

**Common pitfalls:**
- Using an auto-upgrading model alias in production, then being unable to reproduce an eval result from three weeks ago because the underlying weights silently changed.
- Treating a `NotFoundError` from a retired model as something to catch-and-retry-on-a-different-model automatically — this breaks the whole point of pinning, which is deliberate, tested migration.

---

### 1.2 Two starting strategies attack opposite risks

**What it is.** Two opposite starting points for model selection, chosen by task shape:

- **Start cheap, then upgrade:** for high-volume, bounded, latency-sensitive work. Upgrade only the *slices* that miss the quality bar, not the whole pipeline — a lightweight classifier can route the hard 10% up to a stronger tier while the easy 90% stays cheap.
- **Start capable, then optimize:** for ambiguous, multi-step, or high-cost-of-error tasks. Prove the task is solvable at all on the flagship first; only then look for a cheaper tier that clears the same bar.

**Why it matters.** Architects who default to "start cheap everywhere" often burn more engineering time chasing prompt fixes for a model that was never going to clear a genuinely hard reasoning bar than they would have spent validating on the flagship first.

**Implementation — a cascading-tier router implementing "start cheap, then upgrade" with a confidence-based escalation:**

```typescript
import Anthropic from "@anthropic-ai/sdk";

const client = new Anthropic();

interface TierResult { text: string; confidence: number; tier: string; }

async function classifyWithEscalation(ticketText: string): Promise<TierResult> {
  // Start cheap: bounded, high-volume classification task
  const cheap = await client.messages.create({
    model: "claude-haiku-4-5",
    max_tokens: 200,
    messages: [{ role: "user", content: `Classify with a confidence score 0-1: ${ticketText}` }],
  });
  const parsed = JSON.parse((cheap.content[0] as any).text);

  if (parsed.confidence >= 0.85) {
    return { ...parsed, tier: "fast" }; // easy 90% stays cheap
  }

  // Escalate only the uncertain slice, never the whole pipeline
  const escalated = await client.messages.create({
    model: "claude-sonnet-4-5",
    max_tokens: 400,
    messages: [{ role: "user", content: `Classify carefully: ${ticketText}` }],
  });
  return { ...JSON.parse((escalated.content[0] as any).text), tier: "scale" };
}
```

**"Start capable" implementation pattern — validate on flagship first, then test cheaper tiers against the same eval set (Domain 4):**

```python
def validate_task_solvability(eval_cases: list[dict]) -> dict:
    """Run against the flagship FIRST to confirm the task is solvable at all,
    before spending engineering time debugging a cheaper tier that may never clear the bar."""
    flagship_results = run_eval(model="claude-opus-4-8", cases=eval_cases)
    if flagship_results["accuracy"] < 0.90:
        raise RuntimeError("Task not solvable even at flagship tier — this is a task-design "
                            "problem (Domain 1), not a model-selection problem.")
    # Only now does it make sense to test whether a cheaper tier also clears the bar
    scale_results = run_eval(model="claude-sonnet-4-5", cases=eval_cases)
    return {"flagship": flagship_results, "scale_tier_viable": scale_results["accuracy"] >= 0.90}

def run_eval(model: str, cases: list[dict]) -> dict: ...
```

**Common pitfalls:**
- Defaulting to "start cheap" for ambiguous, multi-step reasoning tasks, then attributing repeated failures to prompt quality rather than recognizing the tier itself is the limiting factor.
- Never revisiting a "start capable" decision — once the flagship proves the task solvable, failing to circle back and test whether a cheaper tier also clears the bar leaves cost on the table.

---

### 1.3 Tune effort before switching models

**What it is.** Before reaching for a pricier tier, tune the **effort** parameter — five levels from `low` through `xhigh` to `max`, with `high` as the default when omitted. This governs how much the model reasons before responding, and is cheaper to test and instantly reversible compared to a full model swap.

**Fast mode** is a separate, orthogonal lever — up to 2.5× output tokens/second on Opus-tier models at premium pricing (a research preview at time of writing). It's justified only when a hard latency target is the actual design constraint, not as a default "make it feel snappier" toggle.

**Implementation — testing effort levels before escalating model tier:**

```python
from anthropic import Anthropic

client = Anthropic()

def test_effort_levels(prompt: str, model: str = "claude-sonnet-4-5"):
    """Effort tuning is cheap to test and instantly reversible — always exhaust this
    lever before concluding you need a more expensive model tier."""
    results = {}
    for effort in ["low", "medium", "high", "xhigh", "max"]:
        response = client.messages.create(
            model=model,
            max_tokens=1024,
            effort=effort,  # hypothetical parameter name — verify exact API surface in current docs
            messages=[{"role": "user", "content": prompt}],
        )
        results[effort] = response.content[0].text
    return results

# Fast mode: only when latency IS the design constraint (Domain 1 §1.2's performance pillar)
def call_with_fast_mode_if_latency_bound(prompt: str, is_latency_critical: bool):
    kwargs = {"model": "claude-opus-4-8", "max_tokens": 1024,
              "messages": [{"role": "user", "content": prompt}]}
    if is_latency_critical:
        kwargs["fast_mode"] = True  # premium pricing — justified only by a real latency SLA/SLO
    return client.messages.create(**kwargs)
```

**Common pitfalls:**
- Swapping to a more expensive model tier before testing whether raising `effort` on the current tier already closes the quality gap — this skips the cheapest, most reversible lever available.
- Enabling fast mode by default across all traffic "to feel more responsive" rather than reserving the premium cost for genuinely latency-bound paths.

---

### 1.4 Routing at scale

**What it is.** At production scale, routing itself becomes infrastructure, not a manual per-request decision:
- A **lightweight classifier** sends simple work down-tier and genuinely hard requests up-tier.
- **Narrow subagent workers** run on a fast model while the **orchestrator** stays on a capable one (Domain 1 §3).

**Stack the cost levers, don't pick one:**
1. **Batch API** — roughly half price for latency-insensitive work (Domain 3/4).
2. **Prompt caching** — for stable prefixes reused across calls (§4.5 below).
3. **Tokenizer re-baseline on migration** — a newer model generation can count the *same text* as roughly 30% more tokens than the prior generation, which silently changes your cost model if you don't re-measure after a migration.

**Implementation — stacking all three cost levers in a production routing layer:**

```python
import anthropic

client = anthropic.Anthropic()

def send_batch_with_caching(requests: list[dict]):
    """Stacks Batch API (latency-insensitive discount) + prompt caching
    (stable system prompt reused across every request in the batch)."""
    batch_requests = [
        {
            "custom_id": req["id"],
            "params": {
                "model": "claude-haiku-4-5",
                "max_tokens": 300,
                "system": [
                    {
                        "type": "text",
                        "text": STABLE_CLASSIFICATION_INSTRUCTIONS,
                        "cache_control": {"type": "ephemeral"},  # reused across every request in the batch
                    }
                ],
                "messages": [{"role": "user", "content": req["ticket_text"]}],
            },
        }
        for req in requests
    ]
    return client.messages.batches.create(requests=batch_requests)

STABLE_CLASSIFICATION_INSTRUCTIONS = "You classify network fault tickets into 12 queues..."

def reestimate_costs_after_migration(sample_texts: list[str], old_model: str, new_model: str):
    """Tokenizer re-baseline: a migration can silently change token counts ~30% even for
    byte-identical text. Re-measure — never assume cost parity across model generations."""
    old_counts = [client.count_tokens(model=old_model, text=t) for t in sample_texts]
    new_counts = [client.count_tokens(model=new_model, text=t) for t in sample_texts]
    delta_pct = (sum(new_counts) - sum(old_counts)) / sum(old_counts) * 100
    return {"token_delta_pct": delta_pct}
```

**Common pitfalls:**
- Picking exactly one cost lever (usually caching) and assuming it's sufficient — the source domain is explicit that these stack, and skipping Batch API for genuinely latency-insensitive bulk work leaves roughly half the achievable savings on the table.
- Migrating to a new model generation and reusing old cost projections without re-measuring tokenization — a silent ~30% token-count shift can quietly blow a cost budget that looked fine in planning.

---

## Section 2: Prompt & Guardrail Design

### 2.1 Placement is the core rule

**What it is.** A strict placement discipline for what belongs in the system prompt versus the user turn:

| Lives in system prompt | Lives in user turn |
|---|---|
| Persistent behavior, role, standing constraints | Ephemeral, per-request data |
| The output contract | Customer-specific fields, timestamps, session data |

**Why it matters.** Baking request-specific data into the static scaffold **silently kills your cache** (§4.5) and breaks regression testing, because every "identical" test case now has a different byte-for-byte prompt.

**Implementation — enforcing the placement rule structurally:**

```python
from dataclasses import dataclass

@dataclass
class PromptAssembly:
    system_prompt: str   # must contain ONLY stable content
    user_content: str    # per-request volatile data goes here, never in system_prompt

def assemble_prompt(ticket_text: str, customer_id: str, timestamp: str) -> PromptAssembly:
    # WRONG pattern (commented out) — this would kill caching:
    # system = f"You are a triage assistant. Current time: {timestamp}. Customer: {customer_id}."
    system = "You are a triage assistant for network fault tickets. Respond in valid JSON."
    user = f"Ticket: {ticket_text}\nCustomer: {customer_id}\nTimestamp: {timestamp}"
    return PromptAssembly(system_prompt=system, user_content=user)

def assert_cache_safe(system_prompt_across_calls: list[str]):
    """Regression-test aid: if 'identical' test cases produce different byte-for-byte
    system prompts, volatile data has leaked into the static scaffold."""
    assert len(set(system_prompt_across_calls)) == 1, "System prompt is not stable across calls!"
```

**Common pitfalls:**
- Embedding a timestamp, session ID, or customer name in the system prompt "for convenience" — this is the single most common silent cache-killer (§4.5) and is called out explicitly there too.

---

### 2.2 Layer the system prompt deliberately

**What it is.** Anthropic's recommended system-prompt ordering, which maps to how attention and instruction-following behave in practice:[1]

1. **Role and task** first — tell the model what it is before what to do.
2. **Context and motivation** in the middle — Claude follows instructions measurably better when it understands *why*, not just *what*.
3. **Numbered, sequential steps** for genuinely multi-part work.
4. **Output format + XML delimiters** at the end.

Prefer **positive instructions** ("Respond only in valid JSON matching this schema") over lists of prohibitions ("Don't add commentary, don't use markdown, don't..."). A long "don't" list is harder for the model to satisfy jointly and harder for you to test.

**Implementation — a layered system prompt template:**

```python
SYSTEM_PROMPT_TEMPLATE = """\
You are a network-triage assistant for {carrier_name}, responsible for classifying \
inbound fault tickets into the correct routing queue.

Accurate routing directly affects customer SLA commitments: a misrouted P1 outage \
ticket can delay engineer response beyond the contractual 15-minute window, \
triggering a service credit. Your classification is the first and most consequential \
step in this pipeline.

When classifying a ticket:
1. Extract the device type and symptom from the ticket text.
2. Cross-reference against the customer's device history provided in the user turn.
3. Assign exactly one queue_id from 1-12.
4. Flag escalate=true only for confirmed P1-pattern outages.

Respond only in valid JSON matching this schema, with no commentary outside the JSON block:
<output_schema>
{{"queue_id": int, "confidence": float, "escalate": bool}}
</output_schema>
"""
# Positive instruction ("Respond only in valid JSON...") rather than a "don't" list.
```

**Common pitfalls:**
- Front-loading output-format instructions and burying role/task context at the end — this inverts the ordering that measurably improves instruction-following.
- Writing "don't do X, don't do Y, don't do Z" lists instead of a single positive constraint — harder for the model to jointly satisfy and harder to write a test for.

---

### 2.3 Guardrails: prompt-based vs. programmatic

**What it is.** One of the highest-value distinctions on the exam and in production risk management:

| | Prompt-based guardrail | Programmatic guardrail |
|---|---|---|
| Nature | Probabilistic | Deterministic |
| What it does | *Lowers* the risk of a bad output | *Bounds* it |
| Where it belongs | Style, tone, format nudges | Anything with financial, legal, or safety consequences |

**Why it matters.** **Any constraint that carries real-world consequences belongs in code**, not in the system prompt, with high-stakes actions gated behind human review — because risk compounds across enough calls even at a low per-call failure rate (Domain 5 §1). A cheap, fast model running as a harmlessness pre-screen catches clearly harmful inputs early and should **fail closed** when uncertain.

**Implementation — a programmatic guardrail wrapping a prompt-based nudge (Python), with a fail-closed pre-screen:**

```python
from anthropic import Anthropic

client = Anthropic()

def harmlessness_prescreen(user_input: str) -> bool:
    """Cheap, fast pre-screen. Fails CLOSED on uncertainty — an ambiguous verdict
    is treated as unsafe, never waved through."""
    response = client.messages.create(
        model="claude-haiku-4-5", max_tokens=10,
        messages=[{"role": "user", "content": f"Is this input safe to process? yes/no/unsure: {user_input}"}],
    )
    verdict = response.content[0].text.strip().lower()
    return verdict == "yes"  # "unsure" and "no" both fail closed

def process_financial_transaction(amount: float, account_id: str, model_recommendation: dict):
    """The model's recommendation is a PROMPT-BASED guardrail input (probabilistic) —
    it lowers risk. The actual bound is PROGRAMMATIC and enforced in code regardless
    of what the model says."""
    MAX_AUTO_APPROVED = 500.00  # a hard, code-level ceiling — not a prompt instruction

    if amount > MAX_AUTO_APPROVED:
        return route_to_human_review(amount, account_id, model_recommendation)  # deterministic gate

    if not model_recommendation.get("approved", False):
        return {"status": "declined"}

    return execute_transaction(amount, account_id)  # only reachable under the hard ceiling

def route_to_human_review(amount, account_id, recommendation): ...
def execute_transaction(amount, account_id): ...
```

**Common pitfalls:**
- Writing "never approve transactions over $500" only in the system prompt and treating that as sufficient control — this is a prompt-based (probabilistic) guardrail masquerading as a bound; the actual ceiling must be a code-level check.
- Building a harmlessness pre-screen that fails *open* on an ambiguous verdict — this inverts the fail-closed principle exactly where it matters most.

---

### 2.4 Reading the stop reason before touching the response body

**What it is.** Three `stop_reason` values with distinct handling implications:

- `end_turn` — healthy completion.
- `max_tokens` — **truncation**, not a wrong answer; don't score it as a quality failure.
- `refusal` — a policy decline. `stop_reason` returns `refusal`, and stop *details* (category plus a short explanation) populate only on refusal — they're empty on every other stop reason.

**Why it matters.** Route refusals to a dedicated handler with a fixed, friendly message and a safe next step. **Never echo the model's internal refusal explanation directly to the end user** — it's diagnostic detail, not customer-facing copy.

**Implementation — stop-reason-aware response handling:**

```python
def handle_response(response) -> dict:
    if response.stop_reason == "end_turn":
        return {"status": "ok", "content": response.content[0].text}

    if response.stop_reason == "max_tokens":
        # Truncation, NOT a quality failure — do not log this as an eval miss (Domain 4)
        return {"status": "truncated", "content": response.content[0].text,
                "action": "retry_with_higher_max_tokens_or_summarize"}

    if response.stop_reason == "refusal":
        # stop_reason details (category + explanation) populate ONLY here
        internal_detail = getattr(response, "stop_sequence_details", None)
        log_refusal_for_diagnostics(internal_detail)  # internal use only
        return {
            "status": "declined",
            "content": "I'm not able to help with that request. Please contact support "
                        "if you believe this is an error.",  # fixed, friendly — NEVER the raw model explanation
        }

    return {"status": "unknown_stop_reason", "raw": response.stop_reason}

def log_refusal_for_diagnostics(detail): ...
```

**Common pitfalls:**
- Treating `max_tokens` truncation as an eval failure and feeding it into accuracy metrics as a "wrong answer" — this conflates two different failure classes (Domain 4 distinguishes these carefully).
- Echoing the model's internal refusal category/explanation directly to a customer — this leaks diagnostic detail never intended for external audiences.

---

### 2.5 Prompt injection: direct vs. indirect

**What it is.** Two injection vectors:
- **Direct injection** — the adversary types it straight into user input.
- **Indirect injection** — the payload is hidden in a document, web page, or tool result the agent retrieves. This is the **dominant risk for anything that browses, reads files, or calls external tools**, because the untrusted content arrives disguised as legitimate data.

**Why it matters — defend structurally, with four stacked layers** (not one clever prompt):
1. Isolate untrusted content inside tool-result blocks.
2. JSON-encode or XML-fence payloads so they cannot escape their data context.
3. Grant every tool least privilege to shrink the blast radius of a successful injection.
4. Verify high-stakes actions in code, not by trusting the model's self-report.

**Implementation — the four-layer defense in a tool-calling agent (TypeScript):**

```typescript
import Anthropic from "@anthropic-ai/sdk";

const client = new Anthropic();

// Layer 3: least privilege — this tool can only READ, never write or escalate
const tools: Anthropic.Tool[] = [
  {
    name: "fetch_webpage",
    description: "Read-only fetch of a webpage's text content. Cannot execute code or make further requests.",
    input_schema: { type: "object", properties: { url: { type: "string" } }, required: ["url"] },
  },
];

function isolateUntrustedContent(rawFetchedText: string): string {
  // Layer 1 + 2: isolate inside a fenced, clearly-labeled data block that cannot
  // be mistaken for an instruction, and JSON-encode to prevent delimiter escape.
  return JSON.stringify({ untrusted_web_content: rawFetchedText });
}

async function executeHighStakesAction(actionRequest: { type: string; params: any }, modelSelfReport: string) {
  // Layer 4: verify in code — the model's self-report ("I confirmed this is safe")
  // is NOT the check; a code-level allowlist and validation is.
  const ALLOWED_ACTIONS = new Set(["read_ticket", "update_ticket_status"]);
  if (!ALLOWED_ACTIONS.has(actionRequest.type)) {
    throw new Error(`Action '${actionRequest.type}' is not in the code-level allowlist — ` +
                     `regardless of model self-report: "${modelSelfReport}"`);
  }
  return performAction(actionRequest);
}

async function performAction(req: { type: string; params: any }) { /* ... */ }
```

**Common pitfalls:**
- Assuming direct-injection defenses (input sanitization on user-typed text) also cover indirect injection — a payload hidden in a fetched webpage or document bypasses input sanitization entirely because it never passed through the "user input" path.
- Trusting a model's own claim that a retrieved document's embedded instructions were ignored — this is exactly the self-report Layer 4 says not to trust; the actual bound must be enforced in code (tool scoping, allowlists).
- Granting a browsing/file-reading tool write or execute capability "for convenience" — this maximizes blast radius exactly where indirect injection risk is highest.

---

## Section 3: Prompting Techniques: Zero-Shot, Few-Shot, Chain-of-Thought

### 3.1 Zero-shot is the default on current models

**What it is.** Strong instruction-following on current Claude models clears most well-specified tasks with **no examples at all**. Every example added costs input tokens on *every* call forever.

**Why it matters.** Only add few-shot when plain instructions keep demonstrably missing on your eval set — don't add examples pre-emptively "to be safe." This is the same reversibility-first, cheapest-lever-first discipline as §1.3's effort-before-model-swap rule, applied to prompting technique instead of model tier.

**Implementation — gating few-shot addition on eval evidence:**

```python
def should_add_few_shot(zero_shot_eval_score: float, target: float) -> bool:
    """Never add examples pre-emptively. This is a decision gated on measured
    failure against a held-out eval set (Domain 4), not intuition."""
    if zero_shot_eval_score >= target:
        return False  # zero-shot already clears the bar — adding examples only adds token cost
    return True
```

**Common pitfalls:**
- Adding 3-5 examples "just in case" to every new prompt without first measuring whether zero-shot already clears the target — this pays a permanent per-call token tax for a benefit that was never demonstrated to exist.

---

### 3.2 Few-shot buys three specific things

**What it is.** Few-shot examples buy exactly three things and nothing else: **format lock, call-to-call consistency, and edge-case coverage.** Use **three to five diverse, consistently structured examples**, wrapped in example tags.

**Classic failure modes:**
- **All-happy-path examples** — the model never sees what a hard case looks like, so it doesn't generalize to one.
- **Inconsistent shapes** — examples that don't share a structural pattern teach the model the wrong invariant.
- **Near-duplicates** — three examples that are really one example three times, buying no additional coverage for the token cost.

**Quality and coverage beat raw count every time.**

**Implementation — well-constructed few-shot examples (diverse, consistent shape, includes an edge case):**

```python
FEW_SHOT_EXAMPLES = """
<example>
<ticket>Customer reports total loss of signal, device offline 45 min, single location.</ticket>
<classification>{"queue_id": 3, "confidence": 0.95, "escalate": true}</classification>
</example>
<example>
<ticket>Customer asks about upgrading their data plan next month.</ticket>
<classification>{"queue_id": 7, "confidence": 0.98, "escalate": false}</classification>
</example>
<example>
<ticket>Intermittent slow speeds, only during evening hours, multiple neighbors affected.</ticket>
<classification>{"queue_id": 2, "confidence": 0.71, "escalate": false}</classification>
</example>
"""
# Note: three examples, three DIFFERENT queue_ids, one explicit low-confidence edge case —
# not three near-identical happy-path outage tickets.
```

**Common pitfalls:**
- Writing five examples that are all "textbook" clean cases — the model never learns what an ambiguous or partial-information ticket looks like, and fails exactly where few-shot was supposed to help most.

---

### 3.3 Extended thinking has replaced hand-written CoT scripts

**What it is.** Hand-written "think step by step" scripts are largely obsolete on current models. **Extended thinking** is an API feature that produces separate reasoning blocks natively — and a rigid, hand-authored script can actually *constrain* a better internal reasoning path the model would otherwise take.

On the current Opus-tier model, **adaptive thinking is the only supported mode** — it's off unless explicitly enabled, and legacy `budget_tokens`-style parameters are rejected outright by the API.

**Billing note:** reasoning tokens are billed whether or not you display them. Setting the display parameter to `summarized` changes *visibility only* — it never changes billing. Surface the full trace when you need it for debugging, auditing, or building user trust; otherwise summarize to keep the response clean. Match reasoning depth to the cost of a wrong answer — a customer-facing FAQ bot rarely needs `xhigh` effort; a contract-risk classifier might.

**Implementation — enabling extended thinking with cost-aware display settings:**

```python
from anthropic import Anthropic

client = Anthropic()

def call_with_extended_thinking(prompt: str, show_full_trace: bool = False):
    response = client.messages.create(
        model="claude-opus-4-8",
        max_tokens=2048,
        thinking={"type": "enabled", "display": "full" if show_full_trace else "summarized"},
        # NOTE: display=summarized changes VISIBILITY only — billing is identical either way.
        messages=[{"role": "user", "content": prompt}],
    )
    return response

# WRONG (will be rejected on current Opus-tier): legacy budget_tokens-style thinking config
# response = client.messages.create(model="claude-opus-4-8", thinking={"budget_tokens": 4000}, ...)
```

**Common pitfalls:**
- Continuing to hand-write "let's think step by step" scaffolding on models where extended thinking is available — this can constrain a better native reasoning path.
- Assuming `summarized` display reduces cost — it changes visibility only; reasoning tokens are billed regardless.
- Using `xhigh`/`max` effort or full-trace thinking by default on low-stakes, high-volume paths (e.g., a customer FAQ bot) where the cost of a wrong answer doesn't justify it.

---

### 3.4 Refine prompts like an engineer, not an artist

**What it is.** A disciplined four-step prompt-refinement loop:

1. Test against representative cases (not the three examples you wrote the prompt around).
2. Diagnose *before* touching anything — read the actual failure, don't guess.
3. Change exactly one thing.
4. Re-run the **full** test set to catch regressions the fix introduced elsewhere.

**Why it matters.** Externalize self-correction into a separate call when you need logging, gating, or a cheaper checker model to grade the primary model's output. Every self-critique loop needs an **explicit stopping condition** — "keep revising until good" is not a stopping condition, it's an infinite loop waiting to happen.

**Implementation — the four-step loop enforced as process, plus a bounded self-critique call:**

```python
def refine_prompt_iteration(current_prompt: str, full_test_set: list[dict],
                             change_description: str) -> dict:
    """Enforces: one change at a time, full-suite re-run every iteration."""
    before_results = run_eval(model="claude-sonnet-4-5", cases=full_test_set)  # baseline
    # ... apply exactly ONE change to current_prompt here, described by change_description ...
    after_results = run_eval(model="claude-sonnet-4-5", cases=full_test_set)  # FULL set, not just the failing cases

    regressions = [c for c in full_test_set
                   if before_results[c["id"]]["pass"] and not after_results[c["id"]]["pass"]]
    return {"change": change_description, "regressions_introduced": regressions,
            "net_improvement": after_results["accuracy"] - before_results["accuracy"]}

def externalized_self_critique(draft: str, max_iterations: int = 3) -> str:
    """Explicit stopping condition — bounded iteration count, not 'keep revising until good'."""
    for i in range(max_iterations):
        critique = call_checker_model(draft)  # cheaper model grading the primary output
        if critique["passes"]:
            return draft
        draft = apply_critique(draft, critique)
    return draft  # exits deterministically even if never fully satisfied

def call_checker_model(draft: str) -> dict: ...
def apply_critique(draft: str, critique: dict) -> str: ...
def run_eval(model: str, cases: list) -> dict: ...
```

**Common pitfalls:**
- Testing a prompt fix only against the cases that were failing, and shipping without re-running the full suite — this misses regressions the fix introduced in previously-passing cases.
- Writing a self-critique loop with no iteration cap ("keep revising until good") — this is an infinite loop waiting to happen, not a stopping condition.

---

## Section 4: Context Window Optimization & Prompt Reuse

### 4.1 Context rot is real and directional

**What it is.** As token count climbs, recall of any single fact *drops* — because *n* tokens create roughly *n²* pairwise attention links competing for a fixed attention budget. Padding a prompt with marginal, "might be useful" context doesn't just cost money; it **actively lowers quality** on the facts that matter.

**Why it matters — the guiding principle: informative yet tight** — the smallest set of high-signal tokens that does the job. The model does not prune itself. **Curation is an architecture decision you own**, not something that happens automatically as context grows.

**Implementation — a context budget enforcer that actively curates rather than passively accumulates:**

```python
def curate_context(candidate_snippets: list[dict], token_budget: int) -> list[dict]:
    """Curation is an explicit, owned decision — not automatic. Rank by relevance
    score and cut, rather than concatenating everything 'just in case'."""
    ranked = sorted(candidate_snippets, key=lambda s: s["relevance_score"], reverse=True)
    selected, used_tokens = [], 0
    for snippet in ranked:
        if used_tokens + snippet["token_count"] > token_budget:
            continue  # actively excluded, not silently included past budget
        selected.append(snippet)
        used_tokens += snippet["token_count"]
    return selected
```

**Common pitfalls:**
- Treating a larger context window as "more room to be safe" and padding accordingly — this measurably degrades recall on the facts that matter, it does not merely cost more.
- Assuming the model will "figure out what's relevant" from a large dump of marginal context — the source domain is explicit that the model does not prune itself.

---

### 4.2 Compaction vs. tool-result clearing

**What it is.** Two distinct context-management mechanisms with different costs and use cases:

| | Compaction | Tool-result clearing |
|---|---|---|
| Mechanism | A model call summarizes older turns | Free, mechanical, server-side edit |
| Cost | Real inference cost | Free |
| What survives | Key facts and decisions | Placeholders swap in for old results |
| Default trigger | 150,000 input tokens (documented floor: 50,000) | 100,000 tokens |
| Preserves | — | The 3 most recent tool-use pairs |
| Best for | Dialogue and reasoning growth | Large, re-fetchable tool-result bloat |

**Why it matters.** **Default to free clearing** for big re-fetchable tool output; pay for compaction only when the growth is genuinely dialogue and reasoning, not restated data you can pull again cheaply.

**Implementation — configuring both mechanisms (conceptual API shape):**

```python
def build_context_management_config(task_type: str) -> dict:
    if task_type == "long_diagnostic_session_with_repeated_tool_calls":
        # Large, re-fetchable tool output (e.g., repeated signal-strength checks) —
        # default to FREE clearing, not paid compaction.
        return {"tool_result_clearing": {"enabled": True, "trigger_tokens": 100_000, "keep_recent_pairs": 3}}
    elif task_type == "long_multi_turn_reasoning_dialogue":
        # Genuine dialogue/reasoning growth — this earns the inference cost of compaction.
        return {"compaction": {"enabled": True, "trigger_tokens": 150_000}}
    else:
        return {}
```

**Common pitfalls:**
- Paying for compaction on a session whose growth is actually just accumulated, re-fetchable tool output — free clearing would have achieved the same context reduction at zero cost.
- Assuming the compaction trigger is always exactly 150,000 tokens — the documented floor is 50,000, so verify configuration rather than assuming the default applies unconditionally.

---

### 4.3 Memory and subagent isolation

**What it is.** The **memory tool** persists state in files outside the context window, across sessions — structured note-taking the model writes to deliberately.

**Why it matters.** Write key state to memory **before** clearing or compaction erases it; once it's gone from context, it's gone unless it was externalized. When the valuable state already lives on disk (a git log, a progress file), starting a fresh window often beats compacting an old one in place.

**Subagents isolate context by design** (Domain 1 §3): messy exploration burns tokens in the subagent's own window, and only a distilled 1,000–2,000 token summary returns to the lead. This caps cost, contains failures, and limits leakage — but delegating genuinely simple, single-step work to a subagent just adds coordination overhead for nothing.

**Implementation — writing to memory before a compaction/clearing event:**

```python
def checkpoint_before_context_reduction(session_state: dict, memory_tool_client):
    """Anything not externalized before compaction/clearing is gone. Write
    deliberately, don't assume the model will remember to."""
    diagnostic_timeline = extract_key_facts(session_state)
    memory_tool_client.write(
        path="/memory/incident_diagnostics/ticket_4471.md",
        content=diagnostic_timeline,
    )
    # Only now is it safe to trigger compaction or let tool-result clearing proceed

def extract_key_facts(session_state: dict) -> str: ...
```

**Common pitfalls:**
- Allowing compaction or context clearing to run before critical diagnostic state has been externalized to memory — the information is unrecoverable once it leaves the context window unless it was written to disk first.
- Delegating a trivial, single-step lookup to a subagent — the coordination overhead (spawn, isolated context, summary return) exceeds the value for genuinely simple work.

---

### 4.4 Token accounting you actually need to track

**What it is.** `input_tokens` reported by the API counts **only the uncached remainder.** True prompt size is:

```
true_prompt_tokens = input_tokens + cache_creation_input_tokens + cache_read_input_tokens
```

**Why it matters.** Context awareness — a model tracking its own remaining budget — is scoped to **named models only** (e.g., Sonnet 5, Sonnet 4.6, Sonnet 4.5, Haiku 4.5). Never assume every model in your fleet has this capability; verify per model before relying on it in an agentic loop's stopping logic.

**Implementation — computing true prompt size and gating on verified context-awareness support:**

```python
CONTEXT_AWARE_MODELS = {"claude-sonnet-4-5", "claude-sonnet-4-6", "claude-haiku-4-5"}  # verify per release

def true_prompt_tokens(usage) -> int:
    return (usage.input_tokens
            + getattr(usage, "cache_creation_input_tokens", 0)
            + getattr(usage, "cache_read_input_tokens", 0))

def can_rely_on_self_reported_budget(model: str) -> bool:
    """Never assume every model has context-awareness — verify explicitly before
    using it as a stopping condition in an agentic loop."""
    return model in CONTEXT_AWARE_MODELS
```

**Common pitfalls:**
- Reading `input_tokens` alone as "the prompt size" and under-budgeting cost/context tracking — this ignores cache creation and cache read tokens entirely.
- Building an agentic loop's stopping logic around a model's self-reported remaining context budget without first verifying that specific model actually supports context awareness.

---

### 4.5 Prompt caching mechanics

**What it is.** Caching is a **strict byte-identical prefix match** over content rendered in a fixed order: **tools, then system, then messages.** A single changed character anywhere invalidates everything *after* it in the render order — so stable content leads, volatile per-request data trails, and a cache breakpoint marks the seam between them.

| | Reads | 5-min write | 1-hour write |
|---|---|---|---|
| Relative cost | ~0.1× base input | ~1.25× base | ~2× base |
| Pays off after | — | 1 read | 2+ reads |

**Why it matters.** Caching is a bet that cheap reads repay the write premium — match cache lifetime to how far apart reuses actually land in your traffic pattern. A prefix called only once always loses the bet and should not be cached.

**Debugging a silent cache miss:** verify with usage fields, never assumptions. High `cache_read_input_tokens` = hit. Zero reads across expected matches = silent miss, usually caused by:
- A timestamp or session ID baked into the "static" system prompt.
- Shifting tool order between calls.
- Unsorted JSON keys in a serialized object.
- A prefix below the model's minimum cacheable size (512–4,096 tokens depending on model; Haiku sits at the highest floor, and this fails **silently, with no error**).

Debug by diffing consecutive requests byte-for-byte to find the first differing character.

**Implementation — correctly structured cache breakpoint and a byte-diff debugging helper:**

```python
from anthropic import Anthropic

client = Anthropic()

def call_with_cache_breakpoint(stable_system_text: str, tools: list, volatile_user_content: str):
    return client.messages.create(
        model="claude-sonnet-4-5",
        max_tokens=1024,
        tools=tools,  # rendered FIRST — order must be stable across calls
        system=[
            {"type": "text", "text": stable_system_text,
             "cache_control": {"type": "ephemeral"}},  # cache breakpoint marks the seam
        ],
        messages=[{"role": "user", "content": volatile_user_content}],  # trails, never cached
    )

def verify_cache_hit(response) -> bool:
    """Verify with usage fields — never assume a hit occurred."""
    return getattr(response.usage, "cache_read_input_tokens", 0) > 0

def diff_requests_byte_for_byte(request_a: str, request_b: str) -> int:
    """Debug a silent cache miss by finding the first differing character."""
    for i, (a, b) in enumerate(zip(request_a, request_b)):
        if a != b:
            return i
    return min(len(request_a), len(request_b))  # one is a prefix of the other
```

**Common pitfalls:**
- Assuming a cache hit occurred because the code "looks" correctly structured — always verify via `cache_read_input_tokens`, never by inspection alone.
- Caching a prefix that's only ever called once — this always loses the write-premium bet and should not be cached at all.
- Serializing a dict to JSON with non-deterministic key order between calls — this silently breaks the byte-identical match requirement with no error raised.
- Missing the minimum cacheable size floor (512–4,096 tokens depending on model) and wondering why a short "stable" prefix never shows cache reads — this fails silently.

---

### 4.6 Modular prompts and Skills — two different kinds of reuse

**What it is.** Two distinct reuse mechanisms that are easy to conflate:

**Modular prompts** compose role, policy, format, and task blocks. The first three stay stable up front — giving isolated testability, fix-once consistency, and a naturally cache-friendly prefix — while only the task block and user data vary at the tail. **Modularity reuses *authored content*; caching reuses *computation*.** Never inject volatile data into the fixed skeleton, or you defeat both.

**Skills** package a whole capability — instructions plus optional bundled resources — that Claude invokes autonomously when a task matches its description. This differs from a subagent, which sends work *out* to an isolated context; **a Skill loads *into* the current context.** Skills use **progressive disclosure** in three levels:
1. Description — always in context (cheap).
2. Full instructions — loaded on invocation.
3. Bundled resources — loaded only on demand.

**Idle Skills cost almost nothing.** Scope (personal, project, plugin) sets reach, and because invocation is autonomous, **review a shared Skill's instructions and permissions before trusting it** — it will run with whatever access it's granted, without asking first.

**Implementation — a modular prompt assembler (Python):**

```python
class ModularPromptBuilder:
    """Role/policy/format blocks are stable and lead; task block varies and trails —
    this is BOTH modular (authored-content reuse) AND cache-friendly (computation reuse)."""

    def __init__(self, role: str, policy: str, output_format: str):
        self.role = role
        self.policy = policy
        self.output_format = output_format

    def build(self, task_instructions: str) -> str:
        # Stable blocks first (cache-friendly prefix), task block last (volatile tail)
        return f"{self.role}\n\n{self.policy}\n\n{self.output_format}\n\n---\n{task_instructions}"

builder = ModularPromptBuilder(
    role="You are a network-triage assistant.",
    policy="Escalate only confirmed P1-pattern outages.",
    output_format="Respond in valid JSON matching the triage schema.",
)
# Different tasks reuse the same stable, cacheable prefix:
prompt_a = builder.build(task_instructions="Classify this ticket: ...")
prompt_b = builder.build(task_instructions="Summarize this escalation: ...")
```

**Example Skill file (progressive disclosure, per Domain 7 §2.4's Skill mechanism):**

```markdown
---
name: telecom-triage-escalation
description: Use when a ticket shows confirmed P1-outage patterns and needs formal escalation to engineering.
---
# Escalation Procedure  (loaded only on invocation — Level 2)

1. Confirm P1 pattern via the network.signal_check tool.
2. Draft an escalation summary using the template in resources/escalation_template.md (Level 3 — loaded only on demand).
3. Call escalate_to_engineer with the summary and confidence score.
```

**Common pitfalls:**
- Injecting volatile per-request data into a modular prompt's "stable" blocks — this defeats both the modularity (the block is no longer reusable authored content) and the caching (the byte-identical prefix breaks).
- Trusting a shared, autonomously-invoked Skill without reviewing its instructions and granted permissions first — it runs with whatever access it has, without asking, the moment its description matches the task.
- Confusing a Skill with a subagent — a Skill loads capability *into* the current context; a subagent sends work *out* to an isolated one. Using the wrong one for the wrong problem either bloats the main context (should have been a subagent) or adds unneeded coordination overhead (should have been a Skill).

---

## Worked Example: Prompt & Context Design for the Telecom Triage Agent

Building on Domain 1's architecture:

```
SYSTEM PROMPT (stable, cached prefix)
├── Role: "You are a network-triage assistant for [Carrier]..."
├── Context: why accurate routing matters (SLA consequences, motivation)
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

**Implementation note connecting the worked example to the mechanisms above:** the system-prompt structure follows §2.2's role → context → steps → format ordering exactly; the tier split (Haiku for routing, Opus for diagnosis) implements §1.1–1.2's tiering and "start capable" strategy; the cache breakpoint placement implements §2.1's placement rule and §4.5's caching mechanics; and the memory-before-compaction sequencing implements §4.3 directly.

---

## Architecture & Implementation Best Practices

1. **Pin explicit model snapshots in every production configuration** and maintain a documented deprecation-migration runbook, tested ahead of any announced retirement date.
2. **Exhaust cheap, reversible levers before expensive, structural ones**: tune `effort` before swapping model tier; test zero-shot before adding few-shot; test a cheaper tier only after proving flagship-level solvability on ambiguous tasks.
3. **Enforce the system-prompt/user-turn placement rule with a linter or code review checklist**, not just documentation — this is what actually protects your cache hit rate and regression-test stability at scale.
4. **Put every real-world-consequential constraint in code, gated behind human review for high-stakes actions** — never rely on prompt language alone for anything with financial, legal, or safety consequences.
5. **Treat context as an actively curated, owned budget**, not a passively-filled buffer — build explicit ranking/trimming logic rather than concatenating "everything that might help."
6. **Default to free tool-result clearing over paid compaction** unless the actual growth is dialogue/reasoning, not re-fetchable data.
7. **Externalize critical state to memory before any compaction or clearing event**, and always give self-critique loops an explicit, bounded stopping condition.
8. **Instrument and verify cache hits via usage fields on every deployment**, never by code inspection alone — build an automated alert on unexpected drops in `cache_read_input_tokens`.

---

## Common Pitfalls and Limitations

- **Using auto-upgrading model aliases in production** — breaks reproducibility of evals and regression tests.
- **Defaulting to "start cheap everywhere"** for ambiguous, multi-step reasoning tasks — burns engineering time chasing prompt fixes for a tier that was never going to clear the bar.
- **Baking volatile data (timestamps, session IDs, customer names) into the system prompt** — silently kills caching and regression testing simultaneously.
- **Treating prompt-based guardrails as sufficient for high-stakes actions** — the only real bound is a code-level, deterministic check.
- **Scoring `max_tokens` truncation as a quality failure** — it is a distinct failure class from a genuinely wrong answer.
- **Echoing raw refusal explanations to end users** — diagnostic detail is not customer-facing copy.
- **Assuming direct-injection defenses cover indirect injection** — the dominant risk for tool-using/browsing agents requires the full four-layer structural defense.
- **Adding few-shot examples pre-emptively** — pays a permanent token tax without demonstrated need.
- **Continuing to hand-write chain-of-thought scripts** on models with native extended thinking — can constrain a better reasoning path.
- **Assuming `summarized` thinking display reduces billing** — it changes visibility only.
- **Self-critique loops with no explicit stopping condition** — an infinite loop waiting to happen.
- **Padding context "to be safe"** — context rot means this actively lowers quality, not just cost.
- **Paying for compaction on re-fetchable tool-result bloat** — free clearing achieves the same reduction at zero cost.
- **Assuming a cache hit without checking usage fields** — verify, never assume.
- **Injecting volatile data into a modular prompt's stable blocks** — defeats both modularity and caching simultaneously.
- **Trusting a shared Skill's autonomous invocation without reviewing its permissions** — it runs with whatever access it's granted, unprompted.

---

## Official Documentation References

1. Anthropic Docs, *"Prompt engineering overview"* — https://docs.claude.com/en/docs/build-with-claude/prompt-engineering/overview
2. Anthropic Docs, *"Prompt caching"* — https://docs.claude.com/en/docs/build-with-claude/prompt-caching
3. Anthropic Docs, *"Extended thinking"* — https://docs.claude.com/en/docs/build-with-claude/extended-thinking
4. Anthropic Docs, *"Context editing and the memory tool"* — https://docs.claude.com/en/docs/build-with-claude/context-editing
5. Anthropic Docs, *"Agent Skills overview"* — https://docs.claude.com/en/docs/agents-and-tools/agent-skills/overview
6. Anthropic Docs, *"Models overview"* — https://docs.claude.com/en/docs/about-claude/models/overview
7. Anthropic Docs, *"Messages API reference"* (stop_reason, usage fields) — https://docs.claude.com/en/api/messages
8. Anthropic Docs, *"Batch API"* — https://docs.claude.com/en/docs/build-with-claude/batch-processing

> **Version note:** Effort levels, fast mode availability, extended-thinking parameter names (`thinking`, `display`), and context-awareness model support are actively evolving API surfaces. Verify exact parameter names, default values, and per-model support against `docs.claude.com` before implementation.

---

## Domain Coverage / Traceability Matrix

| Source domain item | Implementation guide section(s) | Coverage |
|---|---|---|
| 1.1 Model tiers table | §1.1 | ✅ Full, with pinning/deprecation code |
| 1.1 Dateless names are fixed snapshots; pin in production | §1.1 | ✅ Full |
| 1.2 Start-cheap vs. start-capable strategies | §1.2 | ✅ Full, with cascading-router + validation code |
| 1.3 Effort parameter (5 levels) before model swap | §1.3 | ✅ Full, with test-harness code |
| 1.3 Fast mode (orthogonal lever) | §1.3 | ✅ Full |
| 1.4 Routing at scale (classifier + narrow workers) | §1.4 | ✅ Full |
| 1.4 Stacked cost levers (Batch, caching, tokenizer re-baseline) | §1.4 | ✅ Full, with batch+cache code |
| 2.1 System prompt vs. user turn placement table | §2.1 | ✅ Full, with assembly/assertion code |
| 2.2 Four-layer system prompt ordering | §2.2 | ✅ Full, with template |
| 2.2 Positive instructions over "don't" lists | §2.2 | ✅ Full |
| 2.3 Prompt-based vs. programmatic guardrail table | §2.3 | ✅ Full, with fail-closed prescreen code |
| 2.4 Three stop_reason values and handling | §2.4 | ✅ Full, with handler code |
| 2.4 Never echo refusal explanation to end user | §2.4 | ✅ Full |
| 2.5 Direct vs. indirect injection | §2.5 | ✅ Full |
| 2.5 Four-layer structural defense | §2.5 | ✅ Full, with TS tool-scoping code |
| 3.1 Zero-shot as default | §3.1 | ✅ Full, with gating code |
| 3.2 Few-shot's three benefits + three failure modes | §3.2 | ✅ Full, with example block |
| 3.3 Extended thinking replacing hand-written CoT | §3.3 | ✅ Full, with API code |
| 3.3 Adaptive-thinking-only on Opus-tier; billing note | §3.3 | ✅ Full |
| 3.4 Four-step refinement loop | §3.4 | ✅ Full, with iteration code |
| 3.4 Externalized self-critique + explicit stopping condition | §3.4 | ✅ Full, with bounded-loop code |
| 4.1 Context rot (n² attention links) + informative-yet-tight | §4.1 | ✅ Full, with curation code |
| 4.2 Compaction vs. tool-result clearing table | §4.2 | ✅ Full, with config code |
| 4.3 Memory tool + subagent isolation | §4.3 | ✅ Full, with checkpoint code |
| 4.4 True prompt token formula + context-awareness scoping | §4.4 | ✅ Full, with formula code |
| 4.5 Cache mechanics (byte-identical, render order, cost table) | §4.5 | ✅ Full, with cache-breakpoint + diff-debug code |
| 4.5 Silent cache miss causes (4 listed) | §4.5 | ✅ Full |
| 4.6 Modular prompts (reuse authored content vs. computation) | §4.6 | ✅ Full, with builder code |
| 4.6 Skills + progressive disclosure (3 levels) | §4.6 | ✅ Full, with example Skill file |
| 4.6 Skill vs. subagent distinction | §4.6 | ✅ Full |
| Worked Example: Prompt & Context Design (full diagram) | Worked Example section | ✅ Full, reproduced and annotated |
| Key Takeaways (7 bullets) | Distributed across Best Practices + Pitfalls | ✅ Full — every bullet maps to at least one item |
| References 1–6 (source domain) | Official Documentation References | ✅ Full, retained and extended with Messages API + Batch API references |

**Coverage verification statement:** Every numbered subsection, table, and key-takeaway bullet in `domain-2-models-prompting-context-engineering.md` is represented above with expanded technical detail, at least one implementation pattern, and a runnable Python and/or TypeScript example wherever the source item names or implies a concrete mechanism. No source item was merged, omitted, or skipped.
