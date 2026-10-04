# ADHD / executive-function design principles (evidence-based)

Design principles only — not medical advice, no diagnostic or treatment claims. Full citations
in `notes/adhd-research.md`; confidence levels marked.

## The nine principles (each: evidence → design implication)

### 1. Intervene at the point of performance (STRONG — Barkley 1997, 2012)

ADHD is best modeled as self-regulation/EF deficit; internal strategies ("remember better")
predictably fail; intervention must externalize information, time, and motivation **at the time
and place the behavior must occur**.
→ Executor's primary surface is the phone in the pocket, at the moment of choice: the "what
now" card must arrive where the user already is (messaging app / widget), not require
remembering to open an app. Immediate, external, frequent beats delayed and internal.

### 2. Implementation intentions (STRONG — Gollwitzer 1999; Gollwitzer & Sheeran 2006 meta-analysis, d≈0.65)

If-then plans ("when situation X arises, I will do Y") substantially raise goal attainment; the
mechanism is cue-delegation: the environment triggers the action.
→ Every task must carry a **trigger context** + a **generated first physical action**. A task
without a trigger and a first action is *unplannable* — that's a schema requirement (expansion
contract), not a nicety. Vague tasks are the enemy; the system's job is to never present one
naked.

### 3. Event-based beats time-based prospective memory (MODERATE — McDaniel & Einstein program; ADHD PM deficit studies)

External cues tied to *detectable events* outperform pure clock-watching, especially with ADHD.
→ Prefer condition triggers ("when I'm at home after 6pm", "when the laundry finishes",
OpenClaw standing-intents style) over bare timestamps; convert deadlines backward into
event-anchored chains ("the night before", "when you're next at the computer").

### 4. Correct estimates with personal actuals (STRONG — planning fallacy: Buehler, Griffin & Ross 1994; reference-class forecasting, Kahneman)

People — ADHD more so — underestimate durations; the corrective is outside-view data: *your own
past actuals* for similar tasks.
→ Log estimate-vs-actual per task type; default estimates = personal medians; pad schedules to
personal p80. "ADHD tax" multipliers (adhd-day-planner's 15→20/30→40/60→75) are a sane static
prior until personal data exists.

### 5. Working memory is ~4 chunks; externalize aggressively (STRONG — Cowan; cognitive offloading: Risko & Gilbert 2016)

Deliberate offloading improves task performance; the system should *be* the working memory.
→ One-step-at-a-time presentation; plans of 3–7 steps; never require the user to hold state
("where was I?" must be answerable by the system); zero-friction capture (one line, any
channel — GTD capture discipline).

### 6. Never require time estimation (STRONG meta-analytic — time perception deficits: Huang 2021; Marx 2022; Metcalfe 2024)

→ Timers, countdowns, relative times ("in 25 min"), visible time budgets everywhere; the user
never sizes a block by feel; the system sizes from actuals (#4).

### 7. Make progress visible and immediate (STRONG — progress principle: Amabile & Kramer 2011; CBT contingency management)

Small wins are the strongest day-level motivation predictor; immediate reinforcement is the
shared core of adult-ADHD psychosocial treatment (Safren 2005/2010; Solanto 2010).
→ Checkbox dopamine (nudge's term — real UI mechanic); instant completion logging (one tap);
day recap that counts *what got done*, never what didn't; no streaks/guilt/red (the
ADHD-affirming projects converge on refusing them).

### 8. Shape entry, don't shame avoidance (STRONG — CBT task-shaping; 10-minute entry steps)

→ The universal offer on any stalled task: a 10-minute (or ≤2-minute first action) session,
optionally "with me" (agent-mediated body doubling — weak evidence, ASSETS 2024 survey n=220,
but cheap and low-risk; instrument initiation rates locally before believing in it).

### 9. Habits ride stable contexts (MODERATE — Lally 2010 median 66 days; Wood & Neal)

→ Routines modeled as context-triggered chains (trigger + anchor + step template); context
changes (travel, schedule shifts) are habit-risk events the agent flags proactively.

## Additional moderate/weak evidence incorporated

- **Self-monitoring reactivity** (Mace 1985): tracking changes behavior — completion logging is
  itself therapeutic, not just data collection.
- **Notification habituation** (mHealth literature): cap cadence, personalize timing, vary
  phrasing; measure notification→action latency and *reduce* when it degrades.
- **Cognitive-load management** (Paas 2020): one decision at a time; the morning brief is 1–3
  MITs, never a wall.

## What the evidence says NOT to build

- Anything requiring the user to remember to check it (violates #1).
- Anything whose core loop is "try harder to estimate/remember/schedule" (violates #4/#5/#6).
- Streaks, red badges, guilt framing (contradicts #7; the ADHD-affirming OSS projects
  independently refuse them).
- Rigid full-day time blocking without derailment recovery (ADHD conflict documented across
  productivity-systems.md).
- Medical/adherence claims of any kind — this is scaffolding, not treatment.
