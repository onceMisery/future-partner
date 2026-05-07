# 20 - 测试与一致性

> 四类硬门禁与三档性能目标见 [testing-matrix.md](./testing-matrix.md)。
> Phase 门禁索引见 [phase-gates.md](./phase-gates.md)。

## 1. 测试类型

```text
Unit Test
Protocol Golden Test
Conformance Test
Security Test
Fault Injection
Performance Bench
Interop Matrix
Plugin Sandbox Test
Fuzz Test
（可选）形式化验证 TLA+
```

## 2. 必测场景

见 `../4.FAP-1最终实现方案.md` 第 19 章。

关键：

```text
SemVer 协商
Replay / Idempotency
Signing canonical
Version Downgrade
Mandate 委托链
Receipt Multisig
Data Backpressure
ObjectRef TTL + Lease
DID/VC 降级
Core Lite 可运行
Node Health FSM
DAG Batch
```

## 3. 性能基准

| 指标 | MVP 目标 |
|---|---|
| 单 Invoke p50/p95/p99 | < 5ms / 20ms / 50ms |
| Receipt 生成（单签） | < 2ms |
| Receipt 多签（3 方，异步补签） | 返回延迟 < 5ms |
| Agent Card 拉取 p99 | < 200ms |
| QUIC Session 建立 p99 | < 100ms |
| 1000 并发 Invoke 吞吐 | ≥ 10k req/s |
| 100MB ObjectRef 传输吞吐 | ≥ 500MB/s |

## 4. Fuzz

```text
cargo-fuzz harness:
  - envelope_decoder
  - receipt_verifier
  - signature_canonical
  - mandate_chain_validator
  - dag_cycle_detector

要求: CI 跑 ≥ 1h；nightly 跑 ≥ 24h
```

## 5. 互操作矩阵

```text
                  Client
  Server     Rust  Java  TS  Python
  Rust        ✓     ✓    ✓    ✓
  Java        ✓     ✓    ✓    ✓
  TS          ✓     ✓    ✓    ✓
  Python      ✓     ✓    ✓    ✓

每个 cell 跑完整 conformance 测试集（golden ≥ 200 scenarios）
```

## 6. FAP-Inspector

```text
fap inspect card <url>
fap inspect session <session_id>
fap inspect envelope <file.bin>
fap inspect receipt <file.bin> --require executor --require audit-center
fap inspect hash-chain <session_id>
fap inspect object <object_id>
fap inspect flow --session <id>
fap replay <corpus_dir>
fap diff <file-a.bin> <file-b.bin>
```
