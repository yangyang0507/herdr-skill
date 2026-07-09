# Herdr Skill

English | [中文](README.zh-CN.md)

This repository builds an improved Agent Skill for [Herdr](https://github.com/ogulcancelik/herdr), focused on making multi-agent coordination inside Herdr more practical and less wasteful.

It is not the Herdr application itself. The deliverable is the installable skill directory at [`herdr/`](herdr/), whose main entry point is [`herdr/SKILL.md`](herdr/SKILL.md).

## Purpose

The original Herdr skill documents the CLI surface, but its coordination recipes can lead agents toward long, low-signal waits such as waiting for another pane's agent status to become `done`. In multi-agent workflows this creates unnecessary blocking, especially when the useful signal is already visible in pane output or should be sent back as an explicit reply.

This repo refines that behavior around a few defaults:

- prefer `herdr agent list` and `herdr pane list` for discovery;
- read concrete pane output **once** before acting (not a poll loop);
- wait for named output markers, not vague completion state;
- use structured agent messages with `reply-to` metadata;
- treat a successful `herdr-msg` send as **DELIVERED**, then stop observing the target;
- if synchronization is needed, use `--wait-reply` (or reconstruct the ack token from receipt `reply_fields` without printing the raw token into SELF first);
- continue local work instead of polling sibling agents;
- keep detailed command reference behind progressive disclosure.

## What Is Included

```text
herdr/
  SKILL.md                         core agent-facing workflow
  scripts/herdr-msg                dependency-free structured message helper
  references/command-reference.md  detailed Herdr CLI reference
  agents/openai.yaml               UI metadata for skill clients
```

The helper script is intentionally plain Bash. It does not require Python or `jq`, so it can run in minimal agent shells.

## References And Influences

This skill is based on these upstream projects and practices:

- [Agent Skills specification](https://github.com/agentskills/agentskills): directory layout, `SKILL.md` frontmatter, progressive disclosure, and validation expectations.
- [Herdr](https://github.com/ogulcancelik/herdr): the CLI, pane/agent/workspace concepts, environment variables such as `HERDR_ENV`, `HERDR_SOCKET_PATH`, and `HERDR_PANE_ID`, and the original Herdr skill.
- [smux](https://github.com/yangyang0507/smux): the cross-agent messaging pattern where messages include sender and reply metadata, and agents reply back instead of forcing the sender to poll.

The main behavioral change is adapting the smux-style message discipline to native Herdr commands. Herdr does not need tmux-bridge; this skill uses Herdr's own `agent`, `pane`, and `wait` commands.

## Install

Preferred install path, using the [`skills`](https://skills.sh/) CLI:

```bash
npx skills add yangyang0507/herdr-skill
```

This installs the skill from this GitHub repository through the Vercel `skills.sh` flow.

Manual fallback: copy the `herdr/` directory into a skill directory supported by your agent client. For example, for clients that use an `.agents/skills` layout:

```bash
mkdir -p .agents/skills
cp -R herdr .agents/skills/herdr
```

The skill should be used from inside a Herdr-managed pane where `HERDR_ENV=1` is available.

## Validate

Basic validation:

```bash
bash -n herdr/scripts/herdr-msg
```

If you have the Agent Skills validation helper available:

```bash
uv run --with pyyaml python /path/to/quick_validate.py herdr
```

Runtime behavior that touches live panes should only be tested from inside Herdr.

## Design Notes

`herdr/scripts/herdr-msg` sends messages with a human header plus machine **sentinels**:

```text
[herdr-msg id:m1a2b3c4 from:codex pane:p_42 reply-to:p_42 at:w1/w1:1 kind:request task:review]
<<<herdr-msg:v1:send:review:m1a2b3c4>>>
Review src/api and reply with DONE or BLOCKED.

When done, reply ... first body line:
<<<herdr-msg:v1:ack:review:m1a2b3c4>>>
```

The `[herdr-msg ...]` header is metadata only (terminals wrap it; TUIs decorate it). **Synchronization matches short sentinel lines** with a unique `MSGID`, not headers and not `agent-status done`.

After a successful `pane run` the helper prints a key=value **receipt** (`state=delivered`, `msg_id`, `reply_fields`, `hint`, ...). Receipts intentionally **omit** the exact waitable `<<<...>>>` token so `herdr wait output` on SELF cannot false-match the helper's own output. Reconstruct with `reply_fields=v1:ack:TASK:MSGID`. Optional flags:

| Flag | Role |
|------|------|
| `--verify` | Optional screen observation of the **outbound sentinel** on the target. Unobserved → `delivered-unobserved`, still exit `0`. |
| `--wait-reply` | Block on SELF for the ack sentinel (token in memory only until a real reply arrives). |
| `--msg-id` / `--ack-of` | Set or correlate message ids (invalid ids rejected; task/msg-id length-capped). |
| `--timeout` / `--verify-timeout` | Millisecond timeouts. |
| `--legacy-wait-header` | Deprecated: **replaces** sentinel wait with header regex. |
| `--allow-self-target` | Allow target pane == SELF (refused by default for `request` / `--wait-reply`). |
| `--allow-empty` | Allow empty body (rejected by default). |
| `--dry-run` | Print payload and receipt without sending. |

Exit codes: `0` ok (including delivered-unobserved), `1` send/resolve fail, `2` usage/env, `4` wait-reply timeout.

Post-send protocol: **deliver → stop polling the target → continue local work or `--wait-reply`**. Repeated `agent read` / `pane read` of the worker after send is a protocol violation.

The skill still documents `agent wait` and `wait agent-status` as fallbacks. Preferred sync is the ack sentinel on the sender's own pane.
