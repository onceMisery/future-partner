# tide-algorithm.md

> Tide 是 Thalamus 组件的核心算法。基于 Gram-Schmidt 多级正交剥离 + 投影熵 + 引力场。

## 1. 算法目标

```text
1. 将 query 向量按 Tag 质心向量做多级正交分解
2. 通过投影熵衡量 query 的意图聚焦程度
3. 通过引力场修正 query 向量
4. 在残差空间中挖掘弱信号 Tag
```

## 2. 数学定义

### 2.1 Gram-Schmidt 多级剥离

```text
输入：query 向量 q，Tag 质心库 {t_1, t_2, ..., t_n}
初始：residual r_0 = q，initial_energy = |r_0|

for i in 1..max_layers:
    选最相关 Tag：
      best = argmax_t cosine(r_{i-1}, t)
    投影：
      proj_i = (r_{i-1} · t_best / |t_best|²) * t_best
    残差：
      r_i = r_{i-1} - proj_i
    layers.append({tag_id: best.id, energy_ratio: |proj_i| / initial_energy})
    若 |r_i| / initial_energy < residual_threshold：break
```

### 2.2 投影熵

```text
probs = [layer.energy_ratio for layer in layers]
entropy = -Σ p_i * ln(p_i)    （p_i > 0）
focus_level = 1 - entropy / ln(|layers|)
```

- `focus_level` ≈ 1：意图高度聚焦（少数主题占主要能量）
- `focus_level` ≈ 0：意图发散（多主题平分能量）

### 2.3 弱信号挖掘

```text
最终残差 r_final 的方向中可能含有未被主主题覆盖的微弱语义。
weak_signal_tags = top-k Tags by cosine(t, r_final)
仅当 |r_final| / initial_energy > 0.05 时才挖掘
```

### 2.4 引力场修正

```text
对每个 GravitySource g（高质量 Tag 群）：
  mass(g) = √doc_count × avg_cooccur × recency
  dist = euclidean(query, g.center)
  force = G × mass(g) / dist^2          （G = gravity_strength）
  direction = normalize(g.center - query)
  query += direction × force

最后归一化：query = normalize(query)
```

## 3. 实现规范

```rust
pub struct ThalamusGate {
    max_layers: usize,           // 默认 5
    residual_threshold: f32,     // 默认 0.05
    gravity_strength: f32,       // 默认 0.2
    gravity_decay_exponent: f32, // 默认 2.0
    weak_signal_threshold: f32,  // 默认 0.05（启动弱信号挖掘的最低残差能量）
    weak_signal_top_k: usize,    // 默认 5
}

impl ThalamusGate {
    pub fn decompose(
        &self,
        query: &[f32],
        tag_centroids: &[TagCentroid],
    ) -> TideResult {
        let mut residual = query.to_vec();
        let initial_energy = l2_norm(&residual);
        let mut layers = Vec::new();

        for _ in 0..self.max_layers {
            let best = tag_centroids.iter()
                .max_by(|a, b| {
                    cosine(&residual, &a.vector)
                        .partial_cmp(&cosine(&residual, &b.vector))
                        .unwrap()
                });
            let Some(tag) = best else { break };

            let projection = project(&residual, &tag.vector);
            let energy_ratio = l2_norm(&projection) / initial_energy;
            layers.push(TideLayer {
                tag_id: tag.id.clone(),
                energy_ratio,
            });
            residual = subtract(&residual, &projection);

            if l2_norm(&residual) / initial_energy < self.residual_threshold {
                break;
            }
        }

        let probs: Vec<f32> = layers.iter().map(|l| l.energy_ratio).collect();
        let entropy: f32 = -probs.iter()
            .filter(|&&p| p > 0.0)
            .map(|p| p * p.ln())
            .sum();
        let focus_level = if layers.len() > 1 {
            1.0 - entropy / (layers.len() as f32).ln()
        } else {
            1.0
        };

        let weak_signal_tags = if l2_norm(&residual) / initial_energy > self.weak_signal_threshold {
            self.mine_residual(&residual, tag_centroids)
        } else {
            vec![]
        };

        TideResult {
            layers,
            focus_level,
            residual,
            weak_signal_tags,
        }
    }

    pub fn apply_gravity_field(
        &self,
        query: &[f32],
        gravity_sources: &[GravitySource],
    ) -> Vec<f32> {
        let mut adjusted = query.to_vec();
        for source in gravity_sources {
            let dist = euclidean_distance(query, &source.center);
            if dist < f32::EPSILON {
                continue;
            }
            let force = self.gravity_strength * source.mass
                       / dist.powf(self.gravity_decay_exponent);
            let direction = normalize(&subtract(&source.center, query));
            adjusted = add(&adjusted, &scale(&direction, force));
        }
        normalize(&adjusted)
    }

    fn mine_residual(
        &self,
        residual: &[f32],
        tag_centroids: &[TagCentroid],
    ) -> Vec<TagId> {
        let mut scored: Vec<_> = tag_centroids.iter()
            .map(|t| (t.id.clone(), cosine(residual, &t.vector)))
            .collect();
        scored.sort_by(|a, b| b.1.partial_cmp(&a.1).unwrap());
        scored.into_iter()
            .take(self.weak_signal_top_k)
            .map(|(id, _)| id)
            .collect()
    }
}

pub struct TideResult {
    pub layers: Vec<TideLayer>,
    pub focus_level: f32,
    pub residual: Vec<f32>,
    pub weak_signal_tags: Vec<TagId>,
}

pub struct TideLayer {
    pub tag_id: TagId,
    pub energy_ratio: f32,
}

pub struct GravitySource {
    pub center: Vec<f32>,
    pub mass: f32,           // √doc_count × avg_cooccur × recency
}
```

