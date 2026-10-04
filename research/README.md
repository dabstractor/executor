# Executor Research — Architectural Reconnaissance

**Question:** What should actually be built, reused, forked, or discarded to create *Executor* — a
persistent, always-on personal AI that functions as an external executive-function layer for one
person with severe ADHD?

**Method:** 9 candidate repositories cloned to `~/src/` and inspected at the code/schema level
(11 delegated research children + orchestrator spot-verification), a survey of ~25 additional
ecosystem projects, and a literature pass over ADHD/executive-function research and established
productivity methodologies. Every load-bearing claim cites a file path in a cloned repo, a repo
URL, or a paper. Where README claims and code disagree, the notes say so.

## Read this first

1. `RESEARCH_SUMMARY.md` (in `~/projects/executive/`) — 2-page executive summary with the final recommendation.
2. `recommendations.md` — answers to the 10 architecture questions (data model, storage split, autonomy boundaries, daily selection, v1 scope, reuse list).
3. `landscape.md` — the ecosystem map and the convergence thesis.
4. `project-comparisons.md` — per-project verdicts (reuse / pattern-donor / avoid) with the evidence.

## Topic files

| File | Question it answers |
|---|---|
| `task-model.md` | A — canonical data model (entities + DDL sketch) |
| `architecture-patterns.md` | B — deterministic vs LLM boundary |
| `memory.md` | C — memory architecture (markdown/SQLite/FTS/vectors/graphs) |
| `behavior-learning.md` | D + G — learning from behavior; failure handling |
| `prioritization.md` | E — "what should I do now" selection algorithm |
| `adhd-design.md` | ADHD/EF evidence → design principles (with citations) |
| `mobile-and-runtime.md` | H + I — proactivity, phone access, runtime choice |
| `security.md` | J — security model for a system holding a person's entire life |

## Supporting material

- `notes/` — the 11 delegated deep-dive reports (one per project + literature/systems/survey), each with file-cited evidence. Start with these if any conclusion here seems wrong.
- `notes/_parent-verification.md` — facts the orchestrator verified directly in the source repos.

## One-paragraph conclusion

Build Executor as a **new, self-contained life-domain layer** (its own SQLite + Markdown + a
deterministic planner/expander engine, exposed over MCP and a small HTTP API) running **on top of
an existing agent runtime — OpenClaw recommended, Hermes the credible alternative** — which
already solves the always-on gateway, phone access over real messaging apps, scheduling,
sessions, provider abstraction, and a provenance-gated memory substrate. No existing project
solves task micro-decomposition from life context or composed behavioral learning; that is
Executor's genuinely open territory, and it is a domain layer, not a runtime.
