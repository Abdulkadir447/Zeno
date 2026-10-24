# Zeno — Technical Requirements Document (TRD)

**Source:** INTAKE.md B1–B8, A8, E7, plus the stack research in INTAKE.md §Research.
The stack was locked by user delegation — B1: *"I don't really know... search what will
be the best thing to do."*
**Status:** Draft for review · **Date:** 2026-10-23

---

## 1. Locked stack

| Layer | Choice | Why |
|---|---|---|
| Language | **TypeScript** | MCP's only Tier 1 SDK is TypeScript |
| Core service | **Node + Fastify** | Small, fast, WebSocket support; an official MCP Fastify middleware package exists |
| Desktop shell | **Tauri 2** | The only shell reaching Windows + Android + iOS from one codebase (C1) |
| Storage | **SQLite + files** | No database server to run on a personal machine |
| Model access | **OpenRouter** behind our own interface | Free tier now, swappable later (A9) |
| Local runtime (later) | **Ollama** | Simplest possible local provider — one interface implementation |
| MCP | **`@modelcontextprotocol/client` v2** | Current stable line; implements the 2026-07-28 spec |
| Skills | **Agent Skills (`SKILL.md`)** | Open standard adopted by 8+ agent tools (A4) |

**Deliberately not chosen:** LangGraph (overkill for a personal assistant), OpenAI
Agents SDK (vendor lock-in contradicts the local-later plan), Electron (no mobile
path), Python (no Tier 1 MCP SDK).

Full sourcing for every row is in INTAKE.md §Research.

## 2. Architecture

```
CLIENTS (thin — contain no AI logic)
  Terminal                          ← Phase 0 (C1)
  Full-screen app + background      ← Phase 2 (C1)
  Android / iPhone                  ← later
        │
        │  localhost API + WebSocket
        ▼
CORE SERVICE
  Agent Loop ──────────────────────────────┐
        │                                  │
        ▼                                  ▼
  Capability Registry             Model Abstraction Layer
  (MCP + Skills)                openrouter.ts  →  ollama.ts
        │                                  │
        └──────────────┬───────────────────┘
                       ▼
              Memory + Principals (SQLite + files)

EXTERNAL CAPABILITIES — discovered, never hardcoded
  MCP servers over stdio / Streamable HTTP
  Agent Skills from disk and from the internet
```

**Two architectural rules:**

1. If business logic lives in a client, it is a bug.
2. If a file imports from a model provider directly, it is a bug.

Rule 2 is what makes the OpenRouter→local swap a config change rather than a rewrite
(A9, local later).

## 3. Model abstraction layer — mandatory, not optional

A9 (free tiers only) turns this from good practice into the single most important
piece of the core.

```ts
interface ModelProvider {
  complete(req: CompletionRequest): Promise<CompletionStream>;
  listModels(): Promise<ModelInfo[]>;
  supportsTools(model: string): boolean;
}

class OpenRouterProvider implements ModelProvider { /* today */ }
class OllamaProvider       implements ModelProvider { /* later, zero app changes */ }
```

**What the free tier forces:**

1. **A fallback chain, not a single model.** Every free model runs on exactly one
   provider with no automatic failover. The core holds an ordered list and switches on
   429, 403, or *"no endpoints found that support tool use."*
2. **Never hardcode a model id.** The free roster turns over completely — DeepSeek R1,
   Qwen3 Coder 480B, Llama 3.3 70B and Llama 4 Scout are all no longer free. Model
   selection is config, with a curated default list updateable without code changes.
3. **Pin provider alongside model.** The same slug behaves differently per provider —
   DeepSeek V4 Flash measured ~81% vs ~58% on TAU-Bench between first-party and
   DigitalOcean.
4. **Turn minimization is a first-class requirement.** Context caching, compressed
   history, a hard cap on loop iterations. This is not optimization, it is viability.
5. **Log every tool-call failure with the provider.** OpenRouter surfaces Tool Call
   Error Rate per provider; we keep our own alongside it.

## 4. MCP integration

**SDK: v2, not v1.** v2 is the current stable line implementing the 2026-07-28 spec;
v1.x is maintenance-only.

**Transports:** **stdio** (spawn local servers) and **Streamable HTTP** (remote).
HTTP+SSE is formally deprecated — not supported.

**Critical spec change.** The 2026-07-28 revision **removed protocol-level sessions**
and made MCP stateless: no `initialize` handshake, and `tools/list` no longer varies
per connection. All MCP lifecycle code must be written against v2 semantics from day
one. Code written around "connect once, keep the session" will not work.

**Per-server lifecycle:**

```
connect (stdio spawn or HTTP)
   → listTools()
   → normalize each tool into a Capability
   → register in the Capability Registry
   → health check
   → on failure: mark unavailable and notify the user
```

