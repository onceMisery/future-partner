# 09 - 插件运行时

> 边界与禁止行为见 [kernel-contract.md](./kernel-contract.md)；多租户配置见 [multi-tenant.md](./multi-tenant.md)。

## 1. 插件分类

FAP-ME 的插件按职责分组：

| 分组 | 高层类别 | Trait |
|---|---|---|
| 算法 | EmbeddingProvider | 向量化 |
| 算法 | VectorIndex | ANN 检索 |
| 算法 | TagGraph | 拓扑扩散 |
| 算法 | CompactStrategy | L0 折叠 |
| 算法 | Reranker | 精排 |
| 算法 | DreamWorker | 梦境提案 |
| 算法 | ForgetEngine | 遗忘规划 |
| 适配 | AuditSink | 审计输出 |
| 适配 | ObjectStore | 大对象存储 |

注意：`PolicyEngine` 不在插件清单中，安全判定由 Kernel 内置（详见 [kernel-contract.md](./kernel-contract.md)）。

## 2. Plugin 基础 trait

```rust
pub trait Plugin: Send + Sync {
    fn name(&self) -> &str;
    fn version(&self) -> SemVer;
    fn capabilities(&self) -> PluginCapabilities;
    fn health_check(&self) -> PluginHealth;
    /// 多租户场景下，按租户配置生成新实例
    fn configure_for_tenant(
        &self,
        tenant_id: &TenantId,
        config: &TenantPluginConfig,
    ) -> Result<Box<dyn Plugin>>;
}

pub struct PluginCapabilities {
    pub supports_streaming: bool,
    pub supports_batch: bool,
    pub supports_resume: bool,
    pub experimental: bool,
}

pub enum PluginHealth {
    Healthy,
    Degraded { reason: String },
    Unhealthy { reason: String },
}
```

## 3. 各插件 Trait 定义

### 3.1 EmbeddingProvider

```rust
pub trait EmbeddingProvider: Plugin {
    fn embed(&self, texts: &[&str]) -> Result<Vec<Vec<f32>>>;
    fn dimension(&self) -> usize;
    fn model_id(&self) -> &str;
}
```

### 3.2 VectorIndex

```rust
pub trait VectorIndex: Plugin {
    fn add(&self, id: MemoryId, vector: &[f32], meta: IndexMetadata) -> Result<()>;
    fn search(&self, query: &[f32], top_k: usize, filter: &SearchFilter) -> Result<Vec<VectorHit>>;
    fn mark_deleted(&self, id: MemoryId) -> Result<()>;
    fn rebuild(&self, chunks: Box<dyn Iterator<Item = IndexChunk>>) -> Result<IndexVersion>;
    /// 双索引切换：embedding 模型升级时
    fn fork_version(&self) -> Result<Box<dyn VectorIndex>>;
    /// 切流评估：双索引共存期路由用
    fn query_routing_score(&self) -> f32;
}
```

### 3.3 TagGraph

```rust
pub trait TagGraph: Plugin {
    fn upsert(&self, memory_id: MemoryId, tags: &[Tag]) -> Result<()>;
    fn expand(&self, seeds: &[Tag], params: &LifParams) -> Result<SpikeTrace>;
    fn prune(&self, min_weight: f32, max_age_days: u32) -> Result<PruneReport>;
    /// 冷热分离
    fn load_hot_edges(&self, top_n: usize) -> Result<HotGraph>;
    fn flush_cold_edges(&self, hot_graph: &HotGraph) -> Result<()>;
    /// V7 有向序位拓扑
    fn ordinal_expand(
        &self,
        seeds: &[TagWithPotential],
        params: &OrdinalLifParams,
    ) -> Result<SpikeTrace>;
}
```

详见 [tag-graph-v7.md](./tag-graph-v7.md)。

### 3.4 CompactStrategy

```rust
pub trait CompactStrategy: Plugin {
    fn should_fold(&self, state: &L0State) -> FoldDecision;
    fn fold(&self, events: &[L0Event]) -> Result<EpisodicSummary>;
    fn retain_score(&self, event: &L0Event) -> f32;
}

pub enum FoldDecision {
    Continue,
    Fold,
    FoldAtBoundary,
    FoldOnTimeout,
    ForceFold,
}
```

详见 [retain-score.md](./retain-score.md)。

### 3.5 Reranker

```rust
pub trait Reranker: Plugin {
    fn rerank(
        &self,
        query: &str,
        candidates: &[CandidateUnit],
    ) -> Result<Vec<RerankedUnit>>;
}
```

### 3.6 DreamWorker

```rust
pub trait DreamWorker: Plugin {
    fn propose(&self, scope: &MemoryScope, params: &DreamParams) -> Result<DreamProposal>;
    /// 仅返回变更描述，由 Kernel 执行（插件不能直接写入）
    fn describe_mutations(&self, proposal: &DreamProposal) -> Vec<MemoryMutation>;
    fn assess_risk(&self, proposal: &DreamProposal) -> RiskLevel;
}
```

详见 [dream-state-machine.md](./dream-state-machine.md)。

### 3.7 ForgetEngine

```rust
pub trait ForgetEngine: Plugin {
    fn plan(&self, req: &ForgetRequest, lineage: &MemoryLineage) -> Result<ForgetPlan>;
    /// 仅返回操作列表，由 Kernel 执行（插件不能直接删除）
    fn mutations(&self, plan: &ForgetPlan) -> Vec<MemoryMutation>;
}
```

