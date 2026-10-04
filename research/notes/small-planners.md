# Small ADHD Planners: adhd-day-planner & evidenceofLife-v2

Reconnaissance for **Executor** (persistent external executive-function agent). Two repos inspected read-only, code-first. Both are shallow clones (1 visible commit each); git dates are clone-truncated artifacts, noted per repo. GitHub API checked live for both.

---

# PART 1 — adhd-day-planner (github.com/Dthen/adhd-day-planner)

## Overview

**This is not an application. It is an LLM-agent skill**: two bash scripts (~264 lines) plus a 346-line prompt file (`SKILL.md`) that instructs a host agent (built for **Hermes** — `SKILL.md:12` "Hermes generates"; paths like `~/.hermes/skills/productivity/adhd-day-planner/`) to plan a day inside an **Obsidian vault**. The entire repo is 8 files, 948 lines. Pipeline:

```
inbox.md → prepare.sh (gather+bucket, JSON out) → LLM writes daily plan → cleanup.sh (drain inbox)
```

## Tech stack & repo stats

- bash + jq + curl + python3 (JSON escaping only). No app language, no DB, no tests, no CI, no package manifest.
- 0BSD license. Single visible commit `a2a5d65` 2026-05-16 "Revert DAILY_FOLDER default from to-do back to daily" (shallow clone; real history exists upstream).
- GitHub API: **0 open issues**. Personal tool maturity; no community.

## ACTUAL data model

Everything is Obsidian-Tasks markdown lines — there is no schema. State lives in three files:

- `~/notes/inbox.md` — capture file. Task line grammar: `- [ ] 🔺 Title 📅 2026-02-21 🔁 every week on Thursday ➕ 2026-02-10`, plus ✨ habit, 🧠 vague markers.
- `~/notes/daily/YYYY-MM-DD.md` — plan output. Sections: Weather, At a Glance, The Plan (timestamped checkboxes), Stretch Goals, Energy Notes, Deferred Tasks, and `### 📅 Upcoming Tasks` (icebox, pre-written by prepare.sh).
- `scripts/context.md` — free-text user context (goals, habit blocks with durations, energy patterns, constraints, biological anchors). Example ships with Spanish/French-input habit blocks and "Variable — caffeine-dependent, mornings need warmup" as the entire energy model.

## Architecture & control flow — deterministic vs LLM

**Deterministic (`scripts/prepare.sh`, 243 lines):**
- Gathers first **10** unchecked inbox items (`prepare.sh:25` `MAX_INBOX=10`) + all unchecked tasks from the most recent daily note (today → `replan`; else latest note whatever its date → `continue`; none → `fresh`) (`prepare.sh:38-50`).
- Buckets by regex: 🔺/⏫/🔽 priority; `📅 date` → future / overdue (< today) / due-today; else untriaged (`prepare.sh:66-123`). Overdue sorted oldest-first via sed|sort|cut (`:124`).
- Focus queue = fixed concatenation: **high → overdue → due-today → inbox → untriaged → medium → low** (`prepare.sh:127`). Cap 30; overflow + future-dated → icebox written into today's skeleton (`:127-135, :160-190`). This concatenation IS the entire prioritization algorithm — no scoring, no weights.
- Weather: Open-Meteo curl + jq one-liner (`prepare.sh:147`), 6s timeout, degrades to "Weather data unavailable".
- Emits JSON `{date, current_time, mode, weather, context, focus_tasks[], task_counts{...}, inbox_processed}` and saves pulled inbox lines to `/tmp/inbox_pulled.txt`.

**LLM (the whole planner):** `SKILL.md` is the algorithm. All scheduling intelligence is prompt instructions:
- **ADHD Tax lookup table** (`SKILL.md:85`): 15→20, 30→40, 45→60, 60→75, 90→115 min. **Not implemented in code** — the LLM must apply it to its own base estimates; nothing verifies compliance.
- 10-min `☕ Break (Transition)` between every task; **Momentum Starter** (easiest win first, `:97`); **Biological Anchors** (60-min meal breaks ~12:30/18:30, never overbooked, `:108`); **Anti-Waiting-Room Protocol** (appointments locked to exact time, 10-min prep break before, only low-friction tasks in the 45-60 min gap to prevent pre-appointment hyperfocus, `:102`); **Zero-Drop Policy** (every task lands in timeline / stretch / deferred, `:157`); 🧠 vague tasks get a 20-min breakdown slot or defer; generation starts at now+15min rounded to 5.
- Interactive phases: vague-task clarification ("what's the actual first step?") and priority clarification (asks when two similar-urgency tasks conflict; "when uncertain, ASK rather than guess").
- Proactive inbox capture: agent watches conversation for "I need to…" statements and offers `echo "- [ ] TASK ➕ $(date +%Y-%m-%d)" >> ~/notes/inbox.md` (`SKILL.md:339-346`).

