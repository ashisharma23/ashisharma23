# Implementation Guide 6: End-to-End Reference Build

> This guide assembles Implementation Guides 0–5 into one buildable repository structure for the running telecom fault-triage example, and maps every file back to the domain and implementation concept it implements. Use it as a starting scaffold, not a finished product — swap the telecom specifics for your own domain and the skeleton holds.

---

## 1. Repository layout

```
telecom-triage-agent/
├── CLAUDE.md                              # project memory (Impl. Guide 2, §3)
├── .claude/
│   ├── settings.json                      # permissions, deny rules (Domain 5 §1.1, 7 §1.3)
│   ├── agents/
│   │   └── diagnostic-specialist.md       # subagent (Domain 1 §3, Impl. Guide 1 §4/2 §6)
│   ├── commands/
│   │   └── triage.md                      # slash command (Impl. Guide 2 §7)
│   └── hooks/
│       └── block-destructive.sh           # deterministic guardrail (Domain 5 §1.1)
├── .mcp.json                              # project-scoped MCP servers (Impl. Guide 4 §3)
├── mcp-servers/
│   └── network/                           # custom MCP server (Impl. Guide 4 §2)
│       └── src/index.ts
├── skills/
│   └── triage-checklist/
│       └── SKILL.md                       # Skill (Impl. Guide 5)
├── src/
│   ├── workflows/
│   │   ├── routing.py                     # routing pattern (Impl. Guide 3 §2)
│   │   ├── p1_vote.py                     # voting pattern (Impl. Guide 3 §3)
│   │   └── fixed_chain.py                 # prompt chaining (Impl. Guide 3 §1)
│   ├── agents/
│   │   └── diagnostic_agent.py            # Agent SDK entrypoint (Impl. Guide 1 §2, 8)
│   ├── tools/
│   │   ├── network_tools.py               # custom tools (Impl. Guide 1 §3)
│   │   └── ticket_tools.py
│   └── observability/
│       └── otel_setup.py                  # OTel spans (Domain 3 §3.3)
├── evals/
│   ├── golden_tickets.jsonl               # held-out eval set (Domain 4 §1.1, 1.3)
│   ├── graders/
│   │   ├── routing_accuracy.py            # code-based grading
│   │   └── diagnostic_quality_judge.py    # LLM-as-judge, isolated dimension
│   └── run_eval_suite.py                  # champion-challenger harness (Domain 4 §2.4)
├── docs/
│   ├── adr/
│   │   └── ADR-001-routing-vs-agent-split.md   # Domain 6 §2.4
│   └── runbook.md                          # Domain 6 §2.6, Domain 7 §3
└── .github/workflows/
    └── claude-ci-review.yml                # headless mode CI step (Impl. Guide 2 §11)
```

---

## 2. Wiring the pieces together

```python
# src/agents/diagnostic_agent.py
from claude_agent_sdk import ClaudeAgentOptions, query
from src.tools.network_tools import network_server
from src.observability.otel_setup import instrument

@instrument("diagnostic-agent")   # Domain 3 §3.3: interaction span with session attrs
async def run_diagnostic(ticket_text: str, confidence_gate=0.6):

    async def pre_tool_hook(input_data, tool_use_id, context):
        # Domain 5 §3.4 escalation-threshold gate, implemented as a hook
        if input_data["tool_name"].endswith("escalate_to_engineer"):
            conf = input_data["tool_input"].get("confidence", 1.0)
            if conf < confidence_gate:
                return {"permissionDecision": "ask", "reason": "Below confidence gate."}
        return {"permissionDecision": "allow"}

    options = ClaudeAgentOptions(
        model="claude-sonnet-4-6",
        system_prompt=open("prompts/diagnostic_system_prompt.txt").read(),
        mcp_servers={"network": network_server},
        allowed_tools=[
            "mcp__network__check_signal_strength",
            "mcp__network__query_firmware_version",
            "mcp__network__escalate_to_engineer",
        ],
        hooks={"PreToolUse": [pre_tool_hook]},
        max_turns=8,                                       # Domain 5 §2.3 step cap
    )
    async for message in query(prompt=ticket_text, options=options):
        yield message
```

```python
# src/workflows/routing.py — Domain 1 §2.4: workflow before agent, always
from src.workflows.fixed_chain import fixed_triage_chain
from src.agents.diagnostic_agent import run_diagnostic
from src.workflows.p1_vote import is_p1_outage

async def handle_ticket(ticket_text: str, category: str):
    if category != "outage":
        return await fixed_triage_chain(ticket_text)        # Domain 1 §2.4 workflow rung

    if await is_p1_outage(ticket_text):                      # parallel vote (Impl. Guide 3 §3)
        return [msg async for msg in run_diagnostic(ticket_text)]  # only P1 earns the agent

    return await fixed_triage_chain(ticket_text)
```

This is the escalation ladder from Domain 1, §2.4 and Implementation Guide 3, §6, made literal in application code: the agent (`run_diagnostic`) is reached **only** on the branch that actually needs runtime-dependent reasoning — every other path stays a cheap, testable, fixed workflow.

---

## 3. The eval harness (Domain 4, made runnable)

```python
# evals/run_eval_suite.py
import json
from src.workflows.routing import handle_ticket
from evals.graders.routing_accuracy import grade_routing
from evals.graders.diagnostic_quality_judge import judge_diagnostic

async def run_suite(golden_path="evals/golden_tickets.jsonl"):
    results = {"routing_accuracy": [], "diagnostic_quality": [], "p1_recall": []}
    with open(golden_path) as f:
        for line in f:
            case = json.loads(line)
            output = await handle_ticket(case["ticket_text"], case["category"])
            results["routing_accuracy"].append(grade_routing(output, case["golden_queue"]))
            if case["category"] == "outage":
                results["diagnostic_quality"].append(await judge_diagnostic(output, case["rubric"]))
    return {k: sum(v) / max(len(v), 1) for k, v in results.items()}

# Champion-challenger gate (Domain 4 §2.4): promote only if every metric clears its bar
# AND the frozen baseline suite from the previous champion still passes.
```

