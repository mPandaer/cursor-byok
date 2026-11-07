# Cursor 3.29.9 协议分析记录

## 来源

- Cursor 版本：3.29.9
- distro：`d5c0e77a0214208f36b56d42e8e787de88d02ea4`
- 平台：Windows x64
- workbench：`5aec61924cced4345b0a5c67c741bf56aa6358f6b791cb71129be0de08e2800f`
- cursor-always-local：`d1c174d4ec5df58a7f2ac6e7670b6405e341fed1f9d20e0294e7a43bd96e2090`
- cursor-agent-host：`819cd89e7f3a609c18a0e0f304695c24af285706efb17320c1c50e343622fb5d`
- cursor-agent-worker：`cb046b78afc0819643e4a0fefd32324d0481eee73b20389d42f55928f664d828`

## 分析方法

直接读取 Bundle 中的 protobuf-es 描述：

- `makeMessageType("...", () => [{ no, name, kind, T, ... }])`
- `makeEnum("...", [{ no, name, ... }])`
- extension bundle 中的 `static $()` compact descriptor

不使用当前 Go 提取器生成的候选文件作为协议来源。当前 Go 提取器对本版本的候选结果存在未解析类型别名、缺失声明和字段编号冲突，因此未覆盖仓库协议。

## 已确认的 aiserver wire 定义

以下定义与仓库当前手写 wire 子集一致：

- `BidiRequestId`
- `BidiAppendRequest`
- `BidiAppendResponse`
- `WriteGitCommitMessageRequest`
- `WriteGitCommitMessageResponse`
- `ExplicitContext`
- `CustomErrorDetails`
- `ErrorDetails`

`CustomErrorDetails` 保持字段 1-6，并确认当前 Bundle 还包含字段 7-10：`additional_info`、`buttons`、`plan_choices`、`analytics_metadata`。当前服务端只消费字段 1-6，未知字段由 protobuf 解码器跳过。

`ErrorDetails.Error` 在 3.29.9 增加 `ERROR_TENANT_RESTRICTION_VIOLATION = 66`。当前服务端只产生 `PROVIDER_ERROR = 57`，因此没有加入手写 Rust 子集。

## 已确认的 agent.v1 变化

当前 Bundle 确认了以下字段变化：

- `AgentConversationTurnStructure` 增加字段 7-16；
- `AgentRunRequest` 删除字段 10、27，并增加字段 28-37；
- `PrewarmRequest` 删除字段 10、25，并增加字段 26-30；
- `ConversationStateStructure` 增加字段 34-40、42-47、49-50；
- `TtftBreakdown` 增加字段 6-8；
- `AgentMode` 删除值 `AGENT_MODE_PROJECT = 6`，保留 0、1、2、3、4、5、7、8。

本次已将上述确认的消息字段和枚举变化同步到 `protocols/cursor/agent_v1.proto`。未补齐与当前服务端无关的所有新消息定义；新增字段使用的引用类型已同步加入：

- `SubagentDispatchStep`
- `SubagentDispatchTool`
- `LongReasoningReminderState`
- `SystemPromptSpec`
- `ClientLlmGatewayCredential`
- `VoiceCallNotes`
- `TurnTelemetryState`
- `VoiceCallUtterance`
- `VoiceCallPendingAnswer`

## 验证状态

- 协议 Bundle 直接分析：完成。
- Go 提取器自身测试：通过，`go test ./...`。
- Rust 测试：未运行，当前环境 PATH 中没有 `cargo`。
- 真实 Cursor wire replay：未完成。
- 完整 Rust 编译：未完成。

因此本次结果是“基于 Bundle 的协议增量分析”，不是发布就绪的兼容性证明。下一步应安装或配置 Rust 工具链，运行 `cargo check` / `cargo test`，并用当前 Cursor 执行最小 Agent、Bidi、继续对话和取消流程验证。
