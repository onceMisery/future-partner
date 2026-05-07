# semver-policy.md

> 四条独立版本线的分线管理见 [version-lines.md](./version-lines.md)。

FAP 协议 SemVer 策略。

## 1. SemVer 范围

FAP 下以下组件独立使用 SemVer：

| 组件 | 版本字段 | 仓/目录 |
|---|---|---|
| 协议 (protocol) | `protocol_version` | `proto/fap/v1/fap.proto` |
| 插件 API (plugin_api) | `plugin_api.version` | `proto/plugin-api/v1/` |
| Capability | `Capability.version` | 插件声明 |
| Agent 实现 | `agent.version` | Agent Card |
| Runtime 实现 | `runtime.version` | Agent Card |

## 2. 格式

```text
MAJOR.MINOR.PATCH[-PRERELEASE][+BUILD]

示例:
  1.0.0
  1.1.0
  2.0.0-alpha.1
  1.2.3+build.20260506
```

## 3. 变化语义

| 变化 | 兼容性 | 示例 |
|---|---|---|
| PATCH | 必须兼容 | 修 bug、不改字段 |
| MINOR | 向后兼容 | 新增可选字段、新增 enum 值 |
| MAJOR | 可能破坏 | 删除字段、改类型 |
| PRERELEASE | 不稳定 | 按 acceptPrereleases 策略 |
| BUILD | 不影响协议 | 仅构建元数据 |

## 4. 协商规则

### 4.1 协议协商

Agent Card 校验：

```text
client.protocol.major == server.protocol.major
client.protocol.version >= server.minCompatibleVersion
required profile 必须 major 相同
optional profile 可降级关闭
plugin_api major 必须一致
```

Hello 协商：

```text
ClientHello.supported_protocol_range { min, max }
ServerHello.selected_protocol_version ∈ [min, max]
若无交集 → AuthReject(VERSION_INCOMPATIBLE)
```

### 4.2 Prerelease Allowlist

默认拒绝 prerelease。仅允许精确 allowlist。

```json
"protocol": {
  "acceptPrereleases": "allowlist",
  "prereleaseAllowlist": ["1.1.0-rc.3", "2.0.0-beta.1"]
}
```

模糊匹配 `same-major` / `any` 已弃用（见 [version-lines.md](./version-lines.md) §4）。

### 4.3 Plugin API

```text
plugin_api.major 与 host 不一致 → 插件不加载
plugin_api.minor 向后兼容 → 自动加载
plugin_api.patch → 自动加载
插件必须声明 min_core_version
插件加载前执行 schema dry-run
```

## 5. Capability 版本

```text
Capability.version          能力独立 SemVer
Capability.schema_version   input/output schema hash 前 16 char
Capability.min_fap_version  此能力要求的最低协议版本

客户端:
  schema_version 未变 → 可复用已缓存的 schema
  schema_version 变了 → 必须重新 Fetch
```

## 6. 演进约束

### MUST

```text
PATCH 变化:
  不改语义
  不增字段
  不改 enum 值

MINOR 变化:
  新增字段必须可选（proto3 默认）
  新增 enum 值必须保留 _UNSPECIFIED=0
  旧客户端忽略未知字段后仍能工作

MAJOR 变化:
  可删除字段（须先 reserved）
  可改字段类型
  必须提供迁移指南
```

### MUST NOT

```text
- 在 MINOR 里删除已发布字段
- 在 MINOR 里改字段类型
- 改已发布字段的 tag 号
- 发布有相同 tag 但不同类型的两个字段
```

## 7. 发布流程

```text
1. RFC 期（≥ 2 周）
2. 代码/文档/CHANGELOG 就绪
3. buf breaking CI 绿
4. 所有 SDK 通过 conformance matrix
5. 签发 git tag
6. 发布 proto descriptor 到镜像仓（可选）
7. 老版本进入 deprecation 窗（≥ 2 个 MINOR）
```

## 8. 废弃

```proto
message Foo {
  string old_field = 1 [deprecated = true];
  string new_field = 2;
}
```

废弃字段：

```text
至少保留 2 个 MINOR 版本
CHANGELOG 明确标记
插件/SDK 编译告警
在下个 MAJOR 中 reserved 后删除
```

## 9. 私有扩展字段号

```text
官方字段号: 1..9999
组织私有扩展: 10000+
私有扩展不进官方 spec
私有字段使用 extensions map<string,string>（反向域名 key）优于私有字段号
```
