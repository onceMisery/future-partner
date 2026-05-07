# mcp-bridge-semantics.md

MCP Bridge 语义映射规范。MCP Bridge 是 BridgePlugin 的一种参考实现。

## 1. 定位

BridgePlugin 把异构协议翻译为 FAP。MCP Bridge 只是其中一例；A2A Bridge、VCP Bridge、私有协议 Bridge 皆可并存。

```text
所有 Bridge 都是插件:
  - 不修改 Core
  - 不绕过 Auth / Risk / Receipt / Idempotency
  - 通过标准 BridgePlugin 接口注册
```

## 2. 消息映射

| MCP | FAP |
|---|---|
| `initialize` | ClientHello / ServerHello |
| `initialized` notification | （并入 ServerHello ACK 语义） |
| `tools/list` | CapabilityBind + advertised_capabilities |
| `tools/call` | InvokeRequest |
| `resources/list` | CapabilityBind (resource.* 能力) |
| `resources/read` | InvokeRequest + ObjectRef |
| `prompts/list` | CapabilityBind (prompt.* 能力) |
| `prompts/get` | InvokeRequest |
| `completion/complete` | InvokeRequest (risk=medium) |
| `sampling/createMessage` | 反向调用（见第 5 节） |
| `elicitation/create` | 反向调用（见第 5 节） |
| `notifications/progress` | ProgressEvent |
| `notifications/message` | ProgressEvent (partial result) |
| `logging/setLevel` | options 传入 InvokeRequest |
| error | Problem |

## 3. JSON Schema ↔ google.protobuf.Struct

MCP Tool 定义用 JSON Schema draft 2020-12；FAP Capability 用 Struct。Bridge 必须提供**无损双向转换**：

```text
JSON Schema → Struct:
  顶层 object → Struct
  嵌套 object → 嵌套 Struct
  array → ListValue
  string/number/boolean/null → Value
  $ref → 展平为内联 schema（推荐）或保留 $ref 字符串
  required/properties/additionalProperties → 保留
  format/pattern/min/max → 保留

Struct → JSON Schema:
  反向操作，保留原始结构
```

实现必须通过以下测试：

```text
∀ schema ∈ MCP_official_examples:
  schema == json_schema_from_struct(struct_from_json_schema(schema))
```

## 4. Progress Token 映射

MCP `progress_token` ↔ FAP `message_id` / `trace_id`：

```text
Bridge 维护映射表:
  { progress_token → fap_invocation_id }
  
MCP 客户端带 _meta.progressToken = "p1" 调用 tools/call
Bridge 生成 invocation_id = "inv-abc"
Bridge 收到 FAP ProgressEvent(invocation_id="inv-abc")
Bridge 发出 MCP notifications/progress(progressToken="p1")
```

映射表 TTL 与 invocation 生命周期一致。

## 5. 反向调用（MCP-full）

MCP 允许 Server 反向调用 Client（`sampling/createMessage`、`elicitation/create`）。FAP 原生不反向；Bridge 需桥接：

```text
方案 A（推荐）: MCP-compatible-subset
  Bridge 不支持反向，声明 capability mode = "compatible-subset"
  收到反向请求返回 MCP error -32601 Method Not Found

方案 B: MCP-full
  Bridge 维护反向通道:
    FAP Server 需通过 BridgePlugin API 发起 "inverse call"
    Bridge 转为 MCP sampling/createMessage
    响应回写为 FAP InvokeResult (反向)
  必须开启 mandate，且 Client 显式允许反向
```

Agent Card 声明支持级别：

```json
"bridges": [
  { "type": "mcp", "mode": "compatible-subset" }
]
```

## 6. 安全约束

```text
Bridge 翻译时必须:
  1. 为每个 MCP 请求生成新的 FAP Envelope（含 message_id / idempotency_key）
  2. 传入 Auth 上下文（MCP transport 的 Bearer Token → FAP JWT 或 Mandate）
  3. 经 Core 安全链路（不得内部短路）
  4. 生成 Receipt（Bridge 是 EXECUTOR_NODE 之一）

Bridge 不得:
  绕过 Risk Policy
  缓存 mandate 状态跨请求
  代 MCP Client 签 FAP 签名（必须 Client 自签或 Bridge 自己作为 subject）
```

## 7. A2A Bridge

| A2A | FAP |
|---|---|
| Agent Card | Agent Card (字段转换) |
| Skill | Capability |
| Task | Invocation |
| Artifact | ObjectRef |
| Message | InvokeRequest / ProgressEvent |
| Streaming | SSE / QUIC stream |

A2A Bridge 规则等同 MCP Bridge。

## 8. 其它协议

任何第三方 Bridge 应提供：

```text
docs/<protocol>-bridge-semantics.md
conformance/bridges/<protocol>/
  fixtures/   请求-响应对
  tests/      单测
  matrix.md   支持级别矩阵
```

## 9. 版本兼容

```text
MCP 协议版本变更:
  Bridge 自身独立 SemVer
  Bridge 声明 supported_mcp_versions
  不影响 FAP Core 版本
```
