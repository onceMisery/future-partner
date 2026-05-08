# 06 - 控制面 API

> 所有 memory 操作通过 FAP-1 InvokeRequest 入口，详见 [fap1-binding.md](./fap1-binding.md)。本文档定义 memory.* capability 的 input/output schema 与裁决路径。

## 1. 设计原则

```text
不定义独立 MemoryControlService
所有外部调用 = FAP-1 InvokeRequest(capability_id, input, finality_policy)
内部 gRPC adapter（fap-me-server）仅作可信内网使用，对外协议仍是 FAP-1
控制面只传操作意图、凭证、引用；大对象通过 ObjectRef
```

## 2. Memory 操作映射到 FAP-1 Capability

| capability_id | input | output | risk_level | finality（默认） |
|---|---|---|---|---|
| `memory.retrieve` | `RetrieveInput` | `RetrieveResult` | low | OPTIMISTIC |
| `memory.store` | `StoreInput` | `StoreReceipt` | medium | VERIFIED |
| `memory.share` | `ShareInput` | `ContextGrant` | medium | VERIFIED |
| `memory.context.redeem` | `RedeemGrantInput` | `RedeemGrantResult` | medium | VERIFIED |
| `memory.handoff.create` | `HandoffCreateInput` | `HandoffPacket` | medium | VERIFIED |
| `memory.handoff.receive` | `HandoffReceiveInput` | `HandoffReceipt` | medium | VERIFIED |
| `memory.forget.soft` | `ForgetInput` | `ForgetReceipt` | high | VERIFIED |
| `memory.forget.hard` | `ForgetInput` | `ForgetReceipt` | critical | FINALIZED |
| `memory.audit.query` | `AuditQueryInput` | stream `AuditEventFrame` | low | OPTIMISTIC |
| `memory.dream.propose` | `DreamProposeInput` | `DreamProposal` | low | VERIFIED |
| `memory.dream.approve` | `DreamApprovalInput` | `DreamProposal` | high | FINALIZED |
| `memory.admin.rebuild` | `RebuildInput` | `RebuildReceipt` | critical | FINALIZED |

详见 [fap1-binding.md](./fap1-binding.md) §2。

## 3. 请求路径

```text
任何 memory 操作的进入路径（FAP-1 + FME 联合）：

  FAP-1 Transport TLS
    → Envelope Decode（FAP-1 Core）
    → Header Validation
    → Replay Cache（FAP-1）
    → Session Check
    → AuthN（FAP-1 AuthPlugin）
    → InvokeRequest 解码
    → Capability Registry 查 capability_id
    → Mandate Verify（FAP-1 + FME constraints）
    → Risk Policy → Finality 路径选择
    → FME PolicyKernel.authorize（细化 capability/layer/action 三元组裁决）
    → FME TenantKernel filter 注入
    → FME ContentSafetyGuard.check_write（仅写入）
    → CapabilityPlugin.invoke（实际执行 memory 操作）
    → FME PolicyKernel.redact（仅检索/共享）
    → FAP-1 Receipt 生成（含 multisig if FINALIZED）
    → FME AuditKernel.append（绑定 fap_receipt_id）
```

任何步骤失败立即终止链路并写审计。详见 [10-security-model.md](./10-security-model.md)。

## 4. Input / Output Schema

以下 `tenant_id` / `namespace` 字段是请求声明值，只能用于与 FAP-1 authenticated session 和 signed Mandate constraints 比对。实际查询过滤由 TenantKernel 注入，不能直接信任请求体。

### 4.1 RetrieveInput

```proto
message RetrieveInput {
  string tenant_id = 1;
  repeated string namespaces = 2;
  string query_text = 3;
  optional bytes query_embedding = 4;       // 可选：客户端预 embed
  RetrievalMode mode = 5;
  uint32 top_k = 6;
  SearchFilter filter = 7;
  google.protobuf.Duration timeout = 8;
}

message RetrieveResult {
  uint32 result_count = 1;
  bool partial_result = 2;                  // 降级或部分超时
  optional RetrievalMode actual_mode = 3;   // 实际使用模式（含降级后）
  // 实际命中通过 Data Plane 的 RecallResultFrame 流式返回
  string data_stream_id = 4;
}
```

