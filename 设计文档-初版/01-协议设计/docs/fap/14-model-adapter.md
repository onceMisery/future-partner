# 14 - 模型侧适配

## 1. 两种格式

**JSON Tool Schema**

```json
{
  "tool": "code.review",
  "input": {
    "path": "src/main.rs"
  }
}
```

**FAP Text Tool Syntax**

```text
<<<[FAP_TOOL_REQUEST]>>>
capability:「始」code.review「末」
path:「始」src/main.rs「末」
mode:「始」strict「末」
<<<[END_FAP_TOOL_REQUEST]>>>
```

## 2. 转换流程

```text
Model Output
  → Tool Syntax Parser
  → Schema Validator
  → InvokeRequest Builder
  → Core Security Check
  → Execution Scheduler
```

## 3. 为什么保留文本语法

```text
兼容不支持 function calling 的模型
降低模型生成严格 JSON 的负担
提升容错性
适合复杂多工具自然语言编排
```

底层 wire protocol 始终是 Protobuf。

## 4. 模型适配器插件

| Plugin | 说明 |
|---|---|
| OpenAIToolAdapterPlugin | OpenAI function calling |
| AnthropicToolAdapterPlugin | Anthropic tool use |
| GeminiToolAdapterPlugin | Google Gemini function calling |
| TextToolAdapterPlugin | FAP Text Tool Syntax |
