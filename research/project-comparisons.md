# Project comparisons — verdicts with evidence

All repos were cloned to `~/src/` and inspected at schema/code level. Full file-cited reports in
`notes/`. "LOC" figures from the child analyses. Activity/stars from GitHub API 2026-10-04.

## Summary table

| Project | Scale / activity | Domain fit | Runtime fit | License | Verdict |
|---|---|---|---|---|---|
| **OpenClaw** | ~3.85M LOC TS, 15.5k test files, 391k★, weekly releases | none (no tasks/projects schema) | excellent | MIT | **Reuse as substrate** |
| **Hermes** | ~150+ py modules, 5.6k test files, 251k★, hourly commits | none (kanban = code dispatcher) | excellent | MIT | Reuse as substrate (alt.) |
| **CORE** | ~2.2k files, 2k★, active, 211 open issues | partial (task+page+thread; no prioritizer/learning) | medium (4 services, 8GB) | **AGPL-3.0** | Pattern-donor |
| **DailyOS** | ~11k LOC, 1★, single squashed commit, dormant-ish | partial (day planner; flat tasks; no learning) | poor (CLI-only) | — | Pattern-donor |
| **Nudge** | ~6.2k LOC Electron, 23★, dormant | partial (ADHD prose, no code) | poor (desktop-only) | — | Pattern-donor |
| **PersonalOS** | 1k-line MCP server + conventions, 566★, dormant | thin (conventions only) | none | CC BY-NC-SA | Pattern-donor |
| beranradek/personal-assistant | serious harness, 9★, active | thin (no life schema) | medium (self-built gateway) | — | Pattern-donor |
| **evidenceofLife-v2** | ~53k LOC React/Supabase, active | **best behavioral data model** | none (browser-hostage) | — | Copy the schema |
| **adhd-day-planner** | 948 lines (a skill) | heuristics only | none | — | Steal heuristics |

## OpenClaw — reuse as substrate

- **What's verified:** gateway daemon (40+ channels incl. WhatsApp/Telegram/Signal/iMessage),
  native iOS/Android/companion apps + Control UI; automations (one-shot/cron/webhook/condition
  watchers, isolated sessions, delivery ledger); heartbeats with outcome states; standing
  intents (deterministic FTS trigger path, fire budgets, no LLM in matching); memory = canonical
  Markdown (`MEMORY.md`, `USER.md`, daily notes) + derived SQLite index with provenance classes
  (`owner/agent/untrusted/system`) enforced as SQL CHECK constraints; dreaming consolidation
  (deterministic gates → bounded model rewrite → structural validation); exec approvals binding
  argv + executable identity; 5 sandbox backends; plan-completion self-check; plugin SDK with
  own-SQLite pattern (memory-core, logboard, workboard); skills = SKILL.md files (~60 bundled).
  (`notes/openclaw.md`, `notes/_parent-verification.md`)
- **Missing for Executor:** structured task/project/deadline store; behavioral analytics;
  decomposition; any ADHD surface; Git versioning of workspace.
- **Risks:** churn (weekly releases, config compat breaks), chat-centricity. Mitigate: pin
  versions, build only on stable surfaces (workspace/skills/plugins/config), keep the domain
  store self-contained and runtime-replaceable.

## Hermes — credible alternative substrate

- **What's verified:** SQLite `sessions/messages` + three FTS5 indexes (BM25, trigram, CJK) with
  cron/subagent demotion and trigger-synced rebuilds; after-every-turn background review that
  writes MEMORY.md/USER.md (capped ~2.2k chars) and skills, with the best "learn from mistakes"
  prompt corpus in OSS; skill ledger with sha256-addressed blobs and rollback; curator
  lifecycle; kanban task substrate (claim locks, heartbeats, circuit breakers); layered
  approvals (floors → guardian LLM with denial circuit-breaker → frozen YOLO env; protected
  instruction files; memory writes staged with matched-entry pinning); ~25 gateway platforms;
  MCP client **and** server; provider-agnostic incl. local. (`notes/hermes.md`)
- **Missing:** any life-domain model; memory intentionally tiny; no native mobile apps
  (messaging is the mobile strategy).
- **Risks:** godfiles, hourly upstream churn, Python+Node weight.

