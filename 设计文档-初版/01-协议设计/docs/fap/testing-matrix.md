# testing-matrix.md

测试硬门禁四大类 + 三档性能目标。

## 1. 四类硬门禁

```text
1. 协议 Golden Test    协议行为跨版本一致
2. 跨语言互操作        Rust / Java / TS / Python 字节级一致
3. 恶意插件/桥接负例   Kernel Contract 边界不被越过
4. 混沌测试            真实部署条件下韧性
```

任何 FAP 发版必须四类全绿。

## 2. 协议 Golden Test

```text
目标: 锁定协议行为，防止 silent 破坏兼容

输入:
  conformance/golden/<phase>/<scenario>/
    envelope.json          人类可读
    envelope.bin           proto binary
    signed_bytes.bin       规范化编码
    expected_signature.hex
    description.md

测试方式:
  每次 proto 变更重新生成 signed_bytes.bin
  对比旧版 signed_bytes.bin
  若 signed_bytes 变化而字段 tag 未变 → 拒绝
  
规模:
  Core Lite:       ≥ 50 scenarios
  Core Secure:     ≥ 80 scenarios
  Plugin Runtime:  ≥ 40 scenarios
  Data Plane:      ≥ 30 scenarios
  HP Local:        ≥ 20 scenarios
  HP Cluster:      ≥ 30 scenarios
  EXT:             每特性 ≥ 10 scenarios
```

## 3. 跨语言互操作

```text
矩阵:
                  Client
  Server     Rust  Java  TS  Python
  Rust        ✓     ✓    ✓    ✓
  Java        ✓     ✓    ✓    ✓
  TS          ✓     ✓    ✓    ✓
  Python      ✓     ✓    ✓    ✓

每个 cell 必测:
  ClientHello / ServerHello（含 version_commitment）
  AuthInit / AuthAck
  InvokeRequest / InvokeResult
  BatchInvoke DAG
  ResumeRequest / ResumeAck
  Receipt Multisig 生成与验证
  DataOpen / DataCommit 与 Merkle 校验
  
CI 并行执行
任一 cell 失败 → 阻断合并
```

## 4. 恶意插件 / 桥接负例

### 4.1 恶意插件

```text
插件沙箱逃逸（按 WASM / Sidecar / Native 分别测）:
  读系统敏感文件 → 拒绝
  建立外联网络 → 拒绝（非 manifest 授权）
  执行 subprocess → 拒绝
  访问环境变量 → 拒绝
  突破内存限制 → OOM
  突破 CPU 限制 → timeout
  访问其他插件私有状态 → 拒绝
  
Kernel Contract 边界（见 kernel-contract.md §8）:
  插件访问其他 session → 拒绝
  插件修改 Receipt → 拒绝
  插件跳过 mandate → 拒绝
  插件伪造 signature → 拒绝
  插件写 Replay Cache → 拒绝
  插件重排 Policy → 拒绝
  插件直接写 hash_chain_tip → 拒绝
  插件 list 非自有 capability → 拒绝
```

### 4.2 恶意桥接

```text
MCP Bridge 负例:
  Bridge 不带 FAP mandate 转发 high 能力 → 拒绝
  Bridge 缓存 Auth 跨请求 → 拒绝
  Bridge 代 Client 签 FAP signature → 拒绝
  Bridge 转发时内部短路 Core → 拒绝
  
A2A Bridge 负例: 同上
  
自定义 Bridge 负例:
  Bridge 声称支持能力但不暴露给 Capability Registry → 拒绝
  Bridge 生成自己的 Receipt 绕过 Core → 拒绝
```

### 4.3 恶意客户端

```text
篡改 Header:
  payload_sha256 不匹配 → 拒绝
  prev_hash 断链 → 拒绝
  expires_at 超窗 → 拒绝
  signature 不匹配 → 拒绝

降级攻击:
  篡改 supported_protocol_range → version_commitment 校验失败
  声明不支持 Secure Profile → 若 required 则拒绝

重放攻击:
  重复 message_id → 拒绝
  0-RTT 重放非幂等 → 拒绝
  跨 session 重放 VP → 拒绝

Mandate 伪造:
  篡改 constraints → issuer_signature 失败
  无签发 issuer_signature → 拒绝
```

## 5. 混沌测试

