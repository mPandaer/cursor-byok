# 交接：Cursor 3.29.9 协议升级（2026-11-07）

## 一句话状态

协议升级与 `data_binary` 修复已完成并通过全部服务端测试。以 DeepSeek 模型实测，BidiAppend 解码、本地路由、BYOK 上游、Run 完成全链路通过；`gpt-6.1-sol` 会话此前的 `AgentRunRequest.conversation_state: ... delimited length exceeded` 未再复现，仅剩一个待用失败样本定位的解码错误（见文末复测记录）。

## 仓库状态

- 分支：`cursor-3.29.9-protocol`（从 `main` 的 `f78856c` 切出）
- 以下改动已提交在该分支的首个提交中：

```text
M protocols/cursor/agent_v1.proto
M protocols/cursor/aiserver_v1.proto
M server/src/api/cursor/bidi.rs
M server/src/cursor/checkpoint/turns.rs
M server/tests/prefix_stability.rs
A docs/cursor-protocol-3.29.9.md
A docs/handoff-2026-11-07-cursor-3.29.9.md
```

## 已完成的工作

### 1. 直接从本机 Cursor Bundle 分析协议

来源：Cursor **3.29.9**，distro `d5c0e77a0214208f36b56d42e8e787de88d02ea4`。
Bundle hash 与分析方法记录在 [cursor-protocol-3.29.9.md](cursor-protocol-3.29.9.md)。

没有采用项目 Go 提取器的输出作为协议来源；该提取器对 3.29.9 的结果存在未解析类型和字段编号冲突。

### 2. 同步 agent.v1 确认变化

见 [agent_v1.proto](../protocols/cursor/agent_v1.proto)：

```text
AgentConversationTurnStructure   新增字段 7-16
AgentRunRequest                  删除 10、27，新增 28-37
PrewarmRequest                   删除 10、25，新增 26-30
ConversationStateStructure       新增 34-40、42-47、49-50
TtftBreakdown                    新增 6-8
AgentMode                        删除 AGENT_MODE_PROJECT = 6
```

新增引用类型：`SubagentDispatchStep`、`SubagentDispatchTool`、`LongReasoningReminderState`、
`SystemPromptSpec`、`ClientLlmGatewayCredential`、`VoiceCallNotes`、`TurnTelemetryState`、
`VoiceCallUtterance`、`VoiceCallPendingAnswer`。

### 3. Bidi `data_binary` 支持

[server/src/api/cursor/bidi.rs](../server/src/api/cursor/bidi.rs#L156-L183)：

```text
data 非空 → hex 解码
data_binary 非空 → 直接 protobuf 解码
两者都非空 → 明确拒绝
```

新增 3 个单元测试。协议相关测试与全量测试均通过。

### 4. 两个非协议问题的修复

- [turns.rs](../server/src/cursor/checkpoint/turns.rs#L92-L100)：新增字段导致的初始化缺字段，补 `..Default::default()`。
- [prefix_stability.rs](../server/tests/prefix_stability.rs#L372-L387)：Windows CRLF 下硬编码 `\n` 断言失败，改为按行断言。

## 未解决的问题（当前阻塞点）

重启后的日志：

```text
400 Bad Request
protobuf decode error: failed to decode Protobuf message:
AgentRunRequest.conversation_state:
AgentClientMessage.message:
delimited length exceeded
```

这意味着：`data_binary` 已进入 `AgentClientMessage::decode`，失败发生在
`AgentRunRequest` 的 `conversation_state` 嵌套解析。未知字段不会触发该错误，
因此优先怀疑嵌套消息的字段编号/类型/承载关系与服务端 Proto 不一致。

**尚未确认具体字段。不要在没有 wire 证据的情况下继续猜改 Proto。**

## 下一步（按顺序）

1. 取得失败请求的脱敏原始字节

```text
优先级：BidiAppendRequest.data_binary 的 hex
位置：本地 trace / capture 产物，或 support/cursor-capture
要求：脱敏，不含用户消息、路径、凭据
```

2. 逐层解析，定位出错字段编号

```text
AgentClientMessage
  → run_request (field 1)
  → conversation_state (field 1)
  → 失败的嵌套消息
```

可对照本机 Bundles：

```text
C:\Users\pandaer\AppData\Local\Programs\cursor\resources\app\out\vs\workbench\workbench.desktop.main.js
C:\Users\pandaer\AppData\Local\Programs\cursor\resources\app\extensions\cursor-always-local\dist\main.js
```

3. 只修改被证据证实的定义，然后：

```text
cargo check
cargo test --lib --tests
重启 Tauri，实际发送一条 Cursor 消息
```

## 常用命令

Cargo 由 Scoop 安装，不在 PATH：

```powershell
$cargo = 'C:\Users\pandaer\scoop\persist\rustup-msvc\.rustup\toolchains\stable-x86_64-pc-windows-msvc\bin\cargo.exe'
& $cargo check --manifest-path 'D:\projects\cursor-byok\server\Cargo.toml'
& $cargo test  --manifest-path 'D:\projects\cursor-byok\server\Cargo.toml' --lib --tests
```

启动 Tauri 开发环境：

```powershell
$env:Path = "C:\Users\pandaer\scoop\persist\rustup-msvc\.rustup\toolchains\stable-x86_64-pc-windows-msvc\bin;" + $env:Path
npm --prefix "D:\projects\cursor-byok\apps\desktop" run tauri:dev
```

健康检查：`http://127.0.0.1:52541/__byok-api__/healthz` 应返回 204。

## 已知干扰项

- Cursor 上游多次返回 `521`（广告、Cpp、FileSync 服务），与本次解码错误无关。
- Tauri 启动时有 `cursor_byok_desktop.pdb` 输出文件名冲突警告，不影响运行。
- Tauri 开发进程此前因后台超时被终止；接手时需重新启动。

## 复测记录（2026-11-07 傍晚）

在 Tauri 开发环境用 DeepSeek 模型（`30f29e67822f33b0`）完成实际发送：

- `BidiAppendRequest.data_binary` 路径解码成功，同一 request 的 seqno 0–10 全部通过；
- `Run ended outcome=Completed`，带 input/output/cache token usage；
- 最后一次 `delimited length exceeded` 停在 16:15，此后未再出现。

解码错误机制已用源码与合成 wire 验证：

- prost 0.13.5 的 `delimited length exceeded` 只在嵌套 `merge_loop` 中发现某个 length-delimited 字段越过外层边界时产生；
- 合成样本确认该错误串对应“`ConversationStateStructure` 内标量字段长度越界”，而嵌套消息不匹配会带更深一层字段上下文，截断则是 `buffer underflow`；
- 逐字段比对 3.29.9 三个 Bundle（workbench / cursor-always-local / cursor-agent-host）与 `protocols/cursor/agent_v1.proto`：`ConversationStateStructure` 1–50、`AgentRunRequest` 1–37 的编号与类型完全一致，排除仓库 Proto 写错字段编号/类型。

剩余排查需要失败请求的原始字节：复现时用环境变量门控转储失败的 `BidiAppend` 负载，再用无 schema 的 wire 扫描器定位越过 `conversation_state` 边界的字段编号与偏移。临时转储代码与探针测试在复测后已删除。
