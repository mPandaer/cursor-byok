# Cursor 协议升级方法

> 本文是一份通用的 Cursor 私有协议升级与维护方法。它描述协议变化发生后，如何获取证据、更新协议来源、同步实现、验证运行时行为并留下可追溯记录。
>
> 本文是流程与边界说明，不是某一次故障的修复方案；阅读本文不会预设具体的字段、消息、版本或错误。

## 目标

Cursor 私有协议不是稳定的公共 API。Cursor 更新后，客户端可能改变：

- protobuf 消息的字段、编号或嵌套关系；
- Connect framing、压缩方式或内容类型；
- 请求与响应的服务路径；
- 消息在字符串、原始字节或其他 envelope 中的承载方式；
- Agent 消息的 oneof 分支、模型标识或会话字段；
- 上行消息的顺序、重试和生命周期语义。

升级工作的目标不是让代码“尽量解析成功”，而是建立一条可审计的证据链：

```text
Cursor 版本 / Bundle
        │
        ▼
Proto 来源与 wire schema
        │
        ▼
真实请求 / 响应 / Connect frame
        │
        ▼
规范化协议模型
        │
        ▼
Rust 入口、路由、Runtime 与上游适配
        │
        ▼
Wire 测试、运行时测试、升级记录
```

完成升级后，代理必须清楚知道：

1. 它接收的外层消息是什么；
2. 如何解码和验证消息；
3. 如何判断本地处理还是转发官方服务；
4. 如何保持消息顺序、请求生命周期和错误语义；
5. 哪些旧客户端继续支持，哪些旧路径被移除。

## 维护原则

### 1. Proto 来源唯一

`protocols/cursor/` 是仓库内 Cursor Proto 定义的唯一来源。提取、人工修订或生成过程都必须最终回写到这里，不能在多个模块维护互相漂移的协议副本。

```text
Cursor 安装包 / JS Bundle
        │
        ▼
support/cursor-protocol-extractor/
        │
        ▼
protocols/cursor/
        │
        ├── agent_v1.proto
        ├── aiserver_v1.proto
        └── 其他服务 Proto
```

### 2. 证据先于实现

协议变化先用当前 Cursor 的 Bundle 和真实流量确认，再修改 Rust。代码中的类型名称、字段名称或旧版本经验不能替代 wire 证据。

### 3. Schema 与语义分开验证

protobuf 能够解码，只能证明字段编号和基础类型暂时匹配，不能证明业务语义匹配。每次升级都要分别验证：

```text
wire schema       # 字段编号、类型、嵌套、oneof
transport         # Connect frame、压缩、HTTP headers
message meaning   # 模型、会话、顺序、生命周期
runtime behavior  # 路由、执行、输出、错误和重试
```

### 4. 兼容范围明确

每次升级必须明确支持范围：

- 只支持当前 Cursor；
- 支持当前版本和一个旧版本；
- 支持多个已验证版本；
- 放弃旧协议并删除旧路径。

不要因为“可能有旧用户”而无限增加运行时分支。兼容策略应由真实需求和测试证据决定。

### 5. 失败要可诊断

未知字段、无法解码、未知 oneof、错误顺序和不支持的服务都应在协议边界产生结构化错误。错误应包含服务路径、阶段和非敏感的消息类别，但不得记录 API Key、Cookie、Authorization、用户内容或完整敏感请求体。

## 当前目录与职责

```text
cursor-byok/
├── protocols/cursor/                         # Cursor Proto 定义的唯一来源
├── support/cursor-protocol-extractor/        # 从 Cursor JS Bundle 提取 Proto
│   ├── scripts/extract.sh                    # 定位 Bundle 并运行提取流程
│   ├── scripts/generate.sh                   # 生成调试工具使用的 Go 类型
│   └── extractor/                            # Proto 提取、解析和渲染
├── support/cursor-capture/                   # 捕获、解压和查看真实流量
│   ├── capture_pipeline.go                   # 捕获请求、响应和 Connect frame
│   └── decode.go                             # 解码 protobuf 与消息视图
├── server/build.rs                           # 生成类型并校验协议来源
├── server/src/cursor/protocol/               # Connect、protobuf、事件和 JSON 流
│   ├── connect.rs                            # Connect framing 与压缩
│   ├── proto.rs                              # 手写或生成的协议类型
│   └── events.rs                             # 下行事件编码
├── server/src/api/cursor/                    # Cursor HTTP/Connect 入口
│   ├── handlers.rs                           # 服务路由和请求入口
│   ├── bidi.rs                               # 双向消息解码和投递
│   ├── run_sse.rs                            # 下行流订阅
│   └── proxy.rs                              # 官方服务转发
├── server/src/cursor/                        # 会话、Agent Runtime 和服务适配
└── server/tests/connect_wire.rs              # Wire 与 Connect 层测试
```

