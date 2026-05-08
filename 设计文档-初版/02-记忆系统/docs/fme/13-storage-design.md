# 13 - 存储设计

> 数据模型见 [05-data-model.md](./05-data-model.md)；部署形态选择见 [14-deployment-profiles.md](./14-deployment-profiles.md)。

## 1. 存储分层

```text
元数据 / 事务      关系型 DB（SQLite WAL / PostgreSQL）
向量索引          HNSW（usearch / sqlite-vec / Qdrant）
Tag 共现          edge table + CSR mmap 快照
对象存储          本地 CAS / S3 / MinIO
审计输出          可插件化（SQLite WAL / Kafka / S3）
热缓存            可选 Redis（服务端模式）
```

## 2. 单机/边缘存储布局

```text
.fap-me/
├── tenant.db                 SQLite WAL 主库
├── tenant.db-wal
├── indexes/
│   ├── hnsw_v1.rkyv
│   ├── hnsw_v2.rkyv
│   └── tag_centroids.rkyv
├── tag_graph/
│   ├── hot.csr.mmap
│   └── cold.snapshot
├── snapshots/
│   └── session_snapshot_*.bin
├── audit/
│   └── audit_chain.log
└── objects/
    └── attachments/
```

## 3. SQLite 核心表

```sql
CREATE TABLE memory_unit (
  memory_id TEXT PRIMARY KEY,
  tenant_id TEXT NOT NULL,
  namespace TEXT NOT NULL,
  layer TEXT NOT NULL,                   -- L0 | L1 | L2
  visibility TEXT NOT NULL,
  owner_subject_did TEXT,
  owner_agent_did TEXT,
  content_ref TEXT,                      -- inline 或 object_ref
  content_hash BLOB NOT NULL,
  embedding_model TEXT NOT NULL,
  index_version TEXT NOT NULL,
  revision INTEGER NOT NULL,
  retention_policy TEXT,
  source_confidence REAL,
  stability REAL,
  reusability REAL,
  created_at INTEGER NOT NULL,
  updated_at INTEGER NOT NULL,
  deleted_at INTEGER
);
CREATE INDEX idx_memory_tenant_layer ON memory_unit(tenant_id, layer);
CREATE INDEX idx_memory_namespace ON memory_unit(tenant_id, namespace);

CREATE TABLE memory_unit_security (
  memory_id TEXT NOT NULL,
  label TEXT NOT NULL,                   -- sbu | secret | pii | internal
  PRIMARY KEY (memory_id, label),
  FOREIGN KEY (memory_id) REFERENCES memory_unit(memory_id)
);

CREATE TABLE memory_lineage (
  child_id TEXT NOT NULL,
  parent_id TEXT NOT NULL,
  PRIMARY KEY (child_id, parent_id)
);

CREATE TABLE tag (
  tag_id TEXT PRIMARY KEY,
  tag_name TEXT NOT NULL UNIQUE,
  tag_type TEXT NOT NULL,
  doc_count INTEGER NOT NULL DEFAULT 0,
  avg_cooccur REAL,
  recency REAL,
  intrinsic_residual REAL                -- V7 内生残差
);

CREATE TABLE memory_tag (
  memory_id TEXT NOT NULL,
  tag_id TEXT NOT NULL,
  PRIMARY KEY (memory_id, tag_id)
);

CREATE TABLE tag_edge (
  source_tag_id TEXT NOT NULL,
  target_tag_id TEXT NOT NULL,
  weight REAL NOT NULL,                  -- Φ_src × Φ_dst
  source_potential REAL NOT NULL,
  target_potential REAL NOT NULL,
  cooccur_count INTEGER NOT NULL,
  last_seen INTEGER NOT NULL,
  is_hot INTEGER NOT NULL DEFAULT 0,
  PRIMARY KEY (source_tag_id, target_tag_id)
);
CREATE INDEX idx_tag_edge_hot ON tag_edge(is_hot, weight DESC);

CREATE TABLE episode (
  episode_id TEXT PRIMARY KEY,
  tenant_id TEXT NOT NULL,
  namespace TEXT NOT NULL,
  title TEXT,
  summary TEXT,
  start_time INTEGER NOT NULL,
  end_time INTEGER,
  confidence_score REAL,
  source_session_id TEXT,
  created_at INTEGER NOT NULL
);

CREATE TABLE semantic_fact (
  fact_id TEXT PRIMARY KEY,
  tenant_id TEXT NOT NULL,
  namespace TEXT NOT NULL,
  subject TEXT NOT NULL,
  predicate TEXT NOT NULL,
  object TEXT NOT NULL,
  confidence REAL NOT NULL,
  stability REAL,
  reusability REAL,
  source_confidence REAL,
  revision INTEGER NOT NULL,
  conflict_state TEXT NOT NULL,          -- active | superseded | forked | rejected
  superseded_by TEXT,
  last_validated_at INTEGER,
  created_at INTEGER NOT NULL,
  updated_at INTEGER NOT NULL
);

CREATE TABLE context_snapshot (
  snapshot_id TEXT PRIMARY KEY,
  tenant_id TEXT NOT NULL,
  namespace TEXT NOT NULL,
  source_session_id TEXT,
  content_hash BLOB NOT NULL,
  created_by_agent_did TEXT NOT NULL,
  created_at INTEGER NOT NULL,
  expires_at INTEGER NOT NULL
);

CREATE TABLE context_grant (
  grant_id TEXT PRIMARY KEY,
  issuer_did TEXT NOT NULL,
  receiver_agent_did TEXT NOT NULL,
  tenant_id TEXT NOT NULL,
  namespace TEXT NOT NULL,
  allowed_layers_json TEXT NOT NULL,
  allowed_ops_json TEXT NOT NULL,
  redaction_policy_id TEXT NOT NULL,
  purpose TEXT NOT NULL,
  mandate_id TEXT NOT NULL,
  expires_at INTEGER NOT NULL,
  revoked INTEGER NOT NULL DEFAULT 0,
  revoked_at INTEGER,
  signature TEXT NOT NULL,
  created_at INTEGER NOT NULL
);
CREATE INDEX idx_grant_revoked ON context_grant(revoked, expires_at);

CREATE TABLE handoff_packet (
  packet_id TEXT PRIMARY KEY,
  task_id TEXT NOT NULL,
  from_agent_did TEXT NOT NULL,
  to_agent_did TEXT NOT NULL,
  redaction_policy_id TEXT NOT NULL,
  redaction_report_json TEXT NOT NULL,
  audit_chain_head TEXT NOT NULL,
  signature TEXT NOT NULL,
  created_at INTEGER NOT NULL
);

CREATE TABLE audit_event (
  event_id TEXT PRIMARY KEY,
  prev_event_hash BLOB,
  event_hash BLOB NOT NULL,
  tenant_id TEXT NOT NULL,
  session_id TEXT,
  actor_did TEXT NOT NULL,
  operation TEXT NOT NULL,
  target_ref TEXT,
  mandate_id TEXT,
  content_hash BLOB,
  index_version TEXT,
  timestamp INTEGER NOT NULL,
  signature TEXT NOT NULL
);
CREATE INDEX idx_audit_tenant_time ON audit_event(tenant_id, timestamp);
CREATE INDEX idx_audit_actor ON audit_event(actor_did, timestamp);

CREATE TABLE chain_state (
  tenant_id TEXT PRIMARY KEY,
  current_tip BLOB NOT NULL,
  last_verified_at INTEGER,
  last_verified_event TEXT
);

CREATE TABLE dream_proposal (
  proposal_id TEXT PRIMARY KEY,
  tenant_id TEXT NOT NULL,
  status TEXT NOT NULL,                  -- 状态机字段
  risk_level TEXT NOT NULL,
  sbu_blocked INTEGER NOT NULL DEFAULT 0,
  mutations_json TEXT NOT NULL,
  reason TEXT,
  proposer_did TEXT NOT NULL,
  approver_did TEXT,
  approval_audit_event_id TEXT,
  created_at INTEGER NOT NULL,
  expires_at INTEGER NOT NULL,
  signature TEXT NOT NULL
);
CREATE INDEX idx_dream_status ON dream_proposal(tenant_id, status);

CREATE TABLE forget_receipt (
  receipt_id TEXT PRIMARY KEY,
  request_id TEXT UNIQUE NOT NULL,
  mode TEXT NOT NULL,                    -- soft | hard
  target_ids_json TEXT NOT NULL,
  cascaded_ids_json TEXT NOT NULL,
  audit_event_id TEXT NOT NULL,
  completed_at INTEGER NOT NULL,
  signature TEXT NOT NULL
);

CREATE TABLE replay_cache (
  message_id TEXT PRIMARY KEY,
  session_id TEXT NOT NULL,
  expires_at INTEGER NOT NULL
);
CREATE INDEX idx_replay_expires ON replay_cache(expires_at);

CREATE TABLE object_ref (
  object_id TEXT PRIMARY KEY,
  media_type TEXT NOT NULL,
  size_bytes INTEGER NOT NULL,
  sha256 BLOB NOT NULL,
  uri TEXT,
  lease_id TEXT,
  expires_at INTEGER,
  created_at INTEGER NOT NULL
);
```

