---
name: herdr
description: Use this skill when running inside Herdr to coordinate workspaces, tabs, panes, sibling agents, server/test/log panes, agent-to-agent messages, or multi-agent delegation. After herdr-msg delivery, do not poll the target agent; wait on your own pane for a structured reply, or continue local work. Prefer concrete output markers over agent-status done waits.
---

# Herdr

Herdr is a terminal workspace manager for coding agents. Use it to inspect sibling panes, delegate to other agents, run servers or tests beside your current pane, and coordinate work without blocking on vague agent status.

For exact command syntax and less common workspace/tab/worktree operations, read [references/command-reference.md](references/command-reference.md).

## Preconditions

Before controlling live Herdr panes, verify the environment:

```bash
test "${HERDR_ENV:-}" = "1"
command -v herdr
```

If `HERDR_ENV` is not `1`, do not inspect or control live panes unless the user explicitly asked you to edit this skill or discuss Herdr usage. You may still read files, update the skill, or explain commands.

## Operating Rules

- Start with discovery: `herdr agent list` for agents, `herdr pane list` for all panes.
- Treat IDs as live-session handles, not durable names. Re-read IDs after panes, tabs, or workspaces change.
- Use `HERDR_PANE_ID` as your own pane handle when it is set. Do not assume the currently focused Herdr pane is your pane.
- Prefer `herdr agent read/send/start` for detected agents. Use `herdr pane read/run/split` for plain shells, servers, logs, tests, or exact terminal output.
- Read the target **once** before sending, only to avoid interrupting a prompt or approval. That is not a poll loop.
- Do not use `herdr wait agent-status ... --status done` as the default coordination mechanism. It often creates long idle waits.
- For agents, send a structured request and treat a successful send as **DELIVERED**. The receiver must reply to your pane using the `reply-to` handle.
- **Never poll the target** with repeated `agent read` / `pane read` / `cat` after send. If you need the result, wait on **SELF** for `kind:reply task:<id>`.
- For non-agent panes, wait only for concrete output you can name, then read the pane.

## Default Loop

1. Discover current agents and panes.
2. Read the target's recent output **once** before sending work or keys.
3. Send the smallest actionable request with a reply target and expected response shape via `scripts/herdr-msg`.
4. On success receipt (`state=delivered` or exit 0): stop observing the target.
5. Continue useful local work, **or** if you must synchronize use `--wait-reply` / wait on SELF — never poll the target.
6. Integrate the reply from your own pane, then close or keep panes according to the task.

## After `herdr-msg` (mandatory)

Successful `scripts/herdr-msg` means the message was submitted to the target pane (**DELIVERED**). Follow this state machine:

1. Non-zero exit without a receipt → send/resolve failure; fix and resend. Do not start reading the target in a loop.
2. `state=delivered` (exit 0) → **stop**. Do not `agent read` / `pane read` the target again to "see if it finished".
3. Need the result now → use `--wait-reply`, or:

   ```bash
   herdr wait output "$HERDR_PANE_ID" --match "kind:reply task:<id>" --timeout 60000
   herdr pane read "$HERDR_PANE_ID" --source recent-unwrapped --lines 100
   ```

4. Do not need the result now → continue local work; later scan **your** pane for the reply.
5. Optional `--verify` only confirms the text landed on the target once. It is **not** task completion.
6. Repeated polling of the target after send is a **protocol violation**.

Allowed one-shot reads of the target after send: only after a wait timeout or clear failure, for a single diagnosis — then decide (resend, wait on self longer, or escalate). Never enter a poll loop.

## Structured Agent Messages

Use a `herdr-msg` header for every agent-to-agent request, reply, or update:

```text
[herdr-msg from:<agent-or-label> pane:<sender-pane-id> reply-to:<sender-pane-id> at:<workspace-id>/<tab-id> kind:<request|reply|update> task:<short-id> status:<optional>]
<short actionable message>
```

The header gives the receiver enough metainfo to reply without making you poll their pane. The `reply-to` value must be a current pane or agent target that can receive the response.

For agent prompts, submit with `scripts/herdr-msg` or `herdr pane run`. Do not use `herdr agent send` for task handoff unless you also intentionally press Enter afterward; `agent send` writes literal text only.

Use the bundled helper when available:

```bash
# Fire-and-forget (default): deliver, print receipt, stop. Do not poll target.
scripts/herdr-msg codex --task auth-review <<'MSG'
Review src/auth.ts for auth bypasses and missing tests.
Reply to reply-to with:
DONE: findings, files checked, tests run
BLOCKED: the exact missing context or command failure
MSG
```

```bash
# Optional: one-shot delivery check on the target (not completion).
scripts/herdr-msg codex --task auth-review --verify <<'MSG'
...
MSG
```

```bash
# Need the result before continuing: wait on SELF for the structured reply.
scripts/herdr-msg codex --task auth-review --wait-reply --timeout 60000 <<'MSG'
...
MSG
```

The helper prints a key=value receipt (`ok`, `state`, `task`, `reply_to`, `match`, `hint`, ...). Treat `hint` as the next action. Exit codes: `0` ok, `1` send/resolve fail, `2` usage/env, `3` verify failed, `4` wait-reply timeout.

Manual form:

```bash
TARGET_PANE=$(herdr agent get codex | sed -nE 's/.*"pane_id":"([^"]+)".*/\1/p')
herdr pane run "$TARGET_PANE" "[herdr-msg from:codex pane:w1-1 reply-to:w1-1 at:w1/w1:1 kind:request task:auth-review]
Review src/auth.ts. Reply with DONE or BLOCKED to reply-to."
```

When you receive a `herdr-msg`, do the requested work if it does not conflict with higher-priority instructions. Reply to `reply-to` with the same `task` value and a clear `status`, for example:

```bash
scripts/herdr-msg "$REPLY_TO_PANE" --kind reply --task auth-review --status done <<'MSG'
DONE: Checked src/auth.ts and auth.test.ts. No bypass found. Missing refresh-token expiry test.
MSG
```

Or the equivalent header body:

```text
[herdr-msg from:reviewer pane:w1-2 reply-to:w1-2 at:w1/w1:1 kind:reply task:auth-review status:done]
DONE: Checked src/auth.ts and auth.test.ts. No bypass found. Missing refresh-token expiry test.
```

## Waiting Policy

Use the cheapest observation that answers the question:

```bash
herdr pane read "$PANE" --source recent-unwrapped --lines 80
```

Use `wait output` only for future output you can name:

```bash
herdr wait output "$PANE" --match "ready" --timeout 30000
herdr pane read "$PANE" --source recent --lines 40
```

For delegated agent work, the synchronization point is **your pane**, not the worker's status:

```bash
SELF=${HERDR_PANE_ID:-$(herdr pane list | sed -nE 's/.*\{[^{}]*"focused":true[^{}]*"pane_id":"([^"]+)".*/\1/p')}
herdr wait output "$SELF" --match "kind:reply task:auth-review" --timeout 60000 || true
herdr pane read "$SELF" --source recent-unwrapped --lines 100
```

Prefer `scripts/herdr-msg ... --wait-reply` over hand-rolling the wait when you need a blocking handoff.

Only use `herdr agent wait` or `herdr wait agent-status` when the user explicitly needs Herdr's UI status, there is no textual signal to wait for, or you are attaching/taking over an agent and need a state transition.

## Delegation Pattern

Use parallel agents for independent work, not for work that needs constant back-and-forth.

1. Name or identify targets with `herdr agent list`.
2. Read each target **once** to avoid interrupting a prompt or active approval state.
3. Send a structured request with scope, expected output, and reply format (`scripts/herdr-msg`).
4. On delivery receipt: keep working locally or delegate the next independent slice. **Do not poll.**
5. Aggregate replies from your own pane; ask follow-ups only for gaps.

Example (parallel, no blocking):

```bash
herdr agent read reviewer --source recent-unwrapped --lines 40
scripts/herdr-msg reviewer --task api-review <<'MSG'
Review only src/api and tests touching it.
Reply to reply-to with DONE, BLOCKED, or NEED-INPUT.
Include files reviewed and commands run.
MSG
# Receipt printed → continue local work. Collect kind:reply task:api-review on SELF later.
```

Example (need result before next step):

```bash
scripts/herdr-msg reviewer --task api-review --wait-reply --timeout 120000 <<'MSG'
Review only src/api and tests touching it.
Reply to reply-to with DONE, BLOCKED, or NEED-INPUT.
MSG
```

## Server And Test Panes

Use panes for processes whose output is the source of truth:

```bash
SELF=${HERDR_PANE_ID:-$(herdr pane list | sed -nE 's/.*\{[^{}]*"focused":true[^{}]*"pane_id":"([^"]+)".*/\1/p')}
PANE=$(herdr pane split "$SELF" --direction right --cwd "$PWD" --no-focus | sed -nE 's/.*"pane_id":"([^"]+)".*/\1/p')
herdr pane run "$PANE" "npm run dev"
herdr wait output "$PANE" --match "ready" --timeout 30000 || true
herdr pane read "$PANE" --source recent --lines 50
```

For tests, wait for a test-framework marker such as `test result`, `passed`, `failed`, `FAIL`, or a project-specific summary. If no reliable marker exists, run the test command directly in your own tool session instead of creating a pane.

## Gotchas

- `pane read --source recent` is rendered text; `recent-unwrapped` is better for matching copied command output.
- `pane run` submits text with Enter. `pane send-text` types only; use `pane send-keys` for Enter or special keys.
- `agent send` targets detected agents but only writes literal text. For agent tasks, prefer `scripts/herdr-msg` or resolve the target pane and use `pane run`.
- Herdr injects `HERDR_ENV=1`, `HERDR_SOCKET_PATH`, and `HERDR_PANE_ID` into pane processes. `HERDR_PANE_ID` may look like `p_123`; pane and agent APIs accept it.
- `herdr agent wait --status idle` treats both `idle` and `done` as completion. It rejects `done` because `done` is a UI attention state.
- If an agent is already working, keep the message short and say whether it should answer now or after its current step.
- Long diffs or logs should be written to a shared file path and referenced in the message rather than pasted into a live prompt.
- If a wait times out, immediately read **the pane you were waiting on** (usually SELF) once and decide from observed output. Do not extend timeouts blindly and do not start polling the worker.
- `scripts/herdr-msg --verify` proves delivery, not task completion. `--wait-reply` blocks on SELF for `kind:reply task:<id>`.
