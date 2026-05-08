# kernel-contract.md

> FAP-ME Kernel Contract — 定义哪些组件**必须**留在 Core，哪些**可以**作为 Plugin。本文档对齐 FAP-1 [kernel-contract.md](../../../01-协议设计/docs/fap/kernel-contract.md)。

## 1. 原则

可插拔架构 ≠ 一切可替换。若安全判定（Policy）、审计写入（Audit）、租户隔离（Tenant）、内容守卫（ContentSafety）、FAP-1 Mandate 约束校验、Replay 防护可被插件替换，整个 FAP-ME 的安全模型将被架空。

```text
插件可以提供：
  算法（embedding / 向量索引 / 拓扑扩散 / 折叠 / 精排 / 梦境提议 / 遗忘规划）
  外部适配（具体 vector DB / object store / audit sink）

插件不能提供：
  安全判定的顺序
  安全判定的规则
  租户隔离的过滤逻辑
  哈希链生成与验证
  FAP-1 Mandate + FME constraints 验证
  内容安全检查
  插件自身的权限裁决
```

## 2. Kernel 必留组件

| 组件 | 必留理由 |
|---|---|
| MandateVerifier | FAP-1 Mandate 签名/过期/撤销及 FME constraints 校验，凭证不能由插件验证 |
| PolicyKernel | 决策顺序固定：MandateVerify → CapabilityTriple Authorize → TenantFilter → ContentSafety |
| TenantKernel | 强制 SQL 过滤注入，插件不能拼装查询 |
| ContentSafetyGuard | Prompt Injection 防护，写入路径不可绕过 |
| AuditKernel | hash chain 必须由 Core 原子生成与验证 |
| Plugin Permission Arbiter | 插件不能判定自己的权限 |
| Header Validator | message_id / created_at / signature 顺序固定 |

## 3. Plugin 提供点

| 扩展点 | Plugin Trait | 作用域 |
|---|---|---|
| 向量化 | EmbeddingProvider | 只产出 embedding |
| 向量索引 | VectorIndex | 只提供 add / search / mark_deleted / rebuild |
| 标签拓扑 | TagGraph | 只提供 upsert / expand / prune（含 V7 有向） |
| L0 折叠 | CompactStrategy | 只提供 should_fold / fold / retain_score |
| 精排 | Reranker | 只提供 rerank |
| 梦境 | DreamWorker | 只 propose / describe_mutations / assess_risk，不写入 |
| 遗忘 | ForgetEngine | 只 plan / mutations，不删除 |
| 审计 Sink | AuditSink | 只负责持久化 SignedAuditEvent |
| 对象存储 | ObjectStore | 只 put / get / delete / lease |

注意：`PolicyEngine` **不**在插件清单中——这是与最终版方案的关键区别，参见 `../../3-记忆系统深度分析与未来方案.md` §3.1 与 §4.3。

## 4. 判定顺序不可变

```text
固定顺序（Kernel 内硬编码）：
  1. Transport TLS (FAP-1)
  2. Envelope Decode (FAP-1)
  3. Header Validation
  4. Replay Cache (FAP-1)
  5. Session Check
  6. AuthN（FAP-1 AuthPlugin，仅协议级）
  7. AuthZ（FAP-1 PolicyPlugin，仅协议级）
  8. MandateVerifier（FME Kernel，校验 FAP-1 Mandate + FME constraints）
  9. PolicyKernel.authorize（含 purpose 词汇表与 capability triple）
 10. TenantKernel filter 注入
 11. ContentSafetyGuard.check_write（仅写入路径）
 12. Plugin Sandbox / Sidecar Boundary
 13. Memory Orchestrator → Plugin 调度（只传 capability-scoped handle）
 14. PolicyKernel.redact（仅检索返回）
 15. AuditKernel.append
```

任何插件**只能在某一步被询问**，不能跳过或重排。

## 5. 插件无法访问的资源

```text
其他 tenant 的状态
其他 plugin 的私有状态
audit_event 表的直接写入权
chain_state.current_tip 的直接修改权
mandate / context_grant / forget_receipt 表的直接写入权
ContentSafetyGuard 的拦截规则
RedactionPolicy 的执行决定
FAP-1 Receipt 与 FME AuditEvent 的绑定字段
```

## 6. Plugin Permission Arbiter

```text
插件发起任何 host API 调用时：
  → Arbiter 查 manifest.permissions
  → 再查本次调用的 capability-scoped handle
  → 决策 allow / deny
  → 记录审计

插件不能：
  绕过 Arbiter
  自行声明"我需要这个权限"
  修改自己的 manifest
  访问 Arbiter 的内部状态
  把一次调用拿到的 handle 缓存到后续请求复用
```

## 7. 反面教材（禁止模式）

