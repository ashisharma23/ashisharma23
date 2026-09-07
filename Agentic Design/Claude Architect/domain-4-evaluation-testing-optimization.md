# Domain 4: Evaluation, Testing & Optimization
**Exam weight: 16% | Claude Certified Architect — Professional**

> An architecture is a hypothesis until it's measured. Domain 4 is about turning the fuzzy success criteria from Domain 1 into datasets and graders you can actually run, then keeping the system honest as it changes over time.

---

## 1. Metrics & Evaluation Frameworks

### 1.1 SMART criteria, adapted for AI systems

**SMART** evaluation criteria are specific, measurable, achievable, and relevant — the classic fifth mark, *time-bound*, is dropped in this context in favor of standing, ongoing criteria (an eval suite doesn't expire on a date).

Rewrite vague goals as numeric thresholds on a **held-out set**:
- "Classify sentiment well" → **F1 ≥ 0.85 on 10,000 held-out items**
- "Keep it safe" → **≥ 99.5% non-toxic outputs**
- "Be fast" → **95% of responses under 200ms**

A held-out test set is reserved from *all* prompt tuning, so scores measure real generalization rather than grading your own homework. **Brittle exact-match grading fails correct rephrasings** — pair every target with a grader that rewards valid variation and awards partial credit, since the criterion and the grader are effectively one design decision, not two.

**Subjective targets** (tone, helpfulness) are quantified with a **rubric** that fixes what a 1 and a 5 look like concretely, a judge model rates each response against it, and you track both the average score *and* the share falling below your bar. Subjective means unquantified in casual conversation — it doesn't mean *unquantifiable* in a production eval.

### 1.2 Three metric families

| Family | Examples | Signal | Transfers across tasks? |
|---|---|---|---|
| **Task-specific** | F1 for classifiers, field-level accuracy for extraction, unit tests for code | Highest signal | No |
| **Generic quality** | Coherence, relevance, tone | Blunter | Yes |
| **Operational** | p95 latency, cost/call, throughput, availability | Gates whether quality ever ships | Yes |

Weigh only the categories the use case actually needs: a support bot weights tone and privacy; a fraud classifier weights task fidelity and latency; a summarizer weights context utilization and relevance. Don't run every family with equal weight on every project — that dilutes signal.

Real systems score **accuracy, latency, cost, and safety together**, and levers that raise accuracy (bigger model, more retrieved context, extra reranking — Domain 3, §2.6) simultaneously add latency and cost. Resolve the tension one of three ways:
- A weighted composite score
- Thresholds-then-optimize (clear the bar on everything, then optimize the one you're being judged on)
- Let the binding constraint decide (Domain 3, §1.3)

**Safety** is scored as a rate (non-toxic percentage, refusal correctness). **Security** is measured as attack-success rate on prompt-injection and jailbreak sets — a defensible but genuinely less standardized dimension than safety across the industry.

### 1.3 Building eval datasets that mirror production

An eval dataset must mirror real traffic — matching the distribution of question types and difficulty, not just covering the happy path. **Deliberately seed adversarial and malformed inputs**, plus four specific edge cases: **irrelevant, overly long, harmful, and ambiguous.** A happy-path set drawn from easy inputs will score green and hide exactly the failures the eval exists to catch.

**Favor volume with automated grading over hand-graded precision.** Authoring questions and golden answers is a one-time cost; grading recurs on *every run, forever*. A hundred cheaply-graded cases beat twenty hand-graded ones for this reason. "Quality" in this principle refers to the grading signal, not the question prose — a beautifully written question wired to a brittle grader still fails you.

### 1.4 The grading hierarchy

Prefer the fastest, most reliable tier available for each check:

1. **Code-based checks** (exact match, regex, schema/assertion validation) — fastest, most reliable, use whenever the answer space is structured
2. **Judge model** — for free-form quality that code genuinely cannot express
3. **Human** — slow, costly, the last resort, reserved for what neither of the above can grade

Prefer **partial credit** over binary exact-match wherever more than one phrasing is genuinely correct; use semantic similarity or reference metrics like ROUGE for those cases.

### 1.5 Hardening an LLM-as-judge

LLM-as-judge needs four specific guardrails, or it becomes a source of false confidence rather than signal:

1. **A judge model different from the model being evaluated** — self-preference bias measurably inflates own-family scores independent of actual quality
2. **Calibration** against a set of human-graded examples before trusting the judge at scale
3. **An explicit escape hatch** — permission to return "cannot determine" rather than fabricating confidence
4. **One isolated judge per dimension**, returning label-only output — so a strong score on one axis (e.g., fluency) never hides a weak one (e.g., factuality)

### 1.6 Process vs. outcome checks

**Process checks** read the transcript itself — reasoning, tool calls, intermediate steps. **Outcome checks** read only the final state in the environment. Together, they separate a genuine *reasoning failure* from an *environment failure* (the model reasoned correctly but a tool returned bad data, or vice versa).

**Before trusting any red score**, run the **fair-failure check**: confirm the grader didn't reject a genuinely valid answer, and confirm the task spec wasn't itself ambiguous. A brittle grader looks *exactly* like a model regression from the outside — this check is what tells them apart.

---

## 2. Testing & Diagnosis

### 2.1 What makes an A/B test valid

A valid A/B test holds the eval set and metrics **identical across both arms** and changes **exactly one variable** — prompt or model, never both at once. Break any of those controls and you're eyeballing outputs, not running an experiment, regardless of how rigorous the write-up looks afterward.

**Statistical rigor** means enough samples to separate a real gain from noise, one change per test, and a margin large enough to matter before promoting a variant. The two classic traps: cherry-picked wins that evaporate on real traffic, and metrics or datasets that quietly shift between arms.

### 2.2 Versioning prompts like code

Version every prompt with: a label or hash on each version, a record of which version produced which eval run, and old versions **retained for rollback.** Without this discipline you cannot reproduce a result or line up a fair comparison six months later when someone asks why a metric moved.

### 2.3 Three ways to run a test

1. **Offline** against a fixed held-out set first — fast, cheap, repeatable
2. **Live traffic split** for real inputs the offline set couldn't anticipate
3. **Canary release** to a small user group that widens only if metrics hold

A winning variant must still **re-run the full eval suite** — a local win on the metric you were optimizing that quietly breaks previously-passing cases elsewhere is a net loss, not a win.

### 2.4 The champion-challenger loop

The production **champion** stays live until a **challenger** clearly beats it on the agreed metrics — the burden of proof sits on the new version, not the incumbent. Guard against regressions with:
- A frozen baseline suite
- Multidimensional scoring across accuracy, latency, safety, and cost
- Guardrail metrics you refuse to let slip, no matter how good the headline number looks

Feed production outputs back as **verification inputs** via self-verify, cross-check, or cite-then-verify patterns (Domain 5, §2 covers the hallucination-mitigation version of this).

### 2.5 Diagnosing failures: the two-layer spine

Failures live in exactly two layers, and diagnosing the wrong one wastes real engineering time:

| Layer | Symptom | Diagnose from |
|---|---|---|
| **Integration failure** | 4xx/5xx status codes, transport errors | Status, error type, request ID |
| **Model-output problem** | A successful 200 that's still wrong | Stop reason, stop details, content |

`end_turn` is healthy. `max_tokens` means **truncation, not a wrong answer.** `refusal` is a policy decline. **None of these are network errors** — blind retries fix nothing when the failure is in this second layer.

**The most-missed failure class: the 200 that still failed.**
- A streamed response can return `200` and then emit an error event mid-stream
- A `refusal` stop reason is a decline, not an answer — don't score it as content
- Truncated output reads as complete unless you actually check the stop reason

Read the whole response, not just the status line.

**A zero pass rate almost never means the model cannot do the task.** It usually signals a broken task spec or a grader rejecting genuinely valid solutions. Read a handful of transcripts and run the fair-failure check (§1.6) before choosing *any* fix — swapping models on a zero-pass-rate eval without this check is a common, expensive mistake.

### 2.6 Grounding hallucination by origin

Hallucination is **plausible, confident content that is wrong or unsupported by context** — and no single toggle removes it. Layer four techniques by origin:

1. **Escape hatch** — explicit permission to say "I don't know"
2. **Direct-quote extraction** before answering, so claims are grounded in retrieved text
3. **Citation-then-verify** with code or a second model pass
4. **Consistency sampling** — fabrications tend to wander across repeated runs; facts stay stable, so sampling the same prompt multiple times surfaces likely hallucinations

### 2.7 Four origins of one bad answer

Never swap the model first. Diagnose in this order:
1. **Prompt failure** (ambiguous instructions, missing context)
2. **Hallucination** (see §2.6)
3. **Model mismatch** (right-size up or down for the actual reasoning depth needed)
4. **Retrieval** (in RAG systems, this is the *dominant* source of bad answers — Domain 3, §2.7)

Answers turning confidently wrong right after a document refresh point at the index, not the model. Precision@k, recall@k, MRR, and NDCG localize whether a RAG miss was a precision problem or a recall problem before you touch the prompt or the model.

---

## 3. Optimization & Monitoring

### 3.1 Latency and cost levers before trading away quality

**Output tokens drive generation time** because they're produced one at a time; **input tokens process in parallel.** A concise-output instruction in the system prompt caps verbosity at the source — cheaper and faster with zero quality cost. Streaming lowers *perceived* latency only, not total generation time. Any request that could run past ten minutes must stream or run as a batch job.

**Pull same-model levers before switching models:**
- Lower `output_config.effort` — the model reasons less, consolidates tool calls, returns sooner
- Anthropic notes effort is often a *better* lever than switching models entirely, because it's cheaper to test and instantly reversible

Move to a faster model only when the SLA is still missed *and* your evals confirm the smaller model clears the quality bar on your actual task distribution — not just on a handful of manual spot checks.

### 3.2 Cache and batch economics, restated for optimization

- **Reads:** ~0.1× base input cost
- **5-minute writes:** ~1.25× base, pay off after a single read
- **1-hour writes:** ~2× base, need two or more reads to win

The cache renders content in a fixed order — tools, then system, then messages — matches the *longest stable prefix* on exact bytes, allows up to four breakpoints, and is model-scoped. Put frozen content first; volatile per-request data last (Domain 2, §4.5 covers the mechanics).

**Silent cache misses look configured but never hit:** a timestamp or session ID buried in the system prompt, unsorted JSON, a shifting tool set, or a prefix below the minimum token floor. Verify with cache-read tokens across repeated calls — never trust the configuration alone.

**Message Batches API:** a flat **50% off both input and output tokens** for asynchronous work, results returned in any order matched by a custom identifier, and **stacks with caching discounts.** This is where deep-reasoning, no-deadline workloads belong by default.

Quality, latency, and cost never max out together. Name the SLO's binding constraint first, pull the cheap, reversible levers (effort, caching, batching, concise output, model routing) before rearchitecting anything, and validate every change against the eval suite — not against a single "it feels better" spot check.

### 3.3 Monitoring: continuous, not a one-time gate

**Evaluation is a pre-deployment check on a fixed dataset. Monitoring is a continuous post-deployment practice on live traffic you never chose.** Anthropic is explicit that monitoring is essential precisely because evals cannot see: unimagined user inputs, upstream data and model updates that shift behavior silently, or rare edge cases that only surface at production volume.

**Three foundations hold up monitoring:**
1. **Telemetry** — rides on OpenTelemetry (Domain 3, §3.3), stays off until deliberately enabled
2. **A baseline** measured once traffic is healthy — without it, you cannot tell a real anomaly from ordinary noise
3. **Alerting** that turns a crossed threshold into a page or a ticket, not a silently-ignored dashboard blip

The loop runs: **observe → detect → diagnose → improve → watch the same metrics to confirm the fix landed.**

**Dashboards vs. alerts:** dashboards are *pull* tools for exploration; alerts are *push* tools that interrupt a human only when a threshold breaks. Good alerts trigger on symptoms users actually feel, set thresholds from the *measured* baseline rather than round-number guesses, and route to someone who can act — a noisy alert teaches the team to ignore the one that eventually matters.

**Watch percentiles, not averages.** A healthy mean can sit comfortably on top of a slow p95/p99 tail that is quietly ruining the experience for the unluckiest fraction of users. On cost: split input from output tokens, track spend per request *and* per user, and watch cache hit rate — a rising cost curve is often the earliest sign of a prompt that quietly grew over a dozen small edits.

**Separate three failure kinds in your dashboards:**
1. **Infrastructure errors** (API failures, rate limits) — fixed in the platform layer
2. **Refusals** — a prompt or policy signal, not an infra bug
3. **Silent wrong answers** — pass every status check, and only quality metrics catch them

Detect drift by sampling live traffic on a schedule, re-scoring with the **same graders** your offline evals use, and comparing against the launch baseline. Metrics say *what* broke; diagnosis says *why*; and captured production failures should feed back in as new eval test cases — closing the loop from Domain 1's feedback stage.

---

## Worked Example: Eval Suite for the Telecom Triage Agent

Building on Domains 1–3's architecture:

| Layer | Metric | Grading | Threshold |
|---|---|---|---|
| Routing classifier | Category accuracy | Code-based (exact match against golden label) | ≥ 92% |
| P1-outage vote | Recall on P1 cases | Code-based | ≥ 99% (false-negative ceiling) |
| Diagnostic agent transcript | Process check: did it ask a clarifying question when signals conflicted? | LLM-judge, rubric-based, isolated dimension | Avg ≥ 4/5 |
| Final routing tag | Outcome check: correct queue | Code-based | ≥ 90% |
| Latency | p95 end-to-end | Operational (OTel) | < 12s (agentic path), < 4s (fixed-chain path) |
| Cost | $/resolved ticket | Operational | Tracked, not gated |
| Safety | Refusal correctness on adversarial ticket set | Code-based + judge escape hatch | ≥ 99% |

Eval dataset seeded with: normal tickets (60%), ambiguous multi-cause tickets (20%), adversarial/malformed input (10%), irrelevant/off-topic tickets (10%) — mirroring the four mandatory edge-case categories from §1.3. Champion-challenger promotion requires the challenger to clear every row above on the frozen held-out set *and* not regress on the frozen baseline suite before a canary release begins.

---

## Key Takeaways

- SMART criteria without time-bound; every target is a number on a held-out set, paired one-to-one with its grader.
- Grading hierarchy: code first, judge second, human last — favor volume with cheap automated grading over a small hand-graded set.
- LLM-as-judge needs a different model, calibration, an escape hatch, and one isolated judge per dimension.
- A/B tests change exactly one variable against an identical eval set — anything else is eyeballing.
- Diagnose the layer first: integration failure (status/error/request ID) vs. model-output problem (stop reason/details/content). A 200 can still be wrong.
- Never swap the model first — check prompt, hallucination, model fit, and retrieval, in that order, and retrieval is the dominant RAG failure origin.
- Pull accuracy-neutral levers (effort, caching, batching, concise output) before trading real accuracy away; batch stacks with caching for a genuine 2x-plus saving on async work.
- Evaluation is pre-deployment; monitoring is continuous. You need both, and production failures should become new eval cases.

---

## References

1. Anthropic Docs, *"Define success criteria"* — https://docs.claude.com/en/docs/build-with-claude/define-success
2. Anthropic Docs, *"Develop test cases"* — https://docs.claude.com/en/docs/test-and-evaluate/develop-tests
3. Anthropic Docs, *"Using the Evaluation tool"* — https://docs.claude.com/en/docs/test-and-evaluate/eval-tool
4. Anthropic Docs, *"Reduce hallucinations"* — https://docs.claude.com/en/docs/test-and-evaluate/strengthen-guardrails/reduce-hallucinations
5. Anthropic Docs, *"Message Batches API"* — https://docs.claude.com/en/docs/build-with-claude/batch-processing
6. Google SRE Book, *"Monitoring Distributed Systems"* — https://sre.google/sre-book/monitoring-distributed-systems/
