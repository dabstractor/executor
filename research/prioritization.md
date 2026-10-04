# Prioritization — "what should I do right now?" (Question E)

## How existing systems decide

| System | Selection mechanism | Assessment |
|---|---|---|
| DailyOS | Deterministic score: priority enum (urgent 60/high 40/normal 20/low 5) + overdue 100 / due-today 70 / has-deadline 20 + proximity; top 5 | Sound but static; no deferral/energy/estimate-fit terms |
| Taskwarrior | Configurable urgency coefficients (due, age, project, tags, priority…) — 20-year reference implementation | Best *prior art* for an extensible deterministic score |
| nudge | None — 100% LLM discretion over markdown | The anti-pattern: unexplainable, untestable, unlearnable |
| adhd-day-planner | Fixed processing queue in prompt: high → overdue → today → inbox → untriaged → med → low, cap 30, then LLM assembles day with anchors + tax | Good corpus, zero enforcement |
| evidenceofLife | Median-duration auto-schedule + plurality day-segment preference; user drags; ghosts need acceptance | The only one where *history changes the schedule* |
| Motion/Reclaim et al. | Full auto-scheduling; market forked on decide-for-user vs guide-user | Decide-for-user fails ADHD derailment recovery |
| CORE | None — tasks are agent work units | — |

**Verdict on static scores: insufficient.** Three independent reasons: (1) they can't see
*initiation likelihood* — the binding constraint in ADHD (task started 4× never at 8pm is a
fact a score must weight); (2) they can't do *context collapse* — the moment of choice needs
context ∧ time ∧ energy ∧ tools (GTD's four criteria) fused with learned personal data;
(3) they can't explain themselves in user language — the ADHD value is the explanation
("start this because…"), which requires narrative on top of features.

## The recommended selection algorithm (three layers + feedback)

### Layer 0 — deterministic filter (SQL, no LLM)

Candidate set = tasks where: status ∈ {ready, doing} (not blocked/waiting), `defer_until`
passed, context_tags ⊆ current context (time/place/tools from phone state or manual toggle),
`energy ≤ current energy`, `size_minutes ≤ available window` (+ padding), WIP cap respected
(≤3 doing, personal-os's enforced cap). Plus due commitment deadline pressure. This is
Taskwarrior-style: explainable, testable, cheap, runs every minute if needed.

### Layer 1 — deterministic score (still SQL; Taskwarrior-extensible coefficients)

Terms (all from structured facts, none from vibes):
- due-ness (commitment deadlines weigh more than soft dues)
- deferral pressure: count + age of last deferral (BuJo signal) — *capped* so it saturates
  instead of guilting forever, and routed to diagnosis after threshold (behavior-learning.md)
- project momentum: days since project's last completed task (stale → rise)
- estimate-fit and time-of-day: learned segment preference (eol plurality rule) and learned
  duration medians
- initiation likelihood: P(start | hour, context, task tags) from session history — tasks with
  a bad local track record get *scheduled to their proven slot*, not punished
- friction/size: smaller wins ties (progress principle — Amabile)
- user-stated focus of the day (morning MIT confirmation)

### Layer 2 — LLM re-rank + narration (bounded proposer)

Top-K (≤10) from layers 0–1 + compact context (project why, last attempt note, expansion first
action) → LLM returns ordered `now / next / later` with one-line rationale each. Validated:
returned ids must be ⊆ candidate set (DailyOS discipline). The rationale is what the user
actually reads; the ranking can't invent candidates or hide commitment pressure (code checks).

### Feedback loop

Every presentation logs an event: offered → (started | not_now | ignored). Ignored-offers are
the *most* informative signal (selection was wrong or initiation failed) — this is the
planned→attempted→completed chain from behavior-learning.md. Coefficients stay fixed (no online
learning magic); the *features* (segment preferences, durations, initiation probabilities) are
learned conservatively from events.

## Daily shape (from the productivity + ADHD evidence)

- **Morning:** agent proposes 1–3 MITs from layer-1 top + today's constraints; user confirms
  with one tap (guide, don't decide). Manufactured deadlines for important-not-urgent work
  (importance doesn't activate ADHD brains; structure does).
- **During the day:** rolling auto-reshuffle when a timebox is missed — never make the user
  re-plan (time blocking's failure mode). The universal offer: "start a 25-min session on X"
  with the first action attached.
- **Anchors:** protected timeboxes for meals/meds/appointments (adhd-day-planner's anchor
  heuristic; also anti-hyperfocus-before-appointments guard).
- **Not a rigid schedule:** blocks are proposals with flexibility policy; the day is a view
  over events, rebuildable at any moment.