```text
错误：CompactStrategy 自己决定是否包含 SBU
对：CompactStrategy 看到的是 PolicyKernel 已脱敏的视图

错误：DreamWorker 直接调用 Hippocampus.put 写入新 fact
对：DreamWorker 返回 MemoryMutation 列表；Kernel 校验 + 执行

错误：ForgetEngine 直接 DELETE FROM memory_unit
对：ForgetEngine 返回级联清单；Kernel 在本地事务中执行本地 mutation，并为外部系统写 mutation ledger

错误：VectorIndex 自己拼接 WHERE tenant_id = ...
对：VectorIndex 接收已注入 filter 的 SearchFilter 参数

错误：第三方插件以 runtime = rust_native 动态加载 .dll/.so
对：第三方插件使用 WASM 或 sidecar；rust_native 仅允许 trusted/builtin 插件

错误：AuditSink 决定哪些 event 要写哪些丢弃
对：AuditKernel 决定写入策略，AuditSink 只负责持久化

错误：EmbeddingProvider 自己缓存 embedding 用作 PII 推断
对：EmbeddingProvider 只产 embedding，不持有内容副本

错误：Reranker 看到原始 SBU 内容
对：Reranker 看到 PolicyKernel 已脱敏的 candidate 视图

错误：DreamWorker 直接修改 mandate 来获得新权限
对：DreamWorker 不能访问 mandate 表；任何权限请求需新 mandate
```

## 8. Kernel Contract 测试

```text
MUST 通过的负例测试（≥ 30 用例）：
  插件试图访问其他 tenant                   → 拒绝
  插件试图修改 chain_state                  → 拒绝
  插件试图跳过 FAP-1 Mandate constraints 校验 → 拒绝
  插件试图伪造 redaction signature          → 拒绝
  插件试图直接写 ContextGrant               → 拒绝
  插件试图直接 DELETE memory_unit           → 拒绝
  插件试图重排 PolicyKernel 顺序            → 拒绝
  插件试图绕过 ContentSafetyGuard           → 拒绝
  插件试图直接生成 AuditEvent               → 拒绝
  插件试图修改自己的 manifest               → 拒绝
  插件试图读取其他 plugin 的私有状态        → 拒绝
  插件试图自行验证 RedactionReport          → 拒绝（必须经 PolicyKernel）
  插件试图升级 SBU 内容到 L2                → 拒绝
  插件试图生成 forget_receipt               → 拒绝（仅 Kernel 签）
  ...
```

详见 [17-testing-conformance.md](./17-testing-conformance.md) 中的"Kernel Contract 负例"门禁。

## 9. PolicyKernel 接口（不可替换实现）

```rust
pub struct PolicyKernel {
    tenant_store: Arc<TenantStore>,
    mandate_verifier: Arc<MandateVerifier>,
    sbu_classifier: Arc<SbuClassifier>,
    purpose_registry: Arc<PurposeRegistry>,
}

impl PolicyKernel {
    /// 唯一授权入口。op 必须是 capability/layer/action 三元组。
    pub fn authorize(
        &self,
        session: &Session,
        op: CapabilityTriple,
        scope: &MemoryScope,
    ) -> Result<AuthorizedOp, PolicyError>;

    /// SBU 脱敏（产出可验证 RedactionReport）
    pub fn redact(
        &self,
        units: Vec<MemoryUnit>,
        session: &Session,
        policy_id: &RedactionPolicyId,
    ) -> (Vec<RedactedMemoryUnit>, RedactionReport);

    /// 验证跨 Agent 脱敏报告
    pub fn verify_redaction_report(
        &self,
        report: &RedactionReport,
        original_scope: &MemoryScope,
    ) -> Result<(), RedactionVerifyError>;
}
```

## 10. AuditKernel 接口（不可替换实现）

```rust
pub struct AuditKernel {
    sinks: Vec<Arc<dyn AuditSink>>,            // sink 可插件，逻辑不可
    chain_store: Arc<dyn ChainStateStore>,     // 按 tenant/namespace 分片
    fanout_policy: AuditFanoutPolicy,
    integrity_scheduler: Arc<IntegrityScheduler>,
}

impl AuditKernel {
    pub fn append(&self, event: AuditEventInput) -> Result<AuditReceipt>;
    pub fn verify_chain(&self, tenant: TenantRef, from: EventId, to: EventId) -> Result<ChainVerifyReport>;
    pub fn detect_tampering(&self, tenant: TenantRef) -> Option<TamperingAlert>;
}
```

## 11. ContentSafetyGuard 接口（不可替换实现）

```rust
pub struct ContentSafetyGuard {
    injection_patterns: Vec<CompiledPattern>,
    max_content_length: usize,
    embedding_max_tokens: usize,
    pii_detector: Arc<dyn PiiDetector>,         // PII 检测可插件，结果由 Guard 处置
}

impl ContentSafetyGuard {
    /// 阻断式写入检查；返回 Err 则中止写入
    pub fn check_write(&self, content: &str) -> Result<ContentSafetyReport, ContentSafetyError>;
}
```

详见 [content-safety.md](./content-safety.md)。

## 12. 与 FAP-1 Kernel 的关系

```text
FAP-1 Kernel             协议级 Kernel（Envelope / Session / Replay / Receipt）
FAP-ME Kernel            记忆级 Kernel（Mandate.purpose / SBU / Redaction / 内容安全）

FAP-1 Kernel 不感知记忆语义
FAP-ME Kernel 服从 FAP-1 Kernel Contract，并附加记忆专属约束

两者通过 Plugin 接口共用：
  SignaturePlugin（FAP-1）  ← FAP-ME RedactionReport / ForgetReceipt 复用
  AuthPlugin（FAP-1）       ← FAP-ME 认证复用
  AuditSink（FAP-1 + FAP-ME）  ← 共用接口，可同时 fan-out
```