## 4. 默认参数

| 参数 | 默认值 | 建议范围 | 说明 |
|---|---:|---:|---|
| `max_layers` | 5 | 3~10 | 最大剥离层数 |
| `residual_threshold` | 0.05 | 0.01~0.2 | 残差终止阈值 |
| `gravity_strength` | 0.2 | 0.05~0.5 | 引力场强度 G |
| `gravity_decay_exponent` | 2.0 | 1.5~3.0 | 距离衰减指数 |
| `weak_signal_threshold` | 0.05 | 0.01~0.20 | 启动弱信号挖掘的最低残差能量 |
| `weak_signal_top_k` | 5 | 3~10 | 挖掘的 Tag 数量 |

## 5. 输出语义

```text
TideResult {
  layers              [(tag_id, energy_ratio), ...]
                      按能量降序，主主题在前

  focus_level ∈ [0,1] 高 = 意图聚焦；低 = 意图发散
                      决定下游检索 Top-K（高聚焦取小 K）

  residual            最终残差向量（用于 weak_signal_tags 与 query 修正）

  weak_signal_tags    被主主题掩盖的隐含语义
                      作为 tagmemo / tagmemo_geodesic 的额外种子
}
```

## 6. 与其他模式的协作

### 6.1 单独使用 (mode = tide)

```text
1. Thalamus.decompose(query, tag_centroids)
2. 用 layers 的 Tag 作为种子 → Synapse.expand
3. 候选 = HNSW 直接命中 ∪ Synapse 扩散命中 ∪ weak_signal 命中
4. 按 final_score 排序
```

### 6.2 与引力场结合

```text
1. apply_gravity_field(query, sources) → query'
2. 用 query' 重新检索
3. Thalamus.decompose 在结果上做意图分析
```

### 6.3 在 hybrid 模式中辅助

```text
hybrid 模式可调用 apply_gravity_field 修正 query 向量后再检索，
但不调用 decompose（避免计算开销）
```

## 7. 性能与降级

```text
计算复杂度：
  decompose       O(max_layers × |tag_centroids|)
  apply_gravity   O(|gravity_sources|)

典型规模：
  tag_centroids ≤ 10000
  gravity_sources ≤ 50
  max_layers ≤ 5

性能预算：
  decompose       ≤ 30ms
  apply_gravity   ≤ 10ms
```

降级触发：详见 [retrieval-fallback.md](./retrieval-fallback.md)。

## 8. 不变量

```text
∀ layer ∈ layers:    0 ≤ layer.energy_ratio ≤ 1
focus_level ∈ [0, 1]
|residual|² + Σ|projection_i|² = |query|²       (Gram-Schmidt 性质)
```

## 9. 与最终版差距

```text
最终版                       本文档
-----                        -----
tide：仅出现在模式列表        完整 Gram-Schmidt + 投影熵 + 引力场规格
       无任何数学定义         参数表 + 实现规范 + 性能预算
                              + weak_signal 挖掘
```

详见 `../../3-记忆系统深度分析与未来方案.md` §3.2 与 §4.5。
