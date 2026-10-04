# Executor

## Product Requirements Document + Technical Specification

**Status:** Implementation specification / v0.1
**Repository:** `~/projects/executor`
**Project name:** Executor
**Primary purpose:** Personal AI executive-function system for task initiation, contextual task decomposition, planning, and behavioral learning.

---

# 1. Executive Summary

Executor is a personal AI system designed to make it substantially easier to decide **what to do next**, begin doing it, and recover when execution stalls.

The system is not primarily a conventional task manager.

Its central job is:

> Given everything Executor knows about the user's projects, commitments, available time, context, history, and recent behavior, identify the small number of actions that are genuinely worth doing next, and make beginning one of them extremely easy.

A user should be able to encounter:

> **Clean the basement**

and, with one interaction, receive something more like:

> **Start here**
>
> 1. Go downstairs.
> 2. Turn on the basement lights.
> 3. Walk to the grow-tent area.
> 4. Put the contractor bags beside the tent.
> 5. Stop there and tell Executor what you found.
>
> **Why this first:** the bags are already available and this is the shortest physical path to making progress on the current remediation project.

Executor must use the user's actual context to produce this kind of decomposition rather than relying on generic productivity advice.

The system must also learn from what actually happens.

If Executor repeatedly recommends a task that is deferred, ignored, abandoned, or consistently takes much longer than expected, that information must affect future planning.

The system therefore has two fundamental loops:

### Immediate loop

```text
Candidate work
    ↓
Selection
    ↓
"Three things worth doing next"
    ↓
User chooses
    ↓
Contextual micro-plan
    ↓
User starts
    ↓
Execution
    ↓
Completion / defer / stuck / abandon
```

### Long-term loop

```text
Observed behavior
    ↓
Append-only event history
    ↓
Derived observations
    ↓
Explicitly marked inferences
    ↓
Planning / estimation / intervention adjustments
    ↓
Different future recommendations
    ↓
New observed behavior
```

Executor must never silently convert an inference into a fact.

---

# 2. Goals

## 2.1 Primary goals

Executor must:

1. Maintain a durable representation of the user's projects, tasks, commitments, routines, and relevant context.
2. Provide a very small set of useful next actions rather than overwhelming the user with a giant prioritized list.
3. Allow any meaningful task to be expanded into concrete, context-aware steps.
4. Make the first action sufficiently concrete that the user does not have to perform additional planning before beginning.
5. Record actual behavior rather than merely planned behavior.
6. Learn from planned vs. actual execution.
7. Detect and support stalled execution.
8. Preserve user control over consequential actions.
9. Be usable from a phone while the primary system runs on an always-on desktop.
10. Keep the domain model independent of whichever agent runtime is eventually selected.
11. Store important durable information in inspectable, portable formats.
12. Make all important behavior reproducible and debuggable.

---

# 3. Non-goals

Executor v0.1 is **not** intended to be:

* A general-purpose project-management SaaS.
* A team collaboration system.
* A replacement for a calendar.
* A generic chatbot with a task list attached.
* A fully autonomous household agent.
* A motivational-gamification system.
* A streak system.
* A guilt/reminder system.
* A social productivity application.
* A generic knowledge-management system.
* A universal personal-data aggregator.
* An autonomous system that silently changes important user commitments.
* A system that requires the LLM to be correct for database integrity.

---

# 4. Core Product Principle

## 4.1 The product is not the task list

A conventional task manager answers:

> What tasks exist?

Executor must answer:

> Given everything I know about this person and their situation, **what is worth doing now?**

And then:

> How can I make starting that thing require as little executive-function work as possible?

This distinction should guide every architectural decision.

---

# 5. Primary User Experience

## 5.1 The three-next-actions model

Executor should generally surface approximately three meaningful opportunities.

The exact number must be configurable, but three is the default product concept.

Example:

```text
RIGHT NOW

1. Finish the Executor database migration
   ~25 min
   Start: run the migration test.

2. Order replacement printer nozzle
   ~5 min
   Start: open the saved product comparison.

3. Clear the basement staging area
   ~15 min
   Start: take the contractor bags downstairs.
```

The purpose is not to claim that these are objectively the three most important things in the universe.

The purpose is to reduce:

* choice overload
* working-memory requirements
* planning overhead
* task-initiation friction

while still making meaningful progress.

---

# 6. Task Expansion

A task is not necessarily executable merely because it has a title.

Executor must support contextual expansion.

Example:

```text
Task:
Clean basement
```

could expand differently depending on:

* current project state
* previous work
* available materials
* physical location
* time available
* user's current context
* known dependencies
* previous failed attempts
* related commitments
* known preferences
* environmental constraints
* current task state

The expansion result should contain:

```text
Objective
First physical action
Ordered steps
Stopping condition
Expected duration
Relevant context
Potential blockers
Optional next step
```

The expansion must be treated as a **versioned proposal**, not as canonical truth.

---

# 7. "I'm Stuck" Is a First-Class Operation

The system must not treat inability to proceed as equivalent to failure.

The user must be able to say:

> I'm stuck.

or:

> This isn't working.

or:

> I can't do this right now.

Executor should then diagnose the likely problem.

Possible categories:

* unclear next action
* missing information
* missing material
* physical/environmental obstacle
* dependency
* task too large
* task incorrectly scoped
* wrong priority
* insufficient energy/time
* emotional resistance
* unexpected complexity
* external dependency
* user simply wants to do something else

The categorization may be proposed by the LLM but must be stored as an observation/inference rather than presented as psychological fact.

---

# 8. Behavioral Model

Executor must distinguish at least:

### Planned

What Executor expected to happen.

### Attempted

The user actually began interacting with the task.

### Completed

The user reports or otherwise establishes completion.

### Deferred

The user deliberately moves it later.

### Blocked

The user cannot proceed because of an external dependency.

### Abandoned

The task is intentionally no longer being pursued.

### Stalled

The user began but progress stopped.

### Reopened

A previously completed/closed task becomes active again.

These distinctions matter because:

> "The user didn't finish it"

is insufficient information.

A task deferred because the user intentionally chose something more important is different from a task that was impossible because a required part was missing.

---

# 9. Architecture

## 9.1 High-level architecture

