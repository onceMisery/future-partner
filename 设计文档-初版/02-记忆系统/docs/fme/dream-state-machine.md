# dream-state-machine.md

> DreamWorker 在 L3 层异步联想/合并/优化记忆，但所有变更必须经过审批。本文档定义状态机、风险分级、SBU 阻断与超时处理。

## 1. 设计原则

```text
DreamWorker 只提议，不直接写入
所有 mutations 由 Kernel 审核 + 执行
SBU 内容永久阻断，不可审批通过
高风险变更必须人工审批
低风险变更可自动审批（可配置）
72h 未审批自动 EXPIRED
```

## 2. 状态机

```text
                     ┌──────────────────┐
                     │  PENDING_REVIEW  │
                     └─────────┬────────┘
                               │
        ┌──────────────────────┼──────────────────────┐
        │                      │                      │
   [SBU 命中]              [低风险]                [高风险]
        │                      │                      │
        ↓                      ↓                      ↓
   ┌─────────┐          ┌──────────┐         ┌────────────────┐
   │ BLOCKED │          │ APPROVED │         │ AWAITING_      │
   │ 永久阻断│          │ 自动     │         │   APPROVAL     │
   └─────────┘          └────┬─────┘         └─┬─────┬─────┬──┘
                             │              [通过][拒绝][72h]
                             │                │     │     │
                             ↓                ↓     ↓     ↓
                       ┌──────────┐      ┌────────┐ ┌────┐ ┌──────┐
                       │ APPLYING │      │APPROVED│ │REJ │ │EXPIRED│
                       └────┬─────┘      └───┬────┘ └────┘ └──────┘
                       [成功]  [失败]        │
                          │     │            ↓
                          ↓     ↓        ┌──────────┐
                     ┌────────┐ ┌──────┐ │ APPLYING │
                     │APPLIED │ │APPLY_│ └────┬─────┘
                     └────────┘ │FAILED│      │
                                └──────┘      ↓
                                          (同上 APPLIED / APPLY_FAILED)
```

## 3. 状态详解

| 状态 | 含义 | 后继 |
|---|---|---|
| `PENDING_REVIEW` | 刚被创建，等待 Kernel 风险评估 | BLOCKED / APPROVED / AWAITING_APPROVAL |
| `BLOCKED` | 含 SBU 内容，永久阻断（不可审批） | 终态 |
| `APPROVED` | 已批准（自动或人工） | APPLYING |
| `AWAITING_APPROVAL` | 等待人工审批 | APPROVED / REJECTED / EXPIRED |
| `APPLYING` | Kernel 正在执行 mutations | APPLIED / APPLY_FAILED |
| `APPLIED` | 应用完成 | 终态 |
| `APPLY_FAILED` | 应用失败（部分回滚） | 终态 |
| `REJECTED` | 审批人主动拒绝 | 终态 |
| `EXPIRED` | 72h 未审批 | 终态 |

## 4. 风险分级规则

| 触发条件 | 风险等级 | 默认行为 |
|---|---|---|
| 含任何 SBU 标记内容 | BLOCKED | 永久阻断 |
| 涉及 L2 Fact 删除或降权 | HIGH | 必须人工审批 |
| 涉及实体合并 且 置信度 < 0.85 | HIGH | 必须人工审批 |
| 涉及实体合并 且 置信度 ≥ 0.85 | MEDIUM | 可配置自动审批 |
| 涉及 L1 摘要压缩 | LOW | 默认自动审批 |
| 涉及 Tag 图剪枝（仅冷边） | LOW | 默认自动审批 |
| 涉及跨 namespace 合并 | HIGH | 必须人工审批 |
| 涉及 retention_policy 修改 | HIGH | 必须人工审批 |

## 5. SBU 检测

`PENDING_REVIEW` 状态下，Kernel 立即检查：

```text
对 proposal.mutations 中每个 MemoryUnit：
  if has_security_label("sbu") 或 has_security_label("secret"):
    proposal.sbu_blocked = true
    proposal.status = BLOCKED
    写审计：dream.blocked_by_sbu
    永久阻断（不允许任何审批操作）
```

`BLOCKED` 状态下：

```text
任何 DreamApprove 请求 → 拒绝（错误码 fap-me/dream-blocked）
任何 DreamReject 请求 → 拒绝（仅写审计，不改变状态）
```

## 6. 自动审批策略

```toml
[dream_worker.auto_approval]
enabled = true
allowed_risk_levels = ["LOW"]    # 仅 LOW 风险自动审批
require_audit_event = true       # 自动审批事件必须写审计
```

