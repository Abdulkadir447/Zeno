# Zeno — Application Flow Document (AFD)

**Source:** INTAKE.md D1–D7, C3–C5, plus A8, E1 and the contradiction resolutions.
**Status:** Draft for review · **Date:** 2026-10-23

---

## How to read this document

Each flow is a numbered sequence. **Decision points** are marked ▸ and require a
user response. Where the intake is silent, the flow states the minimum required by
FR-4 (permission) or FR-5 (clarification) rather than inventing behavior.

Actors: **U** = user · **Z** = Zeno core · **M** = MCP server · **S** = Agent Skill

---

## F-01 · First run and startup

```
 1.  U runs `zeno`
 2.  Z starts the core service
 3.  Z reads local config (gitignored) — API keys, model chain, capabilities
 4.  Z scans for MCP servers configured in config
 5.  Z connects to each (stdio spawn, or Streamable HTTP)
 6.  Z calls listTools() on each
 7.  Z normalizes every tool into a Capability entry
 8.  Z scans the skills directory for SKILL.md files
 9.  Z loads name + description of each skill into context
10.  Z builds the Capability Registry
11.  Z health-checks every capability
12.  ▸  Z reports: "N capabilities available. M unavailable."
        (unavailable ones are listed by name, with the failure reason)
13.  Z presents the terminal interface
```

**Notes:**
- Steps 4–11 are the discovery phase described in TRD §4 and §5.
- Unavailable capabilities are surfaced, never hidden (UXDS P1).
- D1 (which MCP server to prove the design against) is **unanswered**.

---

## F-02 · Send a message, receive a result

```
 1.  U types a message
 2.  Z builds context: memory for this principal · capabilities · identity
 3.  Z calls the model (streamed)
 4.  ▸  If the model returns no tool call → Z streams the reply to U, flow ends
 5.  If the model returns a tool call → go to F-03
```

**Notes:**
- Context is built **per principal** (contradiction 1 resolution) — other people's
  memory is never included.
- The reply is streamed, never buffered (TRD §6).

---

## F-03 · Tool call and permission

```
 1.  Z receives a tool call from the model
 2.  Z looks up the capability in the registry
 3.  Z checks the permission policy for this principal
 4.  ▸  Z presents the action to U:
        capability name · source · arguments · reasoning
 5.  ▸  U chooses:
        1. Approve
        2. Edit arguments, then approve
        3. Deny
        4. Always allow this capability (opt-in allowlist)
 6.  If approved → Z executes the capability
 7.  Z records: capability · arguments · result · timestamp · token cost
 8.  Z returns the result to the model
 9.  Z re-loops (back to F-02 step 3)
```

**Notes:**
- Permission enforcement is **server-side** (TRD §7). The model is never trusted to
  respect a boundary.
- **There is no "read actions run freely" path.** A8 rejected that tiering.
- Step 7 is the audit trail required by FR-6.

---

## F-04 · Interrupt and redirect

```
 1.  U types an interrupt at any point during a task
 2.  Z stops the current loop immediately
 3.  Z reports: actions completed · actions not started · current state
 4.  ▸  Z asks what to do next
 5.  U either:
       a. gives a new instruction → Z re-plans FROM COMPLETED STATE
       b. says "continue" → Z resumes from where it stopped
       c. says "stop" → task ends, partial results retained
```

**Notes:**
- C5: *"I want to be able to stop and redirect it mid task, in case it is a mistake."*
- **Redirect carries completed context.** The user must not restate what was already
  done (UXDS §6.3).
- This flow ships in Phase 0 as text, not with voice (C2).

---

## F-05 · Connect an MCP server

```
 1.  U says "connect <server>" or adds it to config
 2.  ▸  Z asks where the server comes from:
        a. local stdio command        → Z spawns it
        b. remote URL                 → Z connects via Streamable HTTP
 3.  Z connects
 4.  Z calls listTools()
 5.  ▸  Z presents the tools discovered, with their risk level
 6.  ▸  Z asks: "Enable all? Enable none? Pick individually?"
 7.  Z registers the enabled tools in the Capability Registry
 8.  Z health-checks and reports
```

**Notes:**
- D1 is unanswered — which server to use first. GLM/Blender were mentioned as
  examples; a filesystem or Google Calendar server is a lighter first proof-out.
- Step 5 surfaces risk level because FR-4 makes every action gated regardless.

---

## F-06 · Install a skill from the internet

