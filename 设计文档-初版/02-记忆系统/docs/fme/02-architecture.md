# 02 - 总体架构

## 1. 架构总览

```text
┌─────────────────────────────────────────────────────────────────────────┐
│                        FAP-1 Protocol Layer                              │
│  Discovery(Agent Card) │ Session(QUIC/H3) │ Control(Protobuf) │ Data    │
└──────────────────────────────┬──────────────────────────────────────────┘
                               │
┌──────────────────────────────▼──────────────────────────────────────────┐
│                         FAP-ME Gateway                                   │
│                                                                          │
│   ┌─────────────────── Kernel Zone（不可替换）──────────────────────┐    │
│   │  MandateVerifier → PolicyKernel → TenantKernel → ContentSafety │    │
│   │                              ↓                                  │    │
│   │                         AuditKernel                             │    │
│   └───────────────────────────┬─────────────────────────────────────┘    │
│                               │ AuthorizedOp                             │
│   ┌───────────────────────────▼─────────────────────────────────────┐   │
│   │                    Memory Orchestrator                           │   │
│   │            （调度插件，不持有算法实现）                          │   │
│   └──┬──────┬──────┬──────┬──────┬──────┬──────┬───────────────────┘    │
│      │      │      │      │      │      │      │                         │
│  [Embed][VecIdx][TagGr][Comp][Rerank][Dream][Forget][AuditSink]         │
│  可替换  可替换  可替换  可替换  可替换  可替换  可替换  可替换             │
└──────────────────────────────┬──────────────────────────────────────────┘
                               │
┌──────────────────────────────▼──────────────────────────────────────────┐
│              Seahorse-style Rust Cognitive Core                          │
│  Thalamus(Tide/引力场) │ Cortex(HNSW) │ Synapse(LIF+TagGraph V7)        │
│  Hippocampus(SQLite WAL/mmap) │ Cerebellum(后台任务)                    │
└──────────────────────────────┬──────────────────────────────────────────┘
                               │
┌──────────────────────────────▼──────────────────────────────────────────┐
│                         Memory Layers                                    │
│  L0(即时上下文) │ L1(剧集记忆) │ L2(语义本体) │ L3(潜意识后台)         │
└─────────────────────────────────────────────────────────────────────────┘
```

## 2. Kernel Zone（不可替换）

| 组件 | 职责 | 必留理由 |
|---|---|---|
| MandateVerifier | 验证 Intent Mandate 签名、过期、purpose | 安全凭证不能由插件验证 |
| PolicyKernel | 唯一授权入口，输出 AuthorizedOp | 安全判定不外包 |
| TenantKernel | 强制注入 tenant_id / namespace 过滤 | 租户隔离不能被绕过 |
| ContentSafetyGuard | 写入前 Prompt Injection 检测 | 不可绕过的内容守卫 |
| AuditKernel | 维护哈希链，可定期验证完整性 | 链顺序不能由插件改写 |

详见 [kernel-contract.md](./kernel-contract.md)、[10-security-model.md](./10-security-model.md)、[11-audit-and-receipt.md](./11-audit-and-receipt.md)。

## 3. Plugin Zone（可替换）

| 扩展点 | 默认实现 | 备选 |
|---|---|---|
| EmbeddingProvider | OpenAI / local sentence-transformers | bge / cohere |
| VectorIndex | HNSW（usearch） | Qdrant / sqlite-vec |
| TagGraph | SQLite edge + CSR mmap（V7 有向） | SurrealDB / Neo4j |
| CompactStrategy | retain_score 加权折叠 | LLM-based summarizer |
| Reranker | cross-encoder（轻量） | BGE-reranker-v2 |
| DreamWorker | 规则 + LLM proposal | 仅规则、仅 LLM |
| ForgetEngine | tombstone + 级联 purge | hard-physical-delete |
| AuditSink | SQLite append-only | Kafka / S3 / OpenTelemetry |
| ObjectStore | 本地 CAS | S3 / MinIO |

