# agent-relay

**在多 agent 之间路由决策：worker 卡住了，交给 orchestrator；orchestrator 也定不了，升级到人。**

一台机器上同时跑多个 `claude code` / `codex`，一个当 orchestrator，其余当 worker。
worker 停下时自动把问题路由给 orchestrator，决策自动回灌；orchestrator 也判断不了的，
通过飞书 / 邮件升级给人，人**直接回复邮件**即可回灌。

单文件 Python，仅标准库，无第三方依赖。适合低配机器（常驻开销接近零）。

```
              ┌──────────────────────────────────────────────┐
worker 停下 ──┤ relay event（hook）                          │
              └──────────────────┬───────────────────────────┘
                                 ▼
                            inbox/
                                 │ daemon（1s 轮询，一次只放一条）
                                 ▼
                     paste 到 orchestrator pane
                                 │ orchestrator 执行 shell 命令
                                 ▼
                  relay decide / done / escalate  →  outbox/
                                 │ daemon 读取并处理
              ┌──────────────────┴───────────────────┐
              ▼                                      ▼
    paste 回 worker pane                    飞书 / 邮件通知人
                                                     │
                                    人直接回复邮件 → 自动回灌
```

## 为什么值得存在

多 agent 并行时，最贵的不是算力，是**你的注意力**。每个 worker 停在「需要你决定」时都会
打断你一次。常见的处置是：让 worker 写总结 → 你读 → 转发给 orchestrator → 再取回决策 →
发回 worker。全人工，且每次上下文都要重建。

agent-relay 把这段做成自动的，并给失败留了确定性的兜底。你的角色从「每轮的中转站」
变成「只在真正需要人判断的地方出现」。

## 核心设计

### 1. 用 hook 检测停下，不轮询屏幕

| agent | 机制 | 取到的字段 |
|---|---|---|
| Claude Code | `Stop` hook（stdin JSON） | `last_assistant_message`、`background_tasks`、`transcript_path` |
| Codex | `notify`（`agent-turn-complete`） | `last-assistant-message`、`thread-id` |

抓屏解析 TUI 在版本升级时必坏；hook 有结构化字段，且 `relay event` 保证永远退出 0，
不会阻塞被 hook 的 agent。

### 2. 不解析 orchestrator 的 TUI 输出

orchestrator 通过执行 `relay decide/done/escalate` 回复，命令只往 `outbox/` 写文件。
比抓屏可靠，而且 orchestrator 的 TUI 始终可以人工观察、随时接管。

### 3. daemon 是 `state.json` 的唯一写者

连人工的 `relay answer` 也走 `outbox/`，避免多进程写同一份 JSON。

## 安装

需要 Python 3.8+（仅标准库，无第三方依赖）。

### 方式一：单文件（推荐）

hook 里写的是 `relay event --src claude`，需要一个**稳定出现在 PATH 里**的可执行文件。
自己放一份最省心，升级时覆盖即可：

```bash
mkdir -p ~/bin ~/.agent-relay
curl -o ~/bin/relay https://raw.githubusercontent.com/silist/agent-relay/master/bin/relay
chmod +x ~/bin/relay
curl -o ~/.agent-relay/config.json https://raw.githubusercontent.com/silist/agent-relay/master/config.example.json
chmod 600 ~/.agent-relay/config.json          # 会存 SMTP 授权码
$EDITOR ~/.agent-relay/config.json
```

### 方式二：pip / uv

```bash
pip install git+https://github.com/silist/agent-relay
# 或
uv tool install git+https://github.com/silist/agent-relay
```

也可以 `git clone` 后 `pip install .` 本地安装。

> **不要用 `uvx` 跑 daemon 或写进 hook。** `uvx` 每次运行都从缓存里取环境，
> 路径随版本变化，而 hook 里的命令是写死的。`uvx` 只适合临时试用：
> `uvx --from git+https://github.com/silist/agent-relay relay status`

### 初始化

在 orchestrator 的 pane 里执行 `relay orch`（首次可先跑 `relay init` 写默认配置），
然后在 worker 的 pane 里执行 `relay register`。

两者存的都是**稳定目标** `session:window.pane`，不是 `%N` 这种 pane 编号。

> **为什么不用 pane 编号。** `%3` 是 tmux 的运行时编号，server 一重启就全部分配。
> 配置里存 `%3`，重启后它要么不存在、要么落到别的窗口上。实测踩过一次：
> daemon 每秒对着一个已经不存在的 `%3` 重试了 14 次，事件永远派不出去。
> 存 `session:w.p` 后，tmux 重启不需要改任何配置——名字还在，pane 就找得回来。
>
> 万一会话本身被重建、目标也失效了，`relay status` 会显式标 `⚠ 解析不到存活 pane`，
> 在 orchestrator 里重跑一次 `relay orch` 即可，不用手改 `config.json`。

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

