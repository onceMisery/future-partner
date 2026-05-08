# retain-score.md

> 量化标准：L0 折叠触发、retain_score 公式、L2 写入门槛、L2 冲突仲裁。

## 1. retain_score 公式

```text
retain_score(event) =
    0.30 * salience
  + 0.25 * stability
  + 0.20 * user_preference_signal
  + 0.15 * task_reuse_value
  + 0.10 * source_confidence
  - 0.20 * privacy_risk
  - 0.15 * noise_score
```

返回值范围：约 `[-0.35, +1.0]`。

### 各项定义

| 字段 | 含义 | 取值方式 |
|---|---|---|
| `salience` | 显著性，事件在会话中的重要程度 | LLM 评分 / 用户标记 |
| `stability` | 稳定性，事实是否长期成立 | 历史一致性检测 |
| `user_preference_signal` | 用户偏好信号 | 用户主动标记或反复使用 |
| `task_reuse_value` | 任务可复用性 | 任务模板匹配度 |
| `source_confidence` | 来源可信度 | 来源 trust 分数 |
| `privacy_risk` | 隐私风险 | PII 检测 + 安全标签 |
| `noise_score` | 噪音分 | 无意义寒暄 / 重复 / 错误内容 |

所有字段均归一化到 `[0, 1]`。

## 2. retain_score → 分层决策

```rust
pub fn decide_layer(score: f32) -> LayerDecision {
    match score {
        s if s < 0.35 => LayerDecision::L0Only,         // 仅留 L0，到期清除
        s if s < 0.60 => LayerDecision::IntoL1,          // 进入剧集摘要
        s if s < 0.80 => LayerDecision::IntoL2,          // 进入语义本体
        s if s < 0.90 => LayerDecision::L2HighSalience,  // 高显著性 L2
        _             => LayerDecision::IntoL2Protected, // L2 保护级保留
    }
}
```

`IntoL2Protected` 不表示新的记忆层。它只是在 L2 上设置策略标签：

```text
core_memory = true
retention_tier = "protected"
requires_human_forget_approval = true
```

四层模型仍然只有 L0-L3。旧版本中出现的 `CoreMemory` 枚举应迁移为 `IntoL2Protected` + retention metadata。

## 3. L0 折叠触发

```rust
pub struct L0FoldTrigger {
    token_threshold: usize,           // 默认 4096
    turn_threshold: usize,            // 默认 10
    semantic_boundary_score: f32,     // 默认 0.70
    silence_secs: u64,                // 默认 300
}

pub enum FoldDecision {
    Continue,
    Fold,
    FoldAtBoundary,
    FoldOnTimeout,
    ForceFold,
}

impl L0FoldTrigger {
    pub fn should_fold(&self, state: &L0State) -> FoldDecision {
        // 优先级：强制 > 语义边界 > token > 轮次 > 静默
        if state.total_tokens > self.token_threshold * 2 {
            return FoldDecision::ForceFold;
        }
        if state.semantic_boundary_score > self.semantic_boundary_score {
            return FoldDecision::FoldAtBoundary;
        }
        if state.total_tokens > self.token_threshold {
            return FoldDecision::Fold;
        }
        if state.turn_count >= self.turn_threshold {
            return FoldDecision::Fold;
        }
        if state.silence_secs > self.silence_secs {
            return FoldDecision::FoldOnTimeout;
        }
        FoldDecision::Continue
    }
}
```

## 4. L2 写入门槛（量化）

```rust
pub fn l2_eligible(fact: &ExtractedFact) -> bool {
    fact.stability > 0.6
    && fact.reusability > 0.5
    && fact.source_confidence > 0.7
    && !fact.is_ephemeral_chat
    && !fact.has_active_contradiction       // 存在未解决冲突时不升级
}
```

未通过 → 留在 L1 等待更多证据；DreamWorker 后续可重新评估。

## 5. L2 冲突仲裁

```rust
pub fn resolve_l2_conflict(
    old: &MemoryFact,
    new: &MemoryFact,
) -> ConflictResolution {
    if new.source_confidence > old.source_confidence + 0.15 {
        ConflictResolution::Supersede { keep_lineage: true }
    } else if new.source_confidence > old.source_confidence {
        ConflictResolution::Fork { mark_for_dream_arbitration: true }
    } else {
        ConflictResolution::Reject { reason: "insufficient_confidence_delta" }
    }
}

pub enum ConflictResolution {
    /// new 接替 old；old 标记 superseded，保留 lineage 用于审计
    Supersede { keep_lineage: bool },
    /// new 与 old 共存，标记冲突；DreamWorker 在下次 dream 中提案
    Fork { mark_for_dream_arbitration: bool },
    /// new 被拒绝，写审计
    Reject { reason: &'static str },
}
```

## 6. 默认参数表

| 参数 | 默认值 | 范围 | 说明 |
|---|---:|---:|---|
| token_threshold | 4096 | 1024~16384 | L0 折叠的 token 阈值 |
| turn_threshold | 10 | 5~30 | 折叠的轮次阈值 |
| semantic_boundary_score | 0.70 | 0.5~0.9 | 语义边界检测阈值 |
| silence_secs | 300 | 60~3600 | 静默触发折叠 |
| force_fold_multiplier | 2.0 | 1.5~3.0 | ForceFold 是 token 阈值的几倍 |
| L2_stability_min | 0.6 | 0.4~0.8 | L2 写入最低稳定性 |
| L2_reusability_min | 0.5 | 0.3~0.7 | L2 写入最低可复用性 |
| L2_source_confidence_min | 0.7 | 0.5~0.9 | L2 写入最低来源可信度 |
| conflict_supersede_delta | 0.15 | 0.05~0.30 | 接替阈值 |

所有参数均可在配置中覆盖：

```toml
[compact_strategy.default]
token_threshold = 4096
turn_threshold = 10
semantic_boundary_score = 0.70
silence_secs = 300

[l2_eligibility]
stability_min = 0.6
reusability_min = 0.5
source_confidence_min = 0.7

[conflict_resolution]
supersede_delta = 0.15
```

## 7. 与插件的关系

`CompactStrategy` 插件实现 `should_fold` / `fold` / `retain_score`：

```rust
pub trait CompactStrategy: Plugin {
    fn should_fold(&self, state: &L0State) -> FoldDecision;
    fn fold(&self, events: &[L0Event]) -> Result<EpisodicSummary>;
    fn retain_score(&self, event: &L0Event) -> f32;
}
```

默认实现 `compact-default` 严格按本文档公式。租户可注册自定义实现并通过 `tenant_overrides` 覆盖。

## 8. 评估指标

```text
fold_trigger_recall       折叠应触发但未触发的比例
fold_trigger_precision    折叠误触发比例
l2_eligible_precision     l2_eligible 通过后人工确认正确率
conflict_resolution_acc   冲突仲裁人工标注准确率
```

详见 [17-testing-conformance.md](./17-testing-conformance.md)。

## 9. 与最终版的差异

```text
最终版                            本文档
-----                             -----
L0 折叠触发：未定义               4 级优先级 + 阈值表
retain_score：在 OCMS 总体设计    恢复完整公式
                  中存在，最终版    并定义分层决策
                  消失
L2 写入：定性描述                 量化门槛 + 冲突仲裁
CoreMemory 命名混入层级            改为 L2 protected retention tier
冲突仲裁：未定义                  Supersede / Fork / Reject
```

详见 `../../3-记忆系统深度分析与未来方案.md` §3.1 与 §4.5。
