# fap1-binding.md

> FAP-ME 与 FAP-1 协议的具体绑定：capability 映射、Mandate 复用、Receipt-AuditEvent 绑定、finality 与风险路径。

## 1. 设计原则

```text
不重新发明协议。FAP-ME 必须复用 FAP-1：
  Mandate 委托链与撤销表
  Capability Registry 与 risk_level
  Multisig Receipt 与 hash chain
  Data Plane Merkle 化与背压
  AuthPlugin / SignaturePlugin

FAP-ME 只在 FAP-1 之上追加：
  memory.* 系列 capability 注册
  Mandate constraint：memory.purpose / memory.allowed_triples / memory.sbu_access
  AuditEvent 含 fap_receipt_id 反向引用
```

## 2. Memory 操作 = FAP-1 Capability

所有记忆操作通过 FAP-1 `InvokeRequest(capability_id, input, finality_policy)` 入口，**不**通过独立 `MemoryControlService`。

| capability_id | input 主体 | finality_policy（默认） | risk_level |
|---|---|---|---|
| `memory.retrieve` | RetrieveInput | OPTIMISTIC | low |
| `memory.store` | StoreInput | VERIFIED | medium |
| `memory.share` | ShareInput | VERIFIED | medium |
| `memory.context.redeem` | RedeemGrantInput | VERIFIED | medium |
| `memory.handoff.create` | HandoffCreateInput | VERIFIED | medium |
| `memory.handoff.receive` | HandoffReceiveInput | VERIFIED | medium |
| `memory.forget.soft` | ForgetInput | VERIFIED | high |
| `memory.forget.hard` | ForgetInput | FINALIZED | critical |
| `memory.audit.query` | AuditQueryInput | OPTIMISTIC | low |
| `memory.dream.propose` | DreamProposeInput | VERIFIED | low |
| `memory.dream.approve` | DreamApprovalInput | FINALIZED | high |
| `memory.admin.rebuild` | RebuildInput | FINALIZED | critical |

`risk_level` 决定 FAP-1 风险分级执行路径（详见 FAP-1 [risk-execution-paths.md](../../../01-协议设计/docs/fap/risk-execution-paths.md)）。

`finality_policy` 决定凭证完成度：

```text
OPTIMISTIC   单签即完成（仅检索类）
VERIFIED     单签 + AuditSink 持久化（写入类默认）
FINALIZED    Multisig + 协议级 Receipt 最终化；不等于外部系统已全局协调
```

HardForget 的跨系统状态由 ForgetReceipt `COMMITTED_LOCALLY / GLOBALLY_RECONCILED` 表达，不能把 FAP-1 finality 当作外部 S3/Kafka/Qdrant 已删除的证明。

## 3. 内部 gRPC service 的定位

```text
fap-me-server 内部可暴露 gRPC service 作为 adapter：
  仅本地或可信内网使用
  其底层实际仍构造 FAP-1 InvokeRequest 走完整链路
  对外协议只声明 FAP-1 + memory.* capability

任何外部 Agent 不应直接调用 MemoryControlService；
跨网络通信必须走 FAP-1 InvokeRequest。
```

## 4. Mandate 复用（路线 B）

FME **不**定义独立 mandate.proto。所有 memory 授权通过 FAP-1 Mandate 表达：

```proto
// FAP-1 已有
message Mandate {
  string mandate_id = 1;
  string parent_mandate_id = 2;
  string issuer = 3;
  string subject = 4;
  repeated string capabilities = 5;        // ← 含 memory.*
  repeated Constraint constraints = 6;     // ← FME 在这里加约束
  google.protobuf.Timestamp expires_at = 7;
  uint64 budget_units = 8;
  uint32 delegation_depth = 9;
  uint32 max_delegation_depth = 10;
  Signature issuer_signature = 11;
  // ...
}

message Constraint {
  string type = 1;       // 命名空间化的 constraint type
  string value = 2;      // 值（JSON 编码）
}
```

### FME 引入的标准 Constraint 类型

| constraint.type | value 示例 | 含义 |
|---|---|---|
| `memory.purpose` | `"project_debugging"` | 标准 purpose |
| `memory.allowed_triples` | `[{"capability":"MEMORY_RETRIEVE","layer":"L1","action":"READ"}]` | 允许的 capability/layer/action 三元组 |
| `memory.sbu_access` | `"DENY"` \| `"REDACTED_ONLY"` \| `"RAW_ALLOWED"` | SBU 访问级别 |
| `memory.namespaces` | `["proj-x","proj-y"]` | 限定命名空间 |
| `memory.security_label_exclude` | `["secret","sbu"]` | 显式排除标签 |
| `memory.cross_tenant` | `"true"` | 是否允许跨租户 |
| `memory.classifier_version_min` | `"0.3.1"` | 接收 RedactionReport 时要求的最低分类器版本 |

