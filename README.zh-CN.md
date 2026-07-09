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
- 需要同步时优先 `--wait-reply`；或从回执 `reply_fields` 重建 ack token（**不要**先把 raw token 打进 SELF 再 wait）；
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

`herdr/scripts/herdr-msg` 发送带人类可读 header 与机器 **sentinel** 的消息：

```text
[herdr-msg id:m1a2b3c4 from:codex pane:p_42 reply-to:p_42 at:w1/w1:1 kind:request task:review]
<<<herdr-msg:v1:send:review:m1a2b3c4>>>
Review src/api and reply with DONE or BLOCKED.

When done, reply ... first body line:
<<<herdr-msg:v1:ack:review:m1a2b3c4>>>
```

`[herdr-msg ...]` header 只作元数据（终端会折行、TUI 会插入装饰）。**同步匹配的是短 sentinel 行 + 唯一 MSGID**，不是 header，也不是 `agent-status done`。

`pane run` 成功后 helper 打印 key=value **回执**（`state=delivered`、`msg_id`、`reply_fields`、`hint` 等）。回执**故意不打印**完整 waitable token（`<<<...>>>`），以免 `herdr wait output` 在 SELF 上命中自己的输出；用 `reply_fields=v1:ack:TASK:MSGID` 重建。可选参数：

| 参数 | 作用 |
|------|------|
| `--verify` | 可选：在目标 pane 上观察 **outbound sentinel**。未观察到 → `delivered-unobserved`，仍 exit `0` |
| `--wait-reply` | 在自己的 pane 上阻塞等待 ack sentinel（token 只在内存，回执不打印 raw token） |
| `--msg-id` / `--ack-of` | 设置或关联 message id（非法 id 直接拒绝；task/msg-id 有长度上限） |
| `--timeout` / `--verify-timeout` | 毫秒超时 |
| `--legacy-wait-header` | 已废弃：**替换** sentinel 等待为 header 正则（不是叠加） |
| `--allow-self-target` | 允许目标 pane == SELF（`request` / `--wait-reply` 默认拒绝） |
| `--allow-empty` | 允许空 body（默认拒绝） |
| `--dry-run` | 只打印 payload 和回执，不真正发送 |

退出码：`0` 成功（含 delivered-unobserved），`1` 发送/解析失败，`2` 用法/环境错误，`4` wait-reply 超时。

发送后协议：**送达 → 停止轮询对方 → 继续本地工作，或 `--wait-reply`**。发送后反复 `agent read` / `pane read` 对方视为协议违规。

skill 仍把 `agent wait` / `wait agent-status` 当作兜底。默认同步点是发送方自己 pane 上的 ack sentinel。