```text
                       PHONE
                         │
                         │
                  PWA / messaging
                         │
                         ▼
              ┌────────────────────┐
              │  Agent Runtime     │
              │  OpenClaw/Hermes   │
              └─────────┬──────────┘
                        │
                 MCP / HTTP seam
                        │
                        ▼
              ┌────────────────────┐
              │   executor-core    │
              │                    │
              │ Domain API         │
              │ Selection          │
              │ Expansion          │
              │ Behavior           │
              │ Memory             │
              │ Planning           │
              └───────┬─────┬──────┘
                      │     │
                 SQLite     Git/Markdown
                      │     │
                      └──┬──┘
                         │
                  Durable user data
```

The runtime is an adapter, not the application.

Executor must remain usable without OpenClaw/Hermes-specific code being embedded throughout the domain layer.

---

# 10. Runtime Boundary

## 10.1 Recommended initial runtime

OpenClaw is the preferred initial runtime because its gateway architecture already addresses:

* persistent agent execution
* messaging channels
* scheduling
* heartbeat/proactive behavior
* skills
* gateway authentication
* automation
* external tool integration

Its current documentation explicitly provides automation/cron infrastructure and gateway APIs.

Hermes remains a valid alternative and should not be structurally excluded. Its current MCP implementation supports both local stdio and remote HTTP MCP servers on the client side.

**Do not hard-code Executor's domain model around either runtime.**

---

# 11. executor-core

Executor core should be a standalone application/library.

The exact implementation language is an open decision unless the implementation environment establishes a stronger constraint.

The code should have conceptual modules equivalent to:

```text
executor/
    domain/
    storage/
    planning/
    selection/
    expansion/
    behavior/
    memory/
    api/
    runtime/
    cli/
    tests/
```

The actual package layout may differ.

Do not create artificial abstraction layers solely for theoretical portability.

---

# 12. Source of Truth

Executor has two complementary durable stores.

## 12.1 SQLite

SQLite is the canonical operational database.

Use it for:

* tasks
* projects
* relationships
* task state
* events
* sessions
* commitments
* routines
* expansion versions
* derived observations
* inferences
* planning metadata
* estimates
* indexes
* timestamps
* identifiers

SQLite must be queryable without the LLM.

---

## 12.2 Git/Markdown

Git-backed Markdown is the human-readable knowledge layer.

Use it for durable narrative/contextual information such as:

```text
projects/
    project-name.md

journal/
    YYYY-MM-DD.md

context/
    ...

core/
    ...
```

The exact directory hierarchy is intentionally not fixed yet.

Markdown should be:

* inspectable
* editable by the user
* version-controlled
* understandable without Executor

Do not attempt to mirror every database row into Markdown.

---

# 13. Canonical Data Model

The following entities are required conceptually.

## 13.1 Project

A project represents an outcome or body of work that may contain multiple tasks.

Fields:

```text
id
name
description
status
created_at
updated_at
completed_at
parent_project_id (optional)
priority / importance metadata
context reference
```

Status should support at least:

```text
active
paused
completed
archived
```

Do not add additional states unless required.

---

# 14. Task

Required fields:

```text
id
project_id
parent_task_id
title
description
status
created_at
updated_at
completed_at
due_at
scheduled_for
priority metadata
estimated_duration
context metadata
source
```

Task status must support at least:

```text
inbox
ready
planned
in_progress
blocked
deferred
completed
abandoned
```

The implementation may distinguish other internal states if useful.

Tasks may form a hierarchy.

For v0.1:

> Steps should normally be child tasks rather than a special second-class "step" data structure.

This makes steps observable and behaviorally measurable.

---

# 15. Task Event Log

Task events are append-only.

Never rewrite historical behavior.

Minimum event types:

```text
created
updated
planned
expanded
started
paused
resumed
completed
deferred
blocked
unblocked
abandoned
reopened
stuck
```

Each event should include:

```text
id
task_id
event_type
timestamp
actor
source
metadata
session_id (optional)
```

`metadata` may contain structured JSON.

Example:

```json
{
  "reason": "missing_material",
  "note": "Need 6-inch duct connector"
}
```

The event log is strategically important and must not be treated as merely application logging.

It is the raw behavioral dataset from which future planning intelligence is derived.

---

# 16. Event Sourcing Boundary

Executor does **not** need to become a fully event-sourced application.

The rule is narrower:

> User/task behavioral history is append-only.

Current task state may be materialized for efficient querying.

The event history must remain available.

---

# 17. Session Model

A session represents a period of active execution/planning.

A session should be able to capture:

```text
id
started_at
ended_at
context
planned_task_ids
attempted_task_ids
completed_task_ids
notes
```

A session should permit calculation of:

```text
planned duration
actual duration
number of task switches
number of deferrals
number of stalls
completion rate
```

Do not overbuild analytics in v0.1.

Store the raw information required to calculate them later.

---

# 18. Commitments

Executor must distinguish a commitment from an ordinary task.

Examples:

* appointment
* deadline
* promised delivery
* bill/payment obligation
* meeting
* externally imposed requirement

A commitment may be represented as:

```text
id
title
description
deadline
importance
consequences
related_task_id
source
status
```

The exact schema is still open.

The important distinction is:

> Executor must not treat all tasks as equally optional.

---

# 19. Routines

Routines are recurring patterns rather than individual one-off tasks.

Examples:

* morning startup
* weekly planning
* recurring maintenance
* recurring administrative work

Do not build a complex recurrence engine in Executor if the selected runtime already provides reliable scheduling.

Executor should own the semantic definition of a routine.

The runtime should own triggering where practical.

---

# 20. Estimates

Executor must eventually learn personal duration estimates.

Do not use generic productivity estimates when historical personal data exists.

For a task category, project, or action type, Executor may derive:

```text
observed duration
median duration
recent median
sample count
variance
confidence
```

The system should prefer robust statistics such as medians over simple averages where appropriate.

Exact estimation algorithm is intentionally deferred.

---

# 21. Selection Engine

Selection is one of Executor's most important components.

It should be deterministic-first.

## Pipeline

```text
Candidate retrieval
        ↓
Hard filtering
        ↓
Deterministic scoring
        ↓
Optional LLM reranking
        ↓
Three recommended actions
```

---

# 22. Candidate Retrieval

Candidate tasks can come from:

* active projects
* ready tasks
* due commitments
* scheduled work
* deferred tasks becoming eligible
* routines
* explicit user requests
* newly captured tasks

