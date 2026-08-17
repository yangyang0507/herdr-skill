# Herdr Skill

English | [中文](README.zh-CN.md)

This repository builds an improved Agent Skill for [Herdr](https://github.com/ogulcancelik/herdr), focused on practical multi-agent coordination without heavy waiting or polling.

It is not the Herdr application itself. The deliverable is the installable skill directory at [`herdr/`](herdr/), whose main entry point is [`herdr/SKILL.md`](herdr/SKILL.md).

## Purpose

Agents often waste turns polling sibling panes or blocking on vague `done` status. This skill defaults to a **lightweight chat contract**:

- discover with `herdr agent list` / `herdr pane list`;
- send a short message with **reply-to** (and optional **task**);
- treat successful send as **delivered**, then **end the turn**;
- do not poll the peer and do not block-wait for replies;
- when the peer injects a reply into your pane, handle that inbound message;
- keep detailed CLI notes behind progressive disclosure.

## What Is Included

```text
herdr/
  SKILL.md                         core agent-facing workflow
  scripts/herdr-msg                thin header + pane-run helper
  references/command-reference.md  detailed Herdr CLI reference
  agents/openai.yaml               UI metadata for skill clients
```

The helper is plain Bash (no Python/`jq`).

## Install

```bash
npx skills add yangyang0507/herdr-skill
```

Manual: copy `herdr/` into a client skill directory such as `.agents/skills/herdr`.

Use from a Herdr-managed pane where `HERDR_ENV=1`.

## Validate

```bash
bash -n herdr/scripts/herdr-msg
```

## Design Notes

Messages look like:

```text
[herdr-msg reply-to:w1:p1 task:review]
Please review src/api. Reply DONE or BLOCKED to reply-to.
```

`scripts/herdr-msg`:

1. resolves the target agent/pane;
2. autofills `reply-to` from SELF;
3. sends via `herdr pane run`;
4. if the target is an idle/done agent that does not start, sends Enter once;
5. prints a tiny receipt (`state=delivered`, `hint=End turn...`);
6. exits.

There is no default `msg_id`, sentinel, `--verify`, or `--wait-reply`. Those were useful for machine wait-matching; for day-to-day chat they add weight without helping the natural “push reply into my pane” wake-up.

```bash
herdr/scripts/herdr-msg codex --task review <<'MSG'
Review the lightweight protocol.
MSG
```

Post-send protocol: **deliver → stop → end turn → handle inbound reply**.
