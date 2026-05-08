# 14 - 部署形态

## 1. 三档 Profile

```text
Edge Lite          边缘 / 个人 Agent
Standalone         单机服务 / 小团队
Enterprise         多租户企业网关
Research / EXT     实验集群（可选）
```

## 2. Edge Lite（边缘 Profile）

### 适用场景

```text
个人 Agent
本地 IDE Agent
小团队 Agent
离线场景
低功耗设备
```

### 包含组件

```text
Kernel：完整（不可替换）
Plugin：
  EmbeddingProvider：local（sentence-transformers / ONNX）
  VectorIndex：sqlite-vec 或 usearch（mmap）
  TagGraph：SQLite edge + hot mmap
  CompactStrategy：retain_score 默认实现
  AuditSink：SQLite append-only
检索模式：basic + hybrid
不启用：
  Qdrant / Redis / Kafka
  DID/VC（用 mTLS+JWT 即可）
  WASM 沙箱（仅核心 Rust 插件）
  DreamWorker（可选）
```

### 资源要求

```text
内存：≥ 512MB
磁盘：≥ 1GB（含索引）
CPU：单核可用
```

### Edge Lite 容量模型

512MB 目标仅在启用裁剪策略时成立：

| 组件 | 预算 | 策略 |
|---|---:|---|
| Kernel + runtime | 96MB | 固定常驻 |
| SQLite page cache + WAL | 96MB | checkpoint + page cache 上限 |
| 本地 HNSW / sqlite-vec | 160MB | mmap，热层 Top-K，冷索引按需加载 |
| Tag hot graph | 48MB | Top-N 热边，默认 50k，可按设备下调 |
| ContentSafety / redaction rules | 32MB | 预编译规则，分类器可用轻量版 |
| Audit / outbox buffers | 32MB | 小批量刷盘，外部 sink 禁用或异步 |
| 余量 | 48MB | 瞬时查询、序列化、SDK |

裁剪规则：

```text
L0 ring buffer 超预算 → 先 fold，再丢弃低 retain_score 临时项
HNSW mmap 超预算 → 切 sqlite-vec 或只保留 L1/L2 热分区
TagGraph 超预算 → 降低 hot edge Top-N，冷边保留在 SQLite
DreamWorker 默认关闭；LLM sidecar 不计入 Edge Lite 常驻预算
```

### 部署示例

```bash
fap-me-server \
  --profile=edge-lite \
  --data-dir=.fap-me \
  --bind=127.0.0.1:7700
```

## 3. Standalone（单机 Profile）

### 适用场景

```text
单机服务
中等规模团队
本地 NAS 部署
中型 Agent 编排
```

### 包含组件

```text
Edge Lite 全部 + 
  WASM 插件沙箱
  DreamWorker（默认实现）
  ForgetEngine（默认实现）
  Reranker（cross-encoder 轻量）
检索模式：basic / hybrid / tide / tagmemo
存储：SQLite WAL + 本地 HNSW + 本地 CAS
观测：tracing + 本地 OpenTelemetry collector
```

### 资源要求

```text
内存：≥ 4GB
磁盘：≥ 50GB
CPU：≥ 4 核
```

## 4. Enterprise（企业多租户 Profile）

### 适用场景

```text
企业 Agent 平台
多团队协作
跨组织共享（含 DID/VC）
合规审计
```

### 包含组件

```text
Standalone 全部 +
  PostgreSQL（替代 SQLite）
  Qdrant / Milvus（替代本地 HNSW）
  Redis（mandate revocation 缓存 + L0 热缓存）
  Kafka（审计事件流 + 后台任务队列）
  S3 / MinIO（对象存储）
  DID Resolver + VC Verifier
  租户级别 PluginConfig 覆盖
检索模式：全开
观测：Prometheus + Jaeger + ELK
```

### 资源要求

```text
内存：≥ 16GB（每节点）
磁盘：分层（PG + 对象存储独立）
CPU：≥ 8 核
节点数：≥ 3（高可用）
```

### 部署示例

```text
节点拓扑：
  ingress（负载均衡）
    ↓
  fap-me-gateway × N（无状态）
    ↓
  PG primary + replica
  Qdrant cluster
  Redis sentinel
  Kafka cluster
  S3 / MinIO
  
插件运行时：
  Sidecar（Java/Python 算法）
  WASM（第三方插件）
```

## 5. Research / EXT Profile

### 适用场景

```text
研究集群
联邦记忆
跨节点 CRDT 同步
潜空间通信
```

### 包含组件

```text
Enterprise 全部 +
  EXT 插件（默认关闭，编译期可剔除）
  CRDT 跨节点同步（实验）
  Federated Embedding（实验）
  Latent Space Communication（实验）
```

EXT 插件不在 Phase 6 验收范围；仅供研究试点。

## 6. Profile 配置示例

```toml
# fap-me.toml

[profile]
name = "enterprise"

[storage]
metadata_backend = "postgres"
postgres_url = "postgres://..."
vector_backend = "qdrant"
qdrant_url = "http://..."
object_store = "s3"
s3_bucket = "fap-me-prod"
audit_sink = ["kafka", "s3"]

[security]
auth = ["mtls", "jwt", "did", "vc", "dpop"]
fap_mandate_constraints_required = true
content_safety_guard = true
chain_integrity_verify_cron = "30 0 * * *"

[retrieval]
default_mode = "hybrid"
enabled_modes = ["basic", "hybrid", "tide", "tagmemo", "tagmemo_geodesic", "sbu_safe"]
shotgun_max_concurrent = 8
total_timeout_ms = 300

[plugins.embedding]
default = "openai"

[plugins.embedding.tenant_overrides.medical_org]
provider = "biomedical-bert"

[plugins.vector_index]
default = "qdrant"

[plugins.dream_worker]
default = "rule-plus-llm"

[wasm_sandbox]
memory_limit_mb = 64
timeout_ms = 200
deny_network = true
deny_filesystem = true

[observability]
otel_endpoint = "http://otel-collector:4317"
metrics = "prometheus"
```

## 7. Profile 切换约束

```text
Edge Lite → Standalone：插件热加载即可
Standalone → Enterprise：需要数据迁移（SQLite → PG，HNSW → Qdrant）
任何 → Research：开启 EXT 编译标记，不影响主路径
```

## 8. 高可用建议

```text
Gateway：无状态，N 实例
Kernel：无状态，跟随 Gateway
PG：primary + 1+ replica + 自动故障切换
Qdrant：cluster 模式，至少 3 节点
Redis：sentinel 或 cluster
Kafka：≥ 3 broker
S3 / MinIO：跨可用区
```

## 9. 灾备

```text
RPO（Recovery Point Objective）：
  audit chain：0（同步写）
  L1 / L2：≤ 1h（异步快照）
  L0：可丢失（重建从最近 L1）

RTO（Recovery Time Objective）：
  Gateway 重启：≤ 30s
  PG 故障切换：≤ 60s
  全集群恢复：≤ 30min（含索引重建）
```

## 10. 推荐升级路径

```text
1. 起步：Edge Lite 单机部署（POC）
2. 团队化：Standalone 加 Reranker / DreamWorker
3. 企业化：Enterprise 启用多租户 + Qdrant + Kafka
4. 跨组织：启用 DID/VC + 跨 Agent Handoff
5. 研究：EXT 插件试点
```
