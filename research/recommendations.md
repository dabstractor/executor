# Recommendations — what to actually build (the 10 answers)

## 1. What should Executor be?

**A new project: a self-contained "life-domain core" running beside (not inside) an agent
runtime — OpenClaw recommended as that runtime, Hermes the credible alternative. Not a fork of
anything.**

Concretely, three deployable pieces:

1. **executor-core** (new code, the only substantial software you write): a small always-on
   service — SQLite (canonical entities, FTS5) + Markdown canon in git + the deterministic
   engines (selection filter/score, expansion pipeline, event/recurrence math, estimate
   statistics) + bounded LLM calls — exposing **MCP tools** and a tiny HTTP API.
2. **Runtime integration** (thin): OpenClaw workspace (EXECUTOR.md, skills, memory dirs) +
   automations/standing-intents wiring + a plugin or MCP-client config pointing at
   executor-core. This is configuration and glue, days not months.
3. **Phone surfaces**: chat via the runtime's channels (WhatsApp/Telegram/Signal) for capture/
   queries/approvals/briefings + a small PWA "today" view over executor-core's HTTP API
   (now/next/MITs, one-tap start/done/defer, expansion reader).

Why this shape: every runtime problem (gateway, phone, scheduling, sessions, providers,
memory conventions, security machinery) is solved better by OpenClaw/Hermes than by any rebuild
— forking or rewriting re-spends person-years. And no existing project contains the domain
layer (verified across all 9 deep-dives + survey): micro-decomposition from life context and
composed behavioral learning are open territory. Owning the domain in a runtime-agnostic
service (MCP is the seam both runtimes honor) also neutralizes the churn risk of building
inside either.

Rejected alternatives, with reasons:
- **Fork CORE:** AGPL, SaaS-coupled (billing/OAuth server in core), 4-service/8GB footprint,
  tasks oriented to agent work not user EF. Pattern source only.
- **Fork/extend DailyOS, Nudge, PersonalOS, eol, adhd-day-planner:** each is a single-maintainer
  artifact missing ≥80% of the requirement (no runtime, or no learning, or no decomposition);
  their value is patterns, which transfer; their code doesn't.
- **Build a standalone runtime too:** re-spends the person-years above; you'd rebuild delivery
  idempotency, approvals, channel fleets, compaction — badly, alone.
- **Hermes instead of OpenClaw:** defensible (Python preference, deeper learning-loop
  machinery); loses native phone apps/Control UI and the markdown-canonical memory alignment.
  The recommendation is OpenClaw *because* executor-core stays swappable.

## 2. Canonical data model

See `task-model.md` for the full DDL sketch. Entities: **area, project** (why/outcome/next_action
pointer, someday-as-status), **task** (project FK, parent_task steps, energy, size, context
tags, due/defer, blocked_by, waiting_on, recurrence), **task_event** (append-only:
created/planned/expanded/started/completed/missed/deferred/renegotiated/…, actor, provenance),
**session** (planned vs actual on one row, per eol), **commitment** (counterparty, deadline,
source provenance, idempotent), **event** (calendar mirror), **routine + routine_occurrence**
(RRule, anchor, step template, context trigger), **expansion** (versioned micro-plans with
first-action/stop-condition/if-blocked), **inference** (draft hypotheses with evidence,
user-promoted). Derived as views: deferral counts, estimate ratios, initiation probabilities,
segment preferences. **No goals table** (project.outcome + areas cover it); no messages,
notifications, streaks, or gamification entities.

## 3. Storage split