Candidate retrieval must be explainable.

---

# 23. Hard Filters

The deterministic layer should eliminate obviously invalid candidates.

Examples:

* completed tasks
* abandoned tasks
* tasks blocked by unresolved dependencies
* tasks whose prerequisites are incomplete
* tasks explicitly deferred until later
* tasks outside relevant schedule constraints
* tasks whose required resources are known to be unavailable

Do not let an LLM override hard safety/state constraints.

---

# 24. Deterministic Scoring

Potential scoring factors:

```text
deadline proximity
commitment importance
project importance
dependency unlocking
age
user-selected priority
estimated effort
available time
context match
recent deferral frequency
recent attempts
project neglect
```

The initial algorithm should remain simple.

A possible conceptual model:

```text
score =
    urgency
  + importance
  + unblock_value
  + context_match
  + momentum
  - effort_penalty
  - stale_deferral_penalty
```

Do not implement arbitrary numeric weights without documenting them.

The actual weights should be configurable and easy to tune.

---

# 25. LLM Reranking

The LLM may rerank the deterministic candidate set.

It must receive:

* candidate tasks
* relevant project context
* recent behavior
* current session context
* available time/context
* explicit commitments

It should return structured output:

```json
{
  "selected": [
    {
      "task_id": "...",
      "reason": "..."
    }
  ]
}
```

The LLM must not:

* mutate task state directly
* create commitments silently
* invent deadlines
* invent dependencies
* alter historical events
* overwrite canonical context

The domain layer validates its output.

If the LLM fails validation:

1. retry with a constrained repair prompt, if appropriate;
2. otherwise use the deterministic ranking.

Executor must continue functioning without an LLM response.

---

# 26. Expansion Engine

Expansion converts:

```text
Task + relevant context + current situation
```

into:

```text
Executable micro-plan
```

The LLM is the primary proposer here.

The result must be schema-validated.

Conceptual schema:

```json
{
  "task_id": "...",
  "objective": "...",
  "first_action": "...",
  "steps": [
    {
      "id": "...",
      "instruction": "...",
      "expected_minutes": 5
    }
  ],
  "stopping_condition": "...",
  "estimated_total_minutes": 20,
  "blockers": [],
  "context_used": []
}
```

This schema is illustrative, not final.

---

# 27. Expansion Rules

The expansion system should prefer:

* physical actions
* observable actions
* one action at a time
* concrete verbs
* existing resources
* known locations
* known tools
* known project state
* explicit stopping conditions

Avoid:

> "Work on the basement."

Prefer:

> "Take the three contractor bags from the utility room and place them beside the grow tent."

The first action should generally be small enough that the user can perform it without further planning.

---

# 28. Expansion Must Not Hallucinate Context

If Executor does not know whether something exists, it must not state that it exists as fact.

Bad:

> "Take the boxes from the shelf."

if Executor does not know there are boxes there.

Better:

> "If the boxes are still on the shelf, move them beside the tent. If they're not there, tell me what you find instead."

Known facts and inferred assumptions must remain distinguishable.

---

# 29. Context Retrieval

Context retrieval should be selective.

Do not dump the entire user database into every LLM prompt.

Retrieve information based on:

* task
* project
* dependencies
* recent history
* relevant entities
* semantic similarity where useful
* explicit references
* current situation

Context should have provenance.

The system should know where a piece of context came from.

---

# 30. Memory Model

Use three conceptual memory classes.

## 30.1 Curated/core memory

Small, high-confidence information that materially affects planning.

Examples:

* durable preferences
* project facts
* stable workflows
* important constraints
* known user conventions

---

## 30.2 Episodic memory

Large historical record.

Examples:

* task events
* sessions
* conversations
* completed work
* failures
* deferrals
* notes

This should not all be inserted into prompts.

---

## 30.3 Procedural memory

How the user/system performs recurring operations.

Examples:

```text
How Executor deploys itself
How a particular project is normally tested
How a recurring physical task is performed
```

Procedural knowledge may eventually become versioned skills.

---

# 31. Facts vs Inferences

This is mandatory.

Executor must distinguish:

```text
observed fact
user assertion
system-derived observation
LLM inference
hypothesis
confirmed correction
```

Example:

```text
Fact:
Task was deferred three times.

Observation:
This task has a high deferral frequency.

Inference:
The task may be too large or poorly scoped.

Not acceptable:
"The user avoids this task because they are anxious."
```

The latter is an unsupported psychological conclusion.

---

# 32. Inference Lifecycle

Derived knowledge should follow:

```text
observations
    ↓
candidate inference
    ↓
confidence
    ↓
validation through future behavior/user feedback
    ↓
promoted or discarded
```

Inferences must not silently become durable facts.

---

# 33. Intervention Model

Executor should eventually distinguish:

```text
recommendation
intervention
result
```

For example:

```text
Recommendation:
Do task X.

Intervention:
Break task X into one 2-minute physical action.

Result:
User started within 3 minutes and completed the task.
```

This permits Executor to learn not only:

> Which tasks succeed?

but:

> Which kind of assistance helps this person start?

This is a future-facing architectural requirement even if the first implementation is simple.

---

# 34. Behavioral Learning

The system should eventually learn:

* which task sizes are actually completed
* personal duration distributions
* which times/contexts produce successful starts
* which projects are repeatedly deferred
* which types of tasks generate stalls
* whether decomposition improves initiation
* which intervention styles work
* how often planned work is interrupted
* how much work the user realistically completes in a session

Do not implement machine learning merely for the sake of machine learning.

Start with deterministic derived statistics.

---

# 35. Behavioral Metrics

Useful derived metrics include:

```text
time_to_start
planned_duration
actual_duration
estimate_error
completion_rate
deferral_rate
abandonment_rate
stall_rate
restart_rate
task_switch_rate
steps_completed
```

These should be derivable from event/session history.

---

# 36. Time Estimation

The user should generally not be required to estimate task duration.

Executor should estimate.

Historical personal behavior should eventually dominate generic estimates.

Example:

```text
Generic expectation:
15 minutes

User historical median:
31 minutes

Executor estimate:
~30 minutes
```

Exact statistical methodology remains open.

---

# 37. Planning Horizon

Executor should distinguish:

### Now

What should be done next.

