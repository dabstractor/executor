# Security — a system holding one person's entire life (Question J)

## Threat model specifics for Executor

The database is the user's whole life: tasks, commitments, journal, health-adjacent habits,
relationships (counterparties), locations eventually. Distinct threats beyond a coding agent's:

1. **Data loss/corruption** (the life-DB is irreplaceable).
2. **Exfiltration** (model provider, plugin, prompt-injected agent sending data out).
3. **Prompt injection through ingested content** (email, calendar, web, message relays — the
   same channels that feed commitments).
4. **Agent self-modification** (agent rewriting its own instructions/memory to "help" better).
5. **Unauthorized remote access** (gateway exposed to the internet).
6. **Supply chain** (skills/plugins installed into a life-trusted context).

## What the two runtimes get right (verified in source)

**OpenClaw:** exec approvals binding canonical argv + resolved executable identity (content
hash for writable executables), "approvals only tighten"; 5 sandbox backends (off by default);
memory provenance classes as the structural defense against injection-via-memory (untrusted
origin can never promote; network-tool results taint the turn); device pairing with signed
challenges; Tailscale/SSH-tunnel remote-access doctrine; formal threat-model docs
(THREAT-MODEL-ATLAS), ClawHub supply-chain analysis with badges.

**Hermes:** approval floors → dangerous-command detection → guardian-LLM smart approval with
denial circuit-breaker; **YOLO env frozen at import** (in-process code can't flip it);
protected instruction files (AGENTS.md/SOUL.md writes always need human approval, even YOLO);
memory writes staged with matched-entry pinning (concurrent-edit invalidation); skill security
scan + AST audit + content-addressed ledger with rollback; cron prompts scanned for credential
exfiltration; ESTOP file; single documented trust boundary: OS-level isolation (their own
SECURITY.md is honest that in-process heuristics are not boundaries).

**DailyOS (patterns for the domain layer):** risk tiers read/local_write/external_write/
destructive; approval stores proposed vs **decided** (editable) arguments; injection doctrine
verbatim in every prompt touching external content ("external_untrusted_data … never
instructions"); full audit trail tables.

**personal-assistant:** secrets-isolated integration proxy — OAuth tokens live in a separate
process the agent cannot read; the agent holds scoped handles only. **Adopt this.**

## Executor's security model (recommended)

1. **Isolation first:** executor-core runs as its own user/service; the runtime's agent talks
   to it only via MCP/HTTP with scoped tools. Compromised agent ≠ database write access.
2. **Secrets partition:** all OAuth/API credentials live in the runtime's credential store or a
   proxy process (personal-assistant pattern), never in workspace files the model reads.
3. **Tiered write policy on the life-DB** (DailyOS tiers): agent may freely create/modify
   *draft* tasks, expansions, inferences, episodic memory; changing commitments, deadlines,
   routines, or promoted memory requires confirmation; destructive ops (delete project,
   drop history) require typed confirmation + are soft-deleted with undo.
4. **Append-only history:** no agent path mutates task_event/session rows. Ever.
5. **Ingestion is untrusted:** email/calendar/web text enters as evidence with untrusted
   provenance (OpenClaw classes); LLM outputs from ingestion are validated drafts; commitment
   extraction is idempotent and user-confirmed.
6. **Remote access:** Tailscale-only; no public gateway port; phone devices on the tailnet;
   pairing + allowlists for messaging channels (runtime provides).
7. **Backups:** SQLite WAL + nightly snapshot + markdown canon in git (pushed to private
   remote — this is *the* argument for the Git/Markdown half: durable, diffable, encrypted-
   at-rest-friendly). Test restores.
8. **Supply chain:** no unvetted ClawHub/community skills in the life workspace; pin plugin
   versions; skill installs are approval-gated (Hermes pattern).
9. **Egress control:** the runtime's network policy (OpenClaw net-policy / Hermes exfil scans)
   restricts where the agent can send data; expansion/summarization calls go to the configured
   provider only.
10. **Kill switch:** one command suspends automations + gateway turns (ESTOP pattern).

## Honest residual risks

- Any system whose agent can run shell commands on the home box has a large blast radius if
  the model is adversarially steered; sandbox backends and approval tiers mitigate but do not
  eliminate (both projects' own docs say so). Keep the life-DB out of the sandbox's reach.
- Model providers see prompt content; if that's unacceptable, local models (both runtimes
  support Ollama/llama.cpp/vLLM) at reduced capability — decide per data class.
- Prompt injection via ingested email remains the hardest live problem; provenance tiers +
  draft-confirmation flow is the current best practice, not a proof.
