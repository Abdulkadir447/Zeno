# Zeno — Implementation Plan Document (IPD)

**Source:** INTAKE.md F1–F6, A9, A10, plus the success criteria from PRD §6.
**Status:** Draft for review · **Date:** 2026-10-23

---

## 1. Order of work

**F5: *"Several hours per week. Docs first."***

The six documents are the deliverable of this phase. Build starts only after they are
reviewed and any open items resolved. This document is the last of the six.

## 2. Prerequisites before any code

These must be decided before Phase 0 starts. Each one blocks something real.

| # | Decision | Blocks | Status |
|---|---|---|---|
| 1 | **The $10 OpenRouter top-up** — 50 → 1,000 requests/day, permanent | Everything. Determines whether the free tier is viable at all | User is *"open to considering"* |
| 2 | **Which accounts** (A6 says *"mine"* — which?) | BSD §6.1, FR-11 | **Unanswered** |
| 3 | **First MCP server** to prove the design (D1) | Phase 1 | **Unanswered** |
| 4 | **Skill source** for internet installs (D2) | Phase 1, FR-9 | **Unanswered** |
| 5 | **Anything that must never be stored locally** (E7) | BSD schema | **Unanswered** |
| 6 | **Whether Zeno may forget** (E3) | BSD §5, memory design | **Unanswered** |
| 7 | **Maintenance willingness** (A10) | Phase length, scope | **Unanswered** |

**Decision 1 is the one that most changes the plan.** At 50 requests/day, a
multi-turn agent loop is barely usable. At 1,000/day it is a real product. The cost is
a one-time $10, not a subscription, and the higher limit persists even if the balance
later drops to zero.

## 3. Phases

Time estimates assume **several hours per week** (F1). They are not commitments.

### Phase 0 — Foundation (~2–3 weeks)

**Goal:** a working agent loop with a real tool call and a real permission prompt. No
integrations, no MCP yet.

**Deliverables:**

- [ ] Monorepo layout: `core/` + `clients/`
- [ ] SQLite schema from BSD (all tables, migrations)
- [ ] Config + secrets handling (gitignored `config.json`)
- [ ] **Model abstraction layer** — `ModelProvider` interface, `OpenRouterProvider`
- [ ] Model fallback chain + provider pinning (TRD §3)
- [ ] **Agent loop** — streamed, with tool calls
- [ ] Hard caps: iterations, tool calls, tokens per task
- [ ] **Permission service** — default deny, server-side enforcement
- [ ] Audit logging to `tool_calls`
- [ ] **One hardcoded no-op capability** so the loop has something to call
- [ ] **Terminal client** (C1, FR-1)
- [ ] **Interrupt and redirect** (C5, FR-7)
- [ ] **Clarification behavior** (FR-5)
- [ ] **Full tool-call transparency** in the terminal (C3, FR-6)
- [ ] Per-principal isolation + leakage test (BSD §10)

**Exit criteria:**

- [ ] Type a message in a terminal → get a streamed reply
- [ ] A tool call happens, is displayed with reasoning and arguments, and requires
      approval
- [ ] Approve it → it executes → the result feeds back into the model
- [ ] Deny it → it does not execute, and the denial is logged
- [ ] Interrupt mid-task works, and redirect carries completed context
- [ ] Two principals can be created with zero memory leakage between them

**Why interrupt ships in Phase 0:** C5 is not deferred with voice. It is hard, it is
the difference between trust and tolerance, and retrofitting it means restructuring the
loop.

### Phase 1 — Capability system (~2–3 weeks)

**Goal:** Zeno can gain abilities from outside its own codebase.

**Deliverables:**

- [ ] **MCP client v2** — stdio and Streamable HTTP transports
- [ ] Capability Registry (BSD §6.3)
- [ ] Per-server lifecycle: connect → listTools → normalize → register → health check
- [ ] Health reporting for unavailable capabilities
- [ ] **Agent Skills loader** — scan `skills/`, parse `SKILL.md`, progressive disclosure
- [ ] `allowed-tools` enforcement from skill frontmatter
- [ ] **Skill installation** from disk and from the internet (D3, FR-9)
- [ ] **Skill validation** against the SKILL.md spec
- [ ] **Zeno writes its own skills** into a staging directory (D3, FR-9)
- [ ] **Pre-execution script audit** for self-authored skills (TRD §5)

**Exit criteria:**

- [ ] Connect a real MCP server → its tools appear in the registry
- [ ] Call one of those tools → approval prompt → execute
- [ ] Install a skill → it is discovered and registered
- [ ] Zeno generates a skill → the user sees its instructions and scripts before it runs
- [ ] A server that fails to start is reported by name with the reason

**Decision 3 and 4 block this phase.**

### Phase 2 — Background service and full-screen app (~3–4 weeks)

**Goal:** Zeno runs while the window is closed.

**Deliverables:**

- [ ] Core as a **background service** (starts on login, no window needed)
- [ ] **Tauri 2 shell** with chat pane, transparency pane, approval pane
- [ ] **Jobs table + scheduler** (BSD §8, D6, FR-10)
- [ ] Permission prompts surfaced as notifications when the app is closed (UXDS §8)
- [ ] **Inbox triage and drafting** (A2, FR-11) — the first real workflow
- [ ] Job completion notification (C8 — **TBD**)

**Exit criteria:**

- [ ] Close the window → Zeno keeps running
- [ ] *"Remind me to do X tomorrow at 9"* → the job fires unattended
- [ ] Gmail triage produces drafts the user approves
- [ ] **At least 3 real automations run without manual intervention each time** (A7)