**OpenClaw vs Hermes (decision):** OpenClaw wins on phone surface (native apps + Control UI +
board widgets as a "today view"), markdown-canonical memory (aligns with Git/Markdown instinct),
and the standing-intents deterministic trigger design. Hermes wins if you prefer Python, want
its deeper learning-loop machinery closer to the core, or value its approval stack. The
recommendation (OpenClaw) is not overwhelming — which is exactly why the domain layer must stay
runtime-agnostic (see recommendations.md #1).

## CORE — pattern-donor (the task-context oracle)

- **Real:** Zep-style temporal graph with `validAt/invalidAt/invalidatedBy` fact invalidation;
  Task = first-class (dedicated conversation thread + Page spec with `<plan>/<outcome>/<log>`
  zones, RRule recurrence, subtasks with depth cap, `Waiting`→`unblock_task` protocol,
  `reschedule_self`); 2-minute edit-buffer veto instead of modal approvals; Watch Rules as
  user-editable markdown policy ("surfacing ≠ acting"); compaction with coverage watermark;
  RecallLog observability. (`notes/core.md`)
- **Disqualifying:** AGPL-3.0; SaaS-first core (Stripe credits, OAuth server, workspaces);
  Postgres+pgvector+Redis+Neo4j (4 vCPU/8GB for one user); tasks are agent-work units, not a
  user-executive-function surface; prompt-encoded state machines that drifted from code (think
  tool ghost, empty scratchpad-scan branch, recurring-plan freeze bug).
- **Use as:** the reference for "how should a task acquire context" and a catalog of week-3
  failure modes.

## DailyOS — pattern-donor (the trust-architecture reference)

- **Real:** deterministic skeleton with LLM as bounded proposer — regex request parsing,
  deterministic ranking (overdue 100 / due-today 70 / priority weights), deterministic schedule
  fill, single LLM timeline call with ≤3 repair attempts and deterministic fallback; risk-tiered
  approvals (`read/local_write/external_write/destructive`) with **editable decided arguments**;
  full audit trail (`agent_runs/tool_calls/approval_requests`); idempotent commitment upserts;
  systemic injection doctrine ("external_untrusted_data … never instructions"). (`notes/dailyos.md`)
- **Disqualifying as base:** flat tasks (no subtasks/estimates/dependencies; task→project link
  is a JSON blob), no learning, CLI-only, no daemon, one maintainer, 1 star.

## Nudge — pattern-donor (the behavior spec)

- **Real:** a ~350-line system prompt refined through visible LLM-failure fixes: never-guilt
  tone, checkbox dopamine, capture-time "What does starting look like?" micro-step sections,
  anti-drift rules; idea files with `status/priority/type/energy/size/started` frontmatter. (`notes/nudge.md`)
- **Not real:** ranking logic (none — 100% LLM discretion), behavioral learning (none), git sync
  (README claims, code absent), remote access (desktop-only); one install-breaking open issue.

## PersonalOS — pattern-donor (the conventions)

Markdown/YAML conventions + a 1k-line MCP server: fuzzy dedup (SequenceMatcher+Jaccard 0.6),
P0≤3 WIP cap enforced in code, clarify-before-create gate. Vaporware noted: broken imports,
unimplemented `r` status, path-traversal bug. (`notes/personalos.md`)

## personal-assistant (beranradek) — pattern-donor (the heartbeat)

Heartbeat + HEARTBEAT_OK suppression gate (deterministic cadence → context packet → LLM turn →
output-gated notification); markdown-in-git memory with hybrid search (sqlite-vec + BM25 +
recency); episodic store with `outcome/success_score/blockers`; secrets-isolated integration
proxy (OAuth away from the agent); security hook stack. Life layer barely exists. Note: wrapping
subscription-billed CLIs for an always-on agent conflicts with their ToS — API-key billing
required. (`notes/personal-assistant.md`)

## evidenceofLife-v2 — copy the schema

Every todo carries `plan_started_at/plan_ended_at` (intent) **and** `timer_started_at/
timer_ended_at/timer_seconds` (actual); steps are child todos with own timers; focus sessions
persist as tagged events; learning is conservative and tested (median durations with
min-sample gates; plurality day-segment preference ≥3 samples + ≥50%); machine suggestions appear
as ghost-previews the user accepts. History is immutable (unfinished tasks stay on their date;
today merges them read-only). (`notes/small-planners.md`)

## adhd-day-planner — steal the heuristics

ADHD tax (15→20/30→40/60→75 min), transition breaks, momentum starter, meal anchors,
anti-hyperfocus-before-appointments, fixed processing queue — all prompt-only, zero enforcement
(its cleanup script even deletes dropped tasks). Steal the corpus, enforce deterministically.

## Additional ecosystem (from ecosystem-survey.md)

Taskwarrior+Timewarrior (component option; urgency formula prior art), basic-memory (knowledge
notes component), Letta (sleep-time consolidation reference), ntfy (push channel),
Life_OS + hermes-life-os + abi/lilo (watch list), Graphiti (adopt the bi-temporal model),
Khoj (reference), screenpipe (ground-truth capture, privacy caveat).
