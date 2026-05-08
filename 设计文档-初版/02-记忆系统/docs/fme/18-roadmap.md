# 18 - 实施路线图

> 与 FAP-1 [21-roadmap.md](../../../01-协议设计/docs/fap/21-roadmap.md) 对齐：FAP-ME Phase N 在 FAP-1 Phase N 完成后启动或并行。

## 1. 总览

```text
Phase 1   Kernel Contract + FAP-1 绑定门禁   3 周
Phase 2   L0/L1 基础记忆闭环                 3 周
Phase 3   Seahorse-style 认知检索增强        5 周
Phase 4   上下文共享与 Handoff               3 周
Phase 5   安全合规增强                       4 周
Phase 6   DreamWorker + 后台维护 + 生产化    5 周
                                              ─────
                                              23 周
```

## 2. Phase 1：Kernel Contract + FAP-1 绑定门禁（6-8 周）

> 原计划 3 周不现实——交付物覆盖 5 个 Kernel 组件 + WASM 沙箱 + 30+ 个 Conformance + Schema 冻结 + outbox 闭环 + 多语言 SDK。本 Phase 拆为四道顺序门禁，按工程量重新评估为 6-8 周（含两周 Schema 冻结对账）。

未通过前一道，不进入下一道：

```text
Gate 1  FAP-1 Binding Freeze（≈ 1.5 周）
        - 所有外部调用走 FAP-1 InvokeRequest
        - 不暴露独立 MemoryControlService
        - FAP-1 Mandate + FME constraints schema 冻结
        - capability/layer/action 三元组冻结（含 enum 整数值）
        - capability_id 全表（含 .create / .receive / .soft / .hard 拆分）冻结
        - signing-canonical.md 测试向量发布

Gate 2  Kernel Security Baseline（≈ 2 周）
        - PolicyKernel / TenantKernel / MandateVerifier / ContentSafetyGuard 不可插件化
        - capability-scoped plugin handle 生效
        - tenant_id 不信任请求体
        - chain_state 并发模型（CAS + base_revision）落地
        - chain_state_registry 注册中心可枚举

Gate 3  Audit & Receipt Binding（≈ 1.5 周）
        - AuditEvent 包含 fap_receipt_id / fap_invocation_id / fap_envelope_hash
        - AuditEvent canonical encoding 与 signing-canonical 一致（Conformance 测试向量通过）
        - 高风险 finality 策略落地（FINALIZED 默认）
        - 周期 chain integrity verify 调度

Gate 4  Data Plane & Outbox Baseline（≈ 2 周）
        - 大对象与流式 chunk 复用 FAP-1 Data Plane
        - ForgetReceipt 双状态 schema + 双签名落地
        - mutation_ledger / outbox reconciler 最小闭环
        - reconciler leader election + sharding 验收
        - ADVISORY_DELIVERED 状态在跨 Agent 通知场景跑通
```

### 交付物

```text
✅ Kernel 层完整实现
  - PolicyKernel
  - AuditKernel（含哈希链）
  - TenantKernel
  - MandateVerifier
  - ContentSafetyGuard

✅ PluginRegistry 框架
  - 注册 / 发现 / 版本管理
  - 多租户级配置覆盖
  - WASM 沙箱（Wasmtime）

✅ FAP-ME Payload Schema 扩展
  - StandardPurpose 枚举
  - CapabilityTriple
  - RedactionPolicy / RedactionReport
  - DreamProposal 状态机
  - HandoffPacket（含 RedactionReport）
  - ForgetReceipt 双状态

✅ Purpose 词汇表 JSON Schema
✅ Agent Card memory 子节定义
✅ fap-me-server 骨架（axum + tonic）
✅ TypeScript / Python SDK 类型骨架（只封装 FAP-1 Invoke）
```

### 验收标准

```text
1. 任何绕过 PolicyKernel 直接访问存储的路径在 Conformance 用例中均被拒绝
   （编译期 lint 与运行期负例两层保证；不要求"无法编译"绝对承诺）
2. 插件注册版本冲突时系统拒绝启动
3. ContentSafetyGuard 能拦截 ≥ 10 个标准 Prompt Injection 模式
4. Protobuf 可生成 Go / Rust / TypeScript SDK 类型
5. FAP-1 Receipt 与 FME AuditEvent 可端到端互相定位
6. Conformance 负例 ≥ 30 个全部通过
7. signing-canonical 测试向量集所有实现 byte-for-byte 一致
8. capability_id ↔ CapabilityTriple 推导被穷举测试（至少覆盖 12 项 capability）
```

