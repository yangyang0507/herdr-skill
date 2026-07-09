---
name: herdr
description: Use this skill when running inside Herdr to coordinate workspaces, tabs, panes, sibling agents, server/test/log panes, agent-to-agent messages, or multi-agent delegation. After herdr-msg delivery, do not poll the target; use --wait-reply or reconstruct the ack token from receipt reply_fields (never wait on raw tokens printed into SELF). Prefer sentinels over header matching and over agent-status done waits.
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
- Prefer `herdr agent read/start` for detected agents. For agent task handoff, use `scripts/herdr-msg` or `herdr pane run` (not `herdr agent send`, which types text without submitting Enter). Use `herdr pane read/run/split` for plain shells, servers, logs, tests, or exact terminal output.
- Read the target **once** before sending, only to avoid interrupting a prompt or approval. That is not a poll loop.
- Do not use `herdr wait agent-status ... --status done` as the default coordination mechanism. It often creates long idle waits.
- For agents, send via `scripts/herdr-msg` and treat `pane run` success as **DELIVERED**. The receiver replies to `reply-to` with the correct **ack sentinel** as the first body line (`--ack-of`).
- **Never poll the target** with repeated `agent read` / `pane read` / `cat` after send. If you need the result, use `--wait-reply` (preferred) or wait on **SELF** for the ack token reconstructed from receipt `reply_fields` — do not paste the raw token into SELF before waiting.
- For non-agent panes, wait only for concrete output you can name, then read the pane.

## Default Loop

1. Discover current agents and panes.
2. Read the target's recent output **once** before sending work or keys.
3. Send the smallest actionable request via `scripts/herdr-msg` (unique `--task` preferred).
4. On receipt `state=delivered` / `delivered-unobserved` (exit 0): **stop** observing the target.
5. Continue local work, **or** use `--wait-reply` / wait on SELF for the ack token from `reply_fields` — never poll the target.
6. Integrate the reply from your own pane.

## After `herdr-msg` (mandatory)

Successful `scripts/herdr-msg` means Herdr accepted/submitted the input (**DELIVERED**). Follow this state machine:

1. Exit `1` / `state=error` → send/resolve failure; fix and resend. Do not poll.
2. `state=delivered` or `delivered-unobserved` (exit 0) → **stop** observing the target.
3. Need the result now → prefer `--wait-reply` (helper holds the token in memory and does **not** print it into SELF before waiting).

   Manual wait (only if needed): reconstruct from receipt fields without echoing the full token into the pane first:

   ```bash
   # receipt has: reply_fields=v1:ack:auth-review:m1a2b3c4
   # build match off-stream / in the wait command only:
   RF=v1:ack:auth-review:m1a2b3c4
   herdr wait output "$HERDR_PANE_ID" \
     --match "<<<herdr-msg:${RF}>>>" \
     --source recent-unwrapped --timeout 60000
   herdr pane read "$HERDR_PANE_ID" --source recent-unwrapped --lines 100
   ```

4. Do not need the result now → continue local work; later scan **your** pane for the ack sentinel / DONE body.
5. Optional `--verify` only means **screen-observed** the outbound sentinel on the target. Failure becomes `state=delivered-unobserved` (still exit 0), not task failure. Do not resend solely because of unobserved.
6. Repeated polling of the target after send is a **protocol violation**.

**Do not** wait on `[herdr-msg ...]` headers for synchronization. Terminals wrap them and TUIs decorate them; they are human metadata only.

**Do not** print the exact waitable token (`<<<herdr-msg:v1:ack:...>>>`) into SELF (receipts, debug echoes) before `herdr wait output` — Herdr matches the recent buffer and will false-hit.

Allowed one-shot reads of the target after send: only after a wait timeout or clear failure, for a single diagnosis — then decide. Never enter a poll loop.

## Structured Agent Messages

Every agent-to-agent message has:

1. A human header (metadata only).
2. A short machine **sentinel** as the first body line.
3. The actionable body.

```text
[herdr-msg id:<msgid> from:<agent> pane:<sender-pane> reply-to:<sender-pane> at:<ws>/<tab> kind:<request|reply|update> task:<task> status:<optional>]
<<<herdr-msg:v1:send:TASK:MSGID>>>
<short actionable message>

When done, reply ... first body line:
<<<herdr-msg:v1:ack:TASK:MSGID>>>
```

Sentinel forms:

| Direction | Sentinel |
|-----------|----------|
| outbound request/update | `<<<herdr-msg:v1:send:TASK:MSGID>>>` |
| reply ack | `<<<herdr-msg:v1:ack:TASK:MSGID>>>` |

`TASK` and `MSGID` are sentinel fields: `[A-Za-z0-9_-]`, task max 32, msg-id max 16. Invalid `--msg-id` / `--ack-of` / `--task` are rejected (not rewritten).

Receipts print **fields** only (`reply_fields=v1:ack:TASK:MSGID`), never the exact waitable `<<<...>>>` string, so SELF waits are not poisoned by the helper's own output.

For agent prompts, submit with `scripts/herdr-msg` or `herdr pane run`. Do not use `herdr agent send` for task handoff unless you also intentionally press Enter.

### Canonical helper flows

