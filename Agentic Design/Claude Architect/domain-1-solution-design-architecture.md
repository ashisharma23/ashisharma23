# Domain 1: Solution Design & Architecture
**Exam weight: 17% | Claude Certified Architect — Professional**

> The first engineering decision on any Claude project isn't a model choice — it's whether you understand the problem well enough to design against it. Domain 1 tests whether you can translate a vague business ask into a testable target, align that target to the right value pillar, and pick the coarsest architecture (single prompt → augmented LLM → workflow → agent) that still solves the problem reliably.

---

## 1. Business Translation & Value Alignment

### 1.1 From vague ask to designable problem

Every failed AI project can be traced to the same root cause: someone picked a technology before the outcome was defined. "Build us a chatbot" is not a requirement — it's a guess at a solution wearing the clothes of a requirement.

**Translation is complete only when you can state, in one sentence with a number in it:**
- The single decision or task being automated (not "help with support" — "classify inbound tickets into one of 12 routing queues")
- Inputs and outputs in concrete, testable formats (a JSON schema, not "a helpful response")
- A measurable success target set *during discovery*, before any architecture is chosen

**Industry practice:** treat this like a product spec, not a prompt. Anthropic's own guidance on agent design opens with exactly this discipline — start with the simplest solution that could work, and only add complexity (retrieval, tools, multi-step agentic loops) when the simpler option demonstrably fails against your success target.<sup>[1]</sup>

### The prime anti-pattern: technology-first design

| Symptom | What actually happened |
|---|---|
| "Let's build an agent" | A single prompt or augmented LLM call would have solved it |
| "We need fine-tuning" | Retrieval or a better system prompt was never tried |
| "Add a RAG pipeline" | The answer already fits in the prompt with caching |

Each of these adds cost, latency, and failure surface the requirement never asked for. Discovery produces four outputs — the problem, the stakeholders, the constraints, and the success criteria — and **design only starts once all four exist.** Naming a model or pattern during discovery is premature and almost always wrong, because it anchors the team before the constraints (especially latency and cost) are known.

### 1.2 Aligning to business value pillars

A target without a business frame is directionless. Every Claude solution should map to one (rarely more than two) of five value pillars, because each pillar implies a different metric — and optimizing the wrong metric produces a "successful" project that nobody wanted.

| Pillar | What it means | Binding metric | Common mistake |
|---|---|---|---|
| **Efficiency** | Same work, fewer resources/time | Time saved, tasks/hour vs. manual baseline | Confusing this with cost |
| **Transformation** | Doing what wasn't feasible before | Coverage a sample could never reach | Applying a cost lens to a moonshot |
| **Productivity** | Amplifying the people you have | Output per person — never headcount cuts | Framing as "replacing staff" |
| **Cost** | An existing outcome, cheaper | Spend per task, run-rate vs. prior approach | Using it as the default frame for every project |
| **Performance** | Meeting a latency/throughput/availability commitment | The SLI/SLO/SLA itself | Chasing accuracy when latency is the binding constraint |

The most common architectural mistake in industry is framing *every* project as a cost-reduction exercise. A transformation goal (e.g., analyzing 100% of customer calls instead of a 2% QA sample) justifies spend that a cost lens would reject outright. A performance goal (99.9% of API responses under 2 seconds) can force a *more* expensive path — smaller model, less retrieval, more caching — even though it looks like a step backward on "accuracy."

**SLI → SLO → SLA, in one sentence each:**
- **SLI** — the raw measured signal, no target attached (e.g., p95 latency = 1.8s)
- **SLO** — an internal target on that signal, with no external penalty if missed (e.g., "p95 < 2s")
- **SLA** — a customer contract with real consequences (credits, refunds) if missed

The single reliable test: *if missing it triggers a penalty, it's an SLA.* A hard latency SLA can rule out an agentic loop entirely, because every additional model turn adds unpredictable latency and multi-step loops widen the tail (p99), not just the average. This is why the binding metric should be decided **before** architecture — it directly moves three levers: model tier, retrieval depth, and workflow pattern.

