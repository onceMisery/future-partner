# 17 - 存储设计

## 1. StoragePlugin

存储本身可插拔：

| Plugin | 说明 |
|---|---|
| SqliteStoragePlugin | SQLite WAL（默认） |
| PostgresStoragePlugin | PostgreSQL |
| MySQLStoragePlugin | MySQL |
| MongoStoragePlugin | MongoDB |

## 2. SQLite 核心表

```sql
CREATE TABLE plugin (
  plugin_id TEXT PRIMARY KEY,
  name TEXT NOT NULL,
  version TEXT NOT NULL,
  plugin_type TEXT NOT NULL,
  manifest_json TEXT NOT NULL,
  signature TEXT,
  enabled INTEGER NOT NULL,
  created_at INTEGER NOT NULL,
  updated_at INTEGER NOT NULL
);

CREATE TABLE capability (
  capability_id TEXT PRIMARY KEY,
  plugin_id TEXT NOT NULL,
  name TEXT NOT NULL,
  version TEXT NOT NULL,
  schema_version TEXT NOT NULL,
  min_fap_version TEXT NOT NULL,
  risk_level TEXT NOT NULL,
  requires_mandate INTEGER NOT NULL,
  supports_batch INTEGER NOT NULL,
  schema_json TEXT NOT NULL,
  updated_at INTEGER NOT NULL
);

CREATE TABLE session (
  session_id TEXT PRIMARY KEY,
  subject TEXT,
  state TEXT NOT NULL,
  hash_chain_tip BLOB,
  created_at INTEGER NOT NULL,
  updated_at INTEGER NOT NULL,
  expires_at INTEGER NOT NULL
);

CREATE TABLE invocation (
  invocation_id TEXT PRIMARY KEY,
  idempotency_key TEXT UNIQUE NOT NULL,
  session_id TEXT NOT NULL,
  capability_id TEXT NOT NULL,
  mandate_id TEXT,
  status TEXT NOT NULL,
  request_hash BLOB NOT NULL,
  response_hash BLOB,
  created_at INTEGER NOT NULL,
  updated_at INTEGER NOT NULL,
  FOREIGN KEY (session_id) REFERENCES session(session_id)
);

CREATE TABLE receipt (
  receipt_id TEXT PRIMARY KEY,
  invocation_id TEXT NOT NULL,
  subject TEXT NOT NULL,
  capability_id TEXT NOT NULL,
  mandate_id TEXT,
  status TEXT NOT NULL,
  finality TEXT NOT NULL,
  request_hash BLOB NOT NULL,
  response_hash BLOB NOT NULL,
  transcript_hash BLOB NOT NULL,
  previous_receipt_hash BLOB,
  signatures_json TEXT NOT NULL,
  created_at INTEGER NOT NULL,
  FOREIGN KEY (invocation_id) REFERENCES invocation(invocation_id)
);

CREATE TABLE receipt_checkpoint (
  checkpoint_id TEXT PRIMARY KEY,
  period_start INTEGER NOT NULL,
  period_end INTEGER NOT NULL,
  merkle_root BLOB NOT NULL,
  receipt_count INTEGER NOT NULL,
  external_log_ref TEXT,
  created_at INTEGER NOT NULL
);

CREATE TABLE mandate (
  mandate_id TEXT PRIMARY KEY,
  parent_mandate_id TEXT,
  issuer TEXT NOT NULL,
  subject TEXT NOT NULL,
  capabilities_json TEXT NOT NULL,
  constraints_json TEXT NOT NULL,
  delegation_depth INTEGER NOT NULL,
  max_delegation_depth INTEGER NOT NULL,
  expires_at INTEGER NOT NULL,
  issuer_signature TEXT NOT NULL,
  revoked INTEGER NOT NULL DEFAULT 0,
  revoked_at INTEGER,
  created_at INTEGER NOT NULL
);

CREATE TABLE replay_cache (
  message_id TEXT PRIMARY KEY,
  session_id TEXT NOT NULL,
  expires_at INTEGER NOT NULL
);
CREATE INDEX idx_replay_cache_expires ON replay_cache(expires_at);

CREATE TABLE object_ref (
  object_id TEXT PRIMARY KEY,
  media_type TEXT NOT NULL,
  size_bytes INTEGER NOT NULL,
  sha256 BLOB NOT NULL,
  origin_node_id TEXT,
  uri TEXT,
  availability TEXT NOT NULL,
  lease_id TEXT,
  created_at INTEGER NOT NULL,
  expires_at INTEGER,
  ttl_seconds INTEGER,
  last_checked_at INTEGER,
  metadata_json TEXT
);

CREATE TABLE object_replica (
  object_id TEXT NOT NULL,
  node_id TEXT NOT NULL,
  status TEXT NOT NULL,
  last_seen_at INTEGER NOT NULL,
  PRIMARY KEY (object_id, node_id)
);

CREATE TABLE object_gc_queue (
  object_id TEXT PRIMARY KEY,
  reason TEXT NOT NULL,
  scheduled_at INTEGER NOT NULL,
  attempts INTEGER NOT NULL DEFAULT 0
);

CREATE TABLE node_registry (
  node_id TEXT PRIMARY KEY,
  node_name TEXT NOT NULL,
  node_version TEXT NOT NULL,
  labels_json TEXT,
  resource_json TEXT,
  health_state TEXT NOT NULL,
  last_heartbeat INTEGER,
  quarantined_until INTEGER,
  created_at INTEGER NOT NULL
);
```

## 3. 向量索引

```text
USearch mmap:       capability 向量（用于 projection 召回）/ memory 向量 / document 向量
sqlite-vec（轻量）: Core Lite 场景下替代 USearch，单文件部署
```

## 4. Object Store

```text
本地 CAS: <root>/objects/<first-2-hex-of-sha256>/<sha256>
S3/MinIO: sha256 作为 object key 的一部分，保留本地热对象缓存
```
