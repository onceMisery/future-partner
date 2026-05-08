# FAP-ME 设计文档修订进度

## 本轮记录

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

## 待办

- 最终交付前做一次综合 `rg` 验证并汇总修改范围。
