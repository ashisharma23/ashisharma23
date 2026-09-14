# Domain Implementation Reference Guide — Domain 6: Stakeholder Communication & Lifecycle Management

**Source domain:** `domain-6-stakeholder-communication-lifecycle.md` (CCAR-P, exam weight 14%)
**Guide type:** Implementation Reference (technical depth, code examples, API/config detail, process artifacts)
**Audience:** Solution architects responsible for discovery, stakeholder alignment, documentation, and the operate-and-iterate lifecycle of production Claude systems.

> This guide preserves the full structure of the source domain (1:1 section mapping) and expands every item into: what it is, why it matters, the underlying mechanism/process, implementation patterns, pitfalls, and runnable Python examples plus concrete document/artifact templates. A full traceability matrix appears at the end.

---

## Table of Contents

1. [Domain Overview](#domain-overview)
2. [Section 1: Discovery & Communication](#section-1-discovery--communication)
   - 1.1 Structured discovery
   - 1.2 Requirements taxonomy
   - 1.3 The requirements document
   - 1.4 Communicating architectural decisions
   - 1.5 The tiered-criteria decision matrix
   - 1.6 ATAM: evaluating architecture through scenarios
   - 1.7 The executive-translation pattern
3. [Section 2: Alignment & Documentation](#section-2-alignment--documentation)
   - 2.1 SLI → SLO → SLA with error-budget math
   - 2.2 Error budget mechanics
   - 2.3 Feedback cadence matched to stakeholder need
   - 2.4 Architecture Decision Records (ADRs)
   - 2.5 C4 model and arc42
   - 2.6 A real handoff transfers ownership
4. [Section 3: Lifecycle Management](#section-3-lifecycle-management)
   - 3.1 The lifecycle is a cycle, not a line
   - 3.2 Environment progression and pilot rollout
   - 3.3 Why offline evals are never enough alone
   - 3.4 Drift vs. regression
   - 3.5 Iteration is evidence-triggered, not calendar-driven
5. [Worked Example: Lifecycle Artifacts for the Telecom Triage System](#worked-example-lifecycle-artifacts-for-the-telecom-triage-system)
6. [Architecture & Implementation Best Practices](#architecture--implementation-best-practices)
7. [Common Pitfalls and Limitations](#common-pitfalls-and-limitations)
8. [Official Documentation References](#official-documentation-references)
9. [Domain Coverage / Traceability Matrix](#domain-coverage--traceability-matrix)

---

## Domain Overview

Domain 6's opening claim frames the whole domain: **a correct architecture that nobody understood, agreed to, or can operate after handoff is not a shipped solution.** Every other domain (1–5, 7) produces a technically sound system; Domain 6 is what makes that system survivable across organizational boundaries — from the initial discovery conversation, through architectural decisions that must be defensible months later, through a handoff that transfers genuine ownership rather than just files, and through a lifecycle that keeps running honestly for years.

Four threads recur throughout:

1. **Decisions must be locked before they can be gamed.** Discovery must complete before a solution is named; decision-matrix weights must be locked before candidates are seen; SLO targets must be set before a burned budget is explained away. Every one of these guards against the same failure: reverse-engineering a process to justify a conclusion already reached.
2. **Documentation exists to answer "why" months or years later, not just to describe "what" today.** ADRs, the requirements doc, and runbooks are all optimized for a reader who wasn't in the room — including, often, a future version of the same architect.
3. **Downsides must be stated explicitly, never hidden.** The executive-translation pattern (§1.7) and the ADR Consequences section (§2.4) both exist because a record or a pitch that shows only upside is incomplete by design and cannot be defended later.
4. **Two things that look identical on a dashboard can require opposite fixes.** Drift vs. regression (§3.4) is the domain's sharpest instance of this principle, and getting the diagnosis wrong wastes effort chasing the wrong fix entirely.

---

## Section 1: Discovery & Communication

### 1.1 Structured discovery

**What it is.** Discovery establishes the **what** before design decides the **how** — the same discipline as Domain 1's translation step, applied at the organizational rather than the technical level. **The most common failure is the solution-first trap:** a stakeholder asks for "a chatbot" when the real need is faster ticket routing. Naming a model or pattern this early quietly skips the step of actually defining the outcome — and once a specific solution has been said out loud in a kickoff meeting, it's disproportionately hard to walk back later.

**Interview modes run on a spectrum:**

| Mode | Trade-off |
|---|---|
| **Structured** | Fixed script — comparable across interviews, but blind to anything you didn't think to ask |
| **Semi-structured** | Fixed core + open follow-ups — the workhorse; comparable *and* surfaces latent needs |
| **Unstructured** | Open conversation — rich early signal, hard to compare across stakeholders |

**Interview three distinct stakeholder types**, because each sees a different slice of the truth:
- **Sponsors** — own budget and outcomes, but rarely use the system day-to-day.
- **End users** — know the real workflow and its edge cases.
- **Affected parties downstream** — a customer, an auditor, a regulator; easy to forget in the room and costly to ignore later.

**Implementation — a structured discovery-interview tracker that enforces stakeholder-type coverage and flags premature solution-naming (Python):**

```python
from dataclasses import dataclass, field
from enum import Enum

class StakeholderType(Enum):
    SPONSOR = "sponsor"
    END_USER = "end_user"
    AFFECTED_PARTY = "affected_party"

class InterviewMode(Enum):
    STRUCTURED = "structured"
    SEMI_STRUCTURED = "semi_structured"
    UNSTRUCTURED = "unstructured"

SOLUTION_KEYWORDS = {"chatbot", "agent", "fine-tune", "rag pipeline", "model"}  # premature solution-naming flags

@dataclass
class DiscoveryInterview:
    stakeholder_name: str
    stakeholder_type: StakeholderType
    mode: InterviewMode
    raw_notes: str

    def flag_solution_first_trap(self) -> list[str]:
        """The most common discovery failure — catch it at capture time, not
        after a specific solution has already been said aloud in a kickoff."""
        return [kw for kw in SOLUTION_KEYWORDS if kw in self.raw_notes.lower()]

@dataclass
class DiscoveryTracker:
    interviews: list[DiscoveryInterview] = field(default_factory=list)

    def coverage_check(self) -> dict:
        """All three stakeholder types must be represented — each sees a
        different slice of the truth and none can substitute for another."""
        covered = {i.stakeholder_type for i in self.interviews}
        missing = set(StakeholderType) - covered
        return {"missing_stakeholder_types": [m.value for m in missing], "fully_covered": not missing}

tracker = DiscoveryTracker()
tracker.interviews.append(DiscoveryInterview(
    stakeholder_name="VP Support Ops", stakeholder_type=StakeholderType.SPONSOR,
    mode=InterviewMode.SEMI_STRUCTURED, raw_notes="Wants faster ticket routing, current avg 6 min."))
print(tracker.coverage_check())  # flags missing end_user and affected_party interviews
```

**Common pitfalls:**
- Letting a sponsor name a specific technology ("build us a chatbot") pass unchallenged in a kickoff — once said aloud, it's disproportionately hard to walk back, even after discovery reveals a simpler fix would serve the actual outcome.
- Interviewing only sponsors because they're easiest to schedule — end users and downstream affected parties surface entirely different, equally necessary information.
- Using a rigid structured script for every stakeholder regardless of how much is already known — this misses latent needs a semi-structured approach would have surfaced.

---

### 1.2 Requirements taxonomy

**What it is.**

| Type | Answers |
|---|---|
| **Functional** | What the system does |
| **Non-functional** | How well — the quality attributes that later become SLOs (Domain 1 §1.2) |
| **Constraints** | Fixed boundaries you cannot design around |

**Why it matters — AI projects add three qualifiers on top of the standard taxonomy:** data availability, feasibility, and tolerance for error. A requirement that's functionally reasonable can still be infeasible if the training or reference data simply doesn't exist, or if the domain's tolerance for a wrong answer is effectively zero (Domain 5 §3.4's one-way-door test).

**When stakeholders conflict:** trace each ask back to its underlying outcome — competing asks often turn out to share the same outcome once you strip away the specific solution each stakeholder proposed. If a genuine conflict remains after that, **the budget owner decides.** Reconciliation is a decision, not a technical merge — don't try to average two incompatible requirements into a mushy middle that satisfies neither stakeholder.

**Implementation — a requirement classifier with AI-specific qualifiers and a conflict-tracing helper (Python):**

```python
from dataclasses import dataclass
from enum import Enum

class RequirementType(Enum):
    FUNCTIONAL = "functional"
    NON_FUNCTIONAL = "non_functional"
    CONSTRAINT = "constraint"

@dataclass
class AIRequirement:
    req_id: str
    requirement_type: RequirementType
    description: str
    data_available: bool          # AI-specific qualifier
    technically_feasible: bool    # AI-specific qualifier
    error_tolerance: str          # "zero" | "low" | "moderate" | "high" — AI-specific qualifier

    def is_viable(self) -> dict:
        blockers = []
        if not self.data_available:
            blockers.append("No training/reference data available for this requirement.")
        if not self.technically_feasible:
            blockers.append("Not technically feasible with current approach.")
        if self.error_tolerance == "zero":
            blockers.append("Zero error tolerance — apply Domain 5's one-way-door test before proceeding.")
        return {"viable": not blockers, "blockers": blockers}

def trace_conflict_to_outcome(ask_a: str, outcome_a: str, ask_b: str, outcome_b: str,
                                budget_owner_decision: str = None) -> dict:
    """Reconciliation is a DECISION, not a technical merge."""
    if outcome_a == outcome_b:
        return {"resolution": "shared_outcome", "detail":
                f"'{ask_a}' and '{ask_b}' both serve outcome '{outcome_a}' — reconcile the SOLUTION, not the outcome."}
    if budget_owner_decision:
        return {"resolution": "budget_owner_decided", "chosen": budget_owner_decision}
    return {"resolution": "unresolved", "detail": "Genuine conflict remains — escalate to budget owner."}
```

**Common pitfalls:**
- Treating a requirement as purely functional/non-functional without checking the AI-specific qualifiers — a functionally reasonable ask can still be infeasible due to missing data or unacceptable error tolerance.
- Averaging two conflicting stakeholder requirements into a compromise that satisfies neither — reconciliation is a decision made by the budget owner, not a blend.
- Treating every stated conflict as genuine before tracing it back to the underlying outcome — many "conflicts" turn out to be different proposed solutions to the same shared goal.

---

### 1.3 The requirements document

**What it is.** Every classified requirement gets:
- A **stable ID**.
- A tie to an **outcome**.
- A **measurable acceptance criterion** agreed during discovery — these seed the evals directly (Domain 4 §1.1).
- Open questions captured in an **assumption log**, not silently resolved by guessing.

**Why it matters — traceability proves nothing dropped between discovery and design.** When an eval fails later, or a stakeholder asks "why doesn't it do X," the requirements doc is what lets you answer definitively rather than reconstructing intent from memory.

**Implementation — a requirements document schema with traceability (Python):**

```python
from dataclasses import dataclass, field

@dataclass
class RequirementRecord:
    req_id: str                    # stable ID, e.g., "REQ-014"
    outcome: str                   # the business outcome this ties to
    description: str
    acceptance_criterion: str      # measurable — directly seeds a Domain 4 eval case
    source_stakeholder: str

@dataclass
class AssumptionLogEntry:
    assumption_id: str
    statement: str                 # e.g., "Firmware-version API latency assumed <200ms — unverified"
    status: str = "open"           # open | verified | invalidated
    owner: str = None

@dataclass
class RequirementsDocument:
    requirements: list[RequirementRecord] = field(default_factory=list)
    assumptions: list[AssumptionLogEntry] = field(default_factory=list)

    def to_eval_seed_cases(self) -> list[dict]:
        """Acceptance criteria seed evals DIRECTLY — this is the concrete
        mechanism connecting Domain 6's discovery to Domain 4's eval suite."""
        return [{"req_id": r.req_id, "eval_target": r.acceptance_criterion} for r in self.requirements]

    def why_doesnt_it_do_x(self, x_description: str) -> str:
        """Traceability answer — definitively, from the document, not from memory."""
        matches = [r for r in self.requirements if x_description.lower() in r.description.lower()]
        if not matches:
            return f"No requirement ties to '{x_description}' — this was never in scope per REQ doc."
        return f"Covered by {matches[0].req_id}, acceptance criterion: {matches[0].acceptance_criterion}"
```

**Common pitfalls:**
- Silently resolving an open question with a guess instead of logging it in the assumption log — this hides a decision that should have been surfaced and verified.
- Writing requirements without a measurable acceptance criterion — these cannot seed the eval suite (Domain 4 §1.1) and leave "done" undefined.
- Treating the requirements doc as a one-time artifact rather than the traceability record that answers "why doesn't it do X" definitively months later.

---

### 1.4 Communicating architectural decisions

**What it is.** **State quality attributes in customer terms with real numbers**, not engineering abstractions:
- Not "fast" → **"a reply in under two seconds for most requests."**
- Not "accurate" → **"the right resolution nine times in ten."**

**Why it matters.** Measure latency at a **high percentile**, not the average — this is the same p95/p99 discipline from Domain 4 §3.3, now framed for a non-technical audience.

**Implementation — a translation layer from engineering metrics to customer-facing language (Python):**

```python
def translate_to_customer_terms(metric_name: str, percentile_value: float, unit: str) -> str:
    """Never present an average to a non-technical stakeholder as 'typical
    performance' — the percentile is what the customer actually experiences."""
    translations = {
        "latency_p95": f"a reply in under {percentile_value}{unit} for most requests",
        "accuracy": f"the right resolution {round(percentile_value * 10)} times in ten",
    }
    return translations.get(metric_name, f"{metric_name}: {percentile_value}{unit} (p95, not average)")

print(translate_to_customer_terms("latency_p95", 2, "s"))
# "a reply in under 2s for most requests"
```

**Common pitfalls:**
- Reporting a mean latency to a non-technical stakeholder as if it represented typical experience — a healthy mean can sit on top of a slow tail that's quietly ruining the experience for the unluckiest fraction of users (Domain 4 §3.3).
- Using engineering jargon ("p95 latency," "F1 score") in a customer-facing conversation instead of translating to concrete, numeric, plain-language terms.

---

### 1.5 The tiered-criteria decision matrix

**What it is.** Build a weighted decision matrix, but with one critical rule: **rank and weight the criteria in tiers so a top-tier factor cannot be outvoted by several minor ones.** A latency SLA that's a hard business requirement shouldn't lose to five soft "nice to have" criteria that happen to favor a different option on aggregate score.

**Why it matters.** **Lock criteria and weights *before* anyone sees the candidate options.** This single practice is what prevents weights from being quietly reverse-engineered after the fact to justify whichever option a stakeholder already preferred.

**Implementation — a tiered decision matrix that structurally prevents a top-tier factor from being outvoted (Python):**

```python
from dataclasses import dataclass

@dataclass
class TieredCriterion:
    name: str
    tier: int          # 1 = must-pass/binding, 2 = important, 3 = nice-to-have
    weight: float

@dataclass
class DecisionMatrix:
    criteria: list[TieredCriterion]
    locked: bool = False

    def lock(self):
        """MUST be called before any candidate option is shown — this is what
        prevents weights from being reverse-engineered to fit a preference."""
        self.locked = True

    def score_option(self, option_name: str, criterion_scores: dict[str, float]) -> dict:
        if not self.locked:
            raise RuntimeError("Cannot score: criteria/weights not locked before viewing options.")

        # Tier 1 criteria act as hard gates — cannot be outvoted by aggregate score
        tier1 = [c for c in self.criteria if c.tier == 1]
        for c in tier1:
            if criterion_scores.get(c.name, 0) < 0.6:  # example gate threshold
                return {"option": option_name, "status": "ELIMINATED",
                         "reason": f"Failed tier-1 binding criterion: {c.name}"}

        weighted_total = sum(criterion_scores.get(c.name, 0) * c.weight for c in self.criteria)
        return {"option": option_name, "status": "SCORED", "weighted_total": weighted_total}

matrix = DecisionMatrix(criteria=[
    TieredCriterion(name="latency_sla_compliance", tier=1, weight=0.4),
    TieredCriterion(name="cost_efficiency", tier=2, weight=0.35),
    TieredCriterion(name="model_elegance", tier=3, weight=0.25),
])
matrix.lock()  # locked BEFORE candidates are compared
```

**Common pitfalls:**
- Building the decision matrix after candidate options are already known, then tuning weights until the preferred option wins — this is exactly the reverse-engineering the lock-before-viewing rule exists to prevent.
- Using a flat, untiered weighted sum where five minor criteria can mathematically outvote one binding requirement.

---

### 1.6 ATAM: evaluating architecture through scenarios

**What it is.** The **Architecture Tradeoff Analysis Method (ATAM)** evaluates a proposed architecture through concrete scenarios — use-case, growth, and exploratory — and surfaces four findings:

| Finding | Meaning |
|---|---|
| **Risk** | A design decision that could cause problems |
| **Non-risk** | A decision that looks risky but is well-supported |
| **Sensitivity point** | A design property one quality attribute leans on heavily |
| **Tradeoff point** | A sensitivity point *shared* by two or more attributes — tuning it lifts one and lowers another |

**Why it matters.** A **utility tree** ranks each scenario by business importance and technical difficulty; analysis effort goes to the high-importance, high-difficulty leaves — not evenly across every branch.

**Worked example of a tradeoff point:** the context-window choice between long-context loading and retrieval (Domain 3 §2.4) moves accuracy, latency, and cost **together** — it's not a decision you can optimize on one axis in isolation. That's exactly what makes it a tradeoff point rather than a simple sensitivity point.

**Implementation — a utility tree and ATAM finding classifier (Python):**

```python
from dataclasses import dataclass
from enum import Enum

class ATAMFinding(Enum):
    RISK = "risk"
    NON_RISK = "non_risk"
    SENSITIVITY_POINT = "sensitivity_point"
    TRADEOFF_POINT = "tradeoff_point"

@dataclass
class Scenario:
    description: str
    business_importance: int   # 1 (low) - 5 (high)
    technical_difficulty: int  # 1 (low) - 5 (high)

    @property
    def priority_score(self) -> int:
        return self.business_importance * self.technical_difficulty

def build_utility_tree(scenarios: list[Scenario]) -> list[Scenario]:
    """Analysis effort goes to high-importance, high-difficulty leaves —
    NOT evenly across every branch."""
    return sorted(scenarios, key=lambda s: s.priority_score, reverse=True)

def classify_finding(affects_multiple_quality_attributes: bool, is_well_supported: bool,
                      could_cause_problems: bool) -> ATAMFinding:
    if affects_multiple_quality_attributes:
        return ATAMFinding.TRADEOFF_POINT  # e.g., RAG-vs-long-context: accuracy+latency+cost together
    if is_well_supported and could_cause_problems:
        return ATAMFinding.NON_RISK  # looks risky, but well-supported
    if could_cause_problems:
        return ATAMFinding.RISK
    return ATAMFinding.SENSITIVITY_POINT

# Worked tradeoff-point example
context_window_decision = classify_finding(
    affects_multiple_quality_attributes=True,  # moves accuracy, latency, AND cost together
    is_well_supported=True, could_cause_problems=False,
)
assert context_window_decision == ATAMFinding.TRADEOFF_POINT
```

**Common pitfalls:**
- Spreading ATAM analysis effort evenly across every scenario in the utility tree instead of concentrating on high-importance, high-difficulty leaves — this wastes scarce analysis time on low-value branches.
- Misclassifying a tradeoff point as a simple sensitivity point — missing that a design property is shared across multiple quality attributes means optimizing it in isolation will silently move other attributes too.

---

### 1.7 The executive-translation pattern

**What it is.** Lead with the **business outcome** (revenue, cost, risk, speed to market), *then* name the technical choice, *then* name the tradeoff it accepts:

> "To hit the 2-second SLA on 95% of requests [outcome], we're routing routine classification to a smaller model and reserving the flagship for escalations [technical choice]. This means roughly 3% of edge-case tickets will need a second pass, adding a small amount of latency on exactly those cases [tradeoff accepted]."

**Why it matters.** **Present the accepted downside openly.** Hiding it wins approval in the short term but forfeits informed consent — and a sponsor who genuinely owns the downside can defend the decision later when someone questions it in a review, months after the architect who built it may have moved on.

**Implementation — an executive-translation message builder that structurally enforces all three parts (Python):**

```python
from dataclasses import dataclass

@dataclass
class ExecutiveTranslation:
    business_outcome: str    # MUST come first
    technical_choice: str
    accepted_tradeoff: str   # MUST be present — never omitted

    def render(self) -> str:
        if not self.accepted_tradeoff:
            raise ValueError("Hiding the tradeoff forfeits informed consent — it must be stated.")
        return (f"To {self.business_outcome}, {self.technical_choice}. "
                f"This means {self.accepted_tradeoff}.")

pitch = ExecutiveTranslation(
    business_outcome="hit the 2-second SLA on 95% of requests",
    technical_choice="we're routing routine classification to a smaller model and reserving "
                      "the flagship for escalations",
    accepted_tradeoff="roughly 3% of edge-case tickets will need a second pass, adding a small "
                       "amount of latency on exactly those cases",
)
print(pitch.render())
```

**Common pitfalls:**
- Leading with the technical choice ("we're using model routing") instead of the business outcome — this loses the audience before the tradeoff is even reached.
- Omitting the accepted downside to win faster approval — this forfeits informed consent and leaves no sponsor equipped to defend the decision when it's questioned later.

---

## Section 2: Alignment & Documentation

### 2.1 SLI → SLO → SLA, with the error-budget math attached

**What it is.** Restating the chain from Domain 1 with the operational math that makes it actionable:

- **SLI** — the raw measured metric, no target, no promise.
- **SLO** — an internal target on an SLI; missing it triggers only an engineering response.
- **SLA** — a customer contract with penalties (credits, refunds) if missed.

**The what-if-missed test** distinguishes them cleanly: the *consequence clause*, not the number itself, is what makes something a contract.

**Why it matters.** **Always set the internal SLO stricter than the external SLA** — promise 99.9%, target 99.95% internally — so drift gets caught while it's still an internal engineering problem, not a customer-facing breach. Error budgets derive from the **SLO**, never from the contract figure.

**For AI-specific SLIs**, split **time to first token** from **full completion time**, and define failure explicitly — a refusal or a malformed output both count as failures for budget purposes, even though neither is a traditional "error."

**Implementation — SLI/SLO/SLA modeling with AI-specific failure definitions (Python):**

```python
from dataclasses import dataclass

@dataclass
class SLIMeasurement:
    time_to_first_token_ms: float
    full_completion_time_ms: float
    stop_reason: str  # "end_turn" | "max_tokens" | "refusal"

    def is_failure(self) -> bool:
        """AI-specific failure definition: a refusal or malformed output BOTH
        count as failures for error-budget purposes, even though neither is
        a traditional network/transport error."""
        return self.stop_reason == "refusal" or self._output_malformed()

    def _output_malformed(self) -> bool: ...

@dataclass
class SLOTarget:
    metric: str
    internal_target: float   # ALWAYS stricter than the external SLA
    external_sla: float

    def __post_init__(self):
        assert self.internal_target > self.external_sla, \
            "Internal SLO must be stricter than external SLA — drift must be caught internally first."

availability_slo = SLOTarget(metric="availability", internal_target=0.9995, external_sla=0.999)
```

**Common pitfalls:**
- Setting the internal SLO equal to or looser than the external SLA — this removes the internal early-warning margin entirely; a miss becomes a customer-facing breach with no internal buffer.
- Treating a refusal or malformed output as "not really an error" because it isn't a transport failure — both must count against the error budget per the AI-specific failure definition.
- Deriving the error budget from the external SLA figure rather than the (stricter) internal SLO.

---

### 2.2 Error budget mechanics

**What it is.** The error budget is:

```
error_budget = 1 − SLO_target   (over a rolling window)
```

A 99.9% objective leaves **0.1% over a 30-day month** to spend.

**Burn rate** = error rate ÷ (1 − SLO). A burn rate of 1 exhausts the entire monthly budget exactly at the window's end. Google's SRE practice recommends **multi-window burn-rate alerts**:[1]

| Window | Burn-rate threshold | Response |
|---|---|---|
| 1 hour | > 14.4 | Page immediately |
| 6 hours | > 6 | Ticket |
| 3 days | > 1 | Review |

**Why it matters.** **The error-budget policy is an automated release gate agreed in advance:** over budget, all non-critical releases freeze except P0 and security fixes, until the service is back within SLO. An incident that burns over 20% of the monthly budget in one event **mandates a postmortem with at least one P0 action item.** The freeze does not apply when the breach is purely an external dependency's fault — the policy governs what your team controls, not what a third party broke.

**Implementation — error-budget and multi-window burn-rate alerting (Python):**

```python
from dataclasses import dataclass

def error_budget(slo_target: float) -> float:
    return 1 - slo_target

def burn_rate(observed_error_rate: float, slo_target: float) -> float:
    return observed_error_rate / (1 - slo_target)

BURN_RATE_WINDOWS = [
    {"window_hours": 1, "threshold": 14.4, "response": "page_immediately"},
    {"window_hours": 6, "threshold": 6, "response": "ticket"},
    {"window_hours": 72, "threshold": 1, "response": "review"},
]

def check_burn_rate_alerts(error_rate_by_window: dict[int, float], slo_target: float,
                             breach_is_external_dependency_fault: bool = False) -> list[dict]:
    triggered = []
    for w in BURN_RATE_WINDOWS:
        rate = burn_rate(error_rate_by_window.get(w["window_hours"], 0), slo_target)
        if rate > w["threshold"]:
            triggered.append({**w, "observed_burn_rate": rate})
    return triggered

@dataclass
class ReleaseGatePolicy:
    slo_target: float
    current_budget_consumed_pct: float
    single_incident_burn_pct: float
    breach_is_external_fault: bool

    def release_status(self) -> dict:
        if self.breach_is_external_fault:
            # Policy governs what your team controls, not what a third party broke
            return {"status": "no_freeze", "reason": "breach attributable to external dependency"}

        if self.current_budget_consumed_pct >= 100:
            result = {"status": "FROZEN_except_P0_and_security"}
        else:
            result = {"status": "normal"}

        if self.single_incident_burn_pct > 20:
            result["postmortem_required"] = True  # mandates postmortem with >=1 P0 action item

        return result
```

**Common pitfalls:**
- Alerting on a single-window burn-rate threshold instead of the multi-window combination — a short spike and a slow, sustained leak require different response urgency, and one threshold cannot distinguish them.
- Freezing releases for a breach that was purely an external dependency's fault — the policy is meant to govern what the team controls, not to punish the team for third-party outages.
- Treating a burned error budget as a sign the team failed, rather than the policy working as designed (§2.3 makes this framing explicit).

---

### 2.3 Feedback cadence matched to stakeholder need

**What it is.** Match how fast you report to how fast each audience actually needs signal:
- **Weekly** — operational reviews (the team building and running the system).
- **Monthly** — business reviews (sponsors, budget owners).
- **Continuous** — an incident channel (everyone, only when something is actively breaking).

**Why it matters.** Report proactively against the **versioned SLA table**, align expectations and the escalation path *before* launch — not reactively after the first miss — and frame a burned error budget as **the policy working as designed**, not as the team failing. This reframe matters: a well-functioning error-budget policy is supposed to trigger a freeze occasionally. If it never does, either the SLO is too loose or nobody is enforcing it.

**Implementation — a cadence-matched reporting scheduler (Python):**

```python
from dataclasses import dataclass
from enum import Enum

class Cadence(Enum):
    WEEKLY = "weekly"
    MONTHLY = "monthly"
    CONTINUOUS = "continuous"

@dataclass
class ReportingChannel:
    audience: str
    cadence: Cadence
    content_focus: str

CHANNELS = [
    ReportingChannel(audience="engineering_team", cadence=Cadence.WEEKLY,
                      content_focus="operational: latency, cost, error budget burn"),
    ReportingChannel(audience="sponsors_budget_owners", cadence=Cadence.MONTHLY,
                      content_focus="business: outcome metrics vs. manual baseline"),
    ReportingChannel(audience="all_stakeholders", cadence=Cadence.CONTINUOUS,
                      content_focus="incident channel — only when actively breaking"),
]

def frame_budget_burn_message(budget_consumed_pct: float) -> str:
    """Frame as the policy working, not the team failing — this reframe should
    be baked into the reporting template itself, not left to ad-hoc framing."""
    if budget_consumed_pct >= 100:
        return (f"Error budget consumed ({budget_consumed_pct}%). This is the error-budget "
                f"policy working as designed — it caught risk before it became a customer-facing "
                f"issue. Non-critical releases are frozen until we're back within SLO.")
    return f"Error budget consumed: {budget_consumed_pct}%. Within normal operating range."
```

**Common pitfalls:**
- Reporting to sponsors only after an SLA is missed rather than proactively against the versioned table from launch — this is reactive, not the "before launch" alignment the source domain requires.
- Framing a budget freeze to stakeholders as a team failure — this undermines the policy's actual purpose and creates pressure to loosen SLOs rather than respect the signal.
- Using the same cadence for operational and business audiences — this either overwhelms sponsors with weekly engineering detail or starves the engineering team of the fast signal it needs.

---

### 2.4 Architecture Decision Records (ADRs)

**What it is.** An ADR captures **one architecturally significant decision** in five fixed parts: **Title, Status, Context, Decision, Consequences.**

**Why it matters.** Reserve records for choices that shape structure, interfaces, or dependencies and are **costly to reverse** — skip reversible local details that don't warrant the overhead. **The test is significance, not difficulty**; a hard-to-implement but easily-reversible choice doesn't need an ADR, while an easy-to-implement but hard-to-reverse choice does.

**The Consequences section must list all results, upside and downside together.** A record showing only benefits is incomplete by design — naming the accepted downside explicitly is what makes the tradeoff honest and reviewable later (this directly mirrors §1.7's executive-translation pattern, now in written form).

**Status lifecycle:** `proposed → accepted → deprecated → superseded`. Once accepted, a record is **immutable** — never edit it in place. Leave it intact, mark it superseded by the new record's number, and write a fresh record. **The log stays append-only** so the chain of reasoning stays readable years later, including *why* an earlier decision made sense given what was known at the time.

**Implementation — an ADR schema enforcing the five fixed parts, immutability, and mandatory downside (Python):**

```python
from dataclasses import dataclass
from enum import Enum
from datetime import date

class ADRStatus(Enum):
    PROPOSED = "proposed"
    ACCEPTED = "accepted"
    DEPRECATED = "deprecated"
    SUPERSEDED = "superseded"

@dataclass(frozen=True)  # frozen=True enforces immutability once instantiated as ACCEPTED
class ADR:
    number: str          # e.g., "ADR-001"
    title: str
    status: ADRStatus
    context: str
    decision: str
    consequences_upside: list[str]
    consequences_downside: list[str]  # MANDATORY — a record with none is incomplete by design
    date_recorded: date
    superseded_by: str = None

    def __post_init__(self):
        if self.status == ADRStatus.ACCEPTED and not self.consequences_downside:
            raise ValueError(f"{self.number}: an accepted ADR must name at least one downside. "
                              "A record showing only benefits is incomplete by design.")

class ADRLog:
    """Append-only — records are never edited in place."""
    def __init__(self):
        self._records: list[ADR] = []

    def add(self, adr: ADR):
        self._records.append(adr)  # append only

    def supersede(self, old_number: str, new_adr: ADR):
        old = next(r for r in self._records if r.number == old_number)
        # Cannot mutate `old` (frozen) — write a fresh record marking supersession instead
        superseded_version = ADR(
            number=old.number, title=old.title, status=ADRStatus.SUPERSEDED,
            context=old.context, decision=old.decision,
            consequences_upside=old.consequences_upside, consequences_downside=old.consequences_downside,
            date_recorded=old.date_recorded, superseded_by=new_adr.number,
        )
        # In practice this appends a new record reflecting the status change;
        # the original text of `old` remains intact and readable in the log.
        self._records.append(superseded_version)
        self.add(new_adr)

adr_001 = ADR(
    number="ADR-001", title="Routing + parallel-vote + agentic-diagnostic hybrid over full-agent design",
    status=ADRStatus.ACCEPTED,
    context="Need to triage inbound network fault tickets with a hard P1 false-negative ceiling.",
    decision="Use routing classifier + parallel-vote for P1 detection + agent only for diagnosis.",
    consequences_upside=["Cheaper and faster for 70% of tickets."],
    consequences_downside=["The 30% diagnostic path costs more in latency — accepted because "
                            "P1 false-negative rate was the binding constraint."],
    date_recorded=date(2026, 3, 1),
)
```

**Common pitfalls:**
- Writing an ADR for every implementation detail regardless of reversibility — the test is significance (structure/interface/dependency-shaping and costly to reverse), not implementation difficulty.
- Skipping an ADR for an easy-to-implement but hard-to-reverse choice — difficulty is not the test; reversibility is.
- Editing an accepted ADR in place instead of superseding it with a fresh record — this breaks the append-only chain that keeps historical reasoning readable.
- Writing a Consequences section that lists only benefits — incomplete by design; the downside must be named explicitly.

---

### 2.5 C4 model and arc42

**What it is.** **C4 nests three maintained views for three distinct audiences:**

| View | Shows | Audience |
|---|---|---|
| **Context** | The system as one box among users and external systems | Executives |
| **Container** | Deployable pieces and how they talk to each other | The architect's working view |
| **Component** | Building blocks inside one container | Implementers |

**Why it matters.** **arc42** is a document *template* — it tells you what to write down. **C4** tells you *how to draw* the structure. The two pair well together rather than competing: arc42 gives you the sections (context, constraints, solution strategy, quality requirements...); C4 gives you the diagramming convention to fill several of those sections consistently.

**Implementation — a C4 view registry mapped to arc42 sections (Python):**

```python
from dataclasses import dataclass
from enum import Enum

class C4View(Enum):
    CONTEXT = "context"
    CONTAINER = "container"
    COMPONENT = "component"

C4_AUDIENCE_MAP = {
    C4View.CONTEXT: "executives",
    C4View.CONTAINER: "architect_working_view",
    C4View.COMPONENT: "implementers",
}

# arc42 sections these C4 views typically fill
ARC42_SECTIONS_FILLED_BY_C4 = {
    C4View.CONTEXT: ["arc42 §3 System Scope and Context"],
    C4View.CONTAINER: ["arc42 §5 Building Block View (top level)"],
    C4View.COMPONENT: ["arc42 §5 Building Block View (detail level)"],
}

@dataclass
class ArchitectureDocument:
    """arc42 gives the WHAT-to-write sections; C4 gives the HOW-to-draw diagrams
    that fill several of those sections consistently."""
    context_view_diagram: str
    container_view_diagram: str
    component_view_diagram: str
    solution_strategy_text: str   # an arc42 section with no C4 diagram equivalent
    quality_requirements_text: str  # ditto — these are prose sections C4 doesn't cover
```

**Common pitfalls:**
- Treating C4 and arc42 as competing choices rather than complementary — arc42 structures the document; C4 standardizes the diagrams within it.
- Sharing a Component-level diagram with executives, or a Context-level diagram with implementers — each view is built for a specific audience's needed level of detail.

---

### 2.6 A real handoff transfers ownership, not just files

**What it is.** A complete handoff carries four things:
1. **Decision log** — the *why*, so the new team knows what not to accidentally undo (this is your ADR chain).
2. **Implementation guidance** — the *how*: interfaces, configuration, constraints.
3. **A runbook** — deploy, monitor, respond, roll back, written specifically for the on-call team, not the original architect.
4. **Acceptance criteria** — traced straight back to the requirements doc in §1.3.

**Why it matters.** **Shipping code and a slide deck moves files, not ownership.** A runbook that assumes the reader already knows the system's internals the way its original author did is not a handoff artifact — it's a note to a future version of yourself.

**Implementation — a handoff completeness checker (Python):**

```python
from dataclasses import dataclass

@dataclass
class HandoffPackage:
    decision_log_adr_numbers: list[str]
    implementation_guidance_doc: str
    runbook_doc: str
    runbook_written_for_oncall_not_author: bool  # explicit self-check
    acceptance_criteria_traced_to_req_doc: dict[str, str]  # req_id -> criterion

    def is_complete_handoff(self) -> dict:
        missing = []
        if not self.decision_log_adr_numbers:
            missing.append("decision_log")
        if not self.implementation_guidance_doc:
            missing.append("implementation_guidance")
        if not self.runbook_doc or not self.runbook_written_for_oncall_not_author:
            missing.append("runbook (must be written for on-call, not the original architect)")
        if not self.acceptance_criteria_traced_to_req_doc:
            missing.append("acceptance_criteria_traceability")
        return {"complete": not missing, "missing": missing,
                "note": "Shipping code + a slide deck alone moves files, not ownership." if missing else "OK"}
```

**Common pitfalls:**
- Writing a runbook that assumes deep system-internals familiarity, matching the original architect's knowledge rather than the on-call team's — this fails the actual audience test the source domain names explicitly.
- Treating "we shipped the code and gave a walkthrough presentation" as equivalent to a handoff — this moves files, not the accountability and operational capability a real handoff transfers.
- Omitting the acceptance-criteria traceability link back to the requirements doc — this severs the chain that lets a future team verify the system still does what it was built to do.

---

## Section 3: Lifecycle Management

### 3.1 The lifecycle is a cycle, not a line

**What it is.**

```
discovery → design → build → handoff → monitoring → iteration → (big finding reopens discovery)
```

**Why it matters.** **Phase gates are go/no-go checkpoints with exit criteria, not calendar dates.** The gate into production is the strictest of all: evals passing on a dataset that genuinely mirrors real traffic (Domain 4 §1.3), monitoring and SLOs wired up **before** launch (not "we'll add observability in the next sprint"), and a rollback path that's actually been exercised, not just documented.

**Implementation — a phase-gate checklist with exit criteria, not dates (Python):**

```python
from dataclasses import dataclass

@dataclass
class ProductionGateChecklist:
    evals_pass_on_production_mirroring_dataset: bool
    monitoring_wired_up_before_launch: bool
    slos_wired_up_before_launch: bool
    rollback_path_actually_exercised: bool  # NOT just documented

    def can_proceed_to_production(self) -> dict:
        checks = {
            "evals": self.evals_pass_on_production_mirroring_dataset,
            "monitoring": self.monitoring_wired_up_before_launch,
            "slos": self.slos_wired_up_before_launch,
            "rollback_tested": self.rollback_path_actually_exercised,
        }
        failed = [k for k, v in checks.items() if not v]
        return {"go": not failed, "failed_exit_criteria": failed,
                "note": "Gate is exit-criteria-based, not calendar-based — a target ship date "
                        "does not override an unmet criterion."}
```

**Common pitfalls:**
- Treating a phase gate as a calendar milestone ("we launch on the 15th regardless") rather than an exit-criteria checkpoint — this is the exact anti-pattern the source domain warns against.
- Documenting a rollback procedure without ever actually exercising it — an untested rollback path is not verified to work when it's actually needed.
- Deferring monitoring/SLO setup to "the next sprint" post-launch — this violates the production gate's strictest requirement.

---

### 3.2 Environment progression and pilot rollout

**What it is.** Systems travel: **development** (test data only) → **staging** (production-like, safe to break) → **production** (live traffic, where most of the lifecycle actually plays out).

**Why it matters.** **Pilot with two or three champion teams — never just one** — so early findings are validated signal, not a fluke tied to one team's unusual workflow. Then sequence surfaces into production **in risk order**: low-risk, high-reuse surfaces first, with rollback ready at every step to keep the blast radius small if something goes wrong.

**Implementation — a pilot-cohort validator and risk-ordered rollout sequencer (Python):**

```python
from dataclasses import dataclass

@dataclass
class PilotCohort:
    team_names: list[str]

    def is_valid_pilot(self) -> dict:
        """Never just one team — a single-team pilot cannot distinguish
        genuine signal from a fluke tied to that team's unusual workflow."""
        return {"valid": len(self.team_names) >= 2,
                "reason": None if len(self.team_names) >= 2 else
                          "Single-team pilots risk validating a fluke, not real signal."}

@dataclass
class RolloutSurface:
    name: str
    risk_level: int    # 1 (low) - 5 (high)
    reuse_level: int    # 1 (low) - 5 (high, i.e. widely reused)
    rollback_ready: bool

def sequence_rollout(surfaces: list[RolloutSurface]) -> list[RolloutSurface]:
    """Low-risk, high-reuse surfaces first — small blast radius, broad early validation."""
    for s in surfaces:
        assert s.rollback_ready, f"Cannot sequence {s.name}: rollback must be ready at every step."
    return sorted(surfaces, key=lambda s: (s.risk_level, -s.reuse_level))
```

**Common pitfalls:**
- Piloting with a single champion team — an early finding validated by only one team's workflow is not distinguishable from a fluke.
- Sequencing production rollout by convenience or political priority rather than risk order — this needlessly widens the blast radius of an early-stage problem.
- Moving a surface into production without a ready rollback path "because this one seems safe" — every step needs rollback readiness, not just the risky-looking ones.

---

### 3.3 Why offline evals are never enough alone

**What it is.** Pre-deployment evals (Domain 4) **cannot observe findings that only appear live**: real traffic drifts from the distribution you tested against, and rare inputs, genuine load, and adversarial use only reliably surface at production volume.

**Monitor four signal families against SLOs continuously:**
1. Quality.
2. Latency at the percentiles users actually feel (p95/p99, not the mean).
3. Cost per request.
4. Safety signals — refusals, injection attempts (Domain 5).

**Implementation — a continuous monitoring dashboard spec across the four signal families (Python):**

```python
from dataclasses import dataclass

@dataclass
class ContinuousMonitoringSignals:
    quality_score: float
    latency_p95_ms: float
    latency_p99_ms: float
    cost_per_request: float
    refusal_rate: float
    injection_attempt_rate: float

    def against_slos(self, slos: dict) -> dict:
        """Offline evals cannot see this — rare inputs, real load, and
        adversarial use only reliably surface at production volume."""
        return {
            "quality_within_slo": self.quality_score >= slos["quality_floor"],
            "latency_p95_within_slo": self.latency_p95_ms <= slos["max_p95_ms"],  # percentile, not mean
            "cost_within_slo": self.cost_per_request <= slos["max_cost_per_request"],
            "safety_within_slo": self.refusal_rate <= slos["max_acceptable_refusal_rate"],
        }
```

**Common pitfalls:**
- Treating a passing offline eval suite as sufficient proof the system is production-ready — offline evals structurally cannot see distribution drift, rare inputs, or adversarial use that only appears at live volume.
- Monitoring mean latency instead of p95/p99 — this is the same percentile discipline from Domain 4 §3.3, and skipping it here hides the tail experience that matters most.

---

### 3.4 Drift vs. regression

**What it is.**

| | Cause | Fix |
|---|---|---|
| **Drift** | The world moved — inputs shifted away from what you tested | Update evals, retrieval, or scope |
| **Regression** | A change *you* shipped made outputs worse | Caught by a frozen regression test (Domain 4 §2.4) |

**Why it matters.** Both show up on a dashboard as **falling quality** — they look identical from the metric alone. **Naming the cause wrong means fixing the wrong thing**: rolling back a regression that's actually drift fixes nothing; retraining or re-scoping for drift that's actually a regression wastes effort chasing a moving target that never moved.

**Implementation — a drift-vs-regression diagnostic function (Python):**

```python
def diagnose_falling_quality(recent_deploy_in_window: bool, input_distribution_shifted: bool,
                               frozen_regression_suite_still_passes: bool) -> dict:
    """Both causes look identical on a dashboard as 'falling quality' — this
    function encodes the actual diagnostic distinction, not just the symptom."""
    if recent_deploy_in_window and not frozen_regression_suite_still_passes:
        return {"diagnosis": "REGRESSION", "fix": "Roll back the shipped change; "
                "caught by the frozen regression test suite (Domain 4 §2.4)."}
    if input_distribution_shifted and frozen_regression_suite_still_passes:
        return {"diagnosis": "DRIFT", "fix": "Update evals, retrieval corpus, or scope — "
                "rolling back would fix nothing, since no shipped change caused this."}
    return {"diagnosis": "INCONCLUSIVE", "fix": "Gather more evidence before acting — "
            "naming the cause wrong wastes effort chasing the wrong fix."}
```

**Common pitfalls:**
- Rolling back a recent deploy when the actual cause is drift — the deploy wasn't the cause, so rolling back fixes nothing and the quality problem persists.
- Re-scoping or updating retrieval for what's actually a regression from a recent deploy — this wastes effort "chasing a moving target that never moved," while the actual regression remains live.
- Diagnosing from the dashboard trend alone without checking whether a recent deploy exists and whether the frozen regression suite still passes — the two causes are indistinguishable from the metric alone; the diagnostic requires these additional checks.

---

### 3.5 Iteration is evidence-triggered, not calendar-driven

**What it is.** Iteration fires on **an SLO breach, detected drift, or stakeholder feedback — never on calendar churn** ("it's been a quarter, let's improve the prompt"). Diagnose the cause first (Domain 4 §2.7's four origins), and reach for **prompt and retrieval changes before the model** — the model swap is the last and heaviest lever, exactly as in Domain 3's accuracy-latency lever ordering.

**Why it matters — deprecation is a planned phase, not a silent shutoff:** stakeholder notice, a migration path defined *before* anything is pulled, and records preserved for the successor system or a future audit. A deprecated system that simply stops responding one day is an incident, not a lifecycle phase done correctly.

**Implementation — an evidence-gated iteration trigger and a planned-deprecation checklist (Python):**

```python
from dataclasses import dataclass
from enum import Enum

class IterationTrigger(Enum):
    SLO_BREACH = "slo_breach"
    DETECTED_DRIFT = "detected_drift"
    STAKEHOLDER_FEEDBACK = "stakeholder_feedback"
    CALENDAR_CHURN = "calendar_churn"  # NOT a valid trigger

def should_iterate(trigger: IterationTrigger) -> bool:
    """Iteration fires on evidence, never on calendar churn alone."""
    return trigger != IterationTrigger.CALENDAR_CHURN

@dataclass
class DeprecationPlan:
    stakeholder_notice_given: bool
    migration_path_defined_before_pulling: bool
    records_preserved_for_successor_or_audit: bool

    def is_planned_correctly(self) -> dict:
        """A deprecated system that simply stops responding one day is an
        INCIDENT, not a lifecycle phase done correctly."""
        checks = {
            "notice": self.stakeholder_notice_given,
            "migration_path": self.migration_path_defined_before_pulling,
            "records": self.records_preserved_for_successor_or_audit,
        }
        missing = [k for k, v in checks.items() if not v]
        return {"correctly_planned": not missing, "missing": missing}
```

**Common pitfalls:**
- Scheduling prompt/architecture "improvement sprints" on a calendar cadence with no underlying SLO breach, drift, or stakeholder signal to justify them — this is explicitly named as invalid iteration triggering.
- Reaching for a model swap as the first iteration lever instead of diagnosing the cause first (Domain 4 §2.7) and trying prompt/retrieval changes before the heaviest lever.
- Shutting off a deprecated system without prior stakeholder notice or a defined migration path — this turns a plannable lifecycle phase into an unplanned incident.

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

**Implementation note connecting the worked example to the mechanisms above:** the requirements doc and assumption log follow §1.3's schema exactly; ADR-001 follows §2.4's five-part structure with an explicit downside in Consequences; the tiered decision matrix follows §1.5's lock-before-viewing discipline; the handoff package satisfies all four §2.6 requirements; the SLA/SLO pairing follows §2.1's stricter-internal-target rule; and the drift diagnosis in the final line is exactly the §3.4 diagnostic distinction applied in practice — confirmed via the frozen regression suite still passing, which is what correctly ruled out a regression.

---

## Architecture & Implementation Best Practices

1. **Gate every discovery process on covering all three stakeholder types** (sponsor, end user, affected party) before design begins — a single-perspective discovery misses information no other stage will recover.
2. **Lock decision-matrix criteria and weights before any candidate option is shown**, and structure tiers so no minor criterion can mathematically outvote a binding one.
3. **Build the executive-translation pattern into every stakeholder-facing template** — outcome, then technical choice, then explicit accepted downside, every time, not just for major decisions.
4. **Set every internal SLO strictly tighter than its corresponding external SLA**, and derive error budgets only from the SLO.
5. **Treat ADRs as append-only and immutable once accepted** — supersede with a fresh record, never edit history, and require a named downside before any record is marked accepted.
6. **Build handoff packages against the explicit four-part checklist** (decision log, implementation guidance, runbook-for-on-call, traced acceptance criteria) rather than trusting that "code plus a walkthrough" is sufficient.
7. **Treat phase gates as exit-criteria checkpoints, never calendar dates** — especially the production gate, where an unexercised rollback path or a monitoring gap must block launch regardless of the target ship date.
8. **Build a standing drift-vs-regression diagnostic into the monitoring/incident workflow** — checking recent-deploy status and frozen-regression-suite results before naming a cause, since the two failure classes are indistinguishable from the metric trend alone.
9. **Gate iteration on evidence (SLO breach, drift, stakeholder feedback) explicitly in process, not informally** — this prevents calendar-driven "improvement sprints" that lack a real trigger.

---

## Common Pitfalls and Limitations

- **Naming a technical solution during discovery** — the most common and hardest-to-reverse mistake in the entire domain.
- **Interviewing only sponsors** — end users and downstream affected parties surface information no other stakeholder type provides.
- **Averaging conflicting requirements into a mushy compromise** instead of tracing to shared outcome or escalating to the budget owner.
- **Silently resolving open questions with a guess** instead of logging them in the assumption log.
- **Reporting mean latency to non-technical stakeholders** as if it reflected typical experience.
- **Building the decision matrix after seeing candidate options** — invites weight reverse-engineering.
- **Spreading ATAM analysis evenly** instead of concentrating on high-importance, high-difficulty utility-tree leaves.
- **Hiding the accepted downside** in an executive pitch — forfeits informed consent.
- **Setting internal SLO equal to or looser than external SLA** — removes the internal early-warning margin.
- **Framing a burned error budget as team failure** rather than the policy working as designed.
- **Editing an accepted ADR in place** instead of superseding it with a fresh, append-only record.
- **Writing ADR Consequences with only upside** — incomplete by design.
- **Treating "shipped code + a slide deck" as a handoff** — moves files, not ownership.
- **Writing a runbook for the original architect's knowledge level** instead of the on-call team's.
- **Piloting with a single champion team** — cannot distinguish real signal from a team-specific fluke.
- **Treating a passing offline eval suite as production-ready proof** — cannot see drift, rare inputs, or adversarial use that only appears live.
- **Rolling back a regression that's actually drift** (or vice versa) — naming the cause wrong wastes effort on the wrong fix.
- **Iterating on calendar churn** rather than evidence (SLO breach, drift, feedback).
- **Reaching for a model swap before diagnosing the cause and trying prompt/retrieval changes first.**
- **Silently shutting off a deprecated system** without stakeholder notice or a defined migration path — turns a lifecycle phase into an incident.

---

## Official Documentation References

1. Google SRE Workbook, *"Alerting on SLOs"* — https://sre.google/workbook/alerting-on-slos/
2. Google SRE Book, *"Service Level Objectives"* — https://sre.google/sre-book/service-level-objectives/
3. C4 Model, *official documentation* — https://c4model.com/
4. arc42, *official template documentation* — https://arc42.org/overview
5. Michael Nygard, *"Documenting Architecture Decisions"* (originating ADR format) — https://cognitect.com/blog/2011/11/15/documenting-architecture-decisions
6. Software Engineering Institute (Carnegie Mellon), *"Architecture Tradeoff Analysis Method (ATAM)"* — https://www.sei.cmu.edu/library/architecture-tradeoff-analysis-method-collection/
7. Anthropic Docs, *"Define success criteria"* (cross-referenced for acceptance-criteria/eval seeding) — https://docs.claude.com/en/docs/build-with-claude/define-success

> **Version note:** This domain's content (SRE practices, ADR format, C4/arc42 conventions, ATAM) is largely methodology-based rather than API-surface-based, and is comparatively stable relative to Anthropic's product APIs. The cross-referenced Anthropic documentation link should still be verified periodically for currency.

---

## Domain Coverage / Traceability Matrix

| Source domain item | Implementation guide section(s) | Coverage |
|---|---|---|
| 1.1 Structured discovery + solution-first trap | §1.1 | ✅ Full, with `DiscoveryTracker` code |
| 1.1 Three interview modes table | §1.1 | ✅ Full |
| 1.1 Three stakeholder types | §1.1 | ✅ Full, with coverage-check code |
| 1.2 Requirements taxonomy (functional/non-functional/constraints) | §1.2 | ✅ Full, with `AIRequirement` code |
| 1.2 Three AI-specific qualifiers | §1.2 | ✅ Full |
| 1.2 Conflict tracing + budget-owner decision rule | §1.2 | ✅ Full, with `trace_conflict_to_outcome` code |
| 1.3 Requirements doc: stable ID, outcome, acceptance criterion, assumption log | §1.3 | ✅ Full, with `RequirementsDocument` code |
| 1.3 Traceability principle | §1.3 | ✅ Full, with `why_doesnt_it_do_x` code |
| 1.4 Customer-terms quality attribute translation | §1.4 | ✅ Full, with translation code |
| 1.4 High-percentile latency framing | §1.4 | ✅ Full |
| 1.5 Tiered-criteria decision matrix | §1.5 | ✅ Full, with `DecisionMatrix` code |
| 1.5 Lock-before-viewing-options rule | §1.5 | ✅ Full |
| 1.6 ATAM four findings table | §1.6 | ✅ Full, with classifier code |
| 1.6 Utility tree prioritization | §1.6 | ✅ Full, with `build_utility_tree` code |
| 1.6 Worked tradeoff-point example (context window) | §1.6 | ✅ Full, verified in code |
| 1.7 Executive-translation pattern (outcome → choice → tradeoff) | §1.7 | ✅ Full, with `ExecutiveTranslation` code |
| 1.7 Present downside openly | §1.7 | ✅ Full |
| 2.1 SLI/SLO/SLA chain + what-if-missed test | §2.1 | ✅ Full, with `SLOTarget` code |
| 2.1 Internal-stricter-than-external rule | §2.1 | ✅ Full |
| 2.1 AI-specific SLI split (TTFT vs. completion) + failure definition | §2.1 | ✅ Full, with `SLIMeasurement` code |
| 2.2 Error budget formula | §2.2 | ✅ Full, with `error_budget` code |
| 2.2 Burn rate formula + multi-window alert table | §2.2 | ✅ Full, with alert code |
| 2.2 Release-gate policy + postmortem trigger + external-fault exception | §2.2 | ✅ Full, with `ReleaseGatePolicy` code |
| 2.3 Cadence table (weekly/monthly/continuous) | §2.3 | ✅ Full, with `ReportingChannel` code |
| 2.3 Proactive reporting + budget-burn-as-policy-working reframe | §2.3 | ✅ Full, with framing code |
| 2.4 ADR five-part structure | §2.4 | ✅ Full, with `ADR` dataclass |
| 2.4 Significance-not-difficulty test | §2.4 | ✅ Full |
| 2.4 Consequences must include downside | §2.4 | ✅ Full, enforced in code |
| 2.4 Status lifecycle + immutability + append-only log | §2.4 | ✅ Full, with `ADRLog` code |
| 2.5 C4 three views table | §2.5 | ✅ Full, with registry code |
| 2.5 arc42 vs. C4 (template vs. diagramming convention) | §2.5 | ✅ Full |
| 2.6 Four handoff components | §2.6 | ✅ Full, with `HandoffPackage` code |
| 2.6 Files-vs-ownership distinction | §2.6 | ✅ Full |
| 3.1 Lifecycle-as-cycle diagram | §3.1 | ✅ Full |
| 3.1 Phase gates as exit-criteria, not dates | §3.1 | ✅ Full, with `ProductionGateChecklist` code |
| 3.2 Environment progression (dev/staging/prod) | §3.2 | ✅ Full |
| 3.2 Multi-team pilot rule + risk-ordered rollout | §3.2 | ✅ Full, with sequencing code |
| 3.3 Why offline evals are insufficient alone | §3.3 | ✅ Full |
| 3.3 Four continuous monitoring signal families | §3.3 | ✅ Full, with `ContinuousMonitoringSignals` code |
| 3.4 Drift vs. regression table | §3.4 | ✅ Full, with diagnostic code |
| 3.4 Identical-on-dashboard / wrong-diagnosis-wastes-effort principle | §3.4 | ✅ Full |
| 3.5 Evidence-triggered iteration (three valid triggers, one invalid) | §3.5 | ✅ Full, with `should_iterate` code |
| 3.5 Prompt/retrieval-before-model lever order | §3.5 | ✅ Full |
| 3.5 Planned deprecation (three requirements) | §3.5 | ✅ Full, with `DeprecationPlan` code |
| Worked Example: Lifecycle Artifacts (full package) | Worked Example section | ✅ Full, reproduced and annotated |
| Key Takeaways (8 bullets) | Distributed across Best Practices + Pitfalls | ✅ Full — every bullet maps to at least one item |
| References 1–6 (source domain) | Official Documentation References | ✅ Full, retained and extended with success-criteria cross-reference |

**Coverage verification statement:** Every numbered subsection, table, and key-takeaway bullet in `domain-6-stakeholder-communication-lifecycle.md` is represented above with expanded technical detail, at least one implementation pattern, and a runnable Python example (plus concrete artifact schemas/templates) wherever the source item names or implies a concrete mechanism. No source item was merged, omitted, or skipped.