## 3. Phase 2：L0/L1 基础记忆闭环（3 周）

### 交付物

```text
✅ L0 RingBuffer + JSONL WAL
✅ DefaultCompactStrategy
  - should_fold 4 级触发逻辑
  - retain_score 公式
  - L0 → L1 折叠

✅ L1 EpisodeStore（SQLite WAL）
✅ EmbeddingProvider 插件
  - openai 适配
  - local sentence-transformers 适配

✅ VectorIndex 插件
  - HNSW (usearch)
  - sqlite-vec（Core Lite）

✅ append-only AuditSink 插件
✅ mTLS + JWT 认证
✅ basic + hybrid 检索模式
```

### 验收标准

```text
1. L0 折叠在以下条件正确触发：
   - token > 4096
   - 轮次 ≥ 10
   - semantic_boundary_score > 0.70
   - 静默 > 300s
2. L1 Episode 可按 tenant / namespace 检索
3. 每次 retrieve / store 生成带哈希链且绑定 FAP-1 Receipt 的 AuditEvent
4. 不同 tenant 的记忆无法交叉查询
5. Prompt Injection 内容被拒绝或隔离，不进入默认可召回 L0
6. 性能：basic P99 ≤ 20ms / hybrid P99 ≤ 60ms
```

## 4. Phase 3：Seahorse-style 认知检索增强（5 周）

### 交付物

```text
✅ Thalamus（Tide 算法）
  - Gram-Schmidt 多级剥离
  - 投影熵
  - 引力场修正
  - 完整数学规格

✅ ColdHotTagGraph
  - 冷热分离
  - V7 有向序位拓扑
  - intrinsic_residual 计算

✅ LIF SpikeEngine
  - MAX_HOPS = 2
  - FIRING_THRESHOLD = 0.1
  - 收敛检测

✅ GeodesicReranker
  - 复用 LIF 距离场
  - 零额外计算
  - alpha 热调参

✅ RetrievalPipeline
  - 降级链（tagmemo_geodesic → tagmemo → hybrid → basic）
  - 背压
  - 超时控制

✅ 霰弹枪查询（并发上限 8 / 单次超时 100ms）
✅ tide / tagmemo / tagmemo_geodesic / sbu_safe / dream 模式
```

### 验收标准

```text
1. tide 模式返回 TideResult（含 focus_level / layers / weak_signal_tags）
2. tagmemo_geodesic 能正确处理多义词（金融"银行" vs 河岸"银行"）
3. spike_pathway 在结果中可见（可解释性）
4. 任何检索模式超时后自动降级，不返回空结果
5. 霰弹枪并发不超过 8，候选池不超过 500
6. tagmemo P99 ≤ 120ms / tide P99 ≤ 150ms
```

## 5. Phase 4：上下文共享与 Handoff（3 周）

### 交付物

```text
✅ ContextSnapshot
✅ ContextGrant（含签名 + revoke）
✅ RedactionPolicy 执行引擎
✅ RedactionReport 生成（含 JWS 签名）
✅ HandoffPacket（含可验证脱敏报告）
✅ PolicyKernel.verify_redaction_report
✅ foreign_context import 策略
```

### 验收标准

```text
1. Agent A 可安全移交任务给 Agent B
2. B 收到 HandoffPacket 后可验证脱敏报告签名
3. 未授权 SBU 原文不出现在任何跨 Agent 上下文中
4. ContextGrant 过期后不可再访问
5. ContextGrant 撤销后 ≤ 60s 全集群生效
6. redaction_policy 格式符合 fap-redaction-v1 schema
7. 审计链中包含 share / handoff / redeem 完整事件
```

## 6. Phase 5：安全合规增强（4 周）

### 交付物

```text
✅ DID Resolver
✅ VC Verifier
✅ DPoP adapter
✅ FAP-1 Mandate verifier（含 FME constraints 与 purpose 词汇表校验）
✅ SBU 强制遗忘流水线
  - 本地强一致：raw → embedding → snapshot → grant → 本地 CAS
  - 跨系统最终一致：S3 / Kafka / Qdrant / 远端 sink outbox
✅ ForgetReceipt 生成（含 local/global 双状态签名）
✅ AuditKernel 完整性验证
  - 周期哈希链验证（每天）
  - 断链检测
  - TamperingAlert 告警
✅ chain_integrity_verify capability
```

### 验收标准