**Deterministic (`scripts/cleanup.sh`, 21 lines):** `grep -v -F -f /tmp/inbox_pulled.txt` — removes pulled lines from inbox unconditionally.

## Memory / persistence

Markdown only. No history beyond what survives in old daily notes. **No learning of any kind**: estimate inflation never adapts, no actual-vs-planned comparison, no postponement tracking (a deferred task is just an unchecked line that gets re-gathered tomorrow).

## Scheduling / proactivity

None. Planning happens only when the user invokes "plan my day" in an agent session. No daemon, no reminders, no push.

## Integrations

Obsidian vault (read/write), Open-Meteo weather. Nothing else.

## Security / approval model

None needed (local files), but notable trust posture: SKILL.md instructs the agent to write the plan **without asking** ("Write the note, don't ask… breaks the user's flow", `SKILL.md` Workflow Adherence) — friction-optimized over confirmation. Hard rules exist for repo hygiene (never commit without consent; never touch `.obsidian/` config).

## UI & phone/remote access

None. Obsidian + terminal are the UI. Mobile access = Obsidian sync.

## Strengths

- The **densest written specification of ADHD scheduling heuristics** I've seen in 948 lines: ADHD tax, transition costs, momentum-first ordering, biological anchors, anti-hyperfocus-before-appointments, zero-drop guarantee, non-judgmental coaching voice, task NLP normalization (recurrence + implied-due-date extraction anchored to ➕ date).
- Clean deterministic/LLM split conceptually: script does mechanical gather/bucket; LLM does narrative + judgment.
- Honest scoping — interactive clarification is mandated rather than guessed.

## Weaknesses / failure modes (code-verified)

