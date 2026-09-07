# Domain 6: Stakeholder Communication & Lifecycle Management
**Exam weight: 14% | Claude Certified Architect — Professional**

> A correct architecture that nobody understood, agreed to, or can operate after handoff is not a shipped solution. Domain 6 covers the human side of the lifecycle: discovery, decision communication, expectation-setting, documentation, and the operate-and-iterate loop that keeps a system honest for years, not weeks.

---

## 1. Discovery & Communication

### 1.1 Structured discovery

Discovery establishes the **what** before design decides the **how** — this is the same discipline as Domain 1's translation step, applied at the organizational rather than the technical level. **The most common failure is the solution-first trap:** a stakeholder asks for "a chatbot" when the real need is faster ticket routing. Naming a model or pattern this early quietly skips the step of actually defining the outcome — and once a specific solution has been said out loud in a kickoff meeting, it's disproportionately hard to walk back later.

**Interview modes run on a spectrum:**

| Mode | Trade-off |
|---|---|
| **Structured** | Fixed script — comparable across interviews, but blind to anything you didn't think to ask |
| **Semi-structured** | Fixed core + open follow-ups — the workhorse; comparable *and* surfaces latent needs |
| **Unstructured** | Open conversation — rich early signal, hard to compare across stakeholders |

**Interview three distinct stakeholder types**, because each sees a different slice of the truth:
- **Sponsors** — own budget and outcomes, but rarely use the system day-to-day
- **End users** — know the real workflow and its edge cases
- **Affected parties downstream** — a customer, an auditor, a regulator; easy to forget in the room and costly to ignore later

### 1.2 Requirements taxonomy

| Type | Answers |
|---|---|
| **Functional** | What the system does |
| **Non-functional** | How well — the quality attributes that later become SLOs (Domain 1, §1.2) |
| **Constraints** | Fixed boundaries you cannot design around |

**AI projects add three qualifiers on top of the standard taxonomy:** data availability, feasibility, and tolerance for error. A requirement that's functionally reasonable can still be infeasible if the training or reference data simply doesn't exist, or if the domain's tolerance for a wrong answer is effectively zero (see Domain 5, §3.4's one-way-door test).

**When stakeholders conflict:** trace each ask back to its underlying outcome — competing asks often turn out to share the same outcome once you strip away the specific solution each stakeholder proposed. If a genuine conflict remains after that, **the budget owner decides.** Reconciliation is a decision, not a technical merge — don't try to average two incompatible requirements into a mushy middle that satisfies neither stakeholder.

### 1.3 The requirements document

Every classified requirement gets:
- A **stable ID**
- A tie to an **outcome**
- A **measurable acceptance criterion** agreed during discovery — these seed the evals directly (Domain 4, §1.1)
- Open questions captured in an **assumption log**, not silently resolved by guessing

**Traceability proves nothing dropped between discovery and design.** When an eval fails later, or a stakeholder asks "why doesn't it do X," the requirements doc is what lets you answer definitively rather than reconstructing intent from memory.

### 1.4 Communicating architectural decisions

**State quality attributes in customer terms with real numbers**, not engineering abstractions:
- Not "fast" → **"a reply in under two seconds for most requests"**
- Not "accurate" → **"the right resolution nine times in ten"**

Measure latency at a **high percentile**, not the average — this is the same p95/p99 discipline from Domain 4, §3.3, now framed for a non-technical audience.

### 1.5 The tiered-criteria decision matrix

Build a weighted decision matrix, but with one critical rule: **rank and weight the criteria in tiers so a top-tier factor cannot be outvoted by several minor ones.** A latency SLA that's a hard business requirement shouldn't lose to five soft "nice to have" criteria that happen to favor a different option on aggregate score.

**Lock criteria and weights *before* anyone sees the candidate options.** This single practice is what prevents weights from being quietly reverse-engineered after the fact to justify whichever option a stakeholder already preferred.

### 1.6 ATAM: evaluating architecture through scenarios

The **Architecture Tradeoff Analysis Method (ATAM)** evaluates a proposed architecture through concrete scenarios — use-case, growth, and exploratory — and surfaces four findings:

