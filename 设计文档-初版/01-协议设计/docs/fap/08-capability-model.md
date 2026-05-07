# 08 - 能力模型

## 1. Capability 定义

```proto
message Capability {
  string id = 1;
  string name = 2;
  string version = 3;              // 能力 SemVer
  string plugin_id = 4;

  RiskLevel risk_level = 5;

  repeated string input_modes = 6;
  repeated string output_modes = 7;

  bool supports_streaming = 8;
  bool supports_batch = 9;
  bool supports_resume = 10;
  bool requires_mandate = 11;

  google.protobuf.Struct input_schema = 20;
  google.protobuf.Struct output_schema = 21;

  string schema_version = 30;      // schema hash 前 16 char
  string min_fap_version = 31;     // 此能力要求的最低协议版本
}
```

## 2. 风险等级

| 等级 | 示例 | 策略 |
|---|---|---|
| low | 时间、天气、只读文档 | JWT scope 即可 |
| medium | 代码审查、搜索、摘要 | JWT scope + 可选审计 |
| high | 写文件、执行命令、发邮件 | Mandate + Receipt |
| critical | 删除数据、付款、生产发布 | Mandate + 人工审批 + 强审计 |

## 3. Capability Registry

```text
插件加载
  → 读取 manifest
  → 提取 capability
  → 校验 schema
  → 写入 SQLite
  → 建立能力向量索引（ContextPlugin）
  → 更新 Agent Card
```

## 4. Invoke

```proto
message InvokeRequest {
  string invocation_id = 1;
  string capability_id = 2;
  google.protobuf.Struct input = 3;
  repeated ObjectRef input_objects = 4;
  bool stream_result = 5;
  uint32 timeout_ms = 6;
  RiskLevel declared_risk_level = 7;
  string mandate_id = 8;
  repeated string depends_on = 30;   // DAG
  map<string, string> options = 100;
}
```

## 5. BatchInvoke 与 DAG

```proto
enum BatchOrderPolicy {
  BATCH_ORDER_UNSPECIFIED = 0;
  BATCH_ORDER_UNORDERED = 1;
  BATCH_ORDER_RESULT_PRESERVED = 2;
  BATCH_ORDER_SEQUENTIAL = 3;
  BATCH_ORDER_DAG = 4;
}
```

DAG 模式：调度器拓扑排序 → 层内并行 → 层间等待。

## 6. CapabilityRouterPlugin

路由策略可插拔：

| Plugin | 说明 |
|---|---|
| LocalCapabilityRouterPlugin | 本地路由（默认） |
| SemanticCapabilityRouterPlugin | 语义召回 + 重排 |
| LoadBalancingRouterPlugin | 负载均衡 |
| LocalityAwareRouterPlugin | 按 locality 标签路由 |
