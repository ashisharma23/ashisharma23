# Domain Implementation Reference Guide — Domain 7: Developer Productivity & Operational Enablement

**Source domain:** `domain-7-developer-productivity-operational-enablement.md` (CCAR-P, exam weight 7%)
**Guide type:** Implementation Reference (technical depth, code examples, API/config detail)
**Audience:** Solution architects and engineers implementing Claude Code / Claude Agent SDK–based developer workflows in a team or enterprise setting.

> This guide preserves the full structure of the source domain (1:1 section mapping) and expands every item into: what it is, why it matters, the underlying Anthropic technical mechanism, implementation patterns, pitfalls, and runnable Python/TypeScript examples. A full traceability matrix appears at the end.

---

## Table of Contents

1. [Domain Overview](#domain-overview)
2. [Section 1: Team Enablement](#section-1-team-enablement)
   - 1.1 Configure once at the project level
   - 1.2 Settings resolution order
   - 1.3 Deny rules, hooks, and managed policy (the deterministic layer)
   - 1.4 MCP server scopes
3. [Section 2: Improving Developer Workflows with AI-Assisted Tooling](#section-2-improving-developer-workflows-with-ai-assisted-tooling)
   - 2.1 The five-stage loop
   - 2.2 Plan mode
   - 2.3 Closing the loop with a runnable check
   - 2.4 Capturing workflows as the right asset
   - 2.5 Scaling horizontally
4. [Section 3: Operational Support — Debugging and Incident Resolution](#section-3-operational-support-debugging-and-incident-resolution)
   - 3.1 Framing a bug in three parts
   - 3.2 Symptom fix vs. root-cause fix
   - 3.3 Feeding Claude raw signal
   - 3.4 Live access vs. pasted logs
   - 3.5 Session-hygiene moves for triage
5. [Worked Example: Operational Runbook](#worked-example-operational-runbook)
6. [Architecture & Implementation Best Practices](#architecture--implementation-best-practices)
7. [Common Pitfalls and Limitations](#common-pitfalls-and-limitations)
8. [Official Documentation References](#official-documentation-references)
9. [Domain Coverage / Traceability Matrix](#domain-coverage--traceability-matrix)

---

## Domain Overview

Domain 7 is the smallest domain by exam weight (7%), but it is the domain that determines whether the architecture designed under Domains 1–6 (solution design, model/context strategy, integration, evaluation, governance, stakeholder communication) is actually **built consistently and operated safely** by a real engineering team, day after day, under deadline pressure.

Two threads run through the entire domain and recur in almost every subsection:

1. **Probabilistic vs. deterministic control.** `CLAUDE.md`, project memory, and permission *suggestions* are context — they persuade the model but can be deviated from. Deny rules, hooks (`PreToolUse`, etc.), and org-managed settings are enforced outside the model's control loop and cannot be talked out of. Any requirement that *must* hold belongs in the deterministic layer.
2. **Human-in-the-loop placement.** The five-stage engineering loop, plan mode, and incident-response discipline are all specific instantiations of Domain 5's HITL principle: cheap-to-reverse steps (planning, read-only research) are delegated freely to the model; expensive-to-reverse steps (merging, deploying, executing a fix in production) get an explicit human gate.

This guide treats "Claude Code" as the primary implementation surface for Domain 7, since that is the product surface the source domain's language (settings resolution, plan mode, hooks, MCP scopes, headless mode) maps to directly. Where the same capability is exposed programmatically via the **Claude Agent SDK** (for building custom internal tooling rather than using the Claude Code CLI/IDE integration directly), that mapping is called out explicitly, since many enterprises wrap Claude Code's underlying agent loop into custom internal platforms.

---

## Section 1: Team Enablement

### 1.1 Configure once at the project level

**What it is.** Claude Code reads configuration from multiple locations, and the entire point of the "project level" `.claude/` directory (committed to version control at the repo root) is that a team shares *one* environment rather than each engineer hand-configuring their own CLI.

**Why it matters.** Without this, two developers working on the same repo can get materially different agent behavior — different permission prompts, different available MCP tools, different memory/context — which breaks reproducibility of both the developer experience and, more importantly, the *audit trail* of what an agent was and wasn't allowed to do when it made a given change. This is the direct analogue of Domain 2's "informative yet tight, curated deliberately" context discipline, applied to the team's shared tooling surface instead of a single prompt window.

**What's committed vs. local:**

| Committed at project level (`.claude/` in git) | Stays local (gitignored) |
|---|---|
| `.claude/settings.json` (project settings) | `.claude/settings.local.json` (personal overrides) |
| `CLAUDE.md` (project memory) | Personal scratch memory files |
| `.mcp.json` (MCP server definitions) | Locally-scoped MCP servers (`--scope local`) |
| Permission rules (allow/deny lists) | — |

**Claude Code technical detail — file locations:**

```
<repo-root>/
├── CLAUDE.md                      # project memory, loaded into every session's context
├── .claude/
│   ├── settings.json               # committed, team-shared project settings
│   └── settings.local.json         # gitignored, personal overrides (highest local priority)
├── .mcp.json                       # committed MCP server definitions (project scope)
└── .gitignore                      # MUST include .claude/settings.local.json
```

`CLAUDE.md` can also be layered: a root `CLAUDE.md`, subdirectory `CLAUDE.md` files (loaded when Claude works in that subtree), and a user-level `~/.claude/CLAUDE.md` for personal preferences that apply across all projects. `CLAUDE.md` supports `@path/to/file` import syntax so large policy documents can be referenced rather than inlined, keeping the always-loaded context lean (this is Domain 2's context-budget discipline applied directly to memory files).

**Secrets — the one deliberate exception.** Everything above says "commit it." Secrets are the opposite:

- API keys, tokens, and credentials must live in the OS keychain / credential manager, a secrets manager (AWS Secrets Manager, HashiCorp Vault, GCP Secret Manager), or be injected as short-lived, auto-rotating tokens at runtime via a **key helper** script.
- **Never** place a secret in `.claude/settings.local.json`, `CLAUDE.md`, or any gitignored file in the repo. `.gitignore` is a client-side convention, not a server-side security boundary: one contributor with a misconfigured global gitignore, one `git add -f`, or one history rewrite mistake turns a "local-only" secret into a permanent, unrevocable leak baked into git history and (once pushed) potentially into GitHub's search index and any forks.
- This is the direct implementation of Domain 5 §1.1's access-control layer at the tooling level: the control has to hold even when a well-intentioned developer makes a mistake, so it cannot rely on convention alone.

**Implementation pattern — MCP key helper (conceptual, matches `.mcp.json` + external secret store):**

```json
// .mcp.json — committed. No secrets inline.
{
  "mcpServers": {
    "internal-jira": {
      "command": "node",
      "args": ["./scripts/jira-mcp-server.js"],
      "env": {
        "JIRA_TOKEN_HELPER": "aws secretsmanager get-secret-value --secret-id prod/jira-mcp --query SecretString --output text"
      }
    }
  }
}
```

```python
# scripts/token_helper.py — fetches a short-lived token at runtime, never persisted to disk.
import boto3, json, sys

def get_short_lived_token(secret_id: str) -> str:
    client = boto3.client("secretsmanager")
    resp = client.get_secret_value(SecretId=secret_id)
    return json.loads(resp["SecretString"])["token"]

if __name__ == "__main__":
    print(get_short_lived_token(sys.argv[1]))
```

**Common pitfalls:**
- Committing `.claude/settings.local.json` by accident because it wasn't added to `.gitignore` before the first commit of the repo.
- Treating a gitignored `.env` file holding an API key as "safe enough" — it is not a control the exam or a real incident postmortem will accept.
- Forgetting that subdirectory `CLAUDE.md` files are additive context, not overrides — a bloated root `CLAUDE.md` plus five subdirectory `CLAUDE.md` files can silently blow the context budget for every session in that subtree.

---

### 1.2 Settings resolution order

**What it is.** Claude Code merges configuration from five sources. When the same setting is defined in more than one place, **the higher-priority source wins outright** — there is no partial merge of conflicting values within a single key.

```
1. Managed settings (org-pushed, e.g. via MDM / enterprise policy)   ← cannot be overridden by anyone downstream
2. Command-line flags (per-invocation)
3. Personal local override file (.claude/settings.local.json)
4. Committed project settings (.claude/settings.json)
5. Home-directory user settings (~/.claude/settings.json)             ← lowest priority
```

**Why it matters.** This ordering is what makes enterprise governance *actually* enforceable rather than aspirational. If a project's committed settings could be silently overridden by a developer's home directory config, an org could never guarantee that a security rule actually applies everywhere. The ordering guarantees the opposite: the closer a setting is to central IT/security control, the harder it is to override.

**The enterprise lockdown switch: `allowManagedPermissionRulesOnly`.**
When an org sets this in managed settings, **every local and project permission rule is ignored entirely** — only org-pushed rules apply. This is the correct control for regulated environments (finance, healthcare, government contractors) where individual developer discretion over tool permissions is unacceptable, mapping directly to Domain 5 §1.1's access-control layer (principle of least privilege enforced centrally, not per-developer).

**Where managed settings physically live (platform-dependent):**

| OS | Managed settings path |
|---|---|
| macOS | `/Library/Application Support/ClaudeCode/managed-settings.json` |
| Linux | `/etc/claude-code/managed-settings.json` |
| Windows | Pushed via registry / MDM policy, resolved into an equivalent managed settings file |

**Example `managed-settings.json` (org IT/security-controlled):**

```json
{
  "allowManagedPermissionRulesOnly": true,
  "permissions": {
    "allow": [
      "Read(**)",
      "Bash(git status)",
      "Bash(git diff*)"
    ],
    "deny": [
      "Bash(curl:*)",
      "Bash(rm -rf*)",
      "WebFetch(domain:*)"
    ]
  }
}
```

With `allowManagedPermissionRulesOnly: true` set above, no project's `.claude/settings.json` and no developer's local override can add a single additional allow rule — the org's list is the entire universe of permitted actions.

**Verifying effective settings (operational check):**

```bash
# Claude Code exposes the merged, effective configuration for audit purposes
claude config list --effective
```

**Common pitfalls:**
- Assuming project settings are a "floor" that user settings can raise — they cannot; whichever source ranks higher **replaces** the value for that key, it does not merge additively (except where the tool explicitly documents rule lists as unioned, e.g. deny lists are unioned across layers for safety — always deny-wins, never allow-wins, when rules from different layers conflict on the same action; see §1.3).
- Forgetting to check `--effective` output before assuming a rule is in force; a rule defined in a lower-priority file that's shadowed by a higher one will silently never apply.

---

### 1.3 Deny rules, hooks, and managed policy: the only deterministic layer

**What it is.** Claude Code's permission system has two rule types plus one enforcement mechanism outside the permission system entirely:

- **Allow rules** — pre-approve specific, trusted actions (e.g., `Bash(npm test)`) so they execute without an interactive confirmation prompt. They are a *convenience* mechanism, not a security boundary.
- **Deny rules** — hard-block a matched action **regardless of what the model decides to attempt**. A denied action is refused before it ever reaches execution.
- **Hooks** — shell commands that fire on lifecycle events (`PreToolUse`, `PostToolUse`, `UserPromptSubmit`, `Stop`, etc.) and can block, modify, or log an action deterministically, independent of the model's reasoning.

**Why it matters — the critical distinction the exam tests.** `CLAUDE.md` content and permission *guidance* written in prose (e.g., "please don't touch the `payments/` directory") are loaded as **context**. Context persuades; it does not enforce. A sufficiently unusual prompt, a long context window pushing early instructions out of attention, or a subtly adversarial input can all cause the model to deviate from a purely advisory instruction. **Only deny rules, hooks on lifecycle events, and managed policy are genuinely deterministic** — they are evaluated by code outside the model's generation loop, so the model cannot reason its way around them. This exactly mirrors Domain 3 §1.2 ("deny always wins over allow") and Domain 5's defense-in-depth stack: probabilistic self-restraint is a nice-to-have; deterministic gates are the actual control.

**Rule syntax (permission rules, `.claude/settings.json`):**

```json
{
  "permissions": {
    "allow": [
      "Read(**)",
      "Bash(git *)",
      "Bash(npm run test*)"
    ],
    "deny": [
      "Bash(rm -rf*)",
      "Bash(git push --force*)",
      "Write(/etc/**)",
      "WebFetch(domain:*.internal-only.example.com)"
    ]
  }
}
```

**Hooks — the deterministic gate for anything a static rule can't express.** A permission rule matches on a static pattern; a hook can run arbitrary logic (e.g., "block this file write if it touches a file outside the current ticket's declared scope," or "require a passing lint before any `git commit` tool call is allowed to proceed").

**`PreToolUse` hook example — block writes outside an allow-listed directory set (Python hook script):**

```python
#!/usr/bin/env python3
# .claude/hooks/pretooluse_scope_guard.py
# Registered as a PreToolUse hook for the Write and Edit tools.
import json, sys, os

ALLOWED_PREFIXES = ("src/", "tests/", "docs/")

def main():
    payload = json.load(sys.stdin)          # Claude Code passes tool-call context via stdin
    tool_name = payload.get("tool_name")
    tool_input = payload.get("tool_input", {})

    if tool_name in ("Write", "Edit"):
        path = tool_input.get("file_path", "")
        rel_path = os.path.relpath(path)
        if not rel_path.startswith(ALLOWED_PREFIXES):
            # Exit code 2 = deny the tool call deterministically; Claude Code
            # surfaces the stderr message back to the model as the reason.
            print(f"BLOCKED: {rel_path} is outside allowed scope {ALLOWED_PREFIXES}",
                  file=sys.stderr)
            sys.exit(2)

    sys.exit(0)  # allow

if __name__ == "__main__":
    main()
```

```json
// .claude/settings.json — registering the hook
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          { "type": "command", "command": "python3 .claude/hooks/pretooluse_scope_guard.py" }
        ]
      }
    ]
  }
}
```

**Equivalent hook logic via the Claude Agent SDK (TypeScript), for teams building a custom internal agent platform rather than using the Claude Code CLI directly:**

```typescript
// agent-runner.ts — using @anthropic-ai/claude-agent-sdk
import { query, type PreToolUseHook } from "@anthropic-ai/claude-agent-sdk";

const ALLOWED_PREFIXES = ["src/", "tests/", "docs/"];

const scopeGuard: PreToolUseHook = async (input) => {
  if (input.tool_name === "Write" || input.tool_name === "Edit") {
    const path: string = input.tool_input.file_path ?? "";
    const inScope = ALLOWED_PREFIXES.some((p) => path.startsWith(p));
    if (!inScope) {
      return {
        decision: "block",
        reason: `${path} is outside allowed scope [${ALLOWED_PREFIXES.join(", ")}]`,
      };
    }
  }
  return { decision: "allow" };
};

for await (const message of query({
  prompt: "Refactor the retry logic in the payments client",
  options: { hooks: { PreToolUse: [scopeGuard] } },
})) {
  console.log(message);
}
```

**Asset-selection rule this section establishes:** if a rule genuinely *must* hold — no exceptions, no "the model should know better" — it belongs in a deny rule, a hook, or managed policy. If it's guidance that's usually right but can tolerate occasional deviation with human review, `CLAUDE.md` context is the right (cheaper, more flexible) tool. Conflating the two is the single most common Domain 7 exam trap: a scenario describing a security-critical rule placed only in `CLAUDE.md` is *always* the wrong answer.

**Common pitfalls:**
- Writing "never delete files in `prod/`" in `CLAUDE.md` and treating that as a control. It is a suggestion; a matching deny rule or hook is the control.
- Forgetting hook exit-code semantics — exit code 2 with a stderr message is how you deterministically block *and* explain why (the explanation is fed back to the model so it can adapt, but the block itself does not depend on the model accepting the explanation).
- Registering a hook matcher too broadly (e.g., matching all `Bash` calls) and materially slowing down every benign command with unnecessary hook invocation overhead.

---

### 1.4 MCP server scopes

**What it is.** MCP (Model Context Protocol) servers extend Claude Code with external tool access (Jira, internal APIs, databases, monitoring systems — see Domain 3 §3.2). Claude Code scopes server *definitions* at three levels:

| Scope | Reach | Typical config location |
|---|---|---|
| **Local** | Private to you, this machine only | `claude mcp add --scope local ...` (not committed anywhere) |
| **Project** | Ships in a committed file; the whole team shares the *definition* | `.mcp.json` at repo root (committed) |
| **User** | Private to you, but follows you across *all* your projects | `~/.claude/mcp.json` (or platform-equivalent user config dir) |

**Why it matters — the trust boundary is per-developer, not per-repo.** A common misconception: "we committed `.mcp.json`, so the whole team now has this MCP server enabled." **False.** Committing the server *definition* means every developer who clones the repo can *see* the server is available — it does not grant execution trust. **Workspace trust** in Claude Code means a newly-cloned or newly-opened project's MCP servers require an explicit per-developer, first-use approval prompt before Claude Code will actually connect to and invoke tools from that server. This prevents a scenario where cloning a malicious or compromised repository silently grants that repo's authors arbitrary tool execution on your machine the moment you open it.

**Practical implication for architecture design:** if a shared, org-wide MCP server (e.g., a company-wide Salesforce or Slack connector) needs to be available to every developer *without* a first-use prompt, that requires it to be pushed via **managed settings** (org policy), not merely committed to `.mcp.json` — because managed policy is the only layer that can pre-authorize trust across the whole org rather than relying on each individual developer clicking "approve."

**Command-line management:**

```bash
# Add a project-scoped server (writes to .mcp.json, committed)
claude mcp add --scope project internal-jira -- node ./scripts/jira-mcp-server.js

# Add a user-scoped server (private, follows you across projects)
claude mcp add --scope user personal-notes -- npx @myorg/notes-mcp

# Add a local-scoped server (private, this repo/machine only, never committed)
claude mcp add --scope local scratch-db -- python ./local_only_mcp.py

# List effective servers and their scopes for the current session
claude mcp list
```

**Example `.mcp.json` (project scope, committed):**

```json
{
  "mcpServers": {
    "internal-jira": {
      "command": "node",
      "args": ["./scripts/jira-mcp-server.js"],
      "env": { "JIRA_TOKEN_HELPER": "vault-cli read secret/jira" }
    },
    "monitoring": {
      "type": "sse",
      "url": "https://mcp.internal.example.com/monitoring/sse"
    }
  }
}
```

**Common pitfalls:**
- Assuming committing `.mcp.json` is equivalent to org-wide auto-trust — it is not; per-developer first-use approval still gates execution.
- Putting a secret directly in `.mcp.json`'s `env` block instead of a key-helper reference (see §1.1) — this is the same anti-pattern as committing secrets anywhere else in the repo.
- Using `local` scope for something that should be `project` scope, meaning every teammate has to independently rediscover and reconfigure a server that should have been shared.

---

## Section 2: Improving Developer Workflows with AI-Assisted Tooling

### 2.1 The five-stage loop

**What it is.** A fixed sequence for every non-trivial change, deliberately run in full even under deadline pressure:

```
1. RESEARCH        → Claude explores the codebase, reads relevant files, understands context
2. AGREE A PLAN     → Claude proposes an approach; developer reviews and approves before any edit
3. EXECUTE          → Focused, reviewable steps — not one giant undifferentiated diff
4. REVIEW THE DIFF  → Human gate: the developer actually reads the change before it ships
5. SHIP             → After verification (tests, lint, CI) — not before
```

**Why it matters.** This is Domain 5's human-in-the-loop principle (§3.1–3.4) applied to the literal day-to-day unit of engineering work — a code change. **The diff review at stage 4 *is* the HITL gate.** Skipping it because a change "looks small" or "is just a quick fix" is exactly the rubber-stamping failure mode Domain 5 warns against: the entire value of a human gate evaporates the moment the human stops actually exercising judgment at that gate.

**Role split:** Claude does the work (research, drafting, running commands); the developer frames the problem, approves the plan, reviews each diff, and owns the ship-or-rollback call. The developer's judgment is not replaced by the loop — it is *concentrated* at the two decision points (plan approval, diff review) where it has the highest leverage.

**Implementation — enforcing the loop structurally, not just by convention.** A team can encode "plan before execute" as a genuine gate rather than a habit by combining plan mode (§2.2) with a `PreToolUse` hook that requires an approved-plan marker file before any `Write`/`Edit`/`Bash` tool call is allowed to run:

```python
#!/usr/bin/env python3
# .claude/hooks/require_approved_plan.py
import json, sys, pathlib

PLAN_MARKER = pathlib.Path(".claude/.plan_approved")

def main():
    payload = json.load(sys.stdin)
    if payload.get("tool_name") in ("Write", "Edit", "Bash"):
        if not PLAN_MARKER.exists():
            print("BLOCKED: no approved plan on file. Run plan mode and approve first.",
                  file=sys.stderr)
            sys.exit(2)
    sys.exit(0)

if __name__ == "__main__":
    main()
```

**Common pitfalls:**
- Treating the loop as optional overhead for "small" changes — the size of a diff is a poor predictor of its blast radius (a one-line change to a feature flag default can be far more dangerous than a 500-line additive refactor).
- Collapsing stages 2 and 3 (approving a plan that is really just "trust me, I'll figure it out as I go") — this removes the cheap-to-reverse checkpoint precisely where it's most valuable.

---

### 2.2 Plan mode

**What it is.** A Claude Code mode in which Claude can **research and propose an approach but cannot edit files or execute changes** until the plan is explicitly approved by the developer. Invoked interactively (e.g., pressing the plan-mode toggle key, or `claude --permission-mode plan`), it constrains the agent's tool access to read-only operations (`Read`, `Grep`, `Glob`, read-only `Bash` commands) for the duration of the planning phase.

**Why it matters.** A wrong approach is cheap to fix while it exists only as a plan on screen. The same wrong approach is expensive to fix once it has become committed code, with tests already written against the wrong design, other developers already building on top of it, or a deploy already shipped. This is the software-engineering equivalent of Domain 1's "discovery before design" discipline, collapsed into a single interactive session rather than spread across a multi-week solution-design phase.

**CLI invocation:**

```bash
# Start a session directly in plan mode
claude --permission-mode plan

# Or toggle into plan mode mid-session with the keyboard shortcut (Shift+Tab in the default TUI),
# then toggle back out once the plan is approved.
```

**Claude Agent SDK equivalent (Python) — constraining tool access to simulate plan mode programmatically:**

```python
from claude_agent_sdk import ClaudeAgentOptions, query

# Plan phase: read-only tools only
plan_options = ClaudeAgentOptions(
    allowed_tools=["Read", "Grep", "Glob"],
    system_prompt="You are in planning mode. Propose an approach; do not attempt any writes.",
)

async def get_plan(task: str) -> str:
    plan_text = ""
    async for message in query(prompt=task, options=plan_options):
        if message.type == "text":
            plan_text += message.text
    return plan_text

# Execution phase: only after a human approves the returned plan_text,
# a *new* session is started with write tools enabled.
execute_options = ClaudeAgentOptions(
    allowed_tools=["Read", "Grep", "Glob", "Write", "Edit", "Bash"],
)
```

**Common pitfalls:**
- Approving a plan without reading it closely because "it's just the planning step" — the entire value of plan mode depends on genuine human review at this checkpoint, not rubber-stamping it to get to the "real" work faster.
- Using plan mode for trivial one-line fixes where the ceremony adds friction without adding safety — plan mode's value scales with the size and irreversibility of the change, not with its presence in the workflow unconditionally.

---

### 2.3 Closing the loop with a runnable check

**What it is.** Giving Claude a runnable, objective check — a failing test that should pass after the fix, a build command, a lint command, or a script exercising real behavior — so that it can **self-correct before a human ever looks at the output.**

**Why it matters.** Without a runnable check, "is this done?" is a subjective judgment call made by the same system that produced the change — a weak evaluation loop. A runnable check converts "I believe this is fixed" into "this is verifiably fixed," which is the same evidentiary standard Domain 4's evaluation discipline demands of any claimed improvement (an eval score, not a vibe).

**The three-tier verification-gate escalation** (mirrors Domain 5's defense-in-depth stack directly):

| Tier | Mechanism | Nature |
|---|---|---|
| 1 | **In-prompt check** — the model checking its own work (re-reading the diff, running the test itself, reasoning about edge cases) | Probabilistic |
| 2 | **Deterministic hooks** — e.g., a `PostToolUse` hook that runs the test suite automatically after every `Edit` and blocks further action if it fails | Deterministic |
| 3 | **Adversarial reviewer** — a second, fresh-context session (or subagent) whose only job is to actively hunt for flaws in the first session's work; it has no prior reasoning of its own to defend and no incentive to rubber-stamp | Probabilistic, but *decorrelated* from the first session's blind spots |

**`PostToolUse` hook example — auto-run tests after every edit (Tier 2):**

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          { "type": "command", "command": "npm test -- --bail --silent || exit 2" }
        ]
      }
    ]
  }
}
```

**Adversarial-reviewer subagent pattern (TypeScript, Claude Agent SDK) — Tier 3:**

```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

async function adversarialReview(diff: string): Promise<string> {
  let review = "";
  const reviewPrompt = `
You are an adversarial code reviewer with no context on why this change was made.
Your only goal is to find flaws: correctness bugs, missed edge cases, security issues,
and any test that this diff would technically pass but shouldn't. Be skeptical by default.

DIFF:
${diff}
`;
  for await (const message of query({
    prompt: reviewPrompt,
    options: { allowedTools: ["Read", "Grep"] }, // read-only: it reviews, it doesn't fix
  })) {
    if (message.type === "text") review += message.text;
  }
  return review;
}
```

**Common pitfalls:**
- Stopping at Tier 1 (the model grading its own homework) for anything high-stakes — this is the weakest tier precisely because it shares all the blind spots of the session that produced the change.
- Writing a runnable check that's too loose (e.g., "the build doesn't error") to actually pin the intended behavior — a weak check gives false confidence, which is worse than no check because it's mistaken for verification.

---

### 2.4 Capturing workflows as the right asset for the job

**What it is.** A decision framework for which mechanism should hold a given piece of team knowledge or policy:

| Asset | Use for | Enforcement nature |
|---|---|---|
| **Project context** (`CLAUDE.md`) | A fact that always applies (e.g., "we use pnpm, not npm"; "the API base URL is X") | Persuasive |
| **A Skill** | A multi-step procedure the team repeats (Domain 2 §4.6) — e.g., "how we cut a release" | Persuasive (guides execution) but structured/reusable |
| **A hook** | A rule that has to hold, no exceptions (e.g., "never commit directly to `main`") | Deterministic |
| **A subagent** | Isolated, scoped work that should run in fresh context, separate from the main session's history (Domain 1 §3) | Structural isolation, not a policy mechanism per se |

**Why it matters.** This is the same probabilistic/deterministic split running through Domains 2, 3, and 5, now framed as a concrete asset-selection decision an architect makes when designing team tooling: **if a rule genuinely cannot be allowed to slip, it does not belong in `CLAUDE.md` alone — it needs a hook or managed policy backing it.**

**Example Skill for a repeatable procedure (Domain 2 §4.6's Skills mechanism):**

```markdown
---
name: cut-release
description: Use when the user asks to cut a new release or prepare a release branch.
---
# Cutting a Release

1. Confirm all CI checks are green on `main`.
2. Run `npm version <patch|minor|major>` per the user's instruction.
3. Generate release notes from merged PR titles since the last tag (`git log <last-tag>..HEAD --oneline`).
4. Open a PR from the version-bump commit into `release/*`.
5. Do NOT push directly to `release/*` or `main` — a human always merges the release PR.
```

**Common pitfalls:**
- Writing a Skill for something that is actually a hard rule ("never do X") — a Skill is still advisory; wrap the non-negotiable part in a hook instead, and use the Skill only for the sequencing/procedure around it.
- Creating a subagent as a workaround for context bloat without actually scoping its tool access down — a subagent's isolation value comes from *both* fresh context *and* a narrowed toolset; giving it the full toolset defeats half the point (Domain 1 §3.2).

---

### 2.5 Scaling horizontally

**What it is.** Three patterns for running more agent work concurrently without collisions or unbounded blast radius:

1. **Parallel sessions in separate git worktrees.** A git worktree is a second working directory checked out from the same repository, sharing the same `.git` history but with an independent working tree — so two Claude Code sessions can edit different branches simultaneously without file-system contention.

```bash
git worktree add ../myrepo-feature-a feature-a
git worktree add ../myrepo-feature-b feature-b
# Run one Claude Code session in each worktree directory, fully independent.
```

2. **Headless runs as a CI step.** Claude Code can run non-interactively (`claude -p "<prompt>" --output-format json`) with structured output suitable for machine parsing, and with a **scoped token** (least-privilege credential limited to exactly what that CI job needs) — applying least privilege to the CI pipeline itself (Domain 3 §1.2).

```bash
# Headless invocation in a CI pipeline, structured output for downstream parsing
claude -p "Update the changelog for this release based on merged PR titles" \
  --output-format json \
  --allowedTools "Read,Edit(CHANGELOG.md)" \
  > changelog_update_result.json
```

```typescript
// Headless run via the Claude Agent SDK, embedded directly in a CI/CD TypeScript pipeline script
import { query } from "@anthropic-ai/claude-agent-sdk";

async function runHeadlessCiTask() {
  const results: string[] = [];
  for await (const message of query({
    prompt: "Update CHANGELOG.md for this release from merged PR titles since the last tag.",
    options: {
      allowedTools: ["Read", "Edit"],
      permissionMode: "acceptEdits", // no interactive prompt in CI — scope narrowly instead
    },
  })) {
    if (message.type === "text") results.push(message.text);
  }
  return results.join("\n");
}
```

3. **Fan-out to subagents for large changes.** Rather than one session touching every file in a large refactor, spawn multiple subagents, each given only the tools and files relevant to its slice, each in fresh isolated context — containing both **cost** (smaller context windows per subagent) and **blast radius** (a mistake in one subagent's slice doesn't corrupt the others' reasoning) — directly reusing Domain 1 §3.2's context-isolation principle, here applied to a refactor rather than a research task.

**Common pitfalls:**
- Running parallel sessions against the *same* working directory instead of separate worktrees — this causes race conditions on the file system that have nothing to do with the model's reasoning quality and everything to do with basic concurrency hygiene.
- Giving a headless CI token broader scope than the specific job needs "to avoid permission errors later" — this defeats the least-privilege purpose of scoping the token in the first place.

---

## Section 3: Operational Support: Debugging and Incident Resolution

### 3.1 Frame every bug in three parts before touching code

**What it is.** A mandatory framing discipline, applied *before* any code is touched:

1. **The observable symptom** — its trigger and expected behavior, stated as **fact**, not as a theory about the underlying cause (e.g., "the `/checkout` endpoint returns HTTP 500 for orders over $10,000, since 14:02 UTC" — not "I think there's a rounding bug in the pricing calculator").
2. **The likely location** — including recent changes and what's already been ruled out, so Claude doesn't waste a research pass re-discovering what's already known.
3. **A definition of "fixed"** — concrete enough to be recognized unambiguously when it happens (e.g., "the failing test passes and the endpoint returns 200 for a $15,000 order in staging").

**Then reproduce with a failing test.** This pins the symptom precisely, gives Claude a tight change-run-check loop (§2.3), and the test's eventual pass is **proof, not opinion** — the same evidentiary standard Domain 4's eval discipline demands of any claimed fix.

**Common pitfalls:**
- Starting with a theory ("I think it's a race condition") instead of an observed fact — this anchors the investigation toward confirming a guess rather than following the evidence.
- Skipping the failing-test step and going straight to a fix — without the pinning test, "it seems fixed" is not verifiable and the fix can silently fail to address the actual root cause.

---

### 3.2 Symptom fix vs. root-cause fix

| | What it does | What survives |
|---|---|---|
| **Symptom fix** | Catches the error, adds a retry, caps the output, suppresses the visible failure | The cause — it resurfaces through another path later |
| **Root-cause fix** | Corrects *why* it failed | Nothing survives to resurface |

**Why it matters.** A root-cause fix is slower because it requires actually understanding the failure mechanism rather than suppressing its visible symptom. This directly parallels Domain 4 §2.7's "never swap the model first" discipline: a symptom-level patch (swap the model, add a retry, widen a timeout) can look like progress on a dashboard while the actual defect — a bad prompt, a stale retrieval index, a genuine model capability mismatch — sits untouched underneath, waiting to resurface.

**Worked contrast:**

```python
# SYMPTOM FIX — catches the error, doesn't understand it
def get_price(order):
    try:
        return calculate_price(order)
    except OverflowError:
        return order.subtotal  # silently wrong for large orders, just doesn't crash

# ROOT-CAUSE FIX — the actual defect was an int overflow in a currency-cents calculation
def get_price(order):
    return calculate_price_decimal(order)  # switched internal representation to Decimal
```

**Common pitfalls:**
- Declaring victory when the error stops appearing in logs, without confirming *why* it stopped appearing — a retry can mask an intermittent root cause for weeks before it resurfaces under load.
- Treating "swap the model" as a first-line fix for a quality regression, when the actual cause is frequently a prompt, context, or retrieval defect that a bigger/different model will not reliably fix (Domain 4 §2.7).

---

### 3.3 Feed Claude raw signal, not paraphrase

**What it is.** Three concrete practices for incident investigation:

- **Pipe actual log output** directly rather than summarizing it in your own words first — paraphrasing loses exactly the detail (an exact error code, a specific field value, a timestamp precision) that often turns out to matter.
- **Reference the exact files** that matter (`@path/to/file.py:142`) rather than describing them prose-style.
- **Let Claude run CLI tools directly** against external services (`kubectl logs`, `aws cloudwatch get-log-events`, `curl` against an internal API) rather than the human relaying results manually.

**Why the CLI path is the most context-efficient option:** it pulls *only* the data actually queried (not an entire dashboard export), keeps the command and its output auditable together in the session's context (so the reasoning trail is reconstructable later), and reuses tools the team already trusts and has already vetted — rather than building a bespoke, unvetted integration for a one-off debugging session.

**Example — piping raw logs directly into a session (bash):**

```bash
# Bad: human paraphrases ("there's some kind of timeout error in the payments service")
# Good: pipe the actual signal
kubectl logs deployment/payments-service --since=1h | claude -p "What's causing these errors?"
```

**Common pitfalls:**
- Summarizing a stack trace instead of pasting it verbatim — the summary is where the diagnostic detail gets lost.
- Building a custom one-off log-fetching script mid-incident instead of using the CLI tool the team already has vetted and trusts — this adds risk and delay exactly when speed matters most.

---

### 3.4 Live access beats pasted logs in an incident

**What it is.** Connecting monitoring and error-tracking systems (Datadog, Sentry, CloudWatch, PagerDuty) to Claude via **MCP**, so Claude queries live data through one standard interface (Domain 3 §3.2) rather than working only from whatever the human happened to paste into the session.

**Why it matters.**
- **Pasted logs are a frozen snapshot** — stale the moment the incident evolves, and reflecting only what the human *chose* to include, which is often exactly the wrong slice, because the person pasting doesn't yet know what turns out to matter.
- **Direct, live access lets diagnosis follow the trail as it actually moves** — if a new error pattern emerges three minutes into the investigation, a live-connected session can pull it immediately; a pasted-log session cannot, because the human has to notice it, copy it, and paste it in, losing time that matters enormously in an active incident.

**Example MCP configuration for a monitoring connector (project-scoped, per §1.4):**

```json
{
  "mcpServers": {
    "monitoring": {
      "type": "sse",
      "url": "https://mcp.internal.example.com/datadog/sse",
      "env": { "DATADOG_API_KEY_HELPER": "vault-cli read secret/datadog" }
    }
  }
}
```

**Common pitfalls:**
- Treating a one-time log paste at the start of an incident as sufficient context for the whole investigation — the picture is still moving.
- Connecting monitoring MCP servers with broader query/write scope than read-only incident investigation requires (violates least privilege from Domain 3 §1.2 even in an incident-response context).

---

### 3.5 Three session-hygiene moves for clean triage

1. **Clear context after a failed fix attempt.** Every dead end biases the next attempt toward repeating the same wrong assumption (the model, like a human, anchors on its own prior reasoning). The saved history isn't actually lost — just no longer cluttering the active reasoning window. This is Domain 2 §4.1's context-rot principle, applied specifically to incident response.
2. **Send subagents to investigate in their own context**, returning only a distilled summary to the main session — contains the blast radius of a bad investigative path (Domain 1 §3.1).
3. **Checkpoint a known-good state before any risky change**, so a failed fix rolls straight back instead of leaving the system in an unknown intermediate state during an active incident.

**Implementation — checkpointing before a risky live change:**

```bash
# Before attempting a risky live remediation
git tag pre-incident-fix-$(date +%s)
kubectl rollout status deployment/payments-service > /tmp/pre-fix-state.txt

# If the fix attempt fails or worsens the incident:
git reset --hard pre-incident-fix-<timestamp>
kubectl rollout undo deployment/payments-service
```

**Common pitfalls:**
- Continuing to iterate in the same context window after a fix attempt clearly failed, rather than clearing and restarting with the accumulated *understanding* (not the accumulated dead-end reasoning) carried forward manually.
- Making a live remediation attempt without first establishing a rollback point — turning a bad fix into a second incident on top of the first.

---

## Worked Example: Operational Runbook Excerpt for the Telecom Triage System

Continuing the running example from Domains 1–6, this is what actually lives in the on-call runbook (Domain 6 §2.6):

```markdown
## Incident: Routing accuracy dropped below SLO (92% → 81%)

1. SYMPTOM (fact, not theory)
   - Dashboard: routing-accuracy metric, 4-hour rolling window, dropped
     from 92% to 81% starting ~14:20 UTC. No deploy in the prior 6 hours.

2. LOCATION (ruled out so far)
   - Not a regression: no prompt/model change in the window (checked
     ADR log and prompt version history — Domain 6, §2.4).
   - Confirmed: coincides with a vendor firmware release (new device
     category appearing in tickets) → hypothesis: DRIFT, not regression
     (Domain 6, §3.4).

3. DEFINITION OF FIXED
   - Routing accuracy back above 92% on the next 4-hour window, AND
     the new firmware category added to the eval seed set (Domain 4,
     §1.3) so this drift can't silently reoccur unnoticed.

4. ACTION (root-cause, not symptom)
   - Retrieval corpus updated with the new firmware's runbook pages
     (Domain 3, §2). Prompt and model left untouched — this was a
     retrieval-origin failure, confirmed via recall@20 on the new
     corpus (Domain 4, §2.7), not a model-origin one.

5. VERIFICATION
   - Failing eval case added, then passes post-fix (Domain 4, §2.5).
   - Champion-challenger: new retrieval index promoted only after
     clearing the full frozen eval suite, not just the new case.
```

**Implementation note connecting this runbook to the mechanisms above:** step 4's "prompt and model left untouched" is a root-cause fix (§3.2) reached by piping live retrieval-eval data (§3.3–3.4) rather than assuming the fix was a model swap; step 5's champion-challenger promotion gate is exactly the kind of non-negotiable rule (§1.3/§2.4) that should be backed by a deterministic CI gate — e.g., a hook or CI check that refuses to promote a new retrieval index unless the full frozen eval suite (Domain 4) passes — not merely documented as a step someone is expected to remember to run.

---

## Architecture & Implementation Best Practices

1. **Treat `.claude/` as part of the repo's architecture, not tooling trivia.** Review changes to `.claude/settings.json`, `.mcp.json`, and `CLAUDE.md` in PRs with the same rigor as application code — they determine what an autonomous agent is permitted to do in this codebase.
2. **Push non-negotiable rules to the deterministic layer as a default posture**, not as an afterthought once something goes wrong. When designing a new team workflow, ask "what happens if the model ignores this instruction?" for every rule — if the answer is unacceptable, it needs a hook or managed policy, not just `CLAUDE.md` prose.
3. **Default to `allowManagedPermissionRulesOnly: true` in regulated environments** and build the org's allow/deny list centrally, rather than trusting per-project or per-developer configuration to converge on the same safe posture independently.
4. **Scope MCP servers deliberately** — project scope for team-shared tools with per-developer approval as an intentional trust checkpoint, user scope for genuinely personal tools, local scope for genuinely one-off/experimental connectors. Never default to project scope purely for convenience.
5. **Build the five-stage loop into CI/CD gating, not just developer habit** — a merge-blocking check that a PR resulted from an approved plan and passed the runbook checks operationalizes Domain 5's HITL principle rather than relying on discipline alone.
6. **Instrument the verification-gate tiers explicitly.** Log which tier (in-prompt, hook, adversarial reviewer) caught each defect during development — this data feeds directly into Domain 4's evaluation practice and reveals whether your Tier 1 self-checks are actually catching anything, or purely theater.
7. **Wire monitoring/logging MCP connectors before an incident occurs**, not during one. Standing up live access mid-incident wastes exactly the minutes that matter most (§3.4).
8. **Version and review runbooks like code** — the worked example's runbook should live in version control, be updated as part of the incident postmortem, and be referenced by hooks/CI where the "definition of fixed" can be made mechanically checkable (e.g., an eval seed-set addition that CI can verify is present).

---

## Common Pitfalls and Limitations

- **Conflating persuasive and deterministic controls** is the single most common failure mode across this entire domain — `CLAUDE.md`/prompt-level instructions are not a substitute for deny rules, hooks, or managed policy for anything safety- or security-critical.
- **Secrets in gitignored files** — a pervasive but incorrect sense of safety; `.gitignore` is a convention, not a security boundary.
- **Assuming a committed MCP server definition equals org-wide trust** — it does not; per-developer first-use approval (workspace trust) still gates execution unless explicitly overridden via managed settings.
- **Skipping plan mode or diff review "just this once" under deadline pressure** — the value of a human gate is entirely a function of it being exercised consistently; an occasionally-skipped gate provides a false sense of security while providing none of the actual protection.
- **Symptom fixes celebrated as resolutions** — a metric returning to normal is not proof of a root-cause fix; only a reproducing test passing post-fix is that proof.
- **Debugging from paraphrased or stale pasted context** instead of live, raw signal — this domain's guidance on this point is unambiguous: paraphrase loses exactly the detail that matters, and pasted snapshots go stale mid-incident.
- **Not clearing context after a dead end** — compounding bias from a failed hypothesis into subsequent reasoning is a subtle but real failure mode that costs real incident-response time.
- **Under-scoping headless CI tokens is rare; over-scoping is common** — "give it broad access so it doesn't fail on permissions" is a frequent but incorrect shortcut that violates least privilege exactly where automation makes an over-broad credential most dangerous (no human is watching in real time).

---

## Official Documentation References

1. Anthropic Engineering, *"Claude Code: Best practices for agentic coding,"* Apr 18, 2025 — https://www.anthropic.com/engineering/claude-code-best-practices
2. Anthropic Docs, *"Claude Code settings"* — https://docs.claude.com/en/docs/claude-code/settings
3. Anthropic Docs, *"Hooks reference"* — https://docs.claude.com/en/docs/claude-code/hooks
4. Anthropic Docs, *"Manage permissions"* — https://docs.claude.com/en/docs/claude-code/iam
5. Anthropic Docs, *"Headless mode / CI integration"* — https://docs.claude.com/en/docs/claude-code/headless
6. Anthropic Docs, *"Model Context Protocol (MCP) in Claude Code"* — https://docs.claude.com/en/docs/claude-code/mcp
7. Anthropic Docs, *"Claude Agent SDK overview"* — https://docs.claude.com/en/docs/agent-sdk/overview
8. Anthropic Docs, *"Subagents"* — https://docs.claude.com/en/docs/claude-code/sub-agents
9. Anthropic Docs, *"Skills"* — https://docs.claude.com/en/docs/agents/skills
10. Model Context Protocol specification — https://modelcontextprotocol.io

> **Version note:** Claude Code's settings file locations, hook event names, and CLI flags are under active development and have changed across releases. Verify exact syntax (e.g., `--permission-mode`, hook event names, managed settings file paths per OS) against the current `docs.claude.com` pages before implementation, as this guide reflects the documented behavior as of the source domain's writing and may drift from the latest release.

---

## Domain Coverage / Traceability Matrix

| Source domain item | Implementation guide section(s) | Coverage |
|---|---|---|
| 1.1 Configure once at the project level (committed vs. local table) | §1.1 | ✅ Full |
| 1.1 Secrets exception (vault, key helper, never in repo) | §1.1 | ✅ Full |
| 1.2 Settings resolution order (5-tier priority) | §1.2 | ✅ Full |
| 1.2 `allowManagedPermissionRulesOnly` | §1.2 | ✅ Full |
| 1.3 Allow rules vs. deny rules | §1.3 | ✅ Full |
| 1.3 Hooks as deterministic enforcement (`PreToolUse`, etc.) | §1.3 | ✅ Full, with runnable hook code (Python + TS SDK) |
| 1.3 CLAUDE.md/permissions as persuasive, not enforced | §1.3 | ✅ Full |
| 1.4 MCP scopes: local/project/user | §1.4 | ✅ Full |
| 1.4 Per-developer approval / workspace trust | §1.4 | ✅ Full |
| 2.1 Five-stage loop (research → plan → execute → review → ship) | §2.1 | ✅ Full |
| 2.1 Role split: Claude does work, developer gates | §2.1 | ✅ Full |
| 2.2 Plan mode (read-only, cannot edit until approved) | §2.2 | ✅ Full, with CLI + SDK examples |
| 2.3 Runnable check / self-correction loop | §2.3 | ✅ Full |
| 2.3 Three-tier verification escalation (in-prompt, hooks, adversarial reviewer) | §2.3 | ✅ Full, with hook + adversarial-reviewer code |
| 2.4 Asset table: context / Skill / hook / subagent | §2.4 | ✅ Full, with example Skill file |
| 2.5 Parallel sessions via git worktrees | §2.5 | ✅ Full, with commands |
| 2.5 Headless runs as CI step + scoped token | §2.5 | ✅ Full, with bash + SDK examples |
| 2.5 Fan-out to subagents for large changes | §2.5 | ✅ Full |
| 3.1 Three-part bug framing (symptom/location/definition of fixed) | §3.1 | ✅ Full |
| 3.1 Reproduce with a failing test | §3.1 | ✅ Full |
| 3.2 Symptom fix vs. root-cause fix (table + parallel to Domain 4 §2.7) | §3.2 | ✅ Full, with code contrast example |
| 3.3 Pipe raw log output, reference exact files, run CLI directly | §3.3 | ✅ Full |
| 3.3 CLI path as most context-efficient option | §3.3 | ✅ Full |
| 3.4 Live MCP access to monitoring/error-tracking | §3.4 | ✅ Full, with MCP config example |
| 3.4 Pasted logs as stale snapshot vs. live diagnosis | §3.4 | ✅ Full |
| 3.5 Clear context after failed fix attempt | §3.5 | ✅ Full |
| 3.5 Subagent investigation returning distilled summary | §3.5 | ✅ Full |
| 3.5 Checkpoint before risky change | §3.5 | ✅ Full, with rollback commands |
| Worked Example: Operational Runbook (full incident walkthrough) | Worked Example section | ✅ Full, reproduced and annotated with implementation linkage |
| Key Takeaways (source domain summary bullets) | Distributed across Best Practices + Pitfalls | ✅ Full — every takeaway bullet maps to at least one best-practice or pitfall item above |
| References 1–6 (source domain) | Official Documentation References | ✅ Full, retained and extended with SDK/Skills/MCP-spec references |

**Coverage verification statement:** Every numbered subsection, table, code concept, and key-takeaway bullet in `domain-7-developer-productivity-operational-enablement.md` is represented above with expanded technical detail, at least one implementation pattern, and — where the source item names or implies a concrete mechanism (hooks, settings resolution, MCP scopes, plan mode, worktrees, headless mode) — a runnable Python and/or TypeScript example. No source item was merged, omitted, or skipped.
