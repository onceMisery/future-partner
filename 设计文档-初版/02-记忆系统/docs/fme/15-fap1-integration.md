# 15 - FAP-1 协议集成

> 本章描述 FAP-ME 与 FAP-1 协议层的集成点。详细绑定规范见 [fap1-binding.md](./fap1-binding.md)；FAP-1 总览见 [00-overview](../../../01-协议设计/docs/fap/00-overview.md)。

## 1. 集成原则

```text
FAP-ME 不重新发明协议
FAP-ME 复用 FAP-1：
  Discovery（Agent Card）
  Session（QUIC/H3）
  Auth（mTLS / JWT / DID / VC / DPoP）
  Control Plane（InvokeRequest / InvokeResult）
  Data Plane（DataOpen / DataChunkMeta / DataCommit / DataAck + ObjectRef）
  Receipt 链（FAP-1 Multisig Receipt）
  Mandate（FAP-1 Mandate + constraints）

FAP-ME 增加：
  memory.* capability
  Agent Card 中的 memory 子节
  Memory 专属 payload schema（RedactionPolicy / DreamProposal / PurposeRegistry 等）
  FME AuditEvent（强绑定 FAP-1 Receipt）
```

所有外部 Agent 调用 FME 都必须走 FAP-1 `InvokeRequest(capability_id, input, input_objects, mandate_id, finality_policy)`。FME 不暴露绕过 FAP-1 风险路径的独立 `MemoryControlService`。

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
    "layers": ["L0", "L1", "L2", "L3"],
    "capabilities": [
      "memory.retrieve",
      "memory.store",
      "memory.share",
      "memory.context.redeem",
      "memory.forget.soft",
      "memory.forget.hard",
      "memory.handoff.create",
      "memory.handoff.receive",
      "memory.audit.query",
      "memory.dream.propose",
      "memory.dream.approve",
      "memory.admin.rebuild"
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
    "purpose_registry_hash": "sha256:...",
    "capability_triples": true,
    "redaction_policy_schema": "fap-redaction-v1",
    "sbu_redaction_report_schema": "fap-redaction-report-v1",
    "handoff": {
      "supported": true,
      "requires_context_grant": true,
      "sbu_manifest_required_cross_tenant": true
    },
    "security": {
      "sbu_forget": true,
      "fap_mandate_constraints_required": true,
      "audit_chain": true,
      "fap_receipt_binding_required": true,
      "content_safety_guard": true
    }
  },
  "auth": {
    "intra_org": ["mtls", "jwt"],
    "cross_org": ["did", "vc", "dpop"]
  }
}
```

## 3. Protobuf 关键扩展

FME payload 可以定义在 `memory_types.proto` 等文件中，但不得重新定义 FAP-1 `Mandate`、`Envelope` 或控制面 service。

### 3.1 Purpose 与授权三元组

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

message MemoryCapabilityTriple {
  MemoryCapability capability = 1;
  MemoryLayer layer = 2;
  MemoryAction action = 3;
}
```

三元组进入 FAP-1 Mandate `constraints`，详见 [purpose-vocabulary.md](./purpose-vocabulary.md)。

### 3.2 RedactionReport

```proto
message SbuManifest {
  string classifier_id = 1;
  string classifier_version = 2;
  string scan_coverage = 3;       // full | sampled | partial
  double sample_rate = 4;
  uint32 patterns_count_scanned = 5;
  uint32 source_unit_count = 6;
}

message RedactionReport {
  string report_id = 1;
  string policy_id = 2;
  string policy_version = 3;
  string classifier_id = 4;
  string classifier_version = 5;
  string executor_did = 6;
  google.protobuf.Timestamp applied_at = 7;
  bytes source_scope_hash = 8;
  optional SbuManifest sbu_manifest = 9;
  repeated RedactionAction actions_taken = 10;
  repeated AuditSampleHash audit_sample_hashes = 11;
  uint32 sbu_units_removed = 12;
  uint32 sbu_tombstones_included = 13;
  uint32 total_units_before = 14;
  uint32 total_units_after = 15;
  bytes report_signature = 16;
}
```

RedactionReport 证明执行留痕，不证明绝对无 SBU 残留。详见 [redaction-policy.md](./redaction-policy.md)。

### 3.3 ContextGrant 与 HandoffPacket

```proto
message ContextGrant {
  string grant_id = 1;
  string issuer_did = 2;
  string receiver_agent_did = 3;
  string tenant_id = 4;
  string namespace = 5;
  repeated string snapshot_ids = 6;
  repeated MemoryCapabilityTriple allowed_triples = 7;
  AccessMode access_mode = 8;
  string redaction_policy_id = 9;
  string redaction_report_id = 10;
  string purpose = 11;
  string mandate_id = 12;
  string fap_receipt_id = 13;
  google.protobuf.Timestamp expires_at = 14;
  bytes signature = 15;
}

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
  RedactionReport sbu_redaction_report = 17;
  bytes audit_chain_head = 18;
  string fap_receipt_id = 19;
  bytes packet_signature = 20;
}
```

### 3.4 ForgetReceipt

```proto
enum ForgetReceiptStatus {
  COMMITTED_LOCALLY = 0;
  GLOBALLY_RECONCILED = 1;
  RECONCILE_FAILED = 2;
}

message ForgetReceipt {
  string receipt_id = 1;
  string request_id = 2;
  string mode = 3;                    // soft | hard
  ForgetReceiptStatus status = 4;
  repeated string target_ids = 5;
  repeated string cascaded_ids = 6;
  repeated ExternalMutation pending_external_systems = 7;
  string mutation_ledger_id = 8;
  google.protobuf.Timestamp local_committed_at = 9;
  optional google.protobuf.Timestamp globally_reconciled_at = 10;
  string audit_event_id = 11;
  string fap_receipt_id = 12;
  bytes signature = 13;
}
```