详见 [forget-engine.md](./forget-engine.md)。

### 3.8 AuditSink

```rust
pub trait AuditSink: Plugin {
    /// 仅写入；不计算 hash chain（由 AuditKernel 计算）
    fn append(&self, event: &SignedAuditEvent) -> Result<()>;
    fn read_range(&self, from: EventId, to: EventId) -> Result<Vec<SignedAuditEvent>>;
}
```

### 3.9 ObjectStore

```rust
pub trait ObjectStore: Plugin {
    fn put(&self, content: &[u8], meta: ObjectMeta) -> Result<ObjectRef>;
    fn get(&self, object_id: &str) -> Result<Vec<u8>>;
    fn delete(&self, object_id: &str) -> Result<()>;
    fn lease(&self, object_id: &str, ttl: Duration) -> Result<LeaseId>;
}
```

## 4. PluginRegistry

```rust
pub struct PluginRegistry {
    embedding_providers: RwLock<HashMap<String, (SemVer, Arc<dyn EmbeddingProvider>)>>,
    vector_indexes: RwLock<HashMap<String, (SemVer, Arc<dyn VectorIndex>)>>,
    tag_graphs: RwLock<HashMap<String, (SemVer, Arc<dyn TagGraph>)>>,
    compact_strategies: RwLock<HashMap<String, (SemVer, Arc<dyn CompactStrategy>)>>,
    rerankers: RwLock<HashMap<String, (SemVer, Arc<dyn Reranker>)>>,
    dream_workers: RwLock<HashMap<String, (SemVer, Arc<dyn DreamWorker>)>>,
    forget_engines: RwLock<HashMap<String, (SemVer, Arc<dyn ForgetEngine>)>>,
    audit_sinks: RwLock<HashMap<String, (SemVer, Arc<dyn AuditSink>)>>,
    object_stores: RwLock<HashMap<String, (SemVer, Arc<dyn ObjectStore>)>>,
    /// 租户级别配置覆盖
    tenant_overrides: RwLock<HashMap<TenantId, TenantPluginConfig>>,
    /// WASM 沙箱
    wasm_sandbox: Arc<WasmSandbox>,
}

impl PluginRegistry {
    pub fn register<P: Plugin + 'static>(&self, plugin: P) -> Result<()>;
    pub fn unregister(&self, name: &str, version: &SemVer) -> Result<()>;
    pub fn list_plugins(&self) -> Vec<PluginMeta>;
    pub fn resolve_for_tenant(
        &self,
        tenant_id: &TenantId,
        kind: PluginKind,
    ) -> Result<Arc<dyn Plugin>>;
}
```

## 5. Manifest

```toml
id = "fap-me-vector-hnsw"
name = "HNSW Vector Index"
version = "0.1.0"
api_version = "1.0.0"
min_core_version = "1.0.0"
type = "vector_index"
runtime = "rust_native"     # rust_native | wasm | sidecar
entry = "libfap_me_vector_hnsw.so"
risk_level = "low"
experimental = false
signature = "ed25519:<base64>"

[capabilities]
supports_streaming = false
supports_batch = true
supports_resume = false

[permissions]
filesystem = "readwrite_self_dir"
network = false
subprocess = false

[limits]
memory_mb = 1024
init_timeout_ms = 5000
```

## 6. WASM 沙箱

第三方插件强制 WASM：

```rust
pub struct WasmSandbox {
    memory_limit_bytes: usize,      // 默认 64MB
    execution_timeout_ms: u64,      // 默认 200ms
    allowed_hostfns: Vec<String>,   // 白名单宿主函数
    deny_network_access: bool,      // true
    deny_filesystem_access: bool,   // true
}
```

WASM 插件不能：
- 直接访问文件系统
- 发起网络请求
- 调用未在白名单内的宿主函数
- 超时或越界内存（沙箱终止）

## 7. 运行方式

| 插件类型 | 推荐运行方式 |
|---|---|
| Core 内置 | Rust crate（直接链接） |
| 第三方算法 | WASM (Wasmtime) |
| 重型 LLM 算法 | Sidecar (gRPC) |
| 高风险系统 | Sidecar + OS sandbox |
| 实验性 | WASM，默认关闭 |

禁止 V1 使用任意 `.so/.dll` 动态注入。

## 8. 生命周期

```text
UPLOADED → VALIDATED → STAGED → DRY_RUN_PASSED → ACTIVE
  → DRAINING → STOPPED → UNLOADED
```

升级期间 `plugin_id + api_version` 可并存，调度按租户配置选择版本。

## 9. 插件不可绕过 Core

完整清单见 [kernel-contract.md](./kernel-contract.md)。核心禁止项：

```text
插件不能：
  绕过 PolicyKernel 直接访问 MemoryUnit
  绕过 ContentSafetyGuard 写入内容
  直接生成 AuditEvent（必须经 AuditKernel）
  直接修改 hash_chain_tip
  访问其他租户的数据
  访问其他插件的私有状态
  写入 mandate / receipt 表
  伪造 signature
```

## 10. 插件审计

每次插件调用必须留下审计：

```text
plugin.invoke(
  plugin_id,
  api_version,
  input_hash,
  output_hash,
  duration_ms,
  health_status,
)
```

通过 AuditKernel 写入哈希链。
