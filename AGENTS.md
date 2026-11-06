# AGENTS.md — 在这个仓库里工作时

本仓库是 relay：多 agent tmux 编排与人工升级中继。单文件 Python（`bin/relay`），无第三方依赖。

## 先读

- `README.md`：架构、安装、配置、防护机制、已知限制。
- `bin/relay`：约 1000 行，结构是「基础设施 → tmux → 通知 → 邮件回灌 → hook 入口 →
  orchestrator 指令 → daemon → 运维命令」，每段有注释说明设计理由。

## 改代码前必须知道的约束

### 1. `relay event` 永远退出 0，且不能阻塞

它跑在 worker 的 Stop hook 路径上。任何异常、任何阻塞都会卡住被 hook 的 agent。
`cmd_event` 整体包在 try/except 里，stdin 读取有 3 秒 select 超时——改这里时不要破坏这两层。

### 2. daemon 是 `state.json` 的唯一写者

`relay decide/done/escalate/answer` 都只写 `outbox/`，由 daemon 统一 apply。
不要在 CLI 侧直接改 state，会引入并发写。

### 3. `dangerous_regex` 只放不可逆操作

不要把「部署」「删除」「付款」这类动词放进去。它们同样出现在「禁止部署」「不要删除」这类
**否定句**里，正则分不出来，会把合法决策静默拦截。这类词放 `caution_regex`，只做标注。

真实的线上事故：orchestrator 在决策里写「禁止部署、生产写入」，命中 `部署.*生产`，
整条决策被拦，worker 永远等不到回复，只有本地日志留痕。

### 4. 未知 worker 的 `decide` 必须拒绝

曾经用 in-flight 的 pane 兜底，结果 orchestrator 打错 worker 名时，决策被投进了另一个
worker 的 pane。误路由比失败更糟——失败至少有日志。

### 5. 过滤掉的不是「不需要决策」，而是「还没到决策的时机」

`background_tasks` 非空 = 本轮思考结束但任务在跑。这类事件进 `pending/`，靠
`pending_slack` 兜底补派。**兜底不能去掉**：如果 worker 的后台任务超时退出而它没恢复，
没有兜底就是死锁。

## 测试

没有测试框架，历史验证方式是手工 + 隔离 HOME：

```bash
mkdir -p /tmp/t/.agent-relay
HOME=/tmp/t TMUX_PANE=%N relay init
HOME=/tmp/t TMUX_PANE=%N relay event --src claude <<'EOF'
{"last_assistant_message":"测试输出","background_tasks":[]}
EOF
HOME=/tmp/t relay status
```

**状态类测试必须 `export HOME=/tmp/xxx`。** 曾经因为没隔离，把假 worker 写进了真实的
`~/.agent-relay/state.json`，还往一个正在跑生产部署的 worker 里投了一条错误决策。

需要调用内部函数时：`cp bin/relay /tmp/t/relay_mod.py`，然后 `sys.path.insert` 后 import。
直接 `spec_from_file_location` 会失败（无 `.py` 后缀）。

## 提交前

```bash
python3 -c "import ast;ast.parse(open('bin/relay').read())"
```

并且确认没有把凭据写进代码或文档：用 `git grep` 搜一下 `password`、SMTP 授权码、
`@qq.com` 这类字面量，应该只在 `config.example.json` 里以占位符形式出现。
`~/.agent-relay/config.json` 含 SMTP 授权码，已在 `.gitignore` 里，永远不要入库。

## 已完成但未在真实 worker 上验证的改动

改完逻辑后，需要至少一次真实的 worker→orchestrator→回灌闭环才算验证通过。隔离测试只证明
分支逻辑正确，不证明 paste 到真实 TUI、真实模型决策、真实 tmux pane 生命周期都没问题。
