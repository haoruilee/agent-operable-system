# Agent Operations Readiness Checklist

Use this checklist to review whether a system can safely be operated by an AI agent.

## Environment Model

- Development commands are documented and runnable.
- Runtime topology is explicit.
- Data stores and persistent volumes are identified.
- Rebuildable files and rotatable runtime files are identified.
- Public and local health checks exist.
- Rollback path exists and is documented.

## Work Intake

- Work items are stored in a registry, queue, or database.
- Each item has a stable id.
- Item state is explicit, such as pending, reviewing, approved, rejected, applied, failed.
- Duplicate detection exists where repeated ingestion is possible.
- Source evidence or provenance is retained.

## Agent Context

- Each run gets a context bundle.
- Context includes exact outbox path.
- Context includes allowed and forbidden actions.
- Context includes environment model.
- Context includes ready criteria.
- Context includes relevant recent failures and current health.

## Completion Contract

- Completion does not depend on free-form terminal output.
- Outbox or equivalent structured completion exists.
- Run id and intent must match.
- High-risk output is blocked.
- Missing environment understanding is blocked.
- Invalid queue references are blocked.

## Side-Effect Control

- Agent does not directly publish production changes.
- Harness owns publish/deploy side effects.
- Apply steps are idempotent or safely retryable.
- Deploy gates are deterministic and allowlisted.
- Dry-run mode exists.

## Scheduling and Concurrency

- Scheduler is explicit.
- Active run or lease prevents overlap.
- Timeouts are defined.
- Stale active runs can be inspected and cleared.
- Long runs are visible to operators.

## Observability

- Runs table or equivalent exists.
- Events table or equivalent exists.
- Events include phase, severity, message, metadata, timestamp.
- Admin UI shows run status and event stream.
- Operators can inspect context/outbox paths when appropriate.
- Provider failures and gate failures are visible.

## Security

- Harness API is authenticated.
- Secrets are not written into context unless necessary.
- Public web app does not expose outbox files.
- Agent permissions are deliberately scoped.
- Production data writes require harness validation.

## Failure Drills

Test these before trusting autonomous operation:

- provider command missing
- provider not logged in
- invalid outbox JSON
- outbox timeout
- stale run id
- ready false
- high risk
- duplicate work item
- failed validation
- failed build
- failed health check
- unauthorized harness API call

## Done Criteria

The system is agent-operable when an operator can answer all of these from structured state:

- What work did the agent receive?
- What context did it see?
- Which provider ran?
- Did it write outbox?
- Did outbox pass validation?
- Did the agent claim ready?
- Which gate failed or passed?
- What side effect did the harness apply?
- How would we roll back?
