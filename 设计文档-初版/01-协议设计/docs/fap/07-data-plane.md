# 07 - 数据面

数据面用于高效传输大对象：文件、图片、音频、视频、日志、模型输出片段、embedding、压缩包、远程节点对象。

## 1. ObjectRef

```proto
message ObjectRef {
  string object_id = 1;
  string media_type = 2;
  uint64 size_bytes = 3;
  bytes sha256 = 4;

  string origin_node_id = 5;
  string uri = 6;

  google.protobuf.Timestamp created_at = 7;
  google.protobuf.Timestamp expires_at = 8;
  uint32 ttl_seconds = 9;

  ObjectAvailability availability = 10;
  repeated string replica_node_ids = 11;
  string lease_id = 12;
}
```

## 2. 生命周期

```text
CREATED → LOCAL / REMOTE_AVAILABLE / REPLICATED
       → EXPIRED → GC_PENDING → DELETED
```

## 3. Lease 续租

```proto
message ObjectLeaseRenew {
  string object_id = 1;
  string lease_id = 2;
  uint32 requested_extension_seconds = 3;
}
```

规则：Lease 持有期间 object 不可被 GC；续租上限 = max(3 × 原 TTL, 3600s)。

## 4. 数据流

```proto
message DataOpen { ... }
message DataChunk { ... }
message DataCommit { ... }
message DataAck { ... }
```

## 5. 背压

见 [data-backpressure.md](./data-backpressure.md)。

## 6. 远程文件解析

```text
插件引用 file://node-a/path/to/a.png
  → RemoteFileResolver 判断本地不存在
  → 定位 origin_node_id
  → 向 node-a 发起 DataOpen
  → 流式拉取 DataChunk
  → DataCommit 校验 hash
  → 生成 ObjectRef（含 lease_id）
  → 替换插件输入
  → 调用插件
  → 结束后释放 lease
```

## 7. ObjectStorePlugin

对象存储本身可插拔：

| Plugin | 说明 |
|---|---|
| LocalCASObjectStorePlugin | 本地 CAS（默认） |
| S3ObjectStorePlugin | AWS S3 |
| MinIOObjectStorePlugin | MinIO |
| AzureBlobObjectStorePlugin | Azure Blob |
| GCSObjectStorePlugin | Google Cloud Storage |
