# relay — 多 Agent tmux 编排与人工升级中继

在一台 tmux 机器上同时跑多个 codex / claude code：一个当 **orchestrator**，其余当 **worker**。
worker 停下时自动把问题路由给 orchestrator，决策自动回灌；orchestrator 也决定不了的，
通过飞书 / 邮件升级给人。

单文件 Python，仅标准库，无第三方依赖。

## 为什么不用现成方案

| 工具 | 问题 |
|---|---|
| Claude Squad | 只管并行会话 + worktree，没有向 orchestrator 升级 |
| Claude Code subagent / Agent SDK | 进程内派生，不跨进程、不跨 codex/claude |
| claude-flow / OpenHands | 重型框架，低配机器负担大，与已在 tmux 中运行的 CLI 集成差 |

本方案 = tmux 原语 + hook + 一个单文件脚本。

## 架构

```
worker 停下 --hook--> relay event --> ~/.agent-relay/inbox/
                                        │ daemon（1s 轮询，一次只放一条）
                                        ▼
              paste 到 orchestrator pane（worker 名 + 末尾输出 + 回复格式）
                                        │ orchestrator 执行 shell 命令
                                        ▼
                     relay decide | done | escalate --> outbox/
                                        │ daemon 读取并处理
        decide --> paste 回 worker pane     escalate --> 飞书 / 邮件通知人
                                                              │
                                     人执行 relay answer --> 回灌 worker
```

三个关键设计决策：

1. **不轮询屏幕，用 hook 检测 worker 停下**
   - Claude Code：`Stop` hook，取 `last_assistant_message`，缺失时解析 `transcript_path`
   - Codex：`notify`（`agent-turn-complete`），JSON 作为最后一个参数，取 `last-assistant-message`

2. **不解析 TUI 输出**
   orchestrator 通过执行 `relay` 命令回复，命令只写文件到 `outbox/`。比抓屏可靠，且 TUI 始终可人工观察、接管。

3. **daemon 是 `state.json` 的唯一写者**
   连 `relay answer` 也走 `outbox/`，避免多进程写同一份 JSON。

## 安装

```bash
mkdir -p ~/bin ~/.agent-relay
cp bin/relay ~/bin/relay && chmod +x ~/bin/relay
cp config.example.json ~/.agent-relay/config.json
chmod 600 ~/.agent-relay/config.json     # 含 SMTP 凭据
$EDITOR ~/.agent-relay/config.json
```

在 orchestrator pane 里执行 `relay init` 可自动填入当前 pane 的 `orchestrator_pane`。

### 接入 Claude Code

`~/.claude/settings.json`：

```json
{
  "hooks": {
    "Stop": [
      { "hooks": [ { "type": "command", "command": "relay event --src claude" } ] }
    ]
  }
}
```

### 接入 Codex

`~/.codex/config.toml`：

```toml
notify = ["relay", "event", "--src", "codex"]
```

### orchestrator 侧的两条硬约束

**① codex 当 orchestrator 必须放行可写目录。** 默认 `workspace-write` 沙箱下
`~/.agent-relay` 是只读的，`relay decide` 会报 `Read-only file system`：

```toml
[sandbox_workspace_write]
writable_roots = ["/home/USER/.agent-relay"]
```

**② orchestrator 必须知道 `[RELAY ...]` 协议。**
- claude：把 `AGENTS.md` 放到 orchestrator 的工作目录；
- codex：它不读 `~/.codex/AGENTS.md`，只读仓库目录树里的 `AGENTS.md`。
  用全局 `developer_instructions` 写协议，避免污染项目自带的 `AGENTS.md`。

配置改完必须**重启** orchestrator 进程，配置只在启动时读取。

### 启动 daemon

```bash
tmux new-session -d -s relay 'relay daemon'
```

## 命令

| 命令 | 调用方 | 作用 |
|---|---|---|
| `relay init` | 人 | 写 `~/.agent-relay/config.json`（在 orchestrator pane 中执行） |
| `relay register [pane]` | 人 | 把当前/指定 pane 注册为 worker（在 worker pane 中执行） |
| `relay unregister [pane]` | 人 | 移除 worker |
| `relay event --src claude\|codex` | hook | 事件入口，**永远退出 0** |
| `relay decide <worker> <text>` | orchestrator | 回灌决策 |
| `relay done <worker> <text>` | orchestrator | 标记完成 |
| `relay escalate <worker> <text>` | orchestrator | 升级给人 |
| `relay context <worker> [行数]` | orchestrator | 拉取 worker 完整原文（只读） |
| `relay answer <worker> [text]` | 人 | 回灌人的决策 / 无 text 仅解除静默 |
| `relay daemon` | 常驻 | 调度主循环 |
| `relay status` | 人 | 查看状态 |

`relay context` 存在的理由：派发消息里的 worker 输出有长度上限，常会截掉结论。
让 orchestrator 能主动要多一点，而不是拿半截信息去猜。

## 配置

