# Runtime, proactivity, and phone access (Questions H + I)

## The runtime decision

### What a runtime must supply (Executor's actual infra needs)

always-on daemon on a home box • phone-reachable chat with approval round-trips • scheduler
with event/condition triggers • session persistence + compaction • provider abstraction
(including local models) • skills mechanism • memory substrate conventions • security
machinery • active maintenance.

Both OpenClaw and Hermes supply all nine, hardened far beyond a one-person rebuild (each has
5.6k–15.5k test files; the delivery ledgers, crash recovery, and channel fleets alone are
person-years). **Rebuilding any of this is the single largest avoidable cost in the project.**

### OpenClaw vs Hermes for Executor

| Dimension | OpenClaw | Hermes |
|---|---|---|
| Language | TypeScript/Node (+ Rust bits) | Python 3.14 (+ Node TUI) |
| Phone surface | messaging (40+ channels) **+ native iOS/Android/macOS apps + Control UI web + board widgets** | messaging (~25 platforms) only; desktop Electron |
| Memory model | Markdown canonical + derived SQLite index + provenance tiers (aligns with Git/Markdown plan) | SQLite/FTS5 canonical sessions; tiny curated md; skills accrete |
| Prospective memory | standing intents (deterministic FTS triggers) + automations + heartbeats | cron + wake-gates; goals loop |
| Learning loop | dreaming consolidation (gated) | after-turn review + curator + skill ledger (deeper) |
| Security | exec approvals (argv+identity binding), sandboxes, threat models, provenance-as-defense | floors + guardian LLM + circuit breaker + frozen YOLO + protected files |
| MCP | client | client **and** server |
| Extensibility | plugin SDK, own-SQLite plugin pattern, skills | plugin catalog, "plugins never touch core", skills |
| Weight | Node daemon | Python+Node, heavier |
| Churn | weekly releases, huge but fast | hourly commits, godfiles |

**Recommendation: OpenClaw**, primarily for the phone surface (native apps + Control UI +
board widgets give a credible non-chat "today" surface immediately), the markdown-canonical
memory that matches the Git/Markdown requirement, and standing-intents' deterministic trigger
design. **But build the domain layer runtime-agnostic** (own SQLite + MCP server + small HTTP
API) so switching to Hermes — or to a bare daemon — is a configuration change, not a rewrite.
Both runtimes treat MCP-speaking components as first-class; that's the portability seam.

### Deployment shape

```
home box (always-on)
├── OpenClaw gateway (channels, automations, sessions, memory core)
│     ├── workspace/            ← EXECUTOR.md, skills/, memory/ (markdown, git-versioned)
│     └── plugin/MCP client ────┐
└── executor-core (new code)    │
      ├── SQLite (canonical entities + FTS5)
      ├── planner/expander engine (deterministic + bounded LLM calls)
      ├── MCP server + tiny HTTP API (today view, capture, expansion UI)
      └── markdown canon in git (project pages, journal, knowledge notes)

phone → WhatsApp/Telegram/Signal (chat + approvals + briefings)
      → native app / Control UI / small web "today" view (PWA over the HTTP API)
```

## Proactivity (Question H): briefings, check-ins, and not becoming notification spam

Prior art: OpenClaw automations/heartbeat (outcome states, active hours, min spacing, flood
control), Hermes cron wake-gates, personal-assistant's **HEARTBEAT_OK suppression gate**
(deterministic cadence → context packet → LLM turn → notify only if output warrants), CORE's
watch-rules-as-markdown with "surfacing ≠ acting", DailyOS morning briefing.

Recommended policy set:

1. **Scheduled touches, few and fixed:** morning brief (MIT proposal, one tap to confirm),
   evening recap (what got done — progress principle; never what didn't), weekly review
   (agent-prepared: stale projects, aging commitments, open diagnoses — the human makes
   decisions, the agent does the clerical work; the productivity review burden moves to the
   system).
2. **Event-driven touches, gated:** trigger-matched reminders (implementation-intention
   cues at point of performance), deadline-risk escalations, unblock notices. All pass an
   output gate (heartbeat-gate pattern): a touch that would say nothing new is suppressed
   deterministically.
3. **Budgets and adaptation:** per-channel daily caps, quiet hours, min spacing; cadence
   auto-reduces when notification→action latency degrades (habituation).
4. **Policy as data:** the user's proactivity rules live in editable markdown (CORE watch-rules
   pattern) — "surface immediately / batch / handle silently" per source.
5. **Standing intents for the long tail:** "when the landlord emails → draft reply task" as
   deterministic triggers with fire budgets — no LLM in the trigger path.

## Phone interaction (Question I)

- **Chat plane:** WhatsApp/Telegram/Signal via the runtime — capture ("call executor, 'basement
  Saturday, need contractor bags'"), queries ("what now"), approvals, briefings. This is the
  zero-install, always-in-pocket surface; both runtimes are built around it.
- **View plane:** a small PWA / Control-UI board widget over executor-core's HTTP API: today
  card (now/next/MITs), one-tap start/done/defer, expansion reader. Chat is great for capture
  and terrible for at-a-glance state; you need both.
- **Push:** runtime channel notifications cover most; ntfy (self-hosted, one HTTP call) is the
  fallback for native push without a messaging account.
- **Remote access security:** bind to Tailscale/WireGuard (OpenClaw's own remote-access
  doctrine); never expose the gateway port publicly; phone on the same tailnet.
- **Offline/degraded:** executor-core must serve today-view from SQLite locally; the runtime
  being down degrades conversation, not the data.
