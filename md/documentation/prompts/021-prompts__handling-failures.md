# 处理失败

本页介绍如何使用内置的重试和超时机制来处理 LLM clients 和 prompt executors 的失败。

## 重试功能

使用 LLM providers 时，可能会出现速率限制或临时服务不可用等瞬时错误。
`RetryingLLMClient` 装饰器会为 Kotlin 和 Java 中的任何 LLM client 添加自动重试逻辑。

### 基本用法

用重试能力包装任何现有 client：

KotlinJava

```
// Wrap any client with the retry capability
val client = OpenAILLMClient(apiKey)
val resilientClient = RetryingLLMClient(client)

// Now all operations will automatically retry on transient errors
val response = resilientClient.execute(prompt, OpenAIModels.Chat.GPT4o)
```

```
OpenAILLMClient client = openAIClient(apiKey);
RetryingLLMClient resilientClient = new RetryingLLMClient(client);

// Now all operations will automatically retry on transient errors
List<Message.Response> response = resilientClient.execute(prompt, OpenAIModels.Chat.GPT4o);
```

### 配置重试行为

默认情况下，`RetryingLLMClient` 会为 LLM client 配置最多 3 次重试、1 秒初始延迟，
以及 30 秒最大延迟。
你可以通过传给 `RetryingLLMClient` 的 `RetryConfig` 指定不同的重试配置。
例如：

KotlinJava

```
// Use the predefined configuration
val conservativeClient = RetryingLLMClient(
    delegate = client,
    config = RetryConfig.CONSERVATIVE
)
```

```
OpenAILLMClient client = openAIClient(apiKey);
// Use the predefined configuration
RetryingLLMClient conservativeClient = new RetryingLLMClient(
    client,
    RetryConfig.Companion.getCONSERVATIVE()
);
```

Koog 提供了几种预定义重试配置，可在 Kotlin 中通过 `RetryConfig` 使用，在 Java 中通过 `RetryConfig.Companion` 使用：

| 配置 (Kotlin) | 最大尝试次数 | 初始延迟 | 最大延迟 | 使用场景 |
| --- | --- | --- | --- | --- |
| `RetryConfig.DISABLED` | 1（不重试） | - | - | 开发、测试和调试。 |
| `RetryConfig.CONSERVATIVE` | 3 | 2s | 30s | 可靠性比速度更重要的后台或计划任务。 |
| `RetryConfig.AGGRESSIVE` | 5 | 500ms | 20s | 从瞬时错误中快速恢复比减少 API 调用更重要的关键操作。 |
| `RetryConfig.PRODUCTION` | 3 | 1s | 20s | 一般生产用途。 |

你可以直接使用它们，也可以创建自定义配置：

```
// Or create a custom configuration
val customClient = RetryingLLMClient(
    delegate = client,
    config = RetryConfig(
        maxAttempts = 5,
        initialDelay = 1.seconds,
        maxDelay = 30.seconds,
        backoffMultiplier = 2.0,
        jitterFactor = 0.2
    )
)
```

### 重试错误模式