目录责任边界：

```text
抓取工具       负责发现事实和生成可审查的消息视图
protocols/     负责保存协议定义
protocol/      负责传输和消息编解码
api/cursor/    负责入口校验、路由和错误边界
cursor/        负责会话、Agent 生命周期和业务语义
provider/      负责上游模型协议
store/         负责持久化，不负责解释 Cursor wire
```

## 完整升级流程

### 阶段 0：记录升级触发点

先记录：

- Cursor 版本、平台和架构；
- 项目当前版本和 Git 提交；
- 受影响的服务路径；
- 首个失败阶段和外部表现；
- 最近一次已知正常的 Cursor 版本；
- 是否存在可用的旧版流量或测试 fixture。

不要把推测写成结论。此阶段只建立比较范围。

**完成标准：** 能用一条时间线说明“哪个 Cursor 版本、哪个项目版本、哪个协议入口发生了变化”。

### 阶段 1：提取当前协议来源

使用 `support/cursor-protocol-extractor/` 从当前 Cursor 安装目录定位并提取 JS Bundle。通常需要检查：

```text
workbench.desktop.main.js
extensionHostProcess.js
extensions/*/dist/main.js
```

提取时：

1. 保留原始 Bundle 的版本和来源记录；
2. 将 Bundle 复制到隔离的临时目录，不直接修改安装目录；
3. 将输出写入临时协议目录；
4. 对比临时输出与 `protocols/cursor/`；
5. 只把经过审查的变化合并到唯一协议来源。

工具和输入都应与项目代码分离。不要执行来源不明的脚本，也不要把完整 Cursor Bundle 或包含凭据的文件提交到仓库。

**完成标准：** 获得带来源版本信息的临时 Proto 输出，并生成一份可审查的 schema diff。

### 阶段 2：捕获真实 wire 流量

使用 `support/cursor-capture/` 捕获最小、可重复的用户动作。优先覆盖一条完整生命周期：

```text
建立连接
  → 请求配置 / 能力
  → 选择模型
  → 发送一条最小用户消息
  → 接收流式响应
  → 完成或取消
```

捕获工具需要分层保存信息：

```text
HTTP metadata
  → method / path / status / content type / content encoding
Connect frame
  → flags / length / compression / end-of-stream
protobuf
  → message type / known fields / oneof branch
semantic view
  → request id / conversation id / model id / sequence
```

保存前必须脱敏：

- Authorization、Cookie、API Key、证书和用户身份；
- 用户消息、代码内容、文件路径和 provider 请求体；
- 其他可关联真实用户或工作区的数据。

原始流量只在必要的隔离目录中短期保存；提交到仓库的 fixture 必须最小化并可审查。

**完成标准：** 有一组与 Proto diff 对应的脱敏流量样本，能够展示升级前后每个受影响服务的外层 framing、消息类型和关键语义字段。

### 阶段 3：建立 schema diff 和语义 diff

不要只比较生成文件。按以下维度审查：

| 维度 | 需要确认的内容 |
| --- | --- |
| 服务 | RPC 路径、HTTP 方法、请求/响应类型 |
| 字段 | 编号、基础类型、可选性、重复性、默认值 |
| 嵌套 | message、enum、map、oneof、any 或 envelope |
| 传输 | content type、压缩、Connect frame flags、流结束方式 |
| 关联 | request id、conversation id、append seqno、父请求 |
| 路由 | 本地模型、官方模型、插件模型的识别规则 |
| 生命周期 | 创建、继续、重试、取消、恢复和结束 |
| 错误 | 错误 message、状态码、retryable 和用户可见语义 |