| Store | Contents | Why |
|---|---|---|
| **Git/Markdown (canonical)** | project pages (why/outcome/log — CORE Page zones), journal, knowledge notes, EXECUTOR.md curated core, expansion artifacts as readable files | human-editable, diffable, durable backup, agent-readable context; the egress-safe copy of your life |
| **SQLite (canonical)** | all structured entities + events + FTS5 index; invariants, transactions, queries | dependencies, deferral math, recurrence, statistics need a real query engine; markdown can't do this honestly |
| **Embeddings/vector** | *none initially* | FTS5 BM25+trigram suffices at personal scale (Hermes runs exactly this; OpenClaw's own docs say curation beats indexing). Add sqlite-vec later if measured recall gaps appear |
| **Agent/session history** | the runtime's session store (OpenClaw agent.sqlite / Hermes state.db) | it's already durable, searchable, and compaction-managed; don't duplicate |

Linking rule: entities reference markdown by stable path/ID; never duplicate the same fact
canonically in both stores (one fact, one home — Hermes lesson).

## 4. What the agent may change autonomously

- create/modify **draft** tasks, tags, expansions (regenerate versions), journal drafts
- write episodic memory and daily notes (provenance: agent)
- generate **draft inferences** (estimates, patterns, preferences) — display-only until promoted
- reorder proposals within deterministic candidates
- run reviews and prepare decision packets (stale projects, aging commitments)
- write task artifacts for its own work

## 5. What requires user confirmation

- completing/dropping tasks (only the user marks done — CORE rule)
- commitment creation/change; deadlines; messages sent to others; calendar events with others
- routine edits; recurring-task plan changes (CORE's recurring-freeze bug shows why)
- promotion of any inference to confirmed/preference/curated-core
- memory canon edits beyond append (Hermes protected-files + staged writes)
- destructive ops (history is never destroyed, only soft-deleted/archived)
- external writes of any kind (DailyOS tier model: read/local_write auto in trusted context;
  external_write/destructive always gated, with editable arguments)

## 6. Daily-selection algorithm

Three layers + feedback (full detail in `prioritization.md`):
(0) deterministic filter: status/context/energy/defer/WIP/window-fit + commitment pressure;
(1) deterministic score with extensible coefficients: due-ness, deferral pressure (capped),
project momentum, estimate-fit, learned segment preference, initiation likelihood from
history, friction, user-stated focus;
(2) LLM re-rank + one-line rationale over the top-K shortlist (validated subset-only).
Feedback: every offer logs offered→started/not_now/ignored; features (not coefficients) learn
conservatively from events. Morning = 1–3 MITs proposed, one-tap confirm; misses auto-reshuffle;
the universal offer is a 25-min session with the ≤2-min first action attached.

## 7. First usable version (v0.1)

1. **Capture**: brain-dump via phone chat → LLM proposes tasks/projects (schema-validated,
   clarify-gate); confirm creates rows.
2. **Today view**: layers 0–1 only (deterministic now/next/MITs) on a minimal PWA + chat
   command.
3. **Expansion on tap**: the micro-plan contract (first action ≤2 min, 3–7 steps, stop
   condition, if-blocked) generated from project page + task context; versioned.
4. **Behavioral substrate**: start/done/not-now/defer taps + focus-session timer writing
   task_event/session rows. Nothing consumes them yet except durations (median estimates).
5. **Briefings**: morning MIT proposal + evening done-recap via chat (heartbeat-gated).
6. **Runtime integration**: OpenClaw on the home box, Tailscale, one messaging channel, git
   commits of the markdown canon.

Deliberately *not* in v0.1: calendar/email ingestion, routines engine (hand-run cron
automations suffice), inference generation, weekly review automation, board widgets,
embeddings, body-double sessions. v0.2: commitments from messages, routine entities,
diagnosis ladder, weekly review packets. v0.3: standing intents on ingested events,
ghost-preview auto-scheduling, learned initiation probabilities.

## 8. What to reuse

- **OpenClaw** as the runtime (gateway, channels, automations, standing intents, memory core,
  exec approvals, plugin/skills surfaces). Pin versions; stay on stable surfaces.
- **Patterns to port** (not code): DailyOS validate/repair/fallback + approval tiers +
  injection doctrine + idempotent commitments; eol plan-vs-actual schema + conservative
  learning gates + ghost-preview acceptance + immutable history; nudge's system prompt as the
  agent-behavior spec (adapt wholesale — it is the distilled ADHD tone/flow rulebook);
  adhd-day-planner's heuristic corpus as the planner's spec (enforced in code); CORE's
  Page-zones, waiting/unblock, 2-min veto buffer, watch-rules-as-markdown, fact invalidation;
  Hermes's background-review prompts (what to learn/not learn), skill ledger+rollback, FTS
  demotion; personal-assistant's heartbeat gate + secrets-isolated integration proxy;
  personal-os's dedup + WIP caps + clarify-gate.
- **Components worth considering**: ntfy (push fallback), basic-memory (knowledge notes if
  building your own feels heavy), Taskwarrior's urgency formula as calibration reference.

## 9. What NOT to copy

- CORE's Neo4j+pgvector+Redis/Trigger.dev stack, AGPL code, SaaS billing/workspace machinery.
- Nudge/adhd-day-planner's prompt-only planning (no code enforcement) — adopt their *content*,
  never their mechanism.
- DailyOS's JSON-blob task↔project linkage; DailyOS's single-workflow foreground scheduler.
- Hermes's tiny-memory-cap philosophy *for the domain* (fine for the curated core; wrong for
  life context accumulation — that's what SQLite is for).
- eol's browser-hostage scheduling; any history-rewriting carry (their own comments admit the
  scramble).
- Auto-decide-for-the-user auto-scheduling without derailment reshaping (the market's own
  failed fork for ADHD users).
- Streaks, red badges, guilt, gamified debt (all ADHD-affirming projects refuse them; the
  coaching literature backs them).
- Unvetted community skills/plugins in a life-trust context; exposing gateway ports publicly;
  agent-writable credentials; silent application of learned guesses.

## 10. Smallest implementation that genuinely improves life

The v0.1 scope above, which reduces to one sentence: **a phone-reachable thing that always
knows the three things worth doing next, can turn any of them into a ≤2-minute first action on
tap, and quietly records what actually happened so it gets better at both.** That is
achievable as executor-core (~3–5k LOC: schema + engines + MCP/HTTP surface + PWA) plus
OpenClaw configuration. Everything else in this research exists to keep that core honest
(provenance, confirmation gates, deterministic bones) and to give it somewhere to grow
(commitments, routines, diagnoses, reviews) without a rewrite.
