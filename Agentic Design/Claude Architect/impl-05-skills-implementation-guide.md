# Implementation Guide 5: Building Skills

> A Skill packages procedural knowledge — instructions plus optional bundled resources — that Claude invokes **autonomously** when a task matches its description. This guide covers the format, the progressive-disclosure mechanics that keep idle Skills nearly free, and how a Skill differs in practice from a subagent, a plugin, or a slash command.

---

## 1. Anatomy of a Skill

```
triage-checklist/
├── SKILL.md              # required — description + full instructions
├── reference/
│   └── escalation_matrix.md   # loaded only if the instructions reference it
└── scripts/
    └── format_summary.py       # loaded/executed only on demand
```

```markdown
---
name: triage-checklist
description: >
  Use when triaging a telecom network-fault ticket. Walks through the
  standard diagnostic checklist and produces a structured escalation
  summary if the issue can't be resolved automatically.
---

# Network Fault Triage Checklist

When a ticket matches this Skill's description, follow these steps:

1. Confirm the four required facts are present: device type, onset time,
   pattern (WiFi-only vs. all-devices), and any recent account changes.
   If any are missing, ask for exactly the missing one(s) — not a generic
   "tell me more."
2. Check `reference/escalation_matrix.md` for the correct escalation
   queue given the domain and confidence level.
3. If escalating, format the summary using `scripts/format_summary.py`
   rather than hand-writing it, to keep the schema consistent for the
   ticketing system's ingestion.
```

## 2. Progressive disclosure — why idle Skills are nearly free

```
Level 1: description        → always in context (a few dozen tokens)
Level 2: full SKILL.md body → loaded only when the model decides this Skill matches
Level 3: bundled resources  → loaded only when the instructions actually reference them
```

This three-level structure is what makes it safe to have dozens of Skills available without capability bloat (Domain 3, §1.1) — the token cost scales with *how many Skills actually fire*, not how many exist in the catalog. Compare this directly to the tool-bloat problem from Domain 3: a poorly-scoped tool catalog costs tokens on every turn regardless of use; a well-formed Skill catalog does not.

## 3. Writing a description that triggers correctly

The `description` field is the *only* thing always in context, so it carries the entire burden of correct triggering — get this wrong and the Skill either never fires or fires on the wrong tasks.

**Weak description** (too vague, competes with everything):
```yaml
description: Helps with support tickets.
```

**Strong description** (specific triggers, explicit scope):
```yaml
description: >
  Use when triaging a telecom network-fault ticket — specifically for
  connectivity, WiFi, or modem issues. Not for billing or provisioning
  tickets (see billing-checklist Skill for those).
```

This mirrors Domain 2, §2.2's "positive instructions over prohibitions" guidance, adapted for triggering rather than behavior — name what it's *for*, then narrow with what it's explicitly *not* for when two Skills could otherwise be confused for each other (the same wrong-tool-selection failure mode from Domain 3, §1.1's tool-bloat diagnostics, now applied to Skills).

## 4. Skill vs. subagent vs. plugin vs. slash command

| Construct | Trigger | Context | Distribution |
|---|---|---|---|
| **Skill** | Autonomous — model matches description | Loads *into* the current context | Personal, project, or plugin scope |
| **Subagent** | Explicit dispatch by the orchestrator | Runs in its *own* fresh, isolated context | `.claude/agents/*.md` or SDK `agents` dict |
| **Slash command** | Explicit, user-typed invocation | Runs in the current context, like a Skill | `.claude/commands/*.md` |
| **Plugin** | N/A — a packaging container | Bundles Skills/tools/commands together | Installed as one unit (Guide 4, §4) |

**The Skill-vs-subagent distinction is the one people mix up most:** a Skill adds knowledge to the *current* loop; a subagent spawns a *new*, isolated loop. Use a Skill when the task can be done within the existing context cheaply. Use a subagent when the investigation is messy enough that you want its exploration contained and only a distilled result returned (Domain 1, §3.2 and Implementation Guide 1, §4).

## 5. On the Claude API, a Skill cannot make live network calls

