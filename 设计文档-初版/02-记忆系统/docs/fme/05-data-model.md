# 05 - 数据模型

> 表结构详见 [13-storage-design.md](./13-storage-design.md)；签名对象的 canonical 字段顺序与 hash/签名规则详见 [signing-canonical.md](./signing-canonical.md)；Protobuf 定义详见 [15-fap1-integration.md](./15-fap1-integration.md)。

## 1. MemoryUnit — 记忆基本单元

```text
MemoryUnit
├── memory_id              UUID v7（含时间戳）
├── tenant_id              租户隔离主键
├── namespace              业务命名空间（项目、团队等）
├── layer                  L0 | L1 | L2 | L3
├── visibility             private | project | team | org | foreign
├── owner_subject_did      用户 DID
├── owner_agent_did        Agent DID（产生者）
├── content_ref            内容引用（短内容内嵌；长内容指向 ObjectStore）
├── content_hash           SHA256（用于审计与去重）
├── tags[]                 关联 Tag 列表（关联表 memory_tag）
├── entities[]             命名实体（关联表 memory_entity，V1 仅占位）
├── security_labels[]      ["sbu", "secret", "pii", "internal", ...]
├── embedding_signature    "model_id|dim|version" 三元组（详见 09-plugin-runtime EmbeddingProvider）
├── index_version          所在索引版本
├── revision               版本号（乐观锁）
├── lineage_parent_ids[]   上游来源（关联表 memory_lineage：L1→L0、L2→L1）
├── retention_policy       保留策略 ID
├── retention_tier         protected | standard | NULL（取代旧 CoreMemory 概念）
├── source_confidence      来源可信度 ∈ [0,1]
├── stability              稳定性 ∈ [0,1]（用于 L2 写入门槛）
├── reusability            可复用性 ∈ [0,1]
├── created_at
├── updated_at
└── deleted_at             tombstone 时间戳（软删除）
```

字段持久化方式：

```text
inline                     上述字段除 tags / entities / lineage_parent_ids / security_labels 外，
                           直接持久化在 memory_unit 表（详见 13-storage-design）

关联表                      tags         → memory_tag
                           entities     → memory_entity（V1 占位）
                           lineage      → memory_lineage
                           security     → memory_unit_security
```

## 2. Tag — 标签

```text
Tag
├── tag_id                 stable hash（slug 化后哈希）
├── tenant_id
├── namespace
├── tag_name               规范化后的标签名
├── tag_type               topic | entity | intent | technique | other
├── doc_count              拥有此标签的 MemoryUnit 数（用于引力场）
├── avg_cooccur            平均共现度
├── recency                最近活跃时间衰减
├── tag_centroid_vector    所有携带此 Tag 的 MemoryUnit 向量平均（用于 Tide）
└── intrinsic_residual     V7 内生残差（截断 SVD 残差能量）
```

引力场质量公式：

```text
mass = √doc_count × avg_cooccur × recency
```

详见 [tide-algorithm.md](./tide-algorithm.md)。

## 3. Tag 共现边（V7 有向序位）

```text
TagEdge
├── tenant_id
├── namespace
├── source_tag_id          源 Tag
├── target_tag_id          目标 Tag
├── weight                 共现权重（势能积 Φ_src × Φ_dst）
├── direction              source → target（V7 有向）
├── source_potential       Φ_src ∈ [0.5, 0.9]（源在日记中位置势能）
├── target_potential       Φ_dst ∈ [0.5, 0.9]
├── cooccur_count          共现次数
├── last_seen              最近共现时间
└── is_hot                 是否在热图中
```

详见 [tag-graph-v7.md](./tag-graph-v7.md)。

## 4. Episode — 剧集（L1 单元）

```text
Episode
├── episode_id
├── tenant_id
├── namespace
├── title                  自动生成或用户命名
├── summary                LLM 摘要
├── start_time / end_time
├── chunk_ids[]            包含的块（content + embedding）
├── related_tag_ids[]
├── related_entity_ids[]
├── lineage_parent_ids[]   上游 L0 事件
├── confidence_score
└── source_session_id
```

## 5. SemanticFact — L2 事实

```text
SemanticFact
├── fact_id
├── tenant_id
├── namespace
├── subject                可指代的实体或上下文
├── predicate              关系或属性名
├── object                 关系/属性值
├── confidence             置信度 ∈ [0,1]
├── stability              稳定性 ∈ [0,1]
├── reusability            可复用性 ∈ [0,1]
├── source_confidence
├── lineage_parent_ids[]
├── revision
├── conflict_state         active | superseded | forked | rejected
├── superseded_by          冲突仲裁后的接替 fact_id
└── last_validated_at
```

冲突仲裁见 [retain-score.md](./retain-score.md)。

## 6. ContextSnapshot — 上下文快照

```text
ContextSnapshot
├── snapshot_id
├── tenant_id
├── namespace
├── source_session_id
├── content_hash           snapshot 内容哈希（用于 grant 绑定）
├── source_scope_hash      RedactionReport 覆盖范围哈希
├── memory_unit_ids[]      包含的 MemoryUnit
├── created_by_agent_did
├── created_at
└── expires_at
```

