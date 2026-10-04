# Productivity Methodologies → Executor: Mechanism Survey

**Child:** prod-research · **Date:** 2026-02-05 · **Method:** web research (web_search MCP) + analysis
**Question:** Which mechanisms from established productivity systems deserve to shape Executor's data model and algorithms? Nothing assumed correct; each judged on (a) what executive-function (EF) problem it actually solves, (b) whether it survives into software (app/research evidence), (c) cost for an ADHD user.

## 0. The EF problem space Executor must serve

Findings from ADHD-clinical and research-adjacent sources gathered during the survey:

- **Task initiation failure** is the headline problem. Standard clinical advice converges on: break tasks into tiny steps, externalize time (clocks/timers/alarms), micro-starts ("just two minutes"), reduce simultaneous information load. (envisionadhd.com, natlawreview, skillpointtherapy)
- **Time blindness**: ADHD users can't reliably perceive elapsed/remaining time; the consistent recommendation is *externalizing* time — visible timers, time blocking, one calendar. (neurodiversion.org, divergentcoaching, graysonexecutivelearning)
- **Interest-based motivation**: "ADHD motivation is interest-based rather than importance-based" — activation comes from novelty, urgency, challenge, interest; *not* from knowing something is important. (collinspsychology, Dodson-derived framing, fixmybrain)
- **Working-memory limits** → anything held "in the head" is lost; open loops must live outside.
- **Strongest quantitative anchor in the whole survey:** implementation intentions — pre-deciding when/where/how to act — meta-analytic effect d ≈ 0.65 on goal attainment (Gollwitzer & Sheeran 2006), with effects concentrated in *poorer planners* (Ahmadyar et al. 2025). Pre-decision is exactly what an external EF layer can automate.
- Direct RCT-style evidence for branded systems (GTD, BuJo, PARA…) is thin or absent. A 2015 *Computers in Human Behavior* workplace study of GTD-style workflow found reduced mind wandering/improved performance mainly for people in cognitively complex jobs (reported finding; treat as weak evidence). The strongest signal of what "survives" is **what software actually ships and what research on components shows**, not system-level trials.

---

## 1. GTD (Getting Things Done, David Allen)

**Core mechanisms:** capture everything into trusted inbox; clarify (each item → trash / someday-maybe / reference / project); projects always carry a concrete *next physical action* (verb-first, visible when doable); contexts (originally @phone/@errand; now tool/energy/place); someday/maybe as first-class parking; waiting-for for delegated/blocked work; 2-minute rule (do immediately if < 2 min); weekly review (Get Clear → Get Current → Get Creative) — Allen calls it "the critical success factor"; natural planning model (purpose → outcome → brainstorm → organize → next actions); daily selection by four criteria: context, time available, energy, priority.

**EF problem solved:** gets everything out of working memory ("mind like water"); converts vague obligations into concrete startable actions; separates deciding from doing.

**Software survival: overwhelming.** OmniFocus, Things 3, FacileThings, Nirvana, Todoist, Apple Reminders (tags-as-contexts) all encode inbox → project → next-action, someday/maybe lists, defer dates, review checklists. The schema maps cleanly to software. GTD's own forum debates concede the hard part is the *human habit*, not the model.

**Data-model implications**
- `inbox_item` — zero-friction capture; natural language; no required fields.
- Every project MUST expose exactly one (or ≥1) concrete `next_action` with physical-verb phrasing. This is the seed of Executor's "tap task → micro-plan" feature.
- `someday_maybe` as a **status, not deletion** — deferral without guilt.
- `waiting_for` state: owner + date + aging.
- Contexts as **filterable dimensions** (tags: location, tool, energy level, people) — agent-inferable, not user-typed.
- Natural planning model → optional project fields: purpose, desired outcome, notes. Agent fills these on demand during clarification.

**Algorithm implications**
- Clarify pipeline: agent drafts the classification, user confirms — decisions without clerical work.
- Daily selection function = Allen's four criteria (context ∧ time available ∧ energy ∧ priority), computed from data.
- Weekly review as scheduled maintenance surfacing: unprocessed inbox, projects without next action, aged waiting-for, stale someday/maybe.

**ADHD conflict:** GTD's load-bearing wall is the weekly review and trusted capture — both are *discipline*, precisely what ADHD erodes. GTD collapses for ADHD users at exactly these points. **Invert them: the agent performs review maintenance; the human only answers decision prompts.** The 2-minute rule should become the agent's "2-minute micro-start" generator (a first trivially-startable step for any task), not a discard heuristic.

