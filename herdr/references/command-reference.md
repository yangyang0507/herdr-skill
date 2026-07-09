# Herdr Command Reference

Load this file when you need exact CLI syntax, JSON paths, or less common workspace, tab, worktree, or pane operations.

Herdr evolves quickly. If a command here fails with usage output, run `herdr <group> --help` and adapt to the installed binary. The core workflow still prefers discovery, `read`, concrete `wait output`, and structured replies.

## Discovery

```bash
herdr agent list
herdr pane list
herdr workspace list
herdr tab list
```

Most commands print JSON. `pane read` and `agent read` print terminal text.

Current focused pane:

```bash
herdr pane list | sed -nE 's/.*\{[^{}]*"focused":true[^{}]*"pane_id":"([^"]+)".*/\1/p'
```

Current pane from inside a Herdr pane:

```bash
printf '%s\n' "$HERDR_PANE_ID"
herdr pane get "$HERDR_PANE_ID"
```

Prefer `HERDR_PANE_ID` over the focused pane when a command is acting on your own pane. The UI focus can move while your agent keeps running.

## Agent Commands

```bash
herdr agent list
herdr agent get <target>
herdr agent read <target> --source recent-unwrapped --lines 80
herdr agent send <target> <text>
herdr agent rename <target> <name>
herdr agent focus <target>
herdr agent wait <target> --status idle --timeout 30000
herdr agent attach <target> --takeover
herdr agent start <name> --cwd "$PWD" --workspace <workspace-id> --split right --no-focus -- codex
```

Targets accept terminal IDs, unique agent names, detected or reported agent labels, and legacy pane IDs. Prefer unique names or current pane IDs from `agent list`.

`agent wait --status idle` is the completion wait for agent targets: it returns when the target becomes `idle` or `done`. `agent wait --status done` is rejected because `done` is a UI attention state.

`agent send` writes literal text only. It does not submit Enter. For agent-to-agent task handoff, use `scripts/herdr-msg` or resolve the target pane and use `pane run`.

## Pane Commands

```bash
herdr pane list
herdr pane get <pane-id>
herdr pane layout --current
herdr pane neighbor --direction left|right|up|down --current
herdr pane edges --current
herdr pane focus --direction left|right|up|down --current
herdr pane resize --direction left|right|up|down --amount 0.1 --current
herdr pane zoom --current --toggle
herdr pane rename <pane-id> <label>
herdr pane rename <pane-id> --clear
herdr pane read <pane-id> --source visible --lines 40
herdr pane read <pane-id> --source recent --lines 80
herdr pane read <pane-id> --source recent-unwrapped --lines 120
herdr pane read <pane-id> --format ansi --source visible
herdr pane split <pane-id> --direction right --ratio 0.5 --cwd "$PWD" --no-focus
herdr pane split <pane-id> --direction down --ratio 0.4 --cwd "$PWD" --no-focus
herdr pane swap --direction left|right|up|down --current
herdr pane swap --source-pane <pane-id> --target-pane <pane-id>
herdr pane run <pane-id> "npm test"
herdr pane send-text <pane-id> "text without enter"
herdr pane send-keys <pane-id> Enter
herdr pane close <pane-id>
```

Parse a split pane ID:

```bash
NEW_PANE=$(herdr pane split "$SELF" --direction right --cwd "$PWD" --no-focus | sed -nE 's/.*"pane_id":"([^"]+)".*/\1/p')
```

## Waiting

Prefer output waits:

```bash
herdr wait output <pane-id> --match "ready" --timeout 30000
herdr wait output <pane-id> --match "server.*ready" --regex --timeout 30000
```

If you need the exact text that `wait output --source recent` matches, inspect:

```bash
herdr pane read <pane-id> --source recent-unwrapped --lines 80
```

Agent status waits are a fallback:

```bash
herdr agent wait <target> --status idle --timeout 30000
herdr wait agent-status <pane-id> --status done --timeout 30000
```

Use them only when the UI state itself matters or no textual output marker exists.

`wait agent-status` accepts `done`; `agent wait` does not. Prefer `agent wait --status idle` when targeting an agent by name, and prefer structured replies or `wait output` for cross-agent coordination.

## Workspaces And Tabs

```bash
herdr workspace create --cwd /path/to/project --label api --no-focus
herdr workspace focus <workspace-id>
herdr workspace rename <workspace-id> "api server"
herdr workspace close <workspace-id>

herdr tab create --workspace <workspace-id> --cwd "$PWD" --label logs --no-focus
herdr tab focus <tab-id>
herdr tab rename <tab-id> logs
herdr tab close <tab-id>
```

`workspace create` returns `result.workspace`, `result.tab`, and `result.root_pane`. `tab create` returns `result.tab` and `result.root_pane`.

## Worktrees