| Finding | Meaning |
|---|---|
| **Risk** | A design decision that could cause problems |
| **Non-risk** | A decision that looks risky but is well-supported |
| **Sensitivity point** | A design property one quality attribute leans on heavily |
| **Tradeoff point** | A sensitivity point *shared* by two or more attributes — tuning it lifts one and lowers another |

A **utility tree** ranks each scenario by business importance and technical difficulty; analysis effort goes to the high-importance, high-difficulty leaves — not evenly across every branch.

**Worked example of a tradeoff point:** the context-window choice between long-context loading and retrieval (Domain 3, §2.4) moves accuracy, latency, and cost **together** — it's not a decision you can optimize on one axis in isolation. That's exactly what makes it a tradeoff point rather than a simple sensitivity point.

### 1.7 The executive-translation pattern

Lead with the **business outcome** (revenue, cost, risk, speed to market), *then* name the technical choice, *then* name the tradeoff it accepts:

> "To hit the 2-second SLA on 95% of requests [outcome], we're routing routine classification to a smaller model and reserving the flagship for escalations [technical choice]. This means roughly 3% of edge-case tickets will need a second pass, adding a small amount of latency on exactly those cases [tradeoff accepted]."

**Present the accepted downside openly.** Hiding it wins approval in the short term but forfeits informed consent — and a sponsor who genuinely owns the downside can defend the decision later when someone questions it in a review, months after the architect who built it may have moved on.

---

## 2. Alignment & Documentation

### 2.1 SLI → SLO → SLA, with the error-budget math attached

Restating the chain from Domain 1 with the operational math that makes it actionable:

- **SLI** — the raw measured metric, no target, no promise
- **SLO** — an internal target on an SLI; missing it triggers only an engineering response
- **SLA** — a customer contract with penalties (credits, refunds) if missed

**The what-if-missed test** distinguishes them cleanly: the *consequence clause*, not the number itself, is what makes something a contract.

**Always set the internal SLO stricter than the external SLA** — promise 99.9%, target 99.95% internally — so drift gets caught while it's still an internal engineering problem, not a customer-facing breach. Error budgets derive from the **SLO**, never from the contract figure.

**For AI-specific SLIs**, split **time to first token** from **full completion time**, and define failure explicitly — a refusal or a malformed output both count as failures for budget purposes, even though neither is a traditional "error."

### 2.2 Error budget mechanics

The error budget is:

```
error_budget = 1 − SLO_target   (over a rolling window)
```

A 99.9% objective leaves **0.1% over a 30-day month** to spend.

**Burn rate** = error rate ÷ (1 − SLO). A burn rate of 1 exhausts the entire monthly budget exactly at the window's end. Google's SRE practice recommends **multi-window burn-rate alerts**<sup>[1]</sup>:

| Window | Burn-rate threshold | Response |
|---|---|---|
| 1 hour | > 14.4 | Page immediately |
| 6 hours | > 6 | Ticket |
| 3 days | > 1 | Review |

**The error-budget policy is an automated release gate agreed in advance:** over budget, all non-critical releases freeze except P0 and security fixes, until the service is back within SLO. An incident that burns over 20% of the monthly budget in one event **mandates a postmortem with at least one P0 action item.** The freeze does not apply when the breach is purely an external dependency's fault — the policy governs what your team controls, not what a third party broke.

### 2.3 Feedback cadence matched to stakeholder need

Match how fast you report to how fast each audience actually needs signal:
- **Weekly** — operational reviews (the team building and running the system)
- **Monthly** — business reviews (sponsors, budget owners)
- **Continuous** — an incident channel (everyone, only when something is actively breaking)

Report proactively against the **versioned SLA table**, align expectations and the escalation path *before* launch — not reactively after the first miss — and frame a burned error budget as **the policy working as designed**, not as the team failing. This reframe matters: a well-functioning error-budget policy is supposed to trigger a freeze occasionally. If it never does, either the SLO is too loose or nobody is enforcing it.

