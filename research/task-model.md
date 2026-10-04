# Task model — the canonical data model (Question A)

## What the surveyed models actually represent

| Concept | nudge | DailyOS | PersonalOS | CORE | eol | Executor verdict |
|---|---|---|---|---|---|---|
| goals | – | ✔ (table) | prose file | – | – | skip as entity; `area` + project `why` covers it |
| areas | – | – | – | – | – | **add** (PARA; the missing dimension everywhere) |
| projects | idea files | ✔ (FK to goal) | – | – | – | **entity** |
| tasks | idea files | flat table | `Tasks/*.md` | Task table | todos | **entity** |
| subtasks/steps | prose sections | – | – | ✔ (depth-capped) | child todos | **child tasks with own telemetry** |
| next action | prompt convention | – | WIP-capped P0 | Ready status | – | **materialized pointer on project** |
| commitments | – | ✔ (email/meeting) | – | – | – | **entity** (broaden sources) |
| dependencies | – | – | – | Waiting/unblock | – | **status + blocked_by** |
| routines/habits | – | free-text memory | – | RRule on tasks | habit fields | **entity with RRule + occurrences** |
| appointments | – | calendar mirror | – | integration | – | mirror + anchor events |
| deadlines | due | due_at + commitments | priority | schedule | due_date | due/defer + commitment links |
| observations/behavior | – | – | – | – | **plan vs actual, sessions** | **event log (core!)** |
| inferences | – | – | – | – | ghost previews | **draft inferences + promotion** |
| decisions | daily log | – | – | – | – | journal entries + decision notes |
| user state | energy frontmatter | memories | – | VoiceAspects | work-type prefs | profile + per-day state row |

