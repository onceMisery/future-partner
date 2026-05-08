# FAP-ME 设计文档审查发现

## 文档范围

待审查：`设计文档-初版/02-记忆系统/docs/fme` 下全部设计文档，并按需交叉阅读 `设计文档-初版/01-协议设计/docs/fap` 中 FAP-1 协议文档。

## 发现

- `fme` 主规范包括 00-18，共 19 章；专题规范覆盖 kernel contract、purpose vocabulary、redaction、content safety、forget、multi-tenant、算法评分/检索/TagGraph/Tide/LIF 等。
- FME 文档明确将 FAP-1 作为传输、会话、身份、控制面、数据面、审计/回执的底座，FME 主要定义记忆语义、层级、检索、插件、安全策略和部署形态。
- 初步定位的交叉核对重点：FAP-1 控制面/数据面优先级和背压、FAP-1 Kernel Contract 与 FME Kernel Contract、FAP-1 审计回执与 FME 审计链、FAP-1 插件生命周期与 FME 插件运行时。
- 三区架构的方向合理：Kernel Zone 承担不可替换的授权、审计、租户隔离、Mandate、内容安全；Plugin Zone 承担算法/适配；Cognitive Core 负责编排和认知计算。
- 关键风险 1：`tenant_id` 同时出现在请求/数据模型字段中，而文档又要求 TenantKernel 强制注入租户过滤，后续需要确认外部请求是否可携带并伪造 tenant_id。
- 关键风险 2：文档声明插件不能直接访问原始 MemoryUnit，但插件扩展点包含 VectorIndex、TagGraph、ObjectStore、AuditSink 等底层适配，若未提供 capability-scoped handle，插件仍可能越权读取或泄漏。
- 关键风险 3：四层记忆模型中 HardForget 依赖 lineage 级联，但 L1 摘要/L2 事实是聚合产物，删除某个 L0 上游后如何重新计算或证明下游已净化尚未说明。
- 关键风险 4：`put` 与 `audit_kernel.append` 要求同一事务，但写入路径同时包含 Embedding、Cortex.add、Synapse.upsert 等非同库副作用，原子性边界需要进一步审查。
- 控制面问题：`MemoryControlService.Store(stream MemoryChunkFrame)` 没有接收 `MemoryStoreRequest`，但文档又定义 `MemoryStoreRequest` 并要求其中包含 tenant、namespace、target_layer、mandate、purpose，导致流式写入的授权元数据承载位置不一致。
- 幂等性问题：文档要求 Store/Forget/Share/Handoff/Dream/Rebuild 的 `idempotency_key` 必填，但示例 proto 中未定义该字段，也未说明 FAP-1 Envelope 是否统一承载。
- 数据面问题：`ContextFrameStream` 文档称 L0 接收后立刻 append，但安全约束称每个 chunk/frame 解码后立即检测；两者顺序冲突。对 L0 原始上下文而言，先 append 后检测会让 prompt injection/PII/SBU 已经进入 WAL。
- 数据面安全问题：`ObjectRef` 可含 `s3://` 或 `file://` URI，读取方通过 ObjectStore 拉取；若不规定 URI scheme 白名单、租户前缀校验和 lease 绑定，容易形成 SSRF/任意文件引用/跨租户对象引用风险。
- 审计问题：FME 独立维护按 tenant 的 hash chain，但 `AuditKernel` API 示例只有单个 `Arc<Mutex<ChainState>>`；多租户并发下要么成为全局瓶颈，要么实现与规范不一致。
- 审计 fan-out 问题：AuditSink 可同时配置多个 Sink，但未定义部分 sink 失败时是 fail-closed、quorum、降级还是异步补偿，会影响“所有记忆操作必须写审计”的合规语义。
- 插件运行时问题：文档说第三方插件强制 WASM、V1 禁止任意 `.so/.dll` 动态注入，但 manifest 允许 `runtime = "rust_native"`，Registry 示例也允许任意 `Plugin + 'static` 注册，缺少 trusted/builtin 与 third-party 的强制边界。
- Handoff 问题：Grant 撤销只保证后续 RedeemGrant 拒绝，无法撤销已经被接收方读取/导入的 foreign episode；文档需要显式区分访问撤销与已复制数据的合规删除协作。
- ContextGrant 字段语义不一致：数据模型中的 allowed_ops 是 `read | append | merge`，上下文共享章节的导入策略又使用 `read_l0_snapshot | import_l1_temp | append_project_l1 | merge_l2 | view_sbu`，需要统一成可验证的枚举。
- 存储设计问题：SQLite 表没有给 `memory_lineage(parent_id)`、`memory_tag(tag_id)`、`semantic_fact(tenant_id, namespace, subject, predicate)` 等关键查询建立索引，HardForget 级联、Tag 检索和冲突仲裁会退化。
- 存储一致性问题：向量索引要求 `add(memory_id, vector, meta)` 事务一致，但 Qdrant/Milvus/S3/Kafka 等外部系统无法与 PG/SQLite 做本地 ACID 事务，缺少 outbox/saga/reconciliation 设计。
- 部署形态问题：Edge Lite 资源目标为 512MB 内存，但本地 embedding、HNSW mmap、Tag hot graph、SQLite WAL、审计链和内容安全同时存在，缺少容量模型和裁剪策略。
- FAP-1 集成问题：FME 声明复用 FAP-1 Receipt 链，同时又独立维护 FME AuditEvent 链；两条链之间没有强绑定字段（例如 fap_receipt_id / invoke_id / envelope_hash），会造成跨层追责断点。
- 路线图风险：Phase 1 在 2 周内要求 Kernel、WASM、Protobuf、Agent Card、Purpose、Redaction、Dream、Handoff、Gateway、TS/Python SDK 骨架，范围明显过大。
- 路线图依赖风险：Phase 4 上下文共享更依赖 Redaction/Grant/Mandate/SBU，而路线图说依赖 Phase 3 高级检索；Phase 5 又交付 Intent Mandate verifier，但 Phase 1/2/4 已经把 Mandate 作为前置安全基础。
- Purpose 词汇表问题：`allowed_ops` 同时出现 `retrieve`、`retrieve:L1`、`memory.forget:hard`、`admin.rebuild` 等不同命名粒度，而能力注册使用 `memory.admin.rebuild`；PolicyKernel 示例又用 exact contains 校验，容易产生实现间授权差异。
- Purpose 演进风险：文档规定“不允许收紧 allowed_ops / sbu_access”，但安全策略遇到误授权或监管变化时必须能紧急收紧；需要区分 registry 兼容演进与 mandate 强撤销/安全补丁。
- RedactionReport 风险：接收方验证 `sbu_units_removed >= expected_sbu_count`，但 `expected_sbu_count` 来自原始 scope 或发送侧声明，接收方无法在不看原文的情况下独立证明 SBU 未漏检。
- Redaction 隐私风险：tombstone 会暴露“此处曾有敏感内容”的存在；`lineage_parent_ids` 使用 tenant_id 作为稳定 salt 会产生跨报告可关联性。
- ContentSafety 风险：写入路径用 prompt-injection 模式阻断，容易误杀安全文档、测试样例、用户明确要求保存的攻击样本；需要“隔离保存/低信任标签”而不只是拒绝。
- ContentSafety 范围风险：查询路径只做 Sanitizer 不阻断；若召回内容会被直接拼进模型上下文，恶意 query 仍可能诱导检索高风险历史片段，应至少给查询侧风险评分和结果二次过滤。
- ForgetEngine 实现风险：文档声称 HardForget 所有步骤在同一 SQL 事务内完成，但向量索引、对象存储、索引 compaction、外部审计 sink 都不是同一事务资源。
- Forget 语义冲突：前文说 HardForget 级联删除下游，级联清单又说对子级 episode/fact 只是移除 lineage 引用；若聚合摘要/事实仍包含被遗忘内容，合规遗忘并未完成。
- DreamWorker 风险：LOW 风险默认自动审批并可执行压缩/剪枝/事实插入，若 DreamWorker 或风险评估被污染，会形成后台记忆投毒；需要 dry-run diff、人类可见变更、回滚窗口与抽样审查。
- 多租户严重问题：`tag`、`tag_edge`、`memory_tag` 表及 Tag 数据模型未包含 `tenant_id`/`namespace`，但 TagCentroid、doc_count、cooccur、edge 权重会编码租户语义；服务端多租户下存在跨租户统计泄漏和召回污染。
- retain_score 问题：公式依赖 salience/stability/user_preference/task_reuse 等 LLM 或启发式评分，但没有标定数据、置信区间、人工反馈闭环；自定义 CompactStrategy 可改变长期记忆晋升质量。
- retain_score 命名问题：分层决策返回 `CoreMemory`，但四层模型只有 L0-L3，`CoreMemory` 未定义为层或标签。
- 检索打分问题：候选池内 min-max 归一化使 final_score 不具备跨请求可比性，受离群值影响大，审计回放也难稳定复现。
- 检索安全问题：`sbu_safe` 模式把 `w_sbu` 设为 0，前提是候选已经脱敏；若流水线顺序或插件返回绕过脱敏，SBU 不会再被硬过滤。
- 检索降级问题：文档同时要求“任何模式超时不返回空结果”和 “sbu_safe 超时直接失败”，语义冲突；部分结果“不影响 final_score 排序正确性”的断言也不成立。
- Tide 数学问题：投影能量比未归一却用于熵概率，且 Tag 质心彼此不保证正交，`|residual|² + Σ|projection_i|² = |query|²` 的不变量不成立。
- Tide 稳定性问题：引力场用 inverse-square 距离，近距离只用 EPS 跳过而没有 force clamp，可能把 query 向量过度拉向高质量 Tag 群。
- LIF 实现问题：公式定义了 leak，但示例实现没有应用 `leak_factor`；`eligible(dst)` 在流程中出现但代码中没有安全/租户过滤参数，容易让算法实现遗漏权限过滤。
- TagGraph 质量问题：V7 边权 `weight += Φ_src × Φ_dst` 主要随共现次数累积，高频通用 Tag 会支配拓扑；需要归一化、时间衰减和高频惩罚。
- 多租户运维问题：每租户插件实例和每租户向量 collection/schema 会带来实例爆炸，缺少上限、lazy loading、LRU 回收和启动时间预算。
- 错误/警告映射问题：`Warning` HTTP header 不适合 gRPC/streaming 场景，检索 fallback/partial-result 应进入 frame metadata/trailer 或标准 problem extension。
- FAP-1 控制面集成偏差：FAP-1 的主调用入口是 `InvokeRequest(capability_id, input, input_objects, mandate_id, finality_policy)`，FME 又定义独立 `MemoryControlService` RPC；若直连该 service，可能绕过 FAP-1 capability/risk/receipt 路径。
- FAP-1 数据面集成偏差：FAP-1 明确禁止把 chunk bytes 塞入 Envelope，使用 `DataOpen/DataCommit/DataAck/DataChunkMeta` + raw stream + Merkle root；FME 自定义 `DataFrame.payload bytes` 和 `MemoryChunkFrame.chunk_bytes`，与 FAP-1 数据面契约冲突。
- FAP-1 Mandate 集成偏差：FAP-1 Mandate 以 capabilities/resources/constraints/delegation/budget 为核心，FME Intent Mandate 以 allowed_ops/scope/purpose 为核心；需要定义 FME mandate 是 FAP mandate 的 extension 还是独立凭证，否则撤销、委托链和预算语义不一致。
- FAP-1 Receipt 终局性问题：FAP-1 对 high/critical 支持 PROVISIONAL/FINALIZED 多签路径；FME 对 `memory.forget`、`memory.dream.approve`、`memory.admin.rebuild` 等高风险操作未明确要求 FINALIZED 后再执行或使用补偿机制。
- FAP-1 ObjectRef 集成偏差：FAP-1 ObjectRef 包含 `origin_node_id`、availability、replica、lease 生命周期；FME ObjectRef 简化为 object_id/media/size/hash/uri/lease/expires，跨节点解析与 lease 续租语义会丢失。
- 上层文档一致性问题：`2-记忆系统设计最终版.md` 仍把 `PolicyEngine`、`MemoryStore` 等列为可插拔 trait，而 `fme/kernel-contract.md` 已明确 PolicyKernel 不可插件化；需要标注上层文档为历史版本或同步修订。
- 上层路线图与 fme 路线图不一致：最终版 Phase 1 更偏协议骨架，`fme/18-roadmap.md` Phase 1 扩大到完整 Kernel、WASM、Redaction/Dream/Handoff/SDK 骨架，执行范围发生膨胀。
