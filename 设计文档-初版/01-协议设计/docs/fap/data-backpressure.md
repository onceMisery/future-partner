# data-backpressure.md

数据面背压机制。TransportPlugin 与 Core 的背压协同契约。

## 1. 目标

```text
防止大文件传输拖垮 Agent 逻辑线程
防止 Data Plane 挤占 Control Plane
防止慢节点造成内存无限堆积
防止远程对象流在网络抖动下形成雪崩重试
```

## 2. 可插拔视角

```text
TransportPlugin 各自实现底层流控:
  QUIC TransportPlugin → stream window / connection window
  HTTP/JSON TransportPlugin → TCP backpressure + SSE heartbeat
  WebTransport TransportPlugin → 本协议流控
  
FAP Core 提供语义级背压:
  FlowCredit / DataPause / DataResume
  所有 TransportPlugin 必须支持
```

## 3. 背压层级

| 层级 | 机制 | 作用 |
|---|---|---|
| TransportPlugin Transport | 底层字节流控 | 控制传输速率 |
| FAP Data Plane | FlowCredit / DataAck / Pause / Resume | 控制应用层吞吐 |
| Scheduler | per-session quota / per-node quota | 防任务饥饿 |
| Memory Guard | max buffered bytes | 防内存打满 |
| Priority Queue | control > audit > data | 保证控制面优先 |

## 4. FlowCredit

```proto
message FlowCredit {
  string session_id = 1;
  string stream_id = 2;

  uint64 granted_bytes = 3;
  uint32 granted_chunks = 4;

  uint64 valid_until_sequence_no = 5;
  uint32 recommended_chunk_bytes = 6;

  FlowPressure pressure = 7;
  FlowTransportHint transport_hint = 8;
}

enum FlowPressure {
  FLOW_PRESSURE_UNSPECIFIED = 0;
  FLOW_PRESSURE_NORMAL = 1;
  FLOW_PRESSURE_WARN = 2;
  FLOW_PRESSURE_SLOW_DOWN = 3;
  FLOW_PRESSURE_PAUSE = 4;
  FLOW_PRESSURE_CANCEL = 5;
}

enum FlowTransportHint {
  FLOW_TRANSPORT_HINT_UNSPECIFIED = 0;
  QUIC_PIGGYBACK = 1;
  HTTP_PRIMARY = 2;
}

message DataPause {
  string stream_id = 1;
  string reason = 2;
  uint32 retry_after_ms = 3;
}

message DataResume {
  string stream_id = 1;
  uint64 next_offset = 2;
  uint32 recommended_chunk_bytes = 3;
}
```

## 5. 协同规则

### 5.1 QUIC 模式

```text
FlowCredit 仅用于语义级信号（"我磁盘写慢了"）
不参与字节级调度
FlowCredit.granted_bytes 必须 ≥ QUIC MAX_STREAM_DATA 剩余，否则无效
transport_hint = QUIC_PIGGYBACK
```

### 5.2 HTTP 模式

```text
FlowCredit 作为主要背压手段
SSE 心跳事件承载 pressure 状态
transport_hint = HTTP_PRIMARY
客户端按 recommended_chunk_bytes 调整 multipart 分片
```

### 5.3 其它 Transport

```text
WebTransport / 自定义 Transport 实现:
  必须支持 FlowCredit 语义
  必须保证 Control 消息优先级
  必须能传递 DataPause / DataResume
```

## 6. Pressure 事件驱动

不使用加权公式，改独立阈值触发：

```text
MEM:   buffered_bytes > memory_high_water   → SLOW_DOWN
       buffered_bytes > memory_pause_water  → PAUSE
DISK:  fsync_p99_ms > 200                   → SLOW_DOWN
QUEUE: data_queue_depth > cap * 0.8         → SLOW_DOWN
NET:   retransmit_rate > 5%                 → SLOW_DOWN
CPU:   load_avg > cores * 0.9               → WARN

恢复: 全部维度 < low_water 持续 10s → NORMAL
```

每个维度阈值在配置中独立可调，由 OpenTelemetry 暴露实时指标。

## 7. 优先级保证

```text
Envelope.Header.priority ∈ { CONTROL, AUDIT, DATA }

调度队列:
  Priority 0 Control > Priority 1 Audit > Priority 2 Data

必须保证:
  Control Stream 永远优先于 Data Stream
  Receipt / Problem / Cancel 优先于 ProgressEvent
  Data Plane 不得阻塞 Auth / Cancel / Receipt
```

## 8. 取消策略

```text
pressure > 0.95 持续 3s:
  CANCEL 所有低优先级 data stream
  保留 control stream
  触发 Problem: RATE_LIMIT_EXCEEDED
```

## 9. 配置示例

```yaml
backpressure:
  memory_high_water_mb: 512
  memory_pause_water_mb: 768
  memory_max_mb: 1024
  fsync_p99_threshold_ms: 200
  data_queue_cap: 1024
  retransmit_rate_threshold: 0.05
  cpu_load_warn_ratio: 0.9
  normal_hold_seconds: 10
  cancel_pressure_duration_seconds: 3
```

## 10. 测试要点

见 `../20-testing-conformance.md`，必测：

```text
接收端内存压力升高时发送 SLOW_DOWN
接收端磁盘阻塞时发送 PAUSE
控制面消息不被 DataChunk 阻塞
CANCEL 能中断大文件流
chunk size 能根据 FlowCredit 动态调整
QUIC 模式下 FlowCredit 不与 QUIC 窗口冲突
HTTP 模式下 SSE 心跳承载 pressure
```
