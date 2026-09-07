# Domain 5: Governance, Safety & Risk Management
**Exam weight: 14% | Claude Certified Architect — Professional**

> Every earlier domain assumed the system works as intended. Domain 5 assumes it will occasionally fail, be attacked, or process regulated data — and asks whether your architecture survives that reality with bounded, auditable, defensible risk.

---

## 1. Guardrails & Safety Controls

### 1.1 Defense in depth

No single control is airtight. Layer four cooperating controls, each covering the gaps the others miss:

```
┌─────────────────┐   ┌─────────────────┐   ┌─────────────────┐   ┌─────────────────┐
│ ACCESS CONTROLS   │→ │ INPUT SCREENS     │→ │ OUTPUT             │→ │ MONITORING        │
│ bound what the    │   │ filter requests   │   │ CLASSIFIERS        │   │ of production      │
│ model can reach   │   │ before they land  │   │ real-time, on the  │   │ traffic (Domain 4)  │
│ (deterministic)   │   │ (probabilistic)   │   │ way out            │   │                     │
└─────────────────┘   └─────────────────┘   └─────────────────┘   └─────────────────┘
```

**Training, system prompts, and classifiers are probabilistic** — they lower the odds of harm but never reach zero. **Access controls, sandboxing, and egress limits are the only deterministic boundary**, and they are what holds when every probabilistic layer misses simultaneously. This is the same probabilistic-vs-deterministic distinction from Domain 2's guardrail discussion, applied at the system level rather than the single-prompt level.

### 1.2 Constitutional Classifiers

Anthropic's **Constitutional Classifiers** are input and output classifiers trained on a written constitution of allowed versus disallowed content. In Anthropic's published 2025 research, this approach **cut jailbreak success from 86% to 4.4%**, at the cost of added compute and a measured refusal-rate increase.<sup>[1]</sup> A subsequent public red-teaming exercise found that a determined, well-resourced adversary could still eventually find a universal jailbreak after roughly 3,700 collective hours of effort across participants — a concrete demonstration that **classifiers reduce risk substantially but do not eliminate it**, reinforcing why defense in depth (not any single layer) is the actual control.<sup>[1]</sup>

### 1.3 Prompt injection: contain it structurally

- **Direct injection** — the adversary is at the keyboard, typing the payload straight into the input
- **Indirect injection** — the payload is hidden in third-party content the agent retrieves (a document, a web page, a tool result). This is the **dominant risk for anything that browses, reads files, or calls external tools**, because the payload arrives disguised as legitimate data the agent was asked to process.

Contain it structurally, not with a clever prompt instruction:
- Untrusted content only in tool-result blocks
- JSON-encoded or otherwise fenced so it cannot escape its data context and be read as an instruction
- Every tool scoped to least privilege (Domain 3, §1.2), so a successful injection has a small blast radius

### 1.4 Output screening as the exit gate

A lightweight classifier returns a structured boolean verdict on **every** completion and tool result. Flagged outputs are blocked, redacted, escalated, or the account is throttled — the specific action depends on the severity and the domain. Each screen is another model call's worth of latency and cost, so **right-size the stack to the stakes**, and red-team continuously rather than treating the initial stack as permanent.

---

## 2. Risks, Limitations & Failure Modes

### 2.1 The core failure taxonomy

Four failure modes cover nearly every real-world incident:

| Failure mode | What it is | Fix family |
|---|---|---|
| **Hallucination** | Plausible, confident content that's wrong or unsupported | Escape hatches, citation-then-verify, consistency sampling (Domain 4, §2.6) |
| **Prompt injection** | An input-handling flaw that hijacks the instruction channel | Structural isolation (§1.3) |
| **Jailbreak** | A guardrail flaw — the model is talked past its own safety training | Classifiers, defense in depth (§1.1–1.2) |
| **Data disclosure** | Sensitive data leaks through the model's output | Access controls, output screening, least privilege |

**Injection and jailbreak point to different fix families and are easy to conflate.** Injection is fundamentally about *what data the model was allowed to treat as an instruction*. A jailbreak is about *the model's own trained refusal behavior* being circumvented. Diagnosing one as the other sends the fix to the wrong layer.

### 2.2 Silent failures are the dangerous class