snapshot 不可变；任何修改产生新 snapshot。

详见 [12-context-sharing.md](./12-context-sharing.md)。

## 7. ContextGrant — 共享凭证

```text
ContextGrant
├── grant_id
├── issuer_did
├── receiver_agent_did
├── tenant_id
├── namespace
├── snapshot_ids[]         绑定的 snapshot
├── allowed_triples[]      capability/layer/action 精确匹配
├── access_mode            READ_ONLY | IMPORT_FOREIGN_EPISODE |
│                          APPEND_PROJECT_L1 | MERGE_L2
├── redaction_policy_id    脱敏策略 ID
├── redaction_report_id
├── expires_at
├── purpose                标准化 purpose（见 purpose-vocabulary.md）
├── mandate_id             绑定 FAP-1 Mandate
├── fap_receipt_id
└── signature              发行方签名（JWS）
```

## 8. HandoffPacket — Agent 间任务转交包

```text
HandoffPacket
├── packet_id
├── task_id
├── from_agent_did
├── to_agent_did
├── goal
├── current_state_summary
├── completed_steps[]
├── failed_attempts[]
├── open_questions[]
├── constraints[]
├── tool_state_refs[]
├── l0_snapshot_ref
├── l1_episode_refs[]
├── l2_fact_refs[]
├── context_grant
├── sbu_redaction_report   RedactionReport（含可验证签名）
├── redaction_policy_id
├── audit_chain_head       接收方可基于此追溯审计
└── packet_signature
```

详见 [12-context-sharing.md](./12-context-sharing.md)、[redaction-policy.md](./redaction-policy.md)。

## 9. AuditEvent — 审计事件

```text
AuditEvent
├── event_id
├── prev_event_hash        哈希链前向引用
├── event_hash             本事件哈希
├── tenant_id
├── namespace
├── session_id
├── actor_did              操作者 DID
├── operation              memory.retrieve / memory.store / memory.share
│                          memory.handoff.create / memory.handoff.receive
│                          memory.forget.soft / memory.forget.hard / ...
├── target_ref             被操作的 memory_id 或 grant_id
├── mandate_id
├── fap_receipt_id
├── fap_invocation_id      == FAP-1 Envelope.message_id
├── fap_envelope_hash
├── fap_finality           OPTIMISTIC | VERIFIED | FINALIZED
├── capability_triple
├── content_hash
├── object_ref_hashes[]
├── index_version
├── timestamp_micros       Unix 微秒
└── signature
```

`event_hash` 输入字段顺序与 canonical 编码规则在 [signing-canonical.md §2](./signing-canonical.md) 冻结，本文档不重复定义；任何实现按本文示例的字段顺序拼接 hash 都将失败。

详见 [11-audit-and-receipt.md](./11-audit-and-receipt.md)。

## 10. MemoryVersion — 版本元数据

```text
MemoryVersion
├── memory_id
├── revision               单调递增
├── base_revision          乐观锁基础
├── content_hash
├── embedding_signature    "model_id|dim|version"（取代旧 embedding_model + embedding_version 双字段）
├── index_version
├── tag_graph_version
├── ontology_version
└── lineage_parent_ids[]
```

## 11. DreamProposal — 梦境提案

```text
DreamProposal
├── proposal_id
├── tenant_id
├── status                 PENDING_REVIEW / AWAITING_APPROVAL / APPROVED /
│                          APPLYING / APPLIED / REJECTED / EXPIRED / BLOCKED
├── risk_level             LOW | MEDIUM | HIGH
├── sbu_blocked            true 时永久阻断
├── mutations[]            建议的变更
├── reason                 LLM/规则解释
├── created_at
├── expires_at             默认 72h
└── approver_did           审批人（若需）
```

详见 [dream-state-machine.md](./dream-state-machine.md)。

## 12. ForgetReceipt — 遗忘凭证

```text
ForgetReceipt
├── receipt_id
├── request_id
├── mode                       SOFT | HARD
├── status                     COMMITTED_LOCALLY | GLOBALLY_RECONCILED |
│                              RECONCILE_FAILED
├── 阶段一（必填）
│   ├── committed_locally_at_micros
│   ├── locally_committed_mutations[]
│   ├── target_ids[]
│   ├── cascaded_ids[]
│   └── local_signature
├── 阶段二（GLOBALLY_RECONCILED 时填）
│   ├── globally_reconciled_at_micros
│   ├── external_evidence[]              SystemReconcileEvidence
│   └── reconciled_signature
├── 进行中
│   ├── pending_external_systems[]
│   └── reconciliation_deadline_micros
├── 失败（RECONCILE_FAILED 时填）
│   ├── intervention_reason
│   └── failed_mutations[]
├── 关联
│   ├── mutation_ledger_id
│   ├── audit_event_id
│   └── fap_receipt_id
└── cancelled_after_local_commit         bool
```

权威 schema、双签名（`local_signature` + `reconciled_signature`）规则见 [signing-canonical.md §3](./signing-canonical.md)；语义与流程见 [forget-engine.md](./forget-engine.md) 与 [outbox-reconciliation.md](./outbox-reconciliation.md)。