```text
进程级:
  随机 kill worker plugin sidecar
  随机 kill node main process
  随机 panic in plugin
  
网络级:
  随机延迟 50~2000ms
  随机丢包 0.1% ~ 5%
  随机分区（脑裂）
  随机重连
  
资源级:
  内存压力（mlock 占用）
  磁盘满 / fsync 慢
  CPU 打满
  文件描述符耗尽
  
时钟级:
  随机时钟偏移 ±5s
  时钟回拨
  NTP 失联

验证:
  in-flight invocation 不丢失
  Receipt chain 不断裂
  Replay Cache 不泄漏
  Session 能 Resume
  背压自动触发
  健康状态收敛
```

## 6. 性能目标（三档）

### 6.1 local-dev

```text
场景: 开发者笔记本，单进程，本地回环

目标:
  单 Invoke p50/p95/p99:     < 3ms / 10ms / 30ms
  Receipt 单签:              < 1ms
  Receipt 多签同步 3 方:     < 15ms
  Agent Card 拉取 p99:       < 50ms
  1000 并发 Invoke 吞吐:     ≥ 15k req/s
  100MB ObjectRef 传输:      ≥ 800MB/s（本地回环）
  Replay Cache 查询 hit:     < 50μs
  Replay Cache 查询 miss:    < 200μs
  QUIC Session 建立:         < 30ms
```

### 6.2 edge

```text
场景: 边缘设备 / 低功耗 Agent / 单机生产

目标:
  单 Invoke p50/p95/p99:     < 10ms / 30ms / 80ms
  Receipt 单签:              < 3ms
  Receipt 多签同步 3 方:     < 40ms
  Agent Card 拉取 p99:       < 200ms
  200 并发 Invoke 吞吐:      ≥ 3k req/s
  100MB ObjectRef 传输:      ≥ 200MB/s
  Replay Cache 查询 hit:     < 100μs
  Replay Cache 查询 miss:    < 500μs
  QUIC Session 建立:         < 100ms

备注:
  AUDITED_ASYNC 默认，不等待 AUDIT 同步
  APPROVED_ASYNC 默认
```

### 6.3 gateway

```text
场景: 企业网关 / 集群入口 / 多节点

目标:
  单 Invoke p50/p95/p99:     < 15ms / 50ms / 150ms（含远程路由）
  Receipt 多签异步补签:      返回延迟 < 10ms
  Receipt 多签同步 3 方:     < 60ms
  Agent Card 拉取 p99:       < 300ms
  5000 并发 Invoke 吞吐:     ≥ 50k req/s（集群累计）
  100MB ObjectRef 跨节点:    ≥ 500MB/s（内网）
  Replay Cache 查询（共享）: < 2ms
  QUIC Session 建立:         < 150ms
  节点健康收敛时间:          < 30s
```

## 7. CI 集成

```yaml
# .github/workflows/test-matrix.yml
name: Test Matrix
on: [push, pull_request]
jobs:
  golden:
    strategy:
      matrix:
        lang: [rust, java, ts, python]
    steps:
      - run: ./scripts/test-golden.sh ${{ matrix.lang }}

  interop:
    strategy:
      matrix:
        server: [rust, java, ts, python]
        client: [rust, java, ts, python]
    steps:
      - run: ./scripts/test-interop.sh ${{ matrix.server }} ${{ matrix.client }}

  negative:
    steps:
      - run: cargo test --test malicious_plugin
      - run: cargo test --test malicious_bridge
      - run: cargo test --test malicious_client

  chaos:
    if: github.ref == 'refs/heads/main'
    timeout-minutes: 60
    steps:
      - run: ./scripts/chaos/run.sh --duration 30m

  bench:
    strategy:
      matrix:
        tier: [local-dev, edge, gateway]
    steps:
      - run: ./scripts/bench.sh ${{ matrix.tier }}
      - run: ./scripts/bench-regression-check.sh ${{ matrix.tier }} 10
```

任何门禁失败 → PR 阻断。

## 8. Fuzz

```text
cargo-fuzz harness:
  - envelope_decoder
  - receipt_verifier
  - signature_canonical
  - mandate_chain_validator
  - dag_cycle_detector
  - merkle_tree_builder
  - header_validator

要求:
  CI 跑 ≥ 1h
  nightly 跑 ≥ 24h
  语料从 golden 样本演化
  无 panic / OOM / UB
  发现 crash 自动归并到 corpus
```

## 9. 形式化（可选强化）

```text
TLA+ 建模:
  Session state machine
  Receipt hash chain 不断裂
  Multisig 不能伪造
  Resume 不导致重复 Invoke
  Idempotency 唯一性
  Mandate 委托深度有界

至少验证 3 条关键不变量作为 RFC 附录
```
