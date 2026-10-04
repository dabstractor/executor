# OpenClaw — Architectural Reconnaissance for Executor

**Target:** /home/dustin/src/openclaw (GitHub: openclaw/openclaw) — READ-ONLY inspection
**Date:** 2026-10-04 · **Version inspected:** 2026.9.8 (commit f9dfa9a3, shallow clone)

---

## Overview

OpenClaw is a **personal, always-on AI assistant gateway** that runs on your own hardware and connects to your existing messaging channels (WhatsApp, Telegram, Discord, Signal, iMessage, Slack, …). It evolved through names Warelay → Clawdbot → Moltbot → OpenClaw (VISION.md). Its own description: "Multi-channel AI gateway with extensible messaging integrations" (package.json). This is exactly the "always-on home computer + phone access" shape Executor wants — and it is the most mature such project in existence.

Core thesis (VISION.md): "OpenClaw is the AI that actually does things. It runs on your devices, in your channels, with your rules." Priorities in order: security & safe defaults, bug fixes/stability, setup reliability, then providers/channels/performance.

## Tech stack & repo stats

- **TypeScript/Node** monorepo (pnpm workspace) + **Rust** crates (`crates/openclaw-gateway-client`, `crates/openclaw-node-host`) + native apps in Swift (macOS/iOS, incl. a wake-word daemon `apps/swabble`), Android, Linux companion.
- SQLite via Node's native `node:sqlite` (root `node-sqlite.mjs`). No external DB dependency.
- **Scale: ~3.85M lines of non-test TypeScript** across `src/`, `extensions/`, `packages/`, `apps/`; **15,513 test files**; **1,337 docs pages** under `docs/`.
- GitHub: **391,278 stars, 9,281 open issues** (issue numbers now in the 164,000s). Extremely active: recent PR/issue activity dated same-day as inspection. Contribution rules cap PRs at ~5k lines; weekly-ish releases with calver (`2026.9.8`); `package.json.openclaw.schemaVersions: {state: 20, agent: 24}`.
- `git log` on the local clone is a shallow single commit (Peter Steinberger, 2026-10-04); real history lives on GitHub.

## ACTUAL data model

### Agent/session DB (per agent: `agents/<agentId>/agent/openclaw-agent.sqlite`)

Schema verbatim highlights from `src/state/openclaw-agent-schema.sql` (1,032 lines):

```sql
CREATE TABLE session_nodes (
  session_key TEXT NOT NULL PRIMARY KEY,
  current_session_id TEXT NOT NULL,
  entry_json TEXT NOT NULL,          -- hot logical-session facts
  snapshot_revision INTEGER NOT NULL DEFAULT 0,
  status TEXT CHECK (status IS NULL OR status IN ('running','done','failed','killed','timeout')),
  created_via TEXT CHECK (created_via IS NULL OR created_via IN ('operator','spawn','channel','cron','talk','run','plugin','internal')),
  owner_actor_type TEXT, owner_actor_id TEXT,
  parent_session_key TEXT, spawned_by TEXT,
  fork_source_session_key TEXT, fork_source_session_id TEXT, ...
);

CREATE TABLE session_windows (       -- transcript generations
  session_id TEXT NOT NULL PRIMARY KEY,
  reason TEXT CHECK (reason IS NULL OR reason IN ('initial','reset','rollover','fork','rewind','switch','recovery','compaction')),
  session_scope TEXT NOT NULL DEFAULT 'conversation' CHECK (session_scope IN ('conversation','shared-main','group','channel')),
  chat_type TEXT, channel TEXT, account_id TEXT, ...
);

CREATE TABLE conversations (
  conversation_id TEXT PRIMARY KEY, channel TEXT, account_id TEXT,
  kind TEXT CHECK (kind IN ('direct','group','channel')),
  peer_id TEXT, delivery_target TEXT, thread_id TEXT, ...
);

CREATE TABLE conversation_deliveries (  -- idempotent outbound delivery ledger
  operation_id TEXT PRIMARY KEY,
  operation_kind TEXT CHECK (operation_kind IN ('send','turn')),
  status TEXT CHECK (status IN ('created','queued','sent','suppressed','rejected','unknown','replied')),
  platform_message_id TEXT, ...
);

CREATE TABLE heartbeat_outcomes (
  session_key TEXT PRIMARY KEY, run_session_key TEXT,
  outcome TEXT CHECK (outcome IN ('progress','done','blocked','needs_attention')),
  summary TEXT, next_check TEXT, priority TEXT, wake_source TEXT, ...
);
```

