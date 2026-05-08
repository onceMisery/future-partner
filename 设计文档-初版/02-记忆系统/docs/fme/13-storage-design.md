# 13 - 存储设计

> 数据模型见 [05-data-model.md](./05-data-model.md)；签名对象的 canonical 字段顺序见 [signing-canonical.md](./signing-canonical.md)；chain_state 并发模型见 [chain-state-concurrency.md](./chain-state-concurrency.md)；部署形态选择见 [14-deployment-profiles.md](./14-deployment-profiles.md)。

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
  namespace TEXT NOT NULL,                   -- 与 TenantKernel 注入过滤一致
  layer TEXT NOT NULL,                       -- L0 | L1 | L2 | L3
  visibility TEXT NOT NULL,                  -- private | project | team | org | foreign
  owner_subject_did TEXT,
  owner_agent_did TEXT,
  content_ref TEXT,                          -- inline 或 object_ref
  content_hash BLOB NOT NULL,
  embedding_signature TEXT NOT NULL,         -- model_id|dim|version，详见 09-plugin-runtime
  index_version TEXT NOT NULL,
  revision INTEGER NOT NULL,
  retention_policy TEXT,
  retention_tier TEXT,                       -- 默认 NULL；protected = L2 长期保留（取代旧 CoreMemory 概念）
  source_confidence REAL,
  stability REAL,
  reusability REAL,
  created_at INTEGER NOT NULL,
  updated_at INTEGER NOT NULL,
  deleted_at INTEGER
);
CREATE INDEX idx_memory_tenant_layer ON memory_unit(tenant_id, namespace, layer);
CREATE INDEX idx_memory_visibility ON memory_unit(tenant_id, namespace, visibility);
CREATE INDEX idx_memory_retention_tier ON memory_unit(tenant_id, namespace, retention_tier) WHERE retention_tier IS NOT NULL;
CREATE INDEX idx_memory_deleted ON memory_unit(tenant_id, namespace, deleted_at);

-- tags / entities / lineage_parent_ids 不内嵌于 memory_unit，走以下关联表：
--   memory_tag        ↔ tag
--   memory_entity     ↔ entity（V1 仅占位，详见 05-data-model）
--   memory_lineage    ↔ memory_unit（自关联）

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
CREATE INDEX idx_lineage_parent ON memory_lineage(parent_id);

CREATE TABLE tag (
  tenant_id TEXT NOT NULL,
  namespace TEXT NOT NULL,
  tag_id TEXT NOT NULL,
  tag_name TEXT NOT NULL,
  tag_type TEXT NOT NULL,
  doc_count INTEGER NOT NULL DEFAULT 0,
  avg_cooccur REAL,
  recency REAL,
  intrinsic_residual REAL,               -- V7 内生残差
  PRIMARY KEY (tenant_id, namespace, tag_id),
  UNIQUE (tenant_id, namespace, tag_name)
);

CREATE TABLE memory_tag (
  tenant_id TEXT NOT NULL,
  namespace TEXT NOT NULL,
  memory_id TEXT NOT NULL,
  tag_id TEXT NOT NULL,
  PRIMARY KEY (tenant_id, namespace, memory_id, tag_id)
);
CREATE INDEX idx_memory_tag_tag ON memory_tag(tenant_id, namespace, tag_id);

CREATE TABLE tag_edge (
  tenant_id TEXT NOT NULL,
  namespace TEXT NOT NULL,
  source_tag_id TEXT NOT NULL,
  target_tag_id TEXT NOT NULL,
  weight REAL NOT NULL,                  -- Φ_src × Φ_dst
  source_potential REAL NOT NULL,
  target_potential REAL NOT NULL,
  cooccur_count INTEGER NOT NULL,
  last_seen INTEGER NOT NULL,
  is_hot INTEGER NOT NULL DEFAULT 0,
  PRIMARY KEY (tenant_id, namespace, source_tag_id, target_tag_id)
);
CREATE INDEX idx_tag_edge_hot ON tag_edge(tenant_id, namespace, is_hot, weight DESC);

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
CREATE INDEX idx_fact_spo ON semantic_fact(tenant_id, namespace, subject, predicate);
CREATE INDEX idx_fact_conflict ON semantic_fact(tenant_id, namespace, conflict_state);

