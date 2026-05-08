# tag-graph-v7.md

> TagGraph V7 = 有向序位拓扑（势能 Φ）+ 内生残差（截断 SVD）+ 虫洞路由 + 冷热分离。是 Synapse 组件的存储基底。

## 1. 演进路径

```text
V4   单一关键词 → 共现矩阵                        基础
V6   静态共现 → LIF 脉冲扩散                       动态激活
V7   无向 → 有向序位 + 内生残差 + 虫洞路由        本文档
V8   测地线重排（依赖 V7 LIF 距离场）             零成本二次排序（geodesic-rerank.md）
```

## 2. 有向序位拓扑

### 2.1 核心思想

人类联想具有方向性：

```text
"壁炉" → 容易想到 "猫"        Φ_壁炉 高，Φ_猫 中
"猫"   → 不一定想到 "壁炉"    反向激活弱
```

V7 用势能 Φ 表达这种方向性。

### 2.2 势能定义

```text
Φ ∈ [0.5, 0.9]

日记中越靠前的 Tag → Φ 越高（作者心理优先级）
计算：
  Φ_i = 0.9 - (position_i / total_tags) * 0.4
  其中 position_i 是 Tag 在标签序列中的位置（0-indexed）
```

### 2.3 边权重

```text
W(src → dst) = Φ_src × Φ_dst

不再是次数叠加，而是势能积；
反映作者打标签时的逻辑顺序（而非简单共现频次）。
```

### 2.4 累积更新

```text
当新日记加入：
  for each (src_pos, dst_pos) in tag_pairs:
    Φ_src = 0.9 - src_pos / total * 0.4
    Φ_dst = 0.9 - dst_pos / total * 0.4
    edge[src→dst].weight += Φ_src × Φ_dst
    edge[src→dst].cooccur_count += 1
```

## 3. 内生残差（Intrinsic Residual）

### 3.1 概念

衡量一个 Tag 在其局部联想网络中的"信息密度"或"不可替代性"。

```text
低残差   该 Tag 的语义可被邻居完全解释（平庸、从属概念）
高残差   该 Tag 带有邻居不具备的独特语义（核心、独特信息源）
```

### 3.2 计算

```text
对每个 Tag t：
  邻居集 N(t) = {其有向邻居的质心向量}
  使用截断 SVD 分解 N(t) 矩阵：
    N(t) = U Σ V^T
  截断到 top-k 奇异值（保留主子空间 V_k）
  
  t 的向量在主子空间上的残差：
    proj   = V_k V_k^T · t.vector
    residual = t.vector - proj
    intrinsic_residual = |residual| / |t.vector|
```

`intrinsic_residual ∈ [0, 1]`。

### 3.3 在 LIF 中的作用

```text
扩散到节点 t 时：
  effective_decay = base_decay × (1 + node_residual_gain × intrinsic_residual(t))
  → 高残差节点的电位传导更强（信息枢纽特权）
```

## 4. 虫洞路由

### 4.1 问题：稠密陷阱

LIF 在同质化、高聚集的 Tag 区内可能空耗算力打转，无法跨域跳到长尾但致命相关的远端节点。

### 4.2 三机制

#### 动量机制（Momentum/TTL）

```text
为每个初始脉冲赋予动量 m₀（默认 1.0）
每跳消耗动量：m -= base_decay
m ≤ 0 时停止扩散

普通稠密区扩散迅速衰减（精准锁定核心意图）
```

#### 逻辑张力探测

```text
脉冲撞击目标节点 dst 时：
  Tension(src → dst) = W(src→dst) × intrinsic_residual(dst)

若 Tension > WORMHOLE_THRESHOLD（默认 0.6）：
  触发虫洞
```

#### 引力弹弓效应

```text
进入虫洞的脉冲获得特权：
  decay = WORMHOLE_DECAY（0.95，远高于 base_decay 0.7）
  动量不消耗

效果：稠密区聚集的庞大势能瞬间喷射向稀疏但致命相关的远端节点。
```

### 4.3 实现

```rust
pub struct OrdinalLifParams {
    pub base: LifParams,
    pub wormhole_threshold: f32,    // 默认 0.6
    pub wormhole_decay: f32,        // 默认 0.95
    pub momentum_initial: f32,      // 默认 1.0
}

fn ordinal_expand(
    seeds: &[(TagId, f32)],
    graph: &dyn TagGraph,
    params: &OrdinalLifParams,
) -> SpikeTrace {
    // 类似 lif_expand，但每个脉冲带 momentum 字段
    // 撞击目标时检查 Tension：
    //   if Tension > wormhole_threshold:
    //     decay = wormhole_decay
    //     不消耗 momentum
    //   else:
    //     decay = base.decay_factor
    //     momentum -= base.decay_factor
    //     若 momentum <= 0：停止
    ...
}
```

