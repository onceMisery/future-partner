# schema-evolution.md

Proto Schema 演进规则。

## 1. 目标

确保 `fap.proto` 与 `plugin_api.proto` 在长期演进中保持互操作，且所有扩展点通过插件接口而非硬编码字段扩展。

## 2. 契约 vs 实现

```text
契约（协议规范）:
  proto/fap/v1/fap.proto           主协议
  proto/plugin-api/v1/*.proto      插件 API
  proto/fap/v1/extensions/*.proto  EXT 插件扩展点（可选）

实现（可替换）:
  任何具体插件（TransportPlugin/AuthPlugin/...）的 proto
  仅在插件自身仓内维护，不进 fap.proto
```

主 proto 只定义消息框架；具体扩展能力（如潜空间消息格式、DID method 参数）由 ExtensionPlugin 自带 proto，通过 Envelope.extensions 或 `google.protobuf.Any` 承载。

## 3. 硬约束

### MUST

```text
- 永远不删除已发布字段，只能 reserved
- 永远不改字段类型
- 永远不改字段 tag
- 新增字段必须可选（proto3 默认）
- enum 保留 _UNSPECIFIED = 0
- 新增 enum 值不视为破坏兼容
- 跨插件共享的 message 放 proto/fap/v1/shared/
```

### MUST NOT

```text
- 在 fap.proto 里硬编码某个具体 transport/auth/storage 的字段
- 把具体插件的实现细节（Rustls 版本、SQLite 表名）写进 spec
- 让 Core 直接依赖某个扩展的 proto
```

## 4. 扩展点定义

FAP 通过三种方式支持扩展，均为"可插拔"：

### 4.1 Envelope.extensions (map<string,string>)

轻量扩展，只传字符串，key 必须反向域名：

```text
org.future-agent.trace.debug_level = "verbose"
com.example.routing.shard_id = "7"
```

### 4.2 Envelope body 的预留段 + google.protobuf.Any

重型扩展，注册 tag：

```proto
message Envelope {
  oneof body {
    // ...
    google.protobuf.Any extension_body = 900;
  }
}
```

由 ExtensionPlugin 声明承载的 Any 类型 URL（如 `type.googleapis.com/fap.ext.latent.v1.LatentMessage`）。

### 4.3 Profile + feature flag

编译期剔除的重型扩展（如潜空间、CRDT），进 FAP-EXT，独立 proto 包。

## 5. 废弃流程

```proto
message Foo {
  string old_field = 1 [deprecated = true];
  string new_field = 2;
}
```

```text
第 N 版:   标 deprecated，文档公告
第 N+1 版: 继续保留，插件 SDK 发出编译告警
第 N+2 版: 可进入 reserved
MAJOR 版:  可物理删除
```

## 6. CI

```text
buf breaking --against '.git#branch=main' proto/
  
拒绝的变更:
  删除字段
  改 tag
  改类型
  改 enum 值
  
允许但警告:
  新增字段
  新增 enum 值
  新 reserved
```

## 7. 私有字段号

```text
官方: 1..9999
组织私有: 10000+

建议:
  优先走 Envelope.extensions map
  其次走 Any
  最后才考虑私有字段号
  私有字段号不在跨组织互通
```

## 8. 插件 API 演进

`plugin_api.proto` 独立 SemVer、独立发版。host ↔ plugin 双向调用都纳入同一 proto，避免"host 端只升级 outbound 接口但 plugin 端只升级 inbound 接口"的割裂。
