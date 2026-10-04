# Landscape — the open-source ecosystem around Executor

## The convergence thesis

Independent projects built in 2025–2026 for "personal AI that acts in your life" have converged,
without coordination, on the same stack shape (evidence: openclaw.md, hermes.md,
personal-assistant.md, core.md, ecosystem-survey.md notes):

```
always-on agent runtime (home box / VPS)
  + messaging gateway for phone access (Telegram/WhatsApp/Signal/…)
  + scheduler (cron + heartbeats + event/condition triggers)
  + sessions persisted in SQLite, search via FTS5
  + memory split into: small curated core (markdown) + episodic log + procedural skills
  + MCP as the tool/integration seam
  + skills as markdown instruction files
```

Every ingredient of Executor exists somewhere in open source **except its two core innovations**:

1. **Context-driven micro-decomposition** — "clean the basement" → "get the contractor bags from
   the shelf; open the left grow tent; …" generated from accumulated life context.
2. **Composed behavioral learning** — planned/attempted/completed telemetry turned into better
   selection, estimation, and initiation support, with observed facts kept distinct from agent
   inference.

No surveyed project does either. The nearest misses: evidenceofLife-v2 has the telemetry but no
agent; nudge has decomposition-ish prose in prompts but no code, context, or learning; DailyOS
and CORE have agents but flat task models and no behavioral loop.

## Categories

### 1. Agent runtimes (solve: always-on, phone, scheduling, sessions, providers)

| Project | What it is | Verdict |
|---|---|---|
| **OpenClaw** (TS, MIT, ~391k★) | Multi-channel AI gateway daemon; markdown+SQLite memory with provenance gates; automations/heartbeats/standing-intents; plugin SDK; native phone apps | **Reuse as substrate** |
| **Hermes** (Python, MIT, ~251k★) | Self-improving agent: closed learning loop (skills+memory from experience), SQLite+FTS5 sessions, ~25 gateway platforms, layered approvals, MCP client+server | Reuse as substrate (alternative) |
| Goose, Agent Zero, n8n, Huginn | Generic agent/automation runtimes | Skip (no life substrate, weaker fit) |

### 2. "Life OS" agents (solve: parts of the domain layer)

| Project | What it is | Verdict |
|---|---|---|
| **CORE** (RedPlanetHQ, AGPL, ~2k★) | Personal AI OS: temporal KG (Neo4j+pgvector), Task+Page+thread, watch rules, edit-buffer HITL | Pattern-donor only (AGPL, 4-service SaaS stack, work-oriented) |
| **DailyOS** (TS, 1★) | Deterministic day planner + bounded LLM proposer + approval tiers | Pattern-donor (validate/repair pipeline, approvals, injection doctrine) |
| **Nudge** (Electron, 23★) | ADHD chat app over markdown vault; excellent behavioral system prompt | Pattern-donor (prompt = behavior spec; capture-time micro-steps) |
| **PersonalOS** (markdown conventions + MCP server, CC BY-NC-SA, dormant) | Claude-Code-as-life-manager convention | Pattern-donor (frontmatter schema, WIP caps, clarify-gate) |
| beranradek/personal-assistant (Python, 9★, active) | Serious harness (heartbeat gate, episodic memory, integration proxy) around Claude Code/Codex CLI | Pattern-donor (heartbeat gate, proxy pattern; ToS risk noted) |
| EntangledQuantum/Life_OS (37★) | "Agent designs the system, user taps"; nightly self check-in; ADHD-affirming UX contract | Watch / transplant UX ideas |
| hermes-life-os (199★) | 4×daily cron briefings + weekly review over life logs | Watch (proactivity skeleton) |
| abi/lilo | Telegram-first assistant, voice/photo capture, git workspace | Pattern-donor (phone UX) |

### 3. Planners & trackers (solve: data model + learning shape, no agent)

| Project | What it is | Verdict |
|---|---|---|
| **evidenceofLife-v2** (React/Supabase, ~53k LOC) | Planned-vs-actual on every todo; steps-as-child-todos with timers; conservative median-based learning; ghost-preview acceptance | **Best data model found — copy schema concepts** |
| **adhd-day-planner** (948-line Hermes skill) | All planning heuristics live in a SKILL.md prompt (ADHD tax, meal anchors, momentum starter) | Steal the heuristic corpus, enforce in code |
| Taskwarrior + Timewarrior | Battle-tested task store, urgency coefficients, hooks, real time tracking | Pattern/component option; urgency formula = prior art |
| Habitica, Super Productivity, Vikunja, org-mode | Habit gamification / timeboxing / kanban / agenda | Reference UX; org-agenda = canonical deterministic planner prior art |

### 4. Memory components

- **basic-memory** (4.1k★, AGPL) — markdown-first knowledge graph over MCP; candidate component for the knowledge-note store.
- **Letta/MemGPT** — memory blocks + sleep-time agents; reference architecture for nightly consolidation.
- **mem0, Zep/Graphiti, cognee** — memory servers; Graphiti's bi-temporal fact invalidation is the right *model* (adopt model, not the Neo4j deployment).
- Khoj — self-hosted second brain; knowledge-side reference.

### 5. Infra glue

- **ntfy** — self-hosted push to any phone with one HTTP call (notification channel if runtime push is insufficient).
- screenpipe — local 24/7 screen/audio capture ("what did I actually do" ground truth; heavy privacy tradeoff).

### 6. Thin wrappers (deliberately skipped)

"Chat with your notes" wrappers dominate search results for "AI second brain / personal
assistant." They have no persistent model, no scheduling, no behavior loop. Category noted only
so nobody re-surveys it: they are not foundations.

## Demand signal & history

- ADHD-support skills for coding agents draw tens of thousands of stars (ecosystem-survey.md) —
  the audience exists and is unserved for *life* management specifically.
- Earlier (2018–2021) Telegram EF-bot attempts died from lacking a memory substrate; the
  persistence layer is the historical differentiator between toys and systems.
- The commercial auto-scheduling market (Motion, Reclaim, FlowSavvy, Sunsama) has forked on
  "decide for the user vs guide the user" — Executor should take a position (see
  prioritization.md).

## Strategic conclusion

The substrate is a solved problem (don't rebuild it). The domain layer is unsolved (don't expect
to import it). The correct posture is: **consume a runtime, own the domain.**