对于字段变化，区分三类：

```text
可向后兼容
  新增未知字段，旧端可以安全忽略

需要条件兼容
  同一语义改变了承载方式、包装层或 optional 行为

破坏性变化
  字段编号复用、消息类型替换、路由或生命周期语义改变
```

字段编号复用、旧字段改类型、oneof 分支重解释都必须视为破坏性变化处理，不能用宽松解析掩盖。

**完成标准：** 每个 schema 变化都有对应的语义影响、兼容范围和测试要求。

### 阶段 4：确定兼容策略

先选择策略，再写分支。可选策略按复杂度递增：

```text
单一当前格式
  → 删除过时路径，保持边界简单

明确的双格式兼容
  → 根据版本、字段或已验证特征选择唯一解码路径

版本化协议适配器
  → 每个已支持版本拥有独立 decoder / encoder

显式协商
  → 只有 Cursor 实际提供稳定版本协商信号时才采用
```

策略必须回答：

1. 如何识别每种格式；
2. 同一请求是否允许多个格式同时出现；
3. 失败后是否允许回退；
4. 回退是否可能重复执行有副作用的请求；
5. 何时删除旧路径；
6. 日志如何区分版本、格式和失败阶段。

对于有副作用的请求，不要通过“先按格式 A 执行，失败后再按格式 B 重试”实现协议兼容。应在执行前完成格式判定。

**完成标准：** 每个受支持输入都有唯一的解析和执行路径；不支持的输入有明确错误；旧路径有退出条件。

### 阶段 5：同步实现边界

按数据流从外到内修改，避免在业务 Runtime 中补协议判断：

```text
1. protocols/cursor/*.proto
2. server/build.rs / 生成类型或手写稳定子集
3. server/src/cursor/protocol/connect.rs
4. server/src/cursor/protocol/proto.rs
5. server/src/api/cursor/handlers.rs
6. server/src/api/cursor/bidi.rs / run_sse.rs
7. server/src/cursor/transport/
8. server/src/cursor/conversation/
9. server/src/provider/（只有上游语义真的变化时）
```

每层只承担自己的问题：

```text
协议层       解码、编码、压缩、frame、大小限制
入口层       必填字段、方法、路径、错误状态
路由层       request id、model、conversation 的归属
Runtime 层   有序投递、生命周期、取消和输出
Provider 层  统一模型请求与上游 API 差异
```

如果完整 Proto 无法直接生成 Rust，保留当前项目的“生成稳定部分、手写必要子集”边界，并在 `build.rs` 中验证关键定义。不要为了单次升级把所有上游内部消息全部引入业务代码。

**完成标准：** 协议变化只在对应边界处理；运行时模块不包含重复的 wire 解析；每个新字段或分支都有明确所有者。

### 阶段 6：分层验证

验证必须从最窄的 wire 层逐层扩大：

```text
Proto 编译与 schema 校验
  → Connect frame fixture
  → 单条 request / response decode
  → 路由与顺序测试
  → Conversation / Run 生命周期测试
  → Provider 调用测试
  → 脱敏真实流量 replay
  → 当前 Cursor 手工验收
```

至少覆盖：

- 最小合法请求；
- 缺少必填关联字段；
- 未知字段；
- 新增字段；
- 无效压缩或 frame；
- 无效消息类型或 oneof；
- 重复、乱序和跳号的 sequence；
- 新建、继续、取消、重试和恢复；
- 本地路由与官方转发；
- 上游错误、超时和流结束；
- 旧版本 fixture（如果兼容范围要求保留）。

**完成标准：** 协议测试证明 wire 正确，运行时测试证明生命周期正确，真实 replay 证明入口到输出的完整链路正确。

### 阶段 7：记录、发布和回滚

提交升级时记录：

```text
Cursor 来源版本：
项目提交：
变化的服务：
变化的消息 / 字段 / 编号：
传输变化：
支持的 Cursor 范围：
删除的旧路径：
测试 fixture：
已知限制：
回滚条件：
```

发布前确认：

