---
name: herdr
description: Use this skill when running inside Herdr to coordinate workspaces, tabs, panes, sibling agents, server/test/log panes, agent-to-agent messages, or multi-agent delegation. Send a lightweight herdr-msg (reply-to + task), then stop and end the turn. Do not poll siblings and do not block-wait for replies; handle the peer message when it is injected into your pane.
---

# Herdr

Herdr is a terminal workspace manager for coding agents. Use it to inspect sibling panes, delegate to other agents, run servers or tests beside your current pane, and coordinate work without blocking.

For exact CLI syntax and less common workspace/tab/worktree operations, read [references/command-reference.md](references/command-reference.md).

## Preconditions

```bash
test "${HERDR_ENV:-}" = "1"
command -v herdr
```

If `HERDR_ENV` is not `1`, do not control live panes unless the user asked you to edit this skill or discuss Herdr. You may still read files and explain commands.

## Operating Rules

- Discover first: `herdr agent list`, `herdr pane list`.
- Treat IDs as live-session handles. Re-read them after layout changes.
- Prefer `HERDR_PANE_ID` as your pane handle when set.
- For agent task handoff use `scripts/herdr-msg` or `herdr pane run` (not `herdr agent send` alone — it does not press Enter).
- Read a target **once** before sending, only to avoid interrupting a prompt/approval.
- **After send: stop.** Do not poll the target. Do not `herdr wait` / `--wait-reply` for normal chat. End your turn. When a reply is injected into **your** pane, handle that inbound message.
- Do not use `herdr wait agent-status ... --status done` for multi-agent coordination.

## Default Loop

1. Discover agents/panes.
2. Read the target **once** if needed.
3. Send a short request with `scripts/herdr-msg` (header carries **reply-to** + **task**).
4. On `state=delivered`: **end the turn** (or do unrelated local work that does not depend on the reply).
5. When an inbound reply appears in your pane, integrate it.

## After `herdr-msg` (mandatory)

1. Non-zero exit → send/resolve failed; fix and resend. Do not poll.
2. `state=delivered` (exit 0) → **stop**. Do not read the target in a loop. Do not block-wait.
3. End your turn. The wake-up is the peer injecting a message into `reply-to` (your pane).
4. Polling the target after send is a **protocol violation**.

## Lightweight agent messages

Keep it human and thin. One header line + body:

```text
[herdr-msg reply-to:<your-pane-id> task:<short-label>]
<short actionable message>
```

Optional fields the helper may add: `from:<name>`, `kind:reply` / `kind:update` (requests omit kind).

| Field | Role |
|-------|------|
| `reply-to` | **Required.** Where the peer should answer. |
| `task` | Human thread label (not a crypto id). |
| `from` | Optional sender label. |
| `kind` | Optional; default is request. |

**Not required for normal chat:** `msg_id`, sentinels, `ack-of`, status codes in the header, waiting recipes.

### Send (default)

```bash
scripts/herdr-msg codex --task auth-review <<'MSG'
Review src/auth.ts for auth bypasses and missing tests.
Reply to reply-to with DONE or BLOCKED findings.
MSG
# receipt: state=delivered → end turn
```

### Reply

```bash
scripts/herdr-msg "$REPLY_TO" --task auth-review --kind reply <<'MSG'
DONE: Checked src/auth.ts. No bypass found. Missing refresh-token expiry test.
MSG
```

### Manual fallback

```bash
SELF=$HERDR_PANE_ID
TARGET=$(herdr agent get codex | sed -nE 's/.*"pane_id":"([^"]+)".*/\1/p')
herdr pane run "$TARGET" "[herdr-msg reply-to:$SELF task:auth-review]
Review src/auth.ts. Reply to reply-to with DONE or BLOCKED."
```

### Receipt

Tiny key=value lines: `ok`, `state`, `target`, `target_pane`, `reply_to`, `task`, `kind`, `hint`.

Treat `state=delivered` as success. The `hint` says to end the turn.

## Waiting Policy

**Agent chat:** do not wait. Send, end turn, handle inbound reply later.

Use `herdr wait output` only for **non-agent** processes (dev servers, tests) where you can name a concrete marker such as `ready` or `FAIL`:

```bash
herdr wait output "$PANE" --match "ready" --timeout 30000
herdr pane read "$PANE" --source recent --lines 40
```

Only use `herdr agent wait` / `wait agent-status` when the user explicitly needs UI status or you are attaching/taking over an agent.

## Delegation Pattern

1. `herdr agent list`
2. Read target once.
3. `scripts/herdr-msg <target> --task <label> <<'MSG' ... MSG`
4. On delivered: **end turn** (or unrelated local work). No poll, no block-wait.
5. When replies arrive on your pane, integrate; follow up only for gaps.

## Server And Test Panes

```bash
SELF=${HERDR_PANE_ID:-$(herdr pane list | sed -nE 's/.*\{[^{}]*"focused":true[^{}]*"pane_id":"([^"]+)".*/\1/p')}
PANE=$(herdr pane split "$SELF" --direction right --cwd "$PWD" --no-focus | sed -nE 's/.*"pane_id":"([^"]+)".*/\1/p')
herdr pane run "$PANE" "npm run dev"
herdr wait output "$PANE" --match "ready" --timeout 30000 || true
herdr pane read "$PANE" --source recent --lines 50
```

For tests, wait for framework markers (`passed`, `failed`, `FAIL`, …) or run tests in your own session.

## Gotchas

- `pane run` submits with Enter; `agent send` does not.
- `recent-unwrapped` is better than `recent` when matching command output.
- Herdr sets `HERDR_ENV=1`, `HERDR_PANE_ID`, `HERDR_SOCKET_PATH` in panes.
- Long diffs/logs: write a file path into the message; do not paste huge blobs into the peer prompt.
- Request-to-self is refused by default (`--allow-self` to override).
- Keep headers one line; put status and findings in the **body** (`DONE: …` / `BLOCKED: …`).
