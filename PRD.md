# Zeno — Product Requirements Document (PRD)

**Source:** INTAKE.md answers A1–A10, C1–C2, D6. Nothing here is invented; every
requirement traces to a numbered intake answer.
**Status:** Draft for review · **Date:** 2026-10-23

---

## 1. What Zeno is

Zeno is a personal AI that does everything — by having **no features built in**.

Zeno is not a list of integrations. It is an agent that gains abilities from outside
its own codebase: MCP servers it connects to, and Agent Skills it installs or writes.
If a capability is registered, Zeno can use it. If it isn't, Zeno can't — and that
difference is a one-line answer, not a release cycle.

## 2. The problem

**A2** — the user triages and drafts email and messages by hand, repeatedly, and wants
that off their plate.

**A1** — the ambition is larger than the inbox. Zeno should be able to do *anything*,
given the capability exists.

**A7** — success is modest and specific: *"when I can automate some few things."*

These three are held together deliberately. A2 is the first real workflow. A1 is the
long-term shape. A7 is the three-month bar, and it is set low enough to actually hit.

## 3. Users

**Primary — Abdulkadir (you).** Single user for Phase 0–2. Your account, your data,
your machine. (A6)

**Secondary — other people, later.** Zeno will eventually act for people you work with
(A5). They will reach Zeno, but **you stay admin and gate what it may do for them**.
Not in scope until the system is proven. (A6)

## 4. Core requirements

### 4.1 Functional

| ID | Requirement | Source |
|---|---|---|
| **FR-1** | Zeno is addressable from a **terminal** as the first interface | C1 |
| **FR-2** | Zeno runs a tool-calling agent loop: model → tool → observe → repeat until done | A1, A3 |
| **FR-3** | Zeno discovers capabilities from **MCP servers** and **Agent Skills** at startup | A1, A4 |
| **FR-4** | **Nothing happens without permission.** Default deny. Deleting, modifying, everything. | A8 |
| **FR-5** | When an instruction is unclear, Zeno **asks for clarification** rather than guessing | A8 |
| **FR-6** | Every tool call is shown **live, with its reasoning and arguments** | C3 |
| **FR-7** | The user can **stop and redirect** Zeno mid-task | C5 |
| **FR-8** | Zeno **remembers everything automatically**, stored **separately per person** | E1, contradiction 1 |
| **FR-9** | Zeno can **write its own skills** and **install existing ones from the internet** | D3 |
| **FR-10** | Zeno runs **scheduled and background tasks** | D6 |
| **FR-11** | Zeno **triages email and messages and drafts replies** for approval | A2 |
| **FR-12** | Zeno runs on **free-tier models only** (or local) | A9 |

### 4.2 The permission model — FR-4 expanded

This is the most consequential requirement in the document.

**Default: deny. Every action prompts.** No silent execution. The "read actions run
freely" tiering proposed in GUIDE.md was rejected by A8.

**The one exception — self-authored skills (contradiction 2 resolution):**

> If you say "do X" and Zeno writes a skill to do X, that is authorized — the goal was
> given, and the means to reach it fall within it.

What this does **not** license: acting on goals the user did not set. The boundary is
**scope**, not approval count.

**Corollaries:**
- The user is always admin (A5)
- Other people's actions are always gated by the user (A5)
- Every executed action is logged (FR-6 + audit)

### 4.3 Non-functional

| ID | Requirement | Source |
|---|---|---|
| **NF-1** | **Reliable** — correct results | A3 |
| **NF-2** | **Fast** — low latency responses | A3 |
| **NF-3** | **Autonomous** — given a goal, it plans and runs without hand-holding | A3 |
| **NF-4** | **Cost-bounded** — free tier only; a model fallback chain; turn minimization | A9 |
| **NF-5** | **Private** — runs on the user's machine | E7 (TBD) |
| **NF-6** | **Cross-platform eventually** — Windows first, Android and iPhone later | C1 |

A3 requires all three of reliable, fast, and autonomous **equally**. On the free tier,
fast (NF-2) is the one most at risk — see TRD §10.

## 5. Scope

### In scope — Phase 0–2

- Terminal interface (C1)
- Agent loop with tool calling
- MCP client: stdio + Streamable HTTP
- Agent Skills discovery and execution (`SKILL.md`)
- Permission system, deny-all default
- Full tool-call transparency (C3)
- Interrupt and redirect (C5)
- Per-principal memory
- Skill installation from the internet (D3)
- Zeno writing its own skills (D3)
- Background and scheduled tasks (D6)
- Inbox triage and drafting (A2)

### Out of scope for now

- Voice input and output (C2 — later)
- Full-screen desktop app (C1 — later; terminal first)
- Other people as active users (A6 — later)
- Local AI as the primary model (later; free tier now)

## 6. Success criteria

**A7:** *"What will make me say Zeno is working? When I can automate some few things."*

Measurable at three months:

- [ ] **At least 3 real, reusable automations** running without manual intervention each time
- [ ] Each of those runs within the **free-tier request limits** (A9)
- [ ] Every action in those automations was **visible** and **authorized** (FR-4, FR-6)
- [ ] The user has **not** had to hand-execute a task they previously asked Zeno to do twice
- [ ] Zeno has **interrupted to ask for clarification** at least once, and the
      interruption was useful rather than annoying (FR-5)

**Explicitly not success criteria** — A1 is the ambition, A7 is the bar:

- *"Zeno can do anything"* — aspiration, not a milestone
- *"Zeno has a polished UI"* — C1 says the front end is secondary

## 7. Explicit non-goals

- **No commercial product.** Personal project.
- **No feature hardcoded.** Every capability arrives via MCP or Skill.
- **No multi-user authentication** in Phase 0–2 (A6).
- **No vector database required** at start — memory begins file-based (see BSD).

## 8. Open items carried into the build

| Ref | Item | Status |
|---|---|---|
| **A9** | One-time $10 OpenRouter top-up — raises free tier from 50 to 1,000 requests/day, permanently | User is *"open to considering"*. **Decide before Phase 0.** |
| **A10** | Willingness to maintain the project | **Unanswered** |
| **C6** | Visual style / color | **Unanswered** — purple mentioned then retracted |
| **C8** | Notifications when a background task finishes | **Unanswered** |
| **D1** | First MCP server used to prove the design | **Unanswered** |
| **D2** | Where skills are fetched from | **Unanswered** |
| **D5** | What happens when a capability is missing | **Unanswered** |
| **E3** | Can Zeno forget? | **Unanswered** |
| **E4** | Inspect and edit stored data | **Unanswered** |
| **E5** | Which accounts precisely | Partially — A6 says *"mine"* but not which |
| **E7** | Anything that must never be stored locally | **Unanswered** |

---

**Every requirement above is traceable to INTAKE.md.** If something here does not
appear there, it is an error in this document, not a new idea.
