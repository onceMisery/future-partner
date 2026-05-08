# FAP-ME 设计文档审查发现

## 关键发现

- 原规格最大风险是 FME 自定义控制面、数据面、Mandate 与 Receipt 语义与 FAP-1 重叠，容易绕过 FAP-1 capability、risk、finality 与审计路径。
- `allowed_ops` 字符串模型混合 capability、layer、action/risk 三个维度，存在 contains/前缀匹配误授权风险，应改为结构化三元组精确匹配。
- HardForget 不能把本地数据库、本地索引、本地 CAS 与 S3/Kafka/Qdrant/远端 sink 描述为同一 ACID 事务。正确模型是本地强一致 + 外部最终一致。
- ForgetReceipt 若只返回“完成”会误导调用方。必须区分本地提交与全局协调，并暴露 pending external systems。
- RedactionReport 的签名只能证明执行方按声明的 policy/classifier 版本做过处理，不是密码学意义的“无 SBU 残留证明”。
- TagGraph、TagEdge、Tag centroid、doc_count、cooccur 等统计必须按 tenant/namespace 隔离，否则会形成多租户语义侧信道。
- 插件可插拔不能扩展到 PolicyKernel、TenantKernel、AuditKernel、MandateVerifier、ContentSafetyGuard；第三方插件应限制在 WASM/sidecar。
- Edge Lite 资源目标需要容量预算和裁剪策略，否则 SQLite、HNSW、TagGraph、ContentSafety、Audit/outbox 同时常驻会超过 512MB。
- 检索评分不能使用单次候选池 min-max 作为生产 final_score 的基础，否则跨请求不可比、审计回放不稳定。

## 修订方向

- 以 `fap1-binding.md` 作为协议绑定总锚点。
- 以 `outbox-reconciliation.md` 作为跨系统一致性总锚点。
- 以 `purpose-vocabulary.md` 的 CapabilityTriple 作为授权唯一模型。
- 以 `redaction-policy.md` 的分层信任模型替代“独立证明完整脱敏”表述。