## 4. 向量索引

| 实现 | 适用 | 备注 |
|---|---|---|
| usearch (HNSW) | 单机 / 边缘 | mmap 友好，单文件 |
| sqlite-vec | Core Lite 极简部署 | 无外部依赖 |
| Qdrant | 服务端 | 多租户、过滤丰富 |
| Milvus | 大规模集群 | 通过 Plugin 适配 |

要求：

```text
add(memory_id, vector, meta)              事务一致
search(query, top_k, filter)               filter 必经 TenantKernel
mark_deleted(memory_id)                    软删除
rebuild(chunks)                            支持 IndexVersion 切换
fork_version()                             双索引并存
```

详见 [09-plugin-runtime.md](./09-plugin-runtime.md)。

## 5. Tag 图存储

```text
hot：内存 HashMap<TagId, HashMap<TagId, weight>>
       Top-N 边常驻（默认 Top-50000）
       支持 V7 有向边

cold：SQLite tag_edge 表
       服务启动时按 weight DESC 加载 Top-N 进 hot
       后台定期压缩低权重边

mmap：CSR 格式快照
       供 Synapse LIF 扩散使用
       避免热路径加锁
```

详见 [tag-graph-v7.md](./tag-graph-v7.md)。

## 6. 对象存储

```text
本地 CAS：
  <root>/objects/<first-2-hex-sha256>/<sha256>
  GC 队列处理过期对象

S3 / MinIO：
  bucket/objects/<sha256>
  租户隔离用 prefix tenant_<id>/
  保留本地热对象缓存
```

