# 10 - 安全模型

> 详见 [kernel-contract.md](./kernel-contract.md) 边界约束、[content-safety.md](./content-safety.md) Prompt Injection 防护、[purpose-vocabulary.md](./purpose-vocabulary.md) 标准化授权、[redaction-policy.md](./redaction-policy.md) 脱敏与报告边界、[fap1-binding.md](./fap1-binding.md) FAP-1 绑定。

## 1. 安全分层

```text
传输层    mTLS / FAP-1 Wire Protocol
身份层    DID / VC / mTLS / JWT / DPoP
凭证层    FAP-1 Mandate + FME constraints
判定层    PolicyKernel（purpose registry + capability triple）
隔离层    TenantKernel
内容层    ContentSafetyGuard
脱敏层    PolicyKernel.redact
审计层    AuditKernel + FAP-1 Receipt binding
```

每一层在 Kernel 内顺序执行，不可被插件改写或跳过。FME 不定义独立 Mandate schema；所有记忆约束必须进入 FAP-1 Mandate 的签名范围。

## 2. 不可插件化的安全组件（Kernel）

| 组件 | 必留理由 |
|---|---|
| MandateVerifier | 校验 FAP-1 Mandate 签名、过期、撤销、budget 与 FME constraints，不能由插件验证 |
| PolicyKernel | 所有记忆操作的唯一授权入口 |
| TenantKernel | 强制注入 tenant_id / namespace 过滤 |
| ContentSafetyGuard | 写入路径与检索返回路径的内容安全门禁 |
| AuditKernel | FME 哈希链、FAP-1 Receipt 绑定、链完整性验证 |

详见 [kernel-contract.md](./kernel-contract.md)。

## 3. 安全链路（Kernel 固定顺序）

```text
Transport TLS
  → FAP-1 Envelope Decode
  → Header Validation
  → Replay Cache（FAP-1）
  → Session Check
  → AuthN（FAP-1 AuthPlugin）
  → AuthZ（FAP-1 PolicyPlugin，仅协议级 capability 预判）
  → MandateVerifier（FME Kernel）
      验证 FAP-1 Mandate signature / expires_at / revocation / budget
      读取 signed constraints 中的 memory.purpose / memory.allowed_triples / memory.scope
  → PolicyKernel.authorize
      从 capability_id + request payload 推导 CapabilityTriple
      对 mandate.allowed_triples 与 purpose.allowed_triples 做精确匹配
      校验 namespace / layer / sbu / cross_tenant 是否被 signed constraints 覆盖
  → TenantKernel
      强制注入 WHERE tenant_id = ? AND namespace IN (?)
  → ContentSafetyGuard.check_write（仅写入）
      高风险内容隔离保存或拒绝，写入 security_labels
  → Memory Orchestrator → capability-scoped Plugin Handle
  → PolicyKernel.redact（仅检索、共享、handoff 返回）
      执行 RedactionPolicy；产出 RedactionReport
  → AuditKernel.append
      写入 fap_receipt_id / fap_invocation_id / fap_envelope_hash
```

任何步骤失败立即终止链路并写审计。高风险操作必须遵守 FAP-1 `finality_policy`，详见 [fap1-binding.md](./fap1-binding.md)。

## 4. FAP-1 Mandate + FME Constraints

FME 约束示例：

```json
{
  "mandate_id": "mandate_001",
  "issuer": "did:web:user.example",
  "subject": "did:web:agent-a.example",
  "capabilities": [
    "memory.retrieve",
    "memory.store",
    "memory.handoff"
  ],
  "constraints": [
    {"type": "memory.purpose", "value": "project_debugging"},
    {"type": "memory.allowed_triples", "value": [
      {"capability": "MEMORY_RETRIEVE", "layer": "L0", "action": "READ"},
      {"capability": "MEMORY_RETRIEVE", "layer": "L1", "action": "READ"},
      {"capability": "MEMORY_STORE", "layer": "L1", "action": "WRITE"},
      {"capability": "MEMORY_HANDOFF", "layer": "MEMORY_LAYER_UNSPECIFIED", "action": "HANDOFF_CREATE"}
    ]},
    {"type": "memory.namespaces", "value": ["project-x"]},
    {"type": "memory.sbu_access", "value": "DENY"},
    {"type": "memory.cross_tenant", "value": false},
    {"type": "memory.security_label_exclude", "value": ["secret"]}
  ],
  "expires_at": "2026-05-09T23:59:59Z",
  "issuer_signature": "JWS..."
}
```

约束解释：

```text
memory.purpose             必须取自 PurposeRegistry
memory.allowed_triples     capability/layer/action 三元组，精确匹配
memory.namespaces          租户内允许访问的 namespace 集合
memory.sbu_access          DENY | REDACTED_ONLY | RAW_ALLOWED
memory.cross_tenant        是否允许跨租户或跨组织
memory.security_label_*    label allow/deny 过滤
```