```bash
herdr worktree list --cwd "$PWD" --json
herdr worktree create --cwd "$PWD" --branch feature/name --base main --label "feature/name" --no-focus --json
herdr worktree open --cwd "$PWD" --branch feature/name --label "feature/name" --no-focus --json
herdr worktree remove --workspace <workspace-id> --json
```

Use Herdr worktrees when you need sibling agents to make isolated code changes without sharing a dirty working tree.

Without `--path`, Herdr creates worktree checkouts under the configured worktree root as `<worktrees.directory>/<repo>/<branch-slug>`. `worktree remove` removes the checkout with `git worktree remove`; it does not delete the branch.

## Sessions, Status, And Attach

```bash
herdr status
herdr status server
herdr status client

herdr session list --json
herdr session attach <name>
herdr session stop <name> --json
herdr session delete <name> --json

herdr terminal attach <terminal-id> --takeover
```

`agent attach <target>` attaches to an agent target. `terminal attach <terminal-id>` attaches to a raw terminal. In direct attach, detach with `ctrl+b q`.

## Notifications And Integrations

```bash
herdr notification show "build failed" --body "api workspace" --position top-right --sound request

herdr integration status
herdr integration status --outdated-only
herdr integration install codex
herdr integration install claude
herdr integration install opencode
herdr integration uninstall codex
```

Supported integration targets include `pi`, `omp`, `claude`, `codex`, `copilot`, `droid`, `kimi`, `opencode`, `hermes`, and `qodercli`.

Integrations can add native session identity, semantic state reports, or both. If agent state or restore looks wrong, check `herdr integration status` and confirm the agent was launched inside Herdr.

## Metadata Reporting

Most agents do not need to call these directly. Use them when a custom integration needs to report visible state to Herdr:

```bash
herdr pane report-agent <pane-id> --source <id> --agent <label> --state working --message "running tests"
herdr pane report-metadata <pane-id> --source <id> --title "api review" --custom-status "waiting for tests"
```

## `scripts/herdr-msg`

Preferred agent-to-agent transport. Resolves the target agent/pane, builds a human header plus machine **sentinels**, and submits with `herdr pane run`.

```bash
scripts/herdr-msg <target> [--kind request|reply|update] [--task id] [--status text] \
  [--msg-id id] [--ack-of MSGID] [--verify] [--wait-reply] [--timeout MS] \
  [--verify-timeout MS] [--legacy-wait-header] [--allow-self-target] \
  [--allow-empty] [--dry-run] [--quiet] [--] [message...]
```

Message body may be argv or stdin. Empty/whitespace-only bodies exit `2` unless `--allow-empty`.

Machine sentinels (first body line; fixed-string match; unique `MSGID`):

```text
<<<herdr-msg:v1:send:TASK:MSGID>>>
<<<herdr-msg:v1:ack:TASK:MSGID>>>
```

Field rules: `TASK` / `MSGID` are `[A-Za-z0-9_-]`, task max 32, msg-id max 16. Invalid `--task` / `--msg-id` / `--ack-of` exit `2` (no silent rewrite). Keeps lines short so terminals do not wrap tokens.

The `[herdr-msg id:MSGID ...]` header is human metadata only. Do not use headers for waits.

| Flag | Meaning |
|------|---------|
| `--kind` | Must be `request`, `reply`, or `update` (default `request`). |
| `--msg-id` | Message id (default auto-generated). Rejected if invalid. |
| `--ack-of` | With `--kind reply`, emit ack for this send `MSGID`. |
| `--verify` | After send, try to **observe** the outbound sentinel on the **target**. Not delivery; unobserved still exit `0` (`state=delivered-unobserved`). |
| `--wait-reply` | After send, block on **SELF** for the **ack sentinel** (token held in memory; not printed in the pre-wait receipt). |
| `--timeout MS` | Timeout for `--wait-reply` (default `60000`). |
| `--verify-timeout MS` | Timeout for `--verify` (default `5000`). |
| `--legacy-wait-header` | **Replaces** sentinel wait with fragile header regex (does not combine). Deprecated. |
| `--allow-self-target` | Allow target pane == SELF. Required for `--wait-reply` or `kind=request` to self; otherwise exit `2` (self-send embeds ack text that can false-match waits). |
| `--allow-empty` | Allow empty/whitespace-only body. |
| `--dry-run` | Resolve and print payload/receipt; do not send. Payload may contain live tokens. |
| `--quiet` | Suppress the key=value receipt. |

Receipt fields (never include the exact waitable `<<<...>>>` string): `ok`, `state`, `task`, `kind`, `msg_id`, `ack_msg_id`, `target`, `target_pane`, `from`, `reply_to`, `outbound_fields`, `send_fields`, `reply_fields`, `match_mode`, `match_recipe`, `observed`, `verified` (alias of observed), `hint`.

