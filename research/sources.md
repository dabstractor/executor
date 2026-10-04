# Sources

Priority order used: source code > project docs > issues > design docs > maintainer posts >
peer-reviewed research > technical articles. All repos cloned to `~/src/` (depth 1) on
2026-10-04 and inspected directly; delegated per-repo reports with file/line citations live in
`notes/`. GitHub stats from the API on 2026-10-04.

## Primary repositories (code-level inspection)

| Project | URL | Key evidence |
|---|---|---|
| OpenClaw | github.com/openclaw/openclaw | `docs/concepts/memory-architecture.md`, `memory-builtin.md`, `docs/concepts/standing-intents.md`, `docs/automation/cron-jobs/*`, `docs/automation/standing-orders.md`, `docs/agent-runtime-architecture.md`, `docs/tools/exec-approvals.md`, `docs/gateway/sandboxing/`, `docs/security/THREAT-MODEL-ATLAS.md`, `src/state/openclaw-agent-schema.sql`, `packages/memory-host-sdk/src/host/memory-schema-*.ts`, `VISION.md` |
| Hermes | github.com/NousResearch/hermes-agent | `hermes_state_common.py` (SCHEMA_SQL), `hermes_state_fts.py`, `tools/memory_tool.py`, `agent/background_review.py` (review prompts), `agent/curator.py`, `tools/skill_ledger.py`, `hermes_cli/kanban_db.py`, `tools/approval.py`, `approval_smart.py`, `cron/scheduler.py`, `SECURITY.md`, `README.md` |
| CORE | github.com/RedPlanetHQ/core | `packages/database/prisma/schema.prisma`, `services/knowledgeGraph.server.ts`, `services/agent/context.ts`, `services/tasks/*`, `services/skills.defaults.ts` (Watch Rules), `jobs/scratchpad/scratchpad-scan.logic.ts` (empty branch), `hosting/docker/docker-compose.yaml`, AGPL-3.0 LICENSE |
| DailyOS | github.com/stadimeti19/DailyOS | `packages/database/src/schema.ts`, `packages/core/src/planning/daily-plan.ts`, `adaptive-replan.ts`, `packages/core/src/agent/loop.ts`, `apps/cli/src/commands/chat.ts` (timeline-proposer prompt), `docs/architecture.md` |
| Nudge | github.com/thatsjet/nudge-app | `app-bundle/system-prompt.md`, `src/main/agenticLoop.ts`, `default-vault/ideas/_template.md`, `src/shared/types.ts`, `prd.md` |
| PersonalOS | github.com/amanaiproduct/personal-os | MCP server (`core/`), `AGENTS.md`, `Tasks/` conventions, frontmatter schema |
| adhd-day-planner | github.com/Dthen/adhd-day-planner | `SKILL.md` (planning heuristics), gather/cleanup bash scripts |
| evidenceofLife-v2 | github.com/Cyriellewu/evidenceofLife-v2 | `useTodos.ts`, `autoSchedule.ts`, `schedulingProfile.ts`, `carryTodos.ts`, `planTimelinePrimitives.tsx` |
| personal-assistant | github.com/beranradek/personal-assistant | heartbeat gate, episodic store schema, integration proxy, security hooks (see notes/personal-assistant.md) |

## Ecosystem survey (README/code-glance verification)

Taskwarrior taskwarrior.org / github.com/GothenburgBitFactory/taskwarrior (urgency coefficients
docs); Timewarrior; Habitica; org-mode/org-agenda (GNU docs); Letta github.com/letta-ai/letta;
mem0 github.com/mem0ai/mem0; Zep/Graphiti github.com/getzep/graphiti (bi-temporal model docs);
basic-memory github.com/basicmachines-co/basic-memory; Khoj github.com/khoj-ai/khoj;
Goose github.com/block/goose; Agent Zero github.com/frdel/agent-zero; Huginn github.com/huginn/huginn;
n8n; ntfy github.com/binwiederhier/ntfy; Vikunja github.com/go-vikunja; Super Productivity
github.com/johannesjo/super-productivity; screenpipe github.com/mediar-ai/screenpipe;
karakeep github.com/karakeep-app/karakeep; EntangledQuantum/Life_OS; hermes-life-os; abi/lilo;
Motion/Reclaim/FlowSavvy/Sunsama/Morgen/Akiflow (commercial auto-scheduling market, docs).
License flags: Frona (BSL), screenpipe & n8n (non-OSI), CORE (AGPL-3.0), personal-os
(CC BY-NC-SA).

