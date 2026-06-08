# Herdr Skill

English | [中文](README.zh-CN.md)

This repository builds an improved Agent Skill for [Herdr](https://github.com/ogulcancelik/herdr), focused on making multi-agent coordination inside Herdr more practical and less wasteful.

It is not the Herdr application itself. The deliverable is the installable skill directory at [`herdr/`](herdr/), whose main entry point is [`herdr/SKILL.md`](herdr/SKILL.md).

## Purpose

The original Herdr skill documents the CLI surface, but its coordination recipes can lead agents toward long, low-signal waits such as waiting for another pane's agent status to become `done`. In multi-agent workflows this creates unnecessary blocking, especially when the useful signal is already visible in pane output or should be sent back as an explicit reply.

This repo refines that behavior around a few defaults:

- prefer `herdr agent list` and `herdr pane list` for discovery;
- read concrete pane output before acting;
- wait for named output markers, not vague completion state;
- use structured agent messages with `reply-to` metadata;
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

`herdr/scripts/herdr-msg` sends messages like:

```text
[herdr-msg from:codex pane:p_42 reply-to:p_42 at:w1/w1:1 kind:request task:review]
Review src/api and reply with DONE or BLOCKED.
```

The receiver has enough metadata to reply directly to the sender. That avoids a common bad loop: sender sends work, waits for `done`, times out or blocks, then reads the target pane late.

The skill still documents `agent wait` and `wait agent-status`, but treats them as fallback tools. The preferred synchronization point is either a concrete output marker or a structured reply.
