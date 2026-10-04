# Hermes-Agent — Architectural Reconnaissance for Executor

Target: `/home/dustin/src/hermes-agent` (github.com/NousResearch/hermes-agent), shallow clone at commit `639eee30` (2026-10-04, by teknium1). Read-only inspection; all claims cite files in that tree.

## 1. Overview

Hermes is Nous Research's "self-improving AI agent": a Python 3.14 personal agent with a CLI/TUI, a multi-platform messaging gateway daemon, cron scheduling, subagent delegation, an autonomous task board (kanban), and — its distinguishing feature — a closed learning loop that writes skills and memory entries about its own experience. It bills itself as the successor to OpenClaw and ships `hermes claw migrate` to import OpenClaw settings, memories, skills, and API keys (README.md:201+). Estop was "ported from gastownhall/gastown (MIT)" (agent/estop.py:15) — the OpenClaw lineage is explicit in code comments.

## 2. Tech stack & repo stats

- **Languages**: Python core (~150+ top-level modules) + TypeScript workspaces (TUI `ui-tui/`, web dashboard `web/`, Electron desktop `apps/desktop/`). Node ≥22 and Python 3.14 both required; `uv` manages Python deps, PM manages external tools (ripgrep, ffmpeg, git-for-Windows).
- **GitHub** (API, 2026-10-04): 251,083 stars, 53,866 forks, 47,914 open issues+PRs, created 2025-07-22, pushed within the hour. Commits land every few minutes (plugin-catalog pinning churn in the latest 10). Issue numbers run to #132k — enormous volume. Local clone is shallow (depth 1) so history depth was taken from the API.
- **Tests**: 5,653 Python test files under `tests/` — the maintenance rigor is real, and code comments cite issue numbers for nearly every non-obvious decision (e.g. `#94691`, `#68858`, `#114271`).
- **Size pathology**: `hermes_state_messages.py` 128KB, `run_agent.py` 88KB, `cli.py` 80KB. Their own tracker has "Repo-wide godfile eradication" (issue #78647) — they know.
- **Deployment**: Docker/compose, systemd notify support (gateway/systemd_notify.py), Nix flake, Windows native + Termux (aarch64 APT repo).

## 3. Actual data model

### 3.1 state.db — sessions & messages (hermes_state_common.py:373–630, `SCHEMA_SQL`)

11 tables (fresh-install shape; verbatim key fields):

```sql
CREATE TABLE sessions (
  id TEXT PRIMARY KEY, source TEXT NOT NULL,          -- 'telegram','cli','cron','subagent','kanban','tool',...
  user_id TEXT, session_key TEXT, chat_id TEXT, chat_type TEXT, thread_id TEXT,
  display_name TEXT, model TEXT, model_config TEXT, system_prompt TEXT,
  parent_session_id TEXT, started_at REAL, ended_at REAL, end_reason TEXT,
  message_count INTEGER, tool_call_count INTEGER, input_tokens/output_tokens/
  cache_read_tokens/cache_write_tokens/reasoning_tokens INTEGER,
  cwd TEXT, git_branch TEXT, git_repo_root TEXT,
  billing_provider TEXT, estimated_cost_usd REAL, actual_cost_usd REAL,
  title TEXT, title_source TEXT, last_activity_at REAL, last_activity_description TEXT,
  handoff_state TEXT, handoff_platform TEXT,          -- cross-platform conversation handoff
  compression_failure_cooldown_until REAL, ..., profile_name TEXT,
  rewind_count INTEGER, archived INTEGER, pinned INTEGER, hidden INTEGER, ...
);
CREATE TABLE messages (
  id INTEGER PRIMARY KEY AUTOINCREMENT, session_id TEXT NOT NULL,
  role TEXT, content TEXT, tool_call_id TEXT, tool_calls TEXT, tool_name TEXT,
  effect_disposition TEXT, timestamp REAL, token_count INTEGER, finish_reason TEXT,
  reasoning TEXT, reasoning_content TEXT, reasoning_details TEXT,
  platform_message_id TEXT, observed INTEGER,
  _compressed_summary INTEGER, active INTEGER, compacted INTEGER,
  api_content TEXT, display_kind TEXT, display_metadata TEXT, ...
);
```

Plus: `system_prompts` (hash-addressed prompt blobs), `session_model_usage` (per-session/model/task token+cost accounting), `state_meta` (KV), `gateway_routing` (scope+session_key → routing JSON), `gateway_hygiene_state`, `conversation_generations` (monotonic per-peer counter, deliberately never GC'd to close an ABA reuse bug — long comment at hermes_state_common.py:559+), `gateway_heartbeats` (per-backend liveness), `compression_locks`, `session_turn_leases` (one turn per conversation), `async_delegations` (subagent results + at-least-once delivery queue with claim/retry columns).

### 3.2 FTS5 search (hermes_state_common.py:785–1030, hermes_state_fts.py)

- `messages_fts` — external-content FTS5 over `messages(id)`, columns `content, tool_name, tool_calls`; tool-role content truncated to a prefix. Synced by insert/delete/update **triggers** gated on `fts_rebuild_*` watermarks so background rebuilds never double-index.
- `messages_fts_trigram` — trigram tokenizer for CJK/substring search; **excludes** `source IN ('cron','subagent')` sessions and all `role='tool'` rows (trigram index ≈2.6× text size; cron+subagent transcripts were ~70–90% of bytes — comments cite #19434 "recall blindness").
- `messages_fts_cjk` (hermes_state_fts.py:46) + stale-index sentinels (`fts_stale`, `fts_cjk_stale`) and a full rebuild/optimize path (`hermes sessions optimize-storage`).
- Agent-facing surface: `tools/session_search_tool.py` — discovery (FTS BM25, cron demoted, lineage dedup, 300-row scan), scroll-around-message, read, browse. Explicitly "No LLM calls — every shape returns actual DB messages."

### 3.3 Memory: MEMORY.md + USER.md (tools/memory_tool.py, memory_tool_store.py)

- Two §-delimited flat files in `~/.hermes/memories/`: **MEMORY.md** = environment facts (tool quirks, conventions), **USER.md** = who the user is. Default hard char caps: **2200** memory / **1375** user (memory_tool.py:64–66).
- Loaded as a **frozen snapshot at session start**; mid-session writes hit disk but never mutate the live prompt (prefix-cache intact).
- Single `memory` tool: add/replace/remove/batch; whole-entry replace semantics; fcntl/msvcrt-locked store; byte-identical dedup on every mutation.
- Memory writes are **approval-gated** with "matched-entry pinning": the full current entry is staged so the approver reviews exactly what will change, and a concurrent edit invalidates approval (memory_tool.py:52–110).

### 3.4 Skills (procedural memory)

- Layout: `~/.hermes/skills/<category>/<name>/SKILL.md` (+ `references/`, support files), frontmatter with `related_skills`, usage stats in `skills/.usage.json`. Compatible with agentskills.io standard; Skills Hub can install from GitHub/ClawhHub (`tools/skills_hub*.py`).
- `skill_manage` tool (tools/skill_manager_tool.py): create/edit/**patch**(string replace)/delete/write-file, guarded writes, lint, per-skill mutation locks; `_security_scan_skill` on creation.
- **Audit ledger** (tools/skill_ledger.py): every mutation by any actor (curator/agent/user) appends JSONL to `~/.hermes/skills/.curator_ledger.jsonl` with content-addressed (sha256-deduped) before/after blobs under `~/.hermes/.curator_backups/blobs/` — durable, greppable, survives DB resets, supports single-edit rollback.

### 3.5 kanban.db (hermes_cli/kanban_db.py:947+)

A full task-execution substrate: `tasks` (id, title, body, assignee, status, priority, workspace_kind, workspace_path, branch_name, project_id, claim_lock/claim_expires, consecutive_failures, worker_pid + restart-stable fingerprint, max_runtime_seconds, last_heartbeat_at, current_run_id, workflow_template_id, skills JSON, model_override), plus `task_links`, `task_comments`, `task_events`, `task_runs`, `task_attachments`, `kanban_notify_subs`. Oriented to agent-executed work (git worktrees, dispatch workers, CAS on run ownership, circuit breaker after N consecutive failures) — not a user-facing life planner, but the durability machinery (locks, heartbeats, failure counters) is directly relevant to Executor.

### 3.6 Other persistent state

`~/.hermes/cron/` (jobs, notepad, occurrences, incidents, quota_hold modules), `skills/.curator_state` (JSON), `ESTOP` sentinel, profiles/multiplex homes via `HERMES_HOME` (hermes_constants.py:111).

## 4. Architecture & control flow

- **AIAgent** (run_agent.py:241) is the conversation engine: turn loop split across `agent/turn_*.py` (~40 modules: preflight, api request, tool round, overflow, finalizer…). LLM calls go through transports (`agent/transports/`: anthropic, chat_completions/OpenAI-compatible, bedrock, codex, codex_app_server) — provider adapters at `agent/*_adapter.py` (anthropic, bedrock, gemini_native, vertex, codex). Everything else — state, scheduling, FTS, approvals, delivery queues — is deterministic application code.
- **Process model**: `hermes` = interactive CLI/TUI; `hermes gateway` = long-running daemon (`gateway/run.py`) hosting all platform adapters, the cron ticker (60s, file-locked single-tick), turn leases, delivery ledgers, watchdogs (shutdown_watchdog, restart_loop_guard, memory_monitor, cgroup_cleanup), systemd notify. API server platform (`gateway/platforms/api_server.py`) exposes OpenAI-compatible routes + runs/rooms.
- **Subagents** (tools/delegate_tool.py + 8 sibling modules): child AIAgent with fresh conversation, own task_id, parent toolsets minus child-blocked tools; parent sees only the summary. Batch/parallel mode, `max_spawn_depth` default 1 (max 3), orchestrator role, optional git worktree isolation, steer/interrupt, async mode backed by the `async_delegations` table (at-least-once delivery).
- **Goals — "the Ralph loop"** (hermes_cli/goals.py): persistent free-form objective; after each turn an **auxiliary-model judge** decides satisfied/continue (fail-open on judge errors, turn budget as backstop, parse-failure and transport-failure auto-pause). Quality gates = deterministic shell commands whose failed output IS the continuation prompt.

## 5. The learning loop (the differentiator)

1. **After every turn** (`agent/background_review.py`): a daemon thread forks an AIAgent replaying the conversation snapshot and asks "should any skill/memory be saved or updated?" Writes go straight to the stores; the main conversation/prompt cache is never touched; the fork inherits provider/credentials so it hits the same prefix cache. Prompts are class attributes (`_MEMORY_REVIEW_PROMPT`, `_SKILL_REVIEW_PROMPT`, `_COMBINED_REVIEW_PROMPT`, lines 357–526) and are extraordinarily battle-tested:
   - Routing block: USER.md = who the user is; MEMORY.md = environment facts; "One fact goes to ONE store, never both" (fixes #30220 double-write bloat).
   - Skill shape contract: "Procedure first… a pitfall is a generalizable rule + one clause of WHY… The same lesson learned twice is ONE rule… Fix the skill in place when it is wrong: edit the sentence that misled, do not append 'UPDATE: actually…'".
   - Do-not-capture block: no environment-dependent failures, no negative claims about tools ("harden into refusals the agent cites against itself for months"), no unresolved-failure write-ups.
2. **Curator** (`agent/curator.py`): inactivity-triggered (default: 7-day interval, 2h min idle, no daemon); auto-transitions lifecycle states from usage timestamps (stale after 14d, archive after 30d); optional LLM consolidation fork (**opt-in, default False**); **never deletes, only archives**; pinned skills bypass transitions.
3. **System-prompt guidance** (`agent/prompt_builder.py:193–230`): "Skills come first… Memory is the narrow exception for facts that apply to EVERY session… hard character limit." The design philosophy: knowledge accretes in **skills** (load-on-relevance) and **SQLite** (episodic, searchable), while always-loaded memory stays tiny.
4. **Insights** (`agent/insights.py` via `idx_messages_assistant_calls_by_session`): scans tool/skill usage from message history — behavior-derived statistics.

## 6. Scheduling / proactivity

- **Cron** (cron/scheduler.py): tick every 60s under a file lock; jobs have skills, context_from (upstream job output), script preambles with a **wake gate** (`{"wakeAgent": false}` skips the LLM run entirely — #1232), per-job model overrides, delivery to any platform, retry/quota/incident tracking. Prompt assembly runs credential-exfil and config-block scans (cron/scheduler_prompt.py). Cron runs are stored as sessions (`source='cron'`) and demoted in search.
- **Heartbeats/goals**: gateway heartbeat acceptance/restore (run_heartbeat_acceptance.py), per-turn goal judging (above).
- **ESTOP** (agent/estop.py): `hermes pause` writes `$HERMES_HOME/ESTOP`; cron, kanban dispatcher, and NEW gateway turns skip work (in-flight never killed); corrupt/empty file still counts as engaged (fail-safe).

## 7. Integrations

- **Messaging gateway** — the broadest I've seen in OSS: built-ins (Telegram [gateway/config.py:221], Signal, WhatsApp Cloud, webhook, api_server, weixin, qqbot, yuanbao, BlueBubbles iMessage, MS Graph) + plugin platforms (plugins/platforms/: discord, slack, email, matrix, irc, line, teams, feishu, dingtalk, google_chat, mattermost, ntfy, sms, simplex, wecom, a2a, photon, raft, buzz). **Home Assistant moved out of core to the plugin catalog (Oct 2026, plugins/AGENTS.md:28–30)**. Cross-platform conversation continuity via `sessions.handoff_*` columns; voice-memo transcription; DM pairing for authz.
- **Plugin catalog**: 406 YAML entries (plugin-catalog/), with pinned commits and explicit delisting/removal disclosure commits.
- **Memory providers**: `MemoryProvider` ABC (agent/memory_provider.py); in-tree plugins closed to new entries (mem0, byterover, holographic, openviking, retaindb remain); honcho/hindsight/supermemory moved to catalog (plugins/AGENTS.md:19–27).
- **Providers**: `PROVIDER_REGISTRY` (hermes_cli/auth.py:250) — nous (OAuth Portal), openrouter, openai, anthropic, gemini, zai, minimax, deepseek, xai, bedrock, vertex, copilot, kimi-coding, ollama-cloud, custom. Auxiliary-model subsystem (judge/review/title generation) with fallback chains and cooldowns.
- **MCP**: full client (`tools/mcp_tool*.py`, ~20 modules) AND **server mode** — `hermes mcp serve` (mcp_serve.py) exposes conversations/approvals as MCP tools to any client; OpenClaw-compatible 9-tool channel bridge + `channels_list`.
- **Terminal backends**: local, Docker, SSH, Singularity, Modal, Daytona, Vercel Sandbox (README; agent/terminal_env_registry.py).

## 8. Security / approval model

Layered, and unusually well-thought-out (tools/approval.py facade docstring maps the leaves):

1. **Hardline floor** (approval_detection/approval_floors): patterns that always block; user deny globs block unconditionally.
2. **Dangerous-command detection** → approval flow through the active transport: CLI prompt, gateway queue (blocking round-trip to phone), plugin transports, MCP elicitation.
3. **Smart approval** (approval_smart.py): a guardian LLM verdict, with a **denial breaker** — 3 consecutive guardian denials escalate to a "CIRCUIT BREAKER: STOP attempting variations" tool-result instruction (prompt-cache-invariant; approval.py:88–110).
4. **YOLO mode frozen at import** (`_YOLO_MODE_FROZEN`, approval.py:38–40) — reading the env var per call would let any skill running in-process flip it: a prompt-injection escalation path, explicitly closed.
5. **Protected instruction files**: agent writes to AGENTS.md/CLAUDE/SOUL.md always require human approval, even under --yolo; refused when nobody can answer (cli-config.yaml.example:570–577).
6. **Memory/skill writes staged for approval** with matched-entry pinning (§3.3).
7. Skill creation runs `_security_scan_skill`; skills_guard/skills_ast_audit on load; plugin_guard for catalog plugins; tirith optional pre-exec scanner; cron prompts scanned for credential exfil.
8. Subagent defaults: dangerous-command prompts auto-**deny** (default false=deny; auto-approve "once" opt-in, always audit-logged).

Defaults are conservative; audit logging is pervasive (ledger, delivery ledgers, lifecycle ledgers, forensics on shutdown).

## 9. UI & phone/remote access

Phone access = the messaging gateway (Telegram et al.) with slash commands (/new, /model, /skills, /status, /approve), streaming, voice memos, approvals delivered as chat prompts. Desktop = Electron app (`apps/desktop`) with the "learning made visible" journey graph (agent/learning_graph.py renders skills + MEMORY.md/USER.md chunks as a graph). TUI (Ink-style TS) and a web dashboard. No native mobile app — messaging IS the mobile strategy.

## 10. Strengths

- The **learning loop** (background review + curator + ledger + rollback) is the most mature "agent learns from its own experience" machinery in open source; the prompts encode dozens of observed failure modes.
- **Session persistence** done seriously: SQLite + dual FTS5 indexes, compression lineages with parent-session chains, turn leases, WAL handling, portability/repair modules (~10 hermes_state_* files just for repair).
- **Gateway breadth** (~25 platforms) and phone-first approval flow — matches Executor's "reachable from a phone" requirement out of the box.
- **Security posture** exceeds peers: layered approvals, frozen yolo, protected instruction files, exfil scans, estop, audit ledger.
- **Provider-agnostic** incl. local models; auxiliary model separation (cheap judge/guardian vs main model) is exactly the right cost shape.
- Maintenance: 5,653 test files, minute-level commit activity, issue-driven comments, funded org (Nous). MIT.

## 11. Weaknesses

- **No structured life-management model.** Tasks/projects/routines for the *user* don't exist; kanban is a code-task dispatcher (worktrees, branches); memory is capped free text; no notion of energy/context/ADHD scaffolding. Executor's core data model must be built entirely on top.
- **Scale/churn risk**: godfiles, minute-level upstream velocity, schema constantly evolving (schema_version + heal DDL paths). Forking means owning an enormous merge debt; the plugins/AGENTS.md "plugins never touch core" policy exists precisely because they know coupling is fatal.
- **Weight**: Python 3.14 + Node 22 + uv + PM + ffmpeg on the always-on box; heavier than a minimal Executor could be, though fine for a home server.
- Memory char caps (2.2K/1.4K) are the opposite of "accumulated context about the user's life" — by design. All accumulation must live in Executor's own store.
- Cron sessions and subagent transcripts pollute state.db (they mitigate via demotion/exclusion, but the DB grows).
- Young (Jul 2025) despite the lineage; behavior changes under you.

## 12. Surprising design decisions

- After **every turn** a forked agent replays the conversation to decide what to persist — an LLM call multiplier accepted as the cost of learning.
- `conversation_generations` rows are deliberately never garbage-collected (ABA closure comment, hermes_state_common.py:559+).
- Trigram FTS excludes cron/subagent bytes after measuring them at ~70–90% of message volume.
- Memory is intentionally tiny; skills are the accretion layer; SQLite is the episodic layer. Clean three-way split.
- The deny-first subagent default and the frozen-env-var yolo check show real adversarial thinking about the agent attacking itself.
- Hermes' ancestry comments point at nanoclaw/gastown (OpenClaw lineage) — `hermes claw migrate` makes the succession explicit.

## 13. Unfinished / vaporware / rough edges

- Open GitHub issues are operational, not architectural: cron scheduler double-import crash (#132732), Windows update stalls (#132706), memory-provider SQLite backup omitting WAL records (#132705), OAuth issuer placeholder bug (#132730). Nothing found that looks abandoned-at-the-core.
- `workflow_template_id`/`current_step_key` in kanban are marked "forward-compat for v2 workflow routing… dispatcher doesn't consult them yet" (kanban_db.py:947 comment) — announced, not built.
- Godfile eradication is an open, multi-quarter effort (#78647).
- README "closed learning loop" is real code, not vaporware; Honcho "dialectic user modeling" survives only as a catalog plugin, not a differentiator.

## 14. VERDICT for Executor

**REUSE as runtime substrate — do not fork. Build Executor's structured life layer as skills + MCP tools + (optionally) a kanban-repurposed task spine, and treat upstream as a moving platform you track, not own.**

Reasons:
- Runtime requirements Executor actually has — always-on daemon, phone-reachable gateway with approval round-trips, cron with wake gates, subagents, provider abstraction, SQLite+FTS5 persistence, MIT license, extremely active maintenance — are all present and hardened far beyond what Executor could rebuild in a year.
- The learning loop is the closest existing thing to "learn from actual behavior"; its review prompts (background_review.py:357–526) and curator policy are worth porting verbatim even if everything else is discarded.
- Forking is disqualified by velocity + size; contribution-plugin-extension is the designed path ("Plugins never touch core", plugins/AGENTS.md).
- The candidate architecture (SQLite + Git/Markdown + custom layer over a runtime) maps cleanly: Hermes already runs on the home machine; Executor's structured DB (projects, tasks, micro-steps, behavior telemetry) would be exposed via a custom MCP server or toolset plugin — the same seam `hermes mcp serve` and the plugin catalog already define. Markdown/Git fits the skills + MEMORY.md model for human-readable knowledge, with SQLite for structured state — Hermes validates both halves.
- Where Hermes does NOT help: the life-management domain model, ADHD-specific scaffolding (initiation micro-steps, context/energy modeling), and any UI beyond chat. That's genuinely greenfield.

If, instead, Executor is built from scratch on a thinner base (or directly on the pi/OpenClaw-style harness), Hermes demotes to **pattern-donor**: steal (1) the three-layer memory split, (2) the background-review prompt suite, (3) FTS source-demotion + lineage dedup, (4) the approval floor/guardian/breaker stack, (5) kanban's claim-lock/heartbeat/circuit-breaker durability pattern, and (6) the skill ledger with content-addressed rollback.

Compared to OpenClaw (TS, markdown-file memory, multi-channel gateways, cron/heartbeat): Hermes has narrower native mobile-app story but strictly deeper memory machinery (SQLite+FTS vs flat files), a harder security posture, and hotter maintenance; OpenClaw remains the lighter-weight fallback and the migration source Hermes explicitly targets.