Cross-cutting findings: nothing needs "goals" as a separate table (goals are projects with
aspirational framing, or areas); everything that models steps as *data* (CORE, eol) gets
per-step telemetry for free; the GTD next-action constraint ("a project always exposes exactly
one concrete next action") appears only as prompt prose (nudge) — Executor should materialize it.

## The smallest useful canonical model (recommended)

SQLite, canonical for entities. ~10 tables + events. (DDL sketch — fields trimmed to essentials.)

```sql
-- Responsibility domains that never complete (PARA "areas")
CREATE TABLE area (id TEXT PRIMARY KEY, title TEXT NOT NULL,
  review cadence TEXT, status TEXT DEFAULT 'active');

-- Outcome-shaped work. Prose canon in markdown: why/outcome/log linked by project_id.
CREATE TABLE project (id TEXT PRIMARY KEY, area_id REFERENCES area,
  title TEXT NOT NULL, status TEXT NOT NULL DEFAULT 'active',
    -- active | someday | paused | done | dropped  (someday/maybe is a status, not deletion)
  outcome TEXT, next_action_task_id REFERENCES task,  -- GTD invariant: exactly one
  created_at TEXT, completed_at TEXT);

CREATE TABLE task (
  id TEXT PRIMARY KEY,
  project_id REFERENCES project ON DELETE SET NULL,   -- real FK (not DailyOS's JSON blob)
  parent_task_id REFERENCES task ON DELETE CASCADE,   -- steps are child tasks
  title TEXT NOT NULL, status TEXT NOT NULL DEFAULT 'todo',
    -- todo | ready | doing | waiting | blocked | done | dropped
  energy TEXT,            -- low|medium|high
  size_minutes INTEGER,   -- estimate (agent-proposed, user-correctable)
  context_tags TEXT,      -- @home @errands @computer @phone … (GTD contexts as filter dimension)
  due_at TEXT, defer_until TEXT,
  blocked_by_task_id REFERENCES task,
  waiting_on TEXT,        -- person/event (CORE's Waiting, generalized)
  recurrence_rule TEXT,   -- RRule if this is a routine instance template
  created_at TEXT, completed_at TEXT
);

-- The learning substrate: append-only. Never update history (eol lesson).
CREATE TABLE task_event (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  task_id REFERENCES task NOT NULL,
  kind TEXT NOT NULL,     -- created|planned|expanded|started|paused|completed|missed|
                          -- deferred|renegotiated|dropped|unblocked|not_now
  at TEXT NOT NULL, actor TEXT NOT NULL,        -- user | agent | system
  detail_json TEXT,       -- e.g. {"reason":"low energy","offered_at":"20:00"}
  provenance TEXT NOT NULL DEFAULT 'observed'   -- observed | inferred
);

-- Focus sessions: intent vs actual on one record (eol's core innovation)
CREATE TABLE session (
  id TEXT PRIMARY KEY, task_id REFERENCES task,
  planned_start TEXT, planned_end TEXT,      -- may be null (ad-hoc start)
  started_at TEXT, ended_at TEXT, actual_seconds INTEGER,
  steps_completed INTEGER, outcome TEXT      -- done|partial|abandoned|blocked
);

-- Promises with teeth: to other people, or to institutions
CREATE TABLE commitment (
  id TEXT PRIMARY KEY, task_id REFERENCES task,
  counterparty TEXT, deadline_at TEXT,
  source_kind TEXT,       -- stated | email | calendar | message
  source_ref TEXT,        -- provenance for extraction
  confidence TEXT, status TEXT DEFAULT 'open'  -- open|met|renegotiated|broken
);

-- Calendar mirror (read view of external calendars; anchor events protected)
CREATE TABLE event (id TEXT PRIMARY KEY, source TEXT, external_id TEXT,
  starts_at TEXT, ends_at TEXT, title TEXT, all_day INTEGER, calendar_id TEXT);

-- Routines as first-class templates (not free text, not only RRule on tasks)
CREATE TABLE routine (id TEXT PRIMARY KEY, title TEXT, rrule TEXT,
  anchor TEXT,            -- meal/meds/time anchor (adhd-day-planner heuristic)
  step_template_json TEXT, context_trigger TEXT, active INTEGER);
CREATE TABLE routine_occurrence (id TEXT PRIMARY KEY, routine_id REFERENCES routine,
  date TEXT, completed_at TEXT, skipped INTEGER, note TEXT);

-- Generated micro-plans, versioned and regenerable (see behavior-learning.md §expansion)
CREATE TABLE expansion (
  id TEXT PRIMARY KEY, task_id REFERENCES task NOT NULL, version INTEGER,
  steps_json TEXT NOT NULL,          -- [{action, minutes, physical:boolean}]
  first_action TEXT NOT NULL,        -- the ≤2-minute entry step
  stop_condition TEXT, blockers TEXT, if_blocked TEXT,
  context_refs_json TEXT,            -- what was retrieved to generate it
  created_by TEXT, created_at TEXT, superseded_by TEXT
);

-- Agent hypotheses: never silently true
CREATE TABLE inference (
  id TEXT PRIMARY KEY, subject_kind TEXT, subject_id TEXT,
  hypothesis TEXT NOT NULL, evidence_json TEXT, confidence REAL,
  status TEXT DEFAULT 'draft',       -- draft | confirmed | dismissed
  created_at TEXT, decided_at TEXT, decided_by TEXT   -- user decides
);
```

Derived, not stored (materialize as views): deferral counts, estimate ratios, per-context
completion rates, initiation success by hour — computed over `task_event`/`session`.

## Key design decisions and their sources

1. **Steps are child tasks with their own telemetry** (CORE depth cap; eol steps-as-todos) —
   per-step timers/completions fall out for free.
2. **Deferral is an event, not a counter** (BuJo migration as data; productivity-systems.md) —
   the count is a view; reasons live in `detail_json`.
3. **`someday` is a status** (GTD) — stashing must not delete history.
4. **Next action is a materialized pointer** (GTD; nudge's prose convention) — enables the
   "what now" query without LLM.
5. **Commitments are separate from tasks** (DailyOS) with provenance + idempotency on
   `(source_kind, source_ref)` — promises get deadline math and review cadences tasks don't.
6. **Waiting is a status + counterparty** (CORE's Waiting/unblock protocol) — blocked work is
   queryable, aging visible.
7. **Routines are templates with occurrences** (habit research: context-stable cues) — distinct
   from projects (they don't complete) and tasks (they repeat).
8. **Estimates live on tasks but truth lives in sessions/events** (planning-fallacy correction
   via personal actuals — adhd-research.md §4).
9. **No goals table** — project.why/outcome (markdown) carries aspiration; areas carry
   responsibility. Nothing in the surveyed code earns a third layer.
10. **User state is a small per-day row** (energy/mood/available windows) + profile prose —
    feeding selection, never stored as inference.

## What deliberately does NOT get an entity

- Messages/chat (runtime owns sessions). — Notifications (policy config, not data).
  — Locations/geo (v2 if ever; eol shows the shape). — Contacts (link out to phone; a
  `counterparty` string suffices). — Gamification anything (streaks, XP — rejected by the
  ADHD-affirming projects and the coaching literature).