### Mandate 字段语义映射

| FME 字段 | 来源 |
|---|---|
| `mandate_id` | FAP-1 |
| `purpose` | FAP-1 `constraints[type="memory.purpose"]` |
| `allowed_triples` | FAP-1 `constraints[type="memory.allowed_triples"]` |
| `expires_at` | FAP-1 |
| `budget_units` | FAP-1（可用于限制 dream 调用次数等） |
| `revoked` | FAP-1 撤销表 |
| `delegation_depth` | FAP-1 |

`memory.purpose` 与 `purpose-vocabulary.md` 中标准词汇表对应；`max_ttl_hours` / `requires_vc` / `requires_elevated_mandate` 由 PolicyKernel 在 authorize 时校验，不写入 mandate 本身。

### 委托与撤销

```text
委托：直接走 FAP-1 委托链
  父 mandate.capabilities 含 memory.retrieve
  子 mandate.capabilities ⊆ 父
  子 constraints 必须更严或同级（如 allowed_triples 子集 + scope 子集）

撤销：直接走 FAP-1 撤销表
  撤销一份 mandate 同时切断协议级与记忆级（修复路线 A 的安全漏洞）
  RevocationPlugin 查询接口由 FAP-1 提供
```

## 5. Capability 三元组裁决

PolicyKernel 用规范化的三元组精确匹配，不使用字符串 `contains`。`MemoryCapability / MemoryLayer / MemoryAction` 的权威 enum 与字符串映射见 [purpose-vocabulary.md §3](./purpose-vocabulary.md) 与 [signing-canonical.md §8/§10](./signing-canonical.md)：

```rust
pub struct CapabilityTriple {
    pub capability: MemoryCapability,    // enum，9 值
    pub layer: MemoryLayer,              // enum，6 值（含 UNSPECIFIED / AUDIT_CHAIN）
    pub action: MemoryAction,            // enum，13 值（含 UNSPECIFIED）
}
```

完整枚举值见 [purpose-vocabulary.md §3](./purpose-vocabulary.md)。FAP-ME 内禁止再定义 6 值版 `ActionKind` 或其它简化枚举。

PolicyKernel 决策：

```text
请求 op 解析为 CapabilityTriple
mandate.capabilities 校验 capability_id（见 §9）
mandate.constraints[type="memory.allowed_triples"] 解析为 Set<CapabilityTriple>
精确匹配：requested_triple ∈ allowed_triples → 通过
任何字符串前缀 / 子串 / 通配符匹配 → 拒绝
```

详见 [purpose-vocabulary.md](./purpose-vocabulary.md)。

## 6. Receipt-AuditEvent 强绑定

每个 InvokeRequest 走完整 FAP-1 链路生成 Receipt；FME 在 AuditEvent 中反向引用。`AuditEvent` 的 canonical 字段顺序、`event_hash` 输入字段、`signature` 规则在 [signing-canonical.md §2](./signing-canonical.md) 集中冻结。本节摘要：

```proto
message AuditEvent {
  string event_id = 1;
  bytes  prev_event_hash = 2;
  bytes  event_hash = 3;
  string tenant_id = 4;
  string namespace = 5;
  string session_id = 6;
  string actor_did = 7;
  string operation = 8;            // 见 §9 capability_id 集合
  string target_ref = 9;
  string mandate_id = 10;

  // FAP-1 反向绑定
  string fap_receipt_id = 11;
  string fap_invocation_id = 12;   // == FAP-1 Envelope.message_id
  bytes  fap_envelope_hash = 13;
  string fap_finality = 14;        // OPTIMISTIC | VERIFIED | FINALIZED

  bytes  capability_triple = 15;
  bytes  content_hash = 16;
  bytes  object_ref_hashes = 17;
  string index_version = 18;
  int64  timestamp_micros = 19;
  bytes  signature = 20;           // AuditKernel 签名 event_hash
}
```

`fap_finality` 词汇仅 FAP-1 标准三档（`OPTIMISTIC | VERIFIED | FINALIZED`），不允许 `PROVISIONAL`。

实现保证：

```text
1. AuditEvent.append 必须在 InvokeRequest 完成后由 Kernel 主动调用
2. fap_receipt_id 缺失 → 拒绝写入审计（防止伪造审计而无 Receipt）
3. AuditEvent 与 FAP-1 Receipt 可双向追溯：
   FAP-1 Receipt → invocation_id → 多个 AuditEvent
   AuditEvent → fap_receipt_id → 唯一 Receipt
```