- 协议来源和代码变更可以由同一提交追溯；
- 生产日志不会泄露敏感 payload；
- 旧协议支持范围写入发布说明；
- 失败时能区分 schema、transport、routing、runtime 和 provider 阶段；
- 已知不兼容可以快速回滚到上一个协议适配版本。

**完成标准：** 另一个维护者只依靠提交、Proto diff、fixture 和测试即可复现升级判断，不需要重新猜测协议来源。

## 版本适配的推荐结构

当多个 Cursor 版本确实需要同时支持时，优先将差异放在协议边界：

```text
server/src/cursor/protocol/
├── connect.rs                 # 共享 framing 和压缩
├── proto.rs                   # 共享稳定类型
├── adapters/                  # 版本或格式差异
│   ├── current.rs             # 当前已验证格式
│   └── legacy.rs              # 明确仍支持的旧格式
├── detect.rs                  # 无副作用的格式识别
└── mod.rs                     # 对上层暴露统一表示
```

上层只接收统一表示：

```text
Wire adapter
  → DecodedCursorMessage
  → TransportRegistry
  → ConversationRuntime
```

只有在出现真实、稳定、可测试的多版本需求时才增加 `adapters/`。单次字段变化不应自动升级为版本框架。

## 不应采用的做法

| 做法 | 问题 |
| --- | --- |
| 只修改错误判断让请求继续 | 可能把未验证的数据送入错误的 Runtime |
| 通过异常后重试另一种格式 | 有副作用请求可能被执行两次 |
| 在每个业务模块重复解析 wire | 协议知识扩散，升级时容易漏改 |
| 直接完整生成所有内部 Proto | 引入无关类型、命名冲突和不必要耦合 |
| 运行时下载并执行未知 Bundle | 增加供应链风险，破坏协议来源审计 |
| 记录完整请求体帮助排查 | 可能泄露凭据、用户内容和工作区信息 |
| 未经证据保留无限旧版本分支 | 增加测试矩阵和运行时复杂度，没有退出条件 |
| 以 protobuf 能解码作为升级完成 | schema 成功不代表语义、路由和生命周期正确 |

## 升级检查清单

### 证据

- [ ] 已记录 Cursor 来源版本、平台和架构。
- [ ] 已取得当前 Bundle 并保留来源信息。
- [ ] 已生成临时 Proto 并完成 schema diff。
- [ ] 已捕获最小完整生命周期流量。
- [ ] 已对 fixture 和日志完成脱敏。

### 协议

- [ ] 服务路径、方法和请求/响应类型已确认。
- [ ] 字段编号、类型、optional/repeated、oneof 已确认。
- [ ] Connect frame、压缩和 content type 已确认。
- [ ] request id、conversation id、model id 和顺序语义已确认。
- [ ] 错误、取消、重试和流结束语义已确认。

### 实现

- [ ] `protocols/cursor/` 是唯一协议来源。
- [ ] 生成类型或手写子集与 Proto diff 一致。
- [ ] wire 解析集中在协议边界。
- [ ] 路由、Runtime 和 Provider 的所有者边界清楚。
- [ ] 兼容范围和旧路径退出条件已写明。

### 验证

- [ ] schema / build 校验通过。
- [ ] Connect 和 protobuf fixture 测试通过。
- [ ] 路由、顺序和生命周期测试通过。
- [ ] 真实流量 replay 通过。
- [ ] 当前 Cursor 手工验收通过。
- [ ] 日志、错误和回滚信息可用且不泄密。

## 参考入口

- [Cursor Proto 定义](../protocols/cursor/)
- [协议提取脚本](../support/cursor-protocol-extractor/scripts/extract.sh)
- [协议生成脚本](../support/cursor-protocol-extractor/scripts/generate.sh)
- [流量捕获解码器](../support/cursor-capture/decode.go)
- [Connect 协议实现](../server/src/cursor/protocol/connect.rs)
- [Cursor wire 类型](../server/src/cursor/protocol/proto.rs)
- [Cursor API 入口](../server/src/api/cursor/handlers.rs)
- [Wire 测试](../server/tests/connect_wire.rs)

本文的完成条件是：它能够指导一次新的 Cursor 协议变化调查，而不依赖某个具体错误、字段或版本；每个阶段都有输入、边界、输出和可检查的完成标准。
