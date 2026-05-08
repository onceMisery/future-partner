# 15 - FAP-1 协议集成

> 本章描述 FAP-ME 与 FAP-1 协议层的集成点。FAP-1 总览见 [00-overview](../../../01-协议设计/docs/fap/00-overview.md)。

## 1. 集成原则

```text
FAP-ME 不重新发明协议
FAP-ME 复用 FAP-1：
  Discovery（Agent Card）
  Session（QUIC/H3）
  Auth（mTLS / JWT / DID / VC / DPoP）
  Control Plane（Protobuf）
  Data Plane（流式 Frame + ObjectRef）
  Receipt 链（FAP-1 Multisig Receipt）

FAP-ME 只在协议外加：
  memory.* capability
  Agent Card 中的 memory 子节
  Memory 专属 Protobuf（含 RedactionPolicy / DreamProposal / StandardPurpose）
  FAP-ME Kernel（不与 FAP-1 Kernel 重叠，但服从其 Kernel Contract）
```

## 2. Agent Card 扩展

```json
{
  "agent_id": "did:web:agent.example",
  "protocol": {
    "name": "FAP-1",
    "versions": ["1.0"],
    "transports": ["h3", "grpc-over-h3"],
    "control_encoding": ["protobuf"],
    "data_stream": ["fap-frame-stream"]
  },
  "memory": {
    "extension": "FAP-ME",
    "version": "1.0",
    "layers": ["L0", "L1", "L2", "Subconscious"],
    "ops": [
      "memory.retrieve",
      "memory.store",
      "memory.share",
      "memory.forget",
      "memory.handoff",
      "memory.audit",
      "memory.dream.propose",
      "memory.dream.approve"
    ],
    "retrieval_modes": [
      "basic",
      "hybrid",
      "tide",
      "tagmemo",
      "tagmemo_geodesic",
      "dream",
      "sbu_safe"
    ],
    "purpose_registry_version": "1.0",
    "redaction_policy_schema": "fap-redaction-v1",
    "sbu_redaction_report_schema": "fap-redaction-report-v1",
    "handoff": {
      "supported": true,
      "requires_context_grant": true,
      "sbu_report_required": true,
      "redaction_report_verifiable": true
    },
    "security": {
      "sbu_forget": true,
      "intent_mandate_required": true,
      "purpose_vocabulary": "standardized",
      "audit_chain": true,
      "audit_chain_verifiable": true,
      "content_safety_guard": true,
      "prompt_injection_protection": true
    }
  },
  "auth": {
    "intra_org": ["mtls", "jwt"],
    "cross_org": ["did", "vc", "dpop"]
  }
}
```

## 3. Protobuf 关键扩展

### 3.1 标准 purpose 枚举

```proto
enum StandardPurpose {
  PURPOSE_UNSPECIFIED = 0;
  CODE_REVIEW = 1;
  PROJECT_DEBUGGING = 2;
  KNOWLEDGE_TRANSFER = 3;
  COMPLIANCE_AUDIT = 4;
  MEMORY_MAINTENANCE = 5;
  CROSS_AGENT_COLLABORATION = 6;
}
```

详见 [purpose-vocabulary.md](./purpose-vocabulary.md)。

### 3.2 RedactionReport（可验证）

```proto
message RedactionReport {
  string report_id = 1;
  string policy_id = 2;
  string policy_version = 3;
  string executor_did = 4;
  google.protobuf.Timestamp applied_at = 5;
  bytes source_scope_hash = 6;
  repeated RedactionAction actions_taken = 7;
  uint32 sbu_units_removed = 8;
  uint32 sbu_tombstones_included = 9;
  uint32 total_units_before = 10;
  uint32 total_units_after = 11;
  bytes report_signature = 12;       // JWS
}
```

详见 [redaction-policy.md](./redaction-policy.md)。

### 3.3 HandoffPacket 含可验证 RedactionReport

```proto
message HandoffPacket {
  string packet_id = 1;
  string task_id = 2;
  string from_agent_did = 3;
  string to_agent_did = 4;
  string goal = 5;
  string current_state_summary = 6;
  repeated string completed_steps = 7;
  repeated string failed_attempts = 8;
  repeated string open_questions = 9;
  repeated string constraints = 10;
  repeated string tool_state_refs = 11;
  string l0_snapshot_ref = 12;
  repeated string l1_episode_refs = 13;
  repeated string l2_fact_refs = 14;
  ContextGrant context_grant = 15;
  string redaction_policy_id = 16;
  RedactionReport sbu_redaction_report = 17;     // 可验证
  bytes audit_chain_head = 18;
  bytes packet_signature = 19;
}
```

### 3.4 DreamProposal 状态机

