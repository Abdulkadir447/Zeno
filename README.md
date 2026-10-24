# Zeno

A personal AI that does everything — by having no features built in.

Zeno is an agent core plus a capability system. Every ability it has arrives from
outside the codebase: an **MCP server** (Blender, Google Calendar, filesystem) or an
**Agent Skill** it installs or writes itself. If the capability is registered, Zeno can
use it. If it isn't, Zeno can't — and that difference is a one-line answer, not a
release cycle.

---

## Documents

| Document | What it covers |
|---|---|
| **[INTAKE.md](./INTAKE.md)** | The requirements interview. Every answer the user gave, plus the sourced stack research. **This is the source of truth.** |
| **[PRD.md](./PRD.md)** | What Zeno is, who it is for, the 12 functional + 6 non-functional requirements, and what counts as success. |
| **[TRD.md](./TRD.md)** | Locked stack, architecture, model abstraction layer, MCP v2 integration, agent loop, permission enforcement. |
| **[UXDS.md](./UXDS.md)** | Two-phase interaction (terminal now, full-screen app later), the five design principles, and the tool-call transparency spec. |
| **[AFD.md](./AFD.md)** | Ten flows: first run, message → result, tool call → permission, interrupt, MCP connect, skill install, self-authored skill, scheduling, clarification, error. |
| **[BSD.md](./BSD.md)** | Full SQLite schema — principals, sessions, memory, capabilities, approvals, audit trail, jobs — plus the four isolation rules. |
| **[IPD.md](./IPD.md)** | Phases 0–4, exit criteria, risk register, decisions log, and the seven prerequisites that block coding. |

**[GUIDE.md](./GUIDE.md)** is the earlier design discussion that these six supersede
where they differ.

---

## Status

**Documents complete, awaiting review. No code written yet.**

Per F5, all six documents were written before any build begins. Seven open questions
in IPD §2 — especially the $10 OpenRouter top-up and which accounts to connect — are
prerequisites for Phase 0.

Every requirement in the six documents traces to a numbered answer in INTAKE.md. If
something appears in a document that is not in INTAKE.md, it is an error, not a new
idea.