**Example — telecom triage:** A carrier wants an AI system to pre-classify inbound network fault tickets.
- *Wrong frame:* "Reduce support headcount" (cost pillar, politically toxic, wrong metric)
- *Right frame:* "Efficiency — cut average time-to-correct-routing from 6 minutes (manual) to under 90 seconds, with routing accuracy ≥ 92% on a held-out ticket set, and P1 network outages escalated with ≤ 1% false-negative rate (performance pillar operating as a guardrail)."

This single reframe changes the architecture: a hard false-negative ceiling on P1 outages argues for a human-in-the-loop escalation gate (Domain 5), not full automation.

---

## 2. End-to-End & Pattern Architecture

### 2.1 The four-stage anatomy every architecture owns

A Claude system is not "the model call." It's four stages, and skipping any of them produces failures that don't throw exceptions — they just quietly degrade quality until someone notices in production.

```
┌──────────────┐    ┌──────────────────────┐    ┌──────────────────┐    ┌──────────────┐
│ INPUT         │ →  │ PROCESSING            │ →  │ OUTPUT            │ →  │ FEEDBACK      │
│ handling      │    │ (augmented LLM core:  │    │ delivery          │    │ loop          │
│ validate,     │    │  model + retrieval +  │    │ format, validate  │    │ capture       │
│ normalize,    │    │  tools + memory)      │    │ programmatically, │    │ outcomes,     │
│ enrich        │    │                       │    │ deliver           │    │ route back    │
└──────────────┘    └──────────────────────┘    └──────────────────┘    └──────┬───────┘
        ▲                                                                         │
        └─────────────────────────────────────────────────────────────────────────┘
```

| Stage | Silent failure if skipped |
|---|---|
| Input handling | Bad/malformed data quietly degrades every downstream answer |
| Processing | (this is the part everyone designs — rarely the weak point) |
| Output delivery | Confidently wrong results ship with no validation |
| Feedback | No path exists from production outcomes back to improvement |

**Feedback is not logging.** Logging ends at storage. A feedback loop routes captured outcomes into one of three actions: a new eval case, a prompt refinement, or a retraining dataset. In practice, teams over-reach for retraining (expensive, slow, usually unnecessary) and under-use the two cheap routes. The loop only closes when a change ships and is verified against a held-out case — not when a ticket is filed.

### 2.2 The processing core: augmented LLM

The **augmented LLM** — one model call enhanced with retrieval, tools, and memory, where *your code* owns the control flow — is the single building block every workflow and every agent is assembled from.<sup>[1]</sup> This matters for the exam and for real design: **tool use alone does not make a system agentic.** The question is not "does it call tools?" but "who decides the next step?"

- If your code decides → workflow (predictable, testable, cheap)
- If the model decides → agent (flexible, expensive, harder to test)

If the whole knowledge base fits in the prompt with caching, skip retrieval machinery entirely — RAG is a cost you pay for scale you may not have (see Domain 3 for the ~200k token threshold).

In the tool-use loop itself: **read-only tools have no side effects and can run concurrently; state-changing tools run one at a time, sequentially,** to preserve ordering safety. The loop repeats until the model stops requesting tool calls.

### 2.3 Anthropic's five workflow patterns

These are Anthropic's canonical fixed-topology patterns — all workflows, meaning your code (not the model) decides the sequence.<sup>[1]</sup>

| Pattern | Shape | When to use |
|---|---|---|
| **Prompt chaining** | Fixed sequence, programmatic gate between each step | Task decomposes into a known, ordered sequence |
| **Routing** | A classifier (can be a cheap model) sends input to a specialist path | Distinct input categories need distinct handling |
| **Parallelization — sectioning** | Independent subtasks run concurrently, results merged | Subtasks don't depend on each other |
| **Parallelization — voting** | Same task run N times, results aggregated for confidence | High-stakes classification needing consensus |
| **Orchestrator-workers** | Orchestrator decides subtasks *at runtime* from the specific input | Subtask structure can't be known in advance |
| **Evaluator-optimizer** | Generator + critic loop against clear judging criteria | Iterative refinement has a checkable target |

**The one subtlety the exam (and production) will test:** orchestrator-workers and parallelization share the same fan-out shape but differ on *timing*. Parallelization's subtasks are baked into your code before any call runs. The orchestrator decides the subtasks at runtime by reasoning over the input — which buys flexibility at real token cost, because that reasoning step is itself a model call.

