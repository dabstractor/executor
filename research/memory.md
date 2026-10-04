# Memory — how Executor should remember (Question C)

## What the strongest projects actually do

### OpenClaw (the reference design)

Five tiers, Markdown canonical + one derived SQLite index, per-chunk provenance:

| Tier | Surface | Written by | Injected |
|---|---|---|---|
| Instructions | `AGENTS.md` | human only | always, session start |
| Curated core | `MEMORY.md`, `USER.md` | dreaming consolidation; user request | session start, budgeted |
| Episodic | `memory/YYYY-MM-DD.md`, transcripts | agent during work; flush; ingestion | on recall only |
| Prospective | standing intents (SQLite) + cron | `intent` tool | only when trigger fires |
| Review | `DREAMS.md` | dreaming | never |

Key mechanics: promotion through deterministic gates ranked by *recall usefulness* (not
confidence); untrusted/system origins structurally excluded; single-writer consolidation with
validated rewrites and bounded entry loss; recall-loop prevention; ~4k-char budgets on curated
files. Explicit rationale: "Writing is the hard part" — curation quality beats indexing
sophistication (cites LongMemEval).

### Hermes

Three-way split: tiny curated `MEMORY.md`/`USER.md` (2.2k/1.4k chars, frozen snapshot at session
start), skills as the accretion layer (load-on-relevance, ledger+rollback), SQLite+FTS5 as the
episodic layer (BM25 + trigram, cron/subagent demoted, lineage dedup). The after-turn
background-review prompts are the best available codification of *what not to remember*
(no environment-dependent failures, no tool disparagement, one fact one store, fix-in-place).

### CORE

Temporal knowledge graph (episodes/statements/entities, embeddings in pgvector, Neo4j) with
`validAt/invalidAt/invalidatedBy` preference facts and LLM reflection that invalidates
contradicted facts. Real and sophisticated — but requires a 4-service deployment for one user.
Adopt the **model** (facts with validity intervals + explicit invalidation), not the
infrastructure.

## Does Executor need vectors/graphs? No — not initially.

Evidence:
- Personal scale is ~10³–10⁴ documents. Hermes runs BM25 + trigram FTS5 only, deliberately,
  and its search tool disclaims LLM involvement. OpenClaw's memory index has an optional vec
  table but its docs conclude retrieval over well-curated notes "is competitive with far heavier
  designs."
- The personal-assistant repo does use sqlite-vec hybrid search — as *one* signal among BM25 +
  recency — at negligible cost. That's the upgrade path, not the requirement.
- Graph databases buy entity-relationship traversal Executor doesn't need yet; Graphiti's real
  contribution is the bi-temporal fact model, reproducible as columns.

**Decision: SQLite + FTS5 (BM25 + trigram) + structured records + Markdown.** Add an embedding
column + sqlite-vec only when retrieval demonstrably fails (measure recall misses first).

## Executor's memory architecture (recommended)

Five stores, mapped onto the two runtimes' conventions:

1. **Structured state (SQLite, canonical for entities):** tasks/projects/areas/routines/
   commitments/events/task_events/sessions/expansions/draft-inferences. FTS5 over titles+notes.
   This is *not* agent chat memory; it's the life database.
2. **Prose canon (Markdown in git, canonical for narrative):** one file per project
   (`why.md`, `outcome.md`, running `log.md` — CORE's Page zones), journal entries, knowledge
   notes (atomic, linked — Zettelkasten substrate without the ceremony), user profile prose.
   Agent writes are tools that commit; human edits flow back via a sync watcher.
3. **Episodic log (append-only events + runtime session transcripts):** the behavior telemetry
   (see behavior-learning.md). Owned by Executor's SQLite + the runtime's session store
   respectively.
4. **Curated core (small, always-loaded):** `EXECUTOR.md` — who the user is, standing operating
   rules, current focus areas, known friction points. Hard char budget (OpenClaw/Hermes both
   converge on ~2–4k). Promotion to this file is gated (deterministic thresholds + user
   confirmation for identity-level claims).
5. **Procedural memory (skills):** markdown skill files teaching the runtime's agent Executor's
   workflows (capture, expansion, review, diagnosis). The agent may propose skill edits; they're
   approval-gated with ledger+rollback (Hermes pattern).

**Prospective memory** (routines, "when X then Y") lives in structured tables with deterministic
trigger matching — OpenClaw standing-intents pattern (FTS prefilter, cooldowns, fire budgets,
explicit cancellation), plus cron automations for calendar-shaped recurrence.

## Retrieval for task context ("how does a task acquire context")

CORE answers this best and Executor should copy the shape cheaply:

- The task's own **Page** (why/outcome/log) is durable spec — loaded on expansion.
- The parent project's page + next-action linkage.
- On-demand retrieval: FTS5 query generated from task metadata (project key, tags, entities) →
  top-k notes/journal excerpts with recency boost (personal-assistant's hybrid scoring).
- Recent behavioral context: last N attempts (deferrals, sessions, blockers) from task_events.
- Conversation continuity via the runtime's session + compaction (don't rebuild).

## Memory hygiene rules (from observed failure modes)

- One fact, one store (Hermes #30220 fix).
- Never let cron/heartbeat/subagent output become durable memory candidates (OpenClaw
  session-kind gating).
- Never re-ingest recalled content (recall-loop prevention).
- Facts expire: preferences carry validity windows; stale facts invalidate explicitly, not by
  accumulation (CORE VoiceAspect).
- Never remember agent self-talk about tool failures (Hermes do-not-capture).
- The system remembers *for* the user (cognitive offloading is the point — Risko & Gilbert
  2016), so capture must be zero-friction and recall must be push-at-point-of-performance
  (Barkley) — see adhd-design.md.
