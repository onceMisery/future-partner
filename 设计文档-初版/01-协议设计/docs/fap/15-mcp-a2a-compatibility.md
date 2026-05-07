# 15 - MCP / A2A 兼容

详见 [mcp-bridge-semantics.md](./mcp-bridge-semantics.md)。

## 1. BridgePlugin

MCP Bridge / A2A Bridge / VCP Bridge 都是 BridgePlugin 的实现。

```text
所有 Bridge 都是插件:
  - 不修改 Core
  - 不绕过 Auth / Risk / Receipt / Idempotency
  - 通过标准 BridgePlugin 接口注册
```

## 2. MCP 映射

| MCP | FAP |
|---|---|
| initialize | ClientHello / ServerHello |
| tools/list | CapabilityBind + advertised_capabilities |
| tools/call | InvokeRequest |
| resources/read | InvokeRequest + ObjectRef |
| notifications/progress | ProgressEvent |
| error | Problem |

## 3. A2A 映射

| A2A | FAP |
|---|---|
| Agent Card | Agent Card |
| Skill | Capability |
| Task | Invocation |
| Artifact | ObjectRef |
| Message | InvokeRequest / ProgressEvent |

## 4. JSON Schema ↔ Struct

Bridge 必须提供无损双向转换：

```text
∀ schema ∈ MCP_official_examples:
  schema == json_schema_from_struct(struct_from_json_schema(schema))
```

## 5. 反向调用

MCP-full 支持反向调用（`sampling/createMessage`），需 Bridge 维护反向通道 + mandate 显式允许。
MCP-compatible-subset 不支持反向，返回 MCP error -32601。