```text
1. 未授权 purpose 无法读取记忆
2. HardForget 后本地 raw / embedding / snapshot / grant / 本地 CAS 在单事务内级联清除
3. 返回可验证 ForgetReceipt；默认状态可为 COMMITTED_LOCALLY，外部系统通过 reconciler 达成 GLOBALLY_RECONCILED
4. 审计链完整性每天自动验证，断链时告警
5. 跨组织 Agent 可用 DID/VC 建立会话
6. mandate 撤销 ≤ 60s 全集群可见
```

## 7. Phase 6：DreamWorker + 后台维护 + 生产化（5 周）

### 交付物

```text
✅ DreamWorker 插件接口
✅ DefaultDreamWorker 实现（规则 + LLM 提案）
✅ DreamProposal 完整状态机
  - 7 状态 + 转移
  - 风险分级（LOW / MEDIUM / HIGH）
  - SBU 永久阻断
  - 72h 超时
✅ 审批者身份验证
✅ Cerebellum 调度器
  - 优先级队列
  - 重试 / 幂等
  - 不阻塞主路径
✅ L3 后台任务
  - PruneTagGraph
  - CompactMemory
  - HealthAnalyze
✅ 观测
  - Prometheus 指标
  - OpenTelemetry trace
✅ WASM 插件沙箱（生产化）
  - 内存 64MB
  - 超时 200ms
  - 网络/文件系统禁止
```

### 验收标准

```text
1. 含 SBU 内容的 DreamProposal 自动 BLOCKED 且不可审批通过
2. 高风险 proposal 必须人工审批，超时 72h 自动 EXPIRED
3. 后台任务不阻塞 recall 主路径（P99 增量 ≤ 5ms）
4. 可按 tenant 统计 recall_latency / hit_rate / dream_jobs_total
5. WASM 插件超限（内存或时间）时被沙箱终止，不影响主进程
6. Conformance 用例 ≥ 128 全部通过
```

## 8. Phase 间依赖

```text
Phase 1 是所有后续阶段的前提（必须先完成）
Phase 2 依赖 Phase 1
Phase 3 依赖 Phase 2 的 L1 + EmbeddingProvider + VectorIndex
Phase 4 依赖 Phase 1 Gate 3/4 与 Phase 2 的 ContextSnapshot；不依赖 Phase 3 全部高级检索
Phase 5 的 Mandate/Redaction/Forget 基线已在 Phase 1 Gate 中落地；Phase 5 负责生产级强化
Phase 6 依赖前 5 个 Phase
```

## 9. 与 FAP-1 协议路线图的关系

```text
FAP-1 Phase 0   协议冻结              FAP-ME Phase 1 启动前提
FAP-1 Phase 1   Core Lite             FAP-ME Phase 1-2 并行
FAP-1 Phase 2   高性能 Profile        FAP-ME Phase 3 启动前提
FAP-1 Phase 3   分布式节点            FAP-ME Phase 5 部分功能依赖
FAP-1 Phase 4   生产化                FAP-ME Phase 6 同期
```

## 10. 不在 V1 范围

```text
EXT-潜空间记忆                  实验性
EXT-联邦学习记忆共享            实验性
EXT-跨节点 CRDT 记忆同步        FAP-1 EXT 范围
EXT-多模态嵌入                  Plugin 可后续注册
```

## 11. 回滚策略

详见 FAP-1 [22-rollback.md](../../../01-协议设计/docs/fap/22-rollback.md)。

FAP-ME 特有回滚：

```text
embedding 模型升级失败：
  → 双索引切换中切回 v1
  → 不影响已有 L1 / L2

CompactStrategy 升级失败：
  → 切回上一版本，pending L0 不再折叠
  → 后台清理任务暂停

DreamWorker 升级失败：
  → 当前进行中的 proposal 标记 REJECTED
  → 不丢失数据
```

## 12. 关键里程碑

```text
W6-W8   FAP-1 绑定、Kernel Contract、审计绑定、outbox 基线通过
        （signing-canonical 测试向量 + chain-state-concurrency 落地）
W9-W11  L0/L1 闭环，basic + hybrid + sbu_safe 上线
W12-W16 认知检索全模式上线，TagMemo V7 + 测地线重排可用
W17-W19 Handoff + RedactionReport 完整链路通过
W20-W23 合规 + DID/VC + SBU 强制遗忘通过
W24-W28 DreamWorker + 完整生产化，进入 GA
```

> 与原 23 周相比延后约 5 周——主要在 Phase 1 重新计入 Schema 冻结对账与 chain_state 并发实现的工程量。后续 Phase 不变。