禁止把 `tenant_id`、`purpose`、`allowed_triples`、`sbu_access` 放在未签名请求体中作为授权来源。请求体可以携带期望值，但只能被用于与 Mandate 约束比对，不能扩大权限。

## 5. 租户隔离（TenantKernel）

强制隔离维度：

```text
tenant_id
namespace
owner_subject_did
owner_agent_did
visibility（private | project | team | org | foreign）
security_labels（sbu | secret | pii | internal）
mandate_scope
```

所有查询自动追加：

```sql
WHERE tenant_id = :tenant_id
  AND namespace IN (:allowed_namespaces)
  AND layer <= :max_layer
  AND visibility IN (:allowed_visibility)
  AND deleted_at IS NULL
```

`tenant_id` 来自 FAP-1 authenticated session 与 TenantKernel 映射，不信任外部请求体。插件不能拼装 SQL，也不能接收裸连接；只能使用 capability-scoped handle。

## 6. SBU（Sensitive Behavioral Unit）

定义：

```text
敏感行为单元：隐私、身份、凭据、敏感偏好、临时授权、高风险行为、不可传播上下文。
```

SBU 默认策略：

| 行为 | 默认策略 |
|---|---|
| 写入 L0 | 允许隔离保存并标记，不能静默晋升 |
| 提升到 L1 | 需要策略检查与脱敏 |
| 提升到 L2 | 默认禁止，除非 RedactionPolicy 明确允许 |
| 共享 | 默认禁止原文共享 |
| handoff | 默认摘要化并附 RedactionReport |
| 梦境整合 | 禁止使用原文（DreamProposal 永久 BLOCKED） |
| hard forget | 必须级联处理 raw、embedding、snapshot、grant、本地索引和本地 CAS |

详见 [forget-engine.md](./forget-engine.md)、[dream-state-machine.md](./dream-state-machine.md)。

## 7. 跨 Agent 脱敏报告的信任边界

跨 Agent 共享时，发送方必须产出签名 RedactionReport：

```text
HandoffPacket {
  ...
  sbu_redaction_report: RedactionReport {
    policy_id,
    policy_version,
    classifier_id,
    classifier_version,
    sbu_manifest,
    actions_taken,
    report_signature
  }
}
```

RedactionReport 证明的是：

```text
由 executor_did 按 policy_id/policy_version 与 classifier_version 执行过脱敏
source_scope_hash 与被共享 snapshot 匹配
报告中声明的命中项已执行相应 action
```

它不证明源数据绝对没有 SBU 残留。接收方信任来自 policy 质量、classifier 召回率与 executor 身份信誉的组合。跨租户/跨组织共享必须提供 `sbu_manifest`；租户内低风险共享可按策略设为可选。详见 [redaction-policy.md](./redaction-policy.md)。

## 8. 内容安全守卫

写入路径强制执行：

```text
Prompt Injection 模式识别
最大长度限制（默认 32KB）
embedding 输入最大 token 数（默认 8192）
PII / SBU 检测（按租户启用）
高风险样本隔离保存或低信任标记
```

检索返回路径也必须做二次过滤，避免高风险片段被直接拼接进模型上下文。详见 [content-safety.md](./content-safety.md)。

## 9. 凭证撤销

```text
mandate.revoked = true 后：
  ≤ 60s 内全集群可见
  retrieve / store / share / handoff / forget 立即拒绝
  对应 ContextGrant 自动失效
```

撤销事件必须同时写 FAP-1 Receipt 与 FME AuditEvent。

## 10. Replay 防护（复用 FAP-1）

```text
message_id / idempotency_key / created_at / expires_at
payload_sha256 / prev_hash / session_id / nonce
```

详见 FAP-1 [10-security-model.md](../../../01-协议设计/docs/fap/10-security-model.md)。

## 11. 跨组织（DID/VC/DPoP）

跨租户、跨组织协作必须：

```text
DID 解析     发现方 Agent Card
VC 验证      凭证可机器验证
DPoP        证明客户端持有私钥并约束 token 使用者
```

启用条件：

```text
memory.cross_tenant = true
PurposeRegistry[purpose].cross_tenant = true
RedactionPolicy 必须包含 cross_org_safe = true
sbu_manifest 必须存在
```

## 12. 审计强制写入

详见 [11-audit-and-receipt.md](./11-audit-and-receipt.md)。所有以下操作必须写审计：

```text
memory.retrieve（含命中数、评分摘要、fap_receipt_id）
memory.store（含 content_hash、data_commit_hash）
memory.share（含 grant_id、redaction_report_id）
memory.forget（含 local/global 状态、级联范围、ledger_id）
memory.handoff（含 packet_signature）
memory.dream.propose / approve / reject
plugin.invoke（含 input/output hash 与 granted capability handle）
chain.integrity_verify
mandate.revoke
```