### 4.2 StoreInput

```proto
message StoreInput {
  string tenant_id = 1;
  string namespace = 2;
  Layer target_layer = 3;
  // 短内容直接内嵌；长内容通过 ObjectRef
  oneof content {
    repeated MemoryUnitInput inline_units = 4;
    string upload_session_id = 5;            // 数据面预 upload 后的 session
    ObjectRef object_ref = 6;                // 已存在的对象
  }
}

message MemoryUnitInput {
  string content_text = 1;
  repeated string tags = 2;
  repeated string security_labels = 3;
  optional float source_confidence = 4;
}

message StoreReceipt {
  repeated string memory_ids = 1;
  uint32 stored_count = 2;
  uint32 rejected_count = 3;
  repeated RejectedUnit rejected = 4;       // 被 ContentSafetyGuard 拒绝的项
  string audit_event_id = 5;
}
```

`StoreInput` 的"首帧 mandate"问题（最终版的 P1-6）由此解决：mandate 在 FAP-1 InvokeRequest envelope 中，不在 chunk 流里。详见 [07-data-plane.md](./07-data-plane.md)。

### 4.3 ShareInput

```proto
message ShareInput {
  string tenant_id = 1;
  string namespace = 2;
  string receiver_agent_did = 3;
  ContextScope scope = 4;
  string redaction_policy_id = 5;
  google.protobuf.Timestamp expires_at = 6;
  ShareOptions options = 7;
}

message ContextScope {
  repeated string memory_ids = 1;
  optional Layer max_layer = 2;
  // 显式声明源侧 SBU manifest（详见 redaction-policy.md）
  optional SbuManifest sbu_manifest = 3;
}
```

### 4.4 ForgetInput

```proto
message ForgetInput {
  string tenant_id = 1;
  ForgetMode mode = 2;                       // SOFT | HARD
  ForgetTarget target = 3;
  string reason = 4;
  // saga 选项
  bool wait_for_reconciled = 5;              // 阻塞等待最终一致（详见 outbox-reconciliation.md）
  optional google.protobuf.Duration wait_timeout = 6;
  optional string callback_url = 7;          // 异步通知达成最终一致
}

message ForgetTarget {
  oneof kind {
    string memory_id = 1;
    string lineage_root_id = 2;
    string tag_id = 3;
    string namespace = 4;
    string tenant_id_full_purge = 5;         // 需 elevated mandate
    SbuPurge sbu = 6;
  }
}
```

### 4.5 其他 Input 摘要

```proto
HandoffCreateInput { task_id, to_agent_did, ContextScope, share_options }
HandoffReceiveInput { handoff_packet, import_policy }
RedeemGrantInput { grant_id, requested_triple, access_mode }
AuditQueryInput { tenant_id, optional session_id, time_range, filter, limit }
DreamProposeInput { scope, params }
DreamApprovalInput { proposal_id, action, reason }
RebuildInput { tenant_id, target: index_version | tag_graph | embedding_upgrade }
```

完整定义见 [16-rust-implementation.md](./16-rust-implementation.md) 中的 proto 布局；FME 只定义 payload schema，外部入口仍为 FAP-1 InvokeRequest。

## 5. Capability 三元组裁决

PolicyKernel 用规范化三元组精确匹配，**禁止任何字符串子串/前缀匹配**：

```rust
struct CapabilityTriple {
    capability: MemoryCapability,    // enum
    layer: MemoryLayer,              // L0 | L1 | L2 | L3 | MEMORY_LAYER_UNSPECIFIED
    action: ActionKind,              // READ | WRITE | DELETE_SOFT | DELETE_HARD | APPROVE | ADMIN
}
```

具体裁决规则详见 [purpose-vocabulary.md](./purpose-vocabulary.md)。

## 6. 必备 Header（复用 FAP-1）

