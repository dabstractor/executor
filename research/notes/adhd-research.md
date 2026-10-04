# ADHD & Executive-Function Design Research — Findings for "Executor"

**Scope:** Design principles only. This research informs a personal life-management tool; it is not a medical device and nothing here is a medical claim or treatment recommendation. All psychological claims below are cited for their *design implications*, not clinical use.

**Method:** Targeted web search (web_search MCP), prioritizing peer-reviewed primary sources, systematic reviews, and meta-analyses. Confidence: **High** = meta-analysis or broad replication; **Moderate** = consistent primary studies, limited synthesis; **Low** = survey/self-report evidence or thin literature. Links are from search results; DOIs given where stable.

---

## 1. Task initiation & implementation intentions

**Finding.** If-then plans ("When situation X arises, I will do Y") substantially improve goal attainment versus mere goal intentions. The meta-analysis of 94 independent tests found a medium-to-large average effect (d ≈ 0.65), roughly a 28 percentage-point increase in goal attainment across domains. Effects are *larger* for difficult goals and when the critical cue is concrete and specific. Mechanism: the specified cue becomes highly activated and triggers action automatically ("strategic automaticity"), reducing the need for effortful initiation at the moment of action. Known limits: effects shrink or vanish when competing habitual responses are strong, when the cue is vague ("whenever I have time"), or when the goal itself is weakly committed.

**Key citations.**
- Gollwitzer, P. M. (1999). Implementation intentions: Strong effects of simple plans. *American Psychologist*, 54(7), 493–503. DOI: 10.1037/0003-066X.54.7.493 (PDF: cancercontrol.cancer.gov, via search)
- Gollwitzer, P. M., & Sheeran, P. (2006). Implementation intentions and goal achievement: A meta-analysis of effects and processes. *Advances in Experimental Social Psychology*, 38, 69–119. DOI: 10.1016/S0065-2601(06)38002-1 (referenced via scirp.org, ipcrg.org)
- Carrera, M. J., et al. (2018). The limits of simple implementation intentions: Evidence from the domain of snacking. (PMC, via search)

**Confidence:** High (meta-analysis; hundreds of replications; one of the most robust effects in social psychology).

**Contradictions.** Publication-bias concerns exist (like all of social psych); moderator analyses show the effect is not uniform — it fails with weak commitment and abstract cues.

**Design implication.** The data model should treat a "task" as incomplete without an *executable trigger*: `{trigger_context, first_action}`. The agent's job when the user captures a vague task is to auto-complete the if-then: infer when/where the task will become actionable from user context (location, time, preceding event, calendar) and generate the first physical action ("Get the contractor bags from the storage shelf") rather than restating the goal. Tasks lacking a concrete cue should be flagged as "not yet plannable."

---

## 2. Prospective memory (remembering to act)

**Finding.** Prospective memory (PM) — remembering to perform an intended action after a delay — splits into **event-based** (triggered by an external cue) and **time-based** (requires self-initiated clock-checking). Event-based cues with high *focal* distinctiveness benefit from spontaneous retrieval; time-based PM requires continuous monitoring, which is exactly what ADHD impairs. ADHD studies show PM deficits in both types, with time-based and monitoring-dependent PM typically worse; children and adults with ADHD show event-based PM deficits even in controlled tasks.

**Key citations.**
- Altgassen, M., et al. (2014). Task dissociation in prospective memory performance in adults with ADHD. (tud.qucosa.de, via search)
- Costanzo, F., et al. (2021). Event-based prospective memory deficit in children with ADHD. (PMC: pmc.ncbi.nlm.nih.gov, via search)
- Smith, R. E. (2016). Prospective memory: A framework for research on individual differences. (psycnet.apa.org, via search)
- Secondary explainer of the event/time distinction and ADHD: chadd.org, "Remembering the Future: How ADHD Affects Prospective Memory"

**Confidence:** Moderate — consistent findings, but the ADHD PM literature is smaller than the general PM literature and mixes lab/clinical designs; no single large ADHD-specific meta-analysis surfaced.

**Contradictions.** Some adult studies find time-based deficits without event-based deficits (task-dissociation findings); results vary with cue focality and working-memory load.

