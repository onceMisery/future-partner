# signing-canonical.md

> FAP-ME 所有签名对象的 canonical encoding 总锚。AuditEvent / ForgetReceipt / RedactionReport / ContextGrant / HandoffPacket / DreamProposal 必须按本文档的字段顺序与编码规则参与 hash / 签名。任何章节示例与本文不一致时，**以本文为准**。

## 0. 设计原则

```text
1. 单一权威：每个被签名对象的 canonical encoding 在本文定义一次
2. 字段顺序固定：跨实现互通的前提
3. 字段不缺省：缺省字段必须以确定性占位符（zero / empty）参与编码
4. 与 FAP-1 对齐：复用 FAP-1 [signing-canonical.md] 的 canonical 规则与测试向量
5. 演进只能加字段，不能改顺序：新增字段追加在末尾并升 schema_version
```

参考：FAP-1 [signing-canonical.md](../../../01-协议设计/docs/fap/signing-canonical.md)。

## 1. Canonical 编码规则

```text
字符串            UTF-8，无 BOM
整数              i64 大端 8 字节
浮点              IEEE-754 binary64 大端 8 字节
时间戳            int64 Unix 微秒大端 8 字节
布尔              0x00 / 0x01
字节串            原始字节
集合              排序后逐项 canonical 编码 + 拼接
optional 缺省     长度 0 字节
分隔符            ASCII 0x1F (Unit Separator) 拼接同级字段
列表元素分隔      ASCII 0x1E (Record Separator)
```

具体语义见 FAP-1。下文 `||` 即按上述规则拼接。

## 2. AuditEvent

### 2.1 Schema（权威）

```proto
message AuditEvent {
  string  event_id            = 1;   // UUID v7
  bytes   prev_event_hash     = 2;
  bytes   event_hash          = 3;

  string  tenant_id           = 4;
  string  namespace           = 5;
  string  session_id          = 6;
  string  actor_did           = 7;
  string  operation           = 8;
  string  target_ref          = 9;
  string  mandate_id          = 10;

  string  fap_receipt_id      = 11;
  string  fap_invocation_id   = 12;   // == FAP-1 Envelope.message_id
  bytes   fap_envelope_hash   = 13;
  string  fap_finality        = 14;   // OPTIMISTIC | VERIFIED | FINALIZED

  bytes   capability_triple   = 15;   // canonical(triple)
  bytes   content_hash        = 16;
  bytes   object_ref_hashes   = 17;   // canonical(sorted(object_ref_hashes[]))
  string  index_version       = 18;

  int64   timestamp_micros    = 19;

  bytes   signature           = 20;   // AuditKernel signature on event_hash
}
```

### 2.2 hash 输入字段顺序（权威）

```text
event_hash = SHA256(
  prev_event_hash                         ||
  tenant_id           || namespace        ||
  session_id                              ||
  actor_did           || operation        ||
  target_ref          || mandate_id       ||
  fap_receipt_id      || fap_invocation_id||
  fap_envelope_hash   || fap_finality     ||
  canonical(capability_triple)            ||
  content_hash                            ||
  canonical(object_ref_hashes)            ||
  index_version                           ||
  timestamp_micros
)
```

### 2.3 fap_finality 词汇表（与 FAP-1 对齐）

```text
OPTIMISTIC   单签即完成（仅检索类）
VERIFIED     单签 + AuditSink 持久化（写入类默认）
FINALIZED    Multisig + 协议级 Receipt 最终化（高风险类默认）
```

任何 FME 文档不得使用 `PROVISIONAL` 词汇。已有 `PROVISIONAL` 出现的位置应解释为 `OPTIMISTIC`。

### 2.4 capability_triple canonical

```text
canonical(triple) = canonical(MemoryCapability_enum_int)
                 || canonical(MemoryLayer_enum_int)
                 || canonical(MemoryAction_enum_int)
```

枚举值见 [purpose-vocabulary.md §3](./purpose-vocabulary.md)。

### 2.5 签名

```text
signature = JWS(
  alg = EdDSA,
  kid = "audit-kernel#" || tenant_id || "#" || namespace,
  payload = event_hash
)
```

`AuditKernel` 是 Kernel 唯一签名者；插件不得签发。

## 3. ForgetReceipt

### 3.1 Schema（权威）