CREATE TABLE context_snapshot (
  snapshot_id TEXT PRIMARY KEY,
  tenant_id TEXT NOT NULL,
  namespace TEXT NOT NULL,
  source_session_id TEXT,
  content_hash BLOB NOT NULL,
  source_scope_hash BLOB NOT NULL,
  created_by_agent_did TEXT NOT NULL,
  created_at INTEGER NOT NULL,
  expires_at INTEGER NOT NULL
);
CREATE INDEX idx_snapshot_expiry ON context_snapshot(tenant_id, namespace, expires_at);

CREATE TABLE context_grant (
  grant_id TEXT PRIMARY KEY,
  issuer_did TEXT NOT NULL,
  receiver_agent_did TEXT NOT NULL,
  tenant_id TEXT NOT NULL,
  namespace TEXT NOT NULL,
  allowed_triples_json TEXT NOT NULL,    -- capability/layer/action
  access_mode TEXT NOT NULL,
  redaction_policy_id TEXT NOT NULL,
  redaction_report_id TEXT,
  purpose TEXT NOT NULL,
  mandate_id TEXT NOT NULL,
  fap_receipt_id TEXT NOT NULL,
  expires_at INTEGER NOT NULL,
  revoked INTEGER NOT NULL DEFAULT 0,
  revoked_at INTEGER,
  signature TEXT NOT NULL,
  created_at INTEGER NOT NULL
);
CREATE INDEX idx_grant_receiver ON context_grant(receiver_agent_did, expires_at);
CREATE INDEX idx_grant_revoked ON context_grant(tenant_id, namespace, revoked, expires_at);

CREATE TABLE handoff_packet (
  packet_id TEXT PRIMARY KEY,
  task_id TEXT NOT NULL,
  from_agent_did TEXT NOT NULL,
  to_agent_did TEXT NOT NULL,
  redaction_policy_id TEXT NOT NULL,
  redaction_report_json TEXT NOT NULL,
  audit_chain_head TEXT NOT NULL,
  fap_receipt_id TEXT NOT NULL,
  signature TEXT NOT NULL,
  created_at INTEGER NOT NULL
);

CREATE TABLE audit_event (
  event_id TEXT PRIMARY KEY,
  prev_event_hash BLOB,
  event_hash BLOB NOT NULL,
  tenant_id TEXT NOT NULL,
  namespace TEXT NOT NULL,
  session_id TEXT,
  actor_did TEXT NOT NULL,
  operation TEXT NOT NULL,
  target_ref TEXT,
  mandate_id TEXT,
  fap_receipt_id TEXT NOT NULL,
  fap_invocation_id TEXT NOT NULL,
  fap_envelope_hash BLOB NOT NULL,
  fap_finality TEXT NOT NULL,                -- OPTIMISTIC | VERIFIED | FINALIZED
  capability_triple_json TEXT NOT NULL,
  content_hash BLOB,
  object_ref_hashes_json TEXT,
  index_version TEXT,
  timestamp INTEGER NOT NULL,                -- Unix 微秒，与 signing-canonical 一致
  signature TEXT NOT NULL
);
CREATE INDEX idx_audit_tenant_time ON audit_event(tenant_id, namespace, timestamp);
CREATE INDEX idx_audit_fap_receipt ON audit_event(fap_receipt_id);
CREATE INDEX idx_audit_actor ON audit_event(actor_did, timestamp);

CREATE TABLE chain_state (
  tenant_id TEXT NOT NULL,
  namespace TEXT NOT NULL,
  current_tip BLOB NOT NULL,
  revision INTEGER NOT NULL,                 -- 单调递增，CAS 比较的 base
  last_verified_at INTEGER,
  last_verified_event TEXT,
  PRIMARY KEY (tenant_id, namespace)
);
-- 并发更新规则、SchemaPerTenant / DbPerTenant 下的全局注册中心见 chain-state-concurrency.md