## 5. Agent Skills

**Zeno adopts `SKILL.md` outright.** This is not a format decision — it is an existing
open standard, introduced October 2025 and released as open in December 2025, now
supported by Claude Code, Claude.ai, the Claude API, OpenAI Codex, Cursor, Gemini CLI,
Antigravity and Windsurf.

```
skill-name/
├── SKILL.md          # required: YAML frontmatter + instructions
├── scripts/          # optional: executable code (Python/Bash)
├── references/       # optional: documentation, loaded on demand
└── assets/           # optional: templates, resources
```

Required frontmatter: `name`, `description`.
Optional: `license`, `compatibility`, `metadata`, `allowed-tools` (experimental —
pre-approves which tools a skill may use).

Progressive disclosure is built in: core instructions stay in `SKILL.md`, heavy detail
goes in `references/` and loads only when needed. `SKILL.md` should stay under ~500
lines; `scripts/` holds deterministic execution so the agent is not guessing at logic.

**Two sources (D3):**

- **Disk** — locally installed skills, discovered by directory scan
- **Internet** — skills fetched from a source (D2 unanswered — which source)

**Zeno writing its own skills (D3).** The agent produces a `SKILL.md` plus scripts into
a staging directory. Under the contradiction-2 resolution it may run them when the user
asked for the goal. **Two mitigations must ship with this, not after it:**

1. Live visibility of every step (C3, FR-6)
2. A human-readable audit of the generated scripts **before first execution**

This is the largest security surface in the project and it is not deferred.

## 6. Agent loop

```
user message
   │
   ▼
build context  (memory for this principal · capabilities · identity)
   │
   ▼
model call  (streamed)
   │
   ├── no tool call ──────────────────────► return to user
   │
   └── tool call
        │
        ▼
   permission check  (FR-4, server-side)
        │
        ├── denied ──────────────────────► stop and report
        │
        ├── instruction unclear  (FR-5) ─► ask the user, do not guess
        │
        └── execute capability
             │
             ▼
        record result ──► re-loop
```

**Hard limits, all configurable and logged when hit:**

- Maximum iterations per task
- Maximum total tool calls per task
- Maximum tokens per task

These are what stop a confused model burning the free-tier daily quota in a loop (A9).

**Every iteration is logged to SQLite** — see BSD.

## 7. Permission model — server-side enforcement

FR-4 (A8): **default deny, everything prompts.**

This is enforced in the core. It is **not** a prompt-engineering concern — the model is
never trusted to respect a permission boundary.

```ts
interface PermissionService {
  request(principal, capability, args): Promise<PermissionResult>;
}
```

- The only path to execution passes through `PermissionService.request`
- Self-authored skills: authorized within the scope of the user's stated goal
  (contradiction 2 resolution)
- Other principals: always gated by the admin user (A5)

## 8. Memory and principals

- **Per-principal isolation** (contradiction 1 resolution): your context and other
  people's are stored separately and never cross-referenced
- **Remember everything automatically** (E1) — E3 (can it forget?) is unanswered
- Start **file-based**; a vector store is an optimization added only if retrieval is
  genuinely slow
- Schema design: BSD

## 9. Security

- **Secrets:** local config file, gitignored, never in code, never committed
- **Scope:** single machine, local service (A6 — your account, your data, for now)
- **Network binding:** TBD — B4 (reachable from phone on cellular, or same-network
  only) is unanswered
- **Skill scripts:** executable code from two sources, disk and internet. This is the
  largest security surface in the project (D3)
- **E7 unanswered:** anything that must never be stored locally

## 10. Performance

| Requirement | Target | Source |
|---|---|---|
| Time to first token | Under 2s on the free tier | NF-2 |
| Perceived responsiveness | Conversational, not batch | C5 |
| Turns per task | Minimized and capped | A9 |
| Background tasks | Run while the app is closed | D6 |

**A candid note on NF-2.** On the free tier, *fast* is the requirement most at risk.
Measured free-model output speeds range from roughly **8 tok/s** (Laguna M.1) to
**~295 tok/s** (Nemotron 3.5 Lightning). The fallback chain must prefer fast models for
interactive work and route slow, long tasks to background execution.

## 11. Constraints accepted

1. **Free tier only** (A9) — feasible, but the hardest constraint in the project
2. **Terminal-first** (C1) — the UI stays minimal until Phase 2
3. **Text only** (C2) — no voice until later
4. **Single principal** (A6) — multi-user is designed for, not built yet

---

**Every technical choice above is traceable to INTAKE.md.** Where a requirement here
rests on an unanswered intake question, it is marked TBD rather than assumed.