```proto
enum ForgetReceiptStatus {
  COMMITTED_LOCALLY     = 0;
  GLOBALLY_RECONCILED   = 1;
  RECONCILE_FAILED      = 2;
}

enum ForgetMode {
  SOFT = 0;
  HARD = 1;
}

message ForgetReceipt {
  string  receipt_id                     = 1;
  string  request_id                     = 2;
  ForgetMode mode                        = 3;
  ForgetReceiptStatus status             = 4;

  // 阶段一（必填）
  int64   committed_locally_at_micros    = 5;
  repeated string locally_committed_mutations = 6;
  repeated string target_ids             = 7;
  repeated string cascaded_ids           = 8;
  bytes   local_signature                = 9;

  // 阶段二（reconciled 时填）
  optional int64 globally_reconciled_at_micros = 10;
  repeated SystemReconcileEvidence external_evidence = 11;
  optional bytes reconciled_signature    = 12;

  // 进行中
  repeated string pending_external_systems = 13;
  optional int64 reconciliation_deadline_micros = 14;

  // 失败
  optional string intervention_reason    = 15;
  repeated FailedMutation failed_mutations = 16;

  // 关联
  string  mutation_ledger_id             = 17;
  string  audit_event_id                 = 18;
  string  fap_receipt_id                 = 19;

  // 取消语义（P2-9）
  bool    cancelled_after_local_commit   = 20;
}

message SystemReconcileEvidence {
  string  system          = 1;     // qdrant_prod / s3_us_east / kafka_audit
  string  evidence_token  = 2;     // 外部系统返回的删除凭证（如 S3 delete marker）
  int64   at_micros       = 3;
}

message FailedMutation {
  string  ledger_id       = 1;
  string  target_system   = 2;
  string  last_error      = 3;
  uint32  attempts        = 4;
}
```

### 3.2 双签名

`ForgetReceipt` 是双状态对象，两段签名分别覆盖两个阶段：

```text
local_signature = JWS(
  alg = EdDSA, kid = audit-kernel-key,
  payload = SHA256(
    receipt_id || request_id ||
    canonical(mode) || canonical(COMMITTED_LOCALLY) ||
    committed_locally_at_micros ||
    canonical(target_ids) ||
    canonical(cascaded_ids) ||
    canonical(locally_committed_mutations) ||
    mutation_ledger_id || audit_event_id || fap_receipt_id
  )
)

reconciled_signature = JWS(
  alg = EdDSA, kid = audit-kernel-key,
  payload = SHA256(
    receipt_id || request_id ||
    canonical(GLOBALLY_RECONCILED) ||
    globally_reconciled_at_micros ||
    canonical(external_evidence) ||
    canonical(failed_mutations)
  )
)
```

接收方先验 `local_signature`，再（若状态 = `GLOBALLY_RECONCILED`）验 `reconciled_signature`。任一失败 → 拒绝。

## 4. RedactionReport

### 4.1 Schema（权威）

以 [redaction-policy.md §8](./redaction-policy.md) 为基础。canonical 字段顺序：

```text
report_id                 ||
schema                    ||         // "fap-redaction-report-v1"
policy_id                 ||
policy_version            ||
applied_at_micros         ||
executor_did              ||
source_scope_hash         ||
canonical(sbu_manifest)   ||
canonical(actions_taken[])||
canonical(audit_sample_hashes[]) ||
sbu_units_removed         ||
sbu_tombstones_included   ||
total_units_before        ||
total_units_after
```

### 4.2 sbu_manifest 子结构

```text
classifier_id             ||
classifier_version        ||
scan_coverage             ||         // full | sampled | partial
sample_rate               ||
patterns_count_scanned    ||
notes_canonical                       // 缺省 = 0 字节
```

### 4.3 actions_taken[] 子结构

每项：

```text
rule_index                ||
count                     ||
canonical(memory_ids_hashed[])  ||   // 仅 HASH/REMOVE 命中
canonical(fields_affected[])    ||   // 仅 HASH 命中
action_kind                            // REMOVE | HASH | REDACT | SUMMARIZE | MASK | KEEP
```

### 4.4 audit_sample_hashes[] 子结构

```text
memory_id_hash    ||
decision                              // removed | kept | redacted
```

### 4.5 签名

```text
report_signature = JWS(
  alg = EdDSA, kid = executor_did + "#key-1",
  payload = SHA256(canonical(report 除 report_signature 字段))
)
```

## 5. ContextGrant

```proto
message ContextGrant {
  string  grant_id                = 1;
  string  issuer_did              = 2;
  string  receiver_agent_did      = 3;
  string  tenant_id               = 4;
  string  namespace               = 5;
  repeated string snapshot_ids    = 6;
  bytes   allowed_triples         = 7;     // canonical(sorted(triples[]))
  string  access_mode             = 8;     // READ_ONLY | IMPORT_FOREIGN_EPISODE | APPEND_PROJECT_L1 | MERGE_L2
  string  redaction_policy_id     = 9;
  string  redaction_report_id     = 10;
  string  purpose                 = 11;    // PurposeRegistry id
  string  mandate_id              = 12;
  string  fap_receipt_id          = 13;
  int64   expires_at_micros       = 14;
  int64   created_at_micros       = 15;
  bytes   signature               = 16;
}
```