### Today

What is realistically achievable today.

### Later

Important but not currently actionable.

### Someday/parked

Valid ideas that should not compete with current work.

Do not expose all four as equal-weight lists in the primary UX.

---

# 38. Daily Planning

The system should support a daily plan but avoid pretending the user's entire day can be predicted precisely.

A daily plan should contain:

```text
critical commitments
likely useful work
optional work
```

and enough flexibility for replanning.

---

# 39. Replanning

Executor should re-evaluate recommendations when:

* a task completes
* a task is deferred
* a task becomes blocked
* the user says they are stuck
* a commitment changes
* available time changes
* a significant context change occurs

Do not require the user to manually rebuild the plan.

---

# 40. Phone UX

The phone is a primary interaction surface.

The initial implementation should support a small PWA or equivalent mobile-friendly interface.

Primary screen:

```text
TODAY

What matters now?

[ Task A ]
[ Task B ]
[ Task C ]

----------------

[ Capture something ]
```

Tapping a task should show its expansion.

Example:

```text
CLEAR STAGING AREA

Why:
This is blocking the next step in the basement project.

Start here:
Take the contractor bags downstairs.

[ Start ]

[ I'm stuck ]

[ Defer ]

[ Not doing this ]
```

The interface should optimize for one-handed use.

---

# 41. Messaging Interface

Messaging is a secondary but important interface.

Examples:

```text
What's next?

What should I do for the next 20 minutes?

I'm stuck.

I finished that.

Defer this until tomorrow.

Capture: need to order duct connector.
```

The runtime should handle channel transport.

Executor should provide semantic operations.

---

# 42. API

Executor should expose a stable domain API.

Potential operations:

```text
GET  /health
GET  /tasks
POST /tasks
GET  /tasks/{id}
PATCH /tasks/{id}
POST /tasks/{id}/events
POST /tasks/{id}/expand
POST /tasks/{id}/start
POST /tasks/{id}/complete
POST /tasks/{id}/defer
POST /tasks/{id}/stuck
GET  /today
POST /capture
GET  /projects
GET  /projects/{id}
```

The exact transport and URL structure may change.

The important requirement is that runtime adapters should invoke domain operations rather than manipulate SQLite directly.

---

# 43. MCP Boundary

An MCP server is a suitable runtime seam.

Potential tools:

```text
executor_get_today
executor_get_task
executor_create_task
executor_update_task
executor_expand_task
executor_start_task
executor_complete_task
executor_defer_task
executor_mark_stuck
executor_capture
executor_get_project_context
executor_get_relevant_history
```

Tool names are illustrative.

MCP should expose semantic operations rather than raw database access.

---

# 44. Runtime Permissions

The agent should have different classes of capability.

## Read-only

Examples:

```text
read tasks
read project context
read history
read today
```

## User-state mutation

Examples:

```text
create task
defer task
complete task
record event
```

## External side effects

Examples:

```text
send message
send email
purchase something
delete data
modify external system
```

External side effects require explicit approval unless a future policy specifically grants them.

---

# 45. Security Model

Executor contains sensitive personal information.

Minimum requirements:

* local-first operation
* authenticated remote access
* no unauthenticated public API
* secrets outside Git
* explicit environment/config separation
* audit trail for mutations
* no secret values in logs
* principle of least privilege
* runtime tool permissions
* database backups

Do not expose SQLite directly to the internet.

---

# 46. Remote Access

The recommended initial topology is:

```text
Phone
  ↓
secure private network / authenticated gateway
  ↓
home desktop
  ↓
Executor
```

Do not expose Executor's API directly to the public internet unless there is a compelling reason.

The exact remote-access mechanism remains an implementation decision.

---

# 47. Git Repository

Repository should contain:

```text
README.md
LICENSE
.env.example
.gitignore

src/
tests/

migrations/

docs/
    architecture.md
    api.md
    operations.md

data/
    .gitkeep

context/
    ...

projects/
    ...

journal/
    ...

scripts/

config/
```

Exact layout may change.

Do not commit:

```text
.env
API keys
tokens
database files
private transcripts
runtime state
session secrets
```

unless the user explicitly chooses otherwise.

---

# 48. Configuration

Configuration must be explicit.

Potential categories:

```text
EXECUTOR_DATA_DIR
EXECUTOR_DB_PATH
EXECUTOR_CONTEXT_DIR
EXECUTOR_LOG_LEVEL
EXECUTOR_API_HOST
EXECUTOR_API_PORT

LLM_PROVIDER
LLM_MODEL
LLM_BASE_URL
LLM_API_KEY

RUNTIME_PROVIDER

OPENCLAW_URL
OPENCLAW_TOKEN

TIMEZONE

GIT_REPOSITORY_PATH
```

These are examples, not necessarily the final names.

The implementation agent must determine provider-specific variables from the actual selected provider/runtime documentation rather than inventing them.

---

# 49. Required Pre-Implementation Information

The coding agent must gather all of this **before beginning implementation**.

It must produce a preflight report.

## 49.1 Runtime choice

Ask/determine:

```text
OpenClaw or Hermes?
```

If neither is already installed, research current installation requirements.

The agent must not silently install a competing runtime.

---

## 49.2 LLM provider

Determine:

```text
provider
API endpoint
model
API key availability
local vs remote inference
vision requirement
embedding requirement, if any
```

The agent should support an OpenAI-compatible provider where practical.

Do not assume a specific provider.

---

## 49.3 Messaging channel

Determine the first phone interaction mechanism.

Possible options:

* PWA only
* Telegram
* Mattermost
* Signal
* WhatsApp
* another runtime-supported channel

Do not implement every channel.

---

## 49.4 Remote networking

Determine:

```text
Will phone access occur through Tailscale/private networking?
Will a reverse proxy be used?
Is the service LAN-only?
What hostname/address should be used?
```

Do not expose ports publicly without explicit configuration.

---

## 49.5 User timezone

Must be explicitly configured.

Do not infer it from the server's timezone.

---

## 49.6 Git identity

The agent needs:

```text
git user.name
git user.email
```

if it will create commits.

---

## 49.7 Data directory

Confirm where Executor's durable data should live.

Default candidate:

```text
~/projects/executor
```

but runtime-generated state should not automatically be stored in Git.

---

# 50. Required Environment Variables