**Design implication.** The system should **convert time-based intentions into event-based ones wherever possible**: instead of only "3:00 PM — call dentist," fire the reminder when a detectable context event makes it actionable (phone unlocked + idle + errand window open; user near the phone district; pre-departure morning routine). Data model: intentions carry `deadline` (time-based backstop) AND `trigger_events[]` (sensor/context predicates), with the engine preferring event triggers over pure clock alarms.

---

## 3. Barkley's self-regulation model & the "point of performance"

**Finding.** Barkley's model characterizes ADHD as a disorder of self-regulation and future-directed behavior: deficits in behavioral inhibition cascade into impaired working memory, time sense, emotional regulation, and internalized motivation. His central intervention principle is that treatment must operate **at the point of performance** — the time and place where the behavior must occur — and must **externalize** the information, time, and motivation that a neurotypical brain would internalize. Consequences/rewards must be immediate, frequent, and externally delivered, not delayed. Internal strategies ("try harder, remember better") are specifically predicted to fail.

**Key citations.**
- Barkley, R. A. (1997). *ADHD and the Nature of Self-Control.* New York: Guilford Press.
- Barkley, R. A. (2012). *Executive Functions: What They Are, How They Work, and Why They Evolved.* Guilford. (externalization chapters)
- Systematic review applying Barkley (1997) to interventions: openpsychologyjournal.com (via search, "A Systematic Review of the Literature")

**Confidence:** High as an influential, widely-used theoretical framework; it is a *model*, not a set of experiments — its intervention corollaries (externalize at point of performance; immediacy of consequences) are consistent with the broader behavioral literature.

**Contradictions.** Barkley's hybrid model is dominant but contested in emphasis (motivational vs. executive accounts; delay-aversion theory of Sonuga-Barke offers a rival framing). The design principle is shared by all rival theories.

**Design implication.** Architecturally decisive: Executor must be **available at the point of performance and never require the user to consult it from memory**. A phone client backed by the always-on home machine is the right shape; the system pushes the cue to where the user physically is, rather than waiting to be consulted. Any feature that assumes the user will remember to open the app is dead on arrival.

---

## 4. Planning fallacy & reference-class forecasting

**Finding.** People systematically underestimate how long their own future tasks will take, even with experience of past overruns. Corrections that work: (a) the **outside view / reference-class forecasting** — predict from the distribution of comparable past cases rather than the specific scenario; (b) **correcting memory** — the bias persists partly because memories of past task durations are themselves biased, and prompting accurate recall of past actuals improves prediction. For Executor: the user's own historical *actuals* are the best correction data.

**Key citations.**
- Buehler, R., Griffin, D., & Ross, M. (1994). Exploring the "planning fallacy": Why people underestimate their task completion times. *Journal of Personality and Social Psychology*, 67(3), 366–381. DOI: 10.1037/0022-3514.67.3.366
- Kahneman, D., & Tversky, A. (1979). Intuitive prediction: Biases and corrective procedures. *TIMS Studies in Management Science*, 12, 313–327. (reference-class "outside view"; applied overview: shr.swiss, via search)
- Roy, M. M., Christenfeld, N. J. S., & McKenzie, C. R. M. (2005). Underestimating the duration of future events: Memory incorrectly used or memory bias? *Memory & Cognition*, 33(5), 723–729.
- Roy, M. M., et al. (2008). Correcting memory improves accuracy of predicted task duration. *Memory & Cognition*. (pubmed.ncbi.nlm.nih.gov, via search)