```bash
# Fire-and-forget: deliver, print receipt (reply_fields, no raw token), stop. Do not poll.
scripts/herdr-msg codex --task auth-review <<'MSG'
Review src/auth.ts for auth bypasses and missing tests.
Reply with DONE or BLOCKED findings.
MSG
```

```bash
# Optional: observe send sentinel on target (not completion; unobserved still ok).
scripts/herdr-msg codex --task auth-review --verify <<'MSG'
...
MSG
```

```bash
# Need the result: wait on SELF for the in-memory/generated ack sentinel.
scripts/herdr-msg codex --task auth-review --wait-reply --timeout 120000 <<'MSG'
...
MSG
```

Receipt fields include: `ok`, `state`, `task`, `msg_id`, `outbound_fields`, `reply_fields`, `observed`, `hint`. Reconstruct wait tokens from `reply_fields` if needed; prefer `--wait-reply` (raw `<<<...>>>` is never printed in the receipt).

Exit codes: `0` ok (including delivered-unobserved), `1` send/resolve fail, `2` usage/env, `4` wait-reply timeout.

### When you receive a request

Do the work if it does not conflict with higher-priority instructions. Reply to `reply-to` with the **same task** and the **exact ack first line** from the request (or use `--ack-of`):

```bash
scripts/herdr-msg "$REPLY_TO_PANE" --kind reply --task auth-review --ack-of m1a2b3c4 --status done <<'MSG'
DONE: Checked src/auth.ts. No bypass found. Missing refresh-token expiry test.
MSG
```

The helper prepends `<<<herdr-msg:v1:ack:auth-review:m1a2b3c4>>>` as the first body line when missing.

### Manual fallback (no helper)

```bash
SELF=$HERDR_PANE_ID
TARGET_PANE=$(herdr agent get codex | sed -nE 's/.*"pane_id":"([^"]+)".*/\1/p')
MSGID=m$(date +%s)
TASK=auth-review
herdr pane run "$TARGET_PANE" "[herdr-msg id:$MSGID from:agent pane:$SELF reply-to:$SELF at:local kind:request task:$TASK]
<<<herdr-msg:v1:send:${TASK}:${MSGID}>>>
Review src/auth.ts.
When done, reply with first line exactly:
<<<herdr-msg:v1:ack:${TASK}:${MSGID}>>>
DONE or BLOCKED findings."
# Wait on self:
# herdr wait output "$SELF" --match "<<<herdr-msg:v1:ack:${TASK}:${MSGID}>>>" --timeout 120000
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

For delegated agent work, wait on **your pane** for the **ack sentinel** (fixed string), not agent status and not header regex. Prefer the helper so the token is never printed into SELF before the wait:

```bash
scripts/herdr-msg reviewer --task auth-review --wait-reply --timeout 60000 <<'MSG'
...
MSG
```

Manual equivalent (reconstruct from receipt `reply_fields=v1:ack:auth-review:m1a2b3c4` without pre-echoing the token):

```bash
SELF=$HERDR_PANE_ID
RF=v1:ack:auth-review:m1a2b3c4
herdr wait output "$SELF" --match "<<<herdr-msg:${RF}>>>" --source recent-unwrapped --timeout 60000 || true
herdr pane read "$SELF" --source recent-unwrapped --lines 100
```

Only use `herdr agent wait` or `herdr wait agent-status` when the user explicitly needs Herdr's UI status, there is no textual signal to wait for, or you are attaching/taking over an agent and need a state transition.

## Delegation Pattern

Use parallel agents for independent work, not for work that needs constant back-and-forth.

1. Name or identify targets with `herdr agent list`.
2. Read each target **once** to avoid interrupting a prompt or active approval state.
3. Send a structured request with scope, expected output, and reply format (`scripts/herdr-msg`).
4. On delivery receipt: keep working locally or delegate the next independent slice. **Do not poll.**
5. Aggregate replies from your own pane (ack sentinels / DONE bodies); ask follow-ups only for gaps.

Example (parallel, no blocking):

```bash
herdr agent read reviewer --source recent-unwrapped --lines 40
scripts/herdr-msg reviewer --task api-review <<'MSG'
Review only src/api and tests touching it.
Reply with DONE, BLOCKED, or NEED-INPUT.
Include files reviewed and commands run.
MSG
# Receipt has reply_fields → continue local work; collect ack on SELF later.
```

Example (need result before next step):

```bash
scripts/herdr-msg reviewer --task api-review --wait-reply --timeout 120000 <<'MSG'
Review only src/api and tests touching it.
Reply with DONE, BLOCKED, or NEED-INPUT.
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
- Machine sync = **sentinels + msg_id**. Header matching is fragile/deprecated (`--legacy-wait-header` **replaces** sentinel wait, it does not add to it).
- Receipts intentionally omit raw `<<<herdr-msg:...>>>` tokens; use `reply_fields` / `--wait-reply`.
- `--verify` is optional observation of the outbound sentinel, not delivery and not task completion.
- Empty message bodies are rejected unless `--allow-empty` is set. Body first line must be the exact expected sentinel if it looks like a sentinel at all.
- Do not `kind=request` or `--wait-reply` to your own pane unless `--allow-self-target` (outbound ack instructions would false-match).