## 4. Capability 注册

FAP-ME 在 FAP-1 Capability Registry 中注册以下 capability：

| capability_id | risk_level | requires_mandate | minimum finality |
|---|---|---:|---|
| `memory.retrieve` | low | true | OPTIMISTIC allowed |
| `memory.store` | medium | true | VERIFIED；合规关键写入可升级 FINALIZED |
| `memory.share` | medium | true | VERIFIED；跨租户/跨组织升级 FINALIZED |
| `memory.context.redeem` | medium | true | VERIFIED；`MERGE_L2` 升级 FINALIZED |
| `memory.forget.soft` | high | true | VERIFIED |
| `memory.forget.hard` | critical | true | FINALIZED before local commit |
| `memory.handoff.create` | medium | true | VERIFIED；跨租户升级 FINALIZED |
| `memory.handoff.receive` | medium | true | VERIFIED |
| `memory.audit.query` | low | true | OPTIMISTIC allowed；合规导出场景升级 FINALIZED |
| `memory.dream.propose` | low | true | VERIFIED |
| `memory.dream.approve` | high | true | FINALIZED before applying mutation |
| `memory.admin.rebuild` | critical | true | FINALIZED |

`risk_level` 决定 FAP-1 风险分级路径（见 FAP-1 [risk-execution-paths.md](../../../01-协议设计/docs/fap/risk-execution-paths.md)）。

## 5. Mandate 绑定

FME 不定义 `mandate.proto`。FME 只定义 FAP-1 Mandate constraints 的类型与校验规则：

```text
memory.purpose
memory.allowed_triples
memory.namespaces
memory.sbu_access
memory.cross_tenant
memory.security_label_allow / memory.security_label_exclude
memory.classifier_version_min
```

高风险调用必须同时满足：

```text
capability_id ∈ FAP-1 Mandate.capabilities
requested CapabilityTriple ∈ memory.allowed_triples
requested CapabilityTriple ∈ PurposeRegistry[purpose].allowed_triples
request scope 被 signed constraints 覆盖
```

## 6. Plugin 注册（与 FAP-1 共享 PluginRegistry）

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
type = "memory_object_store"
```

满足 FAP-1 [09-plugin-runtime.md](../../../01-协议设计/docs/fap/09-plugin-runtime.md) 中的 manifest、签名、生命周期约束。第三方插件默认 WASM/sidecar；`rust_native` 仅允许 trusted/builtin。

## 7. Receipt 链关系

```text
FAP-1 Receipt          每个 InvokeRequest 的 Multisig Receipt
                       hash chain 由 FAP-1 Core 维护

FAP-ME AuditEvent      每个 memory.* 操作的细粒度审计
                       hash chain 由 FAP-ME AuditKernel 维护
```

两者必须强绑定：

```text
AuditEvent.fap_receipt_id
AuditEvent.fap_invocation_id
AuditEvent.fap_envelope_hash
AuditEvent.fap_finality
```

详见 [11-audit-and-receipt.md](./11-audit-and-receipt.md)。

## 8. 数据面映射

FAP-ME 的流类型直接复用 FAP-1 Data Plane：

| FAP-ME 流 | FAP-1 实现 |
|---|---|
| `RecallResultStream` | DataOpen + DataChunkMeta + raw chunk + DataCommit |
| `MemoryStoreStream` | DataOpen 声明 memory metadata，raw chunk 传内容 |
| `ContextSnapshotStream` | Data Plane + ObjectRef |
| `AuditEventStream` | Data Plane 或控制面分页查询 |
| `DreamProposalStream` | 控制面通知 + Data Plane 拉取详情 |

禁止把 chunk bytes 直接塞进 FAP-1 Envelope。详见 [07-data-plane.md](./07-data-plane.md)。

## 9. 错误码映射

FAP-ME 错误码遵循 FAP-1 [problem-catalog.md](../../../01-协议设计/docs/fap/problem-catalog.md) 的格式：

```text
fap-me/mandate-purpose-mismatch       Mandate 的 purpose 与请求 triple 不匹配
fap-me/capability-triple-denied       capability/layer/action 未被授权
fap-me/sbu-access-denied              未授权访问 SBU
fap-me/redaction-report-invalid       RedactionReport 签名或 scope 验证失败
fap-me/redaction-manifest-missing     跨租户共享缺少 sbu_manifest
fap-me/grant-expired                  ContextGrant 已过期
fap-me/grant-revoked                  ContextGrant 已撤销
fap-me/content-injection-detected     Prompt Injection 拦截或隔离
fap-me/chain-broken                   审计链断裂
fap-me/forget-reconcile-pending       本地已遗忘，外部系统仍在协调
fap-me/forget-reconcile-failed        外部协调失败，需要人工介入
fap-me/dream-blocked                  DreamProposal 含 SBU 被永久阻断
fap-me/dream-expired                  DreamProposal 超过 72h 未审批
fap-me/index-version-mismatch         embedding 模型与索引版本不匹配
fap-me/quota-exceeded                 租户配额超限
```

完整列表见 [problem-catalog.md](./problem-catalog.md)。

## 10. 兼容矩阵

| FAP-1 版本 | FAP-ME 版本 | 关系 |
|---|---|---|
| 1.0 | 1.0 | 完全兼容 |
| 1.1（计划） | 1.0 | 向后兼容，需保持 Invoke/Data/Mandate 字段兼容 |
| 1.0 | 1.1（计划） | 需检查 Agent Card 中 `purpose_registry_version` 与 `purpose_registry_hash` |

详见 FAP-1 [version-lines.md](../../../01-协议设计/docs/fap/version-lines.md)。