**Confidence:** High (planning fallacy is one of the most replicated judgment biases; debiasing via reference classes has both lab and large-project field support — Flyvbjerg's infrastructure work).

**Contradictions.** Memory-bias vs. memory-misuse accounts compete (Roy et al.); either way the fix — surfacing recorded actuals — is the same.

**Design implication.** Keep a **per-task-type duration ledger**: every task gets `estimate_at_capture`, then `actual` on completion. When planning, the engine presents the user's personal reference-class distribution ("your last 6 'clean the basement' class tasks: median 2.1h, p80 3.5h") and can auto-pad estimates to the personal p80 for scheduling. This single feature turns the brief's "tasks consistently underestimated" observation into a self-correcting algorithm.

---

## 5. Cognitive offloading

**Finding.** Offloading cognitive work to the environment (lists, reminders, external notes) is a normal, value-based decision: people offload when internal effort is high and confidence in internal memory is low. Offloading improves task performance for the offloaded content but can weaken memory for the offloaded items (dependency/"Google effect" family of findings) and can be metacognitively miscalibrated (people over- or under-offload). Crucially for a *tool* (vs. training): offloading's costs are about internal memory retention, which is irrelevant when the external store is always present — the failure mode is the store becoming unavailable, stale, or untrusted.

**Key citations.**
- Risko, E. F., & Gilbert, S. J. (2016). Cognitive offloading. *Trends in Cognitive Sciences*, 20(9), 676–688. DOI: 10.1016/j.tics.2016.07.002
- Gilbert, S. J., et al. (2023). Cognitive offloading is value-based decision making. (osf.io preprint, via search; cited 69)
- Bulley, A., et al. (2020). Developmental origins of cognitive offloading. (discovery.ucl.ac.uk, via search)

**Confidence:** High for the basic phenomenon; Moderate for prescriptive rules about when it "backfires" (lab tasks ≠ life management).

**Contradictions.** Literature emphasizes memory-dependency costs; these largely don't apply to an always-on external layer whose whole purpose is to replace internal memory. Noted as a scope caveat, not a contradiction within the tool's frame.

**Design implication.** Capture must be **near-zero friction or users will keep it in their heads** (the value-based account says they offload only when it's cheaper than remembering). Every interaction that asks the user to hold something in memory between sessions is a bug: the system, not the user's head, is the memory of record; the phone entry point must always reflect current system state.

---

## 6. Working memory constraints

**Finding.** Working memory holds only ~4–7 chunks for seconds without rehearsal; performance collapses when a task requires holding more. Cognitive Load Theory generalizes this to all complex task performance: instructional/support designs that chunk information and sequence it (one step at a time) dramatically outperform designs that present everything at once.

**Key citations.**
- Paas, F., et al. (2020). Cognitive-Load Theory: Methods to manage working memory load in the learning of complex tasks. *Perspectives on Psychological Science*. https://journals.sagepub.com/doi/10.1177/0963721420922183 (cited 900+)
- Cowan, N. (2001). The magical number 4 in short-term memory. *Behavioral and Brain Sciences*, 24(1). (background)
- Miller, G. A. (1956). The magical number seven. *Psychological Review*, 63. (background)

**Confidence:** High (foundational cognitive psychology).

**Contradictions.** None material; chunk size estimates vary (4 vs 7), which does not affect design.

**Design implication.** The task card interaction is: **one visible next action, with the full plan one tap away, never displayed as a wall**. Plans should be generated as short sequenced steps (3–7 chunks), each phrased as a physical, observable action; the UI shows exactly one step at a time and marks progress. This is also why "tap a task → immediately executable plan" in the project brief is the correct core interaction.

---

## 7. Time perception in ADHD ("time blindness")

**Finding.** Meta-analyses confirm reliable, medium-sized deficits in time perception/discrimination in ADHD across the lifespan (children: ~1,620 participants; a 25-study synthesis of time discrimination, n=1,633, found medium effects; milliseconds-to-seconds timing also impaired in adults; a 2024 lifespan meta-analysis quantifies deficits throughout adulthood). The deficit is strongest for duration discrimination/reproduction and is worsened by working-memory load. Subjectively this manifests as severe misjudging of elapsed and remaining time.

**Key citations.**
- Huang, Y., et al. (2021). Time perception deficits in children and adolescents with ADHD: A meta-analysis. *Journal of Attention Disorders*. https://journals.sagepub.com/doi/10.1177/1087054720978557
- Marx, I., et al. (2022). Altered perceptual timing abilities in ADHD. *JAACAP*. https://www.sciencedirect.com/science/article/abs/pii/S0890856721020451
- Metcalfe, K. B., et al. (2024). Time-perception deficits in ADHD throughout the lifespan. *Developmental Neuropsychology*. https://www.tandfonline.com/doi/abs/10.1080/87565641.2023.2293712
- Evidence summary: adhdevidence.org, "Time blindness found to be a consistent feature of ADHD"

**Confidence:** High (multiple independent meta-analyses).

**Contradictions.** Earlier single studies were inconsistent (task-dependent); metas resolve this — deficits are consistent but effect sizes vary by task modality.

**Design implication.** Never make the user *estimate* time: render time externally and concretely. Concretely: elapsed-time timers for active tasks, countdown bars for deadlines, and explicit "time until X" annotations on every scheduled item in the agent's own outputs (e.g., "in 40 min", "3 days left" as primary display, with clock times secondary). The agent's model of the day should be a visualized timeline, not a list of clock strings.

---

## 8. Reminder & notification design

**Finding.** Alerts that are frequent, repetitive, or poorly timed lose effectiveness: habituation to repeated stimuli is well documented (even emergency alerts), notification overload depresses engagement, and users differ hugely in attentiveness by moment and context. mHealth intervention research explicitly identifies "overwhelming interaction" as the central problem for reminder agents and explores reinforcement-learning-based *adaptive* notification policies (timing/personalization) as the fix. Predicting the user's momentary receptivity measurably beats naive delivery.

**Key citations.**
- Wang, S., et al. (2021). Optimizing adaptive notifications in mobile health intervention agents. (research-portal.uu.nl, via search; cited 68)
- Pielot, M., et al. (2014). Predicting attentiveness to mobile instant messages. (ink.library.smu.edu.sg, via search; cited 279)
- Habituation to repeated alerts (incl. emergency alerts): arxiv.org WEA study (via search)

**Confidence:** Moderate — direction is clear and consistent, but few RCTs isolate notification design parameters; personalization findings come mostly from mHealth/ML literature with modest effect sizes.

**Contradictions.** Some personalization gains come from *reducing* total notifications (less is more); studies differ on optimal features for receptivity prediction.

**Design implication.** Treat notifications as a **scarce, adaptive resource**: cap daily volume, vary content (never send the same phrasing twice in a row), deliver on context triggers rather than fixed times (ties to §2), and learn per-user response models from observed acknowledgment/act rates, down-weighting channels and phrasings that habituate. The data model needs `notification_events` with outcomes so the agent can do this learning explicitly.

---

## 9. Self-monitoring reactivity

**Finding.** Merely recording one's own behavior changes that behavior (reactivity), strongest when the target behavior is salient, the recording is immediate, and the behavior is one the person wants to change. The effect is the active ingredient behind self-monitoring components in behavioral programs, including ADHD interventions (a meta-analysis of self-monitoring interventions in students with ADHD found positive effects). Long-term reactivity fades unless monitoring continues. Design-relevant nuance: tracking multiple behaviors dilutes reactivity; accuracy matters less than the act of recording.

**Key citations.**
- Mace, F. C., et al. (1985). Theories of reactivity in self-monitoring. *Behavior Modification*, 9(3). https://journals.sagepub.com (cited 97)
- Nelson, R. O., et al. (1982). Long-term effects of self-monitoring: Reactivity and accuracy. (sciencedirect, via search)
- Hanson, A. (2018). Self-monitoring for ADHD: A meta-analysis. (etd.ohiolink.edu, via search; 10 studies)
- Review: openaccesspub.org, "A theoretical and empirical review of self-monitoring" (Hayes & Cavior, via search)

**Confidence:** Moderate-to-High (reactivity itself is classic and replicated; modern effect sizes in digital contexts are less well synthesized).

**Contradictions.** Reactivity decays over time; some studies find monitoring without feedback is weak for maintenance.

**Design implication.** Make **completion logging itself the intervention**: one-tap check-off at the moment of completion (frictionless → maintained), then immediately surface what the tracking shows ("you've cleared 3 basement steps this week; last week 0"). The system should render visible, per-behavior progress immediately after each log — pairing self-monitoring with instant feedback, since reactivity is strongest when recording is immediate and target-salient.

---

## 10. Habit formation

**Finding.** Habits form through **context-cued repetition**, not motivation or repetition alone: performing a behavior consistently in a *stable context* (same time, place, preceding action) builds automaticity; automaticity then runs without deliberate intent and is shockingly insensitive to goals. Median time-to-automaticity is ~66 days with huge variance (18–254 days), and missing a single day does not measurably derail formation. Unlearning/breaking habits works better by *disrupting context cues* than by willpower. Intervention framing (Wood & Neal): add new habits by piggy-backing on existing stable cues ("after I X, I will Y").

**Key citations.**
- Lally, P., et al. (2010). How are habits formed: Modelling habit formation in the real world. *European Journal of Social Psychology*, 40(6), 998–1009. (median 66 days; multiple secondary sources verified)
- Wood, W., & Neal, D. T. (2016). Healthy through habit: Interventions for initiating and maintaining health behavior change. (semanticscholar.org/scribd, via search)
- Wood, W. (2023). Habits are not goal-dependent. Commentary. (psycnet.apa.org, via search)

**Confidence:** High (field study + large supporting literature; "21 days" is pop myth, 66-day median is the real figure).

**Contradictions.** Debate over whether habits are truly goal-independent; debate over automaticity asymptotes. Doesn't affect design.

**Design implication.** Routines in Executor should be modeled as **cue → action chains anchored to detectable stable contexts**: `after_event(device_unlocked_at_home, 08:00–10:00) → routine X`, where each step's completion is the context cue for the next (implementing both §1 chaining and Wood/Neal piggy-backing). The system should also treat *changing contexts* (moving, travel, schedule shifts) as habit-risk events and explicitly re-anchor routines, because that's when automated behavior breaks.

---

## 11. The Progress Principle (small wins)

**Finding.** From ~12,000 daily diary entries of knowledge workers, Amabile & Kramer found that the single strongest day-level predictor of motivation, positive emotion, and subsequent productivity was **making progress in meaningful work** — even small wins. Setbacks disproportionately damage motivation ("negativity bias" of inner work life), and small progress events compound. The actionable corollary: structure work so progress is *visible and frequent*, and remove barriers to it.

**Key citations.**
- Amabile, T., & Kramer, S. (2011). *The Progress Principle: Using Small Wins to Ignite Joy, Engagement, and Creativity at Work.* Harvard Business Review Press.
- Amabile, T., & Kramer, S. (2011). The power of small wins. *Harvard Business Review.* hbr.org/2011/05/the-power-of-small-wins (background)
- Secondary summaries verified via search (theatlantic.com, rosewoodcoaching.com — "28% of minor work events have major inner-work-life impact")

**Confidence:** Moderate — a landmark field study, but single research program, diary methodology, non-ADHD population. Its mechanism (visible completion → motivation) is convergent with reinforcement principles.

**Contradictions.** Little adversarial replication; effect plausibility rests on converging evidence.

**Design implication.** Executor should **manufacture and memorialize small wins**: decompose tasks so that at least one step is completable in a single session (§6), mark step completion loudly (§9), and maintain a persistent "progress log" per project that the agent cites when re-motivating ("last time you did 4 steps in one Saturday morning") and uses when the user is stuck (offer the smallest visible-progress next step, not the most urgent one).

---

## 12. Adult ADHD psychosocial treatment components (CBT / meta-cognitive therapy / coaching)

**Finding.** The psychosocial treatments with RCT support for adult ADHD — Safren's CBT, Solanto's group meta-cognitive therapy, and CBT variants (Young et al.) — share a common operational core that maps almost one-to-one onto Executor's feature list: (1) task breakdown into small concrete steps; (2) external structure — calendaring, list systems, written problem-solving; (3) cue-based reminders at the point of action; (4) contingency management / immediate reward for completion; (5) cognitive restructuring of "I'll never finish this" self-talk; (6) shaping difficult tasks by starting with a 5–15 minute entry step. Meta-analytic effects are medium-to-large against control conditions (treatment-level, not component-level — components are theory-derived).

**Key citations.**
- Safren, S. A., et al. (2005). Cognitive-behavioral therapy for ADHD in medication-treated adults with residual symptoms. *Psychological Medicine.* (RCT; scispace reference via search)
- Safren, S. A., et al. (2010). *JAMA.* CBT vs relaxation+education RCT. (background)
- Solanto, M. V., et al. (2010). Efficacy of meta-cognitive therapy for adult ADHD. *American Journal of Psychiatry*, 167(9). (amanote.com + sagepub citation verified via search)
- Young, S., et al. (2015/2017). RCTs of R&R2ADHD CBT. (cambridge.org, springer, via search)
- Knouse, L. E., et al. (2013). Psychosocial treatment for adult ADHD: An update. *Current Psychiatry Reports.* (scholarship.richmond.edu, via search)
- Meta-analysis of CBTs for adult ADHD (2016, medium-to-large pre–post effects): content.apa.org (via search; authorship not independently confirmed — verify before citing formally)

**Confidence:** High for overall treatment efficacy; **Low-to-Moderate for individual component efficacy** (component-level evidence is theory-plus-practice, not dismantled RCTs).

**Contradictions.** Component lists differ by manual (Safren emphasizes cognitive restructuring; Solanto emphasizes scheduling/contingencies); effects persist but moderate at follow-up.

**Design implication.** Treat Executor as **the externalized delivery vehicle for the shared mechanics of these programs**: structured breakdown dialogs, external calendar/list as single source of truth, immediate completion reinforcement, and "start a 10-minute entry step" offers on stalled tasks — the agent operationalizes Solanto's contingency management and Safren's task-shaping without the therapy frame.

---

## 13. Body doubling

**Finding.** Body doubling — working on a task in the (physical or virtual) presence of another person — is widely reported by ADHD adults to dramatically improve task initiation and persistence. The term dates to Linda Anderson (~1996). The only significant formal study found is a 2024 survey (n=220 neurodivergent people) characterizing how/why it works (social accountability, external structure, reduced isolation, observed arousal regulation). There are no RCTs; mechanism hypotheses (social facilitation, accountability, performance monitoring) are unevaluated. Anecdotally it is one of the highest-leverage initiation hacks in the ADHD community.

**Key citations.**
- An Investigation of Body Doubling with Neurodivergent People (2024). *ACM ASSETS.* https://dl.acm.org (via search)
- Historical origin: addiction-ssa.org note (Linda Anderson, 1996, via search); practice explainer: chadd.org

**Confidence:** Low — survey + self-report only; plausibly real (social facilitation literature is old and solid) but zero controlled evidence.

**Contradictions.** None tested; individual variability reported (some find it anxiety-inducing).

**Design implication.** Cheap to support, unproven to work: add **structured focus sessions with a check-in cadence** (agent-as-minimal-body-double: session start, periodic lightweight "still on step 3?" pings, session-end summary) and integrate external human body doubling (e.g., a slot for Focusmate-style appointments in the routine engine). Instrument it: initiation rate with vs. without sessions is measurable locally, which would be better evidence than exists in the literature.

---

## Cross-cutting summary — strongest design implications

| # | Principle | Mechanism | Executor feature it mandates |
|---|-----------|-----------|------------------------------|
| 1 | If-then plans beat goals | Strategic automaticity | Tasks require trigger + first action |
| 2 | Event-based > time-based cues | Offloads monitoring | Context-triggered reminders, clock as backstop |
| 3 | Point of performance | Externalization at action time | Push-to-phone architecture |
| 4 | Outside view fixes estimates | Reference-class prediction | Personal duration ledger, p80 padding |
| 5 | Offloading is value-based | Capture cost decides use | Zero-friction capture, system = memory of record |
| 6 | WM ≈ 4–7 chunks | Cognitive load | One-step-at-a-time UI, 3–7-step plans |
| 7 | Time blindness is real | External time representation | Timers, countdowns, relative times |
| 8 | Notifications habituate | Stimulus repetition | Adaptive, capped, varied, triggered |
| 9 | Tracking changes behavior | Reactivity | One-tap logging + instant visible feedback |
| 10 | Habits = stable context cues | Context-response association | Cue-anchored routine chains, context re-anchoring |
| 11 | Small wins drive motivation | Progress principle | Step decomposition + progress log |
| 12 | Proven programs share a core | Task breakdown + contingency | Digital coach mechanics |
| 13 | Body doubling (weak evidence) | Social facilitation? | Focus sessions, instrument locally |

**Overall evidence assessment:** The strongest evidence base (meta-analytic) supports implementation intentions (§1), the planning fallacy and its personal-actuals correction (§4), WM limits (§6), time-perception deficits (§7), and habit formation mechanics (§10). Moderate support: point-of-performance externalization (§3 — dominant theory + behavioral tradition), event-based cue design (§2), notification adaptivity (§8), self-monitoring reactivity (§9), small wins (§11), CBT component core (§12). Weakest: body doubling (§13) — survey-only.

**Non-medical-device note repeated:** all of the above informs interaction design only; Executor is not a treatment, and no claim herein should be represented to users as therapeutic.
