# personal-assistant (beranradek/personal-assistant)

**Target repo:** `/home/dustin/src/personal-assistant-br` (read-only clone of github.com/beranradek/personal-assistant)
**Analyst brief:** Executor architectural reconnaissance — external executive-function layer for a user with severe ADHD.

---

## 1. Overview

A **TypeScript daemon that wraps an off-the-shelf agent CLI** (Claude Code via `@anthropic-ai/claude-agent-sdk`, or OpenAI Codex via `@openai/codex-sdk`) as the "brain", and adds everything around it: messaging adapters (Telegram, Slack, GitHub webhook), a periodic **heartbeat** proactivity scheduler, a **markdown-workspace memory** with local hybrid vector search, **episodic memory** in SQLite, agent-managed **cron jobs**, background process exec, a separately-forked **integrations API** (Gmail/Calendar/Slack) that isolates OAuth secrets from the agent, and multi-layer security hooks.

It runs as `pa terminal` (interactive REPL) or `pa daemon` (headless, systemd user service). Phone access = Telegram bot (with voice STT/TTS), not a web UI.

**Honest answer to "is this more than a thin wrapper?":** Yes, substantially. ~24k LOC of non-test TypeScript (111 source files), 84 test files, a retrieval eval suite with a CI gate (`.github/workflows/episode-eval.yml`), OS-level watchdogs, and VPS resource caps. The *agent intelligence* is delegated to Claude Code/Codex, but the harness — memory, proactivity, scheduling, security, integrations — is real, tested infrastructure. However, the **life-management layer Executor needs barely exists**: there is no task decomposition, no project/task schema, no prioritization engine, no "what should I do right now" planner. The closest thing is heartbeat prose instructing the agent to "proactively work your jobs" and a morning "review goals and suggest actions".

## 2. Tech stack & repo stats

- **Language:** TypeScript (ESM), Node.js ≥22. No linter. Vitest, 70% coverage threshold.
- **Runtime deps** (`package.json`): `@anthropic-ai/claude-agent-sdk ^0.3.227`, `@openai/codex-sdk ^0.147.0`, `@modelcontextprotocol/sdk`, `better-sqlite3`, `sqlite-vec`, `node-llama-cpp` (local embeddings), `node-cron`, `cron-parser`, `grammy` (Telegram), `@slack/bolt`, `pino`, `zod`.
- **Stats:** 111 non-test TS files (~23,902 LOC), 84 test files. Single squashed commit in this clone (`e36930a`, 2026-10-04, "feat: report specific failure causes..."). GitHub: created 2026-02-13, pushed 2026-10-04 (clone day — **actively maintained**), 9 stars, 1 fork, 0 open issues. Author: Radek Beran (single maintainer). MIT license.
- `CLAUDE.md` present (dev guidance for Claude Code); `.autocode/` and `docs/plans/` (spec/plan workflow with completed/active/backlog dirs) show heavy AI-assisted development. The backlog spec (`docs/plans/backlog/integration-api/spec.md`) explicitly benchmarks against **OpenClaw**'s integration approach.

## 3. ACTUAL data model

### Workspace markdown (the agent's source of truth) — seeded from `src/templates/` on first run (`src/core/workspace.ts:18`)
- `AGENTS.md` — behavior rules + tool guidance (English)
- `SOUL.md` — personality ("Assistant": anticipate needs, have opinions)
- `USER.md` — user profile/preferences (name, timezone, language, notification style)
- `MEMORY.md` — long-term memory (Decisions/Preferences sections; indexed for search)
- `HEARTBEAT.md` — periodic-check instructions (**default template is Czech**; English `cs/` variants exist alongside)
- `HEARTBEAT_MORNING.md` / `HEARTBEAT_EVENING.md` — extra morning/evening prompts (Czech: morning = review goals from USER.md, suggest actions, check server RAM/disk; evening = lessons-learned memory curation, cleanup temp files/processes)
- `HABITS.md` — YAML frontmatter pillars + daily checklist + history, e.g.:
  ```yaml
  pillars:
    - id: code-commit
      label: "Code Commit"
      type: auto
      detection_command: "git log --since=today --oneline"
    - id: exercise
      label: "Exercise"
      type: manual
  ```
