# Zeno — Requirements Intake

Status: **in progress** · Feeds: PRD, TRD, UXDS, AFD, BSD, IPD

---

## Answers received so far

| Ref | Answer | Where |
|---|---|---|
| **A1** | "All of it" — Zeno is meant to do everything, not a narrow job | chat |
| **A2** | **Inbox & messages** — Zeno triages and drafts replies for me to approve | chat |
| **A3** | **STILL OPEN** — what "ultra working AI" means to you | — |
| **A4** | **"Identity skills" was a mistake** — you meant **agentic skills**. Full
definition recorded below — this is now the most important definition in the project | chat |
| **A5** | **They use it, I control it** — other people reach Zeno, but you stay admin
and gate what it may do for them | chat |
| **A6** | **STILL OPEN** — which email/number accounts, and whose | — |
| **A7** | Success = **"when I can automate some few things"** — a handful of real
tasks, not broad capability. Modest, and achievable | chat |
| **A8** | **HARD LINE: nothing happens without permission.** Deleting, modifying,
*everything*. Plus: if an instruction is unclear, **ask for clarification** instead
of guessing | chat |
| **A9** | **Free tiers only** — but **open to the one-time $10 top-up** for 1,000
requests/day | chat |
| **B1** | **Delegated to you.** "I don't really know... front end is secondary, as long
as it's working... search what will be the best thing to do" | chat |
| **C1** | **Terminal-first to get it working.** When it's done: **full-screen app +
background service + popups.** Two-phase, matches the thin-client architecture | chat |
| **C3** | **Show everything** — each tool call with its reasoning and arguments, live | chat |
| **C4** | Follows from C3 — full transparency means per-action approval is visible | chat |
| **A3** | **All three** — reliable, fast, and autonomous matter equally | chat |
| **A6** | **Mine only, for now.** All automation starts with my own account and my own
data. Other people come later, only when it's ready for them | chat |
| **C2/C5** | **Text + stop/redirect mid-task, now.** Voice, interrupt, redirect, full
control — **later** | chat |
| **D3** | **Zeno writes its own skills AND installs existing ones from the internet.**
Not deferred — you want both | chat |
| **E1** | **Remember everything, automatically** | chat |
| **F1/F5** | **Several hours per week. Docs first** — all six documents, then build | chat |

### Contradictions — RESOLVED

**Contradiction 1 (A5 + E1) — resolved: separate memory per person.**
Zeno keeps your context for you and other people's separately. Never mixed, never
cross-referenced. The schema must be built that way from day one even though only you
exist in it initially (A6).

**Contradiction 2 (A8 + D3) — resolved: trust, with a specific reading of A8.**

You chose maximum capability over the approval-gated option I recommended. Recording
your model accurately, because it is coherent and it is what the PRD will contain:

> "Nothing without permission" means **nothing outside the scope of what you asked
> for.** If you say "do X" and Zeno writes a skill to do X, that is authorized — the
> goal was given, the means to reach it are within it.

What this does **not** license: acting on goals you did not set. The boundary is
*scope*, not *approval count*.

**Residual risk, recorded not argued:** a self-authored skill can do more than was
asked. The mitigations that still hold are C3 (show everything, live) and the
executable-script audit — both must ship with skill authorship, not after it.

### Consequences already visible

- **A2 is now concrete:** inbox/message triage and drafting is the first real workflow.
  Everything else is secondary to that.
- **A5 is a permissions product, not a note.** "They use it, I control it" means the
  core needs a **principal model** (you = admin, others = scoped) and per-principal
  policy. This lands in the BSD schema and the TRD security section.
- **A4 maps to a real open standard.** See research below — this is not a design
  decision we need to make, it's one we need to adopt.
- **A9 is the biggest risk in the project right now.** See the free-tier section
  below before we go any further. It is not impossible, but it changes the design.
- **A8 overrides the risk-tier design in GUIDE.md.** I proposed letting `read`-only
  actions run freely. **You just said no action runs without permission.** That means
  the default is **deny-all + ask-every-time**, with allowlists as an *opt-in* you
  build later. This is stricter and simpler — and it is your call, not mine.
