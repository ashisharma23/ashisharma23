

# Anthropic Certification — 7-Step Roadmap
*How to prepare for and pass a Claude Certified exam (2026)*

> **Program note:** Anthropic's official credential is the **Claude Certified** program, delivered via Pearson VUE and administered through the **Claude Partner Network**. It launched in March 2026 with the Architect exam and expanded by July 2026 to four exams across three roles. Registration currently runs through Anthropic's Partner Academy — check the partner portal for the latest access and pricing details before you commit study time, since a young program like this can change quickly.

---

## The 4 Exams at a Glance

| Exam | Code | Level | Price (approx.) | Who it's for |
|---|---|---|---|---|
| Claude Certified Associate, Foundations | CCAO-F | Entry, non-coding | ~$99 | Business/product users, no-code Claude usage |
| Claude Certified Developer, Foundations | CCDV-F | Developer | ~$125 | Engineers building on the Claude API |
| Claude Certified Architect, Foundations | CCAR-F | Architect | ~$125 | Engineers/architects shipping production Claude systems |
| Claude Certified Architect, Professional | CCAR-P | Advanced architect | ~$175 | Senior architects — scenario-heavy, enterprise scale |

**Shared exam mechanics:** 120 minutes, multiple-choice/multiple-response (each question states how many answers to select), scaled score 100–1,000, cut score 720, delivered via Pearson VUE (OnVUE online proctoring or a physical test center).

**Architect (Foundations) domain weighting** — the most commonly cited blueprint:
- Agentic Architecture & Orchestration — 27%
- Claude Code Configuration & Workflows — 20%
- Prompt Engineering & Structured Output — 20%
- Tool Design & MCP Integration — 18%
- Context Management & Reliability — 15%