- `memory/` — agent-created topic files, `memory/reflection-YYYY-MM-DD.md` (AI-generated daily/weekly reflections with YAML frontmatter)
- `daily/YYYY-MM-DD.jsonl` — audit log (every turn, tool call, error). Retention 90 days.
- `.claude/skills/` — agent-authored skill files (e.g. `skills/integrations/SKILL.md` documenting `pa integapi` commands)

System prompt = `preset: "claude_code"` + append of concatenated `AGENTS.md + SOUL.md + USER.md + MEMORY.md` (joined `\n\n---\n\n`; `src/memory/files.ts:6-7`). Heartbeat file added to daemon turns.

### vectors.db (SQLite + sqlite-vec) — `src/memory/vector-store.ts`
`chunks(id, path, text, embedding, startLine, endLine)` + file-hash table for incremental reindex. Hybrid search = cosine similarity + BM25 keyword, weights 0.7/0.3 default, minScore 0.35 (soft), recency boost `0.1 * 0.5^(daysAgo/7)` (`src/memory/hybrid-search.ts`). Embeddings: **local** EmbeddingGemma-300M GGUF Q8 via node-llama-cpp (~314 MB, CPU, `gpuLayers: 0`; `src/memory/embeddings.ts:25`). No API dependency, no per-token cost.

### episodes.db (SQLite, schema v2) — `src/memory/episodes/schema.ts`
```sql
CREATE TABLE episodes (
  id TEXT PRIMARY KEY, started_at TEXT, ended_at TEXT, source TEXT,
  session_key TEXT, session_id TEXT, initiator TEXT,
  action TEXT, normalized_action TEXT, summary TEXT, why TEXT,
  project_name TEXT, job_name TEXT, issue_id TEXT, pull_request_id TEXT,
  detailed_memory_file TEXT, category TEXT,
  outcome TEXT /* success|partial_success|failure|aborted */,
  success_score REAL /* 1.0/0.6/0.2/0.0 */, model TEXT,
  input_tokens INTEGER, output_tokens INTEGER, location TEXT,
  semantic_embedding_text TEXT NOT NULL
)
```
Plus 9 relational tables: `episode_skills`, `episode_tools`, `episode_tags`, `episode_blockers`, `episode_errors`, `episode_open_questions`, `episode_related` (links to prior episodes), `episode_steps(position, at, kind, label, data_json)` — a full trajectory record. 11 indexes. Episodes immutable; written by agent via `episode_write` MCP tool "at meaningful task boundaries, not every turn". Agent queries via `episode_search` (keyword or `semantic: true`), `episode_recent`, `episode_stats`. Backed by an eval harness (`episode-eval` CLI + CI gate with fixtures asserting retrieval modes: exact_episodic / semantic_episodic / semantic_markdown / raw_audit_fallback).

### cron-jobs.json — `src/core/types.ts:389`
```ts
CronJob = { id, label, schedule: CronSchedule, payload: { text }, createdAt, lastFiredAt, enabled }
CronSchedule = { type: "cron", expression, timezone? } | { type: "oneshot", iso } | { type: "interval", everyMs }
```
CRUD exposed to the agent as MCP tools (`cron_list/create/update/remove`). Timer arms only the single nearest job (`src/cron/timer.ts:56`); firing enqueues a system event, delivered into the next heartbeat prompt.

### data/sessions/<source>--<sourceId>[--threadId].jsonl
Session transcripts: `{role: user|assistant|tool_use|tool_result|compaction, content, timestamp, toolName?, error?}`. SDK session-resume IDs cached in an in-memory `TtlMap` (1-day TTL).

### heartbeat-state.json
Snapshot + notifiedItems for **state diffing** — the heartbeat prompt only includes "New: …; Resolved: …" deltas since last run (`src/heartbeat/prompts.ts:87`).

## 4. Architecture & control flow

```
Adapters (Telegram/Slack/GitHub-webhook/Terminal/Heartbeat/Cron-events)
  → Gateway FIFO queue (serialized, maxQueueSize 20, rate limiter)
  → processLoop → backend.runTurn (Claude SDK query() / Codex exec subprocess)
  → Router → originating adapter.sendResponse()
```