## 7. 服务端 / 多租户存储拓扑

```text
PostgreSQL：
  租户、会话、记忆元数据、版本、审计索引
  支持行级锁、partial index、JSON 类型

Object Store (S3/MinIO)：
  大文本、附件、多模态对象

Vector DB (Qdrant/Milvus 适配)：
  HNSW / IVF-PQ
  多租户 collection 隔离

Graph Store：
  Tag 边、实体关系、ontology edge
  小规模可继续用 PG；大规模用 Neo4j 适配

Redis：
  L0 热上下文
  Mandate revocation cache（TTL ≤ 60s）

Kafka / NATS：
  后台任务队列（DreamRun / RebuildIndex）
  审计事件流（fan-out 到归档）
```

## 8. 数据保留策略

```text
L0：默认 24h 后归档（折叠后保留 7 天）
L1：默认 90 天，可按 tenant 调整
L2：默认无限期，依赖 ForgetEngine 触发
audit：默认 ≥ 7 年（合规要求驱动）
context_snapshot：与 grant.expires_at 对齐
context_grant：grant 过期 30 天后清理（保留审计追溯）
```

## 9. 备份与恢复

```text
SQLite：每天 .backup（WAL checkpoint 后）
索引文件：每天 mmap copy（HNSW 文件可直接 cp）
对象存储：CAS 模式天然支持去重备份
审计链：append-only，可增量备份到 S3
```

恢复后必须验证：

```text
chain_state.current_tip 与 audit_event 末尾一致
索引文件版本与 memory_unit.index_version 匹配
ContextGrant.expires_at 重新计算（避免时钟跳变）
```