签名输入按字段 1-15 顺序拼接。

## 6. HandoffPacket

```proto
message HandoffPacket {
  string  packet_id                       = 1;
  string  task_id                         = 2;
  string  from_agent_did                  = 3;
  string  to_agent_did                    = 4;

  string  goal                            = 5;
  string  current_state_summary           = 6;
  repeated string completed_steps         = 7;
  repeated string failed_attempts         = 8;
  repeated string open_questions          = 9;
  repeated string constraints             = 10;
  repeated string tool_state_refs         = 11;

  string  l0_snapshot_ref                 = 12;
  repeated string l1_episode_refs         = 13;
  repeated string l2_fact_refs            = 14;

  ContextGrant context_grant              = 15;
  RedactionReport sbu_redaction_report    = 16;
  string  redaction_policy_id             = 17;
  string  audit_chain_head                = 18;
  string  fap_receipt_id                  = 19;

  bytes   packet_signature                = 20;
}
```

签名输入：

```text
packet_signature = JWS(
  alg = EdDSA, kid = from_agent_did + "#key-1",
  payload = SHA256(
    canonical(packet 除 packet_signature 字段，按字段编号顺序)
  )
)
```

`context_grant.signature` 与 `sbu_redaction_report.report_signature` 在 packet 内部已经各自签名，验证时分别独立校验。

## 7. DreamProposal

```proto
message DreamProposal {
  string  proposal_id                 = 1;
  string  tenant_id                   = 2;
  string  namespace                   = 3;
  DreamProposalStatus status          = 4;
  RiskLevel risk_level                = 5;
  bool    sbu_blocked                 = 6;

  bytes   mutations                   = 7;     // canonical(MemoryMutation[])
  string  reason                      = 8;
  string  proposer_did                = 9;
  optional string approver_did        = 10;
  optional string approval_audit_event_id = 11;
  string  fap_receipt_id              = 12;

  int64   created_at_micros           = 13;
  int64   expires_at_micros           = 14;

  bytes   signature                   = 15;     // 由 Kernel 签发
}
```

## 8. CapabilityTriple canonical

```proto
message MemoryCapabilityTriple {
  MemoryCapability capability = 1;
  MemoryLayer      layer      = 2;
  MemoryAction     action     = 3;
}
```

```text
canonical(triple) = i32_be(capability_enum) ||
                    i32_be(layer_enum)      ||
                    i32_be(action_enum)
```

枚举值固定（不可重排）见 [purpose-vocabulary.md §3](./purpose-vocabulary.md)：

```text
MemoryCapability:
  0 MEMORY_CAPABILITY_UNSPECIFIED
  1 MEMORY_RETRIEVE
  2 MEMORY_STORE
  3 MEMORY_SHARE
  4 MEMORY_HANDOFF
  5 MEMORY_FORGET
  6 MEMORY_DREAM
  7 MEMORY_AUDIT
  8 MEMORY_ADMIN

MemoryLayer:
  0 MEMORY_LAYER_UNSPECIFIED
  1 L0
  2 L1
  3 L2
  4 L3
  5 AUDIT_CHAIN

MemoryAction:
  0  MEMORY_ACTION_UNSPECIFIED
  1  READ
  2  WRITE
  3  SHARE_CREATE
  4  HANDOFF_CREATE
  5  HANDOFF_RECEIVE
  6  DELETE_SOFT
  7  DELETE_HARD
  8  PROPOSE
  9  APPROVE
  10 REJECT
  11 QUERY
  12 ADMIN
```

## 9. capability_id 全表（canonical 字符串）

签名涉及的 `Mandate.capabilities[]` 字符串值必须**逐字**取以下集合，不允许变体：

```text
memory.retrieve
memory.store
memory.share
memory.context.redeem
memory.handoff.create
memory.handoff.receive
memory.forget.soft
memory.forget.hard
memory.audit.query
memory.dream.propose
memory.dream.approve
memory.admin.rebuild
```

不存在 `memory.handoff` 单串、不存在 `memory.forget` 单串。已有文档示例使用单串的位置一律按 `.create / .receive` 或 `.soft / .hard` 替换。

### 9.1 capability_id ↔ CapabilityTriple 映射

`PolicyKernel` 在 authorize 阶段从 `capability_id + request payload` 推导 triple；下表是允许出现的 triple（其它推导一律拒绝）：