Reconstruct: `match = '<<<herdr-msg:' + reply_fields + '>>>'` (e.g. `reply_fields=v1:ack:TASK:MSGID`).

| `state` | Meaning |
|---------|---------|
| `delivered` | `pane run` succeeded; stop observing the target. |
| `delivered-unobserved` | Delivered, but outbound sentinel not seen on target in time (still ok). |
| `waiting-reply` | Delivered; waiting on SELF for ack (token not in this receipt). |
| `replied` | Ack sentinel observed on SELF. |
| `dry-run` | Nothing sent. |
| `wait-timeout` | No ack on SELF in time (exit `4`). |
| `error` | Send/resolve failure (exit `1`). |

Exit codes: `0` ok (including delivered-unobserved), `1` send/resolve fail, `2` usage/env, `4` wait-reply timeout.

**Matching rules**

- Default wait/verify use **short sentinel lines**, not headers.
- Receipts **must not** print the exact wait token; otherwise `herdr wait output` on SELF false-matches the receipt.
- `--wait-reply` matches in-memory `<<<herdr-msg:v1:ack:TASK:MSGID>>>` (fixed).
- `--verify` matches outbound send/ack sentinel on the target.
- If the body starts with any `<<<herdr-msg:v1:...>>>` line, it must equal the expected outbound sentinel exactly (else exit `2`).
- Replies: `scripts/herdr-msg <reply-to> --kind reply --task TASK --ack-of MSGID` prepends the correct ack.
- `kind=request` or `--wait-reply` to the **same pane as SELF** exits `2` unless `--allow-self-target` (prevents matching outbound ack instructions as a reply). Comparison uses **canonical** `pane_id` from `herdr pane get` / agent resolution, not the raw `HERDR_PANE_ID` string (which may be a terminal alias).

**Do not** poll the target after a successful send. Prefer `--wait-reply`.

Examples:

```bash
# Deliver and stop
scripts/herdr-msg reviewer --task api-review <<'MSG'
Review src/api. Reply DONE or BLOCKED.
MSG

# Deliver + optional observation of outbound sentinel
scripts/herdr-msg reviewer --task api-review --verify <<'MSG'
...
MSG

# Deliver and wait on self (preferred; no token echo)
scripts/herdr-msg reviewer --task api-review --wait-reply --timeout 120000 <<'MSG'
...
MSG

# Receiver reply (ack first line auto-prepended)
scripts/herdr-msg "$REPLY_TO" --kind reply --task api-review --ack-of m1a2b3c4 --status done <<'MSG'
DONE: findings...
MSG

# Manual wait from receipt reply_fields (do not echo full token first)
RF=v1:ack:api-review:m1a2b3c4
herdr wait output "$HERDR_PANE_ID" \
  --match "<<<herdr-msg:${RF}>>>" \
  --source recent-unwrapped --timeout 120000
```

## Message Transport Without The Helper

If `scripts/herdr-msg` is unavailable, create a message with header + sentinel and pass it as one argument:

```bash
SELF=${HERDR_PANE_ID:-$(herdr pane list | sed -nE 's/.*\{[^{}]*"focused":true[^{}]*"pane_id":"([^"]+)".*/\1/p')}
TARGET_PANE=$(herdr agent get reviewer | sed -nE 's/.*"pane_id":"([^"]+)".*/\1/p')
MSGID=m$(date +%s)
TASK=review
MSG="[herdr-msg id:$MSGID from:codex pane:$SELF reply-to:$SELF at:current kind:request task:$TASK]
<<<herdr-msg:v1:send:${TASK}:${MSGID}>>>
Please review src/api.
When done, reply with first line exactly:
<<<herdr-msg:v1:ack:${TASK}:${MSGID}>>>
DONE or BLOCKED."
herdr pane run "$TARGET_PANE" "$MSG"
# DELIVERED. Do not poll TARGET_PANE. Wait on SELF:
# herdr wait output "$SELF" --match "<<<herdr-msg:v1:ack:${TASK}:${MSGID}>>>" --timeout 60000
```

Avoid shell one-liners with complex quoting. Use a variable, heredoc, or `scripts/herdr-msg` for multi-line messages.

## Environment Variables

- `HERDR_ENV=1` means the process is inside a Herdr-managed pane.
- `HERDR_PANE_ID` is the current pane handle, usually an internal `p_...` id accepted by pane APIs.
- `HERDR_SOCKET_PATH` selects the local API socket. Use it only as a low-level override.
- `HERDR_SESSION` selects a named session for CLI commands.
- `HERDR_CONFIG_PATH` overrides the config file path.
- `HERDR_LOG` sets Herdr logging filters.
- `HERDR_DISABLE_SOUND` disables sound playback.
