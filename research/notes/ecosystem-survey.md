# Executor Ecosystem Survey — Beyond the Nine Deep-Dive Projects

**Scope:** Open-source ecosystem relevant to Executor (persistent personal AI as external executive-function layer; task micro-decomposition; behavioral learning; phone access; home-server hosting). Nine projects already covered by sibling briefs and deliberately NOT re-analyzed: thatsjet/nudge-app, stadimeti19/DailyOS, amanaiproduct/personal-os, RedPlanetHQ/core, openclaw/openclaw, NousResearch/hermes-agent, Dthen/adhd-day-planner, Cyriellewu/evidenceofLife-v2, beranradek/personal-assistant.

**Method:** GitHub API metadata (stars/last-push/license/language, via `gh api`), README + repo-layout code glances, web search (personal AI OS, life management agent, ADHD assistant, executive function AI, AI GTD, agentic task manager, second brain agent, local-first personal AI, AI daily planner, personal knowledge agent), HN Algolia, GitHub topic searches (`topic:adhd`, executive function, life-os). Star counts and push dates captured 2026-10-04.

**Problem-area tags:** [RUNTIME] substrate · [MEMORY] memory component · [TASK] task model · [PLAN] planning algorithm · [PHONE] phone access · [NOTIF] notifications · [BEHAVIOR] behavioral data.

---

## Part 1 — Curated candidates, verified

### Taskwarrior + Timewarrior — GothenburgBitFactory
- https://github.com/GothenburgBitFactory/taskwarrior (6.1k★, MIT, C++, pushed 2026-09-26) · https://github.com/GothenburgBitFactory/timewarrior (1.7k★, MIT, C++, pushed 2026-10-02)
- What it actually does: CLI task manager storing JSON objects in `~/.task`. Real strengths: (1) urgency polynomial computing instantaneous priority — a battle-tested *ranking algorithm* for "what should I do now"; (2) user-defined attributes (UDAs) so a custom schema can ride on it without forking; (3) hook system (on-add/on-modify scripts) — the natural interception point for LLM enrichment (e.g. auto-decompose, estimate calibration); (4) `task ready` filtered views, recurrence, dependencies, annotations. Timewarrior clocks actual time spent and tags it — ground-truth [BEHAVIOR] data (planned vs actual duration) in plain files. Both export/import JSON; there is a JSON RPC server (taskchampion sync) and mobile clients exist but are weak.
- Areas: [TASK][PLAN][BEHAVIOR]. Verdict: **use as component** — the durable task store and urgency formula are exactly Executor's "next action" substrate; wrap with LLM hooks rather than reimplement.

### Habitica — HabitRPG
- https://github.com/HabitRPG/habitica (14.2k★, custom MIT-ish "GPL-3.0 + assets" license, JS/Express+Vue, pushed 2026-10-03, active)
- Actually does: full-stack RPG habit tracker (habits/dailies/todos, avatar, parties, guilds). Self-hostable but heavy (Node+Mongo+client build). Clean REST API. Gamification mechanics (streaks, damage on missed dailies) are precisely what ADHD UX research says to avoid in some users (shame spiral) — and catnip for others.
- Areas: [TASK] (habit/daily model) · [BEHAVIOR] (completion history). Verdict: **pattern-donor** (habit schema + API shape); skip self-hosting the whole RPG unless gamification is wanted; if so, integrate via API.

### org-mode / org-agenda (Emacs)
- https://git.savannah.gnu.org/cgit/emacs.git (lisp/org) — part of Emacs; mirrors exist. 15+ yrs mature, GPL.
- Actually does: plain-text outline tasks with SCHEDULED/DEADLINE, repeaters, priorities, clocking (org-clock), agenda views, org-capture (frictionless inbox), org-habit (consistency graph showing behavioral streaks), archiving (planned-vs-done history). The single most battle-tested *personal planning data model* in existence — and it's plain text with parsers in every language (orgmode parsers in Python/JS/Go).
- Areas: [TASK][PLAN][BEHAVIOR]. Verdict: **pattern-donor** — copy the semantics (capture → refile → schedule → clock → review → archive), not the Emacs.

