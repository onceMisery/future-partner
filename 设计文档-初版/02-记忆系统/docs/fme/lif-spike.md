# lif-spike.md

> LIF 脉冲扩散是 Synapse 组件的核心算法。借鉴 Leaky Integrate-and-Fire 神经元模型，使 Tag 共现拓扑变成动态激活网络。

## 1. 算法目标

```text
将 query 命中的 seed Tags 沿共现拓扑图扩散，
让"虽未字面提及但与活跃语义紧密关联"的 Tag 也能被激活。
```

## 2. LIF 神经元模型

### 2.1 基础公式

```text
V_i(t+1) = leak * V_i(t) + Σ W_ji * spike_j(t) + I_i(t)

其中：
  V_i(t)        节点 i 在 t 时刻的电位
  leak          泄漏因子（默认 0.7）
  W_ji          j → i 的边权
  spike_j(t)    j 在 t 时刻是否放电（V_j ≥ FIRING_THRESHOLD）
  I_i(t)        外部注入电流（仅 t=0 时为 seed）
```

### 2.2 放电条件

```text
若 V_i(t) ≥ FIRING_THRESHOLD：
  spike_i(t) = 1
  V_i(t+1) = 0   （重置）
否则：
  spike_i(t) = 0
```

### 2.3 累积能量场

```text
accumulated_energy(i) = Σ_t V_i(t)
```

该字段在 Spike 完成后被缓存，作为 [geodesic-rerank.md](./geodesic-rerank.md) 的语义距离场。

## 3. 算法流程

```text
1. 初始化：
   active_nodes = seed_tags
   for tag in seed_tags:
     V[tag] = seed_energy(tag)
     accumulated_energy[tag] = V[tag]

2. for hop in 1..MAX_HOPS:
     new_active = {}
     for src in active_nodes：
       if V[src] >= FIRING_THRESHOLD:
         for (dst, weight) in tag_graph.neighbors(src):
           if not eligible(dst): continue
           injected = V[src] * weight * DECAY_FACTOR
           if injected > MIN_CURRENT:
             V[dst] += injected
             accumulated_energy[dst] += injected
             new_active.add(dst)
         V[src] = 0   # 放电后重置
     active_nodes ∪= new_active
     若 |active_nodes| > MAX_EMERGENT_NODES：截断（按能量降序保留）

3. 输出：
   SpikeTrace {
     activated_nodes: 所有曾被激活的 Tag
     energy_field: accumulated_energy（用于 geodesic）
     spike_path: 激活路径（可解释性输出）
     isPullback_marks: 仅通过扩散激活而非 seed 命中的节点
   }
```

## 4. V7 有向序位扩散

V7 引入有向边和势能：

```text
W(src→dst) = Φ_src × Φ_dst
  Φ ∈ [0.5, 0.9]：源在日记中的位置势能（前面 = 高）

扩散方向严格按 src → dst：
  V[dst] += V[src] × W(src→dst) × DECAY_FACTOR

逻辑张力探测（虫洞触发）：
  Tension(src→dst) = W × neighbor_residual(dst)
  if Tension > WORMHOLE_THRESHOLD:
    使用 wormhole_decay 而非 base_decay（特权传播）

详见 tag-graph-v7.md。
```

## 5. 实现规范

