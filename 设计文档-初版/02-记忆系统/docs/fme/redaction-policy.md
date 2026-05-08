# redaction-policy.md

> 跨 Agent / 跨租户共享的脱敏策略与可审计报告。本文档修正一个关键表述：**RedactionReport 不能密码学证明"绝无遗漏"，它只能证明"某个可信执行方按某个 policy 与某个分类器版本做了脱敏，并签名留痕"。**

## 1. 设计原则

```text
脱敏策略格式标准化（fap-redaction-v1）
RedactionReport 必须含执行方签名
接收方可验证报告声明：
  - 执行方是谁
  - 用了哪个 policy / version
  - 用了哪个 classifier / version
  - 覆盖了哪个 source scope
  - 移除了多少命中项

接收方不能独立证明：
  - 源数据中所有 SBU 都被识别
  - policy 规则一定覆盖了所有危险模式
  - classifier 没有漏检

因此：
  RedactionReport = 可审计、可归责、可抽样验证
  而不是"绝对无遗漏证明"
```

## 2. 信任模型

跨 Agent 接收方的信任来源于三件事的组合：

```text
1. 信任 policy 设计质量
   - policy_id 在 trusted_policies 中
   - 规则可人工审查

2. 信任分类器质量
   - classifier_id / classifier_version 已知
   - 源侧披露扫描覆盖率（sbu_manifest）
   - 可选抽样审计

3. 信任 executor 不作恶
   - executor_did 有可验证公钥 / VC / 组织身份
   - report_signature 可验证
```

不满足任一项 → 不应将 RedactionReport 当作高置信共享依据。

## 3. 脱敏策略格式

```json
{
  "policy_id": "rp_standard_cross_agent",
  "version": "1.0",
  "schema": "fap-redaction-v1",
  "rules": [
    {
      "match": { "security_labels": ["sbu", "secret"] },
      "action": "REMOVE",
      "produce_tombstone": true
    },
    {
      "match": { "field": "owner_subject_did" },
      "action": "HASH",
      "algorithm": "SHA256",
      "salt_from": "grant_id"
    },
    {
      "match": { "content_pattern": "\\b(password|token|secret_key|api_key)\\b\\s*[:=]\\s*\\S+" },
      "action": "REDACT",
      "replacement": "[REDACTED]"
    },
    {
      "match": { "tags_include": ["pii"] },
      "action": "SUMMARIZE",
      "max_chars": 100
    },
    {
      "match": { "field": "lineage_parent_ids" },
      "action": "HASH",
      "algorithm": "SHA256",
      "salt_from": "redaction_report_id"
    }
  ],
  "produce_report": true,
  "report_schema": "fap-redaction-report-v1",
  "cross_org_safe": true
}
```

## 4. Action 类型

| Action | 行为 | 可逆？ |
|---|---|---|
| `REMOVE` | 完全移除字段或单元 | 否 |
| `HASH` | 用 SHA256(salt + value) 替换 | 不可逆 |
| `REDACT` | 用占位字符串替换 | 不可逆 |
| `SUMMARIZE` | LLM 摘要替换原文（带 max_chars） | 不可逆 |
| `MASK` | 部分字符遮罩（如 `***@***.com`） | 不可逆 |
| `KEEP` | 保留原值（默认动作，显式声明用） | - |

## 5. Match 规则

```text
{ "security_labels": ["sbu"] }       匹配含 sbu 标签的 unit
{ "field": "owner_subject_did" }     匹配字段
{ "content_pattern": "regex" }       内容正则匹配
{ "tags_include": ["pii"] }          匹配含特定 Tag 的 unit
{ "layer": "L0" }                    匹配特定层
{ "size_bytes_gt": 1048576 }         字段大小过滤
```

同一 rule 内 AND；不同 rule 按顺序执行。

## 6. Tombstone

```json
{
  "match": { "security_labels": ["sbu"] },
  "action": "REMOVE",
  "produce_tombstone": true
}
```

REMOVE + tombstone：

```text
原 MemoryUnit                          替换为
{                                       {
  memory_id: "abc",                       memory_id: "abc",
  content_text: "敏感原文",               redacted: true,
  security_labels: ["sbu"],              redaction_action: "REMOVE",
  ...                                    redaction_policy_id: "rp_xxx",
}                                        original_size_bytes: 1024
                                        }
```

接收方知道"此处曾有被策略移除的内容"，但不能恢复原值。跨组织共享时，是否包含 tombstone 由策略决定；高隐私场景可选择不暴露 tombstone，以避免泄漏"此处存在敏感内容"这一事实。

## 7. SBU Manifest（新增）

为避免宣称"完全证明无遗漏"，源侧必须可选披露分类器与扫描覆盖信息：