### Letta (MemGPT) — letta-ai
- https://github.com/letta-ai/letta (25k★, Apache-2.0, Python, pushed 2026-09-10, active)
- Actually does: server platform for stateful agents. Core ideas worth stealing: **memory blocks** (editable, named context segments the agent itself rewrites), **core vs archival/recall memory tiers**, **sleep-time agents** (background job that reorganizes memory while main agent is idle — perfect fit for nightly "what did I learn about this user's behavior" consolidation), message-buffer persistence with Postgres/SQLite. Runs as REST service ("Letta File" = agent+memory state).
- Areas: [MEMORY][RUNTIME]. Verdict: **pattern-donor, optionally component** — the memory-tiering and sleep-time-compute architecture is the reference design; running full Letta may be heavier than needed if OpenClaw/Hermes already supplies a runtime.

### mem0 — mem0ai
- https://github.com/mem0ai/mem0 (66.6k★, Apache-2.0, Python, pushed 2026-10-01, very active)
- Actually does: embeddable memory library: extracts facts from conversations, stores vector+optional graph memories, retrieves via hybrid search, updates/contradicts facts over time. `pip install mem0ai`, point at any LLM+vector store. It is a *library*, not an agent.
- Areas: [MEMORY]. Verdict: **candidate component** for conversational memory; simpler than Letta to embed; lacks Letta's self-editing-context architecture.

