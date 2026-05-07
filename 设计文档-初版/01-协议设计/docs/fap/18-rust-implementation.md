# 18 - Rust 实现

Rust 作为参考实现，但协议本身不绑定 Rust。

## 1. 核心 Crate

```text
fap-proto/              proto + build.rs
fap-core/               envelope/header/codec/error/hash/signature/receipt
fap-session/            manager/state/resume/flow/version
fap-security/           mtls/jwt/mandate/replay/policy/revocation
fap-plugin/             manifest/manager/runtime/wasm/sidecar
fap-capability/         registry/router/scheduler/batch/dag
fap-dataplane/          object_ref/object_store/stream/lease/resolver
fap-discovery/          agent_card/handler
fap-transport/          quic/http_json/webtransport
fap-hp/                 projection/context_folding/semantic_routing/local_store
fap-node/               node/register/remote_invoke/heartbeat/health
fap-audit/              receipt_service/hash_chain/checkpoint/multisig
fap-ext-didvc/          (feature gate)
fap-ext-latent/         (feature gate)
fap-ext-gossip/         (feature gate)
fap-ext-crdt/           (feature gate)
fap-bridge-mcp/
fap-bridge-a2a/
fap-server/
fap-cli/                (含 fap-inspector)
```

## 2. 关键接口

```rust
pub async fn handle_envelope(
    ctx: RequestContext,
    envelope: Envelope,
    services: &Services,
) -> Result<Envelope, FapError> {
    services.validator.validate_header(&envelope.header)?;
    services.replay.check_and_store(&envelope.header)?;
    services.security.verify(&ctx, &envelope).await?;
    services.policy.check(&ctx, &envelope).await?;

    match envelope.body {
        Some(Body::ClientHello(msg))        => services.session.handle_client_hello(ctx, msg).await,
        Some(Body::AuthInit(msg))           => services.security.handle_auth_init(ctx, msg).await,
        Some(Body::InvokeRequest(msg))      => services.invocation.invoke(ctx, envelope.header, msg).await,
        // ...
        _ => Err(FapError::unsupported_message()),
    }
}
```

## 3. Plugin Trait

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

## 4. 技术选型（参考）

| 组件 | 选型 |
|---|---|
| Async Runtime | Tokio |
| QUIC | Quinn（TransportPlugin） |
| HTTP | Axum（TransportPlugin） |
| Protobuf | Prost |
| TLS | Rustls |
| JWT | jsonwebtoken（AuthPlugin） |
| 数据库 | SQLite WAL（StoragePlugin） |
| 向量 | USearch / sqlite-vec |
| 观测 | OpenTelemetry + tracing |
| 测试 | criterion + cargo-fuzz |
