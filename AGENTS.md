# AGENTS.md

本仓库是 **agent-relay**：多 agent tmux 编排与人工升级中继。
单文件 Python（`bin/relay`，约 1000 行），无第三方依赖，无测试框架。

## 代码结构

`bin/relay` 按职责分段，每段开头有注释说明设计理由：

```
基础设施（配置/状态读写、日志）
tmux（pane 解析、bracketed paste）
通知（飞书 / SMTP）
邮件回灌（IMAP 轮询、In-Reply-To 校验）
hook 入口（cmd_event）
orchestrator / 人的指令（enqueue、cmd_answer、cmd_context）
daemon（dispatch / apply_outbox / check_timeout / check_pending）
运维命令（init / orch / register / status）
```

## 六条不可破坏的约束

### 1. pane 用稳定目标，不要存 `%N`

`%N` 是 tmux 的运行时编号，**server 重启后全部重排**。配置里存 `%3`，重启后它
要么不存在、要么指向别的窗口。真实事故：daemon 每秒对着一个不存在的 `%3`
重试，14 次之后被人发现，worker 的事件一直派不出去。

所以 `orchestrator_pane` / `worker_panes` 里存的是 `session:window.pane`，
用的时候才经 `resolve_pane()` 解析成当前 id。新增任何「记住某个 pane」的地方
都要走这两个函数，不要直接存 `os.environ["TMUX_PANE"]`。

注意 `resolve_pane()` 只保证「这个 target 现在指向某个活着的 pane」。tmux 对
越界索引会**静默钳制**（`sess:1.9` → 该窗口最后一个 pane），所以它够用作白名单
判定，不够用作身份识别。

### 2. `relay event` 永远退出 0，且不能阻塞

它跑在 worker 的 Stop hook 路径上。任何异常、任何阻塞都会卡住被 hook 的 agent。
`cmd_event` 整体包在 try/except 里，stdin 读取有 3 秒 select 超时——这两层都不能破坏。

### 3. daemon 是 `state.json` 的唯一写者

`relay decide/done/escalate/answer` 都只写 `outbox/`，由 daemon 统一 apply。
不要在 CLI 侧直接改 state。

### 4. `dangerous_regex` 只放不可逆操作

「部署」「删除」「付款」这类动词同样出现在「禁止部署」「不要删除」这类否定句里，
正则分不出来，会把合法决策静默拦截。

真实事故：orchestrator 写「禁止部署、生产写入」，命中 `部署.*生产`，整条决策被拦，
worker 永远等不到回复，只留一行本地日志。这类词放 `caution_regex`，只标注不拦截。

### 5. 未知 worker 的 `decide` 必须拒绝

曾用 in-flight 的 pane 兜底，结果 orchestrator 打错 worker 名时，决策被投进了另一个
worker 的 pane。误路由比失败更糟——失败至少有日志。

### 6. 过滤掉的不是「不需要决策」，而是「还没到决策的时机」

`background_tasks` 非空 = 本轮思考结束但任务在跑。这类事件进 `pending/`，
靠 `pending_slack` 兜底补派。

**兜底不能去掉**：如果 worker 的后台任务超时退出而它没恢复，没有兜底就是死锁。

## 改 prompt 要谨慎

派发消息、决策回灌、升级通知的文本现在都在 `TEMPLATES`（`bin/relay` 顶部），
可以用 `relay templates export` 导出成文件直接改，daemon 下一轮 tick 生效、无需重启。

`dispatch` 模板生成的 `[RELAY ...]` 文本直接决定 orchestrator 的行为。踩过的坑：

- 写「不要解释、不要做别的操作」→ 抑制推理，orchestrator 给出浅层决策。
- 写「信息不足就 escalate」+ 问「现有信息够不够」→ worker 的输出只截末尾一段，
  orchestrator 永远信息不足，于是每轮都升级给人。
- 把「部署」「生产」列为升级条件 → 部署任务本身就是生产变更，变成全量升级。

现在改成**正面列举三种该升级的情况**，并显式点名不该升级的借口。

模板用的占位符是 `{name}`。模板里想写裸花括号（JSON 示例等）要用双花括号 `{{` `}}`。
模板缺占位符、格式错误或为空时会**回落内置默认并写日志**——派发不会因此中断，
但你会看到一条 `模板 xxx ... 使用内置默认`。改完模板记得看这条有没有出现。

`relay orch` / `relay register` 会把 `role_orch` / `role_worker` paste 到对应 pane，
这是 agent 得知自己角色与 `[RELAY ...]` 协议的**唯一动态来源**。不要在
codex 全局 `developer_instructions`、README 或源码里另写一份协议正文——
那份只留启动期最小注入并指向模板，两处各写一份必然漂移。

## 测试

历史验证方式是手工 + 隔离 HOME：

```bash
mkdir -p /tmp/t/.agent-relay
HOME=/tmp/t TMUX_PANE=%N relay init
HOME=/tmp/t TMUX_PANE=%N relay event --src claude <<'EOF'
{"last_assistant_message":"测试输出","background_tasks":[]}
EOF
HOME=/tmp/t relay status
```

**状态类测试必须 `export HOME=/tmp/xxx`。** 曾因未隔离，把假 worker 写进真实的
`~/.agent-relay/state.json`，还往一个正在跑生产部署的 worker 投了错误决策。

需要调用内部函数时：`cp bin/relay /tmp/t/relay_mod.py`，再 `sys.path.insert` 后 import。
直接 `spec_from_file_location` 会失败（无 `.py` 后缀）。

## 提交前

```bash
python3 -c "import ast;ast.parse(open('bin/relay').read())"
```

并确认没有把凭据写进代码或文档——搜一下 `password`、`@qq.com` 这类字面量，
应该只在 `config.example.json` 里以占位符形式出现。
`~/.agent-relay/config.json` 含 SMTP 授权码，已在 `.gitignore`，永远不要入库。

## 验证标准

改完逻辑后，至少跑一次真实的 worker → orchestrator → 回灌闭环。
隔离测试只证明分支逻辑正确，不证明 paste 到真实 TUI、真实模型决策、
真实 tmux pane 生命周期都没问题。