The output looks clean but is wrong — **no error fires, and retry logic never sees it.** A confident hallucination passes every status check a monitoring system typically watches (Domain 4, §3.3). Catching these requires validating the *content itself* against expectations, not just exception handling — this is exactly why output screening (§1.4) and process/outcome checks (Domain 4, §1.6) exist as separate layers from infrastructure monitoring.

### 2.3 Five systemic failure modes unique to agentic autonomy

Beyond the base taxonomy, autonomy adds five modes that amplify and hide each other:

1. **Compounding errors** — a small mistake in step 2 cascades and grows through steps 3, 4, 5
2. **Non-determinism** — the same prompt can return different output across runs; one green test run proves almost nothing
3. **Tool misuse** — a tool is called correctly in syntax but wrongly in judgment
4. **Distribution shift** — production inputs drift from what the system was validated against
5. **Automation bias** — human reviewers start trusting the system's output *more* than warranted, precisely because it's usually right

**Bound autonomy with step caps, validate intermediate results, and monitor live quality against a baseline** — this directly connects to Domain 1's failure-mode table for multi-agent systems (over-spawning, vague delegation, synchronous bottleneck, compounding errors) and Domain 4's process-vs-outcome checks.

**Certify non-deterministic behavior with repeated runs, distributions of outcomes, and thresholds — not a single pass.** A/B testing (Domain 4, §2.1) and pass@k / pass^k metrics exist specifically because one successful run tells you almost nothing about a non-deterministic system's reliability.

### 2.4 Judging deployment readiness

Judge deployment on the boundary of **competence, reversibility, and blast radius** — a model stays articulate well past the point where it stays correct, so fluency and confidence are never a signal of accuracy. **Irreversible and wide-reaching means rethink the design**, not just add another guardrail layer. Maintain a failure-mode-to-mitigation matrix that pairs each named failure above with **exactly one owned control** — a failure mode with no assigned mitigation is a known gap, not an oversight to discover during an incident.

---

## 3. Human-in-the-Loop & Compliance

### 3.1 Why a guardrail isn't enough by itself

A guardrail runs on *every* call and reduces risk — but a low per-call failure rate still fails eventually across enough calls. **Only a human gate bounds risk on the calls that reach it**, at the cost of a person's time on each one. HITL is a **designed control**, chosen deliberately for specific checkpoints — not a generic fallback bolted on wherever something might go wrong.

### 3.2 Four gate patterns

| Pattern | Mechanism | Adds latency? |
|---|---|---|
| **Approval before action** | Workflow blocks entirely for a human "yes" | Yes, every call |
| **Confirmation prompt** | Restates what's about to happen before proceeding | Yes, every call |
| **Review sampling** | Audits a fraction of actions after the fact | No — but catches problems late |
| **Escalation threshold** | Routes low-confidence or high-stakes cases to a reviewer | Only on the flagged fraction |

**Reserve blocking approval (the first two patterns) for genuinely irreversible steps** — sending, paying, deleting. Apply it too broadly and reviewers start clicking "yes" just to clear the queue, which is worse than no gate at all because it manufactures false confidence in the audit trail.

### 3.3 Three traps that produce rubber-stamping

1. **Alert fatigue** from constant gate fires
2. **A bare approve button** with no evidence or reasoning attached for the reviewer to actually evaluate
3. **Reviewers measured on speed** rather than judgment quality

A working gate **shows the evidence**, makes reject exactly as easy as approve, and **a reject rate near zero is itself a signal the gate has become theater** — not a signal the system is working perfectly.

### 3.4 Four questions that decide whether an action gets a human gate

1. Is it a **one-way door**? (Can it be cheaply undone?)
2. How **wide is the blast radius**?
3. Is it a **regulated domain** — legal, financial, employment, healthcare?
4. How **confident** is the model on this specific case?

**One strong signal alone justifies a gate.** Most reversible, low-stakes work should flow without one — gating everything defeats the purpose of automation and, per §3.3, degrades the quality of the gates that actually matter.

### 3.5 Three compliance regimes, three distinct subjects

**Never cross-attribute** — a BAA never satisfies a GDPR obligation, and a single system can trigger all three regimes simultaneously, each satisfied independently on its own terms.