每个 InvokeRequest Envelope 必含（FAP-1 已规定）：

```text
message_id           UUID v7
session_id
created_at
expires_at
payload_sha256
prev_hash
nonce
signature
mandate_id           ← FAP-1 Mandate（含 FME constraints）
finality_policy      ← OPTIMISTIC | VERIFIED | FINALIZED
```

详见 FAP-1 [03-wire-protocol.md](../../../01-协议设计/docs/fap/03-wire-protocol.md)。

## 7. HTTP/JSON Fallback

仅作互操作；底层仍走 FAP-1 InvokeRequest。映射：

| 功能 | HTTP 方法 | 路径 |
|---|---|---|
| `memory.retrieve` | POST | `/fap/v1/invoke/memory.retrieve` |
| `memory.store` | POST | `/fap/v1/invoke/memory.store` |
| `memory.share` | POST | `/fap/v1/invoke/memory.share` |
| `memory.context.redeem` | POST | `/fap/v1/invoke/memory.context.redeem` |
| `memory.forget.soft` | POST | `/fap/v1/invoke/memory.forget.soft` |
| `memory.forget.hard` | POST | `/fap/v1/invoke/memory.forget.hard` |
| `memory.handoff.create` | POST | `/fap/v1/invoke/memory.handoff.create` |
| `memory.handoff.receive` | POST | `/fap/v1/invoke/memory.handoff.receive` |
| `memory.audit.query` | POST | `/fap/v1/invoke/memory.audit.query` |
| `memory.dream.propose` | POST | `/fap/v1/invoke/memory.dream.propose` |
| `memory.dream.approve` | POST | `/fap/v1/invoke/memory.dream.approve` |
| `memory.admin.rebuild` | POST | `/fap/v1/invoke/memory.admin.rebuild` |

请求体即对应 `*Input` 的 JSON 编码。`mandate_id` / `finality_policy` 在 HTTP header 中：

```text
X-FAP-Mandate-Id: m-001
X-FAP-Finality:   FINALIZED
X-FAP-Idempotency-Key: ...
```

错误码遵循 [problem-catalog.md](./problem-catalog.md)。

## 8. 异步与流式

```text
RetrieveResult           控制响应立即返回，结果通过 data_stream_id 流式
AuditQueryInput          流式返回 AuditEventFrame
DreamProposeInput        立即返回 proposal_id；APPLYING 阶段通过 audit/event 通知
ForgetInput              wait_for_reconciled=false → 立即返回 COMMITTED_LOCALLY
                         wait_for_reconciled=true → 阻塞至 GLOBALLY_RECONCILED 或超时
admin.rebuild            立即返回 job_id；通过 admin.status 查询
```

## 9. 幂等性

```text
所有 capability 的 InvokeRequest 必须含 idempotency_key（FAP-1 已规定）
幂等窗口：60 分钟
重复 key + 同 input → 返回原 receipt
重复 key + 不同 input → 拒绝（fap/idempotency-mismatch）

ForgetInput 在 saga 内部还使用 ledger_id 维持跨系统幂等
```

## 10. 取消与超时

```text
RetrieveInput.timeout         默认 300ms，详见 retrieval-fallback.md
StoreInput                    默认 5s（含 embedding + audit）
ForgetInput.wait_timeout      默认 1800s（30 分钟）
DreamProposeInput             默认 60s（含 LLM 推理）

客户端取消通过 FAP-1 InvokeCancel 控制消息
服务端必须立即停止后续 plugin 调度，但 saga 中已写入 ledger 的 mutation 继续异步达成
```

## 11. 与最终版的差距

```text
最终版                                  本文档
-----                                   -----
独立 MemoryControlService gRPC          所有 op 走 FAP-1 InvokeRequest
独立 MemoryStoreRequest（流式入口）     StoreInput + 数据面 ObjectRef 引用
purpose 字符串字段                       三元组裁决（详见 purpose-vocabulary.md）
ForgetReceipt 单状态                     双状态（详见 outbox-reconciliation.md）
独立 finality 概念                       与 risk_level 绑定
```
