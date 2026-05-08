# FAP-ME 设计文档审查计划

## 目标
深入审查并修订 `设计文档-初版/02-记忆系统/docs/fme` 及相关 FAP-1 集成设计，按用户已同意的 P0/P1/P2 风险清单优化 FAP-ME 规范。

## 阶段

| 阶段 | 状态 | 内容 |
|---|---|---|
| 1 | complete | 建立文档清单，识别 FME 与 FAP-1 关键文档 |
| 2 | complete | 审查架构、记忆层、数据模型、插件运行时 |
| 3 | complete | 审查 FAP-1 集成、控制面、数据面、审计与安全 |
| 4 | complete | 审查部署、多租户、存储、路线图、测试一致性 |
| 5 | complete | 汇总严重程度排序的问题清单与改进建议 |
| 6 | in_progress | 按已同意原则修订 P0：FAP-1 绑定、数据面、HardForget、一致性与多租户 TagGraph |
| 7 | pending | 修订 P1：Mandate 扩展、Purpose 三元组、RedactionReport 信任模型、审计绑定、插件边界 |
| 8 | pending | 修订 P2：CoreMemory、容量模型、评分归一化、旧版迁移说明 |
| 9 | pending | 全文一致性检查并汇总交付 |

## 决策记录

- 不使用 subAgent，避免引入并发上下文差异。
- `fme` 目录包含 35 个 Markdown 文档，需覆盖 00-18 主规范与算法/安全/多租户等专题规范。
- 用户已同意进入文档修订，修订顺序为 P0 → P1 → P2。
- HardForget 采用本地强一致 + 外部最终一致模型：本地 SQLite/PG + 本地索引 + 本地 CAS 可单事务提交；S3/Kafka/Qdrant/远端 sink 走 outbox + 幂等 mutation ledger + reconciler。
- ForgetReceipt 必须区分 `LOCAL_COMMITTED` 与 `GLOBALLY_RECONCILED`，后者允许异步达成。
- RedactionReport 只证明“按指定 policy 与 classifier_version 执行并移除了 N 条命中”，不证明绝对无 SBU 残留；接收方信任来自 policy、classifier、executor 的复合信任链。
- FME Mandate 不再定义独立 schema，作为 FAP-1 Mandate 的 capability + constraint 扩展；purpose 走 constraint。
- Purpose 授权从字符串匹配改为三元组精确匹配：`capability` + `layer` + `action`。

## 错误记录

| 错误 | 处理 |
|---|---|
| 无 | 无 |