Before implementation, the agent must generate an exact checklist based on the selected runtime/provider.

At minimum, determine whether these are required:

```text
LLM API credentials
runtime credentials
messaging credentials
remote access credentials
Git credentials, if remote Git operations are required
optional embedding credentials
```

The agent must tell the user exactly:

```text
VARIABLE
PURPOSE
WHETHER REQUIRED
WHERE TO OBTAIN IT
SAFE EXAMPLE FORMAT
WHETHER IT WILL BE STORED OR ONLY PASSED THROUGH
```

It must never ask the user to paste a secret into chat if the secret can instead be entered into a local environment/config file.

---

# 51. Required sudo / privileged setup

Before implementation begins, the agent must determine every operation that requires elevated privileges.

Examples may include:

```text
system package installation
systemd service installation
reverse-proxy configuration
firewall changes
Tailscale setup
directory ownership changes
```

The agent must produce a single preflight section:

```text
RUN THESE COMMANDS BEFORE CONTINUING
```

with exact commands.

It must not casually execute `sudo`.

It must not use destructive commands.

It must not modify system configuration until the user has explicitly completed the preflight.

If no sudo commands are required, state:

```text
No sudo actions required.
```

---

# 52. Preflight Contract

The implementation agent must begin with:

```text
PHASE 0 — DISCOVERY
```

It may inspect the machine without sudo.

It must determine:

* OS
* architecture
* shell
* Python/Node/etc. versions
* Git
* Docker availability
* systemd availability
* selected runtime installation
* selected runtime version
* network topology
* Tailscale/private network status if relevant
* existing ports
* existing Executor repository state
* available SQLite version
* selected LLM/provider configuration
* available messaging integrations

It must not make architectural assumptions based solely on defaults.

---

# 53. Preflight Output

Before changing anything, the agent must create:

```text
docs/IMPLEMENTATION_PREFLIGHT.md
```

containing:

1. discovered environment
2. selected architecture
3. required external accounts
4. required API keys
5. required environment variables
6. required privileged commands
7. services to be created
8. ports to be used
9. directories to be created
10. unresolved decisions
11. assumptions
12. risks

The user should be able to complete the entire prerequisite phase without another interaction with the coding agent.

---

# 54. Installation Philosophy

The agent should prefer:

1. existing user-installed tools
2. project-local dependencies
3. user's existing runtime
4. containerized dependencies where appropriate

over:

```text
system-wide installation
```

Do not install language runtimes globally merely for convenience.

---

# 55. Service Management

Executor should eventually run unattended.

Preferred architecture:

```text
executor-core service
runtime/gateway service
```

The exact service mechanism depends on the selected environment.

On a normal Linux host, systemd is a reasonable default.

Do not require Docker if native deployment is simpler.

Do not require native deployment if Docker is already the user's established infrastructure.

---

# 56. Backups

At minimum:

```text
SQLite backup
Git-backed Markdown backup
configuration backup without secrets
```

The database must support consistent backups.

The event log makes historical recovery particularly valuable.

Backup policy details remain open.

---

# 57. Logging

Logs must distinguish:

```text
application logs
LLM requests/responses
security/audit events
task events
runtime events
```

Do not log API keys.

Avoid logging entire private LLM prompts/responses by default if they contain sensitive personal information.

Debug logging may be explicitly enabled.

---

# 58. Error Handling

Executor must fail safely.

If the LLM is unavailable:

```text
today view still works
task CRUD still works
events still record
deterministic selection still works
```

If SQLite is temporarily unavailable:

```text
do not claim mutations succeeded
```

If expansion fails:

```text
show a useful fallback task description
```

If runtime communication fails:

```text
core remains independently testable
```

---

# 59. LLM Contract

Every LLM call should have:

```text
purpose
input schema
output schema
validation
timeout
retry policy
fallback
logging policy
```

Do not use free-form LLM text where structured output is required.

---

# 60. Prompt Architecture

Prompts should be treated as implementation artifacts.

Organize them by function:

```text
selection
expansion
stuck diagnosis
capture classification
context extraction
inference generation
```

Prompts must not contain rules that belong in deterministic code.

For example:

Bad:

> "Never complete a task that is already completed."

That belongs in domain logic.

Good:

> "Given these candidate tasks, rank them according to the supplied criteria."

---

# 61. Capture

The user must be able to dump unstructured thoughts into Executor.

Example:

> "Need to figure out what to do with that old computer in the basement and maybe sell it."

Executor should propose:

```text
Project: Basement
Task: Decide disposition of old computer
Possible subtasks:
...
```

But classification must be reviewable.

Do not silently turn every conversational statement into a task.

---

# 62. Capture Pipeline

Conceptually:

```text
raw capture
    ↓
LLM classification
    ↓
structured proposal
    ↓
validation
    ↓
persist
```

Potential result types:

```text
task
project
note
commitment
idea
question
context update
ignore
```

Exact taxonomy remains open.

---

# 63. User Corrections

Corrections are important behavioral data.

If Executor says:

> "You need to order X."

and the user says:

> "No, I already bought it."

the system should:

1. correct the current state;
2. record the correction;
3. avoid continuing to propagate the stale fact;
4. potentially identify the stale source.

The correction should not require editing an LLM prompt manually.

---

# 64. Task Dependencies

Executor should support dependencies.

Example:

```text
Task B requires Task A.
```

Dependencies should be explicit data.

LLMs may propose dependencies.

The domain layer must validate them.

---

# 65. Contextual Eligibility

A task may be eligible only when:

```text
at home
at computer
at store
has material
has enough time
after prerequisite
during a particular time
```

Context matching is important, but the exact context taxonomy is not yet decided.

Do not build a huge ontology prematurely.

Start with a small extensible representation.

---

# 66. Physical-World Context

Because many useful tasks are physical, Executor should eventually support information such as:

```text
location
required tools
required materials
physical prerequisites
environment
```

This is particularly important for high-quality task expansion.

Do not assume Executor can automatically observe the physical environment.

Only use known or user-provided information.

---

# 67. Completion

Completion should be intentionally cheap.

A task should normally be completable with one tap/message.

Example:

```text
[Done]
```

may be enough.

Executor may optionally ask:

> How did that go?

but must not make every completion a questionnaire.

---

# 68. Deferral

Deferral should be semantically meaningful.