## 5. 冷热分离

### 5.1 内存压力

大规模场景下，全量 Tag 边可能数百万。全量加载内存压力大、查询慢。

### 5.2 分离策略

```text
热图（hot）：
  Top-N 高权重边常驻内存（默认 Top-50000）
  支持快速 O(1) 邻居查找
  数据结构：HashMap<TagId, HashMap<TagId, f32>>

冷图（cold）：
  低频边持久化在 SQLite tag_edge 表
  访问时按需加载
  后台周期压缩：低权重边合并或删除

mmap 快照：
  CSR 格式（Compressed Sparse Row）
  避免热路径加锁
  适合 LIF 扩散批量遍历
```

### 5.3 加载流程

```text
服务启动：
  1. SELECT * FROM tag_edge ORDER BY weight DESC LIMIT 50000
  2. 加载到 hot HashMap
  3. 生成 CSR mmap 快照

后台维护：
  1. 每天凌晨：
     - 统计每条边的访问频次
     - hot 中长期未访问 → 移到 cold
     - cold 中频繁访问 → 提升到 hot
     - 重新生成 mmap 快照（双 buffer 切换）
  2. 每周：
     - 删除 weight < min_weight 且 last_seen > 365d 的边
```

## 6. 实现规范

```rust
pub struct ColdHotTagGraph {
    hot: Arc<RwLock<HashMap<TagId, HashMap<TagId, EdgeWeight>>>>,
    cold_store: Arc<dyn TagEdgeStore>,
    ordinal_adjacency: Arc<RwLock<DirectedOrdinalGraph>>,
    csr_snapshot: Arc<RwLock<CsrSnapshot>>,
    config: ColdHotConfig,
}

pub struct ColdHotConfig {
    pub hot_top_n: usize,           // 默认 50000
    pub min_weight: f32,            // 默认 0.05
    pub max_age_days: u32,          // 默认 365
    pub recompute_interval: Duration,
}

pub struct EdgeWeight {
    pub weight: f32,                // Φ_src × Φ_dst 累积
    pub source_potential: f32,      // 最近一次的 Φ_src
    pub target_potential: f32,
    pub cooccur_count: u32,
    pub last_seen: SystemTime,
}

impl TagGraph for ColdHotTagGraph {
    fn upsert(&self, memory_id: MemoryId, tags: &[Tag]) -> Result<()> { ... }

    fn expand(&self, seeds: &[Tag], params: &LifParams) -> Result<SpikeTrace> {
        // 调用 lif_expand，使用 csr_snapshot
        ...
    }

    fn ordinal_expand(
        &self,
        seeds: &[TagWithPotential],
        params: &OrdinalLifParams,
    ) -> Result<SpikeTrace> {
        // 调用 ordinal_expand
        ...
    }

    fn load_hot_edges(&self, top_n: usize) -> Result<HotGraph> { ... }
    fn flush_cold_edges(&self, hot_graph: &HotGraph) -> Result<()> { ... }
    fn prune(&self, min_weight: f32, max_age_days: u32) -> Result<PruneReport> { ... }
}
```

## 7. 默认参数

| 参数 | 默认值 | 说明 |
|---|---:|---|
| `hot_top_n` | 50000 | 热图保留边数 |
| `min_weight` | 0.05 | 边权清理下限 |
| `max_age_days` | 365 | 边最大保留天数 |
| `WORMHOLE_THRESHOLD` | 0.6 | V7 虫洞触发阈值 |
| `WORMHOLE_DECAY` | 0.95 | V7 虫洞内特权衰减 |
| `MOMENTUM_INITIAL` | 1.0 | 初始动量 |
| `intrinsic_residual_top_k` | 8 | SVD 截断维度 |
| `node_residual_gain` | 0.5 | 残差增益因子 |

## 8. 增量重建调度

```text
触发条件：
  file_tags 写入量 ≥ 总标签数的 1%（V7.1）
  或：手动 admin.rebuild
  或：tag_edge 表数量 < hot_top_n × 0.5（启动时）

防抖：
  小规模变动启动 120s 防抖计时（V7.1）

期间：
  CSR 快照不变，使用旧快照
  增量更新写入 tag_edge 表
  防抖后批量重建
```

## 9. 与最终版的差距

```text
最终版                       本文档
-----                        -----
TagGraph trait：仅 upsert    + ordinal_expand
                / expand     + load_hot_edges / flush_cold_edges
                / prune

无向共现图                   V7 有向序位 + 势能
无冷热分离                   Top-N 热图 + 冷边压缩
无虫洞路由                   动量 / 张力 / 引力弹弓
无内生残差                   截断 SVD per-tag
```

详见 `../../3-记忆系统深度分析与未来方案.md` §3.2 与 §4.5。
