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

## Message Transport Without The Helper

If `scripts/herdr-msg` is unavailable, create a message manually and pass it as one argument:

```bash
SELF=${HERDR_PANE_ID:-$(herdr pane list | sed -nE 's/.*\{[^{}]*"focused":true[^{}]*"pane_id":"([^"]+)".*/\1/p')}
MSG="[herdr-msg from:codex pane:$SELF reply-to:$SELF at:current kind:request task:review]
Please review src/api. Reply to reply-to with DONE or BLOCKED."
herdr agent send reviewer "$MSG"
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
