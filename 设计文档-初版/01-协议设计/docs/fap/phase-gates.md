# phase-gates.md

路线图 Phase 依赖门禁。每个 Phase 必须通过门禁才能进入下一阶段。

## 1. 门禁总览

```text
Phase 0 冻结
  ↓ 【G0】 proto 冻结 + 四版本线初始化 + 所有规范文档齐全
Phase 1 Core Lite
  ↓ 【G1】 Golden Corpus 100% + Inspector Decode + Interop
Phase 2 Core Hardening
  ↓ 【G2】 Downgrade/Replay/Mandate/Receipt 负例 + 跨语言互操作
Phase 3 Plugin Runtime
  ↓ 【G3】 Sandbox Escape 测试 + 恶意插件负例
Phase 4 Data Plane
  ↓ 【G4】 Merkle 完整性 + 断点续传 + 大对象性能
Phase 5 HP Local
  ↓ 【G5】 DAG 调度 + 性能基准 + 混沌测试
Phase 6 HP Cluster
  ↓ 【G6】 分布式共识 + NodeHealth + 网络分区
Phase 7 EXT
  ↓ 【G7】 各 EXT 独立门禁（见各特性）
```

## 2. G0 协议冻结门禁

**必过项**：

```text
proto/fap/v1/fap.proto         buf lint 0 错误
proto/plugin-api/v1/*.proto    buf lint 0 错误
proto/fap/v1/ALLOCATIONS.md    字段号分配完整
schema/agent-card.schema.json  JSON Schema 校验通过
schema/plugin-manifest.schema.json
docs/fap/ 下 32 份文档齐全
代码生成通过（Rust / Java / TS / Python）
Golden Protobuf 样本生成
buf breaking CI 就绪
```

**失败回退**：若任一项未过，不得开启 Phase 1 代码。

## 3. G1 Core Lite 门禁

**必过项**：

```text
Golden Corpus:
  ≥ 200 scenarios
  Rust / Java / TS / Python 四实现解码一致（byte-level）
  解码错误率 0%

Inspector Decode:
  fap inspect envelope <file.bin> 完全解码所有 golden 样本
  fap inspect card / session / hash-chain 可用

Interop Matrix:
  Rust ↔ Java 双向 Hello / Invoke / Resume 通过
  Rust ↔ TS 通过
  Java ↔ Python 通过

功能验收:
  低功耗设备可运行（测试设备：Raspberry Pi 4 或等效）
  单机开发者 10 分钟内启动
  重复请求不会重复执行
  Receipt hash chain 可查看
```

**失败回退**：不进 Phase 2，先修 Core Lite bug。

## 4. G2 Core Hardening 门禁

**必过项 - 负例测试**：

```text
Downgrade 攻击:
  中间人篡改 supported_protocol_range → 被拒（version_commitment 校验失败）
  Profile 降级到关闭 Secure → 被拒

Replay 攻击:
  重复 message_id → Problem: REPLAY_DETECTED
  已过期 message → Problem: EXPIRES_AT_OUT_OF_WINDOW
  篡改 payload_sha256 → Problem: PAYLOAD_HASH_MISMATCH

Mandate 负例:
  无 mandate 调 high 能力 → 拒绝
  过期 mandate → 拒绝
  子 mandate 超父 capabilities → 拒绝
  委托深度超限 → 拒绝
  撤销 mandate 后 session 进入 DRAINING

Receipt 负例:
  缺 EXECUTOR 签名 → 拒绝
  high 缺 AUDIT_CENTER → 拒绝
  signature kid 不可信 → 拒绝
  previous_receipt_hash 断链 → 节点进入 DEGRADED

版本协商负例:
  Prerelease 不在 allowlist → 拒绝
  无交集区间 → AuthReject
```

**失败回退**：不进 Phase 3。

## 5. G3 Plugin Runtime 门禁

**必过项 - Sandbox Escape 测试**：

```text
WASM 逃逸:
  插件尝试读 /etc/passwd → 拒绝
  插件尝试 open 网络 socket → 拒绝
  插件内存超限 → OOM Kill
  插件 CPU 超限 → timeout
  WASM import 只含白名单符号

Sidecar 逃逸:
  Sidecar 尝试读宿主文件系统（非 mount 点）→ 拒绝
  Sidecar seccomp 阻止危险 syscall
  Sidecar cgroups 限制 CPU/内存/IO
  Sidecar 网络 namespace 隔离

恶意插件负例（见 kernel-contract.md §8）:
  插件访问其他 session → 拒绝
  插件修改 Receipt → 拒绝
  插件跳过 mandate → 拒绝
  插件伪造 signature → 拒绝
  插件写 Replay Cache → 拒绝
  插件重排 Policy 顺序 → 拒绝
  插件直接写 hash_chain_tip → 拒绝

生命周期:
  STAGED 失败不影响生产
  升级时版本并存成功
  DRAINING 不影响在途调用
  回滚可恢复
```

