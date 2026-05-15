# Outbox Contract

Use an outbox contract when an agent runtime cannot provide a reliable machine callback. The outbox is the agent's structured completion signal.

## Required Properties

- Run-specific path or key.
- Exactly one JSON object.
- Includes `runId` and `intent`.
- Includes `ready` and `riskLevel`.
- Includes environment understanding.
- Includes decisions or proposed changes appropriate to the intent.
- Includes evidence or summary sufficient for audit.

## Generic Schema

```json
{
  "runId": "string",
  "intent": "string",
  "provider": "codex",
  "ready": true,
  "riskLevel": "low",
  "summary": "string",
  "environmentUnderstanding": {
    "dev": "string",
    "runtime": "string",
    "data": "string"
  },
  "decisions": [],
  "proposedCommands": [],
  "checksExpected": []
}
```

Allowed `riskLevel` values:

- `low`
- `medium`
- `high`

Recommended rule: block `high` automatically.

## Release Publish Decision

```json
{
  "candidateId": "string",
  "action": "approve",
  "confidence": 0.91,
  "reason": "Official source confirms the release and duplicate checks passed.",
  "evidenceUrls": ["https://example.com/release-notes"]
}
```

Allowed actions:

- `approve`
- `reject`
- `needs_review`

Recommended rules:

- Reject unknown candidate ids.
- Require source evidence for approve.
- Keep low-confidence approve as pending or needs_review.
- Treat duplicate releases as reject or needs_review.

## Code Deploy Decision

```json
{
  "runId": "2026-05-15T12-00-00Z-code_deploy",
  "intent": "code_deploy",
  "provider": "codex",
  "ready": true,
  "riskLevel": "medium",
  "summary": "Implemented the change and expects validate/build/health gates to pass.",
  "environmentUnderstanding": {
    "dev": "Repo uses npm validation and Docker Compose build.",
    "runtime": "Production runs as compose services behind the public domain.",
    "data": "Persistent state is in the database volume; no schema migration required."
  },
  "decisions": [],
  "proposedCommands": ["npm run validate", "npm run build"],
  "checksExpected": ["validate", "build", "local_health", "public_health"]
}
```

Recommended rules:

- The harness decides which commands actually run.
- The agent may propose commands, but the harness should run a fixed allowlisted gate set.
- `ready=true` starts gates; it does not mean deployment succeeded.

## Validator Checklist

Validate:

- JSON parses and is an object.
- `runId` matches active run.
- `intent` matches active run.
- `ready` is boolean.
- `riskLevel` is allowed.
- `environmentUnderstanding.dev/runtime/data` are non-empty.
- Decisions reference current queue items only.
- Actions are allowed for the intent.
- Confidence values are numbers in range 0 to 1.
- High risk blocks.

## Failure Handling

Timeout:

- Mark run failed or blocked.
- Keep context path and expected outbox path.
- Capture provider/session metadata.

Invalid JSON:

- Mark failed.
- Store parse error and raw path.
- Do not apply side effects.

Ready false:

- Mark blocked.
- Preserve summary and evidence for operator review.

Stale run id:

- Mark failed.
- Do not apply side effects.

Unknown candidate/action:

- Mark failed or blocked.
- Do not partially apply unknown decisions.
