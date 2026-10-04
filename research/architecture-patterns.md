# Architecture patterns — what independently emerged, and the deterministic/LLM boundary

## Patterns that appeared independently in ≥3 projects (strongest evidence they're right)

### 1. Deterministic skeleton, LLM as bounded proposer (the single most repeated pattern)

- DailyOS: deterministic ranking + schedule fill; LLM proposes a timeline; zod validation, ≤3
  repair attempts feeding errors back, deterministic fallback if repair fails
  (`notes/dailyos.md` §4).
- OpenClaw memory: "Deterministic gates, model judgment inside them" is a named design principle;
  dreaming promotion is threshold-gated in code, the model only consolidates gated candidates,
  and its output must pass structural validation or the sweep falls back to append-only
  (`docs/concepts/memory-architecture.md`, verified by parent).
- OpenClaw standing intents: trigger matching is pure FTS in code — **no model call in the
  trigger path**.
- CORE: LLM outputs are always structured calls with cache keys; task state machine is plain TS.
- Hermes: goals loop uses a deterministic quality gate (shell command output *is* the
  continuation prompt); FTS search tool: "No LLM calls — every shape returns actual DB messages."

**Executor rule:** the LLM proposes, code disposes. Every LLM output crossing into state passes
schema validation + invariants (DailyOS pattern), with a deterministic fallback.

### 2. Files for humans, SQLite for machines — with explicit canonicity direction

- OpenClaw: Markdown files canonical; **all memory SQLite is a rebuildable derived index**
  (comment in `memory-schema-base.ts`: "Only rebuildable index state belongs here").
- Hermes: inverted — SQLite/FTS5 canonical for sessions/messages; MEMORY.md/USER.md tiny
  curated caps; skills accrete as files.
- personal-assistant: markdown-in-git canonical, sqlite-vec+BM25 as derived hybrid index.
- CORE: Postgres canonical + markdown skills (policy as data — Watch Rules).

**Executor rule:** two canonical domains — SQLite canonical for *structured entities* (needs
transactions, constraints, queries); Markdown canonical for *prose* (needs human editability,
diff, git). Never let the same fact be canonical in both; link by ID.

### 3. Provenance/trust classes on memory (against silent belief corruption)

- OpenClaw: origin classes `owner/agent/untrusted/system` as SQL CHECKs; session-kind gating;
  taint propagation after network tool results; recall-loop prevention.
- Hermes: approval-gated memory writes with matched-entry pinning; do-not-capture rules in the
  review prompt ("no negative claims about tools… the agent cites against itself for months").
- CORE: VoiceAspect facts carry `validAt/invalidAt/invalidatedBy`.
- eol: machine suggestions only ever ghost-previews until the user accepts.

**Executor rule:** every stored inference carries provenance and a lifecycle
(`draft → confirmed | dismissed`); nothing inferred is ever acted on as truth without
promotion. This directly implements the requirement "must not silently turn guesses into
permanent truths."

### 4. Small curated core + large episodic log + procedural skills (three-way memory split)

OpenClaw (MEMORY/USER + daily notes + skills), Hermes (2.2k-char caps + sessions + skills),
Letta (memory blocks + archives). The convergent reason: always-loaded context must stay small
(prefix cost + attention), so accumulation lives in searchable stores and load-on-relevance
skills. (Details: memory.md.)

### 5. Prospective memory as deterministic triggers, not model polling

OpenClaw standing intents (FTS match, cooldowns, fire budgets, explicit cancellation); Hermes
cron wake-gates; personal-assistant heartbeat gate; DailyOS schedules. Nobody credible lets an
LLM decide "should I remind now?" on a hot path.

### 6. Event-sourced behavior, materialized views for learning

eol (plan vs actual on one row + sessions as events), BuJo migration-as-signal, Taskwarrior's
mod times. The productivity-systems meta-finding: **states are worthless without events** —
deferrals, sessions, re-plans, completions are the learning substrate.

### 7. Approval ladders with edit power