| capability_id | (capability, layer, action) |
|---|---|
| `memory.retrieve` | `(MEMORY_RETRIEVE, L0/L1/L2/L3, READ)` |
| `memory.store` | `(MEMORY_STORE, L0/L1/L2, WRITE)` |
| `memory.share` | `(MEMORY_SHARE, MEMORY_LAYER_UNSPECIFIED, SHARE_CREATE)` |
| `memory.context.redeem` | `(MEMORY_SHARE, L0/L1/L2, READ)` |
| `memory.handoff.create` | `(MEMORY_HANDOFF, MEMORY_LAYER_UNSPECIFIED, HANDOFF_CREATE)` |
| `memory.handoff.receive` | `(MEMORY_HANDOFF, MEMORY_LAYER_UNSPECIFIED, HANDOFF_RECEIVE)` |
| `memory.forget.soft` | `(MEMORY_FORGET, L0/L1/L2 或 UNSPECIFIED, DELETE_SOFT)` |
| `memory.forget.hard` | `(MEMORY_FORGET, L0/L1/L2 或 UNSPECIFIED, DELETE_HARD)` |
| `memory.audit.query` | `(MEMORY_AUDIT, AUDIT_CHAIN, QUERY)` |
| `memory.dream.propose` | `(MEMORY_DREAM, MEMORY_LAYER_UNSPECIFIED, PROPOSE)` |
| `memory.dream.approve` | `(MEMORY_DREAM, MEMORY_LAYER_UNSPECIFIED, APPROVE)` |
| `memory.admin.rebuild` | `(MEMORY_ADMIN, MEMORY_LAYER_UNSPECIFIED, ADMIN)` |

`memory.share` 与 `memory.context.redeem` 共享 `MEMORY_SHARE` capability：发起共享是 `SHARE_CREATE`，接收方读取已发的 grant 是 `READ`。

不允许的 triple 推导（举例）：

```text
capability_id = memory.forget.soft  → action = DELETE_HARD       拒绝
capability_id = memory.audit.query  → layer  = L0                拒绝
capability_id = memory.handoff.create → action = HANDOFF_RECEIVE 拒绝
```

## 10. 枚举字符串到 enum 的映射（备查）

便于 JSON / Mandate constraint `value` 字段使用：

| 字符串 | 数字 |
|---|---|
| `MEMORY_RETRIEVE` | 1 |
| `MEMORY_STORE` | 2 |
| `MEMORY_SHARE` | 3 |
| `MEMORY_HANDOFF` | 4 |
| `MEMORY_FORGET` | 5 |
| `MEMORY_DREAM` | 6 |
| `MEMORY_AUDIT` | 7 |
| `MEMORY_ADMIN` | 8 |
| `L0` | 1 |
| `L1` | 2 |
| `L2` | 3 |
| `L3` | 4 |
| `AUDIT_CHAIN` | 5 |
| `MEMORY_LAYER_UNSPECIFIED` | 0 |
| `READ` | 1 |
| `WRITE` | 2 |
| `SHARE_CREATE` | 3 |
| `HANDOFF_CREATE` | 4 |
| `HANDOFF_RECEIVE` | 5 |
| `DELETE_SOFT` | 6 |
| `DELETE_HARD` | 7 |
| `PROPOSE` | 8 |
| `APPROVE` | 9 |
| `REJECT` | 10 |
| `QUERY` | 11 |
| `ADMIN` | 12 |

JSON 编码必须使用字符串名（避免大小端误解）；数字仅在 canonical 二进制编码中使用。

## 11. 测试向量（Phase 1 Gate 4 验收）

实现必须发布 conformance 测试向量集，至少覆盖：

```text
1. AuditEvent：5 条样本（含 namespace 缺省 / object_ref_hashes 空 / index_version 缺省）
2. ForgetReceipt：3 条样本（仅本地 / 已全局协调 / 协调失败）
3. RedactionReport：3 条样本（cross-tenant / within-tenant / sampled coverage）
4. ContextGrant：2 条样本（同租户 / 跨租户）
5. HandoffPacket：2 条样本
6. DreamProposal：3 条样本（PENDING_REVIEW / APPROVED / BLOCKED）
```

测试向量随 SDK 发布，跨实现必须 byte-for-byte 一致。

## 12. 演进规则

```text
增加字段：必须追加在已有字段编号之后，并升 schema 版本（Mandate constraint memory.canonical_version_min 据此校验）
删除字段：禁止
重排字段：禁止
改变枚举值整数：禁止（枚举仅可追加新值）
```

## 13. 与最终版的差距

```text
最终版                                  本文档
-----                                   -----
canonical 散布在多份文档                 集中至本文档；其它文档仅引用
ForgetReceipt 单签                       双状态双签
AuditEvent fap_finality 词汇分裂        统一 FAP-1 三档
capability_id 含 memory.handoff 单串    拆为 .create / .receive
RedactionReport JSON / proto 字段不一致  本文档统一 canonical 字段顺序
枚举值整数未冻结                         本文档显式列出
```