CREATE TABLE dream_proposal (
  proposal_id TEXT PRIMARY KEY,
  tenant_id TEXT NOT NULL,
  status TEXT NOT NULL,                  -- PENDING_REVIEW | AWAITING_APPROVAL | APPROVED |
                                         -- APPLYING | APPLIED | APPLY_FAILED |
                                         -- REJECTED | EXPIRED | BLOCKED
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
  mode TEXT NOT NULL,                          -- SOFT | HARD
  status TEXT NOT NULL,                        -- COMMITTED_LOCALLY | GLOBALLY_RECONCILED | RECONCILE_FAILED

  -- 阶段一
  committed_locally_at INTEGER NOT NULL,
  locally_committed_mutations_json TEXT NOT NULL,
  target_ids_json TEXT NOT NULL,
  cascaded_ids_json TEXT NOT NULL,
  local_signature TEXT NOT NULL,

  -- 阶段二（GLOBALLY_RECONCILED 时填）
  globally_reconciled_at INTEGER,
  external_evidence_json TEXT,
  reconciled_signature TEXT,

  -- 进行中
  pending_external_systems_json TEXT,
  reconciliation_deadline INTEGER,

  -- 失败（RECONCILE_FAILED 时填）
  intervention_reason TEXT,
  failed_mutations_json TEXT,

  -- 关联
  mutation_ledger_id TEXT,
  audit_event_id TEXT NOT NULL,
  fap_receipt_id TEXT NOT NULL,

  -- 取消语义
  cancelled_after_local_commit INTEGER NOT NULL DEFAULT 0
);
CREATE INDEX idx_forget_receipt_status ON forget_receipt(status, reconciliation_deadline);

CREATE TABLE mutation_ledger (
  ledger_id TEXT PRIMARY KEY,
  request_id TEXT NOT NULL,                    -- ForgetRequest 幂等键
  receipt_id TEXT,                             -- ForgetReceipt（dream/admin 类可空）
  tenant_id TEXT NOT NULL,                     -- 租户隔离
  namespace TEXT NOT NULL,
  mutation_kind TEXT NOT NULL,                 -- forget.vector_delete | forget.s3_delete | forget.kafka_emit
                                               -- forget.context_grant_revoke | forget.handoff_advisory
                                               -- dream.* | admin.*
  target_system TEXT NOT NULL,                 -- qdrant_prod | s3_us_east | kafka_audit | did:web:agent-b ...
  payload_json TEXT NOT NULL,                  -- 幂等执行参数
  idempotency_key TEXT NOT NULL,               -- 外部系统幂等键
  status TEXT NOT NULL,                        -- PENDING | IN_FLIGHT | SUCCESS
                                               -- NEEDS_INTERVENTION | ADVISORY_DELIVERED
  attempts INTEGER NOT NULL DEFAULT 0,
  last_error TEXT,
  scheduled_at INTEGER NOT NULL,
  started_at INTEGER,
  completed_at INTEGER,
  next_retry_at INTEGER,
  signature TEXT NOT NULL,                     -- mutation 描述的本地签名（防 tampering）
  shard_key INTEGER NOT NULL,                  -- (tenant_id, mutation_kind) → shard，用于 reconciler 分片
  created_at INTEGER NOT NULL,
  updated_at INTEGER NOT NULL,
  UNIQUE (target_system, idempotency_key)
);
CREATE INDEX idx_mutation_ledger_status ON mutation_ledger(status, next_retry_at);
CREATE INDEX idx_mutation_ledger_receipt ON mutation_ledger(receipt_id);
CREATE INDEX idx_mutation_ledger_shard ON mutation_ledger(shard_key, status);
CREATE INDEX idx_mutation_ledger_tenant ON mutation_ledger(tenant_id, namespace);

CREATE TABLE replay_cache (
  message_id TEXT PRIMARY KEY,
  session_id TEXT NOT NULL,
  expires_at INTEGER NOT NULL
);
CREATE INDEX idx_replay_expires ON replay_cache(expires_at);

CREATE TABLE object_ref (
  object_id TEXT PRIMARY KEY,
  tenant_id TEXT NOT NULL,
  namespace TEXT NOT NULL,
  media_type TEXT NOT NULL,
  size_bytes INTEGER NOT NULL,
  sha256 BLOB NOT NULL,
  uri TEXT,
  origin_node_id TEXT,
  lease_id TEXT,
  lease_scope_hash BLOB,
  expires_at INTEGER,
  created_at INTEGER NOT NULL
);
CREATE INDEX idx_object_tenant ON object_ref(tenant_id, namespace, expires_at);
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
add(memory_id, vector, meta)              本地索引与本地 DB 同事务；外部索引走 outbox
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
  bucket/tenants/<tenant_id>/namespaces/<namespace>/objects/<sha256>
  ObjectRef 必须绑定 lease_scope_hash 与 origin_node_id
  保留本地热对象缓存
```

ObjectRef 解析必须由 ObjectStore adapter 完成，禁止插件直接解析任意 `s3://` 或 `file://` URI。

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
