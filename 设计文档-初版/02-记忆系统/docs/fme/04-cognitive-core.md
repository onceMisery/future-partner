# 04 - 认知内核（Seahorse 脑区分工）

> Cognitive Core 是 FAP-ME 的认知计算层，借鉴 Seahorse 的脑区分工模型。每个组件单一职责、不越权、可独立替换。

## 1. 脑区分工总览

```text
┌──────────────────────────────────────────────────────────────────┐
│                         Cognitive Core                            │
├──────────────────────────────────────────────────────────────────┤
│  Thalamus      意图门控 / Tide / 引力场       ← 不存储状态        │
│  Cortex        HNSW 向量检索                  ← 不处理 Tag 拓扑   │
│  Synapse       LIF 脉冲 / 共现拓扑 / V7       ← 不做向量检索      │
│  Hippocampus   SQLite WAL / mmap / 事务       ← 不做计算          │
│  Cerebellum    后台任务调度                   ← 不阻塞主路径      │
└──────────────────────────────────────────────────────────────────┘
```

---

## 2. Thalamus（丘脑） — 意图门控

### 职责

```text
查询意图分析、Tide 能量分解、引力场修正、弱信号挖掘
```

### 输入输出

```text
输入：query 向量 + Tag 质心库 + 引力场快照
输出：TideResult { layers, focus_level, residual, weak_signal_tags }
```

### 核心算法：Tide

Gram-Schmidt 多级正交剥离 + 投影熵 + 引力场。完整公式与参数见 [tide-algorithm.md](./tide-algorithm.md)。

### 边界

```text
✅ 只做向量计算与意图分析
❌ 不存储任何持久状态
❌ 不直接读取 Tag 图（通过 Synapse 接口）
❌ 不做 HNSW 检索
```

### 触发条件

`tide` 检索模式必经 Thalamus；其他模式按需调用（hybrid 模式可调用 `apply_gravity_field` 修正 query）。

---

## 3. Cortex（大脑皮层） — 向量检索

### 职责

```text
HNSW 向量索引、KNN 检索、双索引切换、向量空间管理
```

### 输入输出

```text
输入：query 向量 + filter（tenant/namespace/layer）
输出：Vec<VectorHit> { memory_id, score, vector }
```

### 实现要求

- 默认实现：usearch（HNSW，mmap 友好）
- 必须支持 `mark_deleted`（软删除）
- 必须支持 `fork_version`（双索引切换，embedding 模型升级用）
- `query_routing_score` 用于双索引共存期的切流评估

### 边界

```text
✅ 只做向量计算
❌ 不解析 Tag
❌ 不解析 Episode 结构
❌ 不读 SQLite，只读 mmap 索引文件
```

详见 [09-plugin-runtime.md](./09-plugin-runtime.md) 中的 `VectorIndex` trait。

---

## 4. Synapse（突触网络） — Tag 拓扑

### 职责

```text
Tag 共现拓扑维护、LIF 脉冲扩散、V7 有向序位图、冷热分离
```

### 输入输出

```text
输入：seed Tags + LifParams（hops, threshold, decay）
输出：SpikeTrace { activated_nodes, energy_field, isPullback_marks }
```

### 核心算法

- **LIF 脉冲扩散**：详见 [lif-spike.md](./lif-spike.md)
- **有向序位拓扑（V7）**：source→target 有向边，势能 Φ ∈ [0.9, 0.5]
- **冷热分离**：Top-N 热边常驻内存，冷边按需从 SQLite 加载

详见 [tag-graph-v7.md](./tag-graph-v7.md)。

### 边界

```text
✅ 只做图扩散与拓扑维护
❌ 不做向量检索
❌ 不写持久存储（写操作通过 Hippocampus）
```

---

## 5. Hippocampus（海马体） — 持久存储

### 职责

```text
SQLite WAL / PostgreSQL 适配、mmap 快照、事务、版本、原子写入
```