## Peer-reviewed research (citations verified via search; details in notes/adhd-research.md)

- Gollwitzer, P. M. (1999). Implementation intentions. *American Psychologist* 54(7). DOI: 10.1037/0003-066X.54.7.493
- Gollwitzer & Sheeran (2006). Implementation intentions and goal achievement: meta-analysis. *Adv. Exp. Soc. Psychol.* 38. DOI: 10.1016/S0065-2601(06)38002-1
- Barkley, R. A. (1997). *ADHD and the Nature of Self-Control*; (2012). *Executive Functions*. Guilford.
- Buehler, Griffin & Ross (1994). Exploring the planning fallacy. *JPSP* 67(3). DOI: 10.1037/0022-3514.67.3.366
- Roy, Christenfeld & McKenzie (2005/2008). Underestimating duration (reference-class correction).
- Risko & Gilbert (2016). Cognitive offloading. *Trends Cogn Sci* 20(9). DOI: 10.1016/j.tics.2016.07.002
- Cowan (working memory capacity ~4); Paas et al. (2020) cognitive-load management. *Perspect Psychol Sci*.
- Huang et al. (2021) time perception meta-analysis, *J Atten Disord*; Marx et al. (2022) *JAACAP*; Metcalfe et al. (2024) *Dev Neuropsychol*.
- McDaniel & Einstein prospective-memory program (event- vs time-based cues).
- Mace et al. (1985). Reactivity in self-monitoring. *Behav Modif* 9(3).
- Lally et al. (2010). Habit formation (median 66 days). *Eur J Soc Psychol* 40(6). Wood & Neal (context-cued habits).
- Amabile & Kramer (2011). *The Progress Principle*; The power of small wins. *HBR*.
- Safren et al. (2005 *Psychol Med*; 2010 *JAMA*) CBT for adult ADHD; Solanto et al. (2010) meta-cognitive therapy. *Am J Psychiatry* 167(9).
- Body doubling survey (2024). *ACM ASSETS*. n=220; weak evidence.
- LongMemEval (arXiv:2410.10813) — cited by OpenClaw docs for curation-over-indexing; Generative
  Agents reflection (arXiv:2304.03442); sleep-time compute (arXiv:2504.13171) — cited by
  OpenClaw dreaming design.

## Productivity systems (sources in notes/productivity-systems.md)

Allen, *Getting Things Done* (next action, contexts, weekly review, someday/maybe, waiting-for,
2-minute rule, natural planning model); Forte, *Building a Second Brain*/PARA (areas vs
projects); Luhmann Zettelkasten; Carroll, *The Bullet Journal Method* (rapid logging,
migration-as-signal); time blocking/timeboxing literature; Eisenhower matrix; Kanban WIP
limits (Anderson); Pomodoro (Cirillo); "eat the frog" (Tracy); progress-principle-adjacent
motivation research.

## Internal research artifacts

- `notes/_parent-verification.md` — orchestrator-verified facts (runtime docs, schemas).
- `notes/*.md` — 11 delegated reports (nudge, dailyos, personalos, core, openclaw, hermes,
  small-planners, personal-assistant, ecosystem-survey, adhd-research, productivity-systems).
- Briefs used: `~/projects/executor-research/briefs/` (transient process artifacts; the
  delegated-report prompts). Workflow scripts also kept there.

## Known source limitations

- Shallow clones (depth 1): history/recency taken from GitHub API, not local git logs.
- GitHub issue sampling was light (open-issue lists only); no maintainer interviews.
- Literature pass is a design-principles extraction, not a systematic review; confidence levels
  and contradictions are recorded in notes/adhd-research.md.
