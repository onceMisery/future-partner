# FAP-ME 设计文档修订计划

## 目标

按用户已确认的 P0/P1/P2 原则，修订 `设计文档-初版/02-记忆系统/docs/fme` 下 FAP-ME / FME-1 规范，使其与 FAP-1 绑定、HardForget 双状态、RedactionReport 信任边界、Purpose 三元组授权和可插拔架构边界保持一致。

## 阶段

| 阶段 | 状态 | 内容 |
|---|---|---|
| 1 | complete | 完成初始设计审查与风险归类 |
| 2 | complete | 确认修订原则：FAP-1 Mandate 扩展、HardForget outbox、RedactionReport 信任模型、Purpose 三元组 |
| 3 | complete | 新增并吸收 `fap1-binding.md` 与 `outbox-reconciliation.md` 专题规范 |
| 4 | complete | 修订 P0：控制面、数据面、FAP-1 绑定、审计绑定、HardForget、存储一致性、多租户 TagGraph |
| 5 | complete | 修订 P1：安全模型、Mandate constraints、Purpose 三元组、RedactionReport、插件边界、Handoff |
| 6 | complete | 修订 P2：CoreMemory 命名、Edge Lite 容量模型、评分归一化、路线图依赖 |
| 7 | in_progress | 全文一致性检查、残留旧 schema 扫描、交付汇总 |

## 已确认决策

- FME 不定义独立 Mandate schema；复用 FAP-1 Mandate，并通过 signed constraints 表达 `memory.purpose`、`memory.allowed_triples`、`memory.sbu_access` 等。
- Purpose 授权不再使用字符串 `allowed_ops`，改为 `capability + layer + action` 三元组精确匹配，禁止 contains / 前缀 / 后缀匹配。
- HardForget 分为本地强一致与跨系统最终一致：本地 SQLite/PG、本地索引、本地 CAS 单事务提交；S3/Kafka/Qdrant/远端 sink 通过 outbox + 幂等 `mutation_ledger` + reconciler 达成。
- ForgetReceipt 必须区分 `COMMITTED_LOCALLY`、`GLOBALLY_RECONCILED`、`RECONCILE_FAILED`。
- RedactionReport 只能证明按指定 policy 与 classifier_version 执行过脱敏并签名留痕，不证明绝对无 SBU 残留。
- 跨租户/跨组织共享强制 `sbu_manifest`；租户内低风险共享可按策略可选。
- `CoreMemory` 不是新层，迁移为 L2 protected retention tier。
- 第三方插件默认 WASM/sidecar；`rust_native` 仅允许 builtin/trusted。

## 验证记录

| 项 | 状态 |
|---|---|
| 旧 schema 残留扫描 | 已运行 `rg`，关键旧字段无有效残留 |
| Git 工作区检查 | 已运行 `git status --short` |
| 构建/测试 | 文档修订，无代码构建；未运行测试 |