### Zep / Graphiti — getzep
- https://github.com/getzep/graphiti (31.4k★, Apache-2.0, Python, pushed 2026-10-04, very active) · getzep/zep (4.9k★, Apache-2.0 — Zep service itself is being sunset toward Graphiti-core)
- Actually does: Graphiti builds **temporal knowledge graphs** (bi-temporal: event time vs ingestion time) from unstructured input via LLM entity/relationship extraction, supports incremental updates and invalidation (facts expire, don't just accumulate). Neo4j or falkordb backend.
- Areas: [MEMORY][BEHAVIOR]. Verdict: **pattern-donor, heavy component** — temporal invalidation is exactly right for "user's routine changed last month" reasoning; Neo4j dependency is real ops weight for a home box. Related OSS in same space: **cognee** (topoteretes/cognee) and **supermemory** — same category, none clearly dominant for single-user scale.

### Basic Memory — basicmachines-co
- https://github.com/basicmachines-co/basic-memory (4.1k★, AGPL-3.0, Python 3.12, pushed 2026-10-02, active; has paid cloud, OSS core)
- Actually does (README+code glance): MCP server + local sync service over a **Markdown-first knowledge graph**: notes on disk with wikilinks + "observations" (structured subject-predicate-object triples parsed out of Markdown), semantic + keyword hybrid search with optional cross-encoder rerank, two-way human/agent editing of the same files, tool-discovery hints (read-only/destructive tags per MCP tool). This is nearly the "SQLite + Git/Markdown" memory substrate the Executor sketch imagines, already built as MCP.
- Areas: [MEMORY]. Verdict: **use as component** — strongest single match for the planned memory layer; AGPL is fine for personal use.

### Khoj — khoj-ai
- https://github.com/khoj-ai/khoj (37.6k★, AGPL-3.0, Python/Django, pushed 2026-08-02, active-ish)
- Actually does: self-hostable "AI second brain": indexes Obsidian/Notion/PDFs/markdown, chat with any LLM (local or API), **scheduled automations** (research + delivery jobs), deep-research mode, custom agents, web access, WhatsApp integration. Mature multi-platform clients (web/desktop/Obsidian/Emacs). Weakest side: it's query-centric — no persistent model of your commitments, no task decomposition.
- Areas: [MEMORY][RUNTIME-lite][PHONE] (WhatsApp). Verdict: **use as component or strong pattern-donor** for the knowledge/research half of Executor.

### Goose — block/goose (moved to aaif-goose/goose)
- https://github.com/aaif-goose/goose (54.9k★, Apache-2.0, Rust, pushed 2026-10-02, very active; org renamed from block/)
- Actually does: extensible CLI/desktop AI agent (MCP client+server), Rust core, extension system, schedule/cron recipes (`.goose/handlers`), runs any LLM provider. Developer-workflow oriented (coding, shell) but the cron+MCP+extension substrate is general.
- Areas: [RUNTIME]. Verdict: **alternative runtime candidate** behind OpenClaw/Hermes; leaner and more stable, less "life-oriented" gateway story.

### Agent Zero — frdel/agent-zero (moved to agent0ai/agent-zero)
- https://github.com/agent0ai/agent-zero (19.4k★, custom license (Raph's license — source-available, restrictions), Python, pushed 2026-10-02, active)
- Actually does: autonomous personal-agent framework: full-computer access (terminal, browser, file mgmt), long-term memory stored as browsable files/folders, dynamic code generation instead of fixed tools, multi-agent delegation, web UI. Philosophy: agent builds its own tools at runtime.
- Areas: [RUNTIME][MEMORY] (file-based memory). Verdict: **pattern-donor** — memory-as-browsable-folder and self-extending tools are good ideas; license and security posture make it a poor always-on choice.

### Huginn — huginn
- https://github.com/huginn/huginn (50k★, MIT, Ruby, pushed 2026-10-04 — maintenance-mode cadence; 10-yr-old project)
- Actually does: builds agent networks ("Scouts" watch triggers → "TriggerAgents" react): scrape pages, watch RSS/JSON for changes, send Digest/Telegram/Slack messages, propagate events between agents, 100+ community example scenarios ("agent that tells me when a domain expires"). Still the reference grammar for **event-condition-action personal monitoring**. No LLM (pre-dates them; LLM via API-call agents is bolt-on).
- Areas: [NOTIF][BEHAVIOR][RUNTIME-lite]. Verdict: **pattern-donor** — steal the trigger→digest model; implement fresh on an LLM runtime rather than deploying Ruby.

### n8n — n8n-io
- https://github.com/n8n-io/n8n (206.6k★, fair-code "Sustainable Use" license — NOT OSI open source, TypeScript, pushed 2026-10-04, hyper-active)
- Actually does: visual workflow automation, 400+ integrations, AI/agent nodes, self-hostable, cron/webhook triggers. Could serve as Executor's integration surface (email→task, calendar→context) without writing glue code.
- Areas: [NOTIF][RUNTIME-lite]. Verdict: **optional component** — overkill as core; useful as peripheral glue. (Activepieces is the Apache-friendly alternative in the same category.)

### ntfy — binwiederhier
- https://github.com/binwiederhier/ntfy (34.6k★, Apache-2.0, Go, pushed 2026-10-03, active)
- Actually does: pub/sub push notifications over HTTP (`curl -d "time to start" ntfy.sh/topic`), self-hosted server + iOS/Android/desktop apps, websockets/SSE, attachments, scheduled delivery, access control. Trivial to run on a home box; phone apps talk to your server.
- Areas: [NOTIF][PHONE]. Verdict: **use as component** — the nudge/proactive-prompt delivery channel. Near-zero risk.

### Vikunja — go-vikunja
- https://github.com/go-vikunja/vikunja (5.6k★, AGPL-3.0, Go, pushed 2026-10-04, active; API moved off GitHub)
- Actually does: self-hosted to-do manager: lists/kanban/Gantt/calendar, labels, reminders, recurring tasks, REST API + OpenAPI spec, webhooks, CalDAV, Linkding/karakeep integrations, official Dart mobile app (go-vikunja/app). Solid conventional task schema — but no urgency model, no decomposition, no AI anything.
- Areas: [TASK][PHONE]. Verdict: **skip as core** (schema too conventional for micro-step plans) but the OpenAPI+webhook surface makes it a viable UI/frontend if a richer model lives behind Executor.

### Super Productivity — johannesjo (now super-productivity org)
- https://github.com/super-productivity/super-productivity (22.5k★, MIT, TypeScript/Angular+Electron+Nativescript, pushed 2026-10-04, active)
- Actually does: timeboxing-first todo app: drag tasks onto a day plan, Pomodoro/breaks, idle detection, "anti-procrastination" finish-your-tasks prompts, time tracking per task, Jira/GitLab/GitHub/OpenProject sync, local-first IndexedDB with optional WebDAv/Dropbox sync, Android/iOS apps. The **timeboxing + break rhythm + idle-detection** combo is the best ADHD-adjacent UX in mainstream OSS task apps.
- Areas: [TASK][PLAN][BEHAVIOR]. Verdict: **pattern-donor** (day-plan UX, idle detection, break cadence); closed client architecture makes embedding an agent awkward (no server process).

### Screenpipe — mediar-ai (now screenpipe org)
- https://github.com/mediar-ai/screenpipe (21.8k★, source-available (custom license, NOASSERTION), Rust+TypeScript, pushed 2026-10-04, hyper-active; YC S26 per README)
- Actually does: continuously records screen + audio locally, OCRs/ASRs everything into a searchable local store (SQLite+berty/tantivy-style indexing), plugin API and "pipes" (LLM agents over history), OpenComputerHistory framing. The ground-truth answer to "what did the user *actually* do all day" for behavior learning.
- Areas: [BEHAVIOR][MEMORY]. Verdict: **use as component for behavioral ground truth** (privacy tradeoff acknowledged; local-first by design). License is non-OSI — verify terms if redistribution matters; personal use fine.

### Karakeep — karakeep-app (formerly Hoarder)
- https://github.com/karakeep-app/karakeep (29.4k★, AGPL-3.0, TypeScript/Next.js+React Native, pushed 2026-10-03, active)
- Actually does: self-hosted "bookmark everything" (links/notes/images), AI auto-tagging (multimodal LLM), full-text + AI search, mobile apps, browser extensions, Monica-style single-user. 
- Areas: [MEMORY] (capture inbox). Verdict: **optional component** — capture funnel if Executor wants a read-later inbox; otherwise skip (karakeep+Vikunja already integrate).

---

## Part 2 — New discoveries from search

### EntangledQuantum/Life_OS ⭐ closest match found
- https://github.com/EntangledQuantum/Life_OS (37★, MIT per README — no LICENSE file detected via API, TypeScript monorepo ~1.4MB, pushed 2026-09-04; mobile-frontend/ + apps/ + packages/)
- Actually does: **"ADHD-friendly personal execution OS"** — the pitch is a direct inversion of ordinary apps: *"You do the doing (tap to complete). Your agent does the designing (creates habits, schedules the day, writes goals, watches what happened, adjusts)."* Single SQLite file, no account, Node 22.5+, agent connects over **MCP**, agent self-installs via `AGENT_SETUP.md` prompt, schedules its own nightly check-in. Explicit ADHD design refusals: no streaks, no leaderboards, nothing turns red on a bad day; XP measures today vs today's target only. Two-noun model: Habits (recur, scored daily) + Tasks (everything else with optional parts). Android app + web dashboard. Example behavior: "You've skipped the 07:00 reading block four days running. Moving it to 21:30."
- Areas: [TASK][PLAN][BEHAVIOR][PHONE][RUNTIME] — everything but memory substrate. Verdict: **fork/study as primary pattern source** — small community, but it is the nearest existing implementation of Executor's thesis; validate code quality before reuse, else transplant its UX contract.

### Lethe044/hermes-life-os
- https://github.com/Lethe044/hermes-life-os (199★, MIT, Python ~527KB, on PyPI, pushed 2026-09-13; NousResearch Hermes hackathon winner-adjacent)
- Actually does: daily-life logging agent (mood, sleep, meals, hydration, workouts, stress, focus sessions) with **cron-driven cadence**: 07:00 morning brief, 12:00, 18:00 evening, 23:00 consolidation, Monday 08:00 weekly review. Pattern-detection runs as parallel subagents correlating life dimensions ("energy crashes after poor sleep"); Atropos RL reward shapes personalization over time; memory recalled before every response.
- Areas: [BEHAVIOR][PLAN] (proactivity cadence, correlation loop). Verdict: **pattern-donor** — the 4×daily+weekly cron rhythm and "detect pattern → brief user" loop are the right proactive skeleton; tied to Hermes runtime (fine — Hermes is under separate deep-dive).

### abi/lilo
- https://github.com/abi/lilo (44★, MIT, TypeScript, pushed 2026-07-17, alpha)
- Actually does: self-hosted personal assistant whose primary UI is **Telegram** (also WhatsApp/web/desktop/email): photo→calorie tracking, voice note→TODO, "remind me when the Knicks game starts + score updates every 5 min", receipt capture for reimbursement, meeting scheduling with leave-time reminders. Companion app for visual TODO manipulation. Workspace state synced via **git**. Alpha-quality (author's warning) but the chat-as-primary-UX + git-backed-state design is exactly Executor's phone-access pattern.
- Areas: [PHONE][MEMORY-lite][TASK]. Verdict: **pattern-donor** (chat-first UX, voice/photo capture, git state).

### dongdongbh/Mindwtr
- https://github.com/dongdongbh/Mindwtr (2.2k★, AGPL-3.0, TypeScript (React Native + Tauri-style desktop), pushed 2026-10-04, active, 30k+ users)
- Actually does: faithful GTD app (inbox → clarify → projects/next-actions/waiting-for/someday, contexts) across Windows/macOS/Linux/iOS/Android, offline-first, no account. No AI, no server.
- Areas: [TASK]. Verdict: **pattern-donor** for the GTD clarification pipeline (capture→next-action) which is the non-AI half of "what do I do right now"; mobile GTD reference.

### aeshef/obsidian-agent
- https://github.com/aeshef/obsidian-agent (9★, MIT, Python, pushed 2026-09-30)
- Actually does: self-hosted Telegram bot over an Obsidian vault: capture (text/voice/photo) → markdown/SQLite in vault, kanban tasks, notes RAG, personal-finance module. Architecturally interesting: **fail-closed capability manifest** — each module (planning/knowledge/finance) that's off is removed from UI, tools, *and prompts*; optional 24/7 VPS; Obsidian remains the human editor.
- Areas: [PHONE][TASK][MEMORY]. Verdict: **pattern-donor** (fail-closed module gating; Telegram+vault loop) — too small to build on.

### fronalabs/frona
- https://github.com/fronalabs/frona (201★, **BSL 1.1 (source-available, not OSI)**, Rust, pushed 2026-10-02, active; publishes an OpenClaw/Hermes comparison)
- Actually does: self-hosted multi-agent personal-assistant platform: single Rust process, per-principal sandboxing (per-agent syscall filtering, isolated browser sessions), one policy language governing tools+sandbox, messaging channels, inter-agent delegation, phone calls.
- Areas: [RUNTIME]. Verdict: **watch** — best-in-class agent sandboxing model (pattern-donor for Executor's security story), license blocks deep embedding.

### leon-ai/leon
- https://github.com/leon-ai/leon (17.6k★, MIT, TypeScript, pushed 2026-10-04, slow-but-alive; famous 2022 HN launch)
- Actually does: modular "skills"-based open-source assistant (speech, NLP skills) — pre-LLM architecture; today mostly legacy.
- Areas: [RUNTIME]. Verdict: **skip** (historical reference).

### ravila4/claude-adhd-skills + the ADHD-skills micro-genre
- https://github.com/ravila4/claude-adhd-skills (155★, MIT, Python, pushed 2026-03). Sibling examples: ayghri/i-have-adhd (53.3k★!), UditAkhourii/adhd (4.3k★), softcane/human-state-skills, DoxxedDoxie/ef-skill, assafkip/audhd-executive-function.
- Actually does: Claude Code *skills* — prompt/behavior packages that make coding agents ADHD-friendlier (structured output, tree-of-thought with pruning, human-state response modes, EF scaffolding for founders). Not life-management apps; they target the *coding* agent.
- Areas: [PLAN-lite]. Verdict: **category note** — evidence the "agent skills" packaging format works for EF support; Executor's life-layer could ship as skills for an existing runtime. `i-have-adhd` at 53k★ shows enormous latent demand.

### Smaller ADHD-app hits (mention-only)
- adrianwedd/ADHDo (21★): "neurodiversity-affirming, crisis detection, circuit breaker psychology, local-first" — good vocabulary, thin code.
- k4mp3r/executive-functioning-assistant-public: template/fork-me ADHD management system.
- Frizzbay/adhd-ai-assistant, arpit-005/NeuroCoach: student projects, skip.
- ThirdEyeRose/executive-function-bot (2018, Telegram bot for EFD) and nvanbaak/pocket-butler (2021): confirm this idea recurs every few years and dies without a memory substrate.
- kcwoodfield/LifeOS-OSS, arnaldo-delisio/arnos, nbramia/LifeOS, djangonavarro220/agentic-life-os, lifeos-app/lifeos: Obsidian/Claude template kits and small conversational-OS builds; pattern echo, no new architecture. (lifeos-app/lifeos notable only for gamified-PWA angle.)

### Pattern essays (not repos)
- Zoe (zoe.im) "Host your life on GitHub: a personal data architecture for the agent era" (Jun 2026): agents `git pull` on wake, life data as repos — the Git-backed-state argument, third-party validation.
- Mnemosyne (glukhov.org): local-first SQLite memory provider for Hermes Agent (working memory / structured facts / temporal / episodic) — confirms SQLite memory layer is an emerging norm.
- jamesm.blog "Giving Your Home AI Agent Memory That Lasts": practitioner walkthrough of durable structured memory on a home agent.

---

## Part 3 — Categories surveyed and deliberately skipped

1. **Thin "chat with your notes" wrappers** (category, per brief): Quivr, AnythingLLM, most Obsidian-Copilot plugins. RAG over notes ≠ persistent life model. No behavioral learning, no proactivity. Skip; Khoj/basic-memory already represent the defensible end of this category.
2. **Proprietary ADHD/AI planners** (category): Motion, Reclaim, Sunsama, SkedPal (auto-scheduling), Shimmer/Inflow (ADHD coaching), Lindy/Attache (AI EA). No code to reuse; their one transferable idea: **auto-scheduling with protected deep-work blocks and post-hoc plan-vs-actual reconciliation**.
3. **Coding-agent ADHD skills**: see Part 2 — demand signal, not architecture.

---

## Part 4 — Cross-cutting observations

1. **The field is converging on one stack shape**: single-user runtime (OpenClaw/Hermes/goose-class) + MCP for tool/data access + SQLite or Markdown+Git for state + Telegram/WhatsApp for phone + cron for proactivity. Executor's sketch is not exotic; its differentiator must be the *planning/decomposition + behavior-learning layer*, which nobody ships well.
2. **Nobody does micro-decomposition well.** Not one surveyed project generates "open the left grow tent → bag the cardboard" plans from accumulated context. The 9 excluded deep-dives are being checked for this; in the wider ecosystem it's absent. This is the open core of Executor.
3. **Behavioral learning is the second gap.** Screenpipe (ground truth), Timewarrior/org-clock (duration truth), Life_OS & hermes-life-os (pattern-detect → adjust loop), Letta sleep-time compute (consolidation mechanism) — the pieces all exist; no project composes them.
4. **ADHD UX conventions are emerging** (Life_OS's refusals: no streak-guilt, no red; hermes-life-os cadence): design for *self-compassionate prompting* — brief, at the right moment, via a channel that interrupts appropriately (ntfy/Telegram).
5. **Taskwarrior's urgency polynomial** is the only production-grade "what now" ranking algorithm in OSS; treat it as Executor's baseline to beat/extend with learned weights.

---

## Part 5 — Ranked shortlist: 10 most important additions

1. **Taskwarrior + Timewarrior** [TASK/PLAN/BEHAVIOR] — durable JSON task store + UDA extensibility + hook system for LLM enrichment; urgency algorithm as ranking baseline; Timewarrior gives plan-vs-actual durations. Use as component.
2. **EntangledQuantum/Life_OS** [TASK/PLAN/BEHAVIOR/PHONE] — nearest existing implementation of Executor's thesis (agent designs the system, user taps; SQLite; MCP; nightly self check-in; ADHD-affirming UX). Fork or transplant its UX contract.
3. **basic-memory** [MEMORY] — Markdown-first knowledge graph with MCP, hybrid search, human/agent co-editing. Use as the memory component (AGPL, fine personally).
4. **Letta (MemGPT)** [MEMORY] — memory blocks, tiered memory, sleep-time agents: the reference architecture for nightly behavioral consolidation. Pattern-donor (optional component).
5. **ntfy** [NOTIF/PHONE] — self-hosted push to iOS/Android via one HTTP call. The nudge channel. Use as component.
6. **hermes-life-os** [PLAN/BEHAVIOR] — working cron cadence (4×daily + weekly review) and detect-pattern→brief-user loop. Pattern-donor for proactivity skeleton.
7. **abi/lilo** [PHONE] — Telegram/WhatsApp-first assistant UX with voice/photo capture and git-backed workspace. Pattern-donor for phone access despite alpha status.
8. **screenpipe** [BEHAVIOR] — local 24/7 screen+audio capture → searchable index → what you actually did. The ground-truth behavior feed (privacy + non-OSI license caveats).
9. **Khoj** [MEMORY/RUNTIME-lite] — most mature self-hosted second brain with scheduled automations and WhatsApp; component or reference for the knowledge/research half.
10. **Graphiti** [MEMORY/BEHAVIOR] — bi-temporal knowledge graphs with fact invalidation — the right data structure for "routines change"; heavy (Neo4j) — adopt its *model*, not necessarily its deployment.

**Honorable mentions:** super-productivity (timeboxing+idle-detection UX), Mindwtr (GTD pipeline on every platform), Huginn (trigger→digest grammar), Vikunja (OpenAPI task UI), goose (fallback runtime), agent-zero (memory-as-folders), karakeep (capture inbox), Habitica (habit API), org-mode (planning semantics to copy), Frona (sandboxing model), mem0 (simple embeddable memory), claude-adhd-skills micro-genre (skills-format evidence, 53k★ demand signal).
