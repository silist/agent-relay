# Changelog

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