- **Daemon startup order** (`src/daemon.ts`): config (Zod-validated) → workspace seeding → memory services (embedder/vector store/indexer + file watcher + 10-min reindex timer) → MCP servers (memory + assistant) → backend → queue/router → adapters → heartbeat scheduler → daily/weekly reflection crons → cron timer → processing loop → graceful shutdown (10s force-exit, drains in-flight sync).
- **Daemon mode = one agent, serial turns.** No concurrency. Telegram/Slack get streaming "processing" messages updated every 5s while a turn runs.
- **Heartbeat cycle** (every `intervalMinutes` within `activeHours` "8-21"): optional `git pull` of workspace → drain system events (exec completions, fired crons) → prompt = base event prompt + state diff + morning/evening template (`{{DAILY_LOG}}` substitution) + yesterday's reflection + habit status (auto-detection runs, evening nudge lists unchecked habits) → enqueued → agent turn → if response matches `/HEARTBEAT_OK/i`, **the queue suppresses delivery to the user entirely** (`src/gateway/queue.ts` uses `isHeartbeatOk`). Deferred `git push` after 60s. This is the proactivity loop: a deterministic cron that hands a composed context packet to the LLM, with an LLM-output gate suppressing noise.
- **Heartbeat template instructs the agent to autonomously progress "jobs"**: read job specs from the filesystem, continue them without user decision, record progress in `memory/<job-title>.md`, report blockers only once, write a handoff summary before session end. Tasks are prose files — no schema, no scheduler beyond this prompt.

## 5. Deterministic vs LLM boundary (explicit)

**Deterministic application code:** queue serialization/rate limiting/workload guard; heartbeat timing + active-hours; prompt assembly (diffing, morning/evening, habits, reflection injection); habits auto-detection (whitelisted binaries `git,wc,ls,cat,grep,date,stat` run via `execFile`, `src/heartbeat/habits.ts:24`); cron scheduling (cron-parser, nearest-job timer); Gmail unread categorization (rule-based sender/subject/List-Unsubscribe patterns, Czech+English, `categorizeEmail` at `src/integ-api/integrations/gmail/unreads.ts:420` — no LLM); security hooks; memory chunking/indexing; reflection *scheduling*; session compaction *triggering*.

**LLM calls:** (1) the agent turn itself — Claude Code or Codex CLI subprocess with full preset toolbelt (Read/Write/Edit/Glob/Grep/Bash/WebFetch/WebSearch + MCP), `maxTurns: 200`; (2) daily/weekly reflection summarization — direct Anthropic API call with `REFLECTION_PROMPT.md` ("You are a personal memory curator… extract Decisions / Lessons Learned / Facts / Project Updates; IGNORE routine tool calls; output `(nothing to extract)` if empty"); (3) session compaction summarization (Anthropic API); (4) STT/TTS (whisper-1 / gpt-4o-mini-tts) for Telegram voice.

## 6. Memory/persistence summary

Five layers (documented in `docs/agent-memory.md`, code matches):
1. **Semantic** — workspace `.md` files, written by agent with plain file tools (no wrapper), re-embedded on watcher fire. Indefinite TTL, manual consolidation.
2. **Episodic** — SQLite episodes.db (above), agent-written at task boundaries, immutable, linked.
3. **Daily audit log** — JSONL per day, 90-day retention, feeds reflections.
4. **Conversation history** — JSONL per session key; SDK resume + app-level compaction every `maxHistoryMessages/2` turns (summarize → persist compaction entry → reset SDK session → inject summary into next system prompt); pre-compaction "flush key context to daily log".
5. **Reflections** — daily 07:00 + weekly Mon 07:05 cron → LLM summary → `memory/reflection-*.md`, indexed; daily files retained 21 days. Yesterday's reflection is injected into morning heartbeats.

Plus **Git as sync/backup**: heartbeat template mandates pull-before / commit-and-push-after; `heartbeat.gitSync` config with 60s-deferred push (`src/heartbeat/git-sync.ts`). The workspace is a Git repo mirroring to a remote — multi-device via clone, human-inspectable history.

## 7. Scheduling/proactivity

- Heartbeat: `*/N * * * *` within active hours, overnight ranges supported ("22-6").
- Agent-authored cron jobs (persistent JSON, 3 schedule types) — reminders surface as heartbeat events, not push notifications.
- Reflection crons (node-cron).
- Background `exec` tool: spawned processes registered in-memory; completion enqueues a system event handled at next heartbeat.
- Nothing else fires the LLM — no event-driven triggers from Gmail/Calendar; integrations are pull-only, on-demand via the skill.

## 8. Integrations

