# Implementation Guide 3: Implementing Workflow Patterns & Multi-Agent Orchestration

> Domain 1 named five workflow patterns and the orchestrator-worker topology. This guide implements each one, so the pattern names stop being exam vocabulary and start being code you'd actually ship.

---

## 1. Prompt chaining

Fixed sequence, programmatic gate between each step. Use when the sequence is known and each step's output feeds the next.

```python
async def triage_chain(ticket_text: str) -> dict:
    symptoms = await call_claude(
        system="Extract structured symptoms as JSON: {devices, onset, pattern}.",
        user=ticket_text,
    )
    if not symptoms.get("devices"):                       # programmatic gate
        raise ValueError("Extraction failed — missing required field")

    domain = await call_claude(
        system="Classify into: modem_auth, wifi_local, backhaul, billing.",
        user=json.dumps(symptoms),
    )
    diagnosis = await call_claude(
        system=f"Diagnose root cause within the {domain} domain.",
        user=json.dumps(symptoms),
    )
    return await call_claude(
        system="Produce a routing tag from this diagnosis.",
        user=json.dumps(diagnosis),
    )
```

Each step is a plain, non-agentic API call. The "programmatic gate" (the `if not symptoms.get(...)` check) is what makes this a *workflow* and not just four calls in a row — it's your code validating output before committing to the next step, exactly Domain 1's "code owns control flow" definition.

## 2. Routing

```python
ROUTES = {
    "outage": diagnose_outage_agent,
    "billing": billing_chain,
    "provisioning": provisioning_chain,
}

async def route(ticket_text: str):
    category = await call_claude(     # cheap-tier classifier
        model="claude-haiku-4-5",
        system="Classify this ticket into exactly one category: outage, billing, provisioning.",
        user=ticket_text,
    )
    handler = ROUTES[category.strip()]
    return await handler(ticket_text)
```

The classifier is deliberately the cheapest tier available (Domain 2, §1.1) — routing is bounded, high-volume work that rarely needs flagship reasoning.

## 3. Parallelization — sectioning and voting

```python
# Sectioning — independent subtasks, predefined in code, run concurrently
async def parallel_diagnostics(ticket_text: str):
    signal_check, firmware_check, account_check = await asyncio.gather(
        call_claude(system="Assess signal-related risk.", user=ticket_text),
        call_claude(system="Assess firmware-related risk.", user=ticket_text),
        call_claude(system="Assess account/billing risk.", user=ticket_text),
    )
    return merge_findings(signal_check, firmware_check, account_check)

# Voting — same task, run N times, aggregate for confidence
async def is_p1_outage(ticket_text: str) -> bool:
    votes = await asyncio.gather(*[
        call_claude(system="Is this a P1 network outage? Answer yes/no.", user=ticket_text)
        for _ in range(3)
    ])
    yes_votes = sum(1 for v in votes if v.strip().lower() == "yes")
    return yes_votes >= 2   # majority
```

Note the defining difference from an orchestrator (§5): **the three subtasks in `parallel_diagnostics` are hardcoded in the function** — known before any call runs. Nothing about the input changes which calls fire.

## 4. Evaluator-optimizer

```python
async def generate_and_refine(prompt: str, max_iterations: int = 3) -> str:
    draft = await call_claude(system="Draft a customer-facing resolution summary.", user=prompt)
    for _ in range(max_iterations):
        critique = await call_claude(
            system="Judge this summary against: clarity, no jargon, includes next steps. "
                    "Respond APPROVED or list specific issues.",
            user=draft,
        )
        if critique.strip().startswith("APPROVED"):
            break
        draft = await call_claude(
            system="Revise the summary to address this feedback.",
            user=f"Draft:\n{draft}\n\nFeedback:\n{critique}",
        )
    return draft
```

**Always cap iterations explicitly** (`max_iterations`) — this is Domain 2, §3.4's "every self-critique loop needs an explicit stopping condition," now as an actual loop bound rather than a design principle on paper.

## 5. Orchestrator-workers (runtime decomposition)

This is the one pattern that crosses into agentic territory — the orchestrator itself reasons about what subtasks to create, rather than your code hardcoding them.

```python
async def orchestrate(complex_ticket: str):
    plan = await call_claude(
        system=(
            "Break this ticket into 1-4 independent investigation subtasks. "
            "Return JSON: [{\"subtask\": str, \"specialist\": str}]"
        ),
        user=complex_ticket,
    )
    subtasks = json.loads(plan)

    async def run_worker(subtask):
        return await call_claude(
            system=f"You are a {subtask['specialist']} specialist. Investigate and report findings only.",
            user=subtask["subtask"],
        )

    findings = await asyncio.gather(*[run_worker(t) for t in subtasks])   # dispatch
    return await call_claude(                                              # synthesis
        system="Synthesize these specialist findings into one coherent diagnosis.",
        user=json.dumps(dict(zip([t["subtask"] for t in subtasks], findings))),
    )
```

Compare this directly to `parallel_diagnostics` in §3: structurally almost identical (fan-out, gather, merge) — but here the **subtask list itself comes from a model call**, not from source code. This is exactly the "same fan-out shape, different timing" distinction from Domain 1, §2.3, made visible in the diff between these two functions.

**In production, use the Agent SDK's native subagent support (Implementation Guide 1, §4) instead of hand-rolling this** — it gives you proper context isolation, cost attribution per worker (via OTel spans, Domain 3, §3.3), and the four-value permission system for free. The hand-rolled version above is useful for understanding the pattern; don't ship it as-is for anything beyond a prototype.

## 6. The escalation ladder, as a decision function

Domain 1, §2.4's ladder made runnable — a lightweight router that keeps you honest about not reaching for an agent by default:

```python
def select_pattern(task_profile: dict) -> str:
    if task_profile["fits_single_call"]:
        return "single_prompt"
    if task_profile["needs_retrieval_or_tools"] and task_profile["sequence_is_known"]:
        return "augmented_llm"
    if task_profile["sequence_is_known"]:
        return "fixed_workflow"
    if task_profile["next_step_depends_on_prior_result"]:
        return "agent"
    return "fixed_workflow"   # default to the cheaper, more testable option
```

This isn't meant to run in production — it's a design-review artifact: fill in `task_profile` for a candidate feature during architecture review, and if it doesn't clearly land on `"agent"`, don't build one.

---

## Key Takeaways

- Prompt chaining's defining feature is the programmatic gate between steps — without it, you just have four calls in a row with no validation.
- Sectioning and orchestrator-workers share a fan-out/gather shape; the only real difference is whether the subtask list is hardcoded or generated by a model call.
- Voting aggregates N independent runs of the *same* task for confidence — don't confuse it with sectioning, which splits *different* subtasks.
- Every evaluator-optimizer loop needs an explicit iteration cap in code, not just a design intention.
- Hand-rolled orchestration is for understanding the pattern; production multi-agent work should use the SDK's native subagent support for isolation, attribution, and permissions.

---

## References

1. Anthropic Engineering, *"Building Effective Agents"* — https://www.anthropic.com/engineering/building-effective-agents
2. Anthropic Cookbook, *"Patterns for building effective agents"* — https://github.com/anthropics/anthropic-cookbook/tree/main/patterns/agents
3. Anthropic Engineering, *"How we built our multi-agent research system"* — https://www.anthropic.com/engineering/built-multi-agent-research-system
