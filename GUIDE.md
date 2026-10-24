# Zeno — Design Guide

Status: **draft for discussion** · Last updated: 2026-10-23

This is the thinking document, not a spec to implement yet. Read it, argue with it,
and we'll tighten it into a build plan.

---

## 1. What Zeno actually is

Not an app with features. An app that *has* no features by default.

Every capability Zeno gains comes from outside the codebase:

| Source | What it is | Example |
|---|---|---|
| **MCP server** | A live external process exposing tools the agent can call | Blender MCP, Google Calendar MCP, filesystem, a browser |
| **Skill** | A self-contained instruction package + small script you install | "Translate this and post it to X", "Turn meeting notes into a task list" |

So the real product is four things:

1. **An agent loop** — model → tool call → execute → observe → repeat, until it's done
2. **A capability registry** — it knows what it can do right now, and why
3. **A model abstraction layer** — OpenRouter today, a local brain tomorrow, zero app changes
4. **A client** — a window to talk through (Windows first, phones later)

If we build that well, "can it do 3D?" becomes *"have you connected the Blender MCP?"* —
a one-line answer, not a release.

---

## 2. The two hard constraints that shape everything

### Constraint A — OpenRouter now, local later

You can't self-host right now, but it's the end state. That means:

- The model is **never** referenced directly. No `openrouter.chat()` in business logic.
- Every model call goes through one interface: `complete()`, `stream()`, `embed()`, `listModels()`.
- A provider is a module that implements that interface. `openrouter.ts` today, `ollama.ts` later.
- Config decides which provider is active. Changing that is a config edit, not a refactor.

**Rule: if a file imports from `openrouter` directly, it's a bug.**

### Constraint B — Windows first, Android + iPhone eventually

A Windows app cannot later become a phone app. So:

- The **core is a long-running local service** with an API. It owns the model, the
  capability registry, the memory, and the agent loop.
- **Clients are thin.** They render chat, show what the agent is doing, and let you
  approve tool calls. They contain no AI logic.
- Because the core speaks an API, a mobile client is a new frontend, not a rewrite.

**Rule: if business logic lives in the UI, it's a bug.**

---

## 3. Architecture

```
┌──────────────────────────────────────────────────────────┐
│  CLIENTS (thin, no AI logic)                              │
│  Windows  ·  Android  ·  iPhone  ·  CLI  ·  Web           │
└───────────────┬──────────────────────────────────────────┘
                │  localhost API / WebSocket
┌───────────────▼──────────────────────────────────────────┐
│  CORE SERVICE                                              │
│                                                            │
│  ┌──────────────┐  ┌──────────────┐  ┌─────────────────┐ │
│  │ Agent Loop   │  │ Capability   │  │ Memory &        │ │
│  │ (the brain's │  │ Registry     │  │ Identity        │ │
│  │  hands)      │  │ (MCP +       │  │ (who you are)   │ │
│  └──────┬───────┘  │  skills)     │  └─────────────────┘ │
│         │          └──────────────┘                       │
│  ┌──────▼──────────────────────────────────────────────┐ │
│  │ Model Abstraction Layer                               │ │
│  │ openrouter.ts ──── later ──▶ ollama.ts / vllm.ts     │ │
│  └─────────────────────────────────────────────────────┘ │
└───────────────┬──────────────────────────────────────────┘
                │  stdio / streamable HTTP
┌───────────────▼──────────────────────────────────────────┐
│  EXTERNAL CAPABILITIES (discovered, not hardcoded)        │
│  Blender MCP · Google Calendar MCP · filesystem · browser │
└──────────────────────────────────────────────────────────┘
```

### Recommended stack

| Layer | Recommendation | Why |
|---|---|---|
| Language | **TypeScript** | The entire MCP ecosystem is TS-first. Strong types matter when tool schemas are dynamic. |
| Core service | **Node + Fastify** | Small, fast, good WebSocket support for live tool-call streaming. |
| Desktop client | **Tauri 2** | **The one decision that satisfies "Windows now, phones later"** — one Rust shell targeting Windows, Android, and iOS from a single codebase. Electron is heavier and has no real mobile story. |
| Mobile | **Tauri 2 (same code)** or React Native | If Tauri mobile proves awkward, the core API still means this is a frontend swap. |
| Model calls | **OpenRouter** via the abstraction layer | One key, every model, cheap. Swap later without touching the app. |
| Local model runtime (later) | **Ollama** | Simplest possible local provider — one interface implementation and you're done. |
| Storage | **SQLite** (core state, config, logs) + **file-based memory** | No database server to run on a personal machine. SQLite travels with the project. |
| Secrets | **Local config file, gitignored** | API keys never in code, never committed. |

---

## 4. The three subsystems that need designing first

These are the parts where "from scratch" means real work, and where the guide
should get detailed before we build.

### 4.1 Capability Registry

The heart of "do anything." On startup, the core:

1. Reads configured MCP servers, spawns or connects to each
2. Calls `listTools()` on every one
3. Normalizes every tool into one shape: `{ name, description, inputSchema, source, risk }`
4. Merges in installed skills
5. Exposes the union to the agent loop

Each capability carries a **risk tier**, because an agent that can *do* things
needs to be gated:

- `read` — fetching, searching, translating. Runs freely.
- `write` — sending email, creating files, posting. Ask first.
- `spend` — anything with a money or rate-limit cost. Ask first, always.
- `irreversible` — deleting, cancelling, publishing. Confirm explicitly, show the target.

This is the safety model. It should be boring and explicit, not clever.

### 4.2 Skills vs. MCP — one registry, two sources

Don't build two systems. Both produce the same normalized capability entry:

- **MCP tool** = a tool the agent calls, defined by someone else's server
- **Skill** = a bundle of *instructions* the agent reads, plus optional local scripts
  it can execute. A skill is markdown + a manifest. It teaches the agent how to
  use existing tools to accomplish a task.

Blender MCP gives you the tools. A *skill* tells the agent "when the user says
'make me a room', connect to Blender, generate the geometry, set lighting, render."
Both appear in the same list. The agent doesn't care which is which.

### 4.3 Identity & Memory

You described giving it *an email and a number*. That's the anchor — and it's
worth designing as a first-class thing rather than a config field:

- **Identity profile** — your name, how you want things written, your accounts,
  your contacts, timezone, language, boundaries ("never send on my behalf without asking")
- **Long-term memory** — things Zeno should remember across sessions. Start
  file-based and simple. A vector store is an optimization you add when file-based
  retrieval is actually slow, not a starting requirement.
- **Per-capability state** — Blender sessions, calendar connections, auth tokens

---

## 5. The agent loop

"Ultra working AI" is mostly plumbing. The loop:

```
user message
   │
   ▼
┌───────────────┐    yes    ┌──────────────┐
│ build context │──────────▶│ model call   │
│ (memory,      │           │ (streamed)   │
│  capabilities,│           └──────┬───────┘
│  identity)   │                  │
└───────────────┘                  ▼
                         tool_call? ──no──▶ return to user
                              │
                              ▼
                     risk check ──▶ blocked? ──▶ ask user
                              │
                              ▼
                     execute capability
                              │
                              ▼
                     record result, re-loop
```

Non-negotiables for this loop:

- **Stream everything** — tokens and tool calls. A silent agent feels broken.
- **Show tool calls to the user in real time** — what it's about to do, and why.
- **Never let the model call a tool the user blocked.** Enforce server-side, not in the prompt.
- **Cap the loop.** Maximum N iterations, maximum total tool calls, per-task budget.
  Otherwise a confused model burns your OpenRouter balance in a loop.
- **Log every loop** to SQLite. When it breaks, the log is the diagnosis.

---

## 6. Phased roadmap

Deliberately small first. Each phase ends in something you can actually use.

### Phase 0 — Foundation (~2 weeks)
- Repo layout: monorepo, `core/` + `clients/`
- Model abstraction layer with OpenRouter as the only provider
- Hardcoded no-op capability interface (so the loop has something to call)
- Agent loop: single-turn + basic tool calling
- Local config + secrets handling, gitignored
- **Exit:** type a message in a terminal, get a streamed reply, watch a tool call happen

### Phase 1 — Capability system (~2 weeks)
- MCP client supporting **stdio** and **streamable HTTP**
- Capability registry with risk tiers
- Skill format: manifest + markdown, discovery, loading
- Permission prompts in the core, not the UI
- **Exit:** connect a read-only MCP server, see its tools appear, use one

### Phase 2 — Windows client (~2–3 weeks)
- Tauri shell, chat view, live tool-call transparency, approve/deny panel
- Core runs as a local background service
- **Exit:** a real desktop window talking to the real core, approving real actions

### Phase 3 — Memory, identity, autonomy (~2 weeks)
- Identity profile (email + phone as the anchor)
- Persistent memory, file-based
- Background/scheduled tasks (calendar, reminders)
- **Exit:** "remind me to X tomorrow at 9" actually works while the app is closed

### Phase 4 — Local brain + mobile (later)
- `ollama.ts` provider behind the same interface
- Benchmark local vs OpenRouter on your real tasks
- Android + iPhone clients against the same core API
- **Exit:** you turn off OpenRouter and Zeno still works, on your phone

---

## 7. Decisions I've made for you — challenge any of these

| Decision | Recommendation | Reason |
|---|---|---|
| Desktop shell | **Tauri 2**, not Electron | Only option that cleanly reaches Windows + Android + iOS from one codebase |
| Language | **TypeScript** | MCP ecosystem is TS-first; dynamic tool schemas need types |
| Local model runtime | **Ollama** | Simplest possible local provider when Phase 4 arrives |
| Storage | **SQLite + files** | No server to manage on a personal machine |
| Memory | **Start file-based** | Premature vector store is the classic personal-project trap |
| Safety model | **Risk tiers + server-side enforcement** | Trusting the prompt to enforce permissions is a bug, not a feature |
| Scope | **Phase 0 touches no real integrations** | Prove the loop before adding MCP complexity |

---

## 8. Open questions for next discussion

These are the things I'd want answered before Phase 0 starts:

1. **OpenRouter budget** — what's a comfortable monthly spend? This sets the loop
   caps and whether we add a cost guardrail early.
2. **Which MCP server to build Phase 1 against?** Blender is your example, but it's
   a heavy dependency. Google Calendar or a simple filesystem server might be a
   better first proof-out.
3. **Who writes the skills?** You, manually? Or does Zeno write and install its own?
   That's a very different security problem and changes the design significantly.
4. **Multi-identity?** Just you, or is "give it an email and a number" the seed of
   Zeno acting on behalf of *other* people too?
5. **Offline behavior** — if OpenRouter is down, should Zeno degrade to a local
   model automatically, or fail loudly? (Depends on Phase 4 happening.)
6. **What "done" looks like for you in 3 months** — so we can work backward from it.

---

## 9. What I'd do next

Pick one:

- **Write Phase 0 in detail** — file-by-file, interface definitions, the actual
  agent loop signature. Nothing built yet, just the plan nailed down.
- **Decide the stack first** — confirm Tauri + TypeScript + OpenRouter, and I'll
  scaffold the monorepo structure so we have somewhere real to start.
- **Answer the open questions** above and I'll fold the answers into this guide.
