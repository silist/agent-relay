# relay — 多 Agent tmux 编排与人工升级中继

在一台 tmux 机器上同时跑多个 codex / claude code：一个当 **orchestrator**，其余当 **worker**。
worker 停下时自动把问题路由给 orchestrator，决策自动回灌；orchestrator 也决定不了的，
通过飞书 / 邮件升级给人。

单文件 Python，仅用标准库，无第三方依赖。

## 架构

```
worker 停下 --hook--> relay event --> ~/.agent-relay/inbox/
                                        │ daemon（1s 轮询，一次只放一条）
                                        ▼
                    paste 到 orchestrator pane（含 worker 名、末尾输出、回复格式）
                                        │ orchestrator 执行 shell 命令
                                        ▼
                     relay decide | done | escalate --> outbox/
                                        │ daemon 读取并处理
        decide --> paste 回 worker pane    escalate --> 飞书 / 邮件通知人
                                                            │
                                    人 ssh 执行 relay answer --> 回灌 worker
```

设计要点：

- **不轮询屏幕**：用 hook 检测 worker 停下。
  - Claude Code：`Stop` hook，取 `last_assistant_message`，缺失时解析 `transcript_path`。
  - Codex：`notify`（`agent-turn-complete`），JSON 作为最后一个参数，取 `last-assistant-message`。
- **不解析 TUI 输出**：orchestrator 通过执行 `relay` 命令回复，命令只写文件，可靠且 TUI 仍可观察。
- **状态落盘**：`~/.agent-relay/`，daemon 是 `state.json` 的唯一写者。

## 安装

```bash
mkdir -p ~/bin && cp bin/relay ~/bin/relay && chmod +x ~/bin/relay
cp config.example.json ~/.agent-relay/config.json   # 需先 mkdir
# 编辑 config.json，填写 orchestrator_pane 与渠道
```

### 接入 hook

Claude Code（`~/.claude/settings.json`）：

```json
{"hooks":{"Stop":[{"hooks":[{"type":"command","command":"relay event --src claude"}]}]}}
```

Codex（`~/.codex/config.toml`）：

```toml
notify = ["relay", "event", "--src", "codex"]
```

orchestrator 的启动约束：

- **codex**：需要 `[sandbox_workspace_write] writable_roots = ["/home/USER/.agent-relay"]`，
  否则沙箱内该目录只读，`relay decide` 会失败；同时需在 `developer_instructions` 里写
  收到 `[RELAY ...]` 时的行为规范（codex 不读 `~/.codex/AGENTS.md`）。
- **claude**：在 orchestrator 工作目录的 `AGENTS.md` 写明协议。

## 命令

| 命令 | 调用方 | 作用 |
|---|---|---|
| `relay init` | 人 | 写 `~/.agent-relay/config.json`（在 orchestrator pane 中执行） |
| `relay register [pane]` | 人 | 把当前/指定 pane 注册为 worker |
| `relay unregister [pane]` | 人 | 移除 worker |
| `relay event --src claude\|codex` | hook | 事件入口，永远退出 0 |
| `relay decide <worker> <text>` | orchestrator | 回灌决策 |
| `relay done <worker> <text>` | orchestrator | 标记完成 |
| `relay escalate <worker> <text>` | orchestrator | 升级给人 |
| `relay context <worker> [行数]` | orchestrator | 拉取 worker 完整原文（只读） |
| `relay answer <worker> [text]` | 人 | 回灌人的决策 / 无 text 仅解除静默 |
| `relay daemon` | 常驻 | 调度主循环 |
| `relay status` | 人 | 查看状态 |

## 防护机制

| 风险 | 措施 |
|---|---|
| 死循环 | `max_rounds`（默认 64），超限升级给人 |
| 空转 | `skip_background_tasks`：worker 有后台任务在跑时 Stop 不派发 |
| 频繁空转 | `min_dispatch_interval`：同 worker 两次派发最小间隔 |
| 兜底恢复 | `pending_slack`：被过滤的事件超时后补派一次，防死锁 |
| orchestrator 乱批危险操作 | `dangerous_regex` 命中不可逆操作时拦截转人工 |
| 部署类字样 | `caution_regex` 只标注不拦截（`部署`/`上线` 出现在否定句中很常见） |
| orchestrator 无响应 | `timeout`（默认 900s）后升级 |
| 升级期间重复打扰 | `waiting_human` 时丢弃该 worker 的新事件 |
| 误路由 | 未知 worker 的 `decide` 直接拒绝，不按 in-flight 兜底 |
| 上下文成本 | 只喂任务末尾输出；orchestrator 可用 `relay context` 主动拉取 |

## 轮次计数规则

`rounds` 表示「orchestrator 已经给过多少次决策」，不是「worker 停了几次」。

| 动作 | rounds | status |
|---|---|---|
| `decide` | +1 | running |
| `done` | 归零 | idle |
| `answer` | 归零 | running / idle |
| `escalate` / 超时 / 达上限 | 不变 | waiting_human |

被过滤的事件（后台任务、节流、白名单外、空消息）**不消耗轮次**。

## 已知限制

- **codex 的 `Stop` hook 无 `background_tasks` 字段**；实测 codex 也没有真正的后台任务，
  `nohup ... &` 不会存活，且它会在失败时仍报告成功。用 codex 当 worker 时，
  orchestrator 应通过 `relay context` 核对实况，不要只信最后一条消息。
- **codex hooks 需要信任状态**才会执行，默认静默跳过（`--dangerously-bypass-hook-trust` 可绕过）。
  日常用 `notify` 即可。
- **邮件回灌默认关闭**（`mail_reply_enabled: false`）：QQ 对同域自发邮件不做 SPF/DKIM 校验，
  `From` 白名单挡不住外部服务器伪造发件人。
- 飞书为单向通知，回复需 `relay answer`。

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