- **integ-api**: separate forked child process (`src/daemon.ts:395` spawns `pa integapi serve` with restart backoff) exposing localhost HTTP: Calendar (today/week/range/event/RSVP), Gmail (unreads categorized, list, read, labels), Slack (unreads, channel messages). **Design rationale** (backlog spec): secrets (OAuth tokens) live only in the proxy; the AI layer never sees them; guardrails/filtering/redaction at the proxy. Outbound rate limiters per service; inbound rate limiter middleware; audit middleware.
- Agent accesses it via `pa integapi …` CLI documented in `skills/integrations/SKILL.md` (taught as a skill, includes usage discipline like "prefer compact-json").
- GitHub webhook adapter: issues → agent messages with `AuditTaskContext{projectName, jobName, issueId}` derived from repo/issue metadata.
- No todo-app, no phone sensors, no location, no home automation.

## 9. Security/approval model

Three layers **for the Claude backend** (`src/daemon.ts` logs this explicitly): (1) SDK sandbox (`sandbox: {enabled: true, autoAllowBashIfSandboxed: true}`); (2) PreToolUse hooks — `bashSecurityHook` (command allowlist from settings — ~50 commands incl. `rm/kill/curl` needing extra validation; sudo denied; path extraction from pipes/chains) + `fileToolSecurityHook` for Read/Write/Edit/Glob/Grep (path validation vs workspace + additional dirs); (3) `script-content-scanner` (blocks `curl | bash`, stdin execution, missing script files, scans inline `bash -c`, 200KB cap — tokenized pattern matching for dangerous idioms). **With the Codex backend, PA's hooks are NOT used** — security is delegated to Codex CLI sandbox/approval policy (stated in daemon log line). The agent cannot modify its own source/config (workspace confinement). Telegram restricted to `allowedUserIds`.

## 10. UI & phone/remote access

- **Telegram** (grammy; polling or webhook; `allowedUserIds` gate; voice messages → whisper → reply via TTS; `/clear` resets session). **Slack** (socket mode, per-user/channel/thread session keys, streaming "processing" edits). **Terminal** (local REPL with markdown rendering, paste handling, spinner).
- No web UI, no native app. For Executor's phone requirement this is the proven pattern: the chat network *is* the mobile client.

## 11. Strengths

1. **Heartbeat + HEARTBEAT_OK suppression** — the cleanest pattern found yet for LLM proactivity without notification spam: deterministic cadence, layered context injection, output-gated delivery.
2. **Workspace-as-memory in Git** — exactly the Markdown+Git approach Executor is considering, validated in production for ~8 months. Human-editable, versioned, synced, agent-owned.
3. **Episodic memory schema** — the richest structured "what happened / how did it go / what was learned" model seen; outcome + success_score + trajectory + blockers + related-episode links. Directly relevant to Executor's "learn from actual behavior" requirement.
4. **Local-first hybrid search** (sqlite-vec + BM25 + recency boost, CPU embeddings) — zero marginal cost, private, fast; sensible soft-fallbacks.
5. **Secrets isolation via integ-api proxy** — correct security architecture for personal data integrations.
6. **Reflection pipeline** — automated daily/weekly curation of audit logs into durable memory; morning re-injection closes the loop.
7. **Operational seriousness**: watchdog timers, resource caps, degraded-mode startup probes, restart backoff, eval-gated CI, 84 test files, prompt logging + redaction.
8. Habits: auto-detectable pillars via whitelisted shell commands + manual pillars + evening nudge — a working micro-habit loop.

## 12. Weaknesses (for Executor's purpose)

1. **No task decomposition or planning engine.** Nothing decomposes "clean the basement" into micro-steps. The "jobs" concept is prose in HEARTBEAT.md; no schema, no dependencies, no estimates, no prioritization, no "what should I do right now" answer.
2. **No behavioral learning loop.** Nothing tracks postponement, underestimation, initiation failure, or good/bad contexts. Episodes record task outcomes, but nothing consumes them for scheduling or nudging. `episode_stats` is for the agent's own recall, not user modeling.
3. **Pull-only integrations.** Calendar/Gmail never trigger anything; no daily-plan generation from calendar+tasks; the morning heartbeat merely *suggests* the agent "check goals".
4. **Single-agent serial queue** — one turn at a time globally; heartbeat competes with user messages.
5. **In-memory session→SDK-ID map (1-day TTL)** means daemon restarts can lose conversation resumption (transcripts survive; compaction summaries reload).
6. **Mixed-language templates by default** (Czech HEARTBEAT*, English SOUL/USER/MEMORY) — personal dogfood artifacts shipped as defaults; would need rewriting for Executor's user.
7. **Bus factor 1**, 9 stars, squashed public history (this clone shows 1 commit; GitHub shows activity but the local history is opaque), no community.
8. **Cost/ToS friction**: README carries Anthropic's statement banning subscription limits for third-party harnesses — always-on use requires paid API or extra-usage enablement. (Codex backend is the escape hatch.)

