# Zeno — User Experience Design Specification (UXDS)

**Source:** INTAKE.md C1–C8, D1–D7, plus A8 and the contradiction resolutions.
**Status:** Draft for review · **Date:** 2026-10-23

---

## 1. Interaction model — two phases

**Phase 0–1: Terminal.** Text only. (C1, C2)

**Phase 2+: Full-screen app + background service + popups.** (C1)

Both are clients on the same core API. **Nothing is rebuilt between them.** This is a
direct consequence of C1 ("terminal first, then full-screen app and background") and it
is why the architecture separates core from clients.

## 2. Voice — not now

Text only for now (C2). Voice comes later.

**However** — interrupt and redirect arrive in Phase 0 as **text** interactions (C5).
They are not deferred with voice. They are harder than they sound, and they are the
difference between something the user trusts and something they merely tolerate.

## 3. Design principles

**P1 — Nothing is hidden.** (C3)
Every tool call shows its reasoning and arguments, live. The user sees exactly what Zeno
is doing and why, at every step.

**P2 — Nothing happens silently.** (A8, C3)
Every action requires permission by default, and every action is visible.

**P3 — You are never stuck.** (C5)
The user can stop and redirect Zeno at any point in a task.

**P4 — It asks when it doesn't know.** (A8, FR-5)
If an instruction is unclear, Zeno asks. It does not guess, and it does not silently
proceed.

**P5 — The front end is secondary.** (C1, B1)
The user delegated the stack and said the front end can be done anytime as long as it
works. This document specifies what must be **functional**, not what must be beautiful.

## 4. Surfaces

### 4.1 Terminal (Phase 0)

```
$ zeno

┌─ Zeno ──────────────────────────────────────────────────────┐
│ > Summarize my morning emails and draft replies
│
│ ● Reading mail.google.com via MCP gmail
│   ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓  14 messages
│
│ ● Drafting reply to adeola@example.com
│     subject: Re: Q3 numbers
│     body:    [see summary]
│
│   ? Approve sending this draft?
│     1. Approve      2. Edit      3. Deny
│     4. Always allow this capability
│ >
└────────────────────────────────────────────────────────────────┘
```

Required elements:

- Streamed response text
- Live tool-call stream with reasoning and arguments (C3)
- Permission prompts with numbered choices
- Interrupt input, always available (C5)

### 4.2 Full-screen app (Phase 2)

- **Chat pane** — conversation
- **Transparency pane** — the live tool-call stream, always visible, never collapsed
  away (C3)
- **Approval pane** — pending actions awaiting a decision
- **Background tasks indicator** — running while the app is closed (D6)

### 4.3 Background service

Zeno keeps running when the window is closed (C1). Tasks continue (D6). Popups or toasts
surface results. Whether the user wants a completion notification is **C8 — unanswered**.

## 5. Tool-call transparency — the signature experience

The user specified this more precisely than anything else in the intake (C3: *"show
everything"*), so it is specified precisely here.

**Per tool call, display:**

- The capability name and its **source** (which MCP server or which skill)
- The **arguments** it is passing
- Its **reasoning** — why it chose this call
- Its **result**
- Its **token cost** — free-tier awareness (A9)

**Never collapse this away.** The user chose full transparency over quiet operation. A
"less noise" mode is not offered, because C3 was an explicit choice.

## 6. Approval and clarification flows

### 6.1 Standard permission prompt (FR-4)

```
Zeno wants to: Send email to adeola@example.com

  subject: Re: Q3 numbers
  body:    [see summary]

  1. Approve        2. Edit        3. Deny
  4. Always allow this capability
```

**Option 4 is the escape hatch out of deny-everything, and it is opt-in.** The user
builds allowlists deliberately, never by accident (A8).

### 6.2 Clarification prompt (FR-5)

```
Zeno: Before I do this I need one thing —
  Which spreadsheet do you mean? I see three from the last week.
```

A8: *"if I give it an instruction and it feels it is not clear enough then ask for more
details."* This is a required behavior, not an edge case.

### 6.3 Interrupt and redirect (C5)

```
> stop
  ✓ Stopped. 3 actions completed, 2 not started.
> instead, just summarize them — don't reply to anyone
  ✓ Re-routing.
```

**Redirect must carry completed context.** The user should not have to restate what was
already done. This is a specific requirement, not a nicety.

## 7. Visual style

**Unspecified.** C6 was never answered.

The user mentioned purple and immediately retracted it. **No color decision is
recorded and none will be assumed.** This document specifies layout and behavior only.

## 8. Notifications

| Event | Behavior | Status |
|---|---|---|
| Background task completes | Toast / popup | **TBD** — C8 unanswered |
| Permission required while app is closed | Notification | **Required** by FR-4 |
| Error during a task | Inline in conversation | Minimum specified (D7 unanswered) |
| Model fallback occurred | One notice at task end | Required by A9 |

## 9. Error experience

D5 and D7 are unanswered. Specified here as the minimum that FR-4 and NF-1 require:

- **Capability missing:** tell the user plainly, and say how to add it. Do not pretend
  the task failed for another reason.
- **Tool call failed:** show the actual error, not a generic message.
- **Model failed:** fall back to the next model silently, and report once at the end if
  the task degraded (A9).

## 10. Out of scope for this document

- Voice (C2 — later)
- Mobile-specific UX, Android and iPhone (later)
- Authentication UI for other users (A6 — later)

---

**Every behavior above traces to INTAKE.md.** Where the intake is silent, this document
says so rather than inventing a preference.
