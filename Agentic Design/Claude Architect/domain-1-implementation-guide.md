# Domain Implementation Reference Guide — Domain 1: Solution Design & Architecture

**Source domain:** `domain-1-solution-design-architecture.md` (CCAR-P, exam weight 17% — the largest single domain)
**Guide type:** Implementation Reference (technical depth, code examples, API/config detail)
**Audience:** Solution architects designing Claude-based systems from a business ask through to a chosen architecture pattern.

> This guide preserves the full structure of the source domain (1:1 section mapping) and expands every item into: what it is, why it matters, the underlying Anthropic technical mechanism, implementation patterns, pitfalls, and runnable Python/TypeScript examples using the Claude API and Agent SDK. A full traceability matrix appears at the end.

---

## Table of Contents

1. [Domain Overview](#domain-overview)
2. [Section 1: Business Translation & Value Alignment](#section-1-business-translation--value-alignment)
   - 1.1 From vague ask to designable problem
   - The prime anti-pattern: technology-first design
   - 1.2 Aligning to business value pillars (SLI/SLO/SLA)
3. [Section 2: End-to-End & Pattern Architecture](#section-2-end-to-end--pattern-architecture)
   - 2.1 The four-stage anatomy
   - 2.2 The processing core: augmented LLM
   - 2.3 Anthropic's five workflow patterns
   - 2.4 The escalation ladder
4. [Section 3: Multi-Agent Design & Decomposition](#section-3-multi-agent-design--decomposition)
   - 3.1 Orchestrator-worker topology
   - 3.2 When multi-agent earns its cost
   - 3.3 Four failure modes to design against
   - 3.4 Decomposition as substrate
   - 3.5 Sizing the split correctly
5. [Worked Architecture: Telecom Fault-Triage System](#worked-architecture-telecom-fault-triage-system)
6. [Architecture & Implementation Best Practices](#architecture--implementation-best-practices)
7. [Common Pitfalls and Limitations](#common-pitfalls-and-limitations)
8. [Official Documentation References](#official-documentation-references)
9. [Domain Coverage / Traceability Matrix](#domain-coverage--traceability-matrix)

---

## Domain Overview

Domain 1 is the largest domain by exam weight (17%) because it governs the single most consequential decision in any Claude solution: **what to build before deciding how to build it.** Every subsequent domain — model/context strategy (Domain 2), integration (Domain 3), evaluation (Domain 4), governance (Domain 5), stakeholder communication (Domain 6), and developer enablement (Domain 7) — operates *inside* the architecture chosen here. A wrong choice at this stage (an agent where a workflow would do; a workflow where a single prompt would do) cannot be fully corrected downstream by better prompting or a bigger model.

The domain has one unifying discipline that recurs at every scale, from a single sentence of requirements to a multi-agent topology: **climb the complexity ladder only when forced, one rung at a time, and only when the simpler rung demonstrably fails against a measurable target.** This guide implements that discipline concretely: a business-translation checklist, a value-pillar-to-metric mapping, the augmented-LLM building block, Anthropic's five canonical workflow patterns (each with runnable code), the escalation ladder, and the orchestrator-worker multi-agent pattern with its failure modes and sizing heuristics.

---

## Section 1: Business Translation & Value Alignment

### 1.1 From vague ask to designable problem

**What it is.** A discipline for converting an ambiguous business request ("build us a chatbot") into a designable specification, completed **before** any architecture, model, or pattern is chosen.

**Why it matters.** Anthropic's own agent-design guidance opens with this exact discipline: start with the simplest solution that could work, and only add complexity when the simpler option demonstrably fails against a success target.[1] You cannot measure "demonstrably fails" against an undefined target — so the target must exist first. Picking a technology (an agent, a fine-tune, a RAG pipeline) before this translation is complete is the single most common root cause of failed AI projects: **the tool was chosen before the outcome was defined.**

**Translation is complete only when you can state, in one sentence with a number in it:**
1. The single decision or task being automated, stated at the *decision* granularity, not the *domain* granularity — "classify inbound tickets into one of 12 routing queues," not "help with support."
2. Inputs and outputs in concrete, testable formats — a JSON schema, not "a helpful response."
3. A measurable success target, set during discovery, before any architecture is chosen.

**Implementation — turning the discipline into an artifact.** A practical way to enforce this in a real solution-design process is a structured discovery brief that the architecture phase cannot start without:

```python
# discovery_brief.py — a structured artifact that gates architecture kickoff.
# This is intentionally NOT an LLM call — it's a design-process control,
# the equivalent of a "definition of ready" gate before Domain 1 §2 begins.

from dataclasses import dataclass
from typing import Optional

@dataclass
class DiscoveryBrief:
    task_statement: str          # single sentence, decision-level granularity
    input_schema: dict           # JSON schema, not prose description
    output_schema: dict          # JSON schema, not prose description
    success_metric: str          # e.g., "routing accuracy >= 92% on held-out set"
    success_target_value: float  # the number — e.g., 0.92
    baseline_value: Optional[float]  # current manual/legacy baseline, if known

    def is_designable(self) -> bool:
        """A brief is designable only when every field is populated with
        something concrete — not a placeholder like 'TBD' or 'a helpful response'."""
        checks = [
            len(self.task_statement.split()) <= 30,   # forces decision-level, not domain-level, framing
            bool(self.input_schema),
            bool(self.output_schema),
            bool(self.success_metric),
            self.success_target_value is not None,
        ]
        return all(checks)

brief = DiscoveryBrief(
    task_statement="Classify inbound network fault tickets into one of 12 routing queues.",
    input_schema={"ticket_text": "string", "customer_id": "string", "device_history": "object"},
    output_schema={"queue_id": "integer (1-12)", "confidence": "float", "escalate": "boolean"},
    success_metric="routing accuracy on held-out ticket set",
    success_target_value=0.92,
    baseline_value=0.75,  # manual routing baseline
)

assert brief.is_designable()  # architecture phase (Domain 1 §2) cannot start until this passes
```

**Common pitfalls:**
- Naming a model, pattern, or vendor during discovery — this anchors the team before constraints (especially latency and cost) are known, and is almost always wrong once those constraints surface.
- Accepting "a helpful response" as an output spec — if it can't be expressed as a schema, it can't be validated programmatically (Domain 1 §2.1's output-delivery stage) and can't be evaluated (Domain 4).

### The prime anti-pattern: technology-first design

**What it is.** Choosing a technology response before confirming the simpler option was tried and failed.

| Symptom | What actually happened |
|---|---|
| "Let's build an agent" | A single prompt or augmented LLM call would have solved it |
| "We need fine-tuning" | Retrieval or a better system prompt was never tried |
| "Add a RAG pipeline" | The answer already fits in the prompt with caching |

**Why it matters.** Each of these adds cost, latency, and failure surface the requirement never asked for. **Discovery produces four outputs — the problem, the stakeholders, the constraints, and the success criteria — and design only starts once all four exist.**

**Implementation check — a lightweight "have we earned this complexity?" gate before adding RAG, fine-tuning, or an agentic loop:**

```typescript
// complexity_gate.ts — a pre-architecture checklist, run before approving
// any RAG pipeline, fine-tune, or agent proposal.

interface ComplexityJustification {
  simplerOptionTried: boolean;       // was a single prompt / augmented LLM call actually tested?
  simplerOptionResult: number;       // measured score against the success target
  successTarget: number;
  proposedComplexity: "rag" | "fine-tune" | "agent";
}

function isComplexityJustified(j: ComplexityJustification): boolean {
  if (!j.simplerOptionTried) {
    throw new Error(
      `Cannot approve ${j.proposedComplexity}: simpler option was never tested against the success target.`
    );
  }
  return j.simplerOptionResult < j.successTarget; // only justified if the simple path demonstrably failed
}

// Example: RAG was proposed, but a cached-context single call already met the target
const justification: ComplexityJustification = {
  simplerOptionTried: true,
  simplerOptionResult: 0.94,
  successTarget: 0.92,
  proposedComplexity: "rag",
};
console.log(isComplexityJustified(justification)); // false — RAG is NOT justified; ship the simpler option
```

**Common pitfalls:**
- Approving a RAG pipeline or agent proposal without a documented result from the simpler baseline — this is the technology-first anti-pattern institutionalized as a process failure, not just an individual one.

---

### 1.2 Aligning to business value pillars

**What it is.** Every Claude solution should map to one, rarely more than two, of five value pillars — because each pillar implies a *different binding metric*, and optimizing the wrong metric produces a "successful" project (by its own chosen measure) that nobody actually wanted.

| Pillar | What it means | Binding metric | Common mistake |
|---|---|---|---|
| **Efficiency** | Same work, fewer resources/time | Time saved, tasks/hour vs. manual baseline | Confusing this with cost |
| **Transformation** | Doing what wasn't feasible before | Coverage a sample could never reach | Applying a cost lens to a moonshot |
| **Productivity** | Amplifying the people you have | Output per person — never headcount cuts | Framing as "replacing staff" |
| **Cost** | An existing outcome, cheaper | Spend per task, run-rate vs. prior approach | Using it as the default frame for every project |
| **Performance** | Meeting a latency/throughput/availability commitment | The SLI/SLO/SLA itself | Chasing accuracy when latency is the binding constraint |

**Why it matters.** The most common architectural mistake in industry is framing *every* project as a cost-reduction exercise. A transformation goal (analyzing 100% of customer calls instead of a 2% QA sample) justifies spend that a cost lens would reject outright. A performance goal (99.9% of API responses under 2 seconds) can force a *more* expensive path — smaller model, less retrieval, more caching — even though it superficially looks like a step backward on "accuracy." **The binding metric must be decided before architecture, because it directly moves three levers: model tier, retrieval depth, and workflow pattern.**

**SLI → SLO → SLA:**

| Term | Definition | Example |
|---|---|---|
| **SLI** | The raw measured signal, no target attached | p95 latency = 1.8s |
| **SLO** | An internal target on that signal, no external penalty if missed | "p95 < 2s" |
| **SLA** | A customer contract with real consequences (credits, refunds) if missed | "p95 < 2s or 10% service credit" |

**The single reliable test: if missing it triggers a penalty, it's an SLA.** A hard latency SLA can rule out an agentic loop entirely — every additional model turn adds unpredictable latency, and multi-step loops widen the tail (p99) far more than the average.

**Implementation — encoding the binding metric as an architecture-selection input:**

```python
# value_pillar_selector.py
from enum import Enum
from dataclasses import dataclass

class ValuePillar(Enum):
    EFFICIENCY = "efficiency"
    TRANSFORMATION = "transformation"
    PRODUCTIVITY = "productivity"
    COST = "cost"
    PERFORMANCE = "performance"

@dataclass
class SLADefinition:
    metric_name: str
    threshold: float
    has_contractual_penalty: bool

    @property
    def tier(self) -> str:
        return "SLA" if self.has_contractual_penalty else "SLO"

def architecture_constraints_from_pillar(pillar: ValuePillar, sla: SLADefinition) -> dict:
    """Returns architecture-selection guardrails driven by the chosen pillar + SLA tier."""
    constraints = {"max_workflow_pattern": "agent", "model_tier": "flagship"}

    if pillar == ValuePillar.PERFORMANCE and sla.tier == "SLA":
        # A hard latency SLA rules out unbounded agentic loops.
        constraints["max_workflow_pattern"] = "fixed_workflow"
        constraints["model_tier"] = "fast"  # e.g., Haiku-class for latency-bound paths
        constraints["retrieval_depth"] = "shallow_or_cached"

    if pillar == ValuePillar.COST:
        constraints["model_tier"] = "smallest_viable"

    return constraints

# Telecom triage example: performance pillar acting as a guardrail on P1 escalation
sla = SLADefinition(metric_name="P1_false_negative_rate", threshold=0.01, has_contractual_penalty=True)
print(architecture_constraints_from_pillar(ValuePillar.PERFORMANCE, sla))
# {'max_workflow_pattern': 'fixed_workflow', 'model_tier': 'fast', 'retrieval_depth': 'shallow_or_cached'}
```

**Worked example — telecom triage:** A carrier wants an AI system to pre-classify inbound network fault tickets.
- *Wrong frame:* "Reduce support headcount" (cost pillar, politically toxic, wrong metric).
- *Right frame:* "Efficiency — cut average time-to-correct-routing from 6 minutes (manual) to under 90 seconds, with routing accuracy ≥ 92% on a held-out ticket set, and P1 network outages escalated with ≤ 1% false-negative rate (performance pillar operating as a guardrail)."

This single reframe changes the architecture: a hard false-negative ceiling on P1 outages argues for a human-in-the-loop escalation gate (Domain 5), not full automation.

**Common pitfalls:**
- Defaulting to a cost frame for every project regardless of what the stakeholder actually values — this is the single most cited industry mistake in this section.
- Treating every internal target as an "SLA" colloquially — inflating SLO language to SLA status leads teams to over-invest in architecture rigidity where none is contractually required.
- Choosing the model/architecture before the SLI/SLO/SLA tier is known — the binding metric should *move* the model-tier and pattern choice, not be discovered after the fact and used to justify a decision already made.

---

## Section 2: End-to-End & Pattern Architecture

### 2.1 The four-stage anatomy every architecture owns

**What it is.** A Claude system is not "the model call" — it is four stages, and skipping any of them produces failures that don't throw exceptions; they quietly degrade quality until someone notices in production.

```
INPUT handling → PROCESSING (augmented LLM core) → OUTPUT delivery → FEEDBACK loop
        ▲                                                                  │
        └──────────────────────────────────────────────────────────────────┘
```

| Stage | Silent failure if skipped |
|---|---|
| Input handling | Bad/malformed data quietly degrades every downstream answer |
| Processing | (this is the part everyone designs — rarely the weak point) |
| Output delivery | Confidently wrong results ship with no validation |
| Feedback | No path exists from production outcomes back to improvement |

**Why it matters — feedback is not logging.** Logging ends at storage. A feedback loop routes captured outcomes into one of three actions: a new eval case, a prompt refinement, or a retraining dataset. In practice, teams over-reach for retraining (expensive, slow, usually unnecessary) and under-use the two cheap routes. **The loop only closes when a change ships and is verified against a held-out case — not when a ticket is filed.**

**Implementation — the four stages as a pipeline (Python), with output validation and a feedback hook:**

```python
# triage_pipeline.py
import json
from anthropic import Anthropic
from jsonschema import validate, ValidationError

client = Anthropic()

OUTPUT_SCHEMA = {
    "type": "object",
    "properties": {
        "queue_id": {"type": "integer", "minimum": 1, "maximum": 12},
        "confidence": {"type": "number", "minimum": 0, "maximum": 1},
        "escalate": {"type": "boolean"},
    },
    "required": ["queue_id", "confidence", "escalate"],
}

def input_handling(raw_ticket: dict) -> dict:
    """Validate, normalize, enrich. Silent failure here poisons everything downstream."""
    if "ticket_text" not in raw_ticket or not raw_ticket["ticket_text"].strip():
        raise ValueError("Malformed ticket: missing or empty ticket_text")
    raw_ticket["customer_id"] = raw_ticket.get("customer_id", "").strip().upper()
    # enrichment: attach device/account history (stubbed here)
    raw_ticket["device_history"] = fetch_device_history(raw_ticket["customer_id"])
    return raw_ticket

def processing(ticket: dict) -> str:
    """The augmented LLM core — Section 2.2."""
    response = client.messages.create(
        model="claude-sonnet-4-5",
        max_tokens=300,
        system="You classify network fault tickets. Respond with JSON only.",
        messages=[{"role": "user", "content": json.dumps(ticket)}],
    )
    return response.content[0].text

def output_delivery(raw_model_output: str) -> dict:
    """Programmatic validation — confidently wrong output must not ship unchecked."""
    parsed = json.loads(raw_model_output)
    validate(instance=parsed, schema=OUTPUT_SCHEMA)  # raises ValidationError if malformed
    return parsed

def feedback_loop(ticket: dict, result: dict, actual_outcome: dict):
    """Routes to ONE of three destinations — never just a log line."""
    was_correct = result["queue_id"] == actual_outcome["correct_queue_id"]
    if not was_correct:
        append_to_eval_set(ticket, expected=actual_outcome)          # route 1: new eval case
        if is_systematic_pattern(ticket, actual_outcome):
            flag_for_prompt_review(ticket, result, actual_outcome)    # route 2: prompt refinement
        # route 3 (retraining dataset) reserved for volume/severity thresholds only

def fetch_device_history(customer_id: str) -> dict: ...
def append_to_eval_set(ticket, expected): ...
def is_systematic_pattern(ticket, outcome) -> bool: ...
def flag_for_prompt_review(ticket, result, outcome): ...
```

**Common pitfalls:**
- Treating "we log every request/response" as equivalent to a feedback loop — logging without a routing decision back into eval/prompt/training is inert.
- Skipping schema validation on output because "the model is usually right" — this is exactly the confidently-wrong-ships-unchecked failure this stage exists to prevent.

---

### 2.2 The processing core: augmented LLM

**What it is.** The **augmented LLM** — one model call enhanced with retrieval, tools, and memory, where *your code* owns the control flow — is the single atomic building block every workflow and every agent is assembled from.[1]

**Why it matters — the exam's core distinction.** Tool use alone does not make a system agentic. The question is not "does it call tools?" but **"who decides the next step?"**

- If your code decides → **workflow** (predictable, testable, cheap)
- If the model decides → **agent** (flexible, expensive, harder to test)

If the whole knowledge base fits in the prompt with caching, skip retrieval machinery entirely — RAG is a cost paid for scale you may not have (Domain 3's ~200k token threshold).

**Tool-use loop concurrency rule:** read-only tools have no side effects and can run concurrently; state-changing tools run one at a time, sequentially, to preserve ordering safety. The loop repeats until the model stops requesting tool calls.

**Implementation — an augmented LLM call with tool use (Python, Anthropic SDK):**

```python
from anthropic import Anthropic

client = Anthropic()

tools = [
    {
        "name": "check_signal_strength",
        "description": "Read-only: query current signal strength for a device. Safe to run concurrently.",
        "input_schema": {"type": "object", "properties": {"device_id": {"type": "string"}}, "required": ["device_id"]},
    },
    {
        "name": "escalate_to_engineer",
        "description": "State-changing: creates an engineering escalation ticket. Must run sequentially, never concurrently with another state-changing call.",
        "input_schema": {"type": "object", "properties": {"ticket_id": {"type": "string"}, "reason": {"type": "string"}}, "required": ["ticket_id", "reason"]},
    },
]

def augmented_llm_call(user_message: str, conversation: list):
    conversation.append({"role": "user", "content": user_message})
    response = client.messages.create(
        model="claude-sonnet-4-5",
        max_tokens=1024,
        tools=tools,
        messages=conversation,
    )
    return response
```

**TypeScript equivalent (Claude Agent SDK) — the augmented LLM as the building block underneath a workflow controller:**

```typescript
import Anthropic from "@anthropic-ai/sdk";

const client = new Anthropic();

async function augmentedLlmCall(userMessage: string, history: Anthropic.MessageParam[]) {
  history.push({ role: "user", content: userMessage });
  return client.messages.create({
    model: "claude-sonnet-4-5",
    max_tokens: 1024,
    tools: [
      {
        name: "check_signal_strength",
        description: "Read-only. Safe to run concurrently with other read-only tools.",
        input_schema: {
          type: "object",
          properties: { device_id: { type: "string" } },
          required: ["device_id"],
        },
      },
    ],
    messages: history,
  });
}
```

**Common pitfalls:**
- Calling any system that uses tools "an agent" — the deciding factor is control-flow ownership, not tool presence.
- Running state-changing tool calls concurrently to "save time" — this breaks the ordering-safety guarantee the sequential rule exists to protect (e.g., two concurrent escalation-ticket creations racing on the same ticket ID).
- Building a RAG pipeline before confirming the corpus doesn't already fit in a cached prompt context.

---

### 2.3 Anthropic's five workflow patterns

**What it is.** Anthropic's canonical fixed-topology patterns — all are workflows, meaning *your code*, not the model, decides the sequence.[1]

| Pattern | Shape | When to use |
|---|---|---|
| **Prompt chaining** | Fixed sequence, programmatic gate between each step | Task decomposes into a known, ordered sequence |
| **Routing** | A classifier (can be a cheap model) sends input to a specialist path | Distinct input categories need distinct handling |
| **Parallelization — sectioning** | Independent subtasks run concurrently, results merged | Subtasks don't depend on each other |
| **Parallelization — voting** | Same task run N times, results aggregated for confidence | High-stakes classification needing consensus |
| **Orchestrator-workers** | Orchestrator decides subtasks *at runtime* from the specific input | Subtask structure can't be known in advance |
| **Evaluator-optimizer** | Generator + critic loop against clear judging criteria | Iterative refinement has a checkable target |

**The one subtlety the exam (and production) tests:** orchestrator-workers and parallelization share the same fan-out shape but differ on *timing*. Parallelization's subtasks are baked into your code before any call runs. The orchestrator decides the subtasks at runtime by reasoning over the input — buying flexibility at real token cost, because that reasoning step is itself a model call.

**Implementation — each pattern, minimally, in Python:**

```python
# 1. PROMPT CHAINING — fixed sequence with a programmatic gate between steps
def prompt_chain(ticket_text: str) -> dict:
    extracted = call_model(f"Extract key entities from: {ticket_text}")
    if not extracted.get("device_id"):           # programmatic gate
        return {"status": "insufficient_data"}
    diagnosis = call_model(f"Diagnose based on: {extracted}")
    routed = call_model(f"Route based on diagnosis: {diagnosis}")
    return routed

# 2. ROUTING — cheap classifier sends input to a specialist path
def route(ticket_text: str) -> dict:
    category = call_model(f"Classify as outage|billing|provisioning: {ticket_text}", model="claude-haiku-4-5")
    handlers = {"outage": handle_outage, "billing": handle_billing, "provisioning": handle_provisioning}
    return handlers[category](ticket_text)

# 3. PARALLELIZATION — SECTIONING — independent subtasks, merged
import concurrent.futures
def parallel_sectioning(ticket_text: str) -> dict:
    with concurrent.futures.ThreadPoolExecutor() as pool:
        sentiment_f = pool.submit(call_model, f"Sentiment of: {ticket_text}")
        urgency_f = pool.submit(call_model, f"Urgency of: {ticket_text}")
        category_f = pool.submit(call_model, f"Category of: {ticket_text}")
    return {"sentiment": sentiment_f.result(), "urgency": urgency_f.result(), "category": category_f.result()}

# 4. PARALLELIZATION — VOTING — same task N times, majority vote
def parallel_voting(ticket_text: str, n: int = 3) -> bool:
    with concurrent.futures.ThreadPoolExecutor() as pool:
        futures = [pool.submit(call_model, f"Is this a P1 outage? yes/no: {ticket_text}") for _ in range(n)]
        votes = [f.result() == "yes" for f in futures]
    return sum(votes) > n / 2  # majority

# 5. ORCHESTRATOR-WORKERS — orchestrator decides subtasks at runtime
def orchestrator_workers(complex_ticket: str) -> dict:
    plan = call_model(f"Plan the diagnostic subtasks needed for: {complex_ticket}")  # runtime decision (costs a call)
    results = {}
    for subtask in plan["subtasks"]:
        results[subtask["id"]] = call_model(subtask["prompt"])
    synthesis = call_model(f"Synthesize: {results}")
    return synthesis

# 6. EVALUATOR-OPTIMIZER — generator + critic loop against clear criteria
def evaluator_optimizer(draft_prompt: str, max_iterations: int = 3) -> str:
    draft = call_model(draft_prompt)
    for _ in range(max_iterations):
        critique = call_model(f"Critique against routing-accuracy criteria: {draft}")
        if critique["passes"]:
            return draft
        draft = call_model(f"Revise based on critique {critique}: {draft}")
    return draft  # exits after bounded iterations even if not perfect — no unbounded loop

def call_model(prompt: str, model: str = "claude-sonnet-4-5"): ...  # thin wrapper around client.messages.create
def handle_outage(t): ...
def handle_billing(t): ...
def handle_provisioning(t): ...
```

**Common pitfalls:**
- Implementing orchestrator-workers when the subtask structure is actually known in advance — this pays the runtime-planning token cost for flexibility the problem doesn't need; plain parallelization sectioning would have been cheaper and equally effective.
- Running evaluator-optimizer without a bounded iteration cap — an unbounded critic loop can burn tokens indefinitely on a draft that never satisfies the critic.
- Using voting/consensus for low-stakes classifications where a single pass is already well within the accuracy target — voting triples cost for a confidence gain the requirement never asked for.

---

### 2.4 The escalation ladder

**What it is.** Climb it only when forced, one rung at a time:

```
1. Single prompt
2. Augmented LLM (retrieval / tools / memory, one call)
3. Fixed workflow (chaining / routing / parallelization / orchestrator-workers / evaluator-optimizer)
4. Agent (model owns control flow) ← last resort
```

**Why it matters.** Agents bill you three ways every time you reach for one: **higher token cost** from the model's own reasoning loop, **added latency** from sequential dependent steps, and **the risk of compounding errors** across turns. This is not a warning against agents — it's a warning against reaching for one by default. Anthropic's own engineering guidance is explicit that the most successful production implementations use the simplest pattern that clears the bar, not the most sophisticated one available.[1]

**Worked example — the telecom triage system from Section 1:**
- Ticket text alone, single category → **routing** (cheap classifier model).
- "Is this a P1 outage?" needs consensus because false negatives are costly → **parallelization/voting** (3 independent classification passes, majority vote).
- "Diagnose *why* this specific customer is dropping WiFi calls" — the right next question depends on what the model just learned (firmware version → ask about driver; multi-device → ask about backhaul) → this is the one case in the whole system that earns an **agent**, because the path genuinely cannot be scripted in advance.

**Implementation — an escalation-ladder decision function used at design time (not runtime):**

```python
def select_rung(task_structure_known: bool, requires_dynamic_replanning: bool,
                 eval_score_at_simpler_rung: float, target: float) -> str:
    """Design-time decision aid, not a runtime dispatcher — this encodes the
    'climb only when forced' discipline as an explicit, auditable check."""
    if eval_score_at_simpler_rung >= target:
        raise AssertionError("Simpler rung already meets target — do not climb further.")
    if not task_structure_known and requires_dynamic_replanning:
        return "agent"
    if task_structure_known:
        return "fixed_workflow"
    return "augmented_llm"
```

**Common pitfalls:**
- Reaching for an agent for the entire pipeline because *one* sub-decision within it is genuinely ambiguous — the worked example shows only the diagnostic sub-step earning agent status; the rest of the pipeline stays fixed workflow.
- Never revisiting the rung choice — a task that was genuinely ambiguous at launch may become scriptable once enough production cases reveal a stable pattern, at which point a fixed workflow can replace the agent and cut cost/latency/error-compounding risk.

---

## Section 3: Multi-Agent Design & Decomposition

### 3.1 Orchestrator-worker topology

**What it is.** The lead agent owns exactly three jobs:

1. **Planning** — decompose the goal into scoped subtasks at runtime.
2. **Dispatch** — spawn each worker with the specific tools, prompt, and context *that task* needs.
3. **Synthesis** — merge worker results into one coherent answer.

**Why it matters — the detail most new architects get wrong.** **Subagents inherit nothing by default.** Every delegation must explicitly pass the task plus whatever context that task needs. Context isolation is engineered, not automatic: each worker runs in its own fresh conversation, only its final distilled message returns to the lead, and the lead's context grows by a summary, not the full transcript.[2]

**Implementation — orchestrator-worker with explicit context passing (Claude Agent SDK, TypeScript):**

```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

interface WorkerTask {
  id: string;
  scopedPrompt: string;      // explicitly constructed — nothing is inherited
  allowedTools: string[];    // explicitly scoped — least privilege per worker
  contextPayload: string;    // only what this specific task needs, not the lead's full history
}

async function dispatchWorker(task: WorkerTask): Promise<string> {
  let summary = "";
  for await (const message of query({
    prompt: `${task.scopedPrompt}\n\nContext:\n${task.contextPayload}`,
    options: { allowedTools: task.allowedTools },
  })) {
    if (message.type === "text") summary += message.text;
  }
  return summary; // ONLY this distilled result returns to the lead — not the worker's full transcript
}

async function orchestrate(goal: string) {
  // 1. PLANNING (runtime decomposition — this reasoning step itself costs a model call)
  const plan = await planSubtasks(goal);

  // 2. DISPATCH (parallel, isolated contexts)
  const results = await Promise.all(plan.map((task: WorkerTask) => dispatchWorker(task)));

  // 3. SYNTHESIS (lead's context grows by summaries only, not full worker transcripts)
  return synthesize(results);
}

async function planSubtasks(goal: string): Promise<WorkerTask[]> { /* ... */ return []; }
function synthesize(results: string[]): string { return results.join("\n---\n"); }
```

**Common pitfalls:**
- Assuming a worker "knows" what the lead knows because they're part of the same logical system — this is precisely the false assumption the source domain calls out as the most common new-architect error.
- Returning a worker's full transcript to the lead instead of a distilled summary — this defeats the context-isolation benefit entirely and reintroduces the context-bloat problem multi-agent design exists to solve.

---

### 3.2 When multi-agent earns its cost

**What it is.** Multi-agent pays off on work that is:
- **Breadth-first** (many independent angles to explore)
- **Genuinely separable** (workers don't need each other's intermediate state)
- **High-value** (worth the token multiplier)
- **Larger than one context window**

**Why it matters.** A single agent wins whenever steps depend on each other — most coding tasks, for instance, need a shared, consistent view of state, and constant context-syncing between agents would eat the parallelism gain alive. **Shape decides, not size.** A huge but tightly-coupled task is still a single-agent job.

**Anthropic's own benchmark** (2025, Opus-class lead orchestrating Sonnet-class subagents) reported the multi-agent system beating a single agent by **over 90% on an internal research evaluation**, with agents running at roughly 4× chat-level token consumption and multi-agent at roughly 15×. Anthropic reports that **token usage alone explained the majority of the performance variance** — treat this as directional evidence of the pattern's power, not a portable constant for every domain.[2]

**Implementation — a decision check encoding "shape decides, not size":**

```python
def multi_agent_justified(is_breadth_first: bool, is_separable: bool,
                           value_score: float, token_multiplier_budget: float = 15.0) -> bool:
    """Even if the answer is yes, this only tells you multi-agent COULD pay off —
    it still needs to clear the eval bar a single agent doesn't (Domain 4)."""
    if not is_separable:
        return False  # shape decides: tightly-coupled work is single-agent regardless of size
    if not is_breadth_first:
        return False
    estimated_token_cost_multiplier = 15.0  # Anthropic's reported directional figure, not a guarantee
    return value_score > estimated_token_cost_multiplier and estimated_token_cost_multiplier <= token_multiplier_budget
```

**Common pitfalls:**
- Assuming a large task automatically justifies multi-agent decomposition — size is not the qualifying criterion; separability is.
- Applying multi-agent to sequential, state-dependent work (like most coding tasks) purely because the task is large — this triggers exactly the context-syncing overhead that eats the parallelism gain.

---

### 3.3 Four failure modes to design against

| Failure mode | Cause | Design mitigation |
|---|---|---|
| **Over-spawning** | No cap on subagent count | Bound spawning explicitly; cap workers per orchestration |
| **Vague delegation** | Workers given ambiguous scope | Write explicit, non-overlapping task boundaries per worker |
| **Synchronous bottleneck** | Lead waits on the slowest worker | Set per-worker timeouts; design for partial-result synthesis |
| **Compounding errors** | Autonomy grows unchecked across turns | Validate intermediate results; bound step counts |

**Why it matters.** Nearly all of these trace back to one root cause: **delegation quality.** A well-scoped worker with a tight, non-overlapping mandate rarely causes the other three failures.

**Implementation — guardrails for all four failure modes in one orchestration wrapper (Python):**

```python
import concurrent.futures
from dataclasses import dataclass

MAX_WORKERS = 8           # mitigates over-spawning
WORKER_TIMEOUT_SECONDS = 45  # mitigates synchronous bottleneck
MAX_STEPS_PER_WORKER = 10    # mitigates compounding errors

@dataclass
class WorkerScope:
    task_id: str
    boundary: str    # explicit, non-overlapping description — mitigates vague delegation
    max_steps: int = MAX_STEPS_PER_WORKER

def run_orchestration(scopes: list[WorkerScope]) -> dict:
    assert len(scopes) <= MAX_WORKERS, f"Over-spawning: {len(scopes)} exceeds cap of {MAX_WORKERS}"
    boundaries = [s.boundary for s in scopes]
    assert len(boundaries) == len(set(boundaries)), "Vague delegation: overlapping worker boundaries detected"

    results = {}
    with concurrent.futures.ThreadPoolExecutor(max_workers=MAX_WORKERS) as pool:
        futures = {pool.submit(run_worker, s): s.task_id for s in scopes}
        for future in concurrent.futures.as_completed(futures, timeout=None):
            task_id = futures[future]
            try:
                results[task_id] = future.result(timeout=WORKER_TIMEOUT_SECONDS)
            except concurrent.futures.TimeoutError:
                results[task_id] = {"status": "partial", "reason": "timeout"}  # partial-result synthesis
    return results

def run_worker(scope: WorkerScope) -> dict:
    step_count = 0
    # ... agent loop, incrementing step_count each turn ...
    if step_count > scope.max_steps:
        raise RuntimeError(f"Compounding-error guard tripped for {scope.task_id}")
    return {"status": "complete"}
```

**Common pitfalls:**
- Fixing each failure mode with an ad-hoc patch after it occurs in production, instead of designing the four guardrails in up front — the source domain frames all four as *design* considerations, not incident-response reactions.

---

### 3.4 Decomposition: the substrate, not a pattern of its own

**What it is.** Decomposition isn't one more item on the pattern list from §2.3 — it's what every pattern *is*, in a different shape:
- Prompt chaining = decomposition into a fixed, ordered sequence.
- Orchestrator-workers = decomposition at runtime, dispatched to subagents.
- Even a single agent decomposes internally when it plans its own steps.

**Why it matters.** This reframes the exam's pattern list: the question is never "should I decompose?" — every non-trivial system already does, implicitly or explicitly. The real question is *when* the decomposition happens (design-time vs. runtime) and *who* performs it (your code vs. the model), which is exactly the axis §2.3's orchestrator-vs-parallelization subtlety and §2.4's escalation ladder both hinge on.

**Common pitfalls:**
- Treating "should this task be decomposed?" as a yes/no design question — it's already decomposed the moment it's a workflow or agent; the actual design question is *where* the decomposition boundary sits (§3.5) and *when* it's decided (§2.3).

---

### 3.5 Sizing the split correctly

**What it is.** **The right granularity is the coarsest split at which each subtask still runs reliably on its own and can be checked in isolation.**

- **Too coarse:** several hard steps bundled into one unpredictable failure you can't localize.
- **Too fine:** trivial work gets its own handoff, and coordination overhead (latency, tokens, information loss at each seam) swamps any gain in control.

**Pressure-test any proposed split with three questions:**
1. Can each subtask run reliably on its own?
2. Can you write a check that verifies each step independently?
3. Would merging two adjacent steps lose anything real?

If merging costs nothing, the split was too fine — collapse it. **Default to a single, well-structured prompt and decompose only when it demonstrably improves reliability against your held-out eval set (Domain 4), not because it feels more "architected."**

**Implementation — the three-question pressure test as an explicit design checklist:**

```python
from dataclasses import dataclass

@dataclass
class ProposedStep:
    name: str
    can_run_reliably_alone: bool
    has_independent_check: bool

def pressure_test_split(steps: list[ProposedStep]) -> list[str]:
    warnings = []
    for step in steps:
        if not step.can_run_reliably_alone:
            warnings.append(f"'{step.name}' fails reliability test — consider merging with a neighbor")
        if not step.has_independent_check:
            warnings.append(f"'{step.name}' has no independent check — cannot verify this step in isolation")
    # merge-loses-nothing test requires eval data (Domain 4), not just static inspection —
    # flagged here as a required follow-up rather than resolved statically
    if len(steps) > 1:
        warnings.append("Confirm against held-out eval: does collapsing any adjacent pair lose accuracy?")
    return warnings
```

**Common pitfalls:**
- Decomposing further because it "feels more architected" or more impressive in a design review, rather than because a held-out eval showed the coarser version failing.
- Applying the three-question test once at design time and never revisiting it — as production data accumulates, a split that was justified at launch may prove unnecessary (or an under-split monolith may prove too coarse) once real failure patterns are visible.

---

## Worked Architecture: Telecom Fault-Triage System

Pulling Domain 1 together end to end:

```
INPUT HANDLING → ROUTING (cheap-tier classifier) → PARALLEL VOTE: is this P1? (3x, majority)
                                                          │
                              ┌───────────────────────────┴───────────────────────────┐
                          P1 branch                                              not-P1 branch
                  AGENT: dynamic diagnostic loop                    FIXED CHAIN: extract → diagnose → route
                  (tools: signal check, firmware                    (known 4-step sequence)
                   lookup, ask customer)
                              │
                  escalate_to_engineer / resolve_ticket
                              │
                  OUTPUT: validated JSON, routing tag delivered
                              │
                  FEEDBACK: outcome → new eval case / prompt refinement
```

**Notice that only one branch of this system is an agent.** The rest is fixed workflow. This is the pattern industry keeps re-learning the hard way: agentic reasoning is a scalpel for the genuinely ambiguous 30–40% of cases, not a hammer for the whole pipeline.

**Implementation note connecting the worked example to the mechanisms above:** the routing stage (§2.3) uses a cheap classifier per the escalation ladder's "climb only when forced" rule (§2.4); the P1 vote uses parallelization-voting (§2.3) because false negatives on P1 outages carry a contractual performance cost (§1.2's SLA guardrail); only the diagnostic sub-step, where the next question genuinely depends on what was just learned, earns agent status (§2.4); and the feedback stage (§2.1) routes misrouted tickets into the eval set that governs whether this architecture is ever revisited (§3.5's "decompose further only if eval demonstrates it").

---

## Architecture & Implementation Best Practices

1. **Gate architecture kickoff on a completed discovery brief** (task statement, I/O schema, success metric) — do not allow a pattern or model choice to be discussed until all four discovery outputs exist.
2. **Assign one (rarely two) value pillar per project explicitly**, and derive the binding metric from it before any model-tier or pattern decision, rather than discovering the metric retroactively to justify a technology already chosen.
3. **Treat SLA status as a hard architecture constraint**, not a nice-to-have — a contractual latency penalty should mechanically rule out unbounded agentic loops in the design review, not just get flagged as a risk.
4. **Build the four-stage pipeline (input/processing/output/feedback) explicitly in code**, with schema validation at the output boundary and a routing function (not just a log sink) at the feedback boundary.
5. **Implement each workflow pattern as a distinct, testable function** — this makes the escalation ladder auditable: you can point to exactly which rung a given sub-decision occupies and why.
6. **Default every multi-agent design to explicit context-passing** — never rely on implicit inheritance; make the "what does this worker actually receive" question answerable by reading the dispatch code, not by assumption.
7. **Encode the four multi-agent failure-mode guardrails (spawn caps, non-overlapping scopes, timeouts, step caps) as structural code constraints**, not documentation — this is the same probabilistic-vs-deterministic distinction Domain 7 makes: guardrails that must hold belong in enforced code, not in a design doc.
8. **Re-run the three-question decomposition pressure test whenever production eval data accumulates** — granularity decisions made at launch are not permanent; revisit them against real failure data.

---

## Common Pitfalls and Limitations

- **Technology-first design** — naming a model, pattern, or vendor before discovery's four outputs (problem, stakeholders, constraints, success criteria) exist.
- **Cost-lens default** — framing every project as cost reduction regardless of the stakeholder's actual value pillar, producing a "successful" project against the wrong metric.
- **SLA/SLO conflation** — treating internal targets as contractual commitments (or vice versa), which either over-constrains architecture unnecessarily or under-protects a real commitment.
- **Calling any tool-using system "an agent"** — the deciding factor is control-flow ownership (your code vs. the model), not tool presence.
- **Reaching for orchestrator-workers when the subtask structure is already known** — this pays a runtime-planning token cost for flexibility the problem doesn't need.
- **Reaching for a full agent because one sub-decision is ambiguous** — the worked example shows only the genuinely ambiguous sub-step should be agentic; the rest should stay fixed workflow.
- **Assuming subagents inherit context** — the most common new-architect error in multi-agent design; context isolation is engineered, not automatic.
- **Sizing multi-agent decomposition by task size rather than separability** — shape decides, not size; tightly-coupled large tasks remain single-agent jobs.
- **Decomposing further because it "feels" more architected** — the only valid justification for finer granularity is a demonstrated eval improvement, not aesthetic preference in a design review.

---

## Official Documentation References

1. Anthropic Engineering, *"Building Effective Agents,"* Dec 19, 2024 — https://www.anthropic.com/engineering/building-effective-agents
2. Anthropic Engineering, *"How we built our multi-agent research system,"* Jun 13, 2025 — https://www.anthropic.com/engineering/built-multi-agent-research-system
3. Anthropic Engineering, *"Claude Code: Best practices for agentic coding,"* Apr 18, 2025 — https://www.anthropic.com/engineering/claude-code-best-practices
4. Google SRE Workbook, *"Implementing SLOs"* — https://sre.google/workbook/implementing-slos/
5. Anthropic Docs, *Agent SDK overview* — https://docs.claude.com/en/api/agent-sdk/overview
6. Anthropic Docs, *Messages API reference* (tool use, concurrency semantics) — https://docs.claude.com/en/api/messages

> **Version note:** Model names (e.g., `claude-sonnet-4-5`, `claude-haiku-4-5`) and Agent SDK APIs shown in code examples reflect the naming conventions documented at the time of writing. Verify current model identifiers and SDK method signatures against `docs.claude.com` before production use, since Anthropic revises model lineups and SDK interfaces over time.

---

## Domain Coverage / Traceability Matrix

| Source domain item | Implementation guide section(s) | Coverage |
|---|---|---|
| 1.1 Translation checklist (task/I-O/success target) | §1.1 | ✅ Full, with `DiscoveryBrief` code |
| 1.1 Industry practice / simplest-solution-first citation | §1.1 | ✅ Full |
| Prime anti-pattern table (agent/fine-tune/RAG symptoms) | §1.1 (anti-pattern subsection) | ✅ Full, with complexity-gate code |
| Discovery's four outputs gate design start | §1.1 | ✅ Full |
| 1.2 Five value pillars table | §1.2 | ✅ Full, with selector code |
| 1.2 Cost-lens-as-default-mistake | §1.2 | ✅ Full |
| 1.2 SLI/SLO/SLA definitions + reliable test | §1.2 | ✅ Full, with `SLADefinition` code |
| 1.2 Telecom triage worked example (Section 1) | §1.2 | ✅ Full, reproduced |
| 2.1 Four-stage anatomy diagram + failure table | §2.1 | ✅ Full, with pipeline code |
| 2.1 Feedback ≠ logging; three routing destinations | §2.1 | ✅ Full, with `feedback_loop` code |
| 2.2 Augmented LLM definition | §2.2 | ✅ Full |
| 2.2 Agent-vs-workflow control-flow test | §2.2 | ✅ Full |
| 2.2 RAG-fits-in-cached-prompt threshold note | §2.2 | ✅ Full (cross-ref to Domain 3) |
| 2.2 Read-only concurrent / state-changing sequential rule | §2.2 | ✅ Full, with tool schema example |
| 2.3 Five workflow patterns table | §2.3 | ✅ Full, all six variants (5 listed + voting) coded |
| 2.3 Orchestrator-vs-parallelization timing subtlety | §2.3 | ✅ Full |
| 2.4 Escalation ladder (4 rungs) | §2.4 | ✅ Full, with `select_rung` code |
| 2.4 Three costs of reaching for an agent | §2.4 | ✅ Full |
| 2.4 Telecom triage worked example (Section 2) | §2.4 | ✅ Full, reproduced |
| 3.1 Orchestrator's three jobs (plan/dispatch/synthesize) | §3.1 | ✅ Full, with TS SDK code |
| 3.1 Subagents inherit nothing by default | §3.1 | ✅ Full |
| 3.2 Four conditions where multi-agent earns its cost | §3.2 | ✅ Full, with justification code |
| 3.2 Shape-decides-not-size principle | §3.2 | ✅ Full |
| 3.2 Anthropic 2025 multi-agent benchmark (90%, 4x/15x tokens) | §3.2 | ✅ Full, cited with caveat |
| 3.3 Four failure modes table | §3.3 | ✅ Full, with unified guardrail code |
| 3.3 Delegation quality as root cause | §3.3 | ✅ Full |
| 3.4 Decomposition as substrate, not separate pattern | §3.4 | ✅ Full |
| 3.5 Coarsest-reliable-split principle | §3.5 | ✅ Full |
| 3.5 Three pressure-test questions | §3.5 | ✅ Full, with checklist code |
| Worked Architecture: Telecom Fault-Triage System (full diagram) | Worked Architecture section | ✅ Full, reproduced and annotated |
| Key Takeaways (7 bullets) | Distributed across Best Practices + Pitfalls | ✅ Full — every bullet maps to at least one best-practice/pitfall item |
| References 1–5 (source domain) | Official Documentation References | ✅ Full, retained and extended with Messages API reference |

**Coverage verification statement:** Every numbered subsection, table, diagram, and key-takeaway bullet in `domain-1-solution-design-architecture.md` is represented above with expanded technical detail, at least one implementation pattern, and a runnable Python and/or TypeScript example wherever the source item names or implies a concrete mechanism. No source item was merged, omitted, or skipped.