This is a hard constraint worth designing around explicitly (Domain 3, §3.1): a Skill's `scripts/` can process, format, and validate data already in context, but if the task needs *fresh* external data (a live signal reading, a current firmware version), pair the Skill with an MCP server or custom tool — the Skill provides the *procedure*, the tool provides the *data*.

```markdown
<!-- inside SKILL.md -->
3. Call the `mcp__network-tools__check_signal_strength` tool to get the
   current reading — do not estimate or use a cached value from earlier
   in the conversation, signal strength changes minute to minute.
```

## 6. Scoping: personal, project, plugin

| Scope | Location | Reach |
|---|---|---|
| **Personal** | `~/.claude/skills/` | Just you, across all your projects |
| **Project** | `.claude/skills/` (committed) | Whole team, this repo only |
| **Plugin** | Bundled in a plugin (Guide 4, §4) | Anyone who installs the plugin, org-wide |

**Because invocation is autonomous, review a shared Skill's instructions and any scripts it bundles before trusting it** — this is Domain 2, §4.6's warning made concrete: a project- or plugin-scoped Skill runs with whatever access the current session has, the moment its description matches, with no separate confirmation step. Treat onboarding a new shared Skill with the same scrutiny as approving a new MCP server.

## 7. Testing a Skill before sharing it

- **Trigger precision**: run 5–10 tasks that *should* match the description and 5–10 that deliberately shouldn't (adjacent-but-different tasks), and confirm the Skill fires only on the first set — this is the Domain 4 eval-dataset discipline (representative traffic plus deliberate edge cases) applied to a Skill's own triggering behavior.
- **Instruction completeness**: have someone unfamiliar with the underlying task follow only the `SKILL.md` instructions manually — any step they have to guess at is a gap in the instructions, not something to leave to the model's judgment.
- **Resource loading**: confirm bundled scripts and reference files are actually reachable relative to the Skill's own directory, not the caller's working directory — a common packaging bug that only surfaces once the Skill is installed somewhere other than where it was authored.

---

## Worked Example: The Triage Checklist Skill, Fully Wired

```
network-triage-plugin/
├── plugin.json
├── .mcp.json                          # network-tools MCP server (Guide 4)
└── skills/
    └── triage-checklist/
        ├── SKILL.md                    # the procedure (this guide)
        └── reference/
            └── escalation_matrix.md    # loaded only when step 2 fires
```

When a ticket arrives in a session with this plugin installed:
1. The model reads only the Skill's **description** by default (cheap).
2. It recognizes a network-fault ticket, matches the description, and loads the full `SKILL.md` body (Level 2).
3. Step 2 of the instructions references `escalation_matrix.md` — only now does that file load (Level 3).
4. The instructions call an MCP tool from the bundled `.mcp.json` for a live signal reading — the Skill provides the procedure, the tool provides the data (§5).

No step here required a subagent — the whole task fits comfortably in the current context, which is exactly the signal (per §4's table) that a Skill, not a subagent, was the right construct.

---

## Key Takeaways

- Progressive disclosure (description → instructions → resources) is what keeps idle Skills nearly free — design the description carefully, since it's the only part that's always paying rent.
- A Skill adds knowledge into the current loop; a subagent spawns an isolated new one. Pick based on whether the task's mess needs to be contained.
- Skills can't make live network calls on the API — pair with an MCP server or custom tool whenever the task needs fresh external data.
- Review a shared Skill's instructions and scripts before trusting it, the same as you would a new MCP server or plugin — invocation is autonomous, with no separate confirmation step.
- Test trigger precision with both should-fire and shouldn't-fire cases before sharing a Skill beyond personal scope.

---

## References

1. Anthropic Docs, *"Agent Skills overview"* — https://docs.claude.com/en/docs/agents-and-tools/agent-skills/overview
2. Anthropic Docs, *"Skill authoring best practices"* — https://docs.claude.com/en/docs/agents-and-tools/agent-skills/best-practices
3. Anthropic Engineering, *"Equipping agents for the real world with Agent Skills"* — https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills
