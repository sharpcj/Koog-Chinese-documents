# 提示缓存控制

提示缓存控制允许你指示受支持的 LLM 提供方在服务端存储提示的一部分，这样后续共享相同前缀的请求就可以从缓存中提供，而不必重新处理 token。
这可以降低重复性工作负载的延迟和成本，例如多轮对话、大型系统提示或固定工具定义。

提示缓存与响应缓存

提示缓存控制是一个**提供方侧**功能：提供方存储提示前缀，而不是响应。这不同于 [`CachedPromptExecutor`](../../llm-response-caching/)，后者会在本地存储完整的 LLM 响应，使相同提示完全跳过网络调用。

Koog 支持 **Anthropic** 和 **Amazon Bedrock** 的提示缓存控制。

## Anthropic

Anthropic 支持两种互补的提示缓存方式。

### 自动缓存（请求级别）

在 [`AnthropicParams`](../../../llm-parameters/) 上设置 `cacheControl` 属性，并将其传递给提示。
Anthropic 会自动将缓存断点放在请求中最后一个可缓存块处，无需你标注单个消息。
这是多轮对话的推荐方法。

KotlinJava

```
// Enable automatic caching with the default 5-minute TTL
val params = AnthropicParams(cacheControl = AnthropicCacheControl.Default)

val prompt = prompt("assistant", params = params) {
    system("You are a helpful assistant with a very long system prompt...")
    user("What can you help me with?")
}

val response = client.execute(prompt, AnthropicModels.Sonnet_4)
println(response)
```

```
// Enable automatic caching with the default 5-minute TTL
AnthropicParams params = new AnthropicParams(
    null, null, null, null, null, null, null, null,
    null, null, null, null, null, null, null,
    AnthropicCacheControl.Default.INSTANCE
);

Prompt prompt = Prompt.builder("assistant")
    .system("You are a helpful assistant with a very long system prompt...")
    .user("What can you help me with?")
    .build()
    .withParams(params);
```

### 手动缓存（块级别）

将 `cacheControl` 参数附加到单个消息或工具定义上，以便将缓存断点放在特定位置。直到并包括被标注块在内的所有内容都可以缓存。

#### System messages

KotlinJava

```
val prompt = prompt("assistant") {
    // Cache the system prompt for 1 hour
    system("You are a knowledgeable assistant...", AnthropicCacheControl.OneHour)
    user("Summarize the latest AI research.")
}

val response = client.execute(prompt, AnthropicModels.Sonnet_4)
println(response)
```

```
Prompt prompt = Prompt.builder("assistant")
    // Cache the system prompt for 1 hour
    .system("You are a knowledgeable assistant...", AnthropicCacheControl.OneHour.INSTANCE)
    .user("Summarize the latest AI research.")
    .build();
```

#### User and assistant messages

KotlinJava

```
val prompt = prompt("conversation") {
    system("You are a helpful assistant.")
    // Cache after a large user message (e.g. document content)
    user(listOf(MessagePart.Text("Here is a long document: ...", cacheControl = AnthropicCacheControl.Default)))
    assistant(listOf(MessagePart.Text("I have read the document.")))
    user("Summarize it.")
}

val response = client.execute(prompt, AnthropicModels.Sonnet_4)
println(response)
```

```
Prompt prompt = Prompt.builder("conversation")
    .system("You are a helpful assistant.")
    // Cache after a large user message (e.g. document content)
    .user(List.of(new ContentPart.Text("Here is a long document: ...")), AnthropicCacheControl.Default.INSTANCE)
    .assistant("I have read the document.", AnthropicCacheControl.Default.INSTANCE)
    .user("Summarize it.")
    .build();
```

#### Tool definitions

当工具列表在许多请求中保持固定时，缓存最后一个工具定义意味着所有工具 schema 都会一起缓存。

KotlinJava

```
val searchTool = ToolDescriptor(
    name = "web_search",
    description = "Search the web for information.",
    requiredParameters = listOf(
        ToolParameterDescriptor("query", "Search query", ToolParameterType.String)
    ),
    // Cache all tool definitions up to and including this one
    cacheControl = AnthropicCacheControl.Default
)
```

```
ToolDescriptor searchTool = new ToolDescriptor(
    "web_search",
    "Search the web for information.",
    List.of(
        new ToolParameterDescriptor("query", "Search query", ToolParameterType.String.INSTANCE)
    ),
    Collections.emptyList(),
    // Cache all tool definitions up to and including this one
    AnthropicCacheControl.Default.INSTANCE
);
```

### 缓存 TTL 选项

| Option | TTL | Price multiplier |
| --- | --- | --- |
| `AnthropicCacheControl.Default` | 5 minutes | 1.25× base input price |
| `AnthropicCacheControl.OneHour` | 1 hour | 2× base input price |

缓存写入的收费高于常规输入 token，但缓存读取更便宜。
请参阅 [Anthropic prompt caching docs](https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching) 获取当前价格。

### 监控缓存使用情况

Anthropic 会在响应用量中报告缓存统计信息。这些信息可通过原始 API 响应访问，也可以通过 tracing 或 logging 功能观察。

| Field | Meaning |
| --- | --- |
| `cacheReadInputTokens` | 从现有缓存条目读取的 token |
| `cacheCreationInputTokens` | 写入新缓存条目的 token |

### 组合自动缓存和块级缓存

两种模式可以同时使用。块级 `cacheControl` 标记让你精细控制断点位置，而 `AnthropicParams` 中的请求级 `cacheControl` 会自动处理对话尾部。

KotlinJava

```
// Block-level: pin the system prompt in the 1-hour cache tier
// Automatic: let Anthropic manage breakpoints for the conversation tail
val params = AnthropicParams(cacheControl = AnthropicCacheControl.Default)

val prompt = prompt("combined", params = params) {
    system("You are a helpful assistant...", AnthropicCacheControl.OneHour)
    user("Hello!")
}
```

```
// Block-level: pin the system prompt in the 1-hour cache tier
// Automatic: let Anthropic manage breakpoints for the conversation tail
AnthropicParams params = new AnthropicParams(
    AnthropicCacheControl.Default.INSTANCE
);

Prompt prompt = Prompt.builder("combined")
    .system("You are a helpful assistant...", AnthropicCacheControl.OneHour.INSTANCE)
    .user("Hello!")
    .build()
    .withParams(params);
```

---

## Amazon Bedrock

Amazon Bedrock 通过 Converse API 使用块级缓存模型。
当在消息或工具上设置 `cacheControl` 时，Bedrock 会在被标注元素之后立即插入一个 `CachePoint` 块。

Note

Bedrock 提示缓存是 JVM-only 功能，因为 Bedrock 客户端本身仅支持 JVM。

### System messages

KotlinJava

```
val prompt = prompt("assistant") {
    // Cache the system prompt using the default TTL
    system("You are a knowledgeable assistant...", BedrockCacheControl.Default)
    user("What is prompt caching?")
}

val response = client.execute(prompt, BedrockModels.AnthropicClaude4Sonnet)
println(response)
```

```
Prompt prompt = Prompt.builder("assistant")
    // Cache the system prompt using the default TTL
    .system("You are a knowledgeable assistant...", BedrockCacheControl.Default.INSTANCE)
    .user("What is prompt caching?")
    .build();
```

### User and assistant messages

KotlinJava

```
val prompt = prompt("conversation") {
    system("You are a helpful assistant.")
    // Cache after the large context message
    user("Here is the document: ...", BedrockCacheControl.FiveMinutes)
    assistant(listOf(MessagePart.Text("I have read the document.")))
    user("Summarize it.")
}

val response = client.execute(prompt, BedrockModels.AnthropicClaude4Sonnet)
println(response)
```

```
Prompt prompt = Prompt.builder("conversation")
    .system("You are a helpful assistant.")
    // Cache after the large context message
    .user("Here is the document: ...", BedrockCacheControl.FiveMinutes.INSTANCE)
    .assistant("I have read the document.", BedrockCacheControl.Default.INSTANCE)
    .user("Summarize it.")
    .build();
```

### Tool definitions

KotlinJava

```
val searchTool = ToolDescriptor(
    name = "web_search",
    description = "Search the web for information.",
    requiredParameters = listOf(
        ToolParameterDescriptor("query", "Search query", ToolParameterType.String)
    ),
    // Cache all tool definitions up to and including this one
    cacheControl = BedrockCacheControl.Default
)
```

```
ToolDescriptor searchTool = new ToolDescriptor(
    "web_search",
    "Search the web for information.",
    List.of(
        new ToolParameterDescriptor("query", "Search query", ToolParameterType.String.INSTANCE)
    ),
    Collections.emptyList(),
    // Cache all tool definitions up to and including this one
    BedrockCacheControl.Default.INSTANCE
);
```

### 缓存 TTL 选项

| Option | TTL |
| --- | --- |
| `BedrockCacheControl.Default` | Provider default (no explicit TTL sent) |
| `BedrockCacheControl.FiveMinutes` | 5 minutes |
| `BedrockCacheControl.OneHour` | 1 hour |

请参阅 [Amazon Bedrock prompt caching docs](https://docs.aws.amazon.com/bedrock/latest/userguide/prompt-caching.html)，了解受支持的模型和价格。

---

## 选择缓存策略

| Situation | Recommended approach |
| --- | --- |
| 包含大型固定系统提示的多轮聊天 | Anthropic 自动缓存，或在 Bedrock 中对 system 使用块级缓存 |
| 在多个请求中复用的稳定工具定义 | 在最后一个工具定义上使用块级 `cacheControl` |
| 作为用户上下文传入的长文档 | 在用户消息上使用块级 `cacheControl` |
| 任意多轮对话（Anthropic） | 通过 `AnthropicParams.cacheControl` 使用自动缓存 |
| 需要 1 小时缓存保留时间 | `AnthropicCacheControl.OneHour` / `BedrockCacheControl.OneHour` |