| Regime | Subject | Your typical role |
|---|---|---|
| **GDPR** | Personal data of people in the EU | Controller (almost always, in your own application) |
| **HIPAA** | US protected health information (PHI) | Business associate (almost always, as an LLM service) |
| **FedRAMP** | US federal agency cloud data | Cloud service provider seeking certification |

**GDPR — controller vs. processor is a functional test, never a contract label.** Whoever actually determines the *purposes and means* of processing is the controller. In your own application, you are almost always the controller, with the model provider acting as processor under a Data Processing Agreement (DPA). A processor that starts setting its own purpose for the data becomes a controller in fact, and takes on the corresponding liability — regardless of what the contract calls it.

A **DPIA** (Data Protection Impact Assessment) fires on large-scale processing, systematic monitoring, or automated decision-making. **Article 22 restricts solely automated decisions with legal or similarly significant effects**, requiring a route to human review and contest — this is the direct regulatory grounding for the HITL patterns in §3.1–3.4 when the system operates on EU personal data. Cross-border transfers ride on an adequacy decision or safeguards like Standard Contractual Clauses; the model call itself is "processing" wherever it physically runs.<sup>[2]</sup>

**HIPAA regulates exactly three covered-entity types:** health care providers (covered only when they transmit HHS-standard electronic transactions), plus health plans and clearinghouses, which are covered outright. **Your LLM service is almost always the business associate**, not a covered entity. A written **Business Associate Agreement (BAA)** under **45 CFR §164.504(e)** must be signed before any PHI flows through the system — no HIPAA-certified badge or vendor claim substitutes for the actual signed agreement.<sup>[3]</sup>

**FedRAMP** baselines historically ran Li-SaaS, Low, Moderate, and High. Under the 2026 consolidated rules, these are being relabeled **Certification Classes A through D**, with both vocabularies valid during the transition period through the end of 2028.<sup>[4]</sup> The joint authorization board that previously handled cross-agency reuse was dissolved in May 2024 — every new authorization now runs through a single sponsoring agency that owns both the risk decision and the ongoing continuous-monitoring obligation.

---

## 4. Ethical AI

### 4.1 NIST AI RMF: four functions

The **NIST AI Risk Management Framework** organizes trustworthy AI into four functions — **Govern, Map, Measure, Manage** — with fairness as one of several characteristics they protect, alongside safety, security, and transparency.<sup>[5]</sup>

**Bias enters through three sources, each needing a different fix:**
1. The **data** (skewed training or fine-tuning distribution)
2. The **algorithm** (objective function that optimizes for the wrong thing)
3. The **humans** who label and accept outputs (annotator bias, uneven acceptance criteria)

Scrubbing the data alone addresses only one of the three — a common and costly mistake in bias-remediation projects that stop after a data-cleaning pass.

### 4.2 Bias vs. fairness

**Bias is descriptive** — a measurable skew in outcomes you can detect empirically. **Fairness is normative** — a definition you *choose* for the context and then measure against. **Demographic parity, equalized odds, and calibration are mathematically incompatible when base rates differ between groups** — this is a proven result, not an engineering oversight. You pick one fairness objective for the specific deployment and defend the trade-off explicitly; claiming a system satisfies "fairness" without naming which definition is a red flag in review.

### 4.3 Mitigation at three lifecycle stages

| Stage | What it does |
|---|---|
| **Pre-processing** | Rebalances training/reference data |
| **In-processing** | Constrains the objective function during training/tuning |
| **Post-processing** | Adjusts or reviews outputs after generation |
| **Production monitoring** | Ongoing — because bias drifts just like any other quality metric (Domain 4, §3.3) |

Test with **subgroup accuracy, disparate impact ratios, and counterfactual checks** that change only the sensitive attribute and observe whether the output changes.

### 4.4 Three different transparency bars

| Bar | Question it answers |
|---|---|
| **Transparency** | Openness about the system as a whole |
| **Explainability** | How a *specific* output was reached |
| **Interpretability** | Whether the recipient can actually understand that explanation |

**Transparency never requires publishing model weights.** The actual test is whether an informed outsider could meaningfully scrutinize the decision-making process — a much lower and more achievable bar than full model disclosure.

### 4.5 Accountability, made concrete