**失败回退**：不进 Phase 4。

## 6. G4 Data Plane 门禁

**必过项**：

```text
Merkle 完整性:
  篡改任一 chunk → 拒绝（DataCommit 不匹配）
  乱序到达 → 重建后校验通过
  丢失 chunk → DataAck 精确标记缺失

断点续传:
  中断 → partial_merkle_root 可恢复
  分歧子树仅重传
  多端点切换不破坏完整性

性能:
  100MB ObjectRef 本地回环 ≥ 500MB/s
  QUIC stream 单文件 ≥ 1Gbps
  HTTPS 多部件 ≥ 500Mbps
  S3 直传功能验证

背压:
  FlowCredit SLOW_DOWN 生效
  DataPause / DataResume 生效
  Control Plane 不被 Data 阻塞
```

**失败回退**：不进 Phase 5。

## 7. G5 HP Local 门禁

**必过项**：

```text
DAG 调度:
  100 节点 DAG 正确拓扑排序
  环检测在 Ack 前完成
  层内并行
  层间等待
  依赖不存在的 invocation → 拒绝

性能（三档基准，见 performance-targets.md）:
  local-dev / edge / gateway 三档指标达标
  退化 > 10% 阻断合并

混沌测试:
  随机杀死 worker → 任务重试
  随机延迟网络 → 不破坏一致性
  随机磁盘慢 → 背压触发
  随机内存压力 → PAUSE 生效
```

**失败回退**：不进 Phase 6。

## 8. G6 HP Cluster 门禁

**必过项**：

```text
节点健康:
  UNHEALTHY 节点剔除
  QUARANTINED 状态生效
  15min 恢复窗
  人工恢复

网络分区（单 Main 拓扑）:
  Worker 分区后被剔除或进入 DEGRADED
  Main Node 分区时拒绝新 high/critical 请求或进入 DRAINING
  Receipt chain 不断裂
  Object Replica 一致性

多 Main 拓扑:
  不属于 Phase 6 默认交付
  若实现者启用，必须提供外部 leader fencing / quorum 证明

分布式一致性:
  Register Capabilities 最终一致
  Health 状态收敛
  Router 跨节点路由正确

跨节点 RemoteFileResolver:
  origin_node 下线 → 切 replica
  所有副本失效 → OBJECT_UNAVAILABLE
  lease 续租跨节点
```

**失败回退**：不进 Phase 7。

## 9. G7 EXT 独立门禁

每个 EXT 特性独立门禁：

```text
DID/VC:
  did:web / did:key 解析成功
  VC 验证 / 撤销检查
  缓存策略生效
  resolver 超时 fail-closed

Latent Channel:
  text shadow 始终存在
  readable projection 必有
  high/critical 禁用
  fallback Data Plane 可用

Gossip:
  只传 Agent Card 摘要
  不传 token / mandate
  TTL 生效
  限速生效

CRDT:
  收敛性验证
  不承载授权
```

## 10. 门禁 CI 实现

```yaml
# .github/workflows/phase-gates.yml
name: Phase Gate
on: [pull_request]
jobs:
  g1-core-lite:
    if: contains(github.event.pull_request.labels.*.name, 'phase-1')
    steps:
      - run: cargo test --test golden_corpus
      - run: cargo test --test interop_matrix
      - run: fap inspect envelope tests/golden/*.bin
  g2-hardening:
    if: contains(github.event.pull_request.labels.*.name, 'phase-2')
    steps:
      - run: cargo test --test negative_downgrade
      - run: cargo test --test negative_replay
      - run: cargo test --test negative_mandate
      - run: cargo test --test negative_receipt
  g3-plugin-runtime:
    if: contains(github.event.pull_request.labels.*.name, 'phase-3')
    steps:
      - run: cargo test --test sandbox_escape
      - run: cargo test --test malicious_plugin
  # ...
```

门禁失败 = PR 不能合并。

## 11. 门禁回溯

```text
若 Phase N 通过后发现问题:
  回到对应门禁补测
  补测不过 → 该阶段退回 DRAFT 状态
  影响后续 Phase 的需同步回滚或打补丁
```
