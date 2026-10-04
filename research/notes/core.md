# Architectural Reconnaissance: RedPlanetHQ "CORE" (corebrain)

**Analyst:** delegated research child for Executor
**Target:** /home/dustin/src/core (github.com/RedPlanetHQ/core), read-only
**Snapshot:** single shallow commit `4a5b18d` "refactor(conversation): drop the trigger.dev row-event bridge" (clone depth 1 — local history unavailable; recency taken from GitHub API)

---

## 1. Overview

CORE ("Your Personal AI OS", CLI package `@redplanethq/corebrain`, site getcore.me) is an always-on personal AI butler: a hosted-or-self-hosted webapp plus a local "gateway" daemon on the user's machine. It ingests email/meetings/chat/code events into a Zep-style temporal knowledge graph (Neo4j + pgvector), runs a multi-agent LLM layer (Mastra framework) over Tasks, a collaborative daily "Scratchpad" (Yjs/Hocuspocus), voice (Mac app), and messaging channels (WhatsApp/Slack/Telegram/email). It is the closest existing thing to Executor's brief — persistent, memory-backed, task-centric, phone-reachable — but it is built as a VC-backed multi-tenant SaaS first, OSS second.

## 2. Tech stack & repo stats

- **Repo stats (GitHub API):** created 2025-05-27; ~1,989 stars; 196 forks; 211 open issues; 14 watchers; pushed actively (2026-09). Local clone: 2,183 files (excl. node_modules/.git).
- **License: AGPL-3.0** — critical constraint for any Executor fork/embedding.
- **Monorepo:** pnpm workspaces + Turborepo. `packages/`: cli, database, providers, sdk, types, ui, mcp-proxy, gateway-protocol, emails, hook-utils. `apps/`: webapp (Remix), tauri (Rust Mac app). `integrations/`: ~40 integration packages (gmail, slack, github, linear, jira, notion, todoist, google-calendar, whoop, ynab, stripe, swiggy-*…).
- **Runtime stack** (`hosting/docker/docker-compose.yaml`): `core-app` (Remix webapp, port 3033), `pgvector/pgvector:pg18` Postgres, `redis:7`, `redplanethq/neo4j:0.1.0` (custom Neo4j image), optional Ollama. Min 4 vCPU/8GB.
- **Agent framework:** Mastra (`@mastra/core/agent`) + Vercel AI SDK tool format (`ai` `Tool`). Prisma 5.4.1 ORM.
- **Queues:** dual-provider adapter (`lib/queue-adapter.server.ts`) — Trigger.dev or BullMQ (`QUEUE_PROVIDER`); 5 jobs pinned to in-process BullMQ in every deployment (agent-turn, scratchpad-scan, case, task, scheduled-task) because they publish Redis SSE events (`bullmq/workers/always-on.ts:1-24`).
- **Default model** (`.env.example`): `MODEL=gpt-5.2-2025-12-11`, `EMBEDDING_MODEL=text-embedding-3-small` (1536d), OpenAI `responses` API mode; complexity-tiered model routing (high/medium/low; `LLMModel.complexity`).
- **Gateway daemon:** Fastify server on :7787 inside `packages/cli`; wire contract in `packages/gateway-protocol` (Zod schemas, `PROTOCOL_VERSION = "1"`). Drives Playwright browsers, coding-agent PTYs (Claude Code / Codex), shell exec, file scopes. Prebuilt image `redplanethq/core-gateway`.

## 3. ACTUAL data model (Prisma, `packages/database/prisma/schema.prisma`, 1,561 lines)

Postgres, not SQLite; vectors via pgvector (`Unsupported("vector")`). Key entities:

**Task** (schema.prisma:1232-1298) — the central object:
```
title, description, status TaskStatus, conversationIds String[], result, error
assignedAgentId -> Agents, jobId, metadata Json
schedule String?      // RRule string (null = one-time)
nextRunAt, lastRunAt DateTime?, occurrenceCount Int, maxOccurrences Int?
isActive Boolean, startDate/endDate DateTime?, confirmedActive Boolean, unrespondedCount Int
channel String? / channelId -> Channel
parentTaskId -> Task? (self), subtasks Task[], displayId String? (e.g. tk-1.3.2), childCount Int
pageId String? @unique -> Page   // task's page is its rich description
source String @default("manual")
```
`enum TaskStatus { Todo Waiting Ready Working Review Done Recurring }` (schema.prisma:1222-1230). Note `Recurring` as a *status* (marketing/docs instead say "Todo, In Progress, Done" — docs/concepts/tasks.mdx disagrees with code). Depth cap: epic → task → sub-task, max 2 dots in displayId, enforced in `createTask` (services/task.server.ts:60-88).

