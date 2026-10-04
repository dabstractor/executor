# Behavior learning & failure handling (Questions D + G)

## The core requirement restated

The system must represent `planned → attempted → completed/failed/deferred` as *data* and let
that data change future behavior. Of everything surveyed, only evidenceofLife-v2 builds this
substrate (and then never fully exploits it); the agent projects have none of it. This is
Executor's second genuinely open territory.

## Observed facts vs agent inference (the non-negotiable split)

Two tables, two write paths (see task-model.md):

- **Observed facts** — append-only `task_event` + `session` rows written by *user actions and
  system transitions only* (tap started, tap done, missed, deferred-with-reason, timer data).
  The agent can never write these. (CORE's rule "only the user marks Done" generalized.)
- **Inferences** — `inference` rows: agent-generated hypotheses with `evidence_json`, status
  `draft`, and explicit promotion to `confirmed` by the user (eol's ghost-preview pattern;
  OpenClaw's origin-class gating is the same idea at memory level).

Inferences never silently modify selection coefficients, memory canon, or profiles. A confirmed
inference becomes: a preference row, a curated-core line, a task/routine edit, or a coefficient
adjustment *the user accepted*. Dismissed inferences are kept (negative knowledge: don't
re-propose the same hypothesis for N weeks).

## What gets learned, from what evidence, with what effect

| Learning target | Evidence (observed) | Conservative rule | Effect |
|---|---|---|---|
| Duration estimates | session.actual_seconds by tag/task-type | median with ≥3 samples; pad to personal p80 (planning fallacy: Buehler 1994; Roy 2005) | size_minutes defaults; schedule padding |
| Time-of-day/context fit | sessions by hour/segment/context | plurality ≥50% with ≥3 samples (eol rule) | selection features; "morning task" hints |
| Initiation difficulty | offered→started vs offered→ignored ratios per task/tag | ≥4 non-starts triggers diagnosis, not more reminders | re-slotting, expansion regeneration, body-double offer |
| Task friction (vagueness) | expansions regenerated repeatedly; started-but-abandoned rate | agent flags as draft inference | decompose-again, renegotiate, or someday demotion (BuJo triage) |
| Recurring blockers | blockers field + if_blocked usage | clustering is LLM work → draft inference | pre-emptive step in expansions ("check X first") |
| Strategies that worked | completed sessions following a specific first-action pattern | surfaced in reviews, never auto-applied | expansion templates ("start this like last time") |
| Stale goals/projects | no events in N days; someday drift | deterministic aging + review surfacing | weekly review asks the BuJo question: still worth it? |
| User preferences (identity-level) | accumulated confirmations | promotion gated like OpenClaw curated core | EXECUTOR.md lines |

Reference implementations to copy: eol's `getSmartDuration` fallback ladder (exact title →
same tag → static defaults), `schedulingProfile` gates; Hermes's background-review prompt rules
(what not to learn); OpenClaw's deterministic promotion gates.

## Expansion quality as a learned loop (Question F support)

"Get the basement ready" → executable plan is an LLM job, but its *inputs* and *evaluation* are
data:

- **Inputs (context assembly, deterministic):** task + project page (why/outcome/log), last
  expansion + what happened (which steps completed), retrieved notes (FTS5 over project/journal),
  inventory hints if present, current context (time/energy/tools).
- **Output contract (schema-validated):** current state; why it matters (≤2 lines from project
  page); what you need; **first action ≤2 minutes, physical**; 3–7 steps; stop condition;
  estimated effort; likely blockers; if-blocked plan. (Nudge's capture-time sections + GTD
  natural planning + CBT task-shaping, operationalized.)
- **Evaluation:** every step gets its own completion events; abandoned-at-step-k patterns feed
  the next regeneration ("you always stall at step 3 — split it?"). Expansions are versioned
  rows, regenerable, never silently overwritten.
- No surveyed system does this; the closest patterns are nudge's prose sections and CORE's Page
  zones. The architecture above is what makes it *reliable*: grounding in retrieved context +
  validation + regeneration from outcomes.

## Failure handling (Question G): "we've tried this four times"

Rule-based triggers over the event log, escalating through a diagnosis ladder — reminders are
the *last* thing repeated, not the first:

1. **Deferral count ≥2, age <7d:** surface friction at next planning ("want to shrink it?")
2. **Non-starts ≥4 (offered & ignored):** STOP reminding. Open a diagnosis dialog (agent-driven,
   user-decides): too big? → regenerate expansion smaller; wrong time? → re-slot to learned
   segment; actually blocked? → waiting_on + unblock task; don't want it? → someday/drop with
   the BuJo question; scary/boring? → 10-minute entry step + body-double session offer
   (CBT task-shaping; Solanto/Safren operational core).
3. **Deadline at risk while stalled:** escalate to renegotiation support (draft the message,
   propose new date) — commitments get *managed*, not nagged.
4. **Over-planning / unrealistic days:** plan-vs-done ratio computed nightly; when
   completion <50% for 3 days, the morning brief shrinks the proposal (1 MIT) and says why.
5. **Notification fatigue:** measured as notification→action latency; rising latency or
   escalating ignore-rate auto-reduces cadence (mHealth habituation findings), batches, and
   varies phrasing. A quiet channel is a config bug, not a user failure.
6. **Abandoned projects:** no events for 30d + review miss → parking-lot flow (archive with a
   one-line epitaph + restart conditions) — history immutable.

The "diagnose, don't repeat" behavior exists nowhere in the surveyed OSS; the pieces
(event log, thresholds, agent dialogs) are all buildable on the recommended model.