默认情况下，`RetryingLLMClient` 会识别常见的瞬时错误。
此行为由 [`RetryConfig.retryablePatterns`](https://api.koog.ai/prompt/prompt-executor/prompt-executor-clients/ai.koog.prompt.executor.clients.retry/-retry-config/retryable-patterns.html) 模式控制。
每个模式都由
[`RetryablePattern`](https://api.koog.ai/prompt/prompt-executor/prompt-executor-clients/ai.koog.prompt.executor.clients.retry/-retryable-pattern/index.html)
表示，它会检查失败请求的错误消息，并判断是否应重试。

Koog 提供了可跨所有受支持 LLM providers 使用的预定义重试配置和模式。
你可以保留默认值，也可以针对特定需求自定义它们。

#### 模式类型

你可以使用以下模式类型，并组合任意数量的模式：

- `RetryablePattern.Status`：匹配错误消息中的特定 HTTP 状态码（例如 `429`、`500`、`502` 等）。
- `RetryablePattern.Keyword`：匹配错误消息中的关键字（例如 `rate limit` 或 `request timeout`）。
- `RetryablePattern.Regex`：匹配错误消息中的正则表达式。
- `RetryablePattern.Custom`：使用 lambda 函数匹配自定义逻辑。

如果任意模式返回 `true`，该错误会被视为可重试，LLM client 会重试请求。

#### 默认模式

除非你自定义重试配置，否则默认使用以下模式：

- **HTTP 状态码**：

  - `429`：Rate limit
  - `500`：Internal server error
  - `502`：Bad gateway
  - `503`：Service unavailable
  - `504`：Gateway timeout
  - `529`：Anthropic overloaded
- **错误关键字**：

  - rate limit
  - too many requests
  - request timeout
  - connection timeout
  - read timeout
  - write timeout
  - connection reset by peer
  - connection refused
  - temporarily unavailable
  - service unavailable

这些默认模式在 Koog 中定义为 [`RetryConfig.DEFAULT_PATTERNS`](https://api.koog.ai/prompt/prompt-executor/prompt-executor-clients/ai.koog.prompt.executor.clients.retry/-retry-config/-companion/-d-e-f-a-u-l-t_-p-a-t-t-e-r-n-s.html)。

#### 自定义模式

你可以针对特定需求定义自定义模式：

```
val config = RetryConfig(
    retryablePatterns = listOf(
        RetryablePattern.Status(429),   // Specific status code
        RetryablePattern.Keyword("quota"),  // Keyword in error message
        RetryablePattern.Regex(Regex("ERR_\\d+")),  // Custom regex pattern
        RetryablePattern.Custom { error ->  // Custom logic
            error.contains("temporary") && error.length > 20
        }
    )
)
```

你也可以将自定义模式追加到默认的 `RetryConfig.DEFAULT_PATTERNS`：

```
val config = RetryConfig(
    retryablePatterns = RetryConfig.DEFAULT_PATTERNS + listOf(
        RetryablePattern.Keyword("custom_error")
    )
)
```

### 流式传输与重试

流式操作可以选择重试。此功能默认禁用。

```
val config = RetryConfig(
    maxAttempts = 3
)

val client = RetryingLLMClient(baseClient, config)
val stream = client.executeStreaming(prompt, OpenAIModels.Chat.GPT4o)
```

注意

流式重试仅适用于收到第一个 token 之前发生的连接失败。
一旦流式传输开始，重试逻辑就会被禁用。
如果在流式传输期间发生错误，操作会终止。

### prompt executors 中的重试

使用 prompt executors 时，你可以先用重试机制包装底层 LLM client，然后再创建 executor；Kotlin 和 Java 都支持。
要了解更多 prompt executors 信息，请参见 [Prompt executors](../prompt-executors/)。

KotlinJava

```
// Single provider executor with retry
val resilientClient = RetryingLLMClient(
    OpenAILLMClient(System.getenv("OPENAI_API_KEY")),
    RetryConfig.PRODUCTION
)
val executor = MultiLLMPromptExecutor(resilientClient)

// Multi-provider executor with flexible client configuration
val multiExecutor = MultiLLMPromptExecutor(
    LLMProvider.OpenAI to RetryingLLMClient(
        OpenAILLMClient(System.getenv("OPENAI_API_KEY")),
        RetryConfig.CONSERVATIVE
    ),
    LLMProvider.Anthropic to RetryingLLMClient(
        AnthropicLLMClient(System.getenv("ANTHROPIC_API_KEY")),
        RetryConfig.AGGRESSIVE  
    ),
    // The Bedrock client already has a built-in AWS SDK retry 
    LLMProvider.Bedrock to BedrockLLMClient(
        identityProvider = StaticCredentialsProvider {
            accessKeyId = System.getenv("AWS_ACCESS_KEY_ID")
            secretAccessKey = System.getenv("AWS_SECRET_ACCESS_KEY")
            sessionToken = System.getenv("AWS_SESSION_TOKEN")
        },
    ),
)
```

```
// Single provider executor with retry (Java)
RetryingLLMClient resilientClient = new RetryingLLMClient(
    openAIClient(System.getenv("OPENAI_API_KEY")),
    RetryConfig.Companion.getPRODUCTION()
);

MultiLLMPromptExecutor executor = new MultiLLMPromptExecutor(resilientClient);

// Multi-provider executor with flexible client configuration (Java)
LLMClient openai = new RetryingLLMClient(
    openAIClient(System.getenv("OPENAI_API_KEY")),
    RetryConfig.Companion.getCONSERVATIVE()
);

LLMClient anthropic = new RetryingLLMClient(
    anthropicClient(System.getenv("ANTHROPIC_API_KEY")),
    RetryConfig.Companion.getAGGRESSIVE()
);

Map<LLMProvider, LLMClient> clients = Map.of(
    LLMProvider.OpenAI, openai,
    LLMProvider.Anthropic, anthropic
);

MultiLLMPromptExecutor multiExecutor = new MultiLLMPromptExecutor(clients);
```

## 超时配置

所有 LLM clients 在 Kotlin 和 Java 中都支持超时配置，以防止请求挂起。
你可以在创建 client 时使用
[`ConnectionTimeoutConfig`](https://api.koog.ai/prompt/prompt-executor/prompt-executor-clients/ai.koog.prompt.executor.clients/-connection-timeout-config/index.html) 类指定网络连接的超时值。

`ConnectionTimeoutConfig` 具有以下属性：

| 属性 | 默认值 | 描述 |
| --- | --- | --- |
| `connectTimeoutMillis` | 60 秒 (60,000) | 建立到服务器连接的最长时间。 |
| `requestTimeoutMillis` | 15 分钟 (900,000) | 完成整个请求的最长时间。 |
| `socketTimeoutMillis` | 15 分钟 (900,000) | 在已建立连接上等待数据的最长时间。 |

你可以针对特定需求自定义这些值。例如：

KotlinJava

```
val client = OpenAILLMClient(
    apiKey = apiKey,
    settings = OpenAIClientSettings(
        timeoutConfig = ConnectionTimeoutConfig(
            connectTimeoutMillis = 5000,    // 5 seconds to establish connection
            requestTimeoutMillis = 60000,    // 60 seconds for the entire request
            socketTimeoutMillis = 120000   // 120 seconds for data on the socket
        )
    )
)
```

```
String apiKey = System.getenv("OPENAI_API_KEY");
ConnectionTimeoutConfig timeouts = new ConnectionTimeoutConfig(
    5000L,   // connectTimeoutMillis
    60000L,  // requestTimeoutMillis
    120000L  // socketTimeoutMillis
);
OpenAIClientSettings settings = new OpenAIClientSettings(
    "https://api.openai.com", // baseUrl
    timeouts,
    "v1/chat/completions",    // chatCompletionsPath
    "v1/responses",           // responsesAPIPath
    "v1/embeddings",          // embeddingsPath
    "v1/moderations",         // moderationsPath
    "v1/models"               // modelsPath
);
OpenAILLMClient client = openAIClient(apiKey, settings);
```

提示

对于长时间运行或流式调用，请为 `requestTimeoutMillis` 和 `socketTimeoutMillis` 设置更高的值。

## 错误处理

在生产环境中使用 LLM 时，你需要实现错误处理，包括：

- **Try-catch blocks** 来处理意外错误。
- **Logging errors with context** 以便调试。
- **Fallbacks** 用于关键操作。
- **Monitoring retry patterns** 以识别重复出现的问题。

下面是 Kotlin 和 Java 中错误处理的示例：

KotlinJava

```
val logger = LoggerFactory.getLogger("Example")
val resilientClient = RetryingLLMClient(
    OpenAILLMClient(System.getenv("OPENAI_API_KEY")),
    RetryConfig.PRODUCTION
)
val prompt = prompt("test") { user("Hello") }
val model = OpenAIModels.Chat.GPT4o

fun processResponse(response: Any) { /* implmenentation */ }
fun scheduleRetryLater() { /* implmenentation */ }
fun notifyAdministrator() { /* implmenentation */ }
fun useDefaultResponse() { /* implmenentation */ }

try {
    val response = resilientClient.execute(prompt, model)
    processResponse(response)
} catch (e: Exception) {
    logger.error("LLM operation failed", e)

    when {
        e.message?.contains("rate limit") == true -> {
            // Handle rate limiting specifically
            scheduleRetryLater()
        }
        e.message?.contains("invalid api key") == true -> {
            // Handle authentication errors
            notifyAdministrator()
        }
        else -> {
            // Fall back to an alternative solution
            useDefaultResponse()
        }
    }
}
```

```
Logger logger = LoggerFactory.getLogger("Example");
RetryingLLMClient resilientClient = new RetryingLLMClient(
        openAIClient(System.getenv("OPENAI_API_KEY")),
        RetryConfig.PRODUCTION
);
Prompt prompt = Prompt.builder("test")
        .user("Hello")
        .build();
MultiLLMPromptExecutor promptExecutor = new MultiLLMPromptExecutor(resilientClient);

Consumer<Message.Assistant> processResponse = (resp) -> { /* implementation */ };
Runnable scheduleRetryLater = () -> { /* implementation */ };
Runnable notifyAdministrator = () -> { /* implementation */ };
Runnable useDefaultResponse = () -> { /* implementation */ };

try {
    Message.Assistant response = promptExecutor.execute(prompt, OpenAIModels.Chat.GPT4o);
    processResponse.accept(response);
} catch (Exception e) {
    logger.error("LLM operation failed", e);
    String msg = e.getMessage() == null ? "" : e.getMessage().toLowerCase();
    if (msg.contains("rate limit")) {
        scheduleRetryLater.run();
    } else if (msg.contains("invalid api key")) {
        notifyAdministrator.run();
    } else {
        useDefaultResponse.run();
    }
}
```