所有 Plugin 必须实现 [09-plugin-runtime.md](./09-plugin-runtime.md) 中定义的 `Plugin` 基础 trait。

## 4. Cognitive Core（Seahorse 脑区分工）

```text
Thalamus       意图门控、Tide 能量分解、引力场修正        不存储状态
Cortex         HNSW 向量索引、向量计算                    不处理 Tag 拓扑
Synapse        LIF 脉冲扩散、共现矩阵、V7 有向拓扑        不做向量检索
Hippocampus    SQLite WAL、mmap、事务、版本               不做计算
Cerebellum     后台任务调度、梦境、压缩、健康分析         不阻塞主路径
```

详见 [04-cognitive-core.md](./04-cognitive-core.md)。

## 5. 数据流方向

```text
写入路径：
  Agent → FAP Control Plane → Mandate → Policy → ContentSafety
        → L0 Append → CompactStrategy.should_fold
        → [触发] Episode Folder → Embedding → Hippocampus.put
        → Cortex.add(vec) + Synapse.upsert(tags)
        → AuditKernel.append

检索路径：
  Agent → FAP Control Plane → Mandate → Policy → Scope Filter
        → Embedding → Thalamus.decompose
        → 并行: L0 search / L1 hybrid / L2 HNSW + Synapse.expand
        → Candidate Pool → Dedup → Reranker → Geodesic Rerank
        → PolicyKernel.redact → AuditKernel.append
        → FAP Data Plane RecallResultStream

后台路径：
  Cerebellum Trigger → Sample L1 → Exclude SBU
        → DreamWorker.propose → DreamProposal 状态机
        → [APPROVED] Kernel 执行 mutations → Audit
```

## 6. 可插拔边界

```text
Core 提供：
  - Kernel 实现（Policy / Audit / Tenant / Mandate / ContentSafety）
  - Plugin Trait 契约
  - PluginRegistry（注册、发现、版本管理、租户级覆盖、WASM 沙箱）
  - Memory Orchestrator（调度插件，不持有算法）
  - 检索流水线骨架（编排，不实现算法）
  - FAP-1 Protobuf 编解码

Plugin 提供：
  - 具体算法（向量化、索引、拓扑、折叠、精排、梦境、遗忘）
  - 具体外部适配（embedding API、vector DB、object store、audit sink）

Core 不耦合：
  - 任何具体 embedding 模型
  - 任何具体向量数据库
  - 任何具体 LLM Provider
  - 任何具体 reranker 算法
```

## 7. 部署形态

| 形态 | 包含 |
|---|---|
| 边缘 Agent（Core Lite） | Kernel + 内置插件（local emb / HNSW / SQLite TagGraph / append-only audit） |
| 单机服务（Standalone） | + WASM 插件沙箱、+ DreamWorker、+ ForgetEngine |
| 企业网关（多租户） | + Qdrant / Redis / Kafka 适配插件、+ DID/VC 验证、+ tenant 级配置覆盖 |
| 研究集群（EXT） | + 跨节点 CRDT 同步（FAP-1 EXT 范围）、+ 联邦学习插件（实验） |

详见 [14-deployment-profiles.md](./14-deployment-profiles.md)。

## 8. 参考实现选型

仅作为参考，均可替换：

| 组件 | 参考实现 |
|---|---|
| Core Runtime | Rust + Tokio |
| QUIC/HTTP | 复用 FAP-1 Transport |
| 本地数据库 | SQLite WAL（含 vector / fts5 扩展） |
| 向量索引 | usearch（HNSW，mmap 友好） |
| Tag 拓扑 | SQLite edge table + CSR mmap |
| 服务端 vector | Qdrant / Milvus 适配插件 |
| 后台任务 | Tokio + cron + 优先级队列 |
| 插件隔离 | Wasmtime（WASM）/ Sidecar（gRPC） |
| 观测 | OpenTelemetry + tracing |
| 测试 | criterion + cargo-fuzz + 负例 conformance |