### orchestrator 侧的两个硬约束

**① codex 当 orchestrator 必须放行可写目录。** 默认 `workspace-write` 沙箱下
`~/.agent-relay` 是只读的，`relay decide` 会直接失败并报 `Read-only file system`：

```toml
[sandbox_workspace_write]
writable_roots = ["/home/USER/.agent-relay"]
```

**② orchestrator 必须知道 `[RELAY ...]` 协议。**
- claude：把本仓库的 `AGENTS.md` 放进 orchestrator 的工作目录；
- codex：它**不读** `~/.codex/AGENTS.md`，只读仓库目录树内的 `AGENTS.md`。
  用全局 `developer_instructions` 注入协议，避免污染项目自带的 `AGENTS.md`。

改完配置必须**重启** orchestrator 进程——配置只在启动时读取。

### 启动 daemon

```bash
tmux new-session -d -s relay 'relay daemon'
```

## 命令

| 命令 | 调用方 | 作用 |
|---|---|---|
| `relay init` | 人 | 写 `~/.agent-relay/config.json`（在 orchestrator pane 中执行） |
| `relay orch [pane]` | 人 | 在当前/指定 pane 登记 orchestrator（tmux 重启后跑一次） |
| `relay register [pane]` | 人 | 把当前/指定 pane 注册为 worker（在 worker pane 中执行） |
| `relay unregister [pane]` | 人 | 移除 worker |
| `relay event --src claude\|codex` | hook | 事件入口，**永远退出 0** |
| `relay decide <worker> <text>` | orchestrator | 回灌决策 |
| `relay done <worker> <text>` | orchestrator | 标记完成（轮次归零），并投递 text；text 为空则只改状态 |
| `relay escalate <worker> <text>` | orchestrator | 升级给人 |
| `relay context <worker> [行数]` | orchestrator | 拉取 worker 完整原文（只读） |
| `relay answer <worker> [text]` | 人 | 回灌决策 / 无 text 仅解除静默 |
| `relay daemon` | 常驻 | 调度主循环 |
| `relay status` | 人 | 查看状态 |

`relay context` 是为「信息不足」准备的：派发时只附 worker 输出的末尾一段，常会截掉结论。
与其让 orchestrator 凭半截信息猜或直接升级，不如给它一个主动拉取全文的命令。

## 配置要点

完整字段见 `config.example.json`，几个关键的：

| 字段 | 默认 | 说明 |
|---|---|---|
| `orchestrator_pane` | `""` | orchestrator 的稳定目标（`session:w.p`），用 `relay orch` 写 |
| `worker_panes` | `[]` | 白名单，同样是稳定目标。**空数组 = 拒绝所有事件**，这是安全默认 |
| `min_dispatch_interval` | 120 | 同 worker 两次派发最小间隔（秒） |
| `skip_background_tasks` | `true` | worker 有后台任务时不派发 |
| `pending_slack` | 600 | 被过滤事件的兜底补派延迟（秒） |
| `dangerous_regex` | 见下 | 命中即**拦截**转人工 |
| `caution_regex` | 见下 | 命中只**标注**，不拦截 |
| `mail_reply_enabled` | `false` | 邮件回灌开关 |

### `dangerous_regex` 只该放不可逆操作

默认值只匹配 `rm -rf /`、`force push`、`drop table`、`git reset --hard` 这类。

**不要把「部署」「删除」「付款」这类动词放进去。** 真实教训：orchestrator 在决策里写
「禁止部署、生产写入」，被 `部署.*生产` 命中，整条决策被静默拦截，worker 永远等不到回复，
只留下一行本地日志。用正则猜自然语言意图做不到——否定句和命令句用同一批词。
这类词放 `caution_regex`，只做标注。

## 防护机制

| 风险 | 措施 |
|---|---|
| 空转 | `skip_background_tasks`：worker 有后台任务时 Stop 不派发 |
| 频繁空转 | `min_dispatch_interval` 节流 |
| 兜底恢复 | `pending_slack`：被过滤事件超时后补派，防死锁 |

**没有轮次上限。** 曾经有一个 `max_rounds`（默认 64），已移除：它假设 `rounds`
能标识「同一个任务」，但代码里既没有判断「是不是同一个问题」的逻辑，也没有
「开新任务就释放轮次」的入口（`rounds` 只在 `done`/`answer` 时归零）。结果是
单调递增的计数器——跑到上限后每次 Stop 都升级给人，而 `decide` 仍在回灌并继续
`rounds += 1`，worker 被永久锁死在 `waiting_human`，之后所有输出都不再进
orchestrator。真实事故：mp-sql 跑到 68 轮后彻底失联。