DailyOS (risk tiers + editable decided arguments), Hermes (floors + guardian + breaker +
protected files), OpenClaw (exec approvals binding argv+identity), CORE (2-min veto buffer,
Waiting gate, user-only Done). Best UX pattern: CORE's veto buffer + DailyOS's argument editing
beat modal yes/no approvals for low-friction daily use.

### 8. Plugin/MCP seam between domain and runtime

Hermes `hermes mcp serve` + "plugins never touch core"; OpenClaw plugin SDK with own-SQLite
plugins; personal-os as an MCP server; DailyOS tool registries. For Executor this is the
portability insurance: the domain layer speaks MCP, so the runtime is replaceable.

## The deterministic vs LLM boundary for Executor (Question B)

### Deterministic (code owns truth and execution)

- IDs, timestamps, all state transitions (task lifecycle), dependencies, recurrence (RRule),
  due/defer arithmetic, time-window math, timezone handling (DailyOS's offset-bearing ISO
  validation is the prior art)
- storage transactions, audit/event log, git commits of prose
- candidate filtering + scoring inputs: due-ness, deferral counts, estimate-fit, WIP caps,
  context/energy filters, commitment deadlines
- trigger matching (standing-intent style FTS/keyword), cron, heartbeat cadence, notification
  budgets/cooldowns
- estimate statistics (medians, p80 padding) — pure SQL over events
- promotion gates: what's eligible to become durable memory/preference (thresholds in code)
- permissions/approval policy evaluation (OpenClaw: "approvals can only tighten, never loosen")
- dedup (personal-os's SequenceMatcher+Jaccard prior art), idempotency keys for commitments

### LLM (judgment inside deterministic bounds)

- brain-dump interpretation → *proposed* tasks/projects/contexts (validated against schema;
  clarify-gate when under-specified — personal-os pattern)
- micro-step decomposition (the flagship LLM job; constrained by: first action ≤2 min, 3–7
  steps, stop condition, "if blocked" plan; grounded in retrieved context)
- morning MIT proposal and "now/next" re-ranking *of a deterministic shortlist*, with rationale
- summarization: journal, project status, evening review drafts
- behavioral *hypothesis* generation ("this task has failed to start 4× at evenings; it's a
  morning task?") — output is a draft inference with provenance, subject to promotion
- commitment extraction from messages/email (DailyOS pattern: LLM labels, code upserts
  idempotently, user confirms)
- memory consolidation passes (OpenClaw dreaming pattern: gated candidates in, validated
  rewrite out)

### Never LLM

- whether a reminder fires (code decides from policy + budgets)
- whether an action was completed (user action or explicit user statement only — CORE: "only
  the user marks Done")
- what counts as observed fact vs inference (schema-enforced provenance)
- security policy application

## Anti-patterns catalog (observed in the wild)

- **Prompt-only planning logic** (nudge, adhd-day-planner): behavior that matters lives in
  prose; nothing enforces it; scripts even contradict it (cleanup deletes tasks the LLM dropped).
- **Prompt-encoded state machines** (CORE): prompts drift from code after refactors (`think`
  tool ghost; empty scratchpad branch shipping behind docs that describe it).
- **JSON-blob linkage** (DailyOS task→project via `source_metadata_json`): unqueryable,
  no integrity.
- **History rewriting** (eol's earlier rewrite "scrambled the timeline"): behavioral data must
  be append-only; today is a view, not a mutation.
- **Learned guesses applied silently** (the thing no surveyed project does — and the thing
  Executor must not do either): every derived preference surfaces as a ghost-preview/draft.
- **SaaS-first OSS cores** (CORE): billing/OAuth/workspace entangled with domain logic.
- **Auto-scheduling that decides for the user without reshaping on derailment** (Motion-style;
  the market's own fork — rigid days crumble under ADHD derailment; see productivity-systems.md).
- **Streaks/guilt mechanics**: ADHD-affirming design (Life_OS, nudge) explicitly refuses them.