```json
{
  "classifier_id": "fap-me-pii-detector",
  "classifier_version": "0.3.1",
  "scan_coverage": "full",          // full | sampled | partial
  "sample_rate": 1.0,
  "patterns_count_scanned": 142,
  "notes": "tenant override: medical policy pack enabled"
}
```

规则：

```text
租户内共享              sbu_manifest 可选
跨 Agent 同租户         推荐提供
跨租户 / 跨组织共享      强制提供
```

## 8. RedactionReport

```json
{
  "report_id": "rr_abc123",
  "schema": "fap-redaction-report-v1",
  "policy_id": "rp_standard_cross_agent",
  "policy_version": "1.0",
  "applied_at": "2026-05-08T10:00:00Z",
  "executor_did": "did:web:gateway.example",
  "source_scope_hash": "sha256:9f86d081...",

  "sbu_manifest": {
    "classifier_id": "fap-me-pii-detector",
    "classifier_version": "0.3.1",
    "scan_coverage": "full",
    "sample_rate": 1.0,
    "patterns_count_scanned": 142
  },

  "actions_taken": [
    {
      "rule_index": 0,
      "count": 3,
      "memory_ids_hashed": ["sha256:..."]
    },
    {
      "rule_index": 1,
      "count": 12,
      "fields_affected": ["owner_subject_did"]
    }
  ],

  "audit_sample_hashes": [
    { "memory_id_hash": "sha256:...", "decision": "removed" },
    { "memory_id_hash": "sha256:...", "decision": "kept" }
  ],

  "sbu_units_removed": 3,
  "sbu_tombstones_included": 3,
  "total_units_before": 47,
  "total_units_after": 44,
  "report_signature": "JWS..."
}
```

## 9. 签名

```text
report_signature = JWS({
  alg: "EdDSA",
  kid: executor_did + "#key-1",
  payload: SHA256(canonical(report 除 signature 字段))
})
```

签名只能证明：

```text
✅ 该 executor_did 签过这个报告
✅ 报告在传输途中未被改动

不能证明：
❌ 源数据没有漏检 SBU
❌ executor 使用的是"最正确"的 policy
```

## 10. 验证流程

```rust
impl PolicyKernel {
    pub fn verify_redaction_report(
        &self,
        report: &RedactionReport,
        original_scope: &MemoryScope,
    ) -> Result<(), RedactionVerifyError> {
        // 1. 签名验证
        let pubkey = resolve_did_key(&report.executor_did)?;
        verify_jws(&report.report_signature, &pubkey, &canonical_payload(report))?;

        // 2. 策略可信度
        if !self.trusted_policies.contains(&report.policy_id) {
            return Err(RedactionVerifyError::UntrustedPolicy);
        }

        // 3. 范围一致
        if report.source_scope_hash != hash(&original_scope) {
            return Err(RedactionVerifyError::ScopeMismatch);
        }

        // 4. 跨租户/跨组织必须有 manifest
        if original_scope.cross_tenant && report.sbu_manifest.is_none() {
            return Err(RedactionVerifyError::MissingSbuManifest);
        }

        // 5. 最低分类器版本要求（来自 mandate constraint）
        if let Some(min_v) = original_scope.classifier_version_min {
            let actual = report.sbu_manifest.classifier_version;
            if semver_lt(actual, min_v) {
                return Err(RedactionVerifyError::ClassifierVersionTooLow);
            }
        }

        // 6. 只能验证"按此 policy/manifest 执行"，不能证明绝对无遗漏
        Ok(())
    }
}
```

## 11. 抽样审计

对于高风险共享（cross_tenant / cross_org / compliance），建议增加抽样审计：

```text
1. 源侧从已处理的 MemoryUnit 中按固定 seed 抽样 1~5%
2. 记录抽样哈希 + decision 到 audit_sample_hashes
3. 接收方或第三方审计者可要求源侧提供原文对照（在合规域内）
4. 审计者验证：
   - 该样本确实被 remove/redact
   - 未声称 full coverage 但实际 sampled
```

这不是运行时强制协议，而是治理建议。

## 12. 常用策略库

```text
config/redaction_policies/
├── standard_cross_agent.json
├── compliance_audit.json
├── handoff_default.json
├── medical.json
└── financial.json
```

`trusted_policies` 由 tenant 或组织 owner 明确列出。

## 13. 性能

```text
策略编译：启动时编译为内部 IR
正则缓存：所有 content_pattern 预编译
执行复杂度：O(rules × units × match_kinds)

性能预算：
  100 units / 5 rules → < 5ms
  1000 units / 5 rules → < 30ms

超时策略：
  超时 → 切到更严格策略（保守）+ 写审计
  绝不绕过策略
```