Also: `session_participants`, `session_members` (identity namespaces), `session_suggestions` (pending/accepted/dismissed drafts), `session_reactions`, `board_tabs`/`board_widgets` (agent-authored HTML/MCP-app/plugin dashboard widgets with `grant_state` approval), `session_progress_cards` (markdown + steps_json), transcript FTS tables, standing-intents tables, session snapshots/archive tables. Doctrine comment at top of schema: "session_nodes.entry_json owns hot logical-session facts; session_entry_snapshots owns keyed cold values."

### Memory index SQLite (Memory Core plugin)

From `packages/memory-host-sdk/src/host/memory-schema-base.ts` and siblings:

```sql
CREATE TABLE memory_index_sources (id, path, source DEFAULT 'memory', hash, mtime, size, UNIQUE(path, source));
CREATE TABLE memory_index_chunks (
  chunk_rowid INTEGER PRIMARY KEY, id TEXT UNIQUE, path, source,
  start_line, end_line, hash, model, text, embedding BLOB, updated_at);
CREATE TABLE memory_index_chunk_provenance (
  chunk_id TEXT PRIMARY KEY,
  origin_class TEXT NOT NULL CHECK (origin_class IN ('owner','agent','untrusted','system')),
  session_kind TEXT NOT NULL CHECK (session_kind IN ('interactive','cron','heartbeat','subagent','unknown')),
  observed_at INTEGER NOT NULL, supersedes_key TEXT);
CREATE TABLE memory_index_chunk_recall_metadata (
  chunk_id TEXT PRIMARY KEY,
  importance INTEGER CHECK (importance IS NULL OR importance BETWEEN 1 AND 10),
  triggers TEXT, project_key TEXT);
-- plus FTS5 tables, vec table, embedding cache (provider, model, hash, embedding, dims)
```

All memory tables are **derived/rebuildable index state** (`MEMORY_INDEX_DERIVED_TABLES`, memory-schema-base.ts) — the Markdown files are canonical.

### Gateway state DB

`~/.openclaw/state/openclaw.sqlite` — gateway-level state incl. `exec_approvals_config`, device pairing store, cron store (with run receipts/trigger state: `src/cron/store/run-receipt-trigger-state.ts`).

## Architecture & control flow

From `docs/concepts/architecture.md` + code:

