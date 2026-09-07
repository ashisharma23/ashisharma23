# Implementation Guide 2: Claude Code — Team Configuration & Daily Workflow

> Claude Code is the interactive/CLI harness built on the same engine as the Agent SDK (Guide 1). This guide covers rolling it out to a team correctly (Domain 7, §1) and using it in the five-stage development loop (Domain 7, §2).

---

## 1. Install and first run

```bash
npm install -g @anthropic-ai/claude-code
claude    # launches interactive session in the current directory
```

Non-interactive / CI usage:
```bash
claude -p "Fix the failing test in test_routing.py" --output-format json
```

## 2. Project structure and settings hierarchy

```
your-repo/
├── .claude/
│   ├── settings.json          # committed — project-level, shared by the team
│   ├── settings.local.json    # gitignored — personal overrides
│   ├── hooks/                 # PreToolUse / PostToolUse scripts
│   ├── agents/                # subagent definitions (.md files)
│   ├── commands/               # custom slash commands
│   └── skills/                 # project-scoped Skills (see Guide 5)
├── CLAUDE.md                   # committed — project memory, always loaded
└── .mcp.json                   # committed — project-scoped MCP servers
```

**Settings resolve highest-wins (Domain 7, §1.2):**
```
managed (org-pushed) > CLI flags > settings.local.json > .claude/settings.json > ~/.claude/settings.json
```

```json
// .claude/settings.json — committed, shared by the whole team
{
  "permissions": {
    "allow": ["Bash(npm test:*)", "Bash(npm run lint:*)"],
    "deny": ["Bash(rm -rf:*)", "Bash(git push --force:*)"]
  },
  "model": "claude-sonnet-4-6"
}
```

**Enterprise lockdown:** an org-managed policy setting `allowManagedPermissionRulesOnly: true` makes every local and project rule inert — only org-pushed rules apply. This is the correct control when individual developer discretion over permissions is not acceptable for compliance reasons (Domain 5, §3.5).

## 3. `CLAUDE.md` — project memory

```markdown
# CLAUDE.md

## Project
Telecom fault-triage service. Python 3.12, FastAPI, deployed via Claude Agent SDK.

## Conventions
- All new tools go in `src/tools/`, namespaced by service (`network.*`, `billing.*`).
- Run `pytest tests/ -x` before any commit.
- Never hardcode customer PII in test fixtures — use `tests/fixtures/synthetic/`.

## Architecture notes
See `docs/adr/` for accepted decisions. ADR-003 explains why routing uses a
fixed workflow while diagnosis uses an agent (Domain 1, §2.4 pattern).
```

**Remember: `CLAUDE.md` *persuades*, it does not enforce (Domain 7, §1.3).** A rule the model must never violate — not "should usually follow" — belongs in a deny rule or a hook, not here. Keep `CLAUDE.md` for facts and conventions; keep hard constraints in `settings.json` and `.claude/hooks/`.

## 4. Hooks — the deterministic layer, at the CLI level

```json
// .claude/settings.json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [{ "type": "command", "command": ".claude/hooks/block-destructive.sh" }]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [{ "type": "command", "command": ".claude/hooks/run-lint.sh" }]
      }
    ]
  }
}
```

```bash
#!/usr/bin/env bash
# .claude/hooks/block-destructive.sh
input=$(cat)
cmd=$(echo "$input" | jq -r '.tool_input.command // ""')
if echo "$cmd" | grep -qE 'rm -rf|git push --force'; then
  echo '{"permissionDecision": "deny", "reason": "Destructive command blocked."}'
  exit 0
fi
echo '{"permissionDecision": "allow"}'
```

This is the identical mechanism from Guide 1, §5, expressed at the CLI/settings layer instead of in SDK code — the same `PreToolUse` event, the same four-value decision contract.

## 5. MCP servers at project scope

```json
// .mcp.json — committed, whole team shares the definition
{
  "mcpServers": {
    "network-tools": {
      "command": "node",
      "args": ["./mcp-servers/network/dist/index.js"],
      "env": { "NETWORK_API_BASE": "https://internal.api/network" }
    },
    "postgres-readonly": {
      "type": "http",
      "url": "https://mcp.internal.example.com/postgres"
    }
  }
}
```

**Committing this file does not grant blanket trust** — each developer approves the server on first use, and workspace trust gates the whole project, so a malicious commit to `.mcp.json` can't silently execute on checkout (Domain 7, §1.4).

## 6. Subagents as `.md` files

```markdown
---
name: diagnostic-specialist
description: Investigates WiFi and connectivity issues using signal, firmware, and driver data.
tools: mcp__network-tools__check_signal, mcp__network-tools__query_firmware
model: claude-sonnet-4-6
---

You are a network diagnostic specialist. Given a ticket and any prior findings,
determine the most likely root cause among: local interference, access-point
performance, WiFi driver issue, or ISP backhaul contention. Ask for exactly one
piece of diagnostic data at a time if the evidence is ambiguous.
```