**GDPR Article 22 makes explanation a legal expectation** for solely automated decisions with major effects: affected people get human intervention, a way to contest the decision, and meaningful information about the logic involved.<sup>[2]</sup> Accountability in practice means **a named person owns the outcomes**, backed by AI disclosure to affected people and an audit trail of inputs, versions, and rationale captured as the system runs — the same audit-trail discipline from §3.5's compliance section, applied as an ethical obligation rather than only a legal one.

---

## Worked Example: Governance Layer for the Telecom Triage Agent

Extending the architecture from Domains 1–4:

```
┌─────────────────────────────────────────────────────────────┐
│ DEFENSE IN DEPTH                                                │
│  Access: agent scoped to read-only network tools + ticket-write │
│  Input screen: harmlessness pre-check on raw ticket text         │
│  Output classifier: verdict on every routing decision            │
│  Monitoring: OTel + drift sampling vs. launch baseline (Dom. 4)  │
└─────────────────────────────────────────────────────────────┘
                          │
┌─────────────────────────▼─────────────────────────────────────┐
│ HITL GATE (escalation-threshold pattern)                         │
│  Triggers on: confidence < 0.6 OR P1 outage OR regulated-account │
│  flag (e.g., a hospital's dedicated line — healthcare-adjacent)   │
│  Reviewer sees: full diagnostic transcript + confidence + reason  │
└─────────────────────────────────────────────────────────────┘
                          │
┌─────────────────────────▼─────────────────────────────────────┐
│ COMPLIANCE                                                        │
│  GDPR: EU customer tickets — controller role, DPA with model      │
│  provider, Article 22 human-review path wired into the escalation │
│  gate above (not a separate bolt-on)                              │
│  Audit trail: named on-call engineer owns each escalated decision │
└─────────────────────────────────────────────────────────────┘
```

Note how the HITL gate from §3.4 and the Article 22 requirement from §3.5 are **the same mechanism**, not two separate systems — good governance design finds these overlaps rather than building parallel compliance machinery next to the operational one.

---

## Key Takeaways

- Defense in depth: probabilistic layers (training, prompts, classifiers) lower risk; only access controls and sandboxing are deterministic. Constitutional Classifiers cut jailbreak success from 86% to 4.4% but did not reach zero under sustained red-teaming.
- Indirect injection (hidden in retrieved content) is the dominant risk for browsing/tool-using agents — contain it structurally, not with prompt language.
- Four failure modes — hallucination, injection, jailbreak, data disclosure — point to four different fix families. Diagnose which one before choosing a fix.
- Agentic autonomy adds five systemic modes (compounding errors, non-determinism, tool misuse, distribution shift, automation bias) that amplify and hide each other.
- HITL is a designed control at deliberately chosen checkpoints, not a fallback. A near-zero reject rate is a sign the gate is theater, not that the system is flawless.
- GDPR, HIPAA, and FedRAMP have three distinct subjects and never satisfy each other. Controller/processor status is a functional test, not a contract label.
- Fairness definitions (demographic parity, equalized odds, calibration) are mathematically incompatible when base rates differ — pick one and defend the trade-off explicitly.

---

## References

1. Anthropic News, *"Constitutional Classifiers: Defending against universal jailbreaks,"* Feb 3, 2025 — https://www.anthropic.com/news/constitutional-classifiers
2. Regulation (EU) 2016/679 (GDPR), Article 22 — https://gdpr-info.eu/art-22-gdpr/
3. U.S. Dept. of Health & Human Services, *"Business Associate Agreements"* (45 CFR §164.504(e)) — https://www.hhs.gov/hipaa/for-professionals/covered-entities/sample-business-associate-agreement-provisions/index.html
4. FedRAMP.gov, *"FedRAMP 20x / Certification Classes"* — https://www.fedramp.gov/20x/
5. NIST, *"AI Risk Management Framework (AI RMF 1.0)"* — https://www.nist.gov/itl/ai-risk-management-framework
6. Anthropic, *"Responsible Scaling Policy"* — https://www.anthropic.com/rsp
7. Anthropic Docs, *"Reduce prompt injection risk"* — https://docs.claude.com/en/docs/test-and-evaluate/strengthen-guardrails/mitigate-jailbreaks