### 输入输出

```text
put(MemoryUnit)        → CommitAck { revision, audit_event_id }
get(MemoryId)          → Option<MemoryUnit>
scan(MemoryFilter)     → Vec<MemoryUnit>
tombstone(id, mode)    → ForgetReceipt
```

### 实现要求

- 默认实现：SQLite WAL（边缘部署）
- 服务端实现：PostgreSQL + 行级锁
- 必须保证 `put` 与 `audit_kernel.append` 在同一事务
- mmap 快照用于 hot read（HNSW 索引文件、TagGraph CSR）

### 边界

```text
✅ 只做事务与持久化
❌ 不做向量计算
❌ 不做 LIF 扩散
❌ 不解析 Tag 语义
```

详见 [13-storage-design.md](./13-storage-design.md)。

---

## 6. Cerebellum（小脑） — 后台任务

### 职责

```text
梦境整合、压缩、重索引、健康分析、Tag 图剪枝
```

### 输入输出

```text
schedule(JobSpec) → JobId
poll(JobId)       → JobStatus
cancel(JobId)     → Result<()>
```

### 任务类型

| 任务 | 触发 | 默认间隔 |
|---|---|---|
| DreamRun | 阈值 + cron | 每 6 小时 |
| CompressEpisode | L1 容量超限 | 每天凌晨 |
| RebuildIndex | embedding 升级或 health < 0.7 | 按需 |
| PruneTagGraph | 边数 > 阈值 | 每周 |
| ForgetSweep | tombstone 累积 | 每天 |
| ChainIntegrityVerify | AuditKernel 周期检查 | 每天 |

### 边界

```text
✅ 只做后台调度
❌ 不阻塞主检索路径（P99 < 100ms 无受害）
❌ 不直接写记忆（通过 Plugin → Kernel 流程）
❌ 不验证安全凭证（依赖 Kernel）
```

详见 [09-plugin-runtime.md](./09-plugin-runtime.md) 中的 `DreamWorker` / `ForgetEngine` 插件，与 [dream-state-machine.md](./dream-state-machine.md)。

---

## 7. 与 FAP-1 协议层的关系

```text
FAP-1 控制面 → Kernel 授权 → Memory Orchestrator → Cognitive Core
                                                  ↓
                                          Plugin 算法实现
                                                  ↓
                                          Hippocampus 持久化
                                                  ↓
                                          AuditKernel 写链
```

Cognitive Core 不直接对外提供 API。所有外部调用必须经过 Kernel。

---

## 8. 单一职责约束的工程意义

脑区分工不是架构美学，是插件化的工程前提。

```text
若 Cortex 处理 Tag 拓扑       → 替换向量库时必须重写 Tag 逻辑
若 Synapse 做向量检索         → 替换 LIF 算法时必须连同向量索引一起替换
若 Thalamus 持有状态          → 横向扩展时无法无状态化
若 Hippocampus 做计算         → 主从切换时计算结果不一致
若 Cerebellum 阻塞主路径       → 后台任务影响 P99 延迟
```

这些边界是 [09-plugin-runtime.md](./09-plugin-runtime.md) 中各 trait 设计的依据。

---

## 9. 多语言 SDK 与 WASM

Cognitive Core 全部 Rust 实现，通过以下方式暴露给上层：

| SDK | 实现 | 适用场景 |
|---|---|---|
| `fap-me-core` | Rust crate | 服务端 |
| `fap-me-server` | axum + tonic | 网关 |
| `fap-me-python` | PyO3 | Python Agent |
| `fap-me-node` | napi-rs | Node.js Agent |
| `fap-me-wasm` | wasm-bindgen | 浏览器 / Edge Worker |
| `fap-me-java` | gRPC client | Java Agent |
| `fap-me-go` | gRPC client | Go Agent |

接口统一通过 FAP-1 Protobuf 生成；核心计算通过 Rust 暴露。