```text
PENDING_REVIEW + LOW + auto_approval.enabled
  → 自动转 APPROVED → APPLYING

PENDING_REVIEW + LOW + auto_approval.disabled
  → AWAITING_APPROVAL（等待人工）

PENDING_REVIEW + MEDIUM
  → 默认 AWAITING_APPROVAL
  → 可配置 auto_approve_medium_with_threshold（如置信度 ≥ 0.95）

PENDING_REVIEW + HIGH
  → 强制 AWAITING_APPROVAL
```

## 7. 人工审批

### 7.1 审批者身份

```text
审批者必须持有有效 FAP-1 Mandate，且 signed constraints 同时允许：
  purpose = "memory_maintenance"
  requested_triple = MEMORY_DREAM / MEMORY_LAYER_UNSPECIFIED / APPROVE

详见 purpose-vocabulary.md 中的 memory_maintenance purpose。
```

### 7.2 审批接口

```proto
FAP-1 InvokeRequest(capability_id = "memory.dream.approve")

message DreamApprovalRequest {
  string proposal_id = 1;
  ApprovalAction action = 2;     // APPROVE | REJECT
  string approver_did = 3;
  string mandate_id = 4;
  string reason = 5;
}

enum ApprovalAction {
  APPROVE = 0;
  REJECT = 1;
}
```

### 7.3 审批日志

每次审批写审计：

```text
operation = "memory.dream.approve" | "memory.dream.reject"
target_ref = proposal_id
mandate_id = approver's mandate
fap_receipt_id = FAP-1 receipt
content_hash = hash(action + reason)
```

## 8. 超时处理

```text
默认超时：72h（可按 tenant 配置）

Cerebellum 周期任务：
  - 每 30min 扫描 AWAITING_APPROVAL 状态
  - now > created_at + 72h → 转 EXPIRED
  - 写审计：dream.expired

EXPIRED 状态：
  - 不可再审批
  - 不再执行 mutations
  - 提案数据保留用于审计
```

## 9. APPLYING 阶段

```text
APPROVED → APPLYING：
  Kernel 调用 DreamWorker.describe_mutations(proposal)
  按顺序执行每个 MemoryMutation：
    - 写入新事实 / 删除旧事实 / 合并实体 / 压缩 episode / 剪枝边
  在事务内执行；任一失败 → 回滚事务 + 状态转 APPLY_FAILED

APPLY_FAILED：
  写审计：dream.apply_failed（含失败原因）
  不自动重试；可由 admin 创建新 proposal
```

## 10. 提案产出

```rust
pub trait DreamWorker: Plugin {
    fn propose(
        &self,
        scope: &MemoryScope,
        params: &DreamParams,
    ) -> Result<DreamProposal>;

    /// 仅返回变更描述，由 Kernel 执行
    fn describe_mutations(
        &self,
        proposal: &DreamProposal,
    ) -> Vec<MemoryMutation>;

    fn assess_risk(&self, proposal: &DreamProposal) -> RiskLevel;
}
```

注意：

```text
插件不能：
  直接调用 Hippocampus.put / .tombstone
  绕过 Kernel 修改任何记忆
  自行决定 risk_level（assess_risk 仅作建议，Kernel 复查）
  生成 proposal_id 与 signature（由 Kernel 颁发）
```

## 11. Mutation 类型

```rust
pub enum MemoryMutation {
    InsertFact(SemanticFact),
    SupersedeFact { old_id: FactId, new: SemanticFact },
    MergeEntities { keep: EntityId, merge: Vec<EntityId> },
    CompressEpisode { episode_ids: Vec<EpisodeId>, summary: EpisodicSummary },
    PruneTagEdges { edges: Vec<(TagId, TagId)> },
    ForkConflict { fact_id: FactId, mark_for_review: bool },
}
```

每个 Mutation 在执行前由 Kernel 校验：

```text
1. 是否在 proposal 的 mutations 列表中（防篡改）
2. 是否符合 risk_level 的允许范围
3. 是否触发 SBU 检查
4. 是否在 mandate scope 内
```

## 12. 与 ForgetEngine 的区别

```text
DreamWorker      联想式优化（合并 / 压缩 / 剪枝）
                 由 Cerebellum 周期触发
                 默认每 6 小时

ForgetEngine     遗忘式删除（用户请求 / 合规驱动）
                 由用户/admin 触发
                 详见 forget-engine.md
```

两者互不干扰；都通过 Kernel 执行 mutations。

## 13. 监控指标

```text
dream_proposals_total{status}           各状态数量
dream_proposals_blocked_by_sbu_total     SBU 阻断计数
dream_apply_failed_total                 应用失败次数
dream_avg_approval_duration_sec          平均审批耗时
dream_expired_rate                       超时占比（健康度指标）
```
