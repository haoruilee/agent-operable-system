# Agent-Operable Harness Blueprint

Use this blueprint when implementing an autonomous agent operations harness in a real service.

## Architecture

Minimum complete architecture:

```text
Source/Work Registry -> Queue -> Harness Scheduler -> Context Builder
                                         |
                                         v
                              Interactive Agent Runtime
                              Codex CLI / Claude Code / other
                                         |
                                         v
                                  Outbox Contract
                                         |
                                         v
                         Validator -> Gates -> Apply Actions
                                         |
                                         v
                            Runs + Events + Admin Monitor
```

## Runtime Placement

Recommended placement:

- App/web containers run the product.
- Data volumes persist product state.
- Harness runs on the host or in a privileged ops container only if it needs Docker/systemd/git/CLI access.
- Interactive CLIs run in tmux, systemd user services, or another real TTY host.
- Harness runtime files live outside immutable build artifacts, for example `var/agent-harness/`.

Avoid putting host-level deploy authority inside the public web container.

## Directory Layout

Typical layout:

```text
var/agent-harness/
  context/
    <runId>.md
  outbox/
    <runId>.json
  state/
    active-run.json
```

`context/` is input to the agent. `outbox/` is structured output from the agent. `state/` holds busy guards and current run metadata.

## Scheduler

Use cron, systemd timer, queue worker, or hosted scheduler. The scheduler should start a short-lived harness process, not be the harness itself.

For systemd:

```ini
[Timer]
OnCalendar=hourly
Persistent=true

[Install]
WantedBy=timers.target
```

The service should run a deterministic command:

```bash
node scripts/agent-harness.mjs --intent=auto
```

## Agent Runtime

For CLI agents:

```bash
tmux new-session -d -s agent-goal -c /path/to/repo \
  "exec codex -C /path/to/repo --no-alt-screen '<goal prompt>'"
```

Operational commands:

```bash
tmux ls
tmux attach -t agent-goal
tmux capture-pane -pt agent-goal -S -200
tmux kill-session -t agent-goal
```

Use provider adapters so the harness can switch between Codex, Claude Code, or another CLI without changing the lifecycle.

For detailed tmux/session/systemd helper behavior, `/goal` launch forms, and failure handling, use `skills/agent-tmux-harness/SKILL.md`.

## Context Builder

Include:

- run id and intent
- repo root and current git status
- current branch and dirty files
- relevant queue items or candidates
- data boundaries and persistent stores
- runtime status such as containers, services, timers, workers
- local and public health checks
- allowed and forbidden actions
- exact outbox path
- required output schema

Make the context file self-contained enough that a fresh agent session can succeed.

## Harness Lifecycle

Recommended phases:

1. `queued`
2. `context_written`
3. `provider_selected`
4. `goal_injected`
5. `waiting_for_outbox`
6. `outbox_received`
7. `outbox_validated`
8. `ready` or `blocked`
9. `apply_started`
10. `apply_completed`
11. `published`, `deployed`, `dry_run`, or `failed`

Persist every phase with timestamps and metadata.

## Apply Gates

Release publish gates:

- candidate exists in current queue
- source evidence exists
- duplicate check passes
- confidence threshold passes
- API write succeeds
- resulting item is visible in read path

Code deploy gates:

- validation/lint
- tests appropriate to blast radius
- build
- database migration dry run or backup when applicable
- container/service restart
- local health check
- public health check
- worker/timer status
- rollback path known

## Admin Monitor

Expose:

- latest runs with status, provider, intent, ready, risk level
- event stream by run
- active run and deadline
- queue depth and candidate state
- source registry health
- collector/worker health
- last successful apply
- failed phase and error message
- links or paths to context/outbox artifacts when safe

The admin page should explain agent behavior without requiring direct tmux access.
