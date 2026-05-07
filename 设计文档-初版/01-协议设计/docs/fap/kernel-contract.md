# kernel-contract.md

FAP-1 Kernel Contract：定义哪些组件**必须**留在 Core，哪些**可以**作为 Plugin。

## 1. 原则

可插拔架构 ≠ 一切可替换。若 Envelope 解码、Session 状态机、Policy 决策、Replay/Idempotency、Receipt 链、Plugin 权限裁决可被插件替换，整个协议的安全模型将被架空。

```text
插件可以提供:
  算法（签名算法、哈希算法、向量索引、压缩）
  外部适配（具体 Transport / Storage / Auth / Discovery / Bridge 实现）

插件不能提供:
  安全判定的顺序
  安全判定的规则
  会话状态的转移
  Receipt 链的生成与验证
  插件自身的权限裁决
```

## 2. Kernel 必留组件

| 组件 | 必留理由 |
|---|---|
| Envelope Decoder | 消息边界与 tag 解析是所有安全校验的起点 |
| Session State Machine | 状态转移规则不能被插件改写 |
| Policy Engine | 决策顺序固定：Auth → Mandate → Capability → Risk |
| Replay/Idempotency Guard | 插件不得绕过，否则可重放攻击 |
| Receipt Chain Generator | hash chain 必须由 Core 原子生成 |
| Plugin Permission Arbiter | 插件自身不能判定自己的权限 |
| Header Validator | 版本、时间、hash、signature 字段的校验顺序固定 |
| Priority Queue | Control > Audit > Data 优先级 |

## 3. Plugin 提供点

| 扩展点 | Plugin | 作用域 |
|---|---|---|
| 传输 | TransportPlugin | 只负责字节收发 |
| 认证 | AuthPlugin | 只负责凭证验证，不判定是否允许执行 |
| 签名算法 | SignaturePlugin | 只提供 sign/verify 字节函数 |
| 密钥解析 | KeyResolverPlugin | 只提供 kid → pubkey 查询 |
| 撤销 | RevocationPlugin | 只提供 is_revoked 查询 |
| 存储 | StoragePlugin | 只负责持久化，不改表结构语义 |
| 向量索引 | VectorIndexPlugin | 只提供 ANN 查询接口 |
| 对象存储 | ObjectStorePlugin | 只负责 blob 读写 |
| 发现 | DiscoveryPlugin | 只负责 Agent Card 获取 |
| 审计 Sink | AuditSinkPlugin | 只负责 Receipt/checkpoint 持久化与外送 |
| 桥接 | BridgePlugin | 只负责协议翻译，必须经过 Core |
| 业务能力 | CapabilityPlugin | 执行业务逻辑，不得绕过 Core |
| 上下文 | ContextPlugin | 只提供 projection/folding/retrieval |
| 路由 | CapabilityRouterPlugin | 只提供"候选节点"建议，不得强制路由 |
| 节点健康 | NodeHealthPlugin | 只提供健康评分，不做断路决策 |
| 节点健康决策 | 留在 Core（QUARANTINED 状态由 Core 管） |
| 发现协商 | DiscoveryNegotiator（Core） |

## 4. 判定顺序不可变

```text
固定顺序（Core 内硬编码）:
  1. Transport TLS
  2. Envelope Decode
  3. Header Validation
  4. Replay Cache
  5. Session Check
  6. AuthN（可调 AuthPlugin，但决策是 Core）
  7. AuthZ（可调 PolicyPlugin，但链序是 Core）
  8. Mandate Check
  9. Capability Policy
  10. Risk Policy
  11. Plugin Sandbox
  12. Receipt Generation
```

任何插件**只能在其中某一步被询问**，不能跳过或重排。

## 5. 插件无法访问的资源

```text
其他 session 的状态
Replay Cache 的写入权
Receipt Store 的直接写入
Hash Chain Tip 的直接修改
Mandate 撤销表的修改
其他插件的私有状态
Core 配置的修改
```

## 6. Plugin Permission Arbiter

```text
插件发起任何 host API 调用时:
  → Arbiter 查 manifest.permissions
  → 决策 allow/deny
  → 记录审计

插件不得:
  绕过 Arbiter
  自行声明 "我需要这个权限"
  修改自己的 manifest
```

## 7. 反面教材（禁止模式）

```text
错误: SecurityPlugin 决定是否需要 mandate
对: SecurityPlugin 只校验 mandate 签名；是否需要 mandate 由 Capability.risk_level + Core Policy 决定

错误: RoutingPlugin 强制把请求发到节点 X
对: RoutingPlugin 返回"候选节点 [X,Y,Z] + 评分"，Core 做最终选择

错误: AuditSinkPlugin 决定哪些 Receipt 要写 transparency log
对: Core 按策略决定，AuditSinkPlugin 只负责写入

错误: CapabilityPlugin 自己生成 Receipt
对: CapabilityPlugin 返回 result，Core 生成 Receipt

错误: BridgePlugin 把 MCP 请求直接转发给 CapabilityPlugin
对: BridgePlugin 构造 FAP Envelope 交给 Core，Core 走完整链路
```

## 8. Kernel Contract 测试

```text
MUST 通过的负例测试:
  插件试图访问其他 session → 拒绝
  插件试图修改 Receipt → 拒绝
  插件试图跳过 mandate 校验 → 拒绝
  插件试图伪造 signature → 拒绝
  插件试图写 Replay Cache → 拒绝
  插件试图重排 Policy 顺序 → 拒绝
  插件试图直接写 hash_chain_tip → 拒绝
```

见 20-testing-conformance.md 的"恶意插件负例"门禁。