### 2.4 The escalation ladder

Climb it only when forced, one rung at a time:

```
1. Single prompt
2. Augmented LLM (retrieval / tools / memory, one call)
3. Fixed workflow (chaining / routing / parallelization / orchestrator-workers / evaluator-optimizer)
4. Agent (model owns control flow) ← last resort
```

Agents bill you three ways every time you reach for one: **higher token cost** from the model's own reasoning loop, **added latency** from sequential dependent steps, and **the risk of compounding errors** across turns. This is not a warning against agents — it's a warning against reaching for one by default. Anthropic's own engineering guidance is explicit that the most successful production implementations use the simplest pattern that clears the bar, not the most sophisticated one available.<sup>[1]</sup>

**Worked example — the telecom triage system from Section 1:**
- Ticket text alone, single category → **routing** (cheap classifier model)
- "Is this a P1 outage?" needs consensus because false negatives are costly → **parallelization/voting** (3 independent classification passes, majority vote)
- "Diagnose *why* this specific customer is dropping WiFi calls" — the right next question depends on what the model just learned (firmware version → ask about driver; multi-device → ask about backhaul) → this is the one case in the whole system that earns an **agent**, because the path genuinely cannot be scripted in advance.

That last point is the core architectural signal covered next.

---

## 3. Multi-Agent Design & Decomposition

### 3.1 Orchestrator-worker topology

The lead agent owns exactly three jobs:

1. **Planning** — decompose the goal into scoped subtasks at runtime
2. **Dispatch** — spawn each worker with the specific tools, prompt, and context *that task* needs
3. **Synthesis** — merge worker results into one coherent answer

**Subagents inherit nothing by default.** This is the detail most new architects get wrong — every delegation must explicitly pass the task plus whatever context that task needs. Context isolation is engineered, not automatic: each worker runs in its own fresh conversation, only its final distilled message returns to the lead, and the lead's context grows by a summary, not the full transcript.<sup>[2]</sup>

### 3.2 When multi-agent earns its cost

Multi-agent pays off on work that is:
- **Breadth-first** (many independent angles to explore)
- **Genuinely separable** (workers don't need each other's intermediate state)
- **High-value** (worth the token multiplier)
- **Larger than one context window**

A single agent wins whenever steps depend on each other — most coding tasks, for instance, need a shared, consistent view of state, and constant context-syncing between agents would eat the parallelism gain alive. **Shape decides, not size.** A huge but tightly-coupled task is still a single-agent job.

**Anthropic's own benchmark** (2025, Opus-class lead orchestrating Sonnet-class subagents) reported the multi-agent system beating a single agent by **over 90% on an internal research evaluation**, with agents running at roughly 4× chat-level token consumption and multi-agent at roughly 15×. Anthropic reports that **token usage alone explained the majority of the performance variance** — treat this as directional evidence of the pattern's power, not a portable constant for every domain.<sup>[2]</sup>

### 3.3 Four failure modes to design against

| Failure mode | Cause | Design mitigation |
|---|---|---|
| **Over-spawning** | No cap on subagent count | Bound spawning explicitly; cap workers per orchestration |
| **Vague delegation** | Workers given ambiguous scope | Write explicit, non-overlapping task boundaries per worker |
| **Synchronous bottleneck** | Lead waits on the slowest worker | Set per-worker timeouts; design for partial-result synthesis |
| **Compounding errors** | Autonomy grows unchecked across turns | Validate intermediate results; bound step counts |

Nearly all of these trace back to one root cause: **delegation quality.** A well-scoped worker with a tight, non-overlapping mandate rarely causes the other three failures.

### 3.4 Decomposition: the substrate, not a pattern of its own

Decomposition isn't one more item on the pattern list — it's what every pattern *is*, in a different shape:
- Prompt chaining = decomposition into a fixed, ordered sequence
- Orchestrator-workers = decomposition at runtime, dispatched to subagents
- Even a single agent decomposes internally when it plans its own steps

### 3.5 Sizing the split correctly

**The right granularity is the coarsest split at which each subtask still runs reliably on its own and can be checked in isolation.**

