# 05 - 数据模型

> 表结构详见 [13-storage-design.md](./13-storage-design.md)；Protobuf 定义详见 [15-fap1-integration.md](./15-fap1-integration.md)。

## 1. MemoryUnit — 记忆基本单元

```text
MemoryUnit
├── memory_id              UUID v7（含时间戳）
├── tenant_id              租户隔离主键
├── namespace              业务命名空间（项目、团队等）
├── layer                  L0 | L1 | L2
├── visibility             private | project | team | org | foreign
├── owner_subject_did      用户 DID
├── owner_agent_did        Agent DID（产生者）
├── content_ref            内容引用（短内容内嵌；长内容指向 ObjectStore）
├── content_hash           SHA256（用于审计与去重）
├── tags[]                 关联 Tag 列表
├── entities[]             命名实体（用于实体融合）
├── security_labels[]      ["sbu", "secret", "pii", "internal", ...]
├── embedding_model        embedding 模型 ID（用于跨版本路由）
├── index_version          所在索引版本
├── revision               版本号（乐观锁）
├── lineage_parent_ids[]   上游来源（L1→L0、L2→L1）
├── retention_policy       保留策略 ID
├── source_confidence      来源可信度 ∈ [0,1]
├── stability              稳定性 ∈ [0,1]（用于 L2 写入门槛）
├── reusability            可复用性 ∈ [0,1]
├── created_at
├── updated_at
└── deleted_at             tombstone 时间戳（软删除）
```

## 2. Tag — 标签

```text
Tag
├── tag_id                 stable hash（slug 化后哈希）
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
├── allowed_layers         L0 | L1 | L2
├── allowed_ops            read | append | merge
├── redaction_policy_id    脱敏策略 ID
├── expires_at
├── purpose                标准化 purpose（见 purpose-vocabulary.md）
├── mandate_id             绑定 Mandate
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
├── session_id
├── actor_did              操作者 DID
├── operation              memory.retrieve / memory.store / ...
├── target_ref             被操作的 memory_id 或 grant_id
├── mandate_id
├── content_hash
├── index_version
├── timestamp
└── signature
```

哈希构造：

```text
event_hash = SHA256(
  prev_event_hash || tenant_id || session_id ||
  actor_did || operation || target_ref ||
  mandate_id || content_hash || timestamp
)
```

详见 [11-audit-and-receipt.md](./11-audit-and-receipt.md)。

## 10. MemoryVersion — 版本元数据

```text
MemoryVersion
├── memory_id
├── revision               单调递增
├── base_revision          乐观锁基础
├── content_hash
├── embedding_model
├── embedding_version
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
├── mode                   soft | hard
├── target_ids[]
├── cascaded_ids[]         级联清除的下游
├── audit_event_id
├── completed_at
└── signature
```

详见 [forget-engine.md](./forget-engine.md)。