### 2.4 Architecture Decision Records (ADRs)

An ADR captures **one architecturally significant decision** in five fixed parts: **Title, Status, Context, Decision, Consequences.**

Reserve records for choices that shape structure, interfaces, or dependencies and are **costly to reverse** — skip reversible local details that don't warrant the overhead. The test is significance, not difficulty; a hard-to-implement but easily-reversible choice doesn't need an ADR, while an easy-to-implement but hard-to-reverse choice does.

**The Consequences section must list all results, upside and downside together.** A record showing only benefits is incomplete by design — naming the accepted downside explicitly is what makes the tradeoff honest and reviewable later (this directly mirrors §1.7's executive-translation pattern, now in written form).

**Status lifecycle:** `proposed → accepted → deprecated → superseded`. Once accepted, a record is **immutable** — never edit it in place. Leave it intact, mark it superseded by the new record's number, and write a fresh record. The log stays **append-only** so the chain of reasoning stays readable years later, including *why* an earlier decision made sense given what was known at the time.

### 2.5 C4 model and arc42

**C4 nests three maintained views for three distinct audiences:**

| View | Shows | Audience |
|---|---|---|
| **Context** | The system as one box among users and external systems | Executives |
| **Container** | Deployable pieces and how they talk to each other | The architect's working view |
| **Component** | Building blocks inside one container | Implementers |

**arc42** is a document *template* — it tells you what to write down. **C4** tells you *how to draw* the structure. The two pair well together rather than competing: arc42 gives you the sections (context, constraints, solution strategy, quality requirements...); C4 gives you the diagramming convention to fill several of those sections consistently.

### 2.6 A real handoff transfers ownership, not just files

A complete handoff carries four things:
1. **Decision log** — the *why*, so the new team knows what not to accidentally undo (this is your ADR chain)
2. **Implementation guidance** — the *how*: interfaces, configuration, constraints
3. **A runbook** — deploy, monitor, respond, roll back, written specifically for the on-call team, not the original architect
4. **Acceptance criteria** — traced straight back to the requirements doc in §1.3

**Shipping code and a slide deck moves files, not ownership.** A runbook that assumes the reader already knows the system's internals the way its original author did is not a handoff artifact — it's a note to a future version of yourself.

---

## 3. Lifecycle Management

### 3.1 The lifecycle is a cycle, not a line

```
discovery → design → build → handoff → monitoring → iteration → (big finding reopens discovery)
```

**Phase gates are go/no-go checkpoints with exit criteria, not calendar dates.** The gate into production is the strictest of all: evals passing on a dataset that genuinely mirrors real traffic (Domain 4, §1.3), monitoring and SLOs wired up **before** launch (not "we'll add observability in the next sprint"), and a rollback path that's actually been exercised, not just documented.

### 3.2 Environment progression and pilot rollout

Systems travel: **development** (test data only) → **staging** (production-like, safe to break) → **production** (live traffic, where most of the lifecycle actually plays out).

**Pilot with two or three champion teams — never just one** — so early findings are validated signal, not a fluke tied to one team's unusual workflow. Then sequence surfaces into production **in risk order**: low-risk, high-reuse surfaces first, with rollback ready at every step to keep the blast radius small if something goes wrong.

### 3.3 Why offline evals are never enough alone

Pre-deployment evals (Domain 4) **cannot observe findings that only appear live**: real traffic drifts from the distribution you tested against, and rare inputs, genuine load, and adversarial use only reliably surface at production volume.

**Monitor four signal families against SLOs continuously:**
1. Quality
2. Latency at the percentiles users actually feel (p95/p99, not the mean)
3. Cost per request
4. Safety signals — refusals, injection attempts (Domain 5)

### 3.4 Drift vs. regression

| | Cause | Fix |
|---|---|---|
| **Drift** | The world moved — inputs shifted away from what you tested | Update evals, retrieval, or scope |
| **Regression** | A change *you* shipped made outputs worse | Caught by a frozen regression test (Domain 4, §2.4) |

Both show up on a dashboard as **falling quality** — they look identical from the metric alone. **Naming the cause wrong means fixing the wrong thing**: rolling back a regression that's actually drift fixes nothing; retraining or re-scoping for drift that's actually a regression wastes effort chasing a moving target that never moved.

### 3.5 Iteration is evidence-triggered, not calendar-driven

Iteration fires on **an SLO breach, detected drift, or stakeholder feedback — never on calendar churn** ("it's been a quarter, let's improve the prompt"). Diagnose the cause first (Domain 4, §2.7's four origins), and reach for **prompt and retrieval changes before the model** — the model swap is the last and heaviest lever, exactly as in Domain 3's accuracy-latency lever ordering.

**Deprecation is a planned phase, not a silent shutoff:** stakeholder notice, a migration path defined *before* anything is pulled, and records preserved for the successor system or a future audit. A deprecated system that simply stops responding one day is an incident, not a lifecycle phase done correctly.

---

## Worked Example: Lifecycle Artifacts for the Telecom Triage System

Tying Domains 1–6 together as a single delivered package:

```
DISCOVERY
 ├── Requirements doc (IDs REQ-001..REQ-047, each with acceptance criteria)
 └── Assumption log: "Firmware-version API latency assumed <200ms — unverified"

DESIGN
 ├── ADR-001: Chose routing + parallel-vote + agentic-diagnostic hybrid
 │    over full-agent design. Consequences: cheaper/faster for 70% of
 │    tickets; the 30% diagnostic path costs more in latency — accepted
 │    because P1 false-negative rate was the binding constraint.
 ├── C4 Container view: routing classifier | vote layer | diagnostic
 │    agent | ticketing system, shared in the architecture review
 └── Tiered decision matrix: latency (tier 1) > cost (tier 2) > model
      elegance (tier 3), locked before vendor options were compared

HANDOFF
 ├── Runbook: deploy / rollback / on-call escalation path
 ├── SLA table: 99.9% availability (external), 99.95% SLO (internal)
 └── Named owner: on-call lead owns escalated-ticket outcomes (Dom. 5)

MONITOR → ITERATE
 ├── Weekly ops review (latency, cost, error budget burn)
 ├── Monthly business review (routing accuracy vs. manual baseline)
 └── Drift check: quarterly re-score of live sample vs. launch baseline;
      confirmed DRIFT (new device firmware category) not regression →
      retrieval corpus updated, prompt untouched
```

---

## Key Takeaways

- Discovery answers "what," design answers "how" — naming a solution during discovery is the most common and hardest-to-reverse mistake.
- Lock decision-matrix criteria and weights before anyone sees the options, and use tiers so no minor criterion can outvote a binding one.
- ATAM's tradeoff points are shared sensitivities across multiple quality attributes — the context-window choice is the canonical example.
- Lead every stakeholder conversation with the business outcome, then the technical choice, then the accepted downside — never hide the downside.
- SLO stricter than SLA, always. Error budget derives from the SLO. A burned budget is the policy working, not the team failing.
- ADRs are immutable once accepted; supersede, never edit in place. The Consequences section must show the downside, not just the win.
- A handoff transfers ownership via decision log + implementation guidance + runbook + acceptance criteria — code and a deck alone move files, not accountability.
- Drift and regression look identical on a dashboard but need opposite fixes — diagnose which one before touching anything.

---

## References

1. Google SRE Workbook, *"Alerting on SLOs"* — https://sre.google/workbook/alerting-on-slos/
2. Google SRE Book, *"Service Level Objectives"* — https://sre.google/sre-book/service-level-objectives/
3. C4 Model, *official documentation* — https://c4model.com/
4. arc42, *official template documentation* — https://arc42.org/overview
5. Michael Nygard, *"Documenting Architecture Decisions"* (originating ADR format) — https://cognitect.com/blog/2011/11/15/documenting-architecture-decisions
6. Software Engineering Institute (Carnegie Mellon), *"Architecture Tradeoff Analysis Method (ATAM)"* — https://www.sei.cmu.edu/library/architecture-tradeoff-analysis-method-collection/
