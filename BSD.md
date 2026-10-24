# Zeno — Backend Schema Document (BSD)

**Source:** INTAKE.md E1–E7, A5, A6, plus the contradiction resolutions and the
permission model from PRD §4.2.
**Status:** Draft for review · **Date:** 2026-10-23

---

## 1. Storage layout

```
~/.zeno/
├── config.json          # gitignored — API keys, model chain, provider pins
├── zeno.db              # SQLite — all structured state
├── skills/
│   └── <skill-name>/
│       ├── SKILL.md     # required
│       ├── scripts/     # optional, executable
│       ├── references/  # optional
│       └── assets/      # optional
├── memory/              # file-based memory overlay (see §7)
└── logs/
```

**Secrets never enter SQLite.** `config.json` is gitignored and is the only place API
keys live (TRD §9).

## 2. Design rules

**Rule 1 — every stateful table carries `principal_id`.**
Contradiction 1 resolution: memory is isolated per person and never cross-referenced.
This is enforced by schema, not by developer discipline.

**Rule 2 — execution requires an approval row.**
FR-4 / A8: nothing happens without permission. The only path to capability execution
passes through a non-revoked row in `approvals`. Server-side, never prompt-side.

**Rule 3 — every executed action is logged.**
FR-6 / C3: the `tool_calls` table is the audit trail and the source of the transparency
pane.

**Rule 4 — capability defaults to "prompt".**
A8 rejected the "read actions run freely" tiering. There is no tier that skips the
prompt.

## 3. Principals

```sql
CREATE TABLE principals (
  id            TEXT PRIMARY KEY,
  display_name  TEXT NOT NULL,
  kind          TEXT NOT NULL CHECK (kind IN ('admin','scoped')),
  created_at    TEXT NOT NULL DEFAULT (datetime('now')),
  is_active     INTEGER NOT NULL DEFAULT 1
);
```

- `admin` — the user (A6). There is exactly one, and it is created on first run.
- `scoped` — other people (A5). Created later, only when Zeno is ready for them.

**Phase 0–2 has one row.** The schema supports many from the start because building
isolation later is expensive and because A5 is a stated requirement.

## 4. Sessions and messages

```sql
CREATE TABLE sessions (
  id            TEXT PRIMARY KEY,
  principal_id  TEXT NOT NULL REFERENCES principals(id),
  title         TEXT,
  status        TEXT NOT NULL DEFAULT 'active'
                        CHECK (status IN ('active','interrupted','completed','failed')),
  started_at    TEXT NOT NULL DEFAULT (datetime('now')),
  ended_at      TEXT
);

CREATE TABLE messages (
  id            TEXT PRIMARY KEY,
  session_id    TEXT NOT NULL REFERENCES sessions(id) ON DELETE CASCADE,
  principal_id  TEXT NOT NULL REFERENCES principals(id),   -- denormalized; Rule 1
  role          TEXT NOT NULL CHECK (role IN
                        ('user','assistant','system','tool_result','clarification')),
  content       TEXT NOT NULL,
  model         TEXT,          -- which model produced this
  provider      TEXT,          -- which provider served it — free-tier fallback chain (A9)
  token_input   INTEGER,
  token_output  INTEGER,
  created_at    TEXT NOT NULL DEFAULT (datetime('now'))
);

CREATE INDEX idx_messages_session ON messages(session_id, created_at);
CREATE INDEX idx_messages_principal ON messages(principal_id, created_at);
```

`provider` is recorded on every assistant message because on the free tier the same
model behaves differently per host (TRD §3). Without this, a quality regression is
undiagnosable.

`role = 'clarification'` exists because FR-5 makes asking a first-class behavior, not
an error path.

## 5. Long-term memory

```sql
CREATE TABLE memories (
  id                TEXT PRIMARY KEY,
  principal_id      TEXT NOT NULL REFERENCES principals(id),   -- Rule 1
  kind              TEXT NOT NULL CHECK (kind IN
                        ('preference','fact','context','relationship','task','other')),
  content           TEXT NOT NULL,
  source_session_id TEXT REFERENCES sessions(id),
  importance        REAL NOT NULL DEFAULT 0.5,     -- 0..1
  created_at        TEXT NOT NULL DEFAULT (datetime('now')),
  last_used_at      TEXT,
  UNIQUE(principal_id, kind, content)
);

CREATE INDEX idx_memories_lookup ON memories(principal_id, kind, importance DESC);
```