**Page** (schema.prisma:1304-1327) — the Scratchpad/task body, a Yjs collaborative doc: `type {Daily|Task}, date, descriptionBinary Bytes?, butlerLastSeen Bytes?` (the agent's diff baseline for proactive observation), `outlinks Json`. **ButlerComment** (1328-1349): agent comments anchored to selected text, `conversationId` links each to its agent thread.

**Conversation / ConversationHistory** (115-213): per-thread rows; history rows have `parts Json, context Json, thoughts Json, userType {Agent|User|System}`, `status String?` ("working"/"done"/"error"/"cancelled") + `asyncJobId` (row-level lifecycle for agent-authored turns), `delegationDepth Int` (bounded agent↔agent mention fan-out). Conversation has owning `agentId`, `source` (channel slug), `activeStreamId`.

**Memory** (the "temporal knowledge graph"): `IngestionQueue` (raw event, status PENDING→COMPLETED, `graphIds String[]` = Neo4j node ids), `EpisodeEmbedding` (chunked episode text + vector + `sessionId/version/chunkIndex/chunkHash` — versioned document re-ingestion), `StatementEmbedding` (fact triples), `EntityEmbedding`, `LabelEmbedding`, `CompactedSessionEmbedding` (summary + vector). **VoiceAspect** (826-855): user-preference facts with genuine temporal validity — `fact, aspect {Directive|Preference|Habit|Belief|Goal}, episodeUuids[], validAt DateTime, invalidAt DateTime?, invalidatedBy String?` — plus its embedding table.

**Agents** (54-88): multi-agent roster per workspace — `handle, displayName, basePrompt, capabilities String[], model, personality (tars|alfred|hobson|hudson|jeeves|custom), gatewayId?` (an agent can be backed by a gateway machine).

**Channel** (788-804): slack/telegram/whatsapp/email configs (bot tokens in `config Json`). **VoiceInboxMessage** (964-986): proactive outbound messages with `checked DateTime?` — unread rows power the voice "catchup". **Gateway** (470-512): registered machines (`encryptedSecurityKey` AES-256-GCM, status, lastSeenAt). **CodingSession** (1445-1480) / **BrowserSession** (1482-1510, implicit lock: one active task per `(gatewayId, profileName)`).

**SaaS baggage:** Subscription/BillingHistory/CreditTopup/UserUsage (Stripe credits; BYOK workspaces bypass), DailyTokenUsage (per-day token rollup keyed by operation "cacheKey"), OAuth2 provider stack (client/grant/installation — CORE acts as an OAuth server for third-party apps), InvitationCode, UserWorkspace roles. `RecallLog` (743-787) records every memory access (method, score, response time) — real observability.

## 4. Architecture & control flow (deterministic code vs LLM calls)

**Deterministic backbone:** Remix webapp; Hocuspocus WebSocket collab for Scratchpad; BullMQ/Trigger.dev queues; Prisma; gateway polling. All orchestration — task state machine, edit buffers, scheduling, retry, channel fan-out, approval routing — is plain TypeScript.

**LLM surface (all via `makeModelCall`/`makeStructuredModelCall` with per-op cache keys like "reflect-world", "recurrence-extraction"):**
1. **Memory ingestion pipeline** (`services/knowledgeGraph.server.ts:80-410`, Zep/Graphiti-style): normalize episode → *comprehend+classify* (`extract-voice`/`extract-world` + classify into aspects) → save triples to Neo4j + embeddings to pgvector → *reflect* prompts (reflect-voice/reflect-world) perform temporal invalidation of contradicted facts. Runs inside ingest queue workers.
2. **Recurrence extraction** (`services/tasks/recurrence.server.ts:20-50`): NL → `<output>{json}</output>` parsed RRule, temperature 0.1.
3. **Turn agent** (`services/agent/context.ts:105-860`, `buildAgentContext`): assembles system prompt = agent `basePrompt` (rendered with `{{VOICE}}/{{TIME}}/{{USER}}/{{PERSONA}}`) + identity block + connected integrations + gateway capability tags + channels + **skills list with "SKILL CHECK FIRST" rule** + current datetime + waiting tasks + task/trigger/scratchpad context blocks. The main agent owns every orchestrator tool directly (`createCoreTools`); gateway machines are Mastra sub-agents (`agent-gateway_<slug>`).
4. **Memory query generator** (`services/agent/memory.ts:44-100`): intent → 1-N search queries (parallel, PQueue); **search-v2 router** (`services/search-v2/router.ts`): structured call extracts query type/aspects/entities/temporal filters, then dispatches to hybrid vector+keyword+graph retrieval.
5. **Summarizer** (`services/summarize.server.ts`): voice catchup, "≤18 words per sentence, 2-4 sentences" spoken-cadence prompt.

**Context window strategy** (`services/agent/context-window.ts`): three modes — `full` / `compact+recent` / `budget-trim`, switching on history length; compaction summary is stored as a `Document` (unique `sessionId`) with a **coverage watermark** = max episode `validAt` folded in, bridging summary to verbatim tail. Long-running task threads stay coherent via: compaction + the task's own Page (description holds `<plan>/<outcome>/<log>` zones) + memory retrieval per turn.

## 5. Task lifecycle (what actually runs a task)

`createTask` (task.server.ts:60-140): a task created **Ready** gets `nextRunAt = now+2min` — the "editing buffer" (user veto window) — then a wake-up handler enqueues execution. Statuses: Todo (backlog) → Ready (buffered) → Working → Waiting (blocked on user via `send_message`; any channel reply matching it calls `unblock_task` and the system re-enqueues with the reply) → Review (agent done; **only the user marks Done**) → Done. Recurring tasks use RRule `schedule`, `occurrenceCount`, auto-rescheduling; on recurring runs the `<plan>` is frozen — background mode only allows `<log>` appends (and there's an open GitHub issue: "Cannot update recurring task plan/description"). Each task has a dedicated conversation (`Conversation.asyncJobId = taskId`) and a Task Page. Subtask completion auto-marks the parent Done. `reschedule_self(minutesFromNow)` is how an agent yields and retries later (used heavily for coding-session polling: gateway owns sleep/poll).

**Agent-prompt-encoded task rules** (context.ts `<task_execution>`): distinguish PLAN/RUNBOOK input vs GOAL input; decomposition only via the "Decompose Task" skill with a heads-up message ("Splitting into A, B, C — each starts in 2 min. Stop me if wrong") — buffer-as-veto again; strict rules against marking Done or cascading failures to siblings.

## 6. Scheduling & proactivity

- **Triggers** (`services/agent/decision-agent-pipeline.ts:1-30`): integration webhook, scheduled task, memory ingest, reminder → resolve/reuse per-(agent,source) conversation → `processInboundMessage` with `triggerContext`. Header comment (current code): "**There's no separate think stage and no `shouldMessage` gate; the agent owns delivery.**"
- **Observation vs runbook distinction** (context.ts trigger block): inbound webhooks/memory-ingest get "surfacing ≠ acting" rule (surface per Watch Rules; don't take irreversible action); user-authored scheduled tasks are "pre-authorized runbooks — execute as written, don't re-ask."
- **Watch Rules** (`services/skills.defaults.ts:35-90`): a user-editable markdown skill defining what to surface immediately vs handle silently vs write as "Live finds" suggestions into today's scratchpad. Proactivity policy as data, not code. Persona skill similarly editable.
- **Scratchpad scanning** (`services/collab-scanner.server.ts`): Hocuspocus store hook; @mentions → 10s-delayed job; proactive diffs vs `butlerLastSeen` → 20s-delayed job. **BUT the job processor's proactive branch is empty** (`jobs/scratchpad/scratchpad-scan.logic.ts:35-37`: `if mention {...} else { }`) — see §12.
- **Missed-job recovery**: `task-scheduler.ts` re-enqueues missed scheduled tasks on startup.
- **Morning Brief** (`services/morning-brief.ts`): recurring task whose HTML description is the prompt; **Gmail only** — GitHub/Calendar deliberately stubbed.
- **Voice catchup**: VoiceInboxMessage unread rows → summarized per the spoken-cadence prompt.

## 7. Integrations

~40 integration packages in `integrations/`, loaded as MCP tools (`IntegrationLoader.getConnectedIntegrationAccounts`); agent calls `get_integration_actions` / `execute_integration_action`. Ingestion flows: integration → `Activity` row / `IngestionQueue` → memory pipeline → may fire `memory_ingest` trigger → agent turn (surfacing decision). `IngestionRule` (free-text rules per source) filters ingestion. Webhooks out: `WebhookConfiguration`/`WebhookDeliveryLog` on `activity.created`. Integration calls are logged (`IntegrationCallLog`) with source (Claude-Code/Cursor/mcp/api). OAuth2 server built in so external apps (Claude, Cursor) can use CORE's connectors.

## 8. Security & approval model

- Human-in-the-loop is **status-machine-based**, not plan-approval-based: 2-min edit buffer on Ready; agent never marks Done (Review→Done is user's); Waiting gates on user input; background/trigger contexts run non-interactive.
- `requireApproval` (AI SDK suspension) on risky integration writes in interactive mode only (`services/agent/agents/orchestrator.ts:237-258`; `core.ts:83-110` — note: `ask_user` was "retired").
- Gateway: bearer `gwk_` keys, sha256 constant-time compare, no localhost bypass; encrypted per-gateway secret; browser sessions isolated per profile with task-locking.
- Sensitive-data replacer service for logs; tokens encrypted (AES-256-GCM) via `ENCRYPTION_KEY`. CASA Tier 2 claim on README. PATs hashed+encrypted.

## 9. UI & phone/remote access

211 routes in webapp (Remix): conversations, daily scratchpad, tasks, memory graph browser, people/contacts, integrations, agents/skills admin, coding sessions with inline terminal ("Court"), widgets/embeds. Phone access = **messaging channels (WhatsApp/Telegram/Slack/email) + responsive web**; no native mobile app. Mac: Tauri app (voice hotkey Ctrl+Option, macOS Accessibility screen-context snapshot of frontmost window, tray, scratchpad panel). Coding/browser work runs on the always-on gateway so "sessions keep running when your laptop is closed" (true if gateway is on a desktop/Docker host).

## 10. Strengths

1. **Memory architecture is real and sophisticated** — Zep-style reified temporal graph (episodes/statements/entities + voice aspects with validAt/invalidAt/invalidatedBy), hybrid retrieval, versioned document re-ingestion via content hashing/diffing, recall observability. Not README vapor.
2. **Task-as-first-class-object with dedicated thread, page, subtasks, RRule recurrence, waiting/unblock protocol, reschedule_self** — the strongest existing match for Executor's "task with surrounding context" requirement.
3. **Proactivity policy as user-editable markdown (Watch Rules/Persona skills)** + surfacing≠acting rule — exactly the safety shape Executor needs.
4. **Edit-buffer/veto windows** (2-min before execution, buffer on decomposition) instead of modal plan approvals — low-friction HITL.
5. Multi-agent roster with bounded delegation depth; gateway machines as sub-agents with capability tags; coding-session lifecycle (start/questions/plan/execute/resume) is well thought out.
6. Channel-agnostic identity (same memory from WhatsApp/web/voice); missed-schedule recovery; compaction with coverage watermark.
7. Active, fast-moving repo; real docs; OSS self-host path genuinely works (docker-compose with all services).

## 11. Weaknesses (for Executor's use case)

1. **Heavyweight**: Postgres+pgvector, Redis, Neo4j, webapp, workers, optional Ollama — 4 vCPU/8GB floor for one user. vs Executor's SQLite/Git-markdown instinct.
2. **SaaS-first**: credits/billing/Stripe, OAuth-server, invitations, marketing telemetry baked into core schema and flows.
3. **Work-oriented**: integrations skew dev/CRM (GitHub/Linear/Jira/HubSpot); no home-context sensing, no routines/body-data widgets wired to task selection (whoop/ynab integrations exist but are data sources only). Task model has no effort/energy/duration estimation, no postponement learning, no "what should I do right now" scheduler — tasks are delegated work units for the *agent*, not an executive-function surface for the *user*.
4. **Prompt-brittleness**: enormous hand-rolled system-prompt blocks (task rules, trigger flows) with internal contradictions after refactors (see §12); behavior lives in prose.
5. AGPL-3.0 + hosted-service coupling makes direct reuse legally/onerously awkward.
6. Single-shallow-commit code churn is violent (trigger.dev bridge added then dropped); 211 open issues; Windows setup broken; OpenAPI drift.

## 12. Surprising design decisions / unfinished-vaporware

- **`think` tool ghost**: `decision-agent-pipeline.ts` header says the think stage was removed ("no shouldMessage gate") yet `context.ts` `<trigger_context>` still instructs "Call `think` first. It returns an ActionPlan…" — stale prompt text shipped against removed code. Similarly `handleScratchpadStore` is `@deprecated`.
- **Scratchpad auto-pickup stubbed**: README/docs promise "`[ ]` becomes a task within ~2-3 minutes"; the proactive scan job's processor branch is literally empty (scratchpad-scan.logic.ts:35-37). Only explicit @butler mentions currently work from the scratchpad.
- **Docs/code plan-approval mismatch**: docs describe "drafts a plan, presents for approval"; code says "single execution mode — the plan/execute phase toggle is gone"; approval is the 2-min buffer + Review status + risky-write requireApproval.
- **Recurring plan freeze** causes a user-facing bug (open issue: cannot edit recurring task description).
- Morning Brief is Gmail-only despite the "pull from email, GitHub, and Slack" docs example.
- Benchmark claim (88.24% LoCoMo) has an open issue: grader defaults to WRONG on ambiguous output and README figures don't reproduce.
- Neo4j as a *required* third datastore for one user's facts (graph providers: neo4j|falkordb|helix — packages/providers/src/types.ts:75); embeddings separately in pgvector.
- `Task.conversationIds String[]` denormalized alongside `Conversation.asyncJobId`.

## 13. Answers to the specific questions

- **How does a task acquire context?** At turn time `buildAgentContext` assembles: agent basePrompt+persona+voice, task's own Page HTML (spec/plan/log zones), parent-task page for subtasks, skills list (SKILL CHECK FIRST), connected integrations, gateway capabilities, waiting tasks, plus memory retrieval on demand (memory.ts query-generator → search-v2 hybrid). Not full history: compaction (compact+recent, watermark) + per-turn retrieval.
- **Agent threads?** Conversation rows per (agent, source); ConversationHistory rows with working/done lifecycle and asyncJobId; background turns as BullMQ jobs; agent↔agent via mentions with delegationDepth cap; supersede/cancel on fresh mention.
- **Live/current state?** Task.status/nextRunAt/result/error + Page content + VoiceInboxMessage + Activity; gateway heartbeats (`lastSeenAt`); butler activity states (watching/thinking/acting) derived from running conversations.
- **Autonomous cadence/approval?** RRule-scheduled tasks, webhook/memory-ingest triggers, reminders; approval = edit buffer, Waiting gate, Review-before-Done, requireApproval on risky writes (interactive only); user-editable Watch Rules decide surfacing.
- **External state flow?** integration → Activity/IngestionQueue → (optional IngestionRule filter) → episode → KG + embeddings → optional trigger → agent turn → send_message/scratchpad suggestion.
- **Weeks-long coherence?** Compacted session docs + watermark, KG facts with temporal invalidation, task Page as durable spec, recurring `<log>` accumulation. Genuine, though heavy.
- **Temporal KR real?** Yes — real code (VoiceAspect validity intervals; reflect prompts invalidating statements; versioned episodes), not aspiration. But it needs Neo4j-class infra.
- **Borrow:** task Page-as-spec with `<plan>/<outcome>/<log>` zones; 2-min veto buffers; Waiting/unblock protocol; reschedule_self; Watch-Rules-as-markdown policy; surfacing≠acting; memory query decomposition; compaction watermark; gateway capability manifest + browser profile locking; RecallLog observability; task displayId hierarchy with depth cap.
- **Avoid:** Neo4j+pgvector+Redis+Trigger.dev sprawl for one user; credits/Stripe/OAuth-server in core; prompt-encoded state-machine rules; AGPL entanglement; status-as-prose (`Recurring` as a TaskStatus); trusting docs over schema.

## 14. VERDICT for Executor

**Pattern-donor, not a foundation. (Reuse concepts/schema shapes; do not fork, do not build on.)**

Reasons: (1) AGPL-3.0 + deep SaaS coupling (billing, OAuth server, workspaces) make extraction costlier than reimplementation of the ~15% Executor needs; (2) its center of gravity is *delegating knowledge work to agents*, while Executor's is *helping the user initiate their own life-tasks* — CORE has no "what should I do right now" prioritizer, no effort/postponement learning, no routine/body context; (3) operational weight (4 services, 8GB RAM) is wrong for a home always-on box vs Executor's SQLite/Git instinct. However, CORE is the single best reference implementation to steal design vocabulary from: its Task+Page+thread triad, edit-buffer/veto HITL, Waiting/unblock protocol, Watch-Rules policy-as-markdown, temporal voice-aspect store, and compaction watermark all map directly onto Executor's needs and can be re-implemented over SQLite with Markdown mirroring. Treat its code as an oracle for "what breaks at week 3 of an autonomous life-agent" (recurring-plan freeze, stale prompts, trigger churn) — each of those bugs is a lesson for Executor's simpler, inspectable core.
