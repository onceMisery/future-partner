# version-lines.md

四条版本线分开管理。

## 1. 四条版本线

| 版本线 | 定义物 | 仓/目录 | 变更影响 |
|---|---|---|---|
| Protocol | Envelope / Header / Session 行为 | `proto/fap/v1/fap.proto` | 所有实现 |
| plugin_api | Plugin 接口契约 | `proto/plugin-api/v1/` | 所有插件 |
| Capability Schema | 单个 capability 的 input/output schema | 插件自身 | 该能力调用方 |
| Agent Card Schema | Agent Card JSON schema | `schema/agent-card.schema.json` | 发现方 |

四条线**独立发版**，互不强绑定。

## 2. 各线语义

### 2.1 Protocol

```text
MAJOR: Envelope 破坏兼容、Header 破坏兼容、Session 状态机变化、签名规范变化
MINOR: 新增可选消息、新增 Priority 值、新增 Problem 类型
PATCH: bug 修复、文档澄清
```

### 2.2 plugin_api

```text
MAJOR: Plugin trait 破坏兼容、host→plugin 回调签名变化
MINOR: 新增可选方法、新增 context 字段
PATCH: bug 修复

独立演进: 协议 v1.2 可能对应 plugin_api v2.0
```

### 2.3 Capability Schema

```text
MAJOR: 输入字段改名/删除、输出结构破坏兼容、风险等级升高
MINOR: 新增可选输入字段、新增输出字段
PATCH: 文档澄清、约束放宽

支持多版本并存（见 plugin-lifecycle.md）
capability_id 固定，不同 version 视为不同路由目标
```

### 2.4 Agent Card Schema

```text
MAJOR: 字段改名、字段类型变更
MINOR: 新增可选字段
PATCH: 文档澄清

发现方按 Agent Card Schema 版本解析；未知字段忽略
```

## 3. Hello 双向版本区间

```proto
message ClientHello {
  VersionRange supported_protocol_range = 4;
  VersionRange supported_plugin_api_range = ?;
  repeated ProfileVersionRange supported_profiles = 6;
  // ...
}

message ServerHello {
  string selected_protocol_version = 3;
  string selected_plugin_api_version = ?;
  repeated ProfileVersion selected_profiles = 4;
  // ...
}

message ProfileVersionRange {
  string id = 1;
  VersionRange range = 2;
}
```

### 3.1 双向区间意义

```text
Client 说: 我支持 protocol 1.0.0 ~ 1.2.0
Server 说: 我支持 protocol 1.1.0 ~ 1.3.0
协商结果: 交集 1.1.0 ~ 1.2.0，server 选 1.2.0（取最高）

若无交集:
  AuthReject {
    code: VERSION_INCOMPATIBLE,
    required_range: { min: "1.1.0", max: "1.3.0" },
    current_range: { min: "1.0.0", max: "1.2.0" }
  }
```

### 3.2 选择策略

```text
服务端默认取交集最高版本（最新）
可配置为"取交集最低"（更保守）
或"取 client 首选版本"（compat preference）
```

## 4. Prerelease Allowlist

```text
默认: 拒绝 prerelease（含 -alpha / -beta / -rc）

Agent Card 配置:
  "acceptPrereleases": "allowlist",
  "prereleaseAllowlist": [
    "1.1.0-rc.3",
    "2.0.0-beta.1"
  ]

语义:
  只接受精确匹配的 prerelease 版本
  若版本不在 allowlist → 视作不支持
  不使用 "same-major" / "any" 模糊匹配（它们已弃用）

迁移:
  旧 acceptPrereleases="same-major" 视作不安全，Phase 0 之后必须移除
  旧 acceptPrereleases="any" 视作等同 "none"（拒绝）
```

## 5. 版本线间的约束

```text
Capability 可声明:
  min_fap_version: 要求的最低协议版本
  min_plugin_api_version: 要求的最低插件 API 版本
  
Plugin 可声明:
  min_core_version: 要求的最低协议版本
  plugin_api_version: 插件实现的 plugin_api 版本
  
Host 加载插件:
  plugin_api_version MAJOR 必须匹配
  min_core_version ≤ 当前协议版本
  否则拒绝加载
```

## 6. 演进 CI

```text
每个版本线独立仓库或目录:
  proto/fap/            → buf breaking 独立
  proto/plugin-api/     → buf breaking 独立
  schema/agent-card/    → JSON Schema diff 独立
  capability schemas    → 每插件自管

CHANGELOG 分四套
发版 tag 分四套: fap-protocol-v1.0.0 / plugin-api-v1.0.0 / ...
```

## 7. 兼容矩阵

发版时发布兼容矩阵：

```text
Protocol v1.2.0 兼容:
  plugin_api v1.x 全部
  Agent Card schema v1.x 全部
  
Plugin API v2.0.0 兼容:
  Protocol v1.1.0 及以上
  
不保证:
  Capability Schema 兼容性由插件自己声明
```

## 8. 退役

```text
MAJOR 退役:
  公告 ≥ 6 个月
  仍需保留 Bridge 能力（与新版兼容翻译）
  退役后 Problem: VERSION_RETIRED

MINOR 废弃:
  下个 MAJOR 可物理移除
  保留至少 2 个 MINOR 周期
```
