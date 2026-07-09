# Herdr Skill

[English](README.md) | 中文

这个仓库用于构建更适合 Herdr 多 agent 协作的 Agent Skill。它不是 [Herdr](https://github.com/ogulcancelik/herdr) 应用本体；可安装目录是 [`herdr/`](herdr/)，入口是 [`herdr/SKILL.md`](herdr/SKILL.md)。

## 目的

agent 容易反复读 sibling pane，或长时间阻塞等 `done`。本 skill 默认用 **轻量聊天契约**：

- 先用 `herdr agent list` / `herdr pane list` 发现；
- 发一条短消息，header 主要带 **reply-to**（和可选 **task**）；
- 发送成功 = **delivered**，然后 **结束本轮**；
- 不轮询对方，也不阻塞等回信；
- 对方把回复注入你的 pane 时，再处理这条 inbound；
- 详细 CLI 放在 `references/`。

## 仓库内容

```text
herdr/
  SKILL.md                         核心工作流
  scripts/herdr-msg                薄 header + pane run helper
  references/command-reference.md  CLI 参考
  agents/openai.yaml               客户端 UI 元数据
```

helper 是纯 Bash，不依赖 Python / `jq`。

## 安装

```bash
npx skills add yangyang0507/herdr-skill
```

或把 `herdr/` 拷到客户端 skill 目录（例如 `.agents/skills/herdr`）。

在 `HERDR_ENV=1` 的 Herdr pane 中使用。

## 校验

```bash
bash -n herdr/scripts/herdr-msg
```

## 设计说明

消息形态：

```text
[herdr-msg reply-to:w1:p1 task:review]
Please review src/api. Reply DONE or BLOCKED to reply-to.
```

`scripts/herdr-msg` 只做：

1. 解析目标 agent/pane；
2. 自动填 `reply-to`；
3. `herdr pane run` 发出；
4. 打印小回执（`state=delivered`，hint 要求结束本轮）；
5. 退出。

默认 **没有** msg_id、sentinel、`--verify`、`--wait-reply`。那些是为机器阻塞匹配准备的；日常聊天用「对方 push 进你的 pane」即可。

```bash
herdr/scripts/herdr-msg codex --task review <<'MSG'
Review the lightweight protocol.
MSG
```

发送后协议：**送达 → 停止 → 结束本轮 → 处理 inbound 回信**。
