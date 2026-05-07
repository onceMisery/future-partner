# 09 - 插件运行时

> 相关专题：[plugin-lifecycle.md](./plugin-lifecycle.md) 发布模型与版本并存；[kernel-contract.md](./kernel-contract.md) 插件边界。

## 1. 插件类型

插件运行时按少数高层类别组织，但安全、存储、审计等能力继续暴露为更细的 Provider 子接口。细粒度边界以 [kernel-contract.md](./kernel-contract.md) 为准。

| 高层类别 | 职责 | 常见子接口 |
|---|---|
| CapabilityPlugin | 业务能力（工具） | CapabilityPlugin |
| TransportProvider | 传输适配 | TransportPlugin |
| SecurityProvider | 认证、签名、密钥、撤销 | AuthPlugin / SignaturePlugin / KeyResolverPlugin / RevocationPlugin |
| StateProvider | 持久化、对象、索引、审计外送 | StoragePlugin / ObjectStorePlugin / VectorIndexPlugin / AuditSinkPlugin |
| ContextProvider | 上下文构建、路由建议、节点健康评分 | ContextPlugin / CapabilityRouterPlugin / NodeHealthPlugin |
| BridgePlugin | MCP/A2A/旧协议桥接 | BridgePlugin |

任何插件可声明 `experimental = true`。

## 2. 生命周期

发布状态机：

```text
UPLOADED → VALIDATED → STAGED → DRY_RUN_PASSED → ACTIVE
  → DRAINING → STOPPED → UNLOADED
```

细节见 [plugin-lifecycle.md](./plugin-lifecycle.md)。

升级期间 `capability_id + schema_version` 可并存，路由按版本约束选择。

## 3. Manifest

```toml
id = "fap-plugin-code-review"
name = "Code Review Plugin"
version = "0.1.0"
api_version = "1.0.0"
min_core_version = "1.0.0"
type = "capability"
runtime = "wasm"
entry = "plugin.wasm"
risk_level = "medium"
experimental = false
signature = "ed25519:<base64>"

[capability]
id = "code.review"
supports_streaming = true
supports_batch = true
supports_resume = false
requires_mandate = false

[permissions]
network = false
filesystem = "readonly"
subprocess = false
env = false
secret = false

[limits]
timeout_ms = 60000
max_memory_mb = 256
max_output_bytes = 10485760

[fallback]
strategy = "deny"
message = "code.review unavailable"
```

## 4. 运行方式

| 插件类型 | 推荐运行方式 |
|---|---|
| Core 内置 | Rust crate |
| 第三方能力 | WASM (Wasmtime) |
| Java 企业 | Sidecar (gRPC) |
| Python AI | Sidecar (gRPC) |
| 高风险系统 | Sidecar + OS sandbox |
| 研究性 | WASM / Sidecar，默认关闭 |

禁止 V1 使用任意 `.so/.dll` 动态注入。

## 5. 插件不可绕过 Core（Kernel Contract）

完整清单见 [kernel-contract.md](./kernel-contract.md)。核心禁止项：

```text
插件不能绕过认证 / 授权 / Mandate
插件不能直接写 Receipt / hash_chain_tip
插件不能返回未审计结果
插件不能修改 risk_level / idempotency_key
插件不能伪造 signature
插件不能访问其他 session 状态
插件不能写 Replay Cache
插件不能重排 Policy 判定顺序
插件不能代 Client 签 FAP signature
```

## 6. Plugin Trait

```rust
#[async_trait::async_trait]
pub trait FapPlugin: Send + Sync {
    fn metadata(&self) -> PluginMetadata;
    async fn init(&self, ctx: PluginInitContext) -> Result<(), FapError>;
    async fn start(&self) -> Result<(), FapError>;
    async fn stop(&self) -> Result<(), FapError>;
    async fn health(&self) -> PluginHealth;
}

#[async_trait::async_trait]
pub trait CapabilityPlugin: FapPlugin {
    async fn list_capabilities(&self) -> Result<Vec<Capability>, FapError>;
    async fn invoke(&self, ctx: InvocationContext, request: InvokeRequest) -> Result<InvocationResult, FapError>;
    async fn cancel(&self, invocation_id: &str) -> Result<(), FapError>;
}
```
