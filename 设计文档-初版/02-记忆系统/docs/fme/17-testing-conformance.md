# 17 - 测试与一致性门禁

## 1. 测试金字塔

```text
单元测试           各 crate 内部
集成测试           整合 Kernel + Plugin + Storage
Conformance        负例 / Kernel Contract 验证
性能测试           criterion + 三档基线
模糊测试           cargo-fuzz（Envelope / Mandate / Redaction）
E2E                跨 Agent Handoff / 跨组织 DID-VC
```

## 2. 强制门禁（必须通过）

### 2.1 Kernel Contract 负例（不可妥协）

```text
插件试图绕过 PolicyKernel 直接读 MemoryUnit  → 拒绝
插件试图绕过 ContentSafetyGuard 写入        → 拒绝
插件试图直接生成 AuditEvent                  → 拒绝
插件试图修改 chain_state.current_tip         → 拒绝
插件试图访问其他 tenant 数据                 → 拒绝
插件试图访问其他 plugin 状态                 → 拒绝
插件试图伪造 FAP-1 Mandate signature          → 拒绝
插件试图复用过期 capability-scoped handle     → 拒绝
插件试图重排 PolicyKernel 判定顺序           → 拒绝
插件试图写 ContextGrant 表                   → 拒绝
插件试图直接删除 raw content                 → 拒绝
```

### 2.2 安全门禁

```text
含 "Ignore previous instructions" 的内容写入  → 拒绝
超长内容（> 32KB）                              → 拒绝
embedding 输入超 token 限制                    → 拒绝
SBU 标签内容自动晋升 L2                         → 拒绝
RedactionReport 签名错误                       → 拒绝
ContextGrant 过期后访问                        → 拒绝
未授权 capability/layer/action 三元组访问      → 拒绝
mandate 已撤销但仍在 60s 内被使用              → 必须拒绝
```

### 2.3 数据完整性

```text
audit chain 断链检测                          → 必须告警
embedding 模型升级后 index_version 不匹配      → 必须拒绝读取
L2 写入 stability < 0.6                       → 必须拒绝
HardForget 本地提交后 raw / embedding / snapshot / 本地 CAS 仍存在 → 测试失败
HardForget 外部系统未入 mutation ledger         → 测试失败
冲突 fact 未走 resolve_l2_conflict             → 测试失败
```

### 2.4 性能门禁（参考）

| 测试 | 阈值 |
|---|---|
| basic P99 | ≤ 20ms |
| hybrid P99 | ≤ 60ms |
| tagmemo P99 | ≤ 120ms |
| store + audit P99 | ≤ 100ms |
| 后台任务对主路径 P99 增量 | ≤ 5ms |
| ContentSafetyGuard 单次 | ≤ 5ms |
| RedactionReport 验签 | ≤ 10ms |

## 3. 单元测试覆盖率

```text
fap-me-kernel             ≥ 95%
fap-me-core               ≥ 85%
fap-me-plugin             ≥ 80%
fap-me-retrieval          ≥ 85%
fap-me-data-model         ≥ 80%
plugins/*                 ≥ 70%
```

## 4. 集成测试场景

### 4.1 完整写入与检索

```text
1. Agent A → FAP-1 InvokeRequest(capability_id = memory.store)
2. 触发 CompactStrategy.should_fold
3. fold → L1 episode 写入
4. embedding + Cortex.add + Synapse.upsert
5. AuditKernel.append
6. Agent A → FAP-1 InvokeRequest(capability_id = memory.retrieve, mode=hybrid)
7. 验证返回 RecallResult 含 ScoreBreakdown
8. 验证审计链中存在两个事件（store + retrieve）
```

### 4.2 跨 Agent Handoff

```text
1. Agent A 创建 ContextSnapshot
2. Agent A 申请 ContextGrant（含 RedactionPolicy）
3. PolicyKernel.redact 产出 RedactionReport
4. Agent A 创建 HandoffPacket（含 RedactionReport）
5. Agent B 验证 packet_signature
6. Agent B 调用 PolicyKernel.verify_redaction_report
7. Agent B redeem grant，读取脱敏后内容
8. 验证未授权 SBU 原文不出现在 Agent B 的访问中
9. 审计链中存在 share / handoff / redeem 三个事件
10. 跨租户场景下 RedactionReport 必须含 sbu_manifest
```