```rust
pub struct LifParams {
    pub max_hops: u32,             // 默认 2
    pub firing_threshold: f32,     // 默认 0.10
    pub decay_factor: f32,         // 默认 0.7
    pub leak_factor: f32,          // 默认 0.7
    pub min_current: f32,          // 默认 0.01
    pub max_emergent_nodes: usize, // 默认 50
}

pub struct OrdinalLifParams {
    pub base: LifParams,
    pub wormhole_threshold: f32,   // 默认 0.6
    pub wormhole_decay: f32,       // 默认 0.95（虫洞特权）
    pub momentum_initial: f32,     // 默认 1.0
}

pub fn lif_expand(
    seeds: &[(TagId, f32)],
    graph: &dyn TagGraph,
    params: &LifParams,
) -> SpikeTrace {
    let mut potentials: HashMap<TagId, f32> = seeds.iter().cloned().collect();
    let mut accumulated: HashMap<TagId, f32> = potentials.clone();
    let mut active: HashSet<TagId> = potentials.keys().cloned().collect();

    for _ in 0..params.max_hops {
        let mut new_active = HashSet::new();
        let firing: Vec<_> = active.iter()
            .filter(|t| potentials[*t] >= params.firing_threshold)
            .cloned()
            .collect();

        for src in &firing {
            let v_src = potentials[src];
            for (dst, weight) in graph.neighbors(src) {
                let injected = v_src * weight * params.decay_factor;
                if injected < params.min_current { continue; }
                *potentials.entry(dst.clone()).or_insert(0.0) += injected;
                *accumulated.entry(dst.clone()).or_insert(0.0) += injected;
                new_active.insert(dst);
            }
            potentials.insert(src.clone(), 0.0);  // 放电重置
        }

        active.extend(new_active);
        if active.len() > params.max_emergent_nodes {
            // 按 accumulated_energy 降序截断
            let mut sorted: Vec<_> = active.iter().collect();
            sorted.sort_by(|a, b|
                accumulated[*b].partial_cmp(&accumulated[*a]).unwrap()
            );
            active = sorted.into_iter()
                .take(params.max_emergent_nodes)
                .cloned()
                .collect();
        }
    }

    SpikeTrace {
        activated_nodes: active.into_iter().collect(),
        energy_field: accumulated,
        spike_path: vec![],   // 详细路径可选记录
        is_pullback_marks: HashSet::new(),
    }
}
```

## 6. 默认参数

| 参数 | 默认值 | 建议范围 | 说明 |
|---|---:|---:|---|
| `MAX_HOPS` | 2 | 1~4 | 扩散最大跳数 |
| `FIRING_THRESHOLD` | 0.10 | 0.05~0.30 | 放电阈值 |
| `DECAY_FACTOR` | 0.7 | 0.5~0.9 | 边传导衰减 |
| `LEAK_FACTOR` | 0.7 | 0.5~0.9 | 节点电位泄漏 |
| `MIN_CURRENT` | 0.01 | 0.001~0.05 | 微电流截断（性能） |
| `MAX_EMERGENT_NODES` | 50 | 20~200 | 涌现节点数硬上限 |
| `WORMHOLE_THRESHOLD` (V7) | 0.6 | 0.4~0.8 | 虫洞触发阈值 |
| `WORMHOLE_DECAY` (V7) | 0.95 | 0.85~0.99 | 虫洞内特权衰减 |

## 7. 性能与降级

```text
计算复杂度 O(MAX_HOPS × top_k_neighbors × |activated_nodes|)
单次预算 ≤ 80ms

降级触发：
  hop 内时间预算 > 30ms       → 提前结束（仅返回当前已激活）
  total > 80ms               → 退化到仅向量检索
  active.len() 增长率 > 阈值 → 提前终止（拓扑爆炸保护）
```

详见 [retrieval-fallback.md](./retrieval-fallback.md)。

## 8. 与其他算法的协作

```text
LIF 输入：
  seed Tags   ← Tag 提取 / Tide.layers / 用户传入

LIF 输出供：
  tagmemo 模式：activated_nodes 作为补充候选
  tagmemo_geodesic：accumulated_energy 作为测地线距离场（geodesic-rerank.md）
  Tide：weak_signal_tags 可作为补充 seed
  Dream：宽松参数（更大 hop / 更低阈值）做后台联想
```

## 9. 可解释性

```text
SpikeTrace 必须暴露：
  activated_nodes
  energy_field
  spike_path（可选，按 hop 顺序记录激活源）
  is_pullback_marks（仅通过扩散激活的节点，对解释"涌现"很关键）
```

## 10. V6 → V7 → V8 演进

```text
V6   无向共现图 + LIF                       基础涌现
V7   有向序位势能 + 内生残差 + 虫洞路由      跨域非线性涌现
V8   复用 V7 的 accumulated_energy 做测地线  零成本二次重排
```

详见 `../../方案探索/VCPToolBox 研究与多智能体编排.md` 与 [geodesic-rerank.md](./geodesic-rerank.md)。