```jsonc
{
  "orchestrator_pane": "%3",       // orchestrator 的 tmux pane id
  "worker_panes": ["%1"],          // worker 白名单；空数组 = 拒绝全部事件
  "channels": ["feishu", "email"], // 升级通知渠道
  "feishu_webhook": "https://open.feishu.cn/open-apis/bot/v2/hook/xxx",
  "email": {
    "host": "smtp.qq.com", "port": 465,
    "user": "you@example.com", "password": "授权码", "to": "you@example.com",
    "imap_host": "imap.qq.com", "imap_port": 993,
    "from_whitelist": []
  },

  "max_rounds": 64,                // 单任务最大决策轮次
  "timeout": 900,                  // orchestrator 多久不回就升级(s)
  "min_dispatch_interval": 300,    // 同 worker 两次派发最小间隔(s)
  "skip_background_tasks": true,   // worker 有后台任务时不派发
  "pending_slack": 3600,           // 被过滤事件的兜底补派延迟(s)
  "notify_done": true,             // 完成时也发通知

  "dangerous_regex": "...",        // 命中即拦截转人工（只放不可逆操作）
  "caution_regex": "...",          // 命中只在决策里加提示，不拦截
  "worker_tail_chars": 2000,       // 派发时附带 worker 输出的字符数
  "escalate_tail_chars": 1500,     // 升级通知里附带的字符数

  "imap_poll_interval": 30,
  "mail_reply_enabled": false      // 邮件回灌开关，默认关（见「已知限制」）
}
```

### `dangerous_regex` 只该放不可逆操作

默认值只匹配 `rm -rf /`、`force push`、`drop table`、`git reset --hard` 这类。

**不要把「部署」「删除」「付款」这类动词放进去。** 实测教训：orchestrator 在决策里写
「禁止部署、生产写入」，被正则命中 `部署.*生产`，整条决策被静默拦截，worker 永远等不到回复。
用正则猜自然语言意图做不到——否定句和命令句用的是同一批词。这类词放 `caution_regex`，
只做标注。

## 防护机制

| 风险 | 措施 |
|---|---|
| 死循环 | `max_rounds`，超限升级给人 |
| 空转 | `skip_background_tasks`：worker 有后台任务在跑时 Stop 不派发 |
| 频繁空转 | `min_dispatch_interval`：同 worker 两次派发最小间隔 |
| 兜底恢复 | `pending_slack`：被过滤的事件超时后补派，防死锁 |
| 乱批不可逆操作 | `dangerous_regex` 拦截转人工 |
| orchestrator 无响应 | `timeout` 后升级 |
| 升级期间重复打扰 | `waiting_human` 时丢弃该 worker 的新事件 |
| 误路由 | 未知 worker 的 `decide` 直接拒绝，不用 in-flight 兜底 |
| 投递失败 | 重试 `max_paste_retries` 次后转人工，不无限重试 |
| 上下文成本 | 只喂末尾输出；orchestrator 可用 `relay context` 主动拉取 |

### 为什么需要 `skip_background_tasks`

worker 等异步事件（备份、CI、Monitor）时，每轮思考完都会触发 `Stop`。若每次都派发：

```
实测 mp-sql 一次部署：33s、35s、57s 连发 5 轮，orchestrator 每轮只回「继续等」，
5 次高耗模型调用全部浪费，最后撞上轮次上限。
```

Claude 的 Stop payload 带 `background_tasks`，非空即「本轮结束但任务还在跑」，这是确定性信号，
不需要模型判断。

## 轮次计数规则

`rounds` 表示「orchestrator 已经给过多少次决策」，不是「worker 停了几次」。

| 动作 | rounds | status |
|---|---|---|
| `decide` | +1 | running |
| `done` | 归零 | idle |
| `answer` | 归零 | running / idle |
| `escalate` / 超时 / 达上限 | 不变 | waiting_human |

被过滤的事件（后台任务、节流、白名单外、空消息）**不消耗轮次**。

`done` 归零很重要：否则 worker 做完一个任务就永久停在 `done`，之后的新任务会被静默丢弃。

## 状态目录

```
~/.agent-relay/
├── config.json     # 配置（含凭据，chmod 600）
├── state.json      # 各 worker 状态，daemon 唯一写者
├── inbox/          # 待派发事件
├── pending/        # 被 background_tasks 过滤、等待兜底的事件
├── outbox/         # orchestrator / 人的指令
├── dropped.log     # 被丢弃的事件及原因
└── relay.log       # 运行日志
```

`dropped.log` 里每行是 `时间 \t 原因 \t worker`。常见原因：`background-tasks`、`throttled`、
`not-in-worker-panes`、`is-orchestrator`、`waiting_human`、`empty-message`。

## 已知限制

- **codex 没有 `background_tasks`**。实测 codex 也没有真正的后台任务：`nohup ... &` 不会存活，
  且它会在失败时仍报告成功。用 codex 当 worker 时，orchestrator 应通过 `relay context`
  核对实况，不要只信最后一条消息。
- **codex hooks 需要信任状态**才执行，默认静默跳过（`--dangerously-bypass-hook-trust` 可绕过）。
  日常用 `notify` 即可，字段已够用。
- **邮件回灌默认关闭**（`mail_reply_enabled: false`）：QQ 对同域自发邮件不做 SPF/DKIM 校验，
  `From` 白名单挡不住外部服务器伪造发件人。白名单本身已用 `parseaddr` 全等比对（子串匹配
  会让 `user@example.com.evil.com` 通过），但不解决伪造问题。
- **飞书是单向通知**。要能收消息需企业自建应用 + 公网回调地址；回复请用 `relay answer`。
- **paste 后需要一次 Enter**。Claude 与 Codex 的 TUI 实测一次即可提交，但这是版本相关的行为。
- **所有升级/拦截只写本地日志**，除非配了 `channels`。不配渠道时故障是静默的。
