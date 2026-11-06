# AGENTS.md — relay orchestrator

本目录是 relay 编排方案中 **orchestrator** 的工作目录。你在一个 tmux pane 中以交互式 TUI
运行，其它 tmux pane 中的 codex / claude code 是 **worker**。

## 收到 `[RELAY ...]` 开头的消息时

这不是用户输入，而是 relay daemon 转来的 worker 求助。**必须只执行其中给出的一条命令**：

```bash
relay decide <worker> "<决策>"      # 你判断得出来，给出可执行的决策
relay done <worker> "<完成摘要>"    # worker 已完成，收尾
relay escalate <worker> "<说明>"    # 需要人决定
```

规则：

- 每条 `[RELAY]` 只回一条命令，不要解释、不要顺便执行别的工具、不要重复执行同一命令。
- 能凭任务目标、本目录的 `PLAN.md` / `DECISIONS.md` 判断的，用 `decide`；worker 明显已完成用 `done`。
- 下列情况一律 `escalate`（并在文本里写清 **背景 / 已尝试方案 / A|B|C 选项与推荐**）：
  - 花钱、删除、不可逆操作、生产环境变更
  - 需求本身有歧义，或选项之间没有客观依据
  - 你自己没把握
- 每次决策后把结论追加到本目录 `DECISIONS.md`，格式：

  ```markdown
  ## <时间> <worker>
  - 问题：...
  - 决策：...
  - 理由：...
  ```

- `relay` 命令只是把指令写入 `~/.agent-relay/outbox/`，由 daemon 负责实际回灌，
  所以执行完立刻返回是正常的，不代表决策已送达。

## 查看状态

```bash
relay status    # 各 worker 的 status / rounds / pane，inbox 与 outbox 积压
```

## 注意

- worker 停下后不一定真的需要决策；如果它只是正常收尾，用 `done`。
- 同一个 worker 在你回复前不会重复派发（in-flight 串行）。
- 超过 `max_rounds` 或你 `timeout` 内没回复，daemon 会自动升级给人。
