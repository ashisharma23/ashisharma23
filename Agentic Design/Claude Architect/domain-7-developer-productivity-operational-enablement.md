# Domain 7: Developer Productivity & Operational Enablement
**Exam weight: 7% | Claude Certified Architect — Professional**

> The smallest domain by exam weight, but the one that decides whether the architecture from Domains 1–6 actually gets built and operated the way it was designed — or quietly drifts as every developer configures their own environment differently.

---

## 1. Team Enablement

### 1.1 Configure once at the project level

Roll Claude Code out to a team as **one shared, version-controlled environment**, not N individually configured laptops:

| Committed at project level | Stays local (gitignored) |
|---|---|
| Project settings | Personal overrides |
| Project memory (`CLAUDE.md`) | — |
| MCP server definitions | — |
| Permission rules | — |

**Every developer inherits one reproducible environment.** This is the same "informative yet tight, curated deliberately" discipline from Domain 2's context management, applied to team tooling instead of a single prompt.

**Secrets are the one deliberate exception to "commit everything."** API keys live in the OS credential store or a vault, with a key helper fetching short-lived, rotating tokens at runtime — **never in the repo**, even in a gitignored file, because gitignore is a convention, not a security boundary, and a single misconfigured `.gitignore` entry turns a secret into a permanent, unrevocable leak in git history.

### 1.2 Settings resolve highest-wins

```
1. Managed settings (org-pushed)         ← cannot be overridden
2. Command-line flags
3. Personal local override file
4. Committed project settings
5. Home-directory user settings          ← lowest priority
```