```proto
enum DreamProposalStatus {
  PENDING_REVIEW = 0;
  AWAITING_APPROVAL = 1;
  APPROVED = 2;
  APPLYING = 3;
  APPLIED = 4;
  REJECTED = 5;
  EXPIRED = 6;
  BLOCKED = 7;       // SBU 阻断，永久不可审批通过
}

enum RiskLevel {
  RISK_UNSPECIFIED = 0;
  LOW = 1;
  MEDIUM = 2;
  HIGH = 3;
}

message DreamProposal {
  string proposal_id = 1;
  string tenant_id = 2;
  DreamProposalStatus status = 3;
  RiskLevel risk_level = 4;
  bool sbu_blocked = 5;
  repeated MemoryMutationDesc mutations = 6;
  string reason = 7;
  string proposer_did = 8;
  optional string approver_did = 9;
  optional string approval_audit_event_id = 10;
  google.protobuf.Timestamp created_at = 11;
  google.protobuf.Timestamp expires_at = 12;
  bytes signature = 13;
}
```

详见 [dream-state-machine.md](./dream-state-machine.md)。

## 4. Capability 注册

FAP-ME 在 FAP-1 Capability Registry 中注册以下 capability：

| capability_id | risk_level | requires_mandate |
|---|---|---|
| `memory.retrieve` | low | true |
| `memory.store` | medium | true |
| `memory.share` | medium | true |
| `memory.forget` | high | true |
| `memory.handoff` | medium | true |
| `memory.audit.query` | low | true |
| `memory.dream.propose` | low | true |
| `memory.dream.approve` | high | true |
| `memory.admin.rebuild` | critical | true |

`risk_level` 决定 FAP-1 风险分级路径（见 FAP-1 [risk-execution-paths.md](../../../01-协议设计/docs/fap/risk-execution-paths.md)）。

## 5. Plugin 注册（与 FAP-1 共享 PluginRegistry）

FAP-ME 的算法插件类型在 FAP-1 PluginRegistry 中按 `type = "memory_*"` 注册：

```text
type = "memory_embedding"
type = "memory_vector_index"
type = "memory_tag_graph"
type = "memory_compact"
type = "memory_reranker"
type = "memory_dream_worker"
type = "memory_forget_engine"
type = "memory_audit_sink"
```

满足 FAP-1 [09-plugin-runtime.md](../../../01-协议设计/docs/fap/09-plugin-runtime.md) 中的 manifest、签名、生命周期约束。

## 6. Receipt 链关系

```text
FAP-1 Receipt          每个 InvokeRequest 的 Multisig Receipt
                       hash chain 由 FAP-1 Core 维护

FAP-ME AuditEvent      每个 memory.* 操作的细粒度审计
                       hash chain 由 FAP-ME AuditKernel 维护

两者共享：
  签名算法（FAP-1 SignaturePlugin）
  Sink 出口（可同时 fan-out 到同一 AuditSink）

两者独立：
  各自的 chain head
  各自的 schema
  各自的查询接口
```

## 7. 数据面映射

FAP-ME 的流类型直接复用 FAP-1 Data Plane 的 Frame 结构：

| FAP-ME 流 | 实现 |
|---|---|
| `RecallResultStream` | FAP-1 Frame 携带 `RecallResultFrame` payload |
| `MemoryChunkStream` | FAP-1 Frame + ObjectRef |
| `ContextFrameStream` | FAP-1 Frame |
| `AuditEventStream` | FAP-1 Frame |
| `DreamProposalStream` | FAP-1 Frame |

详见 [07-data-plane.md](./07-data-plane.md)。

## 8. 错误码映射

FAP-ME 错误码遵循 FAP-1 [problem-catalog.md](../../../01-协议设计/docs/fap/problem-catalog.md) 的格式：

```text
fap-me/mandate-purpose-mismatch       Mandate 的 purpose 与请求 op 不匹配
fap-me/sbu-access-denied               未授权访问 SBU
fap-me/redaction-report-invalid        RedactionReport 签名验证失败
fap-me/grant-expired                   ContextGrant 已过期
fap-me/grant-revoked                   ContextGrant 已撤销
fap-me/content-injection-detected      Prompt Injection 拦截
fap-me/chain-broken                    审计链断裂
fap-me/dream-blocked                   DreamProposal 含 SBU 被永久阻断
fap-me/dream-expired                   DreamProposal 超过 72h 未审批
fap-me/index-version-mismatch          embedding 模型与索引版本不匹配
fap-me/quota-exceeded                  租户配额超限
```

完整列表见 [problem-catalog.md](./problem-catalog.md)。

## 9. 兼容矩阵

| FAP-1 版本 | FAP-ME 版本 | 关系 |
|---|---|---|
| 1.0 | 1.0 | 完全兼容 |
| 1.1（计划） | 1.0 | 向后兼容 |
| 1.0 | 1.1（计划） | 需检查 Agent Card 中 `purpose_registry_version` |

详见 FAP-1 [version-lines.md](../../../01-协议设计/docs/fap/version-lines.md)。