- **A8 adds a second behavior: clarification.** "If it feels the instruction is not
  clear enough, ask for more details." That is a first-class agent requirement, not an
  edge case, and it needs to be in the PRD and the agent loop.
- **C1 confirms the architecture.** Terminal-first is exactly the thin-client model —
  a CLI is just another client on the core API. The full-screen app with background
  service and popups in Phase 2 is the same core, new frontend. Nothing needs redoing.
- **C1 color note:** you mentioned purple and immediately retracted it. **No color
  decision is recorded** — the UXDS will leave this open rather than guess.
- **A7 lowers the bar in a useful way.** "Automate some few things" is a target we can
  actually hit, and it means Phase 0 should ship 2–3 real automations, not a framework.
- **C2/C5 is another two-phase answer**, matching C1. Text + interrupt now, voice later.
  Build the interrupt path in Phase 0 — it is harder than it sounds and it is the
  difference between something you trust and something you tolerate.
- **D3 is the largest security surface in the project** and it is not deferred.
- **F1/F5 sets the order of work:** all six documents, then code.

---

## Research owed on the stack (B1)

You asked me to search and recommend. Findings below — sourced, dated, and tied to
the decisions they change. **Everything here is sourced, not remembered.**

### Tauri 2 — desktop shell

**Verdict: confirmed.** It is the only shell that reaches Windows + Android + iOS
from one codebase, which is exactly your constraint.