---

## 2. PARA (Projects, Areas, Resources, Archive — Tiago Forte)

**Core mechanism:** organize everything by *actionability* into four buckets: Projects (finite efforts with an end state, e.g. "file taxes"), Areas (ongoing responsibilities with a maintenance standard, e.g. "Finances"), Resources (topics of interest), Archive.

**EF problem solved:** the areas-vs-projects distinction is genuinely load-bearing: projects are finishable; areas never finish and instead need periodic health checks. Conflating them produces either guilt (treating "Health" as a never-completing project) or neglect (no area ever reviewed).

**Software survival: partial, and telling.** Things 3 ships "Areas" natively (predates PARA's popularity); Notion/Obsidian PARA templates are ubiquitous. Criticism in the wild: it's a *filing taxonomy* dressed as a productivity system; Resources is "where things go to die"; heavy upfront organizing conflicts with capture friction (catangel.ch critique; community consensus).

**Data-model implications**
- `area` = distinct entity from `project`: no done-state; carries review cadence + "what maintained looks like" standard.
- `project → area` link required (gives the agent the "why" for prioritization).
- Archive as a state transition, not a place.

**Algorithm implications:** area health checks on cadence; project creation demands an owning area; completion auto-archives.

**Reject:** Resources bucket (agent full-text search makes topical folders pointless); any upfront-organizing ceremony. **Adopt:** areas/projects/entities only.

---

## 3. Zettelkasten (Luhmann)

**Core mechanism:** atomic notes, densely linked, no imposed hierarchy; structure emerges; slow manual processing *is the thinking*.

**EF problem solved:** **none directly.** It's a knowledge-synthesis tool for long-horizon creative work — it does not address initiation, prioritization, or time. "Second brain" ≠ external executive function.

**Software survival: massive but orthogonal** — Obsidian, Roam, Logseq, The Archive prove linked atomic Markdown is a viable substrate.

**Implication:** adopt the **substrate**, reject the methodology. Executor's context store = atomic Markdown + links (agent-maintained, agent-mined). Never require the user to do Zettelkasten-style slow processing. Hype flag: linking thinking will not answer "what should I do right now."

---

## 4. Bullet Journal (Ryder Carroll)

**Core mechanisms:** rapid logging (short entries, signifiers); daily log as append-only stream; **migration** — undone tasks must be *manually rewritten* into the next month/collection, a deliberate friction/forcing function; monthly/weekly logs; index + threading.

**EF problem solved:** migration makes deferral a *conscious re-commitment decision* and surfaces avoidance patterns that silent auto-rollover hides. ADHD-relevant: "the migration ritual itself acts as a re-engagement trigger" (goalsandprogress.com).

**Software survival:** deliberately analog; software that auto-rolls tasks forward **destroys the migration signal**. But software can compute the signal *better than paper*: count deferrals automatically.

**Data-model implications**
- **Deferral events as first-class data**: every rollover logged (task, date, count).
- Active set kept small by design (daily log = today's page).
- Capture format: one-line natural language, no schema.

**Algorithm implications:** deferral-count thresholds trigger agent action: decompose the task (vague → generate micro-plan), renegotiate (deadline), or force the BuJo question ("is this still worth it?" → someday/maybe demotion). Chronic deferral is Executor's highest-value learning signal.

**Reject:** manual rewriting (won't be sustained), signifiers/index (agent does retrieval). **Adopt:** the migration *decision point* and deferral telemetry.

---

## 5. Time blocking & timeboxing

**Core mechanism:** time blocking = assign tasks to calendar slots; timeboxing = fix a time budget for work rather than leaving it open-ended (counteracts Parkinson's law).

**EF problem solved:** externalizes time for time-blind users; converts "when" decisions to made-in-advance; makes task-vs-time-fit visible.

**Software survival: strong and recent.** A whole auto-scheduling market exists: Motion (aggressive AI auto-scheduling), Reclaim (focus-time protection), FlowSavvy, Morgen, Sunsama (guided *manual* daily planning ritual), Akiflow. The market has already forked on the key design question: **decide for the user vs. guide the user.** That fork is Executor's to answer too.

**ADHD conflict:** rigid pre-planned days crumble at the first derailment → cascade abandonment; ADHD time estimation is systematically wrong, so blocks are wrong; manual re-planning burns the EF the schedule was supposed to save.

**Data-model implications**
- `timebox` = first-class entity (task instance + start + duration + flexibility policy + status: planned/kept/moved/missed).
- Calendar events are **derived views**, not the source of truth.
- Log re-plan events → feed effort-estimation learning (attacks "consistently underestimated tasks" directly).

**Algorithm implications:** rolling auto-reshuffle on miss (never make the user re-plan); learned per-task-type duration estimates from actual history; protected anchors (sleep, meals, meds) + generous default buffers; propose, don't impose. **Adopt timeboxing-as-data; reject rigid full-day blocking.**

---

## 6. Eisenhower matrix (urgent × important)

**Core mechanism:** 2×2 triage — do / schedule / delegate / delete.

**EF problem solved:** distinguishing real deadlines from noise… in principle.

**Software survival: weak as a core model.** Ubiquitous as a concept, rare as an app backbone. Documented limitations: ignores effort and dependencies; ratings are subjective and unstable; quadrants overload (worksection, ideascale, projectmanager critiques).

**Implication: do not store static urgency/importance scores.** Derive *urgency* from deadline data (time-to-deadline vs. estimated effort) and *importance* from area/project linkage + stated goals + observed choices. The matrix survives only as explanation vocabulary.

**ADHD-specific insight (the useful part):** the "important, not urgent" quadrant (health, finances, taxes) is where ADHD life quietly rots, because importance doesn't activate an interest-based nervous system. Executor should **manufacture urgency** for Quadrant-2 work: staged internal deadlines with reminders that carry real deadlines' force. This is an *algorithm*, not a data model.

---

## 7. Kanban / WIP limits (Personal Kanban — Benson & Barry)

**Core mechanism:** visualize all work; limit work-in-progress; pull, don't push; flow metrics (cycle time, blocked items).

**EF problem solved:** WIP limits are an external constraint against start-new-thing impulses — finishing over starting. Probably the single most ADHD-relevant team-method import: hyperfocus/novelty-seeking yields 12 open projects; a WIP limit is the polite bouncer.

**Software survival: dominant** in team tools (Jira, Trello, GitHub Projects); Personal Kanban (Benson) reduces it to two rules — visualize + limit WIP. Agile-research reviews credit WIP limits with preventing overload and surfacing blockers.

**Data-model implications:** status column per task; `blocked` flag + reason; started_at/completed_at timestamps → cycle time, "started but stalled" detection; user-configurable WIP policy.

**Algorithm implications:** starting something new when active ≥ limit triggers *friction, not a gate* (agent: "finish X, consciously park it, or bump something") — hard blocks trigger demand-avoidance in ADHD. Stalled-WIP detection → agent prompts finish-or-park.

---

## 8. MIT / "Eat the Frog"

**Core mechanism:** pre-select 1–3 Most Important Tasks for the day; do the hardest/most aversive first (Tracy; Twain attribution apocryphal).

**EF problem solved:** pre-commitment answers "what now" before willpower is spent. The *pre-selection* half is well supported by implementation-intentions research (planning what/when in advance, d ≈ 0.65). The *morning-willpower* half leans on ego depletion, which failed replication well; "hardest first" is folk wisdom, not settled.

**Software survival:** weak as a feature (a flag/star at best) — surprisingly under-implemented relative to its popularity.

**Data-model implications:** daily plan entity = 1–3 MITs + rough order + when/where (the implementation-intention slots).

**Algorithm implications:** nightly, agent proposes tomorrow's MITs from deadlines + areas + WIP state; morning confirmation is one tap; default answer to "what should I do right now?" = current MIT + its micro-start. **Make frog-first vs. warm-up-win-first a learned per-user preference** (ADHD users split on this; it's observable from session data).

---

## 9. Pomodoro

**Core mechanism:** fixed 25-min work / 5-min break cycles, external timer, count pomodoros per task.

**EF problem solved:** lowers activation energy ("commit to only 25 minutes"); externalizes time perception; breaks counteract runaway time blindness; countable units give a progress signal.

**Software survival: enormous app ecosystem** (Forest, Focus To-Do, dozens more); standard ADHD-coach recommendation as "pomodoro blocks."

**Evidence:** research support for the *specific* 25/5 numbers is thin; benefits plausibly reduce to timeboxing + external timer + enforced breaks.

**Data-model implications:** work `session` events (task, start/end, interruptions, self-rated focus) — session telemetry is the substrate for effort estimation and context modeling ("user works well 9–11am in 45-min blocks").

**Algorithm implications:** "start a 25-min session on X" as the universal start affordance on every task; adapt interval length from history; **never force a break mid-flow** — interrupting hyperfocus is costly for ADHD. Pomodoro = offer, not regime.

---

## 10. Systems vs Goals (Adams; Clear)

**Core mechanism:** outcomes (goals) vs. repeatable processes (systems); Clear: "you do not rise to the level of your goals; you fall to the level of your systems."

**EF problem solved:** goals without next actions are fantasy (Allen's claim too); systems externalize repeated decisions into routines.

**Software survival:** habit trackers everywhere; process metrics standard in org software.

**Data-model implications:** `routine` = first-class entity (recurring process, cadence, checklist, adherence history) **distinct from** `goal` (outcome + target + rationale) and `project` (finite effort). Goals link to the routines/projects that allegedly achieve them; a goal with no supporting system is flagged aspirational.

**Algorithm implications:** track system *adherence* with kind recovery — **no streak-shaming** (streak loss is a known ADHD abandonment trigger; broken streaks should trigger re-planning, not guilt). Review "system health," not just goal status. The slogan is hype-level; the engineering content is: represent processes explicitly and measure their execution.

---

## 11. Weekly / daily review practices

**Core mechanism:** recurring reflect-and-reset ritual. GTD weekly review: Get Clear (empty inboxes), Get Current (project/by-context review, waiting-for, calendar), Get Creative. Sunsama's daily shutdown/planning ritual is the same loop compressed.

**EF problem solved:** externalized prospective memory; catches rot (stale projects, aged waiting-for); planned-vs-actual comparison closes the loop that makes "learning from behavior" possible.

**Evidence:** the nightly-planning pattern *is* implementation intention at scale — the best-supported mechanism in this survey (d ≈ 0.65; strongest for poor planners, i.e., the exact Executor user).

**Software survival:** review checklists exist (OmniFocus, FacileThings) but remain discipline-dependent — the canonical failure point of every system in this survey.

**Data-model implications:** review events (what was reviewed, decisions made); planned-vs-actual daily log; staleness fields (last_touched, last_reviewed).

**Algorithm implications:** **agent-driven review** — the agent performs the clerical 80% (inbox triage drafts, stale list, deferral counts, waiting-for aging) and presents a short decision list; reviews triggered by *decay signals* (inbox nonempty > 48h, project without next action, deferrals > N), not just the calendar; missed reviews catch up without penalty. Cadence: nightly micro (≤5 min) + weekly structural.

---

## Cross-cutting findings

1. **Every surviving paper system shares one skeleton:** capture → clarify → select → execute → review. Review is the load-bearing wall and the universal ADHD failure point. Executor's defining inversion: **the agent does maintenance; the human only makes decisions.**
2. **Pre-decision is the best-researched mechanism** (implementation intentions, d ≈ 0.65, strongest for poor planners). Executor should auto-generate when/where/how + first physical step for everything — this is also exactly the "grow tent" micro-plan requirement.
3. **Events over states.** The learning substrate is behavioral telemetry: deferral events (BuJo), session events (Pomodoro), re-plan events (timeboxing), review events, planned-vs-actual. This matches Executor's stated goal of learning from actual behavior; static snapshots can't support it.
4. **Rigidity is the common failure mode.** Rigid blocks, forced 25-min intervals, manual rewriting, hard WIP gates, streaks — all conflict with ADHD. Their *signals* survive when made frictionless and automatic.
5. **Importance doesn't move an ADHD brain; structure does.** Don't store importance ratings. Manufacture structure: staged internal deadlines for Quadrant-2 work, micro-starts, one obvious current task, WIP friction.

## Adopt / Reject table

| Mechanism | Source | Verdict | Data model | Algorithm |
|---|---|---|---|---|
| Capture/inbox | GTD | **Adopt** | inbox_item, zero required fields | agent drafts clarification |
| Next-action concreteness | GTD | **Adopt** | project ⇄ exactly-one visible next action | micro-plan generator |
| 2-minute rule | GTD | **Adapt** | — | 2-minute micro-start generator |
| Contexts | GTD | **Adapt** | filterable tag dims (place/tool/energy) | daily selection filter |
| Someday/maybe | GTD | **Adopt** | first-class status | — |
| Waiting-for | GTD | **Adopt** | blocked state: owner+date+aging | aging alerts |
| Weekly review | GTD | **Adapt** (agent-driven) | review events, staleness fields | decay-triggered, agent-clerical |
| Four-criteria selection | GTD | **Adopt** | — | selection function |
| GTD-as-discipline | GTD | **Reject** | — | — |
| Areas vs projects | PARA | **Adopt** | area ≠ project entities; project→area link | area health checks |
| Resources bucket | PARA | **Reject** (agent search) | — | — |
| Archive | PARA | **Adapt** | state transition | auto-archive |
| Atomic linked notes | Zettelkasten | **Adapt substrate** | Markdown + links, agent-maintained | context retrieval |
| Zettelkasten practice | Zettelkasten | **Reject** | — | — |
| Rapid logging | BuJo | **Adopt** | one-line NL capture | — |
| Migration | BuJo | **Adopt signal** | deferral events first-class | threshold → decompose/renegotiate/demote |
| Signifiers/index | BuJo | **Reject** | — | agent retrieval |
| Time blocking (rigid) | Time mgmt | **Reject** | — | — |
| Timeboxing | Time mgmt | **Adopt** | timebox entity; derived calendar; re-plan events | auto-reshuffle; learned estimates |
| Eisenhower scoring | Eisenhower | **Reject as data** | derive urgency/importance | — |
| Important-not-urgent handling | Eisenhower | **Adapt** | — | staged manufactured deadlines |
| Visualize work | Kanban | **Adopt** | status, blocked flags | stalled detection |
| WIP limits | Kanban | **Adopt w/ friction** | WIP policy | advisory gate, finish-or-park prompts |
| Flow metrics | Kanban | **Adopt** | timestamps → cycle time | velocity per context |
| MIT daily selection | MIT | **Adopt** | daily plan: 1–3 MITs | nightly proposal, morning confirm |
| Frog-first ordering | Eat the Frog | **Adapt** | — | learned frog-vs-warmup preference |
| Pomodoro intervals | Pomodoro | **Adapt** | session events | universal start affordance; adaptive length; no forced breaks |
| Routines ≠ goals ≠ projects | Systems vs goals | **Adopt** | routine entity + adherence history | kind recovery, no streak-shaming |
| Daily/nightly planning | Review practices | **Adopt** | planned-vs-actual log | agent-drafted nightly plan |

## Sources (selected)

- Gollwitzer & Sheeran 2006 implementation-intentions meta-analysis, d≈0.65 — goodmedicine.org.uk summary; Frontiers (Wang 2021); KOPS (Wieber); Sheeran 2005; Ahmadyar et al. 2025 (poorer planners benefit most)
- GTD mind-wandering/job-performance study, Computers in Human Behavior 2015 (S1071-5819(15)00190-1) — reported finding, weak evidence
- GTD Weekly Review checklist — gettingthingsdone.com; "critical success factor" framing
- PARA — fortelabs.com; The PARA Method reviews; catangel.ch critique
- Bullet Journal — Ryder Carroll method summaries; azuliadesigns (migration "cornerstone"); goalsandprogress.com (migration as ADHD re-engagement trigger)
- Time blindness / externalizing time — neurodiversion.org, divergentcoaching.com.au, graysonexecutivelearning.com; fluidwave.com (2018 ADHD timer-training study)
- Auto-scheduler market — thesarahai.com, saner.ai, morgan.so, dupple.com, get-alfred.ai (Motion vs Reclaim vs Sunsama design fork)
- Personal Kanban — personalkanban.com (Benson), kanbantool.com; WIP-limit evidence in agile reviews (academia.edu systematic review)
- Eisenhower critiques — ideascale.com, worksection.com, projectmanager.com
- Interest-based motivation — collinspsychology.com, adhdnow.com, fixmybrain.ai; dopamine framing mindlabneuroscience.com
- Systems vs goals — jamesclear.com/goals-systems, modelthinkers.com, Scott Adams *How to Fail…*
- GTD software — nextthinggtd.com (Things 3 areas, GTD app landscape), facilethings.com (subprojects), Apple Remmaries tags/contexts
- Pomodoro — focusclock.app guide; ADHD pomodoro-block advice (cincinnaticenterfordbt.com)
