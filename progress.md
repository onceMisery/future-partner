# FAP-ME 设计文档修订进度

## 第一轮：基础修订

- 使用 `planning-with-files` 恢复并继续计划；因用户已在当前工作区修改文档，未新建 worktree，避免丢失未提交上下文。
- 以当前磁盘内容为准，保留用户已新增/修订的 `fap1-binding.md` 与 `outbox-reconciliation.md`，在其基础上做全文一致性修订。
- 已修订 Purpose 授权模型：去除有效 schema 中的 `allowed_ops` 字符串授权，改为 `CapabilityTriple` 精确匹配。
- 已修订安全模型：FME Mandate 收敛为 FAP-1 Mandate constraints，TenantKernel 不信任请求体 `tenant_id`。
- 已修订审计/凭证引用：AuditEvent 增加 FAP-1 Receipt 绑定字段，ForgetReceipt 使用本地/全局双状态，RedactionReport 增加 classifier 与 sbu_manifest 语义。
- 已修订上下文共享：ContextGrant 使用 `allowed_triples` + `access_mode`，明确撤销不等于已复制数据自动删除。
- 已修订存储设计：Tag/TagEdge/MemoryTag 按 tenant/namespace 隔离，Grant/Audit/Forget/Mutation ledger schema 对齐新模型。
- 已修订插件边界：第三方插件默认 WASM/sidecar，`rust_native` 限 builtin/trusted；host API 需 capability-scoped handle。
- 已修订部署与算法：补 Edge Lite 512MB 容量预算；`CoreMemory` 改为 L2 protected tier；检索评分改为版本化校准归一化。
- 已修订路线图：Phase 1 拆成 FAP-1 绑定、Kernel 安全、审计绑定、Data Plane/outbox 四道门禁。
- 已运行残留扫描，关键旧风险项未再作为有效 schema 出现。

## 第二轮：审查反馈修订（基于深度分析报告）

按 P0 → P1 → P2 顺序完成下列修订。

### P0 — schema 冻结对账

- 新增 `signing-canonical.md`：AuditEvent / ForgetReceipt / RedactionReport / ContextGrant / HandoffPacket / DreamProposal 的 canonical 字段顺序、hash 输入、签名规则、capability_id ↔ triple 映射、enum 整数值。
- ForgetReceipt schema 五份对齐（05-data-model / 11-audit-and-receipt / outbox-reconciliation / forget-engine / 13-storage-design）；引入双签名 `local_signature` + `reconciled_signature`、`cancelled_after_local_commit` 字段。
- AuditEvent hash 公式三份对齐（一律引用 signing-canonical §2.2）；`fap_finality` 词汇统一为 FAP-1 标准三档（`OPTIMISTIC | VERIFIED | FINALIZED`），删除 `PROVISIONAL`。
- mutation_ledger schema 对齐：合并 outbox §4 与 13-storage §3 字段；统一 `mutation_kind / target_system / attempts / payload_json + idempotency_key + UNIQUE`，新增 `tenant_id / namespace / shard_key`。
- memory_unit DDL 补 `namespace / visibility / retention_tier / embedding_signature` 列与索引；明确 tags / entities / lineage 走关联表。
- MemoryAction 枚举权威 = purpose-vocabulary §3 的 12 值版本；删除 fap1-binding §5 / 06-control-plane §5 中的 6 值 ActionKind。
- capability_id 拆分：禁止 `memory.handoff` / `memory.forget` 单串；统一为 `.create / .receive` 与 `.soft / .hard`。

### P1 — 设计漏洞 / 治理空白

- 新增 `chain-state-concurrency.md`：CAS + base_revision 算法、SchemaPerTenant / DbPerTenant 下的 chain_state_registry 注册中心、ChainIntegrityScheduler。
- outbox-reconciliation §11 加 leader election（PG advisory lock / Etcd / Redis）+ shard_key 分片 + 5 分钟 IN_FLIGHT 超时回收。
- redaction-policy §14.1 加"SBU 过滤责任分布图"，明确 sensitivity_penalty / PolicyKernel.redact / sbu_safe 三层职责与不变式。
- mutation_ledger 新增 `ADVISORY_DELIVERED` 状态：跨 Agent / 远端接收方等不可强制目标，通知送达即结案，不进 NEEDS_INTERVENTION 告警。
- EmbeddingProvider trait 加 `embedding_signature() = (model_id, dim, version)` 三元组；VectorIndex 按 signature 隔离索引。
- RedactionReport schema 一致：11-audit-and-receipt §7.4 改为引用 signing-canonical §4 + redaction-policy §8。

### P2 — 表述 / 可行性

- Edge Lite 默认模式集 = `basic + hybrid + sbu_safe`；CompactStrategy 明确 rule-only fold 路径。
- 02-architecture §4 加"脑区不是部署单元，也不是插件接口"澄清；脑区 ↔ Plugin 的 9 trait 不是 1:1 关系。
- Phase 1 时间从 3 周改为 6-8 周；4 道 Gate 各自给出工程量；W23 GA 改为 W28 GA。
- Phase 1 验收标准 #1 从"无法编译"改为"Conformance + lint 双层保证"。
- ForgetReceipt 加取消语义（`cancelled_after_local_commit`）。

### 新增错误码

- `fap-me/chain-cas-exhausted`
- `fap-me/chain-registry-missing`

### 新增文档

- `signing-canonical.md`
- `chain-state-concurrency.md`

README 与阅读顺序已更新。

## 待办

- 算法专题（tide-algorithm / lif-spike / geodesic-rerank / tag-graph-v7 / retain-score）未在本轮审查范围；建议另开一轮专门审查公式可重现性与参数边界。
- Phase 1 Gate 启动前需要发布 signing-canonical 测试向量集（≥ 18 条样本），并跑通跨实现 byte-for-byte 一致性。