*(Domain weights differ for Associate/Developer — always confirm against the current official exam guide for the specific exam you're taking.)*

---

## Step 1 — Build Your Foundation with Free Training Courses
Anthropic Academy (hosted on Skilljar) is the official, free training platform. No Anthropic account required — just an email.

- **Portal:** `anthropic.skilljar.com`
- **Link (as shared):** https://lnkd.in/dVgiUCKx
- Courses span three tracks: **AI Fluency / general use**, **Product training**, and **Developer deep-dives**.
- Every course ends in a graded quiz and issues a free Anthropic completion certificate.

**Recommended developer path:**
1. Claude 101 (or "AI Capabilities and Limitations" if you want the neutral foundations first)
2. Building with the Claude API (the flagship, 8+ hour course — covers system prompts, tool use, context windows, architecture patterns)
3. Introduction to Model Context Protocol (MCP)
4. Claude Code 101 (Explore → Plan → Code → Commit workflow)
5. Introduction to Subagents
6. Cloud-specific course if relevant: Claude on Amazon Bedrock, or Claude on Google Cloud Vertex AI

*Tip: the exam draws heavily on the API and Claude Code courses — don't skip them even if you already use Claude daily.*

---

## Step 2 — Learn by Building: the Anthropic Cookbook
Reading documentation only gets you partway. The **Anthropic Cookbook** (on GitHub) is a collection of runnable, practical Claude implementation patterns — tool use, RAG, PDF/vision handling, multi-agent patterns, evaluation, and more.

- **Link (as shared):** https://lnkd.in/dQzjSTAa

- Clone it, run the notebooks locally, and modify them rather than just reading.
- Cross-reference each recipe with the corresponding docs page so you understand *why* it's built that way — that's the level of understanding the scenario-based exam questions test.

---

## Step 3 — Deep Dive: the Two Core Pillars (API + MCP)
The exams are built almost entirely on two documentation sets. Know these cold:

**Claude API — key areas to master:**
- Messages API, system prompts, and structured/JSON output
- Tool use / function calling, including multi-tool and parallel tool calls
- Prompt caching (know the TTL behavior) and its cost implications
- Batch API and when the discount pays for itself
- Model lineup, context window sizes, rate limits, and token pricing
- Extended thinking and streaming

**Model Context Protocol (MCP) — key areas to master:**
- Server/client architecture and the transport layer
- Tool schemas and error envelopes (scenario questions often test this)
- Resources vs. Tools vs. Prompts as MCP primitives
- Building and connecting an MCP server end-to-end (do this hands-on, not just in theory)

*Official docs:*
- Claude API docs: `docs.claude.com`
- MCP specification & docs: `modelcontextprotocol.io`

**Links (as shared):** [MCP Docs](https://lnkd.in/dvMVkvR8) · [API Docs](https://lnkd.in/dh5Wj6j8)

---

## Step 4 — Study the Official Exam Guide
Each of the four exams has its own **official Exam Guide** published by Anthropic (PDF), which lists:
- The exact domain weightings
- Sub-objectives under each domain (the real syllabus — more granular than the marketing blurb)
- Sample questions with answer rationale
- Format, timing, and scoring mechanics

Get the exam guide for the *specific* exam you're sitting (CCAO-F, CCDV-F, CCAR-F, or CCAR-P) from the Anthropic Partner Academy or Pearson VUE's Anthropic program page — don't study a different exam's blueprint by mistake, since the domain weights differ meaningfully between Associate/Developer/Architect.

- **Link (as shared):** https://lnkd.in/dwE7UTgJ

---

## Step 5 — Take Free Practice Questions
Use the sample questions embedded in the official exam guide first — they come with Anthropic's own answer rationale, which teaches you the reasoning pattern the real exam rewards, not just the correct letter.

Common gaps flagged by people who've taken the exam:
- Underestimating the Associate exam's responsible-use/safety content (it's heavier than people expect for a "non-technical" exam)
- Skipping Claude Code content on the Developer exam because it's a small percentage — those are still free points
- Not having hands-on MCP server-building experience, which shows up as scenario questions about tool schemas and error handling
- Not knowing prompt caching's exact TTL and cost math

**Link (as shared):** https://lnkd.in/dwE7UTgJ

---

## Step 6 — Join the Partner Network
The **Claude Partner Network** is Anthropic's free program for organizations and individuals building on Claude commercially or seriously.

- Free to join — takes a few minutes
- Gives access to Anthropic Academy training paths, sales/technical playbooks, and (for many exams) is currently the gate for exam registration/pricing tiers
- Connects you to Anthropic's technical team and partner-only resources beyond the public docs

Sign up at: `claude.com/partners`

**Link (as shared):** https://lnkd.in/d9sqXrhk

---

## Step 7 — Register & Take the Exam
1. Confirm which exam matches your role (Associate / Developer / Architect Foundations / Architect Professional).
2. Register and pay through the **Anthropic Partner Academy**; exam delivery is handled by **Pearson VUE**.
3. Choose online proctoring (OnVUE) or an in-person test center, whichever is available in your region.
4. It's closed-book, no-AI-assistance — go in having actually built something (an MCP server, a Claude Code workflow, an agent with tool use), not just read about it.
5. You'll receive a scaled score (100–1,000, cut score 720) with a percent-correct breakdown per domain — useful for a retake if needed.
6. On passing, you get a digital badge via Credly to share on LinkedIn/resume.

**Link (as shared):** https://lnkd.in/dwE7UTgJ

---

## Quick Reference — Official Links to Bookmark
| Resource | URL |
|---|---|
| Anthropic Academy (free courses) | anthropic.skilljar.com |
| Anthropic Cookbook | github.com/anthropics/anthropic-cookbook |
| Claude API docs | docs.claude.com |
| MCP docs/spec | modelcontextprotocol.io |
| Claude Partner Network | claude.com/partners |
| Pearson VUE — Anthropic exam program | pearsonvue.com/us/en/anthropic.html |
| Anthropic product news | anthropic.com/news |

*Note: the `lnkd.in` links throughout this doc are LinkedIn redirect shortlinks copied as originally shared — LinkedIn blocks automated tools from resolving them, so I could not independently verify their destinations. The official domains in the table above are confirmed. Click-test the `lnkd.in` links yourself before relying on them, and prefer the official URLs if in doubt.*

---

## Study Time Estimate
Most developers with existing Claude experience report **15–20 hours** of focused study for an Architect Foundations-level exam: roughly half spent on the API/MCP docs and Cookbook, and half on hands-on building (an MCP server + a Claude Code-driven project) plus practice questions.