- Tauri **2.12.0** is the stable line as of 26 Sept 2026; Tauri 3 is still alpha [1](https://unixgaming.org/tauri-desktop-app-tutorial-2026/)
- Desktop is production-proven — Spacedrive, AppFlowy, Clash Verge, GitButler, Hoppscotch run on it [2](https://www.buildmvpfast.com/blog/tauri-v2-vs-electron-desktop-apps-2026)
- Size and RAM are not close: ~3–15 MB bundle, 20–100 MB idle vs Electron's 80–200 MB and 100–300 MB [3](https://vanja.io/tauri-2-new-default/)
- Mobile targets iOS 9+ / Android 8+ (API 26). Android needs the SDK + NDK; iOS needs Xcode and a provisioning profile [1](https://unixgaming.org/tauri-desktop-app-tutorial-2026/)
- **The honest caveat, which I'm not going to hide:** mobile is described as "functional" and fine for internal tools, but production-critical mobile is better served by `flutter_rust_bridge` or UniFFI [4](https://rustify.rs/articles/rust-tauri-v2-desktop-app-tutorial-2026)

**Consequence for the roadmap:** desktop first, mobile is a Phase 4 experiment, not a
promise. The thin-client architecture is what makes that safe — if Tauri mobile
disappoints, the phones become React Native or Flutter apps against the same core
API and nothing else changes.

### MCP TypeScript SDK

**Verdict: use v2, not v1.** v2 is the current stable line and implements the
2026-07-28 spec; v1.x is maintenance-only [5](https://ts.sdk.modelcontextprotocol.io/)

- Packages: `@modelcontextprotocol/server` and `@modelcontextprotocol/client` [6](https://github.com/modelcontextprotocol/typescript-sdk)
- Transports: **stdio** and **Streamable HTTP**. HTTP+SSE is formally deprecated — do not build against it [7](https://modelcontextprotocol.io/specification/2026-07-28/changelog)
- It ships an official **Fastify middleware** package (`@modelcontextprotocol/fastify`) — which matches the backend I proposed. The two are designed to work together [6](https://github.com/modelcontextprotocol/typescript-sdk)
- Client surface is small: `Client`, `StdioClientTransport`, `StreamableHTTPClientTransport` [8](https://github.com/modelcontextprotocol/typescript-sdk/blob/main/docs/client.md)
- TypeScript is the **only Tier 1 SDK** for MCP [9](https://modelcontextprotocol.io/docs/2026-07-28/sdk)

**One thing that will bite us:** the 2026-07-28 spec **removed protocol-level
sessions** and made MCP stateless — no `initialize` handshake, and `tools/list` no
longer varies per connection [7](https://modelcontextprotocol.io/specification/2026-07-28/changelog).
Any code written around "connect once, keep the session" must be written against v2
semantics from day one, not migrated later.

### Agent framework — build it or borrow it?

You said from scratch. Here is the 2026 landscape:

| Framework | What it gives you | Why it does not fit |
|---|---|---|
| **LangGraph.js** | Durable execution, checkpointing, pause/resume, time-travel debugging | Steepest learning curve on the list; a graph runtime is overkill for a personal assistant [10](https://pharosproduction.com/insights/engineering/ai-agent-frameworks-comparison-2026/) |
| **Mastra** | TS-native agents, workflows, memory, evals | Best "borrow" candidate — but a full framework, not a loop [11](https://cadenya.com/other-content/typescript-ai-agent-frameworks-compared) |
| **OpenAI Agents JS** | Native MCP support (April 2026), `needsApproval` for human-in-the-loop | Ties you to OpenAI's model roadmap — directly contradicts OpenRouter-now-local-later [12](https://uvik.net/blog/agentic-ai-frameworks/) |
| **Vercel AI SDK** | Best streaming UI primitives (`useChat`, generative UI) | A UI/streaming library, not an agent runtime [11](https://cadenya.com/other-content/typescript-ai-agent-frameworks-compared) |

The strongest advice in the research, and the one I'm acting on:

> "Build a framework abstraction layer from the start. The AI agent framework
> landscape is evolving rapidly. Defining your own agent interfaces with the
> framework as an implementation detail behind them prevents costly rewrites." [10](https://pharosproduction.com/insights/engineering/ai-agent-frameworks-comparison-2026/)

**Verdict: write the loop ourselves.** A tool-calling loop is roughly 200 lines, and
you said you want to own the plumbing. Define our own `AgentLoop`, `ModelProvider`,
and `CapabilityRegistry` interfaces. Optionally pull in **Vercel AI SDK** later,
purely for streaming UI primitives in the client, and never let a framework own model
routing.

### Models on OpenRouter

OpenRouter standardizes the tool-calling interface across models and providers, so
one integration covers all of them [13](https://openrouter.ai/docs/guides/features/tool-calling). That is a real argument for your choice.

| Role | Model | Why |
|---|---|---|
| Primary / all-rounder | Claude Sonnet 5.5 | Called the best all-rounder of mid-2026; strong at building, fixing, producing documents [14](https://ask-coreai.com/blog/best-openrouter-models-2026-by-use-case) |
| Flagship / hard tasks | Claude Opus 5.5 | Long-horizon agentic work, multi-step changes, code review [15](https://openrouter.ai/collections/tool-calling-models) |
| Cheap volume | DeepSeek V4.1 Flash, GLM 5.3 Flash | Cost-efficient tier for high-volume tool calling [15](https://openrouter.ai/collections/tool-calling-models) |
| Best per-call tool accuracy | Gemini 3.5 Flash | 83.6% on MCP Atlas [16](https://www.aimadetools.com/blog/best-ai-models-for-agents-2026/) |
| Best long-session reliability | MiMo V2.5 Pro | 97.2% tool accuracy sustained over 1,000+ calls [16](https://www.aimadetools.com/blog/best-ai-models-for-agents-2026/) |

**A trap I want to flag before it costs you money:** the same model slug can behave
very differently depending on which provider serves it. One practitioner measured
DeepSeek V4 Flash at ~81% vs ~58% on TAU-Bench between first-party and DigitalOcean.
Quantization is a poor quality proxy, `reasoning.effort` is silently ignored on some
hosts, and pinning three "reliable" providers still collapsed collectively [17](https://redreamality.com/blog/openrouter-agent-model-selection-eval-gates-escalation-routing/).

OpenRouter tracks **Tool Call Error Rate** per provider and uses it for Auto Exacto
routing [13](https://openrouter.ai/docs/guides/features/tool-calling).

**Verdict: pin an explicit provider alongside the model slug, log every tool-call
failure with the provider, and keep a cheap fallback model behind the same
interface.** This is exactly what the model abstraction layer exists for.

### Local models (Phase 4)

Ollama is the right provider to implement, and the hardware bar is lower than you
might think:

| Model | Size | Hardware | Notes |
|---|---|---|---|
| **GLM-4.7-Flash** | 30B MoE, 3B active | ~8 GB | Ollama's own recommended local pick; 128K context; tool-calling 79.5% on agent benchmarks [18](https://www.morphllm.com/ollama-claude-code) |
| **Qwen3.6 35B-A3B** | 35B MoE, 3B active | ~24 GB | Strongest open-weight tool-calling; 256K context; ~100 tok/s on RTX 3090 [19](https://modelfit.io/tools/claude-code/) |
| **Qwen3-Coder-Next** | 80B, 3B active | ~52 GB | Closest to Claude on SWE-bench (~70% vs Claude's ~72%) [20](https://insiderllm.com/guides/pi-agent-local-models-ollama/) |
| **gpt-oss:20b** | 21B MoE | ~12 GB | ~o3-mini quality, adjustable reasoning [21](https://www.morphllm.com/best-ollama-models) |

**The honest ceiling, stated plainly:** even the best local models do not match
Claude Sonnet on hard tasks [20](https://insiderllm.com/guides/pi-agent-local-models-ollama/).
Local is a fallback and a privacy win, not a quality upgrade. That is fine for your
plan — but it means Phase 4 should be measured by "does it work offline and
privately," not "is it as good."

### What this means for the stack decision

You said the front end is secondary and the core has to just work. The research
supports that ordering exactly.

**Locked:** TypeScript · Fastify core service · Tauri 2 desktop · SQLite + files ·
OpenRouter behind our own model interface · Ollama as the future local provider ·
MCP TypeScript SDK **v2**

**Deliberately not chosen:** LangGraph (overkill) · OpenAI Agents SDK (vendor lock-in
contradicts your plan) · Electron (no mobile path) · Python (MCP's only Tier 1 SDK is
TypeScript)

**Still open in the stack:** the primary free model (see A9 — free tiers only, which
changes this), and whether we build a cost guardrail in Phase 0.

---

### Agent Skills — the format you were describing

This is the biggest finding in the research, and it changes the design.

When you described "agentic skills" — something like the coding skills in Claude
Code, or a front-end design skill, that "just works on its own, runs on prompt, takes
actions for me, combines tools" — you were describing **Agent Skills**, which is a
real open standard, not a concept I need to invent.

Anthropic introduced the format in **October 2025** and released it as an **open
standard in December 2025**. It is now supported by Claude Code, Claude.ai, the
Claude API, OpenAI Codex, Cursor, Gemini CLI, Antigravity and Windsurf [1](https://github.com/ComposioHQ/awesome-claude-skills).

**The format is a directory and one required file:**

```
skill-name/
├── SKILL.md          # Required: YAML frontmatter + instructions
├── scripts/          # Optional: executable code (Python/Bash)
├── references/       # Optional: documentation, loaded on demand
└── assets/           # Optional: templates, resources
```

`SKILL.md` is YAML frontmatter plus Markdown:

```yaml
---
name: summarize-meetings        # required, max 64 chars, lowercase + hyphens
description: Summarizes a meeting transcript into decisions, action items, and open questions. Use when the user pastes a meeting or asks what came out of it.   # required, max 1024 chars
license: MIT                     # optional
compatibility: requires network  # optional, max 500 chars
allowed-tools: web_search, write_file   # optional, experimental
---

# Summarise meetings
[the actual instructions the agent follows]
```

**Why this matters more than it looks:**

- **Progressive disclosure** is built into the design. Core instructions stay in
  `SKILL.md`; heavy detail goes in `references/` and is loaded only when needed. That
  is exactly what keeps an agent's context window from drowning [2](https://academy.claude.com/courses/introduction-to-agent-skills/configuration-and-multi-file-skills)
- `SKILL.md` should stay under ~500 lines; `scripts/` holds deterministic execution so
  the agent isn't guessing at logic [3](https://github.com/mgechev/skills-best-practices)
- Discovery is filesystem-based — skills are found by scanning a directory, not
  registered through an API [4](https://www.totalum.app/blog/claude-code-skills-totalum)
- `allowed-tools` is an **experimental** field that pre-approves which tools a skill
  may use. That is a permission boundary sitting inside the skill format itself [5](https://agentskills.io/specification)

**Verdict: Zeno adopts `SKILL.md` outright rather than inventing a skill format.**
This is not a small optimization. It means:

- Skills you write work in Zeno **and** in Claude Code, Codex, Cursor, and Gemini CLI
- There is a specification to validate against, not opinions to argue about
- The `allowed-tools` field gives us a second layer of permission control, on top of
  the risk tiers from GUIDE.md
- Phase 1 can ship skills that are genuinely useful on day one

The one thing we do need to add ourselves: a **skill installer and manager**, because
Zeno is multi-principal (A5) and needs to know which principal a skill is scoped to.
That part is ours.

### Free tiers only — what that actually costs you

A9 said free tiers only, so this needs to be said plainly rather than quietly
folded into the design. **It is feasible, but it is the hardest constraint in the
project and it touches the agent loop itself.**

**The limits, as of October 2026:**

- **20 requests per minute, always**
- **50 requests per day** on an unfunded account
- **1,000 requests per day** once you have ever purchased **$10 of credits** — and
  that higher limit **sticks permanently**, even if your balance later drops to zero [6](https://pinggy.io/blog/free_ai_model_apis_unlimited_tokens_openrouter/)

**That last line is the single most valuable thing in this research.** A one-time
$10 top-up, not a subscription, moves you from 50 requests/day to 1,000/day forever.
Whether you want to do that is your call, but it is very different from "free means
free," and you should decide it with that number in front of you.

**Why the free tier is hard for an agent specifically:**

- An agent loop spends **several requests per turn** — one per model call, plus retries.
  At 50/day unfunded that is a handful of real tasks. At 1,000/day it is a real product.
- There is a **known bug** where tool calling against `:free` models fails with
  *"No endpoints found that support tool use"* [7](https://lilting.ch/en/articles/openrouter-free-models)
- **Every free model runs on exactly one provider, so there is no automatic failover**
  when that host has a bad day [8](https://www.aieatingtheworld.com/articles/openrouter-free-models)
- The free roster **turns over completely**. DeepSeek R1, Qwen3 Coder 480B, Llama 3.3
  70B and Llama 4 Scout are all **no longer free**. Anything hardcoded to a free model
  id will break without notice [9](https://klymentiev.com/blog/openrouter-free-tier)
- Provider-side rate limits are the real ceiling, not OpenRouter's counters. In one
  test **6 of 19 free models failed**, refused upstream by their providers [9](https://klymentiev.com/blog/openrouter-free-tier)
- The best all-rounder free model, Thinking Machines Inkling, **returns 403 on plain
  API calls** — it is only available to agentic harnesses [9](https://klymentiev.com/blog/openrouter-free-tier)
- Measured uptime varies wildly: most free models held 99.8–100%, but Nemotron 3 Ultra
  sat at **75.7%** and Nemotron 3 Nano Omni at **7.8%** over a single day [8](https://www.aieatingtheworld.com/articles/openrouter-free-models)

One source puts it directly: **"running a serious agent workflow on the free tier is
hard"** [7](https://lilting.ch/en/articles/openrouter-free-models).

**Best free models by job, October 2026:**

| Job | Model | Notes |
|---|---|---|
| Agentic coding | Poolside Laguna S 2.1 | 118B/8B active, 70.2% Terminal-Bench 2.1, 262K context [8](https://www.aieatingtheworld.com/articles/openrouter-free-models) |
| Long-horizon agents | NVIDIA Nemotron 3 Ultra | 550B/55B active, 1M context — but only 75.7% uptime [8](https://www.aieatingtheworld.com/articles/openrouter-free-models) |
| All-rounder | Thinking Machines Inkling | Closest to frontier-class at $0 — **but 403 on plain API calls** [9](https://klymentiev.com/blog/openrouter-free-tier) |
| High throughput | Nemotron 3.5 Lightning | 30B/3B active, 1M context, ~295 tok/s [10](https://yakwp.com/openrouter-review-free-ai-models/) |
| Small coding | Cohere North Mini Code | Host states it does not train on prompts [8](https://www.aieatingtheworld.com/articles/openrouter-free-models) |

**Verdict and what it forces into the design:**

1. **The model abstraction layer is now mandatory, not a nice-to-have.** With a paid
   model you pick one and forget it. On the free tier you need a **fallback chain**
   of 3–4 models and automatic switching when a provider 429s or returns no tool endpoint.
2. **Turn minimization becomes a first-class requirement.** Aggressive context caching,
   compressed history, and a hard cap on loop iterations are not optimizations — they
   are what makes free tier viable.
3. **Never hardcode a free model id.** The roster turns over. Config-driven, with a
   curated default list that can be updated without code changes.
4. **The $10 question needs an answer before Phase 0.** It is the difference between
   "a toy that demos well" and "something you actually use." Your call, but make it
   deliberately.
5. **Local models are more attractive than they were.** GLM-4.7-Flash runs on ~8 GB
   and does tool calling at 79.5% [11](https://www.morphllm.com/ollama-claude-code).
   If free tier is too painful, moving Phase 4 forward is a legitimate strategy, not
   a compromise.

---

## Original questions

Nothing in the six documents gets written from guesswork. Every requirement in them
traces back to a numbered answer in this file. If you skip a question, I'll either
leave it blank in the doc or mark it **"TBD — user to decide"** — your choice which.

Answer inline in this file, in chat, or say *"your call"* for anything you don't
care about.

---

## A. Product — feeds the PRD

**A1.** What is the actual *job* Zeno is doing for you? Not features — the thing you
do today that you want off your plate.
> e.g. "I answer my own emails", "I schedule meetings by hand", "I keep forgetting follow-ups"

**A2.** Describe one real task you'd hand to Zeno tomorrow, in your own words.
> This is the single most useful answer in this file. Be concrete.

**A3.** When you say **"ultra working AI"**, what does that mean to you?
Reliable? Fast? Autonomous? Understands context? Something else?

**A4.** You said **"identity skills"** and **"agency skills"**. What's the difference
in your head? Give one example of each.
> I currently don't have a definition for either — I will not invent one.

**A5.** Is Zeno for **you only**, or will it act on behalf of other people too?

**A6.** The **email and the number** — whose are they? Yours? Someone else's? The
identity Zeno is supposed to adopt when it acts?

**A7.** What would make you say *"this is working"* in three months? And what would
make you say it failed?

**A8.** Anything you explicitly do **not** want Zeno to do, even if it's capable of it?

**A9.** What's a comfortable **monthly API budget**? (OpenRouter spending)

**A10.** How much of your own time are you willing to spend maintaining this?
This decides how much we build vs. how much we adopt.

---

## B. Architecture & stack — feeds the TRD

**B1.** Confirm or change the stack: **TypeScript** core · **Fastify** service ·
**Tauri 2** desktop shell · **SQLite** + files · **OpenRouter** now, **Ollama** later.

**B2.** Where does Zeno run — your laptop only, or an always-on machine at home?

**B3.** Does it need to work **offline**? If OpenRouter is down, what should happen —
fail loudly, degrade, or queue?

**B4.** Should it be reachable from your **phone on cellular**, or only on the same
network as the machine?

**B5.** Security model: single local user, or do you want a **login on the core service**
before a client can talk to it?

**B6.** Where do API keys live — a local config file on your machine? Anything else?

**B7.** Do you already have a language or framework you're strong in and prefer?

**B8.** Performance expectation — must replies feel instant, or is 2–4 seconds fine?

---

## C. Experience — feeds the UXDS

**C1.** Chat window, or something else? (dashboard, command palette, voice-only, panel)

**C2.** Text only, or **voice** input/output too?

**C3.** When Zeno calls a tool, what do you want to see — a full explanation, a
one-liner, or just the result?

**C4.** Approval style: a pop-up **per action**, or a **queue** you review afterwards?

**C5.** Do you want to be able to **interrupt and steer** it mid-task?

**C6.** Visual style preference, or don't care?

**C7.** Windows-first — what does "using it" look like for you? Full-screen window,
tray app, background service with no window at all?

**C8.** Notify you (toast / sound) when a background task finishes, even if the app
is closed?

---

## D. Flows — feeds the AFD

**D1.** Walk me through **connecting your first MCP server**. What should that
experience look like? Where do you get the server, what do you type, what happens?

**D2.** Skills — how do you install one, and where do they come from?

**D3.** Do you want to be able to **create a skill through chat** ("teach Zeno to X")?
> Note: this is a much bigger security problem than it sounds. Your call.

**D4.** What's the flow for *"remind me to do X tomorrow at 9"*?

**D5.** What happens when Zeno hits something it **can't** do?

**D6.** Do you want **scheduled / background tasks** at all, or does everything happen
while you're in the app?

**D7.** How should errors surface to you — inline, a log, a notification?

---

## E. Data — feeds the BSD

**E1.** What should Zeno **remember across sessions**? Give real examples.

**E2.** Should it remember everything, or only what matters? **Who decides** what
gets kept — you, or Zeno?

**E3.** Can Zeno **delete or forget** something you told it? How?

**E4.** Do you want a way to **inspect and edit** everything it has stored about you?

**E5.** Which accounts do you actually use — Gmail, Google Calendar, iCloud,
WhatsApp, Telegram, something else?

**E6.** Do you want logs of every action it took, and how long should they persist?

**E7.** Anything that **must never be stored locally**, under any circumstances?

---

## F. Plan — feeds the IPD

**F1.** Realistic hours per week you can put in?

**F2.** Hard deadline, or open-ended?

**F3.** What is the **first thing** you want working, even if it's tiny?

**F4.** Are you learning the stack as you go, or do you already know it?

**F5.** Do you want **all six documents written first**, then build — or
document-then-build **per phase** so Phase 0 ships sooner?

**F6.** Should I **scaffold the repo now** so there's somewhere real to work, or wait
until the documents are settled?

---

## G. Open questions carried over from GUIDE.md §8–§9

Nothing here is new — I'm just making sure none of it falls through. Each one is
already mapped to an answer above.

| From GUIDE.md | Question | Maps to |
|---|---|---|
| §8 Q1 | OpenRouter budget | **A9** |
| §8 Q2 | Which MCP server to prove Phase 1 against | **D1**, **B** |
| §8 Q3 | Who writes the skills | **D3** |
| §8 Q4 | Multi-identity | **A5**, **A6** |
| §8 Q5 | Offline behavior | **B3** |
| §8 Q6 | What "done" looks like in 3 months | **A7** |
| §9 | Next step | **F5**, **F6** |

---

## How this turns into the six documents

| Document | Built from |
|---|---|
| **PRD** | A1–A10, C1–C2, D6 |
| **TRD** | B1–B8, A8, E7 |
| **UXDS** | C1–C8, D1–D7 |
| **AFD** | D1–D7, C3–C5 |
| **BSD** | E1–E7, A5–A6 |
| **IPD** | F1–F6, A9, A10 |

Every one of the six will carry a short **"source"** line at the top listing which
intake answers it draws from. That way you can audit any claim in any document back
to your own words, and spot anything I invented.