- **Too coarse:** several hard steps bundled into one unpredictable failure you can't localize
- **Too fine:** trivial work gets its own handoff, and coordination overhead (latency, tokens, information loss at each seam) swamps any gain in control

**Pressure-test any proposed split with three questions:**
1. Can each subtask run reliably on its own?
2. Can you write a check that verifies each step independently?
3. Would merging two adjacent steps lose anything real?

If merging costs nothing, the split was too fine — collapse it. Default to a single, well-structured prompt and decompose only when it demonstrably improves reliability against your held-out eval set (Domain 4), not because it feels more "architected."

---

## Worked Architecture: Telecom Fault-Triage System

Pulling Domain 1 together end to end:

```
                         ┌─────────────────────────────────────────────┐
                         │              INPUT HANDLING                  │
                         │  validate ticket schema, normalize customer  │
                         │  ID, enrich with account + device history    │
                         └───────────────────┬───────────────────────────┘
                                              ▼
                         ┌─────────────────────────────────────────────┐
                         │        ROUTING (cheap-tier classifier)       │
                         │  category: outage | billing | provisioning   │
                         └───────────────────┬───────────────────────────┘
                                              ▼ (outage path)
                         ┌─────────────────────────────────────────────┐
                         │  PARALLEL VOTE: is this P1? (3x, majority)   │
                         └───────────────────┬───────────────────────────┘
                                 P1 ▼                    not-P1 ▼
                  ┌────────────────────────┐   ┌───────────────────────────┐
                  │  AGENT: dynamic         │   │  FIXED CHAIN: extract →   │
                  │  diagnostic loop        │   │  diagnose → route          │
                  │  (tools: signal check,  │   │  (known 4-step sequence)   │
                  │   firmware lookup,      │   └───────────────────────────┘
                  │   ask customer)         │
                  └───────────┬─────────────┘
                              ▼
                  ┌─────────────────────────┐
                  │ escalate_to_engineer /   │
                  │ resolve_ticket           │
                  └───────────┬─────────────┘
                              ▼
                  ┌─────────────────────────┐
                  │ OUTPUT: validated JSON,  │
                  │ routing tag delivered    │
                  └───────────┬─────────────┘
                              ▼
                  ┌─────────────────────────┐
                  │ FEEDBACK: outcome →      │
                  │ new eval case / prompt   │
                  │ refinement                │
                  └─────────────────────────┘
```

Notice that **only one branch of this system is an agent.** The rest is fixed workflow. This is the pattern industry keeps re-learning the hard way: agentic reasoning is a scalpel for the genuinely ambiguous 30–40% of cases, not a hammer for the whole pipeline.

---

## Key Takeaways

- Translation is done only when the target fits in one sentence with a number in it. Pick the architecture last, not first.
- Five value pillars, five different metrics. A cost lens on a transformation project (or vice versa) produces the wrong architecture.
- SLA = penalty on miss. Everything else is an SLI or SLO, not a contract.
- The augmented LLM is the atomic unit; workflows and agents are both built from it. The only axis that separates them is who owns control flow.
- Climb the escalation ladder one rung at a time — single prompt → augmented LLM → workflow → agent — and only when the simpler rung demonstrably fails your eval.
- In multi-agent systems, nothing is inherited automatically; context isolation and delegation scope are engineering decisions, not defaults.
- Right-size decomposition with three questions: reliable alone? checkable alone? does merging lose anything real?

---

## References

1. Anthropic Engineering, *"Building Effective Agents,"* Dec 19, 2024 — https://www.anthropic.com/engineering/building-effective-agents
2. Anthropic Engineering, *"How we built our multi-agent research system,"* Jun 13, 2025 — https://www.anthropic.com/engineering/built-multi-agent-research-system
3. Anthropic Engineering, *"Claude Code: Best practices for agentic coding,"* Apr 18, 2025 — https://www.anthropic.com/engineering/claude-code-best-practices
4. Google SRE Workbook, *"Implementing SLOs"* — https://sre.google/workbook/implementing-slos/
5. Anthropic Docs, *Agent SDK overview* — https://docs.claude.com/en/api/agent-sdk/overview
