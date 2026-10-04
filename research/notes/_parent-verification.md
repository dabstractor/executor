# Parent verification log — facts checked against source by the orchestrator itself

Purpose: the final report must distinguish child-agent findings from facts the parent verified
directly in the cloned repositories. Everything below was read by the orchestrator in
~/src/ on 2026-10-04. Clones are `--depth 1` (no history beyond HEAD).

## OpenClaw (github.com/openclaw/openclaw)

- Large TS monorepo (903 MB checkout, pnpm workspace): `packages/` incl. `agent-core`,
  `gateway-client`, `gateway-protocol`, `memory-host-sdk`, `markdown-core`, `worker-runtime`,
  `plugin-sdk`, `model-catalog-core`, `net-policy`; `crates/` (some Rust), `apps/`, `ui/`,
  `skills/`, `custodian-skills/`, `extensions/`, `security/`, `SECURITY.md`.
- Lineage per VISION.md: "Warelay -> Clawdbot -> Moltbot -> OpenClaw". Current priorities:
  "Security and safe defaults; Bug fixes and stability; Setup reliability and first-run UX".
  Companion apps on macOS/iOS/Android/Windows/Linux listed as *next* priorities.
- `docs/concepts/memory-architecture.md` (459 lines) — verified in detail:
  - Memory Core = plain files + one SQLite index. Tiers: Instructions (`AGENTS.md`, human-only),
    Curated core (`MEMORY.md`, `USER.md`), Episodic (`memory/YYYY-MM-DD.md` daily notes + transcripts),
    Prospective (standing intents in SQLite + cron jobs), Review (`DREAMS.md`).
  - Provenance classes on every indexed entry: `owner` / `agent` / `untrusted` / `system`;
    session-kind gating (cron/heartbeat/sub-agent output can't be promoted); recall-loop
    prevention (recalled content can't be re-extracted as new memory); taint propagation after
    network-sourced tool results.
  - Single primary writer for durable memory = "dreaming" background consolidation pass:
    deterministic gate (weighted signals, thresholds; untrusted/system excluded structurally)
    then a bounded model consolidation turn; output must pass structural validation (entry-loss
    budget, file budget) else falls back to append-only.
  - Design principles verbatim: "No hidden state"; "Writing is the hard part" (cites LongMemEval
    arXiv:2410.10813); "The write path is the security boundary"; "Deterministic gates, model
    judgment inside them"; "Failures never block replies".
- `docs/automation/` — automations = built-in scheduler: one-shot & recurring jobs, delivery to
  chat channel/webhook/none, run history, `openclaw cron` alias; webhook + Gmail PubSub triggers;
  condition watchers ("Automation schedules ... condition watchers").
- `docs/automation/standing-orders.md` — standing orders = permanent operating authority defined
  in workspace files (AGENTS.md auto-injected; also SOUL.md, IDENTITY.md, USER.md, BOOTSTRAP.md,
  MEMORY.md auto-injected each session). Program anatomy: Authority / Trigger / Approval gates /
  Escalation rules.
- `docs/channels/` — 40+ channel docs: WhatsApp, Telegram, Signal, iMessage (incl. BlueBubbles),
  Discord, Slack, MS Teams, Matrix, SMS, IRC, Google Chat, Feishu, LINE, Nostr, Twitch, and more.
- `docs/security/` — CONTRIBUTING-THREAT-MODEL.md, THREAT-MODEL-ATLAS.md (315 lines), formal
  verification doc, incident response, network-proxy.
- `docs/concepts/` also has: agent-workspace, compaction, context-engine, dreaming, memory-search,
  multi-agent, multi-user, session-state, standing-intents, user-model, model-failover etc.

## Hermes (github.com/NousResearch/hermes-agent)

- Python project (317 MB checkout), MIT. README verified:
  - "Lives where you do: Telegram, Discord, Slack, WhatsApp, Signal, and CLI — all from a single
    gateway process. Voice memo transcription."
  - "Closed learning loop: agent-curated memory with periodic nudges. Autonomous skill creation
    after complex tasks. Skills self-improve during use. FTS5 session search with LLM
    summarization for cross-session recall. Honcho dialectic user modeling. agentskills.io standard."
  - "Scheduled automations: built-in cron scheduler with delivery to any platform."
  - Subagents; seven terminal backends (local, Docker, SSH, Singularity, Modal, Daytona, Vercel
    Sandbox); provider-agnostic (OpenRouter/OpenAI/custom); TUI + desktop; Termux/Android APT repo.
- Storage (verified by grep over `hermes_state_*.py`, `agent/*.py`): SQLite tables
  `sessions`, `messages`, `messages_fts` (+`messages_fts_cjk`, `messages_fts_trigram` — FTS5 with
  trigram tokenizer), `system_prompts`, `session_model_usage`, `compression_locks`,
  `session_turn_leases`, `gateway_routing`, `gateway_heartbeats`, `async_delegations`,
  `meta`, `schema_version`. Plus separate SQLite in `agent/verification_evidence.py`
  (verification_events/state) and `hermes_cli/kanban_db.py` (a kanban board DB!).
- Learning-loop code files exist: `agent/learning_graph.py`, `agent/learning_mutations.py`,
  `agent/background_review.py`, `agent/context_compressor.py`, `trajectory_compressor.py`.
- `skills/` ships many categories incl. `productivity/`, `note-taking/`, `email/`, `research/`.
- SECURITY.md verified: single-tenant personal agent; "The only security boundary against an
  adversarial LLM is" OS-level isolation (terminal backends); in-process heuristics explicitly
  declared non-boundaries.

## DailyOS (github.com/stadimeti19/DailyOS)

- TS pnpm monorepo: `apps/cli`, `packages/{config,core,database,integrations,logging,shared}`.
- `packages/database/src/schema.ts` (drizzle sqliteTable) — tables verified:
  `tasks` (id,title,notes,status enum pending/in_progress/completed/cancelled, priority enum
  low/normal/high/urgent, dueAt, sourceMetadataJson — NOTE: no projectId FK, no dependencies,
  no estimates, no recurrence), `goals`, `projects` (goalId FK → goals), `commitments`
  (agentRunId, sourceType email|meeting, category promised_action|deadline|reply_needed|question,
  responsiblePerson, deadlineAt+deadlineText, confidence low|med|high, status, sourceReferencesJson),
  `daily_plans` (agentRunId, planDate, timezone, structuredPlanJson, sourceReferencesJson),
  `proposed_actions` (dailyPlanId, actionType, requiresApproval bool, supported bool, snoozedUntil),
  `meeting_briefings`, `agent_runs`, `tool_calls`, `approval_requests`, `agent_events`,
  `memories`, `integration_connections`, `schedules`, `schedule_runs`.
- `packages/core/src/` has real deterministic packages: agent, context, notifications,
  organization, permissions, planning, scheduling, tools.

## CORE (github.com/RedPlanetHQ/core)

- TS pnpm monorepo (`apps/`, `packages/`, `integrations/`, `plugin/`, `hosting/`, `docker/`),
  turbo. `docs/` includes: automations, channels, gateway, integrations, mcp, memory, skills,
  self-hosting, providers, toolkit, access-core, migrating-from-v1.
  Architectural shape closely parallels OpenClaw (channels + gateway + automations + memory).
  Latest commit: "refactor(conversation): drop the trigger.dev row-event bridge".

## Environment note

- Host `pi` 1.0.0 + pi-subagents 0.74.0: detached/foreground workflow children fail
  (`pi-agent-core/node` export missing; fixed upstream in pi 1.0.1+). Delegation executed via
  core `agent_bg` (native pi background agents) instead. Same briefs, same notes-file contract.