## 13. Surprising design decisions

- The heartbeat template **mandates the agent git-commit and push its own memory** each cycle, with conflict resolution instructions and secret-leak warnings.
- HEARTBEAT.md default says checks run "every whole hour" while settings default is 30 minutes (README/code) — template prose vs config drift.
- The evening heartbeat doubles as a **self-maintenance slot**: curate memory, fix wrong assumptions, create/update skills, clean temp files and stray processes.
- Morning heartbeat includes **server health checks** (RAM thresholds 300MB/150MB, swap 70%) — dogfooded on a Hetzner CAX11 VPS.
- A 314 MB local embedding model loaded per MCP child process was a real production bug — solved by a shared HTTP MCP server for Codex subagents and a systemd watchdog that reaps wedged `codex exec` trees from `/proc`.
- Templates: `cs/` directory exists but the *default* English-named files are already partly Czech — localization is tangled.
- `docs/plans/` shows a spec→plan→completed workflow (dated like 2026-02-13-core-implementation) — the repo develops itself with Claude Code assistance.

## 14. Unfinished / vaporware

- Nothing overtly dead: no TODOs/FIXMEs in src, tests match implementation closely, 0 open GitHub issues.
- Active plan `docs/plans/active/assistant-improvements/` is nearly empty ("Spec itself is sufficiently detailed. No plan needed.") — thin.
- Habits have `historySection` parsed but history is written by the agent manually; no analytics over habit history.
- No usage of `episode_stats`/success_score for anything behavioral — data collected, never exploited.

## 15. VERDICT for Executor

**Pattern-donor (strong) — steal the harness patterns; do not adopt as the base, and the life-management layer must be built fresh.**

Reasons:
- The candidate architecture Executor is considering (custom life layer + SQLite + Git/Markdown on an agent runtime) is **exactly what this repo is**, minus the life layer. It validates feasibility and supplies battle-tested answers for the hard plumbing: heartbeat cadence + suppression gate, layered proactive prompt assembly, markdown-in-git memory with local hybrid retrieval, episodic SQLite schema, secrets-isolated integration proxy, security hooks, daemon supervision.
- But its agent brain is Claude Code/Codex with a *coding* preset toolbelt and its behavior is prose-prompted; Executor needs a first-class task/project data model, decomposition, prioritization, and behavioral analytics that simply aren't here. Building them inside this fork means maintaining a 24k-LOC single-maintainer codebase whose author's priorities (GitHub issue automation, VPS ops) diverge from Executor's.
- Concretely worth stealing (with sources): heartbeat scheduler + `isHeartbeatOk` gate (`src/heartbeat/scheduler.ts`, `src/gateway/queue.ts`); diff-aware prompt builder (`src/heartbeat/prompts.ts`); habits module (`src/heartbeat/habits.ts`); episodes schema (`src/memory/episodes/schema.ts`); hybrid search config & recency boost (`src/memory/hybrid-search.ts`); reflection pipeline (`src/memory/daily-reflection.ts` + templates); integ-api secret isolation (`src/integ-api/`); bash/path/script-content hooks (`src/security/`); Telegram voice adapter pattern (`src/adapters/telegram.ts`); session compaction with summary injection (`src/core/agent-runner.ts:207`).
- If a fork path were chosen anyway, the honest blocker is the brain: an ADHD executive-function layer wants tight control over task tools and planning prompts, not a preset Claude-Code session. Fork only the daemon skeleton; expect to rewrite templates (Czech→English) and replace the jobs-prose with a real task store.

**Key lesson for Executor's architecture decision:** this repo proves the "Git/Markdown + SQLite around an agent runtime" substrate works long-term for an always-on personal assistant — and proves that the *substrate is the easy 80%*. The differentiating work (task decomposition, prioritization, behavioral learning) is exactly the part nobody has built yet.
