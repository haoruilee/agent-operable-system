---
name: agent-operable-system
description: Design, implement, or review systems that should be safely operated by an AI agent or interactive AI CLI. Use when Codex is asked to make a service agent-operable, build an autonomous operations harness, define tmux/outbox/queue/event-log patterns, let Codex or Claude Code review releases or deployments, add admin monitoring for agent behavior, or assess whether an agent-driven ops architecture is safe and complete.
---

# Agent-Operable System

Use this skill to turn a normal service into a system an AI agent can operate safely. The core pattern is: give the agent rich context and inspection ability, require a structured outbox as the completion signal, and let a deterministic harness own validation, publish/deploy side effects, event logging, and rollback boundaries.

## Core Model

An agent-operable system has four planes:

1. **Work plane**: source registry, collectors, queues, candidates, jobs, incidents, or deployment requests.
2. **Agent plane**: a real agent runtime such as Codex CLI, Claude Code, or another reviewer/executor.
3. **Harness plane**: deterministic code that creates context, starts or prompts the agent, waits for outbox, validates output, runs gates, and applies approved actions.
4. **Observation plane**: durable runs, lifecycle events, metrics, logs, and an admin UI that explains what the agent and harness did.

Keep the agent plane and harness plane separate. The agent can inspect, reason, edit, and recommend. The harness owns final authority.

## Workflow

1. **Inventory the real system**
   - Locate repo root, deploy path, runtime type, environment files, data stores, queues, workers, timers, and health endpoints.
   - Separate development, runtime, and data planes explicitly.
   - Record what is persistent, what is rebuildable, and what may be rotated.

2. **Define agent intents**
   - Start from concrete intents such as `release_publish`, `code_deploy`, `incident_triage`, `dependency_update`, or `data_repair`.
   - For each intent, define allowed observations, allowed edits, forbidden direct actions, required gates, and success criteria.

3. **Create the context bundle**
   - Generate a run-specific context file containing run id, intent, repo/runtime facts, queue items, health checks, git status, deploy status, and exact outbox path.
   - Include a required protocol that says the agent must write exactly one JSON object to the outbox and must not directly publish or deploy unless the intent explicitly allows it.

4. **Use a durable completion contract**
   - Do not infer completion from terminal output, chat text, or process exit.
   - Use a run-specific outbox JSON file, database row, or message queue event.
   - Require run id and intent matching to prevent stale results from being applied.

5. **Run the agent in a controlled host**
   - For interactive CLIs, use tmux or an equivalent TTY host with known session names.
   - Prefer a clean session per run when contamination between runs is risky.
   - Preserve login state only where needed; never make the web app container the owner of host-level deploy powers.

6. **Validate before side effects**
   - Parse and schema-check the outbox.
   - Require environment understanding for dev/runtime/data.
   - Block high-risk output, missing evidence, stale run ids, unknown queue items, and invalid actions.
   - Treat `ready` as the agent's opinion, not as proof that publish/deploy succeeded.

7. **Let the harness apply**
   - For publish flows, call the application API or queue API from the harness.
   - For deploy flows, run deterministic gates such as validate, tests, build, container restart, health checks, worker checks, and timer checks.
   - Make each side effect idempotent or safely retryable.

8. **Persist full lifecycle events**
   - Store both final run snapshots and event streams.
   - Log phases such as `queued`, `context_written`, `goal_injected`, `waiting_for_outbox`, `outbox_received`, `outbox_validated`, `ready`, `blocked`, `apply_started`, `published`, `deployed`, `failed`.
   - Expose these events in an admin page; do not make operators read tmux panes as the source of truth.

9. **Prove it with failure drills**
   - Test timeout, invalid JSON, stale run id, `ready=false`, high risk, duplicate work item, provider unavailable, failing gate, and API auth failure.
   - Confirm the admin UI shows where the run stopped.

## Required Boundaries

- Do not let the agent directly own production publish/deploy authority.
- Do not parse free-form agent text as a machine contract.
- Do not rely on a callback unless the agent runtime provides a stable, supported callback protocol.
- Do not start overlapping runs for the same intent unless the queue and actions are explicitly partitioned.
- Do not hide agent behavior in logs only; persist queryable run and event records.
- Do not mark a run successful just because the agent wrote `ready: true`; apply gates must pass.

## Resource Guide

- Read `references/harness-blueprint.md` when implementing the architecture in a repo.
- Read `references/outbox-contract.md` when defining the completion schema, validator, and examples.
- Read `references/ops-readiness-checklist.md` when reviewing whether a system is truly ready for agent operations.

## Output Expectations

When using this skill, produce concrete artifacts:

- file paths and modules to add or change
- runtime topology and trust boundaries
- outbox schema
- harness lifecycle
- event names and database/API shape
- scheduler and tmux/session strategy when applicable
- validation and deploy gates
- admin monitoring requirements
- failure-mode tests or drills

Prefer implementation over advice when the user asks to build it. If only reviewing, lead with concrete gaps and operational risks.
