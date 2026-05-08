# 16 - Rust 实现与目录结构

## 1. 顶层布局

```text
fap-memory-engine/
├── crates/                 Rust workspace
├── plugins/                可替换插件实现
├── proto/                  Protobuf 定义
├── sdk/                    多语言 SDK
├── config/                 配置示例
├── docs/                   文档（本目录）
└── tests/                  集成测试 + conformance 负例
```

## 2. crates 拆分

```text
crates/
├── fap-me-kernel/          Kernel Zone（不可替换）
│   ├── policy_kernel.rs
│   ├── audit_kernel.rs
│   ├── tenant_kernel.rs
│   ├── mandate_verifier.rs
│   ├── content_safety_guard.rs
│   └── purpose_registry.rs
│
├── fap-me-core/            认知内核（Seahorse 脑区）
│   ├── thalamus.rs         Tide 算法
│   ├── cortex.rs           HNSW 适配
│   ├── synapse.rs          LIF + TagGraph V7
│   ├── hippocampus.rs      SQLite WAL + mmap
│   └── cerebellum.rs       后台任务
│
├── fap-me-plugin/          插件接口定义
│   ├── traits.rs           Plugin / EmbeddingProvider / VectorIndex / ...
│   ├── registry.rs         PluginRegistry
│   ├── tenant_config.rs    租户级覆盖
│   └── wasm_sandbox.rs     Wasmtime 集成
│
├── fap-me-server/          axum + tonic 网关
│   ├── grpc_service.rs
│   ├── http_gateway.rs
│   └── stream_handler.rs
│
├── fap-me-retrieval/       检索流水线
│   ├── orchestrator.rs
│   ├── pipeline.rs
│   ├── fallback.rs         降级链
│   └── scoring.rs          final_score 计算
│
├── fap-me-data-model/      数据结构（serde + prost）
│   ├── memory_unit.rs
│   ├── tag.rs
│   ├── episode.rs
│   ├── grant.rs
│   ├── handoff.rs
│   ├── dream.rs
│   ├── audit.rs
│   └── redaction.rs
│
├── fap-me-storage-sqlite/  SQLite WAL 实现
├── fap-me-storage-pg/      PostgreSQL 实现（可选）
└── fap-me-wasm/            wasm-bindgen SDK
```

## 3. plugins 目录

```text
plugins/
├── embedding-openai/
├── embedding-local/             sentence-transformers / ONNX
├── vector-hnsw-usearch/
├── vector-qdrant/
├── vector-sqlite-vec/
├── taggraph-sqlite/             V7 有向序位 + 冷热分离
├── compact-default/             retain_score 默认实现
├── reranker-cross-encoder/
├── reranker-bge/
├── dream-rule-llm/              规则 + LLM 提案
├── forget-default/              tombstone + 级联 purge
├── audit-sink-kafka/
├── audit-sink-s3/
├── audit-sink-otel/
└── object-store-s3/
```

每个 plugin 是独立 cargo crate，含 Manifest（详见 [09-plugin-runtime.md](./09-plugin-runtime.md)）。

## 4. proto 目录

```text
proto/
├── memory.proto            MemoryControlService + 控制消息
├── handoff.proto           HandoffPacket + RedactionReport
├── mandate.proto           Intent Mandate + StandardPurpose
├── dream.proto             DreamProposal 状态机
├── audit.proto             AuditEvent + ChainState
└── redaction.proto         RedactionPolicy
```

详见 [15-fap1-integration.md](./15-fap1-integration.md) 中的 Protobuf 定义。

## 5. sdk 目录

```text
sdk/
├── typescript/             napi-rs 绑定
├── python/                 PyO3 绑定
├── go/                     gRPC client
└── java/                   gRPC client + sidecar runtime
```

## 6. config 示例

```text
config/
├── purpose_registry.json   标准化 purpose 词汇表
├── redaction_policies/
│   ├── standard_cross_agent.json
│   ├── compliance_audit.json
│   └── handoff_default.json
├── fap-me.toml             主配置
└── plugins/
    └── *.manifest.toml
```

## 7. 关键 Trait 总览

```rust
// kernel
pub struct PolicyKernel { ... }
pub struct AuditKernel { ... }
pub struct TenantKernel { ... }
pub struct ContentSafetyGuard { ... }
pub struct MandateVerifier { ... }

// 插件基础
pub trait Plugin { ... }

// 算法插件
pub trait EmbeddingProvider: Plugin { ... }
pub trait VectorIndex: Plugin { ... }
pub trait TagGraph: Plugin { ... }
pub trait CompactStrategy: Plugin { ... }
pub trait Reranker: Plugin { ... }
pub trait DreamWorker: Plugin { ... }
pub trait ForgetEngine: Plugin { ... }

// 适配插件
pub trait AuditSink: Plugin { ... }
pub trait ObjectStore: Plugin { ... }
```

详见 [09-plugin-runtime.md](./09-plugin-runtime.md)。

## 8. 异步与并发模型

```text
Runtime：Tokio
检索路径：fully async
存储路径：tokio::task::spawn_blocking 包裹同步 SQLite 调用
后台任务：Tokio task + 优先级队列（PriorityQueue<JobSpec>）
插件调度：每个 Plugin 调用都有独立超时（tokio::time::timeout）
```

## 9. 性能指标基线（参考）

| 操作 | P50 | P99 |
|---|---:|---:|
| basic 检索 | 5ms | 20ms |
| hybrid 检索 | 15ms | 60ms |
| tagmemo 检索 | 30ms | 120ms |
| tide 检索 | 40ms | 150ms |
| store（不含 LLM） | 20ms | 100ms |
| ContentSafetyGuard 检测 | < 1ms | < 5ms |
| AuditKernel.append | < 1ms | < 5ms |

测试方法见 [17-testing-conformance.md](./17-testing-conformance.md)。

## 10. 依赖清单（参考）

| 用途 | crate |
|---|---|
| async runtime | tokio |
| HTTP / gRPC | axum / tonic |
| Protobuf | prost |
| SQLite | rusqlite + bundled |
| 向量索引 | usearch（Rust binding） |
| 序列化 | serde + serde_json + rkyv |
| WASM 沙箱 | wasmtime |
| TLS | rustls |
| JWT / JWS | jsonwebtoken / josekit |
| 哈希 | sha2 / blake3 |
| 观测 | tracing / opentelemetry |
| 测试 | criterion / proptest / cargo-fuzz |

## 11. Feature flag

```toml
[features]
default = ["edge-lite"]

edge-lite = ["sqlite", "usearch"]
standalone = ["edge-lite", "wasm-sandbox", "dream-default"]
enterprise = ["standalone", "postgres", "qdrant", "kafka", "did-vc"]
research-ext = ["enterprise", "ext-crdt", "ext-latent"]

ext-crdt = []          # 实验，默认关闭
ext-latent = []        # 实验，默认关闭
```

## 12. 编译矩阵

```text
- linux-x64        生产部署
- darwin-arm64     开发机
- windows-x64      Windows Agent
- wasm32-unknown   浏览器 / Edge Worker（fap-me-wasm crate 单独）
```