**E1 says remember everything automatically.** The schema does that — nothing is
filtered on write. `importance` exists for *retrieval ordering*, not for deciding
whether to keep something.

**E2 (who decides what's kept) is unanswered.** The default above — keep everything,
rank for retrieval — is the reading that satisfies E1. If the user later says Zeno
should curate, that is a write-path change, not a schema change.

**E3 (can Zeno forget?) is unanswered.** The schema supports deletion. Behavior is
TBD.

## 6. Capabilities

### 6.1 MCP servers

```sql
CREATE TABLE mcp_servers (
  id         TEXT PRIMARY KEY,
  name       TEXT NOT NULL UNIQUE,
  transport  TEXT NOT NULL CHECK (transport IN ('stdio','streamable_http')),
  command    TEXT,          -- stdio
  args       TEXT,          -- stdio, JSON array
  url        TEXT,          -- streamable_http
  headers    TEXT,          -- JSON
  enabled    INTEGER NOT NULL DEFAULT 1,
  health     TEXT NOT NULL DEFAULT 'unknown'
                     CHECK (health IN ('unknown','healthy','degraded','dead')),
  created_at TEXT NOT NULL DEFAULT (datetime('now'))
);
```

Transports match TRD §4 exactly — stdio and Streamable HTTP, nothing else. HTTP+SSE is
deprecated and is not represented here.

### 6.2 Skills

```sql
CREATE TABLE skills (
  id            TEXT PRIMARY KEY,
  name          TEXT NOT NULL UNIQUE,
  description   TEXT NOT NULL,
  directory     TEXT NOT NULL,
  source        TEXT NOT NULL CHECK (source IN ('local','internet','self_authored')),
  origin_url    TEXT,               -- internet
  license       TEXT,
  allowed_tools TEXT,               -- from SKILL.md frontmatter (experimental)
  enabled       INTEGER NOT NULL DEFAULT 1,
  installed_by  TEXT NOT NULL REFERENCES principals(id),
  installed_at  TEXT NOT NULL DEFAULT (datetime('now'))
);
```

`source = 'self_authored'` is the D3 requirement that Zeno writes its own skills. It
exists as a first-class value so self-authored skills can be audited separately from
third-party ones.

`allowed_tools` is carried through from the Agent Skills spec's experimental field — a
second permission layer on top of the `approvals` table.

### 6.3 The registry

```sql
CREATE TABLE capabilities (
  id              TEXT PRIMARY KEY,
  name            TEXT NOT NULL UNIQUE,        -- e.g. 'gmail.send_message'
  description     TEXT NOT NULL,
  source_kind     TEXT NOT NULL CHECK (source_kind IN ('mcp','skill')),
  source_id       TEXT REFERENCES mcp_servers(id),
  skill_id        TEXT REFERENCES skills(id),
  input_schema    TEXT NOT NULL,               -- JSON Schema
  risk_tier       TEXT NOT NULL DEFAULT 'prompt'
                          CHECK (risk_tier IN ('prompt','spend','irreversible')),
  enabled         INTEGER NOT NULL DEFAULT 1,
  last_seen_at    TEXT
);

CREATE INDEX idx_capabilities_source ON capabilities(source_kind, source_id);
```

`risk_tier` is **labeling, not permission.** A8 made everything prompt by default, so
`risk_tier` exists to make the prompt meaningful — *"this will spend money"* reads
differently from *"this will delete a file"*. It does not create a path that skips the
prompt. Rule 4.

The union of MCP tools and skills lands in this one table. The agent loop does not care
which source produced a capability — that is the whole point of the registry.

## 7. Permissions and audit

### 7.1 Approvals — the enforcement point

```sql
CREATE TABLE approvals (
  id              TEXT PRIMARY KEY,
  principal_id    TEXT NOT NULL REFERENCES principals(id),
  capability_name TEXT NOT NULL,
  scope           TEXT NOT NULL DEFAULT 'once'
                        CHECK (scope IN ('once','session','always')),
  granted_by      TEXT NOT NULL REFERENCES principals(id),   -- always the admin (A5)
  granted_at      TEXT NOT NULL DEFAULT (datetime('now')),
  expires_at      TEXT,
  revoked_at      TEXT
);

CREATE INDEX idx_approvals_lookup
  ON approvals(principal_id, capability_name, revoked_at);
```

**This table is the permission model.** A capability executes if and only if there is
a matching row where `revoked_at IS NULL` and either `expires_at IS NULL` or
`expires_at > now`.

- `scope = 'once'` — the default. Single action, single approval.
- `scope = 'session'` — valid until the session ends.
- `scope = 'always'` — **opt-in only** (UXDS §6.1 option 4). A8 made allowlists
  deliberate, never accidental.

`granted_by` is always the admin principal. A scoped principal can never grant itself
permission (A5: *"they use it, I control it"*).

### 7.2 Audit trail

```sql
CREATE TABLE tool_calls (
  id              TEXT PRIMARY KEY,
  session_id      TEXT REFERENCES sessions(id),
  principal_id    TEXT NOT NULL REFERENCES principals(id),   -- Rule 1
  capability_name TEXT NOT NULL,
  arguments       TEXT NOT NULL,              -- JSON
  reasoning       TEXT,                       -- model's stated reason (C3)
  status          TEXT NOT NULL CHECK (status IN
                        ('approved','denied','executed','failed','blocked')),
  result          TEXT,
  error           TEXT,
  model           TEXT,
  provider        TEXT,
  token_input     INTEGER,
  token_output    INTEGER,
  duration_ms     INTEGER,
  created_at      TEXT NOT NULL DEFAULT (datetime('now'))
);

CREATE INDEX idx_tool_calls_principal ON tool_calls(principal_id, created_at);
CREATE INDEX idx_tool_calls_status ON tool_calls(status, created_at);
```

Every row, including **denied** and **blocked** ones. A denied action is as important
to the audit trail as an executed one.

`status = 'blocked'` is distinct from `'denied'`: blocked means the permission service
refused it before it reached the user; denied means the user was asked and said no.

`reasoning` is stored because C3 requires showing why a call was made — and because
that reasoning is the best diagnostic when a task goes wrong.

## 8. Jobs

```sql
CREATE TABLE jobs (
  id             TEXT PRIMARY KEY,
  principal_id   TEXT NOT NULL REFERENCES principals(id),   -- Rule 1
  prompt         TEXT NOT NULL,
  schedule_kind  TEXT NOT NULL CHECK (schedule_kind IN ('once','cron','interval')),
  run_at         TEXT,
  cron_spec      TEXT,
  status         TEXT NOT NULL DEFAULT 'pending'
                       CHECK (status IN ('pending','running','completed','failed','cancelled')),
  last_run_at    TEXT,
  last_result    TEXT,
  created_at     TEXT NOT NULL DEFAULT (datetime('now'))
);
```

D6 requires tasks to run while the app is closed, so jobs are persisted state in SQLite
rather than in-memory timers. The background service picks them up (UXDS §4.3).

## 9. Config

```sql
CREATE TABLE config (
  key   TEXT PRIMARY KEY,
  value TEXT NOT NULL
);
```

Non-secret configuration only. Secrets live in the gitignored `config.json`.

## 10. Isolation enforcement

Contradiction 1 resolution — *separate memory per person, never mixed, never
cross-referenced* — needs to be enforced, not assumed:

1. **Schema.** `principal_id` is `NOT NULL` on every stateful table (Rule 1).
2. **Queries.** Every read on `memories`, `sessions`, `messages`, `tool_calls` carries
   a `principal_id` predicate. Verified by review.
3. **No cross-principal joins on memory content.** A scoped principal's context is
   assembled only from their own rows.
4. **Tests.** A test that inserts two principals and asserts zero leakage between them.

This is cheap to build now and expensive to retrofit. Phase 0 ships it even though only
one principal exists (A6).

## 11. Open items

| Ref | Item | Status |
|---|---|---|
| **E3** | Can Zeno forget? Schema supports it; behavior TBD | **Unanswered** |
| **E4** | A way to inspect and edit stored data | **Unanswered** — needs a UI decision, not just schema |
| **E5** | Which accounts precisely | **Unanswered** — A6 says *"mine"* but not which |
| **E6** | Log retention period | **Unanswered** |
| **E7** | Anything that must never be stored locally | **Unanswered** — if answered, needs a `never_store` flag on relevant tables |
| **E2** | Who decides what is kept | Defaulted to *keep everything, rank for retrieval* per E1 |

---

**Every table above traces to an intake answer or is explicitly marked as an
unanswered-question default.**