- **One long-lived Gateway daemon** per host owns all channel connections (WhatsApp via Baileys, Telegram via grammY, etc.). Exposes a typed **WebSocket API** (default `127.0.0.1:18789`) with JSON-schema-validated frames: `req/res/event`. Events: `agent`, `chat`, `presence`, `health`, `heartbeat`, `cron`.
- **Control-plane clients** (CLI, web Control UI, macOS app) and **nodes** (iOS/Android/macOS/headless devices, `role: node` with caps like `camera.*`, `screen.record`, `location.get`) all connect over the same WS. Device-based pairing with challenge-nonce signing (v3 payload binds platform + deviceFamily).
- **Agent loop** (docs/concepts/agent-loop.md): runs are **serialized per session key** (session lane) plus optional global lane. Before streaming, a run records a durable `activeWriterRunId` claim; every transcript append verifies the claim in-transaction (stale runs can't commit). Prompt = base prompt + skills + bootstrap context + per-run overrides. Embedded agent runtime with tool streaming, cancellation, steering, queue modes per channel (steer/followup/collect/interrupt — docs/concepts/queue.md).
- **Deterministic vs LLM:** scheduling, session lanes, writer claims, delivery idempotency, standing-intent matching, memory promotion gates, provenance classification, approval policy evaluation are all deterministic application code. LLM calls are confined to: agent turns, compaction summarization, dreaming consolidation ("model judgment inside deterministic gates"), memory flush extraction, and utility tasks (activity summaries etc.).
- **Hooks:** internal `HOOK.md` scripts for command/lifecycle events; typed plugin hooks at every layer (`before_model_resolve`, `before_prompt_build`, `before_agent_reply`, `agent_end`, `before/after_tool_call`, `tool_result_persist`, `message_received/sending/sent`, `session_start/end`, `gateway_start/stop`, `before/after_compaction`).
- **Compaction** (docs/concepts/compaction.md): auto-compaction near context limit; "safeguard" mode with summary quality audits (required headings, pending asks, exact identifiers must survive; corrective attempts bounded; on failure keeps original history). Tool-call/tool-result pairs kept together; full history stays on disk; CJK-aware budgets.
- **Scheduler** (GatewayScheduler): one host timer drives all wakeups; durable deadline stores; survives sleep by coalescing missed ticks; shutdown budgets.

## Memory / persistence

`docs/concepts/memory-architecture.md` is one of the most sophisticated personal-agent memory designs I've seen. Five design principles (quoted): **1. No hidden state** ("the model only remembers what is written to files in the agent workspace"), **2. Writing is the hard part** (cites LongMemEval arXiv:2410.10813 — curation matters more than indexing; moved off the reply path into background), **3. The write path is the security boundary**, **4. Deterministic gates, model judgment inside them**, **5. Failures never block replies**.

Tier model (verbatim table):

| Tier | Surface | Written by | Injected |
|---|---|---|---|
| Instructions | `AGENTS.md` + workspace instruction files | Human only | Always, at session start |
| Curated core | `MEMORY.md`, `USER.md` | Dreaming consolidation; direct user request | Session start when provenance eligible; budgeted |
| Episodic | `memory/YYYY-MM-DD.md` daily notes, transcripts | Agent during work; flush; transcript capture | On recall; not at session start |
| Prospective | Standing intents (SQLite) + cron jobs | `intent` tool; scheduled tasks | Only when trigger fires |
| Review | `DREAMS.md`, dreaming reports | Dreaming phases | Never |

- **Dreaming** (docs/concepts/dreaming.md): background consolidation, three phases per sweep (light → REM → deep). Only the deep phase writes durably (to `MEMORY.md`); rewrites keep preimages in SQLite for audit; promoted entries get `<!-- trigger: ... -->` tags (≤3 phrases) and `<!-- importance: N -->` (1–10).
- **Provenance enforcement:** every index chunk carries origin_class (owner/agent/untrusted/system) + session_kind. Cron/heartbeat/subagent sessions never produce durable memory candidates (kills scaffolding noise). Recalled content is structurally marked and never re-extracted (recall-loop prevention). Network-derived tool results **taint the whole turn** — assistant text after a web fetch is classified `untrusted` even in an owner turn. "Nothing crosses from episodic to curated without passing the promotion gates."
- `openclaw memory forget` removes tracked entries by source session; admission policy can exclude sources from ingestion.
- **Standing intents** (docs/concepts/standing-intents.md): event-conditioned prospective memory ("when X is mentioned, remind me to Y") stored in agent SQLite. Deterministic FTS keyword prefilter — **no model call in the matching path**; conservative defaults (24h cooldown, max 3 fires, 90-day expiry); lifecycle pending/armed/fired/done/cancelled/expired; cancellation is always explicit, never inferred. Cites TriggerBench/ProEvent research for the failure modes this avoids.

**What this is NOT:** there is no structured task/project/commitment store, no deadlines table, no analytics over postponement/underestimation. Memory is prose-first with search. Executor's behavior-learning layer would be new code on top.

## Scheduling / proactivity

- **Automations** (docs/automation/; `src/cron/`): the built-in scheduler. Job kinds: one-shot timestamps, cron expressions (DOM/DOW OR-logic), intervals, **stream sources**, **condition watchers (event triggers)**, dynamic cadence "pacing", `/loop` chat shortcut. Payloads: agent-turn (with per-run `--message/--model/--fallbacks/--thinking/--light-context/--tools` overrides), command payloads, script payloads. Execution styles: **main session / current / isolated (fresh session) / custom** — isolated jobs get hardening (delivery awareness, auth-profile propagation; see `src/cron/isolated-agent/`). Delivery to chat channel, webhook, or nowhere; failure notifications; run receipts and reconciliation after missed runs.
- **Heartbeat** (docs/gateway/heartbeat.md): proactive check-ins with active hours, minimum spacing, flood control, busy retries; outcomes recorded in `heartbeat_outcomes` (`progress/done/blocked/needs_attention` + `next_check`). Legacy heartbeat `tasks:` blocks were migrated to ordinary automations in v2026.8.1 (doctor-driven) — the system deliberately converged on one scheduler.
- **Inbound webhooks:** Gateway HTTP endpoints trigger agent work (docs/automation/cron-jobs/webhooks).
- **Plan-completion self-check** (PR #160297, being refined in #164896, code `src/agents/embedded-agent-runner/run/attempt-stream-prepare.ts`, key `openclaw.plan-completion-check`): when a run ends with an unfinished saved plan, the agent gets exactly one extra model response to "continue feasible work, reconcile the plan, or explain a concrete blocker." Directly relevant to Executor's initiation-failure concern — this is production machinery against agents stopping prematurely.

## Messaging gateways & routing

- Core channels: Telegram, WebChat, A2A, Reef. Plugin channels (one-command install): WhatsApp, Discord, Signal, iMessage, Slack, Matrix, Google Chat, Teams, IRC, LINE, SMS, Voice Call, Nostr, Twitch, Zalo, and ~20 more (`extensions/` dir).
- Routing: DMs collapse into shared `main` session; groups isolated by default with mention activation; multi-agent routing with isolated sessions per workspace or sender (docs/concepts/multi-agent.md); session attachment; queue steering.
- Multi-user: personal install is single-owner; shared Gateway carries multiple humans, their credit, and history (identity namespaces, `commands.ownerAllowFrom`, access groups, session members/suggestions/reactions — real collaborative-session machinery).

## UI & phone/remote access

- **The primary phone story is "use the messaging apps you already have"** (WhatsApp/Telegram/iMessage/Signal/SMS) — zero additional app needed.
- Native **iOS and Android node apps** (pairing, chat, voice, camera, location, screen record, media playback) — `apps/ios`, `apps/android`.
- Browser **Control UI** + WebChat served by the Gateway; macOS menu-bar app; Linux companion; Windows Hub.
- Remote access doctrine: Tailscale or SSH tunnel (`ssh -N -L 18789:127.0.0.1:18789`); optional TLS + pinning; auth modes token/password/trusted-proxy/Tailscale-identity.
- **Board widgets:** the agent can author dashboard tabs/widgets (HTML, MCP-apps, plugin views) into the Control UI, subject to operator `grant_state` approval — a plausible surface for an Executor "today view."

## Tools & skills

- Built-in tools (docs/tools/): `exec` (host auto/sandbox/gateway/node; PTY; backgrounding; `yieldMs`), `process`, `browser` automation, `computer` use, `web_fetch`, multiple web-search providers, `apply_patch`, `code` execution, **MCP client support** (`docs/tools/mcp.md`, `mcporter` skill), `automations`/cron tool, session/conversation tools, `canvas`, `goal`, `ask_user`, `agents` spawn/wait, `sessions_send`, image/video/music generation, TTS/transcription.
- **Skills** (docs/tools/skills.md): a skill is a directory with `SKILL.md` — YAML frontmatter (`name`, `description`, optional `metadata.openclaw` with install specs like brew formulas/bins) + markdown body of instructions. They teach the agent how/when to use tools; filtered at load by config, env, and binary presence. Loading precedence: workspace `skills/` > project `.agents/skills` > `~/.agents/skills` > state-dir managed > workshop > bundled. ~60 bundled skills (`skills/`), remarkably life-adjacent: apple-reminders, things-mac, obsidian, notion, trello, weather, spotify, sonos, healthcheck, blogwatcher, summarize. Community **ClawHub**; **Skill Workshop** reviews agent-drafted skill proposals before promotion.
- **Plugins** (`extensions/`, docs/plugins/): full extension surface — channels, model providers, tools, hooks, memory providers, context engines, storage providers, CLI backends. Capability-ladder doctrine (AGENTS.md): prefer existing owner → existing plugin contract → narrow SDK capability → universal core. Plugin SDK ships as `packages/plugin-sdk`. A custom life-management plugin with its own SQLite + tools is a first-class citizen, not a hack.

## Security & approval model

- **Exec approvals** (docs/tools/exec-approvals.md): modes deny/allowlist/ask/auto/full; enforced locally on execution host (gateway or node). Approvals can only tighten config, never loosen. Allowlist entries bind canonical argv + resolved executable identity (real-path; content hash for writable executables) + best-effort file-operand binding; drift during approval window = deny. Approvals stored in host SQLite. Native chat approval UX (e.g. Matrix reaction shortcuts ✅/♾️/❌).
- **Sandboxing** (docs/gateway/sandboxing/ — 12 pages): off by default; Docker/Podman/SSH/OpenShell/Crabbox backends; workspace access modes; honest framing: "not a perfect security boundary, but it materially limits filesystem and process access."
- **Prompt-injection defense via provenance:** the memory tier system above is the structural defense — untrusted-origin content can never promote to curated memory; network taint propagation; recall-loop prevention.
- Dedicated `SECURITY.md`, threat-model docs (`docs/security/THREAT-MODEL-ATLAS.md`, `CONTRIBUTING-THREAT-MODEL.md`, incident response, network proxy policy), even a formal-verification page. Device pairing with signed challenges; DM allowlists; session-title identity hiding (configurable per #164875).
- Observed open security-adjacent issues are quality-grade (e.g. #164878 community install provenance reporting), not architectural.

## Provider/model abstraction

Extensions for Anthropic, OpenAI, Google, Bedrock, Azure, Groq, Mistral, DeepSeek, OpenRouter, xAI, and ~40 more; **local models via Ollama, llama.cpp, LM Studio, vLLM, SGLang** (OpenAI/Anthropic-compatible endpoints); model catalog, model failover (docs/concepts/model-failover.md), subscription OAuth, usage tracking. Utility/agent models can differ; cron payloads can override model per job.

## Workspace conventions

`~/.openclaw/workspace` (per-agent): `AGENTS.md` (operating instructions), `SOUL.md` (persona), `USER.md` (directive-based user model with dated active/superseded entries, 4k-char budget), `IDENTITY.md`, `BOOT.md` (startup checklist hook), `BOOTSTRAP.md` (first-run ritual), `MEMORY.md`, `memory/YYYY-MM-DD.md`, `skills/`. Config: `~/.openclaw/openclaw.json` (JSON5). **No Git anywhere in the workspace model** — Executor's Git/Markdown idea would layer cleanly but is not provided.

## Operational footprint (always-on home machine)

Designed exactly for this: one Node Gateway daemon (systemd/launchd supervision documented), SQLite (WAL) storage, optional Docker sandbox, optional native companion apps. Heartbeat + automations make it proactive when idle. Memory dreaming runs as scheduled background jobs. No server infra required beyond the box itself.

## Strengths

1. Solves Executor's hardest infra problems out of the box: always-on gateway, phone access via real messaging apps AND native apps, scheduling with isolated sessions, multi-provider incl. local models, session persistence, compaction, crash recovery.
2. Memory architecture is state-of-the-art for personal agents — provenance-gated, file-canonical, inspectable, with a real answer to prompt-injection-via-memory.
3. Security posture is unusually serious for this genre (approvals, sandboxing options, threat models, safe defaults as priority #1).
4. Extensibility doctrine explicitly supports what Executor needs: skills (instructions) + plugins (code/tools/channels) + workspace files; a custom life-management layer is a supported pattern.
5. Extremely active, extremely tested (15.5k test files), huge community, MIT license.
6. Standing intents + automations + heartbeat cover "prospective memory" (time- and event-triggered action) with deterministic trigger paths.

## Weaknesses / risks for Executor

1. **Scale and churn.** 3.85M LOC, weekly releases, 9.3k open issues, config schema explicitly breaks compat (doctor migrations; VISION.md: "We do not keep long-lived aliases"). Building deeply on internals means tracking a fast target. Mitigation: build against the stable surfaces (workspace, skills, plugins, config) and pin versions.
2. **Chat-centric.** The interaction model is conversations; there is no task list UI, no project dashboard as first-class product, no structured task/project/deadline entities. The Control UI boards are agent-authored widgets, not a task manager.
3. **No behavioral analytics.** "Learn from actual behavior" (postponement counts, underestimation, initiation-failure patterns) does not exist; memory records facts as prose, not measurable task telemetry. Executor's learning layer must be built from scratch.
4. **No Git-backed memory.** The Git/Markdown half of the candidate architecture is absent (Markdown yes, versioning no).
5. Single-maintainer culture at the core (Peter Steinberger) with a large contributor tail; multiple renames suggest identity/searchability churn.
6. The memory system's sophistication is also complexity: provenance rules, dreaming phases, and compaction safeguards take real study to operate well (though defaults are sane).

## Surprising design decisions

- Markdown files are canonical memory; all SQLite memory state is a rebuildable derived index (memory-schema-base.ts comment: "Only rebuildable index state belongs here").
- Origin-class CHECK constraints in SQLite make trust non-forgeable by the model through prose ("provenance metadata stored as SQLite columns the model cannot write through prose").
- Standing intents: pure deterministic FTS matching with fire budgets, citing research on proactive-agent overaction; cancellation is durable state, never model judgment.
- Plan-completion check: one bounded extra model response when the agent stops with unfinished planned work (#160297) — a targeted anti-"gave up too early" mechanism, with live-model regression evidence in PRs (#164896).
- The AGENTS.md for the repo itself encodes an unusually disciplined engineering culture ("One owner per responsibility", capability ladder, proof requirements).
- Heartbeat tasks were deliberately migrated INTO the general cron system (2026.8.1) rather than kept as a separate subsystem.
- Windows/Linux/macOS/iOS/Android companion apps + a local wake-word daemon (swabble) — the "runs on your devices" claim is literal.

## Unfinished / moving parts (from open issues, 2026-10-04)

Issue stream shows active hardening rather than vaporware: #164896 (plan-completion check delivering only a restated blocker — fix in flight), #164895 (Control UI triple-render), #164887 (silent cron turn marked incomplete after success), #164866 (MEMORY.md provenance hash stale after native promotion — memory provenance edge), #164874 (WhatsApp self-chat reply notifications), #164860 (reply reported sent when it only reached transcript). No abandoned megafeatures found; docs are exhaustive and match code structure closely (spot checks of schema/SQL against docs found no disagreements — code is the richer source).

## VERDICT for Executor

**REUSE as the runtime substrate; build Executor's life-management layer as workspace + skills + a custom plugin. Do not fork.**

Reasons:
- The candidate architecture (custom life-management layer + SQLite + Git/Markdown on an agent runtime) maps almost one-to-one onto OpenClaw's own extension surfaces: workspace Markdown for human-inspectable state, plugin-owned SQLite (pattern: memory-core, logbook, workboard each own their SQLite via the same contracts) for structured storage, skills for teaching the agent the life-management workflows, automations/standing-intents for proactive nudges, messaging channels for phone access. Nothing needs forking.
- Forking or rewriting would re-solve security, delivery idempotency, session durability, compaction, provider failover, and 20+ channel integrations at a cost of person-years; the local inspection shows these are done unusually well.
- What Executor must ADD (genuinely missing): structured task/project/deadline schema with queries; behavior telemetry (postponement, estimate-vs-actual, initiation-failure contexts); task decomposition quality (micro-step generation) — likely as an agent skill + plugin tools + its own SQLite tables; Git versioning of workspace memory if desired; an ADHD-friendly "what now" surface (could be a board widget or a dedicated small web app talking to the Gateway WS API).
- Pattern-donor value even if not adopted: memory provenance model, deterministic-gates/LLM-judgment split, standing-intent trigger design, delivery ledger, and the session-window/compaction model are all worth copying in any alternative build.
- Primary risk to manage: upstream churn — pin releases, keep Executor's layer confined to stable extension surfaces, and keep the structured store self-contained so the runtime is replaceable.

Key file references: agent schema `src/state/openclaw-agent-schema.sql`; memory schemas `packages/memory-host-sdk/src/host/memory-schema-{base,provenance,recall}.ts`; memory doc `docs/concepts/memory-architecture.md`; standing intents `docs/concepts/standing-intents.md` + `src/state/openclaw-agent-standing-intents-schema.ts`; automations `docs/automation/cron-jobs/` + `src/cron/`; skills `docs/tools/skills.md` + `skills/`; sandbox `docs/gateway/sandboxing/`; exec approvals `docs/tools/exec-approvals.md`; workspace `docs/concepts/agent-workspace.md`; vision `VISION.md`.
