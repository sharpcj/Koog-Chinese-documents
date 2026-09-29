# HTTP clients

Koog 中的每个 LLM client 都需要一个 [`KoogHttpClient`](api:http-client-core::ai.koog.http.client.KoogHttpClient) —— 这是框架用于与 providers 通信的抽象 HTTP 契约。你在构造时传入它。

你可以自己构建该 `KoogHttpClient`，但这需要不少工作：每个 provider 都有自己的 base URL、auth header 形式、content-type 和 SSE 约定。针对每个 provider 正确处理所有这些细节，正是 [`KoogHttpClient.Factory`](api:http-client-core::ai.koog.http.client.KoogHttpClient.Factory) 要替你省去的工作。你传入一个 `Factory`，provider client 会用适合其 API 的参数调用 `Factory.create(...)`。

开箱即用提供四种后端 factory —— Ktor、JDK `HttpClient`、OkHttp 和 Spring 的 `WebClient` —— 你也可以实现自己的 factory。

## 工作原理

一个 factory 可用于任何 provider：选择一次后端，并在各个 clients 中复用它。

KotlinJava

```
fun main() {
    val factory = KtorKoogHttpClient.Factory()

    val openai = OpenAILLMClient(
        apiKey = System.getenv("OPENAI_API_KEY"),
        settings = OpenAIClientSettings(),
        httpClientFactory = factory,
    )

    val anthropic = AnthropicLLMClient(
        apiKey = System.getenv("ANTHROPIC_API_KEY"),
        settings = AnthropicClientSettings(),
        httpClientFactory = factory,
    )
}
```

```
import ai.koog.http.client.ktor.KtorKoogHttpClient;
import ai.koog.prompt.executor.clients.anthropic.AnthropicClientSettings;
import ai.koog.prompt.executor.clients.anthropic.AnthropicLLMClient;
import ai.koog.prompt.executor.clients.openai.OpenAIClientSettings;
import ai.koog.prompt.executor.clients.openai.OpenAILLMClient;

KtorKoogHttpClient.Factory factory = new KtorKoogHttpClient.Factory();

OpenAILLMClient openai = new OpenAILLMClient(
    System.getenv("OPENAI_API_KEY"),
    new OpenAIClientSettings(),
    factory
);

AnthropicLLMClient anthropic = new AnthropicLLMClient(
    System.getenv("ANTHROPIC_API_KEY"),
    new AnthropicClientSettings(),
    factory
);
```

## 支持的 HTTP client 类型

| 模块 | 说明 |
| --- | --- |
| [`http-client-ktor`](api:http-client-ktor::) | 唯一可从非 JVM target 使用的后端。 |
| [`http-client-java`](api:http-client-java::) | 包装 JDK 11+ 的 `java.net.http.HttpClient`。 |
| [`http-client-okhttp`](api:http-client-okhttp::) | 基于 OkHttp。Android 友好。 |
| [`http-client-spring-webclient`](api:http-client-spring-webclient::) | 基于 Spring `WebClient`。 |

## 便捷 API 和 factory 自动发现

在 JVM 和 Android 上，你可以构造每个 LLM client，而无需显式传入 factory。

在幕后，[`HttpClientFactoryResolver`](api:http-client-core::ai.koog.http.client.HttpClientFactoryResolver) 使用 `java.util.ServiceLoader` 从运行时 classpath 解析 `KoogHttpClient.Factory`：

- 每个后端模块都提供一个 `ServiceLoader` 注册。
- 只有当运行时 classpath 中恰好可见一个 factory 时，解析才会成功。
- `prompt-executor-llms-all` 将 `http-client-ktor` 声明为 `runtimeOnly` 依赖，因此默认会得到 Ktor，而不会在编译期暴露该模块。
- `simple<Provider>Executor(apiKey)` 和 `PromptExecutorBuilder.<provider>(apiKey)` 使用相同的解析路径。

KotlinJava

```
fun main() {
    val apiKey = System.getenv("OPENAI_API_KEY")

    val client = OpenAILLMClient(apiKey)
    val executor = simpleOpenAIExecutor(apiKey)
}
```

```
import static ai.koog.prompt.executor.clients.openai.OpenAIClientFactory.openAIClient;
import static ai.koog.prompt.executor.llms.all.SimplePromptExecutors.simpleOpenAIExecutor;

String apiKey = System.getenv("OPENAI_API_KEY");

OpenAILLMClient client = openAIClient(apiKey);
PromptExecutor executor = simpleOpenAIExecutor(apiKey);
```

目前 KMP 不支持自动发现，因此 JVM 之外也无法使用这些便捷方法。从 `commonMain` 中使用时，请显式传入 `Factory`。

### 自动发现注意事项

- **运行时 classpath 上没有后端** → 首次解析时抛出 `IllegalStateException`。请将后端模块添加到运行时 classpath，或显式传入 `Factory`。
- **两个或更多后端** → 同样抛出异常；消息会列出找到的 providers。使用 Gradle 排除到只剩一个（在有问题的依赖上使用 `exclude(module = "http-client-ktor")`），或在调用处显式传入 `Factory`。

## 自定义后端

任何实现 `KoogHttpClient.Factory` 的类都可以使用。若要使其在 JVM 上可自动发现，请将其注册为 `ServiceLoader` provider：

```
src/main/resources/META-INF/services/ai.koog.http.client.KoogHttpClient$Factory
```

该文件包含一行内容：你的 factory 类的完全限定名。字面量 `$`（嵌套 `Factory` 类的分隔符）是正确的 —— 文件名是 `KoogHttpClient$Factory`，不是 `KoogHttpClient.Factory`。

如果你不想使用自动发现，请跳过注册，并在所有位置显式传入你的 factory。