Instead of simply changing a date, record:

```text
deferred
timestamp
reason if supplied
new intended time/date if supplied
```

The user should not be forced to provide a reason.

---

# 69. Abandonment

Abandonment is not failure.

Examples:

* no longer relevant
* superseded
* intentionally rejected
* duplicate
* wrong idea

Record it without guilt-oriented UX.

---

# 70. UX Tone

Executor should be:

* direct
* concrete
* calm
* nonjudgmental
* action-oriented

Avoid:

* guilt
* shame
* streak pressure
* fake enthusiasm
* excessive praise
* gamification
* productivity clichés

The system should not say:

> "You can do it! 💪"

unless the user explicitly wants that style.

---

# 71. Notifications

Notifications must be useful rather than constant.

Potential future notification classes:

```text
important commitment approaching
task became unblocked
planned session starting
user explicitly requested reminder
```

Avoid repeated nagging.

A reminder that has already been ignored should not simply repeat indefinitely.

---

# 72. Proactivity

Executor may proactively wake up.

Examples:

```text
morning briefing
daily planning
commitment warning
evening review
blocked-task detection
```

The runtime should handle scheduling.

Executor should determine semantic content.

Current OpenClaw automation supports recurring and event-driven scheduling, making this boundary practical.

---

# 73. Morning Briefing

v0.1 should eventually support:

```text
Good morning.

Today has:
- commitment A
- commitment B

The three things most worth doing:
1. ...
2. ...
3. ...

If you only do one thing, do X.
```

Do not produce a huge daily report by default.

---

# 74. Evening Review

v0.1 may support:

```text
Completed:
...

Deferred:
...

Still blocked:
...

Anything important to reconsider tomorrow?
```

The review should primarily collect useful behavioral information.

---

# 75. Testing Strategy

Testing must emphasize deterministic domain behavior.

## Unit tests

Required for:

* task state transitions
* dependency rules
* event recording
* candidate filtering
* deterministic scoring
* estimate calculations
* inference state transitions
* expansion validation

## Integration tests

Required for:

* SQLite persistence
* API
* MCP adapter
* runtime adapter
* Git/Markdown persistence

## LLM contract tests

Use fixtures for:

* valid expansion
* malformed expansion
* hallucinated fields
* missing required fields
* invalid task IDs
* invalid state transitions
* unsafe recommendations

The application must remain testable without making live LLM calls.

---

# 76. Deterministic Test Fixtures

Create fixtures such as:

```text
small_project
blocked_project
overdue_commitment
repeatedly_deferred_task
long_running_task
successful_microtask
failed_expansion
```

Tests should prove that the same state produces the same deterministic selection.

---

# 77. Acceptance Test: Core Loop

A v0.1 implementation is not complete until this works:

```text
Create project
    ↓
Create several tasks
    ↓
Executor calculates candidate set
    ↓
Executor produces three recommendations
    ↓
User taps one
    ↓
Executor retrieves relevant context
    ↓
Executor generates structured expansion
    ↓
User starts task
    ↓
Executor records started event
    ↓
User completes/defer/stalls
    ↓
Executor records event
    ↓
Next recommendation changes appropriately
```

---

# 78. Acceptance Test: Failure Loop

This must also work:

```text
Task selected
    ↓
User says "I'm stuck"
    ↓
Executor records stuck event
    ↓
Executor asks/infers why
    ↓
Executor proposes a smaller next action or identifies blocker
    ↓
User continues/defer/block
    ↓
History is preserved
```

---

# 79. Acceptance Test: LLM Failure

Disable the LLM.

The system must still:

* start
* read tasks
* create tasks
* record events
* show current state
* calculate deterministic recommendations
* complete tasks
* defer tasks

Only intelligent expansion/reranking should degrade.

---

# 80. Acceptance Test: Data Integrity

Verify:

* completed tasks cannot accidentally become active through malformed LLM output
* historical events are never modified
* duplicate event handling is deterministic
* invalid foreign keys are rejected
* API mutations are authenticated
* secrets never appear in Git
* restart does not lose state

---

# 81. Operational CLI

A CLI is strongly recommended.

Potential commands:

```bash
executor doctor
executor status
executor today
executor task list
executor task show <id>
executor task create
executor task complete <id>
executor task defer <id>
executor task stuck <id>
executor project list
executor expand <id>
executor events <id>
executor db backup
executor export
```

The exact command names are not mandatory.

The CLI should make the system usable without the phone or LLM.

---

# 82. `executor doctor`

This command should check:

```text
database
migrations
configuration
LLM connectivity
runtime connectivity
Git repository
data directory
permissions
network/API binding
required environment variables
```

It should clearly distinguish:

```text
OK
WARNING
ERROR
```

---

# 83. Database Migrations

Schema changes must use migrations.

Do not modify the production database schema ad hoc from application startup.

Migration history must be tracked.

The migration mechanism should be simple and inspectable.

---

# 84. IDs

Use stable IDs.

UUIDs are a reasonable default.

IDs must not encode semantic information.

---

# 85. Timestamps

Store timestamps in UTC.

Convert to the user's configured timezone at presentation time.

Do not rely on the machine timezone.

---

# 86. Concurrency

SQLite is sufficient for v0.1.

The application should avoid unnecessarily long transactions.

Concurrent API requests must not corrupt task state.

Use database transactions for state transitions and event insertion.

---

# 87. State Transition Atomicity

A state-changing operation should generally:

```text
BEGIN
update current state
insert event
COMMIT
```

If either operation fails, neither should appear to have happened.

---

# 88. Derived Data

Derived observations may be recomputed from event history.

Do not make irreversible decisions based solely on cached derived data.

Caches should be disposable.

---

# 89. Search

Initial search may use:

* SQLite indexes
* full-text search
* simple metadata filters

Semantic/vector search should only be added if actual context-retrieval quality requires it.

Do not add a vector database merely because an AI project "should" have one.

---

# 90. Git/Markdown Synchronization

Markdown and SQLite must not compete as independent sources of truth for the same mutable fields.

Recommended boundary:

```text
SQLite:
operational state

Markdown:
human-authored durable context
```

If the user manually edits Markdown, Executor should treat that as authoritative contextual input after re-indexing.

Do not implement bidirectional synchronization of every field.

---

# 91. Import / Export

Executor should eventually support exporting:

```text
tasks
projects
events
context
```

into portable formats.

v0.1 only needs a basic export/backup path.

---

# 92. Privacy

Assume all Executor data is private.

The implementation must make no assumption that sending all data to an external LLM is acceptable.

Context passed to the LLM should be deliberately selected.

The provider boundary should make local inference possible.

---

# 93. Provider Abstraction

Executor should not embed one vendor's API throughout the code.

The minimum conceptual interface is:

```text
generate_structured(...)
generate_text(...)
```

Potential future capabilities:

```text
vision
embeddings
tool calling
streaming
```

Do not implement these until required.

---

# 94. Model Roles

It may eventually be useful to distinguish:

```text
fast/cheap model
reasoning model
vision model
embedding model
```

But v0.1 should support the minimum model configuration necessary.

Do not prematurely build a multi-model orchestration framework.

---

# 95. Agent Autonomy

Executor must have explicit autonomy boundaries.

### Safe automatic actions

Examples:

* calculate recommendations
* create derived observations
* generate task expansions
* record user events
* update internal planning metadata

### User-visible mutations

Examples:

* create/modify tasks
* defer work
* change project state

These should be attributable to Executor and auditable.

### External side effects

Examples:

* purchases
* messages to third parties
* emails
* destructive operations
* changes to external systems

Require explicit approval initially.

---

# 96. Auditability

For every agent-originated mutation, retain:

```text
actor = user | agent | system
source
timestamp
related session
```

This allows the user to answer:

> Why does Executor think this?

---

# 97. Explainability

The user should be able to ask:

> Why are you recommending this?

The response should cite relevant reasons such as:

```text
Due tomorrow
Unblocks project X
You already started it
You've deferred it three times
You have approximately 30 minutes available
```

Do not expose fabricated chain-of-thought.

Expose concise decision factors instead.

---

# 98. Recommendation Diversity

Executor should avoid returning three nearly identical tasks when alternatives provide better coverage.

Potential diversity dimensions:

* project
* effort
* context
* urgency
* task type

Exact diversity algorithm remains open.

---

# 99. Energy / Capacity

Executor should eventually account for actual capacity.

Do not require the user to manually assign a complex energy score to every task.

Possible signals:

* available time
* recent completion rate
* time of day
* current session length
* historical performance

The exact model is not yet decided.

---

# 100. "Minimum Viable Progress"

For large tasks, Executor should identify a meaningful minimum unit of progress.

Example:

```text
Task:
Set up basement remediation area.

Minimum progress:
Move the required storage containers into the staging area.
```

This is different from arbitrarily making a task smaller.

The micro-action must move the actual project forward.

---

# 101. Project Context

Each project should have enough contextual information for expansion.

Potential project document:

```markdown
# Project Name

## Objective

## Why this matters

## Current state

## Known constraints

## Resources

## Decisions

## Open questions

## Recent work

## Next likely actions
```

This is an example structure, not a mandatory schema.

---

# 102. Project History

Executor should be able to summarize:

```text
what happened recently
what remains
what is blocked
what was decided
what changed
```

This is more useful than dumping every event into the prompt.

---

# 103. Context Compression

Large project histories should eventually be summarized into durable project context.

However:

> summaries are derived artifacts, not replacements for history.

Original events remain available.

---

# 104. Agent Context Assembly

Before an intelligent operation, Executor should assemble a bounded context package.

Conceptually:

```text
system rules
+
current task
+
parent project
+
relevant project facts
+
recent task history
+
relevant behavioral observations
+
current session
+
user request
```

The package should be inspectable in debug mode.

---

# 105. Implementation Order

The coding agent should implement in this order.

## Phase 0 — Discovery

No implementation.

Produce preflight.

---

## Phase 1 — Foundation

Implement:

* project
* task
* task hierarchy
* task state
* SQLite
* migrations
* event log
* basic CLI
* tests

---

## Phase 2 — Deterministic planning

Implement:

* candidate retrieval
* eligibility
* scoring
* today/next-three operation
* duration handling
* dependencies

---

## Phase 3 — LLM expansion

Implement:

* context retrieval
* expansion schema
* validation
* retry
* fallback
* expansion versioning

---

## Phase 4 — Behavioral loop

Implement:

* sessions
* start/completion/defer/stuck
* behavioral metrics
* derived observations
* personal duration statistics

---

## Phase 5 — Runtime integration

Implement:

* MCP/domain adapter
* OpenClaw/Hermes integration
* messaging
* scheduled briefing
* runtime authentication

---

## Phase 6 — Mobile UX

Implement:

* PWA
* today view
* task expansion
* start
* complete
* defer
* stuck
* capture

---

## Phase 7 — Proactive behavior

Implement:

* morning briefing
* evening review
* commitment alerts
* selective proactive intervention

---

# 106. v0.1 Definition of Done

v0.1 is complete when the user can:

1. Capture a task.
2. Organize it under a project.
3. See the three most useful things to do.
4. Tap one.
5. Get a context-aware micro-plan.
6. Start it with one interaction.
7. Mark it complete.
8. Defer it.
9. Say they're stuck.
10. Have all of those actions recorded.
11. Restart the system without losing state.
12. Access the system from the phone.
13. Receive a basic daily briefing.
14. Inspect the underlying task/history data.
15. Continue using the core system if the LLM is unavailable.

---

# 107. v0.1 Explicitly Excluded

Do not implement initially:

* autonomous purchasing
* autonomous email
* broad browser automation
* complicated habit systems
* gamification
* streaks
* social features
* multi-user accounts
* complex calendar synchronization
* dozens of integrations
* vector databases unless required
* custom ML training
* elaborate analytics dashboards
* autonomous modification of external services
* sophisticated psychological profiling

---

# 108. Recommended Repository Documentation

The implementation should produce:

```text
README.md
docs/architecture.md
docs/data-model.md
docs/api.md
docs/runtime.md
docs/security.md
docs/operations.md
docs/IMPLEMENTATION_PREFLIGHT.md
docs/open-questions.md
```

The final implementation should also document:

```text
how to start Executor
how to run tests
how to backup
how to restore
how to inspect database
how to configure LLM
how to configure runtime
how to configure phone access
```

---

# 109. Agent Implementation Rules

The coding agent must follow these rules.

## Rule 1