## 7. Finality 与高风险操作

```text
high / critical 风险的 memory 操作必须使用 FINALIZED 或显式补偿路径：

memory.forget.hard       FINALIZED（默认）
                         补偿路径：本地 COMMITTED_LOCALLY 状态 +
                                  外部异步 reconciliation
                         详见 outbox-reconciliation.md

memory.dream.approve     FINALIZED（默认）
                         审批是不可逆操作

memory.admin.rebuild     FINALIZED
                         索引重建涉及双索引切换

memory.handoff.create    VERIFIED（默认）
                         可升级到 FINALIZED 当 grant.scope 跨租户

memory.context.redeem    VERIFIED（默认）
                         若 access_mode = MERGE_L2，升级到 FINALIZED
```

OPTIMISTIC 不允许用于 risk_level ≥ medium 的 capability。

## 8. Data Plane 复用 FAP-1 Merkle

记忆相关的所有数据流复用 FAP-1 Data Plane（详见 FAP-1 [data-plane-merkle.md](../../../01-协议设计/docs/fap/data-plane-merkle.md)）：

```proto
// FAP-1 已有的 frame 类型，FME 不重新定义
DataOpen          数据流开始
DataChunkMeta     单 chunk 元数据 + Merkle 节点 hash
DataAck           确认与流控
DataCommit        流结束 + Merkle root
DataReset         异常重置
```

FME 的语义层 payload（如 RecallResultFrame、ContextFrame）作为 FAP-1 DataChunkMeta 的 payload 类型出现，但 frame 包装、签名、Merkle、断点续传、背压全部由 FAP-1 处理。

详见本目录 [07-data-plane.md](./07-data-plane.md)。

## 9. ObjectRef 复用 FAP-1

大对象（embedding 输入文档、附件、quaranine 内容）通过 FAP-1 `ObjectRef` 引用：

```proto
message ObjectRef {
  string object_id = 1;
  string media_type = 2;
  uint64 size_bytes = 3;
  bytes sha256 = 4;
  string uri = 5;
  string lease_id = 6;
  google.protobuf.Timestamp expires_at = 7;
}
```

实现：FAP-1 `ObjectStorePlugin` 即 FAP-ME `ObjectStore` 插件（同一接口，不重复定义）。

## 10. 错误码命名空间

```text
FAP-1 错误：fap/<category>-<reason>
FAP-ME 错误：fap-me/<category>-<reason>

两者命名空间不冲突；客户端必须同时识别。

例：
  fap/auth-mandate-revoked         协议级
  fap-me/sbu-access-denied         记忆级
  fap-me/forget-pending-reconcile  Saga 状态（详见 outbox-reconciliation.md）
```

详见 [problem-catalog.md](./problem-catalog.md)。

## 11. Agent Card 一致性

Agent Card 中 `memory` 子节必须与 FAP-1 主声明一致：

```json
{
  "protocol": {
    "name": "FAP-1",
    "versions": ["1.0"]
  },
  "capabilities": {
    "memory.retrieve": { "risk_level": "low", "supports_streaming": true },
    "memory.store":    { "risk_level": "medium", "supports_batch": true },
    "memory.context.redeem": { "risk_level": "medium", "default_finality": "VERIFIED" },
    "memory.forget.hard": {
      "risk_level": "critical",
      "default_finality": "FINALIZED",
      "supports_async_reconciliation": true
    }
  },
  "memory": {
    "extension": "FAP-ME",
    "version": "1.0",
    "purpose_registry_version": "1.0",
    "mandate_constraints": [
      "memory.purpose",
      "memory.allowed_triples",
      "memory.sbu_access",
      "memory.namespaces",
      "memory.security_label_exclude",
      "memory.cross_tenant",
      "memory.classifier_version_min"
    ]
  }
}
```

`capabilities.memory.*` 在 FAP-1 主清单中声明 risk_level 与 finality；`memory` 子节仅声明记忆专属元数据。

## 12. 与最终版的差距

```text
最终版                                  本文档
-----                                   -----
独立 MemoryControlService               全部走 FAP-1 InvokeRequest（capability）
独立 FME Mandate schema                  FAP-1 Mandate + FME constraints（路线 B）
purpose 自由文本字段                     constraint type "memory.purpose" + 三元组裁决
AuditEvent 不引用 Receipt                AuditEvent.fap_receipt_id 强绑定
finality 未与 risk_level 绑定            high/critical 默认 FINALIZED
自定义 DataFrame                         复用 FAP-1 DataOpen/DataChunkMeta/DataCommit
独立 ObjectRef 概念                      复用 FAP-1 ObjectRef + ObjectStorePlugin
```