- **Zero-drop is a promise, not a mechanism**: `cleanup.sh` deletes inbox lines even if the LLM run failed or dropped tasks — data loss on LLM failure is silent.
- **Replan destroys the morning plan**: `prepare.sh:160-190` overwrites today's note with a skeleton (completed tasks + icebox only) *before* the LLM regenerates; an interrupted run leaves the day's plan gone.
- ADHD tax compliance, time arithmetic, and rule adherence are entirely unverified LLM behavior.
- `continue` mode pulls from the *most recent* note regardless of date; no dedup between inbox and carried-over lines (a task can enter the pool twice via inbox + yesterday's note).
- No energy/context modeling in code — energy is a sentence in context.md.
- Single-user assumptions hardcoded (`~/.hermes/...` paths in SKILL.md contradict README's generic install instructions — a README/code disagreement).
- No tests; no way to regression-check prompt behavior.

## Unfinished / vaporware

Nothing half-built — because almost nothing is built. The "planner" is a prompt. All learning/feedback-loop ideas from the README's framing (deferred-task rotation is the only one realized, mechanically).

## VERDICT for Executor: **PATTERN-DONOR (prompt-level), NOT code reuse**

- **Steal the SKILL.md heuristic corpus** almost verbatim as Executor's planner-persona spec: ADHD-tax table (but enforce in code with real arithmetic, not LLM compliance), transition breaks, momentum starter, biological anchors, anti-waiting-room protocol, zero-drop policy (but enforce mechanically — eol's ghost-preview/accept pattern below shows how), vague-task 🧠 protocol, and the proactive capture reflex.
- **Do not fork**: no learning loop, no decomposition persistence (breakdowns happen in chat and evaporate into plan lines), no runtime, markdown-only state. It demonstrates the ceiling of prompt-only planning: all judgment, zero feedback.

---

# PART 2 — evidenceofLife-v2 (github.com/Cyriellewu/evidenceofLife-v2)

## Overview

A serious, production-shaped **personal life-tracking SPA**: plan on a visual day timeline, run focus timers, capture "moments" (notes/photos/places), track dues/habits, review planned-vs-actual, browse history by calendar and map. Bilingual EN/中文. ~52,700 LOC TypeScript across `src/` + `supabase/`, 30 SQL migrations, 30 vitest suites, Playwright smoke, Deno edge-auth tests. README claims ~30-40 early users. Explicitly AI-assisted (Cursor/Codex) with human-gated merges — and it shows in the unusually disciplined docs (`llms.txt`, `AGENTS.md` agent rules, `docs/oss/` runbooks).

## Tech stack & repo stats

- React 18 + TypeScript (strict) + Vite 5 (SWC) + Tailwind 3 + shadcn/ui + TanStack Query + React Router 6; Recharts, Leaflet, date-fns.
- Supabase: Postgres with RLS on every personal table, Auth, Storage (private `moment-photos` bucket, signed URLs), Deno edge functions: `geo`, `smart-input`, `life-replay`, `link-preview`, `image-proxy`, `google-calendar-auth/callback/sync`.
- Static hosting (Vercel). Latest visible commit `01e1141` 2026-10-03 (PR #53 merge, cursor[bot]). Open GitHub issues: mobile layout fixes, a11y labeling, dep bumps, test coverage — active maintenance, no architectural rot visible.
- `package.json` v0.1.0; no tagged release yet (draft notes only in `docs/oss/`).

## ACTUAL data model (verbatim where short)

Core tables (migrations 20260213…20260730; RLS `auth.uid() = user_id` on all):

- **`todos`** — the workhorse. `id, user_id, title, date TEXT -- yyyy-MM-dd, time_segment TEXT DEFAULT 'anytime' (morning|afternoon|evening|anytime), progress INTEGER 0-100, is_completed, due_date, sort_order, tags text[], photos text[], note, links jsonb, parent_due_id uuid → todos (STEPS), habit_category, timer_started_at, timer_ended_at, timer_seconds, plan_started_at, plan_ended_at, show_in_recap_daily, is_recurring, recurrence_source_id, promoted_to_habit_id`. Steps are todo rows with sentinel `date='_step_'` + `parent_due_id` (`useTodos.ts:281,511-521`).
- **`moments`** — universal activity/event log: `date, text, emoji, photos[], links, tags[], location_{name,lat,lng,category}, is_special, timer_*`. **Focus sessions are persisted as moments** tagged `todo-session:<parentId>` (`stepSession.ts`, `stepTimelineSegments.ts:59`) — one table feeds timeline, map, memory, recap.
- **`profiles`** — `display_name, avatar_url, bedtime_hour 23, bedtime_minute 30`.
- **`reminders`** — recurring nudges: `interval_days, next_reminder_at, last_reminded_at, is_active`.
- **`due_reminders`** — per-due: `reminder_type ('browser'|'email'), remind_before_minutes DEFAULT 1440, recurring_interval_days, last_notified_at`.
- **`places` / `cities` / `visits`**, **`imported_events`** (ICS), **`google_calendar_tokens`** (service-role only), **`sticky_notes` + `sticky_note_items`, **`link_groups`/`link_sections`**/links.

Notable data-model smells: dates as TEXT `yyyy-MM-dd` (timezone rollover handled by guards + tests instead of types); `'_step_'` magic date; several cross-cutting concerns in localStorage (`work-type overrides`, `recurring-clone-done:<user>:<date>`, rhythm presets) → multi-device drift.

## Architecture & control flow — deterministic vs LLM

**~99% deterministic.** All planning, scheduling, learning, rollover, reminders are pure TypeScript in `src/lib/` + hooks, with tests. The LLM exists only behind two optional edge functions (omit `LOVABLE_API_KEY` → features disable):

1. **`smart-input`** (`supabase/functions/smart-input/index.ts`, 242 lines): Gemini-flash via Lovable gateway. One system prompt, three tool-calls: `classify_input` (free text → `recap|plan|due` + title cleanup, tags, time extraction, `is_habit` only on explicit recurrence keywords, `start_pomodoro` flag), `query_response` ("how long did I work today?"), `modify_item` ("add a time to the last one"). Last-6-turn conversation history + today's events as context. This is voice/text capture triage — nothing more.
2. **`life-replay`**: "You are a poetic life narrator" — 2-3 sentence warm day summary, <60 words.

Everything else — auto-scheduling, carry-over, recurrence cloning, mood (`moodClassifier.ts` — bilingual keyword table → happy/calm/tired/sad/love) — is deterministic code.

## The planning model (answers the brief's specific questions)

**`src/lib/autoSchedule.ts`** — suggestion-based auto-scheduler, "intentionally rule-based (Phase 1)":
- **Duration estimation = real learning**: median of user's recorded `timer_seconds` for exact `tag::title` match (sessions ≥60s count); fallback same-tag median (≥3 samples); fallback `getSmartDuration` static tag→minutes (admin 15, life 20, social 30, health 45, study/work 60…) (`autoSchedule.ts:99-120`).
- **`schedulingProfile.ts` (Phase 2 learning)**: from last 400 todos (`useSchedulingHistory.ts`), derives preferred day-segment (morning/afternoon/evening) per **work type** (deep/shallow/admin/errand/recovery, keyword-classified with localStorage user overrides, `workType.ts`). Conservative: min 3 samples AND ≥50% plurality, else no preference. No new table — derived on the fly, read-only.
- **Two lanes**: `focus` (exclusive — greedily consumes first free span that intersects the preferred window, 5-min snap, 120-min clamp) vs `background` (errand/recovery — may overlap existing blocks, staggered). Tasks that don't fit are **skipped, not forced** (`autoSchedule.ts:140-200`).
- **Ghost-preview interaction model**: autoSchedule returns `Suggestion`s with human-readable reasons ("afternoon (you usually) · ~35m fits here"); UI paints ghost blocks; user accepts/drags/switches lane; **commits only on explicit accept**. Invoked from `PlanTimelineView.tsx:534` with free regions computed against wake 06:00 / bed 23:30 constants (`planTimelineDayBounds.ts` — fixed, not per-user).
- Day windows: `SEG_BOUNDS` morning <12:00, afternoon 12-18, evening 18-24, intersected with day bounds.

**No ADHD tax / estimate inflation**: no inflation factor anywhere; estimates are raw medians. (Roadmap lists "time-estimation reflection — planned vs actual duration over time" as *exploratory, not built*.)

**Planned-vs-actual (the core differentiator)**: same todo row carries `plan_started_at/plan_ended_at` (drag-placed intent) AND `timer_started_at/timer_ended_at/timer_seconds` (actual focus session). `TimeBlock` renders `planStartMin/planEndMin` vs `actualStartMin/actualEndMin` with `hasActual` (`planTimelinePrimitives.tsx:336-340`). Cross-midnight sessions split correctly (`continuesNextDay`/`continuedFromPrevDay`); stale/forgotten timers auto-quarantined (24h guard, `staleTimers.ts`, `stepSession.ts:9`). **But this data is display + duration-learning only** — nothing computes "you underestimate X by 2×" yet, and nothing feeds the gap back into future plans beyond the median-duration effect.

**Deferred work (`carryTodos.ts`)**: unfinished tasks *stay on their original date* (immovable history — an earlier rewrite "scrambled the timeline", per comments at `useTodos.ts:170-176`); today's list merges them read-only via `mergeCarriedTodos` (dedup by title; survivor picked by `workScore = timer_seconds + progress*100000`). Steps/dues/habits excluded from plan carry (own surfaces).

**Recurrence (`recurringTodos.ts`)**: source row `is_recurring` clones onto today on first fetch (localStorage once-per-day guard), steps cloned fresh; triple dedup (recurrence_source_id, self-date, same-title).

**Habits**: `habit_category` + streaks surfaced in Dues view; `promoted_to_habit_id` migration path from dues. Habit *representation* is a category tag + recurrence clone — no flexible routine scripting (no "meds at 08:00 then breakfast" structure).

**Prioritization**: essentially none beyond overdue-first display and user drag. No urgency/importance scoring. This is a *timeline-shaped* planner, not a prioritizing one.

## Memory / persistence

Postgres (RLS), localStorage for prefs, Obsidian nothing. History is first-class: calendar/year views, "On this day", `memoryHorizons.ts` (week/month/year digests: activeDays, focusMinutes, moments, photos, places, kept reflections), JSON evidence export (`exportEvidence.ts`).

## Scheduling / proactivity — the critical gap

**There is no server-side scheduler.** All reminder logic is browser-resident React hooks: `useReminders`/`useDueReminders` (window.setInterval + Notification API), `useLifeReminder` (configurable 1-4h "what just happened?" capture nudge; localStorage throttle; quiet hours 23:00-07:00; optional **ntfy push published client-side** with clickUrl). Consequences: reminders fire **only while a tab is open**; `due_reminders.reminder_type='email'` is stored but **no email sender exists anywhere** (verified: no resend/smtp in functions or src). Day rollover, recurring clones, and running-timer pulls are all client-fetch side effects — with an acknowledged race ("two instances race on localStorage and one can no-op the real carry", `useTodos.ts:161-163`).

## Integrations

Google Calendar (OAuth, HMAC-signed state, redirect allowlist, token table locked to service role), ICS import, ntfy (client-side), optional Amplitude/GA4, Lovable AI gateway. Geolocation → reverse geocode via `geo` edge function → places.

## Security / approval model

Strongest of anything surveyed so far: RLS everywhere, signed URLs for photos, JWT checks in edge functions, service-role isolation for OAuth tokens, SECURITY_MODEL.md + Deno-tested auth helpers. Approval model for planning = the ghost-preview/accept pattern (suggestions never auto-commit). Account deletion/export completeness explicitly listed as unfinished.

## UI & phone/remote access

Responsive mobile-first PWA (manifest, icons, bottom-sheet components), deployable as static SPA → phone access works via browser/PWA with Supabase auth. No offline mode, no local-first. Demo at `/demo-app` on synthetic data (no backend) — nice pattern for safe screenshots/tests.

## Strengths

- **The planned-vs-actual substrate Executor needs**: intent window and actual session on the same row, steps-as-child-rows with their own timers, sessions-as-moments unifying timeline/map/memory. This is exactly the behavioral ground truth an executive-function agent must accumulate.
- **Conservative, tested learning**: median durations + plurality time-of-day preference with minimum-sample gates; derived, not stored; display-only until user accepts. The right shape for "learn from actual behavior" without spooky auto-magic.
- LLM discipline: AI confined to capture-triage and narrative behind optional edge functions; all scheduling deterministic and unit-tested (17 tests in `auto-schedule.test.ts` alone).
- History is immutable-by-design after the rollover bug — carries are merges, not rewrites.
- Ghost-preview accept pattern = a real approval model for machine-generated plans.

## Weaknesses

- No agent: it's a human-driven tool. No "what should I do right now" recommender, no decomposition generation (steps are user-authored only), no prioritization logic, no energy model (mood keywords are passive tags), no ADHD tax.
- No proactivity without an open browser tab — architecture cannot deliver an always-on executive layer.
- TEXT dates + localStorage guards + client-side side-effects = the class of bugs their own tests encode (crossday timers, rollover, clone races).
- Single-user per account; no headless/API access path for an external agent to read/write the plan.

## Surprising design decisions

- Focus sessions persisted as **moments** (the event log), then re-derived onto the timeline via tags — one source of truth for time-usage across four views.
- Scheduler *skips* rather than forces tasks that don't fit (anti-overcommitment).
- `reminders` table stores scheduling rules but no server ever fires them.
- A one-person AI-built app with multi-user auth, RLS, OAuth hardening, and 30 test suites — evidence that an AI-assisted solo build of Executor's scale is feasible.

## Unfinished / vaporware

Email reminders (schema only). Account deletion/export completeness. Live-DB RLS verification. No tagged release. Roadmap's most Executor-relevant items — plan-vs-actual overlay analytics, estimation reflection, short-window (5/15/30-min) planning — are explicitly *exploratory, not built*.

## VERDICT for Executor: **PATTERN-DONOR, with the best data model to copy; not a fork**

- **Copy the schema concepts** into Executor's SQLite: `plan_started_at/plan_ended_at` vs `timer_started_at/timer_ended_at/timer_seconds` on one task row; steps as child tasks (`parent_due_id` + sentinel date) each with own timers; sessions as append-only event log; `progress` 0-100; tags + work-type classification with user overrides; immutable history + read-only carry merge.
- **Copy the learning modules' shape** (median duration by task/tag with min-sample gates; plurality segment preference) and the ghost-preview/accept approval pattern for machine-placed blocks.
- **Copy the LLM boundary**: deterministic scheduler, LLM only for capture triage and narrative, optional/disablable.
- **Do not copy the runtime posture**: browser-resident scheduling/proactivity is precisely what Executor must not do — their reminders-die-with-the-tab problem is the strongest argument yet for Executor's always-on home daemon + push (ntfy) design. Their TEXT-date/localStorage/rollover pain also argues for SQLite with proper timestamps and server-side (daemon-side) side effects.

---

# Cross-cutting takeaways for Executor

1. The two repos are **complementary halves**: adhd-day-planner holds the best *planner heuristics* (prompt-spec), evidenceofLife-v2 holds the best *behavioral data model + conservative learning*. Neither has an always-on runtime; both fail Executor's "proactive from a home computer, reachable from phone" requirement in opposite ways (no runtime at all vs. browser-hostage runtime).
2. Both encode the same insight Executor should keep: **planning artifacts must be durable files/rows the user can inspect** (markdown daily notes; Postgres rows) — and machine suggestions should preview, then commit on accept.
3. Verification matters: the skill's unenforced zero-drop/ADHD-tax promises vs eol's tested deterministic scheduler shows the failure mode of trusting the LLM with bookkeeping. Executor should compute durations, inflation, and drop-guarantees in code and let the LLM only do decomposition/selection/narrative.
