# 02 - 总体架构

## 1. 架构总览

```text
┌──────────────────────────────────────────────┐
│ Agent / LLM / Client Application             │
├──────────────────────────────────────────────┤
│ Model Adapter                                │
│ JSON Tool Schema / FAP Text Tool Syntax      │
├──────────────────────────────────────────────┤
│ FAP-HP Profile (可插拔 Plugin 集合)          │
│ Projection / Folding / DAG Batch / Scheduler │
├──────────────────────────────────────────────┤
│ FAP Core Runtime                             │
│ Session / Auth / Capability / Data / Receipt │
├──────────────────────────────────────────────┤
│ Plugin Runtime                               │
│ Rust Native / WASM / Sidecar                 │
├──────────────────────────────────────────────┤
│ Transport Layer (TransportPlugin)            │
│ QUIC / HTTP3 / HTTP JSON / WebTransport      │
├──────────────────────────────────────────────┤
│ Storage Layer (StoragePlugin)                │
│ SQLite WAL / USearch / Object Store          │
└──────────────────────────────────────────────┘
```

## 2. 核心模块

| 模块 | 职责 | 可插拔点 |
|---|---|---|
| Discovery Service | 暴露 Agent Card | DiscoveryPlugin |
| Session Manager | 会话状态、Resume、流控 | - |
| Security Manager | mTLS/JWT/Mandate/Replay | AuthPlugin / SignaturePlugin |
| Capability Registry | 能力清单、路由 | - |
| Invocation Engine | 调度、DAG、异步任务 | CapabilityRouterPlugin |
| Data Plane | 大对象、ObjectRef 生命周期 | ObjectStorePlugin |
| Plugin Manager | 插件加载、隔离、生命周期 | - |
| Node Manager | 分布式节点、健康断路 | NodeHealthPlugin |
| Receipt Service | 审计、hash chain、多签 | SignaturePlugin / AuditSinkPlugin |
| Extension Manager | 研究能力，编译期可剔除 | - |

## 3. 参考实现选型

仅作为参考，均可替换：

| 组件 | 参考实现 |
|---|---|
| Core Runtime | Rust |
| Async Runtime | Tokio |
| QUIC | Quinn（TransportPlugin 实现） |
| HTTP Gateway | Axum（TransportPlugin 实现） |
| Protobuf | Prost |
| TLS | Rustls |
| JWT | jsonwebtoken（AuthPlugin 实现） |
| 本地数据库 | SQLite WAL（StoragePlugin 实现） |
| 向量索引 | USearch / sqlite-vec |
| 对象存储 | 本地 CAS → S3/MinIO |
| 插件隔离 | Wasmtime + Sidecar |
| 观测 | OpenTelemetry + tracing |
| 测试 | criterion + cargo-fuzz |

## 4. 可插拔边界

```text
Core 提供:
  - 消息定义（fap.proto）
  - 会话/安全/审计 核心流程
  - Plugin 接口契约
  - 消息路由与分发

Plugin 提供:
  - 具体能力实现
  - 具体 Transport 实现
  - 具体 Auth/Sign/Storage 实现

Core 不耦合:
  - 任何具体 TCP/QUIC 库
  - 任何具体 TLS 库
  - 任何具体数据库引擎
  - 任何具体 LLM Provider
```

## 5. 部署形态

| 形态 | 包含 |
|---|---|
| 单机 CLI | Core Lite + HTTP Plugin + 本地 Storage Plugin |
| 边缘 Agent | Core Lite + QUIC Plugin + SQLite |
| 企业网关 | Core Full + QUIC + WASM Plugin + 分布式节点 |
| 研究集群 | Core Full + EXT Plugins（DID/VC / 潜空间 / ...） |