This harness is what a challenger prompt or model version must clear (Domain 4, §2.1–2.4) before it's allowed to become the new production champion — wired directly into the CI workflow at `.github/workflows/claude-ci-review.yml` so no promotion happens on a manual "looks good to me."

---

## 4. Observability wiring (Domain 3, §3.3, made runnable)

```python
# src/observability/otel_setup.py
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.http.trace_exporter import OTLPSpanExporter

provider = TracerProvider()
provider.add_span_processor(BatchSpanProcessor(OTLPSpanExporter(endpoint="http://otel-collector:4318/v1/traces")))
trace.set_tracer_provider(provider)
tracer = trace.get_tracer("telecom-triage-agent")

def instrument(interaction_name: str):
    def decorator(fn):
        async def wrapper(*args, **kwargs):
            with tracer.start_as_current_span(interaction_name) as span:
                span.set_attribute("service.name", "telecom-triage-agent")
                span.set_attribute("service.tenant", kwargs.get("tenant_id", "default"))
                async for item in fn(*args, **kwargs):
                    yield item
        return wrapper
    return decorator
```

**A process must flush the `BatchSpanProcessor` on exit** (Domain 3, §3.3) — register a shutdown handler, or the last spans of any run (often the most diagnostically useful ones, right before a crash) are silently lost.

---

## 5. Governance checklist before production launch (Domain 5 + 6, as a gate)

```markdown
## Pre-launch gate — telecom-triage-agent

- [ ] Eval suite passing on a held-out set that mirrors real traffic (Domain 4 §1.3)
- [ ] SLOs wired into OTel dashboards BEFORE launch, not after (Domain 6 §3.1)
- [ ] Rollback path tested, not just documented (Domain 6 §3.1)
- [ ] Every tool scoped to least privilege; no generic passthrough tools (Domain 3 §1.2, Impl. Guide 4 §2.3)
- [ ] Deny rules cover destructive actions in .claude/settings.json (Domain 7 §1.3)
- [ ] HITL escalation gate wired for confidence < 0.6 AND all P1 cases (Domain 5 §3.4)
- [ ] GDPR: Article 22 human-review path confirmed reachable from the escalation gate
      above, not a separate unused mechanism (Domain 5 §3.5)
- [ ] ADR-001 written and reviewed: routing-vs-agent split, consequences section
      includes the accepted downside (Domain 6 §2.4)
- [ ] Runbook complete: deploy / monitor / respond / rollback, written for on-call,
      not the original author (Domain 6 §2.6)
- [ ] Named owner assigned for escalated-decision accountability (Domain 5 §4.5)
```

---

## 6. What to build first (a realistic sequencing)

For a team starting from zero, build in this order — each step is usable on its own and de-risks the next:

1. **Fixed chain only** (Implementation Guide 3, §1) — no agent, no MCP, prove the domain classification and routing logic works at all.
2. **Add the eval harness** (§3 above) before adding any complexity — you need the held-out set in place before you can tell whether the next step is actually an improvement.
3. **Add routing + voting** (Implementation Guide 3, §2–3) — still no agent.
4. **Add exactly one custom tool**, called from a plain API call, no agentic loop yet — prove the integration works in isolation.
5. **Wrap the tool in an MCP server** only once you have a second consumer that needs it, or a clear team-reuse requirement (Implementation Guide 4, §1's decision tree).
6. **Introduce the agentic diagnostic branch** (Implementation Guide 1) — the *last* piece added, and only on the one branch that demonstrably needs it (Domain 1's escalation ladder, respected in build order, not just in the diagram).
7. **Add hooks, HITL gates, and the compliance checklist** (§5 above) before any production traffic — not as a follow-up sprint.

This sequencing is the practical enforcement of Domain 1's core lesson: the simplest thing that clears the bar, built and validated first, with the agent as the last piece added rather than the first thing scaffolded.

---

## Key Takeaways

- Every file in a well-structured agentic repo traces back to a specific domain concept and implementation guide — if a file doesn't, ask what it's actually for.
- The agent is reached on exactly one branch of the running example; everything else stays a cheap, testable, fixed workflow — build order should reflect this, not just the final diagram.
- The eval harness should exist *before* the agentic branch does, so every addition after that point is measured against a real baseline, not vibes.
- OTel spans must be flushed on exit or the most useful diagnostic spans — the ones right before a failure — are the ones you lose.
- A pre-launch governance checklist that references specific domain sections (not generic "review security") is what actually gets enforced under deadline pressure.

---

## References

1. Anthropic Engineering, *"Building Effective Agents"* — https://www.anthropic.com/engineering/building-effective-agents
2. Anthropic Docs, *"Agent SDK overview"* — https://docs.claude.com/en/api/agent-sdk/overview
3. Anthropic Docs, *"Claude Code overview"* — https://docs.claude.com/en/docs/claude-code/overview
4. Anthropic Docs, *"Model Context Protocol (MCP)"* — https://docs.claude.com/en/docs/agents-and-tools/mcp
5. Anthropic Docs, *"Agent Skills overview"* — https://docs.claude.com/en/docs/agents-and-tools/agent-skills/overview
6. OpenTelemetry, *official documentation* — https://opentelemetry.io/docs/
