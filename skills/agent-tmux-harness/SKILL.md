---
name: agent-tmux-harness
description: Run, restart, implement, or diagnose tmux-backed interactive AI CLI harnesses. Use when Codex is asked about tmux sessions for Claude Code or Codex CLI, /goal prompt injection, systemd helper services, outbox completion, TTY behavior, or why an agent did or did not finish.
---

# Agent Tmux Harness

Use this skill when the agent runtime is an interactive CLI hosted in `tmux`.
The goal is to keep terminal behavior operationally useful while making the
deterministic harness, not terminal text, own completion and side effects.

## Mental Model

- `tmux` is the durable TTY host for a real interactive CLI.
- Codex CLI / Claude Code are the agent plane.
- The harness creates context, starts or prompts the CLI, waits for outbox,
  validates the result, records events, and applies approved actions.
- The outbox JSON is the completion signal. Terminal output is only diagnostic.

Do not infer success from a tmux pane, a CLI process exit, or a natural-language
message. Success requires a valid run-specific outbox plus passing harness gates.

## Standard Names

Use repo-local names where available. A common convention:

```text
Codex tmux session:        agent-goal
Claude Code tmux session:  agent-goal-claude
Codex systemd helper:      agent-goal-session.service
Claude systemd helper:     agent-goal-claude-session.service
Harness runtime dir:       var/agent-harness/
Context dir:               var/agent-harness/context/
Outbox dir:                var/agent-harness/outbox/
State dir:                 var/agent-harness/state/
```

For a concrete repo, read its deploy units, runbook, env vars, and harness
script before assuming names.

ReleaseLog's current concrete names are:

```text
Claude tmux session:       releaselog-ai-goal-claude
Codex tmux session:        releaselog-ai-goal
Claude systemd helper:     releaselog-ai-goal-claude-session.service
Codex systemd helper:      releaselog-ai-goal-session.service
Hourly harness timer:      releaselog-ai-review.timer
Fast harness timer:        releaselog-ai-review-fast.timer
Harness runtime dir:       /root/releaselog/var/ai-harness/
```

## Inspect Current State

Check sessions:

```bash
tmux ls
tmux has-session -t agent-goal-claude
tmux capture-pane -pt agent-goal-claude -S -200
```

Check helper services:

```bash
systemctl status agent-goal-claude-session.service --no-pager -l
systemctl status agent-goal-session.service --no-pager -l
```

Check scheduled harness runs:

```bash
systemctl list-timers --all '*ai*' --no-pager
journalctl -u agent-harness.service -n 100 --no-pager
```

Check active run and outbox:

```bash
jq . var/agent-harness/state/active-run.json
ls -lah var/agent-harness/outbox/
```

## Starting Persistent Sessions

For a persistent idle Codex session:

```bash
tmux has-session -t agent-goal 2>/dev/null || \
  tmux new-session -d -s agent-goal -c /path/to/repo \
    'exec codex -C /path/to/repo --dangerously-bypass-approvals-and-sandbox --no-alt-screen'
```

For a persistent idle Claude Code session:

```bash
tmux has-session -t agent-goal-claude 2>/dev/null || \
  tmux new-session -d -s agent-goal-claude -c /path/to/repo \
    'exec claude --permission-mode dontAsk --allowedTools Read,Glob,Grep,LS,Bash,Write,Edit,MultiEdit --add-dir /path/to/repo'
```

If systemd helpers are `Type=oneshot` with `RemainAfterExit=yes`, a manual
`tmux kill-session` can leave systemd showing `active (exited)`. Use
`systemctl restart`, not just `start`, to recreate the session:

```bash
systemctl restart agent-goal-claude-session.service
```

Always verify the tmux session exists after systemd says success.

## Injecting A Run-Specific Goal

For autonomous runs, prefer a clean session per run:

1. Build `context/<runId>.md`.
2. Choose provider.
3. Kill the provider's old tmux session.
4. Start a new tmux session with the run-specific `/goal` prompt.
5. Write `state/active-run.json` with run id, provider, session, outbox path,
   injected time, and deadline.
6. Wait for `outbox/<runId>.json`.

Codex prompt form:

```bash
tmux new-session -d -s agent-goal -c /path/to/repo \
  'exec codex -C /path/to/repo --dangerously-bypass-approvals-and-sandbox --no-alt-screen "Goal marker /goal: read /path/to/context.md and write exactly one JSON object to /path/to/outbox.json"'
```

Claude Code prompt form:

```bash
tmux new-session -d -s agent-goal-claude -c /path/to/repo \
  'exec claude "Goal marker /goal: read /path/to/context.md and write exactly one JSON object to /path/to/outbox.json" --permission-mode dontAsk --allowedTools Read,Glob,Grep,LS,Bash,Write,Edit,MultiEdit --add-dir /path/to/repo'
```

Claude Code is more sensitive to argument order. Put the prompt before the
permission and directory flags when launching a one-shot goal session.

## Completion Contract

Wait for outbox by filesystem polling, not by terminal scraping:

```js
while (Date.now() < deadline) {
  if (existsSync(outboxPath)) return readJson(outboxPath);
  await sleep(5000);
}
throw new Error(`outbox_timeout:${outboxPath}`);
```

Validate at minimum:

- JSON parses as exactly one object.
- `runId` equals active run id.
- `intent` equals active intent.
- `ready` is boolean.
- `riskLevel` is one of `low`, `medium`, `high`; block `high`.
- `environmentUnderstanding.dev`, `.runtime`, and `.data` are present.
- Decisions reference only current queue items.
- Proposed side effects are applied by the harness, not the CLI.

## What Tmux Is For

Use tmux for:

- preserving a real TTY and CLI login state
- observing what the agent is doing
- supporting terminal-first CLIs that expect interactive behavior
- giving operators a way to attach or capture diagnostics

Do not use tmux for:

- completion detection
- parsing machine-readable decisions from pane text
- granting production authority
- storing audit history

Persist audit facts in database runs/events or durable local run files.

## Common Failures

Session missing but service is active:

- Cause: oneshot helper stayed `active (exited)` after the tmux session was
  killed manually.
- Fix: `systemctl restart <helper>.service`, then `tmux has-session`.

Duplicate session:

- Cause: the harness tried to create a new session without killing or detecting
  the old one.
- Fix: kill the provider session before run-specific launch, or make creation
  idempotent with `tmux has-session`.

Outbox timeout:

- Cause: agent is stuck, lacks tools, did not understand protocol, or cannot
  write the target path.
- Inspect: `active-run.json`, `tmux capture-pane`, context path, and outbox dir.
- Fix the provider permissions or prompt; do not fake the outbox.

Invalid outbox:

- Cause: markdown fences, multiple JSON objects, stale run id, missing fields,
  or unknown candidates.
- Inspect with `jq . outbox/<runId>.json`.
- Mark the run failed or blocked; do not partially apply side effects.

Claude cannot write:

- Ensure `--allowedTools` includes `Write` and relevant read/search/shell tools.
- Ensure `--add-dir` covers the repo or writable harness directory.

Codex cannot write:

- Ensure the invocation uses the intended sandbox/approval mode for the host
  harness. If bypass mode is used, keep publish/deploy authority in the harness.

## Operator Checklist

Before saying the tmux harness is working:

- helper service can recreate the tmux session
- provider CLI is logged in and accepts a goal prompt
- context file includes exact outbox path and required protocol
- outbox directory is writable by the provider process
- harness validates stale run id, invalid JSON, high risk, and ready false
- events show `goal_injected`, `waiting_for_outbox`, `outbox_received`, and
  `outbox_validated` or a clear failure phase
- publish/deploy side effects are performed only by harness gates