`rounds` 现在只作计数展示，不参与任何判断。要中断循环，用 `relay escalate`
或直接 `relay answer`。
| 乱批不可逆操作 | `dangerous_regex` 拦截 |
| orchestrator 无响应 | `timeout` 后升级 |
| 升级期间重复打扰 | `waiting_human` 时丢弃该 worker 的新事件 |
| 误路由 | 未知 worker 的 `decide` 直接拒绝，不用 in-flight 兜底 |
| 投递失败 | 重试上限后转人工，不无限重试 |
| 上下文成本 | 只喂末尾输出，orchestrator 可 `relay context` 主动拉取 |

### 为什么需要 `skip_background_tasks`

worker 等异步事件（备份、CI、Monitor）时，每轮思考完都会触发 `Stop`。若每次都派发：

```
实测某次部署：33s、35s、57s 连发 5 轮，orchestrator 每轮只回「继续等」，
5 次高耗模型调用全部浪费，最后撞上轮次上限。
```

Claude 的 Stop payload 带 `background_tasks`，非空即「本轮结束但任务还在跑」。
这是确定性信号，不需要模型判断——用模型来做这个过滤反而更贵、更慢，且误判会导致死锁。

## 轮次计数规则

`rounds` 表示「orchestrator 已经给过多少次决策」，不是「worker 停了几次」。
**它只作计数展示，不参与任何判断**——没有轮次上限。

| 动作 | rounds | status |
|---|---|---|
| `decide` | +1 | running |
| `done` | 归零 | idle |
| `answer` | 归零 | running / idle |
| `escalate` / 超时 | 不变 | waiting_human |

被过滤的事件（后台任务、节流、白名单外、空消息）**不消耗轮次**。

`done` 必须归零：否则 worker 做完一个任务就永久停在 `done`，之后的新任务会被静默丢弃。

`done` 的文本会投递给 worker（`[已完成] …`）。只记账、不想惊动 worker 时传空字符串：
`relay done mp-sql ""`。

空文本的 `done` 也**不发完成通知**。否则一次收尾会有两封邮件：worker 的完成摘要一封，
orchestrator 对它的空确认又一封——后者没有信息量。`notify_done` 只管有内容的 `done`。

## 邮件回灌的验证

`mail_reply_enabled: true` 后，直接回复升级邮件即可回灌。验证链：

```
1. From 精确等于白名单（parseaddr 全等，防 user@example.com.evil.com）
2. In-Reply-To / References 命中本系统发出的那封升级邮件的 Message-ID
3. 该工单仍在 waiting_human（天然防重放）
4. 正文非空、且不含其他待处理工单号
```

**第 2 条是核心。** 单纯验 `From` 是不够的——SMTP 允许发件人任意填写，而邮件服务商对
同域自发邮件往往不做 SPF/DKIM 校验。`In-Reply-To` 由邮件客户端回复时自动带上，指向的
Message-ID 只出现在发给你的那封信里，伪造者无从得知。

实现上有个坑：QQ 会改写客户端设置的 `Message-ID`，所以只能发信后按主题把它捞回来存下。

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

`dropped.log` 每行 `时间 \t 原因 \t worker`。常见原因：`background-tasks`、`throttled`、
`not-in-worker-panes`、`is-orchestrator`、`waiting_human`、`empty-message`、
`worker-pane-gone`（事件所属的 worker 已退出，无法投递）。

## 已知限制

- **codex 没有 `background_tasks`**，且实测它也没有真正的后台任务：`nohup ... &` 不会存活，
  且它会在失败时仍报告成功。用 codex 当 worker 时，orchestrator 应通过 `relay context`
  核对实况，不要只信最后一条消息。
- **codex hooks 需要信任状态**才执行，默认静默跳过。日常用 `notify` 即可，字段够用。
- **飞书是单向通知**。要能收消息需企业自建应用 + 公网回调地址；回复请用 `relay answer`。
- **不配 `channels` 时所有故障都是静默的**，只有本地日志。建议至少配一个通知渠道。
- **paste 后靠一次 Enter 提交**，Claude 与 Codex TUI 实测均可用，但属版本相关行为。
- **pane 目标靠 `session:window.pane` 定位**。同名会话被重建、或 window/pane 索引变了，
  需要重跑 `relay orch` / `relay register`。`relay status` 会把这些标成 `⚠`。
- 未在 Windows / 非 tmux 环境测试。

## 开发

```bash
python3 -c "import ast;ast.parse(open('bin/relay').read())"   # 唯一的语法检查
pip install -e .                                              # 需要可编辑安装时
```

`bin/relay` 是唯一权威源。`pyproject.toml` 用 `script-files` 把它按原样装进
`bin/`，不为了打包把单文件拆成模块再维护两份。

没有测试框架。验证方式见 `AGENTS.md`——关键是**状态类测试必须隔离 `HOME`**，
否则会把测试数据写进真实的 `~/.agent-relay/state.json`。
