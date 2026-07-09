# Herdr Skill

[English](README.md) | 中文

这个仓库用于构建一个更适合 Herdr 多 agent 协作的 Agent Skill。它不是 [Herdr](https://github.com/ogulcancelik/herdr) 应用本体，真正可安装的 skill 目录是 [`herdr/`](herdr/)，主入口是 [`herdr/SKILL.md`](herdr/SKILL.md)。

## 创建目的

原版 Herdr skill 对 CLI 能力说明较多，但对多 agent 协作工作流描述不足，容易让 agent 默认走低信号等待，例如长时间执行 `herdr wait agent-status ... --status done`。在实际协作中，很多等待并不必要：目标 pane 的输出已经足够判断状态，或者目标 agent 应该主动把结果回复给发起方。

这个仓库的目标是把 Herdr skill 调整成更明确的协作指南：

- 先用 `herdr agent list` 和 `herdr pane list` 做发现；
- 行动前 **只读一次** 目标 pane 或 agent 的当前输出（不是轮询）；
- 等待明确的输出标记，而不是笼统等待 `done`；
- agent 间消息带上 `reply-to`、发送者、任务 id 等元信息；
- `herdr-msg` 发送成功即视为 **DELIVERED**，之后停止观察对方；
- 需要同步时在 **自己的 pane** 上等 `kind:reply task:<id>`（或使用 `--wait-reply`）；
- 发出委托后继续处理本地可推进的工作，不轮询 sibling agent；
- 把详细命令参考放到 `references/` 中，保持主 `SKILL.md` 精简。

## 仓库内容

```text
herdr/
  SKILL.md                         核心 agent 工作流
  scripts/herdr-msg                无 Python/JQ 依赖的结构化消息 helper
  references/command-reference.md  Herdr CLI 详细参考
  agents/openai.yaml               skill 客户端 UI 元数据
```

`scripts/herdr-msg` 使用纯 Bash 实现，避免在没有 Python 或 `jq` 的 agent shell 中失败。

## 参考来源和借鉴做法

这个 skill 主要参考了以下项目和规范：

- [Agent Skills specification](https://github.com/agentskills/agentskills)：skill 目录结构、`SKILL.md` frontmatter、渐进式披露和校验约束。
- [Herdr](https://github.com/ogulcancelik/herdr)：Herdr 的 workspace/tab/pane/agent 模型、CLI 行为、环境变量如 `HERDR_ENV`、`HERDR_SOCKET_PATH`、`HERDR_PANE_ID`，以及原版 Herdr skill。
- [smux](https://github.com/yangyang0507/smux)：借鉴其跨 agent 通信思路，让消息中包含发送者和回复目标，让接收方主动回复，而不是让发送方不断轮询。

核心变化是把 smux 中有效的消息纪律迁移到 Herdr 原生命令上：这里不依赖 tmux-bridge，而是使用 Herdr 自己的 `agent`、`pane` 和 `wait` 命令。

## 安装方式

推荐使用 [`skills`](https://skills.sh/) CLI 安装：

```bash
npx skills add yangyang0507/herdr-skill
```

这个命令会通过 Vercel `skills.sh` 的方式从当前 GitHub 仓库安装 skill。

手动备用方式：把 `herdr/` 目录复制到你的 agent 客户端支持的 skill 目录中。例如使用 `.agents/skills` 布局的客户端：

```bash
mkdir -p .agents/skills
cp -R herdr .agents/skills/herdr
```

这个 skill 应该在 Herdr 托管的 pane 中使用，也就是 agent 进程里能看到 `HERDR_ENV=1`。

## 校验

基础脚本校验：

```bash
bash -n herdr/scripts/herdr-msg
```

如果本地有 Agent Skills 的校验 helper：

```bash
uv run --with pyyaml python /path/to/quick_validate.py herdr
```

涉及 live pane 控制的行为应只在 Herdr 内部测试。

## 设计说明

`herdr/scripts/herdr-msg` 会发送并提交类似这样的消息：

```text
[herdr-msg from:codex pane:p_42 reply-to:p_42 at:w1/w1:1 kind:request task:review]
Review src/api and reply with DONE or BLOCKED.
```

接收方可以直接从消息头中知道应该回复到哪里，从而避免常见的低效循环：发送任务后等待目标 agent 进入 `done`，长时间阻塞，超时后才读取目标 pane。

发送成功后 helper 会打印 key=value **回执**（`state=delivered`、`match=...`、`hint=...`），明确告知「已送达」以及下一步该做什么。可选参数：

| 参数 | 作用 |
|------|------|
| `--verify` | 在目标 pane 上做 **一次性** 送达确认（不是任务完成） |
| `--wait-reply` | 在发送方自己的 pane 上阻塞等待 `kind:reply task:<id>` |
| `--timeout` / `--verify-timeout` | 上述等待的毫秒超时 |
| `--dry-run` | 只打印 payload 和回执，不真正发送 |

退出码：`0` 成功，`1` 发送/解析失败，`2` 用法/环境错误，`3` verify 失败，`4` wait-reply 超时。

期望的发送后协议是：**送达 → 停止轮询对方 → 继续本地工作，或在自己 pane 上等回复**。skill 将发送后反复 `agent read` / `pane read` 对方视为协议违规。

skill 中仍然保留 `agent wait` 和 `wait agent-status` 的说明，但把它们视为兜底工具。默认同步点应是明确输出标记，或发送方自己 pane 上的结构化回复。