**This phase is where PRD §6's success criteria are met.** A7 is the bar, and it is
deliberately modest.

### Phase 3 — Memory and identity (~2–3 weeks)

**Goal:** Zeno knows who you are and carries context across sessions.

**Deliverables:**

- [ ] File-based memory overlay (`~/.zeno/memory/`)
- [ ] Memory writes on every session (E1 — remember everything automatically)
- [ ] Retrieval by `importance` ranking
- [ ] **Identity profile** — preferences, how you want things written, boundaries
- [ ] Memory inspection and editing surface (E4 — **TBD** whether wanted)
- [ ] Deletion / forgetting behavior (E3 — **TBD**)

**Exit criteria:**

- [ ] Zeno recalls a preference from a session days earlier
- [ ] Two principals' memories remain provably isolated
- [ ] The user can see and edit what Zeno stored

**E2, E3 and E4 are unanswered and will shape this phase.**

### Phase 4 — Local brain and mobile (later, not scheduled)

**Goal:** Zeno works without OpenRouter, and on a phone.

**Deliverables:**

- [ ] `OllamaProvider` implementing the same `ModelProvider` interface
- [ ] Local model selection — GLM-4.7-Flash (~8 GB) is the default bar, Qwen3.6 35B-A3B
      if 24 GB is available (TRD §3)
- [ ] Benchmark local vs. free tier on the user's real tasks
- [ ] **Android client** against the same core API
- [ ] **iPhone client** against the same core API

**Exit criteria:**

- [ ] Turn off OpenRouter → Zeno still works
- [ ] A task started on the desktop continues on the phone

**Honest expectation:** local models do not match frontier models on hard tasks. Phase 4
should be measured by *"does it work offline and privately,"* not *"is it as good."*
Tauri mobile is functional but less mature than desktop; if it disappoints, the phones
become React Native or Flutter apps against the same core API and nothing else changes.

## 4. What is deliberately not built

- **Voice** (C2 — later)
- **Other people as users** (A6 — later; schema is ready, product is not)
- **Vector database** (memory starts file-based; add only if retrieval is genuinely slow)
- **A framework-based agent loop** (TRD §5 — the loop is ~200 lines and is owned)
- **Any feature hardcoded** (every capability arrives via MCP or Skill)

## 5. Risk register

| # | Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|---|
| 1 | **Free-tier request limits** (A9) — an agent loop burns several requests per turn; 50/day unfunded is a handful of tasks | High | High | Fallback chain, turn minimization, hard caps, and the **$10 decision (prereq 1)** |
| 2 | **Free model tool-calling reliability** — known failure mode: *"no endpoints found that support tool use"* | High | High | Provider pinning, error logging, automatic fallback, curated config list |
| 3 | **Self-authored skills are code execution** (D3) | Medium | High | Pre-execution script audit, live visibility, self-authored skills audited separately in the schema |
| 4 | **Memory privacy across principals** (A5 + E1) | Medium | High | Per-principal isolation enforced in schema, query and test (BSD §10) |
| 5 | **Scope creep from "do anything"** (A1) | High | Medium | A7 is the bar — *"automate some few things"* — not broad capability |
| 6 | **MCP spec churn** — 2026-07-28 removed protocol-level sessions | Medium | Medium | SDK v2 does the work; no hand-rolled session management |
| 7 | **Tauri mobile maturity** (Phase 4 only) | Medium | Low | Thin clients; core API unchanged if the shell is swapped |
| 8 | **Front end being secondary** (C1, B1) — the app may feel rough for a while | Certain | Low | User's explicit choice. Terminal-first is accepted, not a risk to manage |

## 6. Decisions log

Recorded so the reasoning survives being forgotten.

| # | Decision | Why | Source |
|---|---|---|---|
| 1 | Tauri 2 over Electron | Only shell reaching Windows + Android + iOS from one codebase | B1 delegation + research |
| 2 | TypeScript over Python | MCP's only Tier 1 SDK is TypeScript | B1 + research |
| 3 | Write the agent loop, don't adopt a framework | ~200 lines; user wants to own the plumbing; framework lock-in contradicts the local-later plan | D3/build approach + research |
| 4 | Adopt `SKILL.md` rather than invent a skill format | It is a real open standard used by 8+ agent tools | A4 + research |
| 5 | MCP SDK **v2**, not v1 | v2 is current stable; v1 is maintenance-only | Research |
| 6 | Default deny, everything prompts | User rejected the "read actions run freely" tiering | A8 |
| 7 | Scope-based trust for self-authored skills | User chose maximum capability over approval-gating | Contradiction 2 |
| 8 | Per-principal memory isolation | Resolves the A5 + E1 contradiction | Contradiction 1 |
| 9 | Only the user's account, for now | Multi-user waits until Zeno is proven | A6 |
| 10 | Free tier only | User's explicit budget choice — with the $10 top-up still open | A9 |
| 11 | Terminal first, app later | User's explicit two-phase request | C1 |
| 12 | Text + interrupt now, voice later | User's explicit two-phase request | C2/C5 |

## 7. Immediate next step

Answer the seven prerequisites in §2 — **especially decision 1 ($10 top-up) and
decision 2 (which accounts)** — and Phase 0 can start.

Until then, this document and the other five are the deliverable.

---

**Traceability:** every phase deliverable traces to a PRD requirement, a BSD table, or
an intake answer. Where an intake question is unanswered, the plan says so rather than
filling the gap.