Do not invent missing environmental information.

Inspect it.

## Rule 2

Do not invent credentials.

Ask for them only in the preflight phase.

## Rule 3

Do not use sudo without first listing the command in the preflight.

## Rule 4

Do not expose services publicly without explicit configuration.

## Rule 5

Do not let LLM output directly mutate canonical state.

## Rule 6

Do not delete historical behavioral events.

## Rule 7

Do not silently promote LLM inferences to facts.

## Rule 8

Do not build runtime functionality inside executor-core when the runtime already provides it.

## Rule 9

Do not introduce a dependency merely because it is fashionable.

## Rule 10

Every new persistent field must have a reason.

---

# 110. Unknowns / Decisions Intentionally Deferred

This specification intentionally does **not** resolve the following.

These are the areas the next research/design pass should address.

## 110.1 OpenClaw vs Hermes

The architectural seam supports either.

The final choice should be based on:

* current stability
* user's preferred workflow
* messaging support
* local model support
* MCP behavior
* security model
* operational simplicity
* resource usage

Current research indicates both are viable runtime foundations.

---

## 110.2 Exact database schema

The entities and invariants are defined here.

Exact SQL tables, indexes, constraints, and JSON fields should be designed after reviewing the implementation environment.

Do not pretend the final schema is known yet.

---

## 110.3 Exact selection algorithm

The architecture is intentionally:

```text
filter → deterministic score → optional LLM rerank
```

but the correct scoring factors and weights require experimentation with actual user behavior.

Do not hard-code a supposedly scientifically optimal formula.

---

## 110.4 Exact ADHD intervention model

The product should be informed by executive-function research, but the system should learn which interventions work for this particular user.

The first implementation should therefore instrument interventions rather than claiming to know the optimal intervention policy.

---

## 110.5 Exact context taxonomy

Possible dimensions include:

```text
location
device
available tools
available materials
time
energy/capacity
social context
environment
```

The correct minimum set should be established empirically.

---

## 110.6 Memory implementation

The conceptual distinction between:

```text
curated
episodic
procedural
```

is important.

Whether implementation uses:

* SQLite FTS
* embeddings
* filesystem search
* runtime memory
* a dedicated memory system

is not yet settled.

Do not add a memory product until retrieval requirements justify it.

---

## 110.7 PWA vs runtime-native phone UI

The system needs a very good phone interaction.

Whether the first version should be:

```text
PWA
runtime web UI
messaging-first
hybrid
```

should be decided after inspecting the selected runtime's capabilities.

---

## 110.8 Notification mechanism

The runtime likely already solves much of this.

Executor should define **what** deserves notification.

The runtime should determine **how** it reaches the phone.

---

## 110.9 Calendar integration

Calendar is potentially valuable because commitments strongly affect task selection.

However, calendar integration should not be added until the core loop works.

---

## 110.10 External integrations

Potential future integrations include:

```text
calendar
email
GitHub
filesystem
home automation
browser
financial systems
location
```

None should be considered mandatory for v0.1.

---

## 110.11 Learning algorithm

Do not prematurely build reinforcement learning.

Start with:

```text
event history
→ deterministic statistics
→ explicit observations
→ bounded inference
```

Then evaluate whether anything more sophisticated is justified.

---

# 111. Second-Session Research Agenda

The next design session should specifically resolve:

1. OpenClaw vs Hermes.
2. Exact runtime-to-Executor MCP/API boundary.
3. Exact SQLite schema.
4. Exact task state machine.
5. Exact selection scoring model.
6. Behavioral metrics.
7. Intervention taxonomy.
8. Context model.
9. Memory/retrieval implementation.
10. Phone UX architecture.
11. Authentication/remote-access architecture.
12. Backup/restore architecture.
13. Calendar/commitment model.
14. Daily planning algorithm.
15. LLM provider/model strategy.
16. Exact preflight requirements for the user's machine.
17. Deployment/service architecture.
18. Test fixtures and acceptance suite.

These are the areas where additional research or actual environmental inspection can materially improve the implementation.

---

# 112. The Most Important Product Test

Ignore every feature in this document for a moment.

The system succeeds if this interaction becomes genuinely useful:

```text
User:
What's next?

Executor:
Three things are worth doing:

1. Finish X.
   Start by doing Y.

2. Do A.
   Start by doing B.

3. Handle C.
   Start by doing D.

User:
1

Executor:
Start here:
Y.

[Start]
[I'm stuck]
[Defer]

User:
[Start]

Executor:
Recorded.

...

User:
Done.

Executor:
Recorded.

Next:
Z.
```

And over weeks:

```text
Executor becomes better at choosing
what this particular user can and should
actually do next.
```

That is the product.

Everything else exists to make that loop reliable.

---

# 113. Implementation Principle

The fundamental architectural rule is:

> **Code owns truth. The LLM proposes. History records what actually happened.**

More specifically:

```text
Code:
    state
    invariants
    permissions
    transitions
    persistence
    scheduling semantics
    event history
    validation

LLM:
    interpretation
    decomposition
    summarization
    contextual reasoning
    candidate explanation
    bounded reranking
    inference proposals

Runtime:
    sessions
    messaging
    scheduling
    gateway
    authentication
    tool transport

User:
    reality
    corrections
    decisions
    approvals
    completion
```

The system should be designed so that a wrong LLM response is an ordinary recoverable application error rather than a corruption of the user's life-management state.

---

# 114. Final Implementation Directive

A naive implementation agent receiving this specification should **not begin coding immediately**.

It must first:

```text
1. Inspect the existing repository.
2. Inspect the host environment.
3. Determine the runtime.
4. Determine the LLM provider/model.
5. Determine phone transport.
6. Determine network topology.
7. Determine required credentials.
8. Determine required environment variables.
9. Determine every required sudo operation.
10. Produce IMPLEMENTATION_PREFLIGHT.md.
11. Stop and present the complete prerequisite checklist.
```

Once those prerequisites have been satisfied, implementation proceeds without requiring the agent to rediscover infrastructure requirements halfway through the build.

Where this specification says something is **open**, the implementation agent must research and/or make a documented local decision rather than silently treating an arbitrary choice as part of the product contract.

The implementation should optimize for:

**small surface area, inspectability, deterministic behavior, durable history, excellent task initiation, and the ability to improve from observed behavior.**