The managed setting **`allowManagedPermissionRulesOnly`** is the enterprise lockdown switch: when set, only organization rules apply, and every local and project rule is ignored entirely — the correct control for regulated environments where individual developer discretion over tool permissions is not acceptable (see Domain 5, §1.1's access-control layer).

### 1.3 Deny rules, hooks, and managed policy: the only deterministic layer

**Allow rules pre-approve trusted actions so they run without interactive prompts. Deny rules hard-block dangerous actions regardless of what the model decides** — this mirrors Domain 3, §1.2's rule that deny always wins over allow.

**`CLAUDE.md` and permission rules load as context, so they *persuade* — the model can still deviate from them.** This is the critical distinction the exam and real incident postmortems both test: **the three genuinely deterministic enforcement mechanisms are deny rules, hooks on lifecycle events (e.g., `PreToolUse`), and managed policy.** Any rule that *must* hold — not just "should usually hold" — belongs in one of those three, not in memory or a system prompt where it's advisory at best.

### 1.4 MCP server scopes

| Scope | Reach |
|---|---|
| **Local** | Private to you |
| **Project** | Ships in a committed file, whole team shares the definition |
| **User** | Private, but follows you across all your projects |

**A shared project server is still approved per developer on first use** — committing the server definition doesn't grant blanket trust to everyone who clones the repo. **Workspace trust gates the project**, so a committed MCP configuration never executes silently the moment someone checks out the branch.

---

## 2. Improving Developer Workflows with AI-Assisted Tooling

### 2.1 The five-stage loop

Run this loop every time, without skipping stages under deadline pressure:

```
1. RESEARCH  →  2. AGREE A PLAN  →  3. EXECUTE (focused, reviewable steps)
        →  4. REVIEW THE DIFF (human gate)  →  5. SHIP (after verification)
```

**Claude does the work; the developer frames the problem, approves the plan, reviews each diff, and owns the ship-or-rollback call.** This is a direct application of Domain 5's human-in-the-loop principle (§3.1–3.4) to the day-to-day engineering workflow itself — the diff review *is* the HITL gate, and skipping it for a "quick" change is exactly the rubber-stamping failure mode Domain 5 warns against.

### 2.2 Plan mode

**Plan mode is read-only:** Claude researches and proposes an approach but **cannot edit files or run changes** until the plan is explicitly approved. A wrong approach is cheap to fix while it's still a plan on screen, and painfully expensive once it's already committed code with tests written against the wrong design. This is the software-engineering equivalent of Domain 1's discipline — discovery before design — collapsed into a single interactive session.

### 2.3 Closing the loop with a runnable check

Give Claude a **runnable check** — a failing test, a build or lint command, or a script exercising real behavior — so it can **self-correct before a human even looks at the output.** Verification gates escalate in the same order as Domain 5's defense-in-depth stack:

1. **An in-prompt check** (probabilistic — the model checking its own work)
2. **Deterministic hooks** that can block an action regardless of what the model decides (Domain 5's deterministic layer, applied here)
3. **An adversarial reviewer** — a second session with fresh context that actively hunts for flaws, because it has no prior reasoning of its own to defend and no incentive to rubber-stamp the first session's conclusions

### 2.4 Capturing workflows as the right asset for the job

| Asset | Use for |
|---|---|
| **Project context** | A fact that always applies |
| **A Skill** | A multi-step procedure (Domain 2, §4.6) |
| **A hook** | A rule that has to hold, no exceptions |
| **A subagent** | Isolated, scoped work in fresh context (Domain 1, §3) |

**Context and Skills persuade; hooks and permissions are deterministic enforcement** — the same probabilistic/deterministic split that runs through Domains 2, 3, and 5 shows up here as a practical asset-selection decision: if a rule genuinely cannot be allowed to slip, it does not belong in `CLAUDE.md` alone.

### 2.5 Scaling horizontally

- **Parallel sessions**, each in its own **git worktree** (a separate checkout of the same repo), so concurrent edits never collide
- **Headless runs** make Claude a CI step with structured output and a **scoped token** — least privilege applied to the CI pipeline itself (Domain 3, §1.2)
- **Fan out large changes to subagents** given only the tools and files their specific slice needs, in fresh isolated context, to contain both cost and blast radius (Domain 1, §3.2's context-isolation principle, applied to a refactor instead of a research task)

---

## 3. Operational Support: Debugging and Incident Resolution

### 3.1 Frame every bug in three parts before touching code

1. **The observable symptom** — its trigger and expected behavior, stated as fact, **not your theory** about the cause
2. **The likely location** — including recent changes and what you've already ruled out
3. **A definition of "fixed"** — stated concretely enough that you'll recognize it when it happens

**Then reproduce with a failing test.** It pins the symptom precisely, gives Claude a tight change-run-check loop (§2.3), and its eventual pass is **proof, not opinion** — the same evidentiary standard Domain 4's eval discipline demands of any claimed fix.

### 3.2 Symptom fix vs. root-cause fix

| | What it does | What survives |
|---|---|---|
| **Symptom fix** | Catches the error, adds a retry, caps the output | The cause — it resurfaces through another path |
| **Root-cause fix** | Corrects *why* it failed | Nothing survives to resurface |

A root-cause fix is slower, because it requires actually understanding the failure rather than suppressing its visible symptom — but **it is the only fix that holds.** This directly parallels Domain 4, §2.7's "never swap the model first" discipline: a symptom-level patch (swap the model, add a retry) can look like progress on a dashboard while the actual defect — a bad prompt, a stale retrieval index, a genuine model mismatch — sits untouched underneath.

### 3.3 Feed Claude raw signal, not paraphrase

- **Pipe actual log output** rather than summarizing it in your own words — paraphrasing loses exactly the detail that often matters
- **Reference the exact files** that matter, not a description of them
- **Let it run CLI tools** against external services directly

**The CLI path is the most context-efficient option**: it pulls only the data actually queried, keeps the command and its output auditable in context, and reuses tools the team already trusts and has already vetted — rather than building a bespoke integration for a one-off debugging session.

### 3.4 Live access beats pasted logs in an incident

**Connect monitoring and error-tracking systems through MCP** so Claude queries live data through one standard interface (Domain 3, §3.2) rather than working from whatever the human happened to paste in. **Pasted logs are a frozen snapshot** that goes stale mid-incident and reflects only what the human chose to include — often exactly the wrong slice, because the person pasting doesn't yet know what turns out to matter. **Direct access lets diagnosis follow the trail as it actually moves**, which matters enormously in an incident where the picture is still changing minute to minute.

### 3.5 Three session-hygiene moves for clean triage

1. **Clear context after a failed fix attempt** — every dead end biases the next attempt toward repeating the same wrong assumption, and the saved history isn't actually lost, just no longer cluttering the active reasoning window (Domain 2, §4.1's context-rot principle, applied to incident response specifically)
2. **Send subagents to investigate in their own context** and return only a distilled summary — contains the blast radius of a bad investigative path (Domain 1, §3.1)
3. **Checkpoint a known-good state before any risky change**, so a failed fix rolls straight back instead of leaving the system in an unknown intermediate state during an active incident

---

## Worked Example: Operational Runbook Excerpt for the Telecom Triage System

Continuing the running example from Domains 1–6, this is what actually lives in the on-call runbook (Domain 6, §2.6):

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

---

## Key Takeaways

- Configure once at the project level; secrets live in a vault, never in the repo, even gitignored. Settings resolve highest-wins, with managed org policy at the top.
- `CLAUDE.md` and permission rules persuade; only deny rules, hooks, and managed policy are genuinely deterministic enforcement.
- MCP servers are scoped local/project/user, and a shared project server still requires per-developer approval on first use.
- The five-stage loop — research, plan, execute, review the diff, ship — puts a human gate exactly where Domain 5 says one belongs: before an irreversible action (the merge).
- Plan mode is read-only by design: a wrong plan is cheap; a wrong committed diff is not.
- Root-cause fixes are slower but permanent; symptom fixes look like progress and let the real defect resurface elsewhere.
- Live MCP access to monitoring/logs beats pasted snapshots in an active incident, because the picture is still moving.
- Clear context after a dead end, delegate risky exploration to subagents, and always checkpoint before a risky change.

---

## References

1. Anthropic Engineering, *"Claude Code: Best practices for agentic coding,"* Apr 18, 2025 — https://www.anthropic.com/engineering/claude-code-best-practices
2. Anthropic Docs, *"Claude Code settings"* — https://docs.claude.com/en/docs/claude-code/settings
3. Anthropic Docs, *"Hooks reference"* — https://docs.claude.com/en/docs/claude-code/hooks
4. Anthropic Docs, *"Manage permissions"* — https://docs.claude.com/en/docs/claude-code/iam
5. Anthropic Docs, *"Headless mode / CI integration"* — https://docs.claude.com/en/docs/claude-code/headless
6. Anthropic Docs, *"Model Context Protocol (MCP) in Claude Code"* — https://docs.claude.com/en/docs/claude-code/mcp