```
 1.  U says "install <skill name>" or points Z at a source
 2.  Z fetches the skill (SKILL.md + scripts/ + references/ + assets/)
 3.  Z validates against the SKILL.md spec:
        frontmatter present · name format valid · description present
 4.  ▸  Z presents the skill: name · description · what its scripts do
 5.  ▸  U chooses: install · inspect further · decline
 6.  If installed → Z adds it to the skills directory
 7.  Z rescans and registers the skill in the Capability Registry
 8.  Z reports it is available
```

**Notes:**
- D2 (which source) is **unanswered**. The flow above is source-agnostic.
- Step 4 is the security gate. Skills are executable code (D3) — the user must see what
  they are installing before it runs.

---

## F-07 · Zeno creates its own skill

```
 1.  U asks Zeno to do something that requires a repeatable procedure
 2.  Z determines a skill is warranted (or U says "save this as a skill")
 3.  Z writes SKILL.md + optional scripts/ into a staging directory
 4.  ▸  Z presents the generated skill for review:
        full instructions · every script, human-readable
 5.  Under the contradiction-2 resolution, Z may execute it
        when the user's goal was the request that produced it
 6.  ▸  Z offers: save as a permanent skill · run once · discard
 7.  If saved → skill moves from staging to the skills directory
```

**Notes:**
- D3: Zeno creates its own skills **and** installs existing ones. Both are in scope.
- **Step 4 is a mandatory mitigation, not an optional one.** It ships with skill
  authorship, not after it (TRD §5).
- The user chose **trust** over approval-gating. This flow honors that while keeping the
  one control that was agreed: visibility.

---

## F-08 · Schedule a background task

```
 1.  U asks for a future or recurring action
        ("remind me to do X tomorrow at 9")
 2.  ▸  Z confirms: what · when · which principal's context to use
 3.  Z persists the job
 4.  Z acknowledges and exits the conversation
 5.  When the job fires:
        Z runs the agent loop using the stored context
        ▸  permission prompts are raised as notifications (UXDS §8)
 6.  Z records the result
 7.  Z notifies U of completion (C8 — TBD whether the user wants this)
```

**Notes:**
- D4 describes exactly this. The flow requires the job to run **while the app is
  closed** (D6), which is why the core is a background service (C1).
- D4's exact wording was *"remind me to do X tomorrow at 9"* — reminders are the
  canonical first background task.

---

## F-09 · Unclear instruction → clarification

```
 1.  Z receives an instruction it judges ambiguous
 2.  Z does NOT guess and does NOT proceed
 3.  ▸  Z asks specifically what it needs:
        "Before I do this I need one thing — which spreadsheet do you mean?
         I see three from the last week."
 4.  U answers
 5.  Z proceeds with the clarified instruction
```

**Notes:**
- A8: *"if I give it an instruction and it feels it is not clear enough then ask for
  more details."*
- This is FR-5 and it is a **required behavior**. It is not an error path.
- The prompt must be specific about what is missing, not a generic "can you clarify?"

---

## F-10 · Capability missing or error

```
 1.  Z needs a capability that is not in the registry
 2.  Z does NOT pretend the task failed for another reason
 3.  ▸  Z tells U plainly:
        "I can't do this yet — there is no capability for X."
        "MCP servers that could provide it: [names]"
 4.  ▸  Z offers: connect one now · install a skill · tell me what to build
 5.  If a tool call errors:
        Z shows the actual error, not a generic message
 6.  If the model errors:
        Z falls back to the next model in the chain (A9)
        and reports once at the end if the task degraded
```

**Notes:**
- D5 (what happens when Zeno can't do something) is **unanswered**. The above is the
  minimum required by UXDS P1 and NF-1.
- Step 6 is the free-tier fallback chain from TRD §3.

---

## Flow coverage map

| Intake ref | Flow |
|---|---|
| D1 (connect MCP) | F-05 |
| D2 (install skill) | F-06 |
| D3 (Zeno writes skills) | F-07 |
| D4 (schedule) | F-08 |
| D5 (can't do it) | F-10 |
| D6 (background tasks) | F-08 |
| D7 (errors) | F-10 |
| C3 (show everything) | F-03 step 4, F-07 step 4 |
| C5 (interrupt) | F-04 |
| A8 (clarify) | F-09 |
| A8 (permission) | F-03 |
| A5 (other people gated) | F-03 step 3 (principal-scoped) |
| E1 / contradiction 1 | F-02 step 2 (per-principal context) |

**Unanswered and carried forward:** D1 (first MCP server), D2 (skill source), D5 (error
semantics beyond the minimum), D7 (error detail level).