### 4.3 SBU 强制遗忘

```text
1. 写入含 SBU 标签的 MemoryUnit
2. 该 unit 的 lineage 衍生 L1 episode、L2 fact、HNSW vector、ContextGrant
3. HardForget 请求
4. ForgetEngine.plan 返回完整级联清单
5. Kernel 执行本地事务 mutations，签发 COMMITTED_LOCALLY ForgetReceipt
6. Reconciler 处理 S3/Qdrant/Kafka 等外部 mutation ledger
7. 验证：
   - raw content 不可读
   - embedding 不在索引中
   - snapshot 不存在
   - grant 已撤销
   - audit 链有 ForgetReceipt 且绑定 fap_receipt_id
   - 外部系统达成后 receipt 状态更新为 GLOBALLY_RECONCILED
```

### 4.4 DreamProposal 审批

```text
1. DreamWorker.propose 返回含 SBU 内容的 proposal
2. 验证 proposal.status = BLOCKED 且 sbu_blocked = true
3. 任何审批尝试均拒绝
4. 另一个 LOW 风险的 proposal
5. 验证默认配置下自动 APPROVED → APPLIED
6. HIGH 风险 proposal 等待人工审批
7. 72h 后自动 EXPIRED
```

## 5. Conformance 测试矩阵

| 维度 | 用例数 |
|---|---:|
| Kernel Contract 负例 | ≥ 30 |
| 安全门禁 | ≥ 20 |
| Mandate 验证 | ≥ 15 |
| Redaction 校验 | ≥ 10 |
| Forget 级联 | ≥ 10 |
| Dream 状态机 | ≥ 12（覆盖每个转移） |
| 检索降级 | ≥ 8 |
| 多租户隔离 | ≥ 10 |
| Embedding 模型升级 | ≥ 5 |
| 审计链完整性 | ≥ 8 |

总计 ≥ 128 个 Conformance 用例必须通过才能 Phase 验收。

## 6. 模糊测试

```text
cargo-fuzz 目标：
  - Envelope 解码（防畸形输入）
  - Mandate JSON 解析
  - RedactionPolicy 规则匹配
  - HNSW 索引文件解析
  - TagGraph CSR mmap 解析

运行：每次 PR ≥ 5 分钟，每周 ≥ 24h
```

## 7. 性能基线

```text
工具：criterion + flamegraph

测试集：
  - 1K MemoryUnit / tenant（小规模）
  - 100K MemoryUnit / tenant（中等）
  - 10M MemoryUnit / tenant（大规模，仅 enterprise profile）

指标：
  recall_latency_p50 / p99
  store_latency_p50 / p99
  hit_rate@5 / @10
  redaction_correctness（人工标注子集）
  chain_integrity_verify_duration
```

## 8. CI 门禁

```text
必须通过才能合并 PR：
  ✅ 单元测试
  ✅ 覆盖率门槛
  ✅ Kernel Contract 负例（128+ 用例）
  ✅ Schema breaking 检测（与 Phase N-1 对比）
  ✅ Manifest 签名验证
  ✅ cargo-fuzz 5 分钟
  ✅ 性能基线（不允许 P99 退化 > 10%）
  ✅ Lint（clippy --deny warnings）
  ✅ 文档测试（cargo test --doc）
```

## 9. Phase 验收标准

每个 Phase 必须达到对应章节的"验收标准"。详见 [18-roadmap.md](./18-roadmap.md)。

## 10. 互操作性测试

```text
与 FAP-1 不同实现互操作：
  - Rust 实现 ⇄ Java SDK
  - Edge Lite ⇄ Enterprise（升级路径）
  - Embedding v1 ⇄ Embedding v2（双索引切换）

跨组织：
  - DID/VC 验证
  - VC 撤销流程
  - DPoP token 绑定
```

## 11. 故障注入

```text
对 Cerebellum 注入：
  - 后台任务超时
  - DreamWorker 失败
  - ForgetEngine 部分级联失败
  - 索引重建中途宕机

对 Hippocampus 注入：
  - SQLite 写满
  - WAL 损坏
  - 主从切换

对 Plugin 注入：
  - WASM 内存超限
  - WASM 超时
  - 网络插件超时

验证：
  - 主路径不被阻塞
  - 审计链完整
  - 回滚或重试逻辑正确
```