## 14. 与 ContentSafetyGuard 的区别

```text
ContentSafetyGuard       写入前阻断式（拒绝写入）
RedactionPolicy          共享/读取前变换式（脱敏视图）
```

详见 [content-safety.md](./content-safety.md)。

## 14.1 SBU 过滤责任分布（检索路径三层职责）

检索路径上 SBU 过滤分散在三处，以下为权威分工。任何实现把这三层合并、跳过、或重排即违反 [kernel-contract.md](./kernel-contract.md)。

```text
┌──────────────────────────────────────────────────────────────────────┐
│ Layer 1：候选池打分阶段（ranking）                                       │
│   位置：scoring-formula §3 中的 sensitivity_penalty                    │
│   职责：以 w_sbu = 1.0 把 SBU 候选挤到分数末端                            │
│   输入：候选 + mandate.sbu_access                                        │
│   输出：调整后的 final_score                                              │
│   不做：删除 / 重写 / 隐去原文                                             │
│   边界：仅作排序信号，不能视为安全保障；候选仍含 SBU 原文                       │
└──────────────────────────────────────────────────────────────────────┘
                              ↓
┌──────────────────────────────────────────────────────────────────────┐
│ Layer 2：返回路径前的策略脱敏（PolicyKernel.redact）                      │
│   位置：10-security-model §3 / 06-control-plane §3                     │
│   职责：按 RedactionPolicy 执行 REMOVE / HASH / REDACT / SUMMARIZE       │
│   输入：候选 + grant.redaction_policy_id 或 tenant 默认 policy            │
│   输出：脱敏后视图 + RedactionReport                                      │
│   不做：决定是否需要脱敏（由 mandate.sbu_access 决定，PolicyKernel 调度）     │
│   边界：插件不得参与；策略未匹配命中也不能保证零残留（详见 §1）                  │
└──────────────────────────────────────────────────────────────────────┘
                              ↓
┌──────────────────────────────────────────────────────────────────────┐
│ Layer 3：sbu_safe 模式的最后一道硬过滤                                    │
│   位置：08-retrieval-modes §4.7                                        │
│   职责：在 Layer 2 结果上再做 security_labels 与 mandate.sbu_access 校验    │
│   输入：脱敏后视图                                                         │
│   输出：去掉所有仍带 sbu / secret 标签的项                                  │
│   不做：策略变换（已由 Layer 2 处理）                                        │
│   边界：sbu_safe 不可降级（fap-me/sbu-safe-cannot-fallback）                │
└──────────────────────────────────────────────────────────────────────┘
```

各层之间的不变式：

| 不变式 | 谁保证 |
|---|---|
| Layer 1 不能替代 Layer 2（penalty 不是脱敏） | scoring-formula 注释 + Conformance 用例 |
| Layer 2 必须经过 PolicyKernel.redact，插件不得改写 | kernel-contract §3 |
| Layer 3 不假设 Layer 2 已完美脱敏（不可依赖前提） | 08-retrieval-modes §4.7 |
| `mandate.sbu_access = DENY` 在三层任一处命中即返回空 / 拒绝 | PolicyKernel.authorize |
| `RAW_ALLOWED` 仅 elevated mandate + 强审计；三层全部跳过 redact 仍要写 `sbu.raw_returned` 审计 | PolicyKernel.authorize + AuditKernel |

实现 checklist：

```text
✅ Layer 1 不读 grant 与 redaction_policy_id（仅看 mandate.sbu_access）
✅ Layer 2 不修改候选打分；只做内容变换并产出 RedactionReport
✅ Layer 3 仅做白名单 / 黑名单过滤，不依赖 RedactionPolicy
✅ 任一层异常 → 返回空结果 + 写审计；不允许"按部分结果返回"
```

## 15. 失败模式

| 失败 | 行为 |
|---|---|
| policy 解析失败 | 拒绝创建 Grant，写审计 |
| 签名验证失败 | 接收方拒绝 handoff，写审计 |
| source scope hash 不一致 | 拒绝，告警 |
| policy 未登记在 trusted_policies | 拒绝跨 Agent 使用 |
| 跨租户共享缺少 sbu_manifest | 拒绝 |
| classifier_version 低于要求 | 拒绝 |

## 16. 与最终版的差距

```text
最终版                                  本文档
-----                                   -----
宣称接收方可独立证明无 SBU 遗漏         改为：可审计、可归责、可抽样验证
无 classifier/version 披露              sbu_manifest 强化信任模型
无抽样审计建议                           audit_sample_hashes + 源侧抽样
redaction_policy 只是 ID 引用            完整 JSON Schema + 动作语义
sbu_redaction_report 格式未定义          完整 report schema + 签名
```