Saved to `.claude/agents/diagnostic-specialist.md`, this becomes callable by the lead session exactly like the SDK's `agents` dict in Guide 1, §4 — same isolation guarantees, same "only the final message returns" rule.

## 7. Custom slash commands

```markdown
<!-- .claude/commands/triage.md -->
---
description: Run full triage on a ticket ID
---
Fetch ticket $ARGUMENTS from the ticketing system, then run the standard
triage workflow: extract symptoms, classify domain, and if ambiguous,
delegate to the diagnostic-specialist subagent.
```

Invoked as `/triage 48213` in an interactive session — a reusable, versioned, team-shared shortcut for a common request, distinct from a Skill (Guide 5) in that a command is explicitly invoked, while a Skill is autonomously triggered by the model recognizing a match.

## 8. Plan mode in daily use

```bash
claude --permission-mode plan
```

Research-only: Claude proposes an approach, cannot edit files or run state-changing commands until you approve (Domain 7, §2.2). Use this for any change with real blast radius — a schema migration, a permission change, a refactor touching shared code — before switching to normal mode to execute the approved plan.

## 9. The five-stage loop, mapped to Claude Code commands

| Stage | Claude Code mechanism |
|---|---|
| 1. Research | `--permission-mode plan`, or ask Claude to read and summarize before proposing |
| 2. Agree a plan | Review the plan output; approve or redirect before any edit happens |
| 3. Execute | Normal mode; Claude edits files in small, reviewable steps |
| 4. Review the diff | `git diff` after each step — the human gate (Domain 5, §3.1's HITL, applied to code review) |
| 5. Ship | Commit only after a runnable check passes (§10 below) |

## 10. Closing the loop with a runnable check

```bash
claude -p "Fix the WiFi-detection bug. Reproduce it first with a failing test in tests/test_diagnosis.py, then fix it, then confirm the test passes." --allowedTools "Bash(pytest:*),Edit,Read"
```

Giving Claude a concrete, runnable verification command turns "trust me, it's fixed" into "the test that was failing now passes" — the code-level version of Domain 4's `held-out set` discipline: you don't grade the fix on the model's self-report, you grade it on an independent, executable check.

## 11. Scaling: worktrees, headless CI, and fan-out

```bash
# Parallel sessions — isolated worktrees, no collision
git worktree add ../repo-feature-a feature-a
git worktree add ../repo-feature-b feature-b
(cd ../repo-feature-a && claude -p "Implement feature A" &)
(cd ../repo-feature-b && claude -p "Implement feature B" &)
```

```yaml
# .github/workflows/claude-review.yml — headless mode as a CI step
- name: Claude Code Review
  run: |
    claude -p "Review this PR diff for the patterns in CLAUDE.md" \
      --output-format json \
      --allowedTools "Read,Grep" > review.json
  env:
    ANTHROPIC_API_KEY: ${{ secrets.CLAUDE_CI_TOKEN }}   # scoped token, least privilege
```

The CI token here should be scoped as narrowly as any other credential (Domain 3, §1.2) — a review-only CI step has no business holding a token with write access to production systems.

---

## Key Takeaways

- Settings resolve highest-wins: managed org policy > CLI flags > local overrides > project settings > user settings.
- `CLAUDE.md` persuades; hooks and deny rules enforce. Put anything that must never be violated in the latter.
- `.mcp.json` is committed for team-wide sharing, but each developer still approves a server on first use — commit ≠ blanket trust.
- Plan mode is the CLI's implementation of Domain 7's "wrong plan is cheap, wrong diff is expensive" principle.
- Always close the loop with an independently runnable check — a failing test that then passes — rather than trusting the model's self-report that something is fixed.
- Worktrees for parallel human-driven sessions; headless mode with a scoped token for CI.

---

## References

1. Anthropic Docs, *"Claude Code overview"* — https://docs.claude.com/en/docs/claude-code/overview
2. Anthropic Docs, *"Claude Code settings"* — https://docs.claude.com/en/docs/claude-code/settings
3. Anthropic Docs, *"Hooks reference"* — https://docs.claude.com/en/docs/claude-code/hooks
4. Anthropic Docs, *"Subagents"* — https://docs.claude.com/en/docs/claude-code/sub-agents
5. Anthropic Docs, *"Slash commands"* — https://docs.claude.com/en/docs/claude-code/slash-commands
6. Anthropic Engineering, *"Claude Code: Best practices for agentic coding"* — https://www.anthropic.com/engineering/claude-code-best-practices
