# data-plane-merkle.md

Data Plane 大文件传输 Merkle 化。Envelope 与 raw stream 分离。

## 1. 原则

```text
Envelope（走签名链路）只承载:
  DataOpen     - 声明将要传输的对象
  DataCommit   - 声明传输完成 + Merkle root
  ObjectRef    - 对象句柄

Raw chunk 流（旁路）:
  不走 Protobuf bytes 主路径
  不经过 Envelope 签名
  完整性由 Merkle root 保证
  可选择独立传输（QUIC stream / HTTP multipart / S3 直传）
```

## 2. 为什么

```text
问题: 把整个 chunk 塞进 Envelope.DataChunk.data bytes 会让:
  - 签名链路被 100MB 数据拖垮
  - Protobuf 解码成为性能瓶颈
  - QUIC 流控与应用层背压耦合混乱
  - 对象存储与消息管道混合

方案: 把 chunk 流从签名链路剥离，只签元数据
      chunk 完整性由 Merkle root（签在 DataCommit 里）保证
```

## 3. 消息定义

```proto
message DataOpen {
  string stream_id = 1;
  string object_id = 2;
  string media_type = 3;
  uint64 expected_size_bytes = 4;
  bytes expected_merkle_root = 5;      // 可选：事先声明
  uint32 chunk_size_bytes = 6;         // 建议 chunk 大小
  MerkleAlgorithm merkle_algorithm = 7;
  TransferEndpoint transfer = 10;
}

enum MerkleAlgorithm {
  MERKLE_ALGORITHM_UNSPECIFIED = 0;
  MERKLE_SHA256_BINARY = 1;            // 标准二叉 Merkle
  MERKLE_BLAKE3 = 2;                   // BLAKE3 Merkle
}

message TransferEndpoint {
  TransferScheme scheme = 1;
  string uri = 2;                      // quic://stream/... / https://... / s3://...
  map<string, string> credentials = 3; // 预签名参数（时效短）
}

enum TransferScheme {
  TRANSFER_SCHEME_UNSPECIFIED = 0;
  TRANSFER_QUIC_STREAM = 1;            // 同一 QUIC 连接内独立流
  TRANSFER_HTTPS_STREAM = 2;           // HTTPS chunked
  TRANSFER_HTTPS_MULTIPART = 3;        // multipart/form-data
  TRANSFER_OBJECT_STORE = 4;           // S3/MinIO 直传
}

message DataCommit {
  string stream_id = 1;
  string object_id = 2;
  uint64 total_size_bytes = 3;
  bytes merkle_root = 4;               // 最终 Merkle root，签名在 Envelope 上
  uint64 chunk_count = 5;
  MerkleAlgorithm merkle_algorithm = 6;
}

message DataChunkMeta {
  // 仅用于断点续传/校验，不含 data
  string stream_id = 1;
  uint64 chunk_index = 2;
  uint64 offset = 3;
  uint32 length = 4;
  bytes chunk_hash = 5;
  bool eof = 6;
}

message DataAck {
  string stream_id = 1;
  string object_id = 2;
  uint64 received_bytes = 3;
  uint64 next_offset = 4;
  bool complete = 5;
  bytes partial_merkle_root = 6;       // 可选：已接收部分的 Merkle root
}
```

说明：**没有** `DataChunk.data = bytes` 字段。data 走旁路流。

## 4. 传输流程

```text
发送方:
  1. 准备对象，分片（chunk_size_bytes）
  2. 计算每 chunk 的 hash
  3. 构造 Merkle tree
  4. 计算 Merkle root
  5. 发送 DataOpen（Envelope 签名，含 expected_merkle_root 可选）
  6. 通过 TransferEndpoint 发送 chunk 流（旁路）
     每个 chunk 前附 DataChunkMeta（可选，便于断点）
  7. 发送 DataCommit（Envelope 签名，含最终 merkle_root）

接收方:
  1. 收到 DataOpen，校验签名
  2. 从 TransferEndpoint 拉取 chunk
  3. 边接收边构造本地 Merkle tree
  4. 定期发 DataAck 含 partial_merkle_root
  5. 收到 DataCommit，比对本地 Merkle root 与 commit.merkle_root
  6. 若一致 → 对象可用
  7. 若不一致 → 拒绝，Problem: OBJECT_HASH_MISMATCH
```

## 5. TransferEndpoint 选择

### 5.1 QUIC Stream

```text
scheme = TRANSFER_QUIC_STREAM
uri = quic://current/stream/<stream_id>
在同一 QUIC 连接内开独立 unidirectional stream 传 chunk
不走 Envelope 解码，纯 bytes 传输
```

### 5.2 HTTPS Stream / Multipart

```text
scheme = TRANSFER_HTTPS_STREAM / TRANSFER_HTTPS_MULTIPART
uri = https://host/fap/v1/data/<object_id>
POST 流式 body 或 multipart
credentials 含时效短 token（类似 S3 pre-signed URL）
```

### 5.3 Object Store 直传

```text
scheme = TRANSFER_OBJECT_STORE
uri = s3://bucket/key
credentials 含 pre-signed PUT/GET URL
接收方直接从对象存储拉取
适合大文件 + 多接收方场景
```

## 6. 断点续传

```text
中断后重连:
  1. 发 DataAck(received_bytes, next_offset, partial_merkle_root)
  2. 发送方比对 partial_merkle_root
  3. 若一致 → 从 next_offset 继续
  4. 若不一致 → 找出分歧分片，仅重传该分片子树
```

## 7. 签名与完整性

```text
签名覆盖:
  DataOpen（含 expected_merkle_root 可选）
  DataCommit（含最终 merkle_root）
  DataAck 汇报点

不签:
  raw chunk bytes
  DataChunkMeta（可选签）

完整性保证:
  DataCommit.merkle_root 已被 Envelope 签
  chunk bytes 通过 Merkle root 间接被签
  篡改任何 chunk → 本地 Merkle root 与 commit 不符 → 拒绝
```

## 8. 流量控制

```text
Data Plane 背压对 raw stream 适用:
  QUIC stream 有 MAX_STREAM_DATA
  HTTPS 有 TCP 流控 + chunked size 控制
  S3 直传由客户端控制拉取速率

FAP FlowCredit 用于语义级信号:
  接收方内存压力 / 磁盘压力 → SLOW_DOWN
  FlowCredit 经 Envelope 传输，不影响 raw stream
```

## 9. 兼容性

```text
Core Lite（无 Data Plane）:
  InvokeRequest.input 内嵌小对象（≤ maxControlFrameBytes）
  不支持大文件

Data Plane 启用:
  大对象必须走 Merkle 化 Data Plane
  InvokeRequest 传 ObjectRef，不传原始 bytes
```

## 10. 测试要点

```text
Merkle root 校验:
  篡改任一 chunk → 拒绝
  乱序到达 chunk → 按 offset 重建后仍正确
  丢失 chunk → DataAck 提示缺失

断点续传:
  中断 → partial_merkle_root 正确
  恢复后 → 从 next_offset 继续
  分歧时 → 仅重传分歧子树

多端点:
  QUIC / HTTPS / S3 均能传输
  切换端点不破坏 Merkle 校验
```
