# Changelog

## Unreleased

### 修复

- **pane 用稳定目标而不是 `%N`**。tmux server 重启后 pane 编号会重排，配置里存的
  `%3` 变成无效 target，daemon 每秒重试同一件事、事件永远派不出去。现在
  `orchestrator_pane` / `worker_panes` 都存 `session:window.pane`，使用时再解析。
- **worker 的 pane 已退出时直接丢弃事件**，不再无限重试（`dropped.log` 记
  `worker-pane-gone`）。
- **orchestrator 解析不到时不再丢事件**：保留在 inbox 等重新登记，且告警只记一次。
- **`relay done` 带文本时会投递给 worker**。此前它只改本地状态，文本被静默丢弃，
  orchestrator 以为交代过、worker 一个字都没收到。带空文本仍为纯记账。
- **`looks_like_request` 补句式**。「等你的授权」「待您确认」「是否继续」这类请求
  不被识别，会被当成空转压进 `pending/` 排队一小时。新增 5 组句式覆盖「等人给东西」
  和「向人提问」。这类枚举必有漏网，所以 `pending_slack` 兜底保留。
- **移除 `max_rounds` 轮次上限**。它假设 `rounds` 能标识「同一个任务」，但既没有
  判断「是不是同一个问题」的逻辑，也没有「开新任务就释放轮次」的入口。于是它只是
  个单调递增的计数器：到上限后每次 Stop 都升级，而 `decide` 仍回灌并继续 `rounds += 1`，
  worker 被永久锁死在 `waiting_human`。真实事故：mp-sql 到 68 轮后彻底失联。
  `rounds` 现在只作计数展示。
- **空文本的 `done` 不再发完成通知**。`done` 现在会投递文本，worker 收到后再次
  Stop，orchestrator 常以 `relay done <worker> ""` 收尾——而 `notify_done` 无条件
  发信、用 `(无摘要)` 兜底，于是一次真实收尾收到两封邮件。现在无内容则不通知。
- **`done` 判定统一用 `text.strip()`**。此前 paste 用 `text`、通知用 `text.strip()`，
  纯空白文本会一边被 paste 出去、一边跳过通知。

### 新增

- `relay orch [pane]`：在 orchestrator 的 pane 里登记自己，与 `relay register` 对称。
  tmux 重启后跑一次即可，不用手改 `config.json`。
- `relay status` 会标出解析不到的目标（`⚠ 解析不到存活 pane` / `⚠ 已退出`）。

## 0.1.0 — 2026-11-06

首个可发布版本。从一份设计文档落地为可运行的实现，并在真实的多 agent 部署中跑通。

### 能力

- **hook 检测 worker 停下**：Claude Code `Stop` hook、Codex `notify`，不轮询屏幕。
- **决策路由**：worker → orchestrator → 人。orchestrator 通过执行 `relay` 命令回复。
- **`relay context`**：给 orchestrator 主动拉取 worker 全文的命令，避免凭截断的输出做判断。
- **升级通知**：飞书 webhook / SMTP。
- **邮件回灌**：直接回复升级邮件即可回灌，用 `In-Reply-To` 校验来源。

### 防护

- `skip_background_tasks`：worker 有后台任务在跑时不派发（确定性信号，不用模型）。
- `min_dispatch_interval`：同一 worker 的派发节流。
- `pending_slack`：被过滤事件的兜底补派，防死锁。
- `max_rounds`：单任务决策轮次上限。
- `dangerous_regex` / `caution_regex`：不可逆操作拦截、可回滚操作标注。
- 未知 worker 拒绝投递，防误路由。

### 设计取舍

- **不用模型做路由判断**。过滤是否派发是确定性信号（`background_tasks` 是否为空），
  加一次模型调用比空转更贵，且误判会导致静默死锁。
- **`dangerous_regex` 不放「部署」「删除」这类动词**。它们同样出现在否定句里，
  正则无法区分。真实事故记录在 README 与本文件对应的提交里。
- **邮件回灌以 `In-Reply-To` 为主判据**，`From` 白名单只是第一道过滤。
