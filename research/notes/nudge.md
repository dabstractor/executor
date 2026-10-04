# Nudge (thatsjet/nudge-app) — Architectural Reconnaissance

Investigated 2026-10-04. Local clone at `/home/dustin/src/nudge-app` (shallow, depth-1). Target of eval: candidate base/donor for **Executor** (persistent external executive-function layer, ADHD user, phone-accessible, behavior-learning).

---

## 1. Overview

Nudge is a small single-author open-source **Electron desktop app** — "a gentle productivity companion for ADHD brains" (README.md). It is a chat-first conversational wrapper around an LLM that reads/writes a local folder of plain markdown files ("the vault"). Core mantra: *"Starting is success, completion is optional."* It is explicitly **not** a task manager or planner: it is a "supportive presence that surfaces small, concrete next steps."

- Author: Jet Anderson (thatsjet), developed heavily with Claude Code/OpenClaw (repo has `.claude/skills/`, `CLAUDE.md`, `ui-rewrite.md` audit doc, branch `ui-updates-openclaw`).
- GitHub: created **2026-02-10**, last push **2026-03-25** (~6 weeks of active development, then quiet). 23 stars, 3 forks, 5 open issues, not archived. Version v0.1.8.
- Local git log shows a single commit (shallow clone): `39aa244 Merge pull request #46 ... feat: scheduled nudges + UI polish (v0.1.8)` dated 2026-03-24/25. API history shows rapid iterative small commits (PRs #43, #46), many fixing real LLM misbehavior.
- Size: ~6,240 lines of TS/TSX total; main process ~2,680 lines; the "brain" is a ~350-line markdown system prompt (`app-bundle/system-prompt.md`).

## 2. Tech Stack & Repo Stats

- **Electron 40** + **React 19** + **Vite 7** + TypeScript 5.9. Vitest tests (providers, utils, some components).
- `@anthropic-ai/sdk` ^0.74, `openai` ^4.104 (custom OpenAI-compatible base URL supported → Ollama etc., issue #44 asks for docs), `keytar` (OS keychain), `electron-updater`, `react-markdown`.
- **No SQLite. No database. No server. No git integration in code. No telemetry.** Settings are a JSON file; chat sessions are JSON files in Electron `userData` (outside the vault).
- macOS-centric (electron-builder, hiddenInset title bar); README only documents macOS install.

## 3. ACTUAL Data Model

### 3.1 The vault (plain markdown, default `~/Nudge/`, seeded from `default-vault/`)

```
~/Nudge/
├── ideas/*.md          # one file per project/idea, YAML-ish frontmatter
├── daily/YYYY-MM-DD.md # reflective daily log
├── tasks.md            # quick checklist, FIXED sections
├── config.md           # user context/preferences
├── system-prompt.md    # (per README; see disagreement note below)
└── archive/            # archived_tasks.md (date headers) + done idea files
```

**Idea frontmatter** (`default-vault/ideas/_template.md`; interface `IdeaFrontmatter`, `src/shared/types.ts:3-11`):

```yaml
---
status: active | someday | paused | done
priority: high | medium | low
type: work | personal
energy: low | medium | high
size: small | medium | large   # <30min / 1-2hr / half-day+
tags: []
started: false
---
# Idea Title
## What is it?
## What does "starting" look like?   # tiny `- [ ]` micro-steps
```

**tasks.md** — fixed sections only; the system prompt *forbids* creating any others: `## Today`, `## Recurring Daily`, `## Recurring Weekly`, `## Later`; items are `- [ ]` checkboxes; priority is an inline tag: `- [ ] Fix login bug #high` (no tag = medium).

**daily logs** — prose sections: `## What I chose to work on`, `## Wins`, `## How it felt`, `## What was hard`, `## Notes` (prd.md:264-284).

**config.md** — `## About Me`, `## Mantra`, `## Energy Patterns` (Morning/Afternoon/Evening), `## Preferences`, `## Current Focus Areas`.

Frontmatter parsing is a naive regex — `main.ts:198-207` (`/^---\r?\n([\s\S]*?)\r?\n---/`, split on first colon, all string values). `vault:read-frontmatter` is used by the FileEditor UI only; **the agent itself never reads structured frontmatter — it just reads raw file text**. The typed `IdeaFrontmatter` is essentially vestigial.

### 3.2 App state (outside vault, not user-visible/syncable)

- `userData/settings.json` — vaultPath, theme, model, activeProvider, nudges config, baseUrls, fallback API keys.
- `userData/sessions/<uuid>.json` — one file per chat session: `{id, title, createdAt, updatedAt, starred, messages: [{id, role, content, timestamp}]}` (`main.ts:508-560`). Full conversation history persists, but the agent does **not** load prior sessions as context — each session starts fresh with only the vault as memory.

## 4. Architecture & Control Flow

**Everything interesting is the LLM.** Deterministic code is: Electron chrome, file I/O sandboxing, one string-processing tool (`archive_tasks`), time-resolution, and a dumb scheduler.

### 4.1 Deterministic application code

- `main.ts` (1,114 lines): window/menu, all IPC handlers (`vault:*`, `settings:*`, `sessions:*`, `api:*`, `updater:*`), tool execution, nudge firing.
- `agenticLoop.ts` (74 lines): `runAgenticLoop` — stream model → if toolCalls: execute each via `processToolCall`, append provider-formatted tool results, loop; stop when no tool calls. **No iteration cap, no per-tool approval.** Only escape is user cancel (abort).
- `providers/tools.ts` (103 lines): 8 neutral tool defs — `read_file, write_file, edit_file, list_files, create_file, move_file, archive_tasks, update_nudge_settings` — all params `Record<string, string>` (stringly-typed).
- `providers/anthropic.ts` / `openai.ts`: provider adapters mapping neutral defs to Anthropic tool_use vs OpenAI function calling, incl. streamed tool-call assembly (`openai.ts:289-336`).
- `archive_tasks` (`main.ts:553-620`): real deterministic logic — scans lines, tracks `inTodaySection` via heading regex, moves `- [x]` items to `archive/archived_tasks.md` under `## <date>` with same-day header dedup, never touches Recurring sections.

### 4.2 LLM-driven behavior (the actual product)

System prompt = `app-bundle/system-prompt.md` (bundled, read-only copy shipped with app) + injected `config.md` contents + current date/time + vault path. Assembly duplicated in `App.tsx:188-215` (interactive) and `handleNudgeFire` (`main.ts:781-805`, scheduled). Key prompt content (abbreviated):

- **Philosophy**: "warm, low-pressure… never guilt, never nag… celebrate every start… biased toward action… suggest small things."
- **Morning review** ("start my day"): read config → yesterday's log ("note carryover but don't guilt") → scan ideas/ → tasks.md → **reset Recurring Daily checkboxes (uncheck for new day), Weekly on Mondays** → "Surface 3-5 approachable suggestions as `- [ ]` checkboxes — prioritize high-priority items first, then filter by energy level for the time of day and size. Low-priority items should only appear if nothing higher-priority fits." → user picks → create daily log.
- **Time window** ("I have 30 minutes"): parse window, filter by size/energy, "Suggest 1-3 options with a concrete first step… emphasize starting, not finishing."
- **Idea capture**: minimal clarifying questions; infer priority from cues ("urgent/blocking/deadline"→high); create idea file "with at least one tiny step"; "use the user's words, not yours" (anti-embellishment).
- **Task capture**: add to tasks.md, "No questions — just add and confirm."
- **End of day**: strict two-message protocol (one gathering question, one closing, then silence); silent research reads; write daily log; archive checked Today items; "Do not mention incomplete tasks."
- **Vault structure rules**: never create directories/sections; fixed routing table (task→tasks.md, project→ideas/, etc.).
- **Task formatting rule**: everything actionable must be `- [ ]` checkboxes — "People with ADHD get a real dopamine hit from checking things off."
- **Boundaries**: cannot set timers/alarms, calendar, email/messages, open URLs/apps, access internet, run code.

### 4.3 "What should I do right now?" — answer to the brief's question

**There is no ranking/selection code.** Selection is 100% LLM discretion guided by prompt instructions (priority → energy-for-time-of-day → size filter over markdown it just read via tools). The only deterministic inputs are the frontmatter vocabulary the LLM is told to respect and the checkbox-reset rules it is told to apply. No scoring function, no queryable index, no recency/postponement data exists anywhere.

### 4.4 Micro-step generation ("how do I get started?")

Also purely prompt-driven: the "What does starting look like?" section in the idea template + capture-time instruction to include "at least one tiny step". There is no on-demand task-decomposition engine, no "expand this task" tool, no validation that steps are concrete. Quality depends entirely on the model. Nothing at Executor's "contractor bags / grow tent" granularity is enforced or persisted beyond prose in the idea file.

## 5. Memory / Persistence / Learning

- **Persistence**: vault markdown only (+ session JSONs for transcripts). Everything survives restarts; files are user-editable; built-in FileExplorer/FileEditor UI for direct editing; `vault:changed` event refreshes UI on any agent write.
- **Learning from behavior**: **none.** No structured record of postponement, deferral counts, estimate vs. actual, initiation failures, or completion. Daily logs are free prose the LLM may skim at the next morning review. Frontmatter (energy/size/priority) is written once at capture and never revised or scored against outcomes. `started` flag exists but nothing reads it programmatically. No statistics, no adaptation, no feedback loop whatsoever in code or prompt.

## 6. Scheduling / Proactivity

`nudgeScheduler.ts` (121 lines): `setInterval` every 60s while the Electron app runs.
- Three fixed nudges: morning (default 08:00), midday (11:00), endOfDay (15:00) — **all disabled by default**; user enables by chatting ("set my morning nudge to 8:30" → LLM calls `update_nudge_settings`; `+N` relative times resolved by code, not the model — `main.ts:653-660`).
- Fire condition is **exact string equality** `currentTime === config.time` (`nudgeScheduler.ts:105`) — if the app is closed/asleep at that minute, or the tick lands at HH:MM:59, the nudge is **silently missed**; no catch-up, no persistence of missed nudges.
- Do Not Disturb suppresses all; auto-resets at EOD time or midnight.
- On fire: `handleNudgeFire` (`main.ts:770-864`) runs a **headless agentic loop** with a synthetic user message `[Nudge triggered]` and a type-specific prompt addendum ("suggest one small thing to start with. Keep it to 1-2 sentences"), saves a new session, shows an **OS notification** with the first 200 chars; clicking focuses the app on that session.
- Proactivity exists only in this narrow, desktop-app-running sense. **No phone push, no background service, no server.**

## 7. Integrations

None. Deliberately: system prompt lists everything it cannot do (timers, calendar, email, messages, notifications beyond nudges, URLs, internet, code execution). Only external touchpoints: LLM provider API (Anthropic/OpenAI/custom), OS keychain (keytar), electron-updater → GitHub releases.

## 8. Security / Approval Model

- Path containment: `resolveVaultPath` (`main.ts:127-135`) and per-tool checks use `resolved.startsWith(vaultPath)` — a prefix check, not a boundary-safe comparison (classic `startswith` weakness; `path.resolve` makes trivial escapes hard but symlink traversal is unguarded).
- **No approval gate**: the LLM auto-executes every tool call, including `write_file` full overwrites of any vault file. No diffs, no confirmation, no undo, no versioning. `edit_file` is `content.replace(oldText, newText)` — **replaces first occurrence only**, single match, silent partial application if text repeats (e.g. two identical checkbox lines).
- No backups or history in code. README's "sync them with git" is advice to the *user*, not a feature — **there is no git code anywhere in src/** (rg for git/exec/child_process: zero hits in main logic).
- API keys: keytar w/ plaintext settings.json fallback; keychain-to-fallback migration code exists. Electron hardening is reasonable (contextIsolation on, nodeIntegration off, external links via shell).
- SECURITY.md present; issue #14 (Semgrep CI) open, unactioned.

## 9. UI & Phone/Remote Access

- Single Electron window: Sidebar (sessions) + ChatPanel + FileExplorer/FileEditor + Settings + 10-step Onboarding (provider → API key → vault path → "About You" profile → mantra etc.).
- Chat: streaming, tool-use indicators, markdown rendering, suggestion chips on empty state.
- **No server, no web UI, no mobile client, no API.** "Accessible remotely from a phone" is simply not addressed. The always-on-home-computer + phone-access architecture Executor needs would have to be built from scratch; Nudge's Electron GUI *is* the harness (scheduler, notification, provider config all live in the desktop main process).

## 10. README vs. Code Disagreements

1. README says the system prompt is a file "in your vault" (`system-prompt.md`, "fully editable"); code loads it from the **app bundle** (`app:get-system-prompt`, `main.ts:330-339`) and `default-vault/` contains **no** system-prompt.md. Users cannot actually edit the personality through the vault (only config.md is injected).
2. README: "sync them with git, back them up however you want" — implies git sync; no git functionality exists.
3. README implies vault holds "all your data"; chat transcripts actually live in `userData/sessions/` outside the vault (unportable, excluded from any vault backup/sync).
4. README "Getting Started" clone URL is a placeholder (`your-username/nudge-app`); issue #47 files exactly this.

## 11. Strengths

1. **The system prompt is a genuinely excellent, battle-tuned ADHD interaction spec** — the most valuable artifact in the repo. Guardrails: never guilt, no streaks/scores, "respect 'not today' without pushback," minimal clarifying questions at capture, checkbox-dopamine formatting, fixed sections to stop LLM drift, two-message EOD cap, "use the user's words" anti-embellishment, "never mention skipped days or gaps." Commit history shows these rules were added in response to real failure modes (Claude creating idea files from casual mentions; tool errors killing streams; model time arithmetic errors → `+N` resolution moved into code).
2. **Local-first, plain-markdown vault as source of truth** — user owns data; rigid schema-in-prose; deterministic archive operation.
3. **Clean minimal agentic loop + neutral cross-provider tool abstraction** (74-line loop; neutral tool defs mapped to Anthropic/OpenAI streaming APIs) — a good reference implementation to copy.
4. **Capture-time micro-steps** ("What does starting look like?") — the right instinct for ADHD initiation support; the frontmatter vocabulary (status/priority/type/energy/size/started) is a decent seed ontology.
5. Honest privacy posture (no telemetry/accounts; SECURITY.md; keychain).

## 12. Weaknesses (vs. Executor's requirements)

1. **No behavioral learning whatsoever** — nothing tracks postponement, underestimation, vagueness, initiation failure, contexts of good/bad performance. This is Executor's central requirement and it is absent at every layer (no data captured, no analysis, no adaptation).
2. **No deterministic planning/selection** — "what should I do now" is LLM discretion over raw markdown on every query; unverifiable, un-testable, un-tunable, token-expensive (re-reads vault files every session).
3. **Desktop-only, no remote access** — Electron app must be running for nudges; exact-minute fire condition silently misses; no phone story at all.
4. **No git/versioning/undo despite markdown vault** — LLM can overwrite any file with zero guardrail; first-occurrence replace is fragile.
5. **Memory architecture gap**: transcripts locked in app-data JSON; no cross-session agent memory; only "memory" is what the LLM re-reads from vault prose.
6. Small, single-maintainer, ~6-week-old, already quiet since 2026-03-25; open install-blocking bug (#48 "Electron Framework not found"); 23 stars — bus factor 1.
7. Stringly-typed tools, no schemas, no typed results; no loop bound; no cost controls.

## 13. Surprising Design Decisions

- The "product" is a markdown prompt; the TypeScript exists largely to be a safe-ish file-I/O sandbox for an LLM. Product iteration happens by editing prose (commit stream confirms: most feature work = prompt/tool-def edits).
- Scheduled nudges run a full headless agentic loop and persist the transcript as a chat session — proactive messages are "just" synthetic user turns.
- Chat sessions deliberately excluded from the vault (data separation), contradicting README's "all your data lives in markdown."
- Vault structure is deliberately frozen (no new dirs/sections ever) — schema stability via prompt prohibition rather than validation.
- prd.md (37KB) embeds the author's real dogfooding vault content (projects "SSDLC Policy", "Morse Magic", "Wake Up Light") — useful as realistic sample data.
- Development was visibly Claude-Code/OpenClaw-assisted (`.claude/`, `CLAUDE.md`, `ui-rewrite.md`, openclaw branch).

## 14. Unfinished / Vaporware

- No TODO/FIXME markers in src/. Open issues: #48 (packaged app won't launch for at least one user — unresolved 6+ months), #47 (install docs), #45 "Create connection profiles" (feature idea), #44 Ollama docs, #14 Semgrep CI.
- `ui-rewrite.md` audit recommendations only partially landed in v0.1.8.
- The editable-in-vault system prompt (README feature) is not implemented (see §10.1) — reads as planned-but-unwired.
- No roadmap artifacts for the things Executor needs (learning, ranking, remote access) — they were never started, not merely unfinished.

## 15. VERDICT for Executor

**Pattern-donor / selective-reuse. Do not fork or build on.**

Reasons:
- **Architecture mismatch at the root**: Nudge is a GUI-desktop chat app whose entire intelligence is prompt-improvised per query. Executor needs a headless always-on service with phone access, a real data layer (SQLite), deterministic selection/ranking, and a behavior-learning loop. Nothing of that exists here to build on — the useful parts are all documents and patterns, not infrastructure.
- **Learning is absent** — Executor's core differentiator would have to be invented from zero anyway.
- Single-maintainer, 6-week-old, already-dormant Electron app; adopting it as a base inherits Electron packaging constraints (#48) for a use case (server + phone) Electron actively fights.

**What to steal (ranked):**
1. `app-bundle/system-prompt.md` wholesale as a v0 behavior spec for Executor's conversational layer (ADHD tone rules, morning-review/time-window/capture/EOD flows, checkbox formatting, anti-drift vault rules). It is several iterations of real-world prompt hardening for exactly Executor's user persona — do not rewrite from scratch; adapt.
2. The **idea-file schema**: frontmatter vocabulary (status/priority/type/energy/size/started) + "What is it?" / "What does starting look like?" sections + capture-time micro-steps + "use the user's words" constraint. Extend with deadlines, estimates-vs-actuals, deferral counters, and context tags for the learning loop.
3. The **minimal agentic loop + neutral tool definitions** (`agenticLoop.ts`, `providers/{tools,anthropic,openai,registry}.ts`) as the reference for a provider-agnostic tool-calling core — small, clean, test-covered; add iteration bounds and typed schemas.
4. The **`archive_tasks` pattern** (LLM decides, deterministic code executes structured file surgery) and the `+N` time-resolution pattern (never let the model do clock arithmetic) — both directly transferable.
5. The **commit history as a failure-mode catalog**: LLM creating files from casual mentions, tool errors crashing streams, scheduler not ticking on start, exact-time misses — pre-solved problems to design around.
