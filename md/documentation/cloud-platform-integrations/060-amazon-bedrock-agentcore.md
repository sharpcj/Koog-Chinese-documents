# Amazon Bedrock AgentCore

Beta

此功能属于 beta 模块（`1.3.0-beta`）。API 可能会在未来的版本中发生变化。
有关详细信息，请参阅[模块版本控制](../module-versioning/)。

Koog 提供了用于通过 Amazon Bedrock AgentCore 服务运行 agent 的集成。

## Amazon Bedrock AgentCore Runtime

`koog-bedrock-agentcore-runtime` 模块提供了一个 Ktor 路由安装器，可通过
[Amazon Bedrock AgentCore Runtime](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/runtime.html) HTTP
契约公开 Koog agent。它会在配置该安装器所在的 Ktor 路由下安装以下端点：

- `POST /invocations` 处理 agent 请求。
- `GET /ping` 报告 agent 健康状况和后台活动。

该模块支持类型化 JSON 处理器，以及文本、二进制、multipart 和流式负载。调用处理器
在 Ktor `RoutingContext` 中运行，因此当安装了 `koog-ktor` 插件时，它们可以使用诸如 `aiAgent()` 之类的 Koog 路由扩展。

### 添加依赖项

将 AgentCore Runtime 模块添加到你的 Gradle 构建中：
```
dependencies {
    implementation("ai.koog:koog-bedrock-agentcore-runtime:$koogVersion")
}
```
该模块要求 JVM 17 或更高版本、Kotlin 2.x 和 Ktor 3.x。

### 安装运行时路由

以下示例安装 Koog 和 Ktor 内容协商，然后暴露一个类型化的 JSON 调用处理器：
```
import ai.koog.agentcore.runtime.agentCoreRuntime
import ai.koog.agentcore.runtime.handle
import ai.koog.ktor.Koog
import ai.koog.ktor.aiAgent
import ai.koog.ktor.llm
import ai.koog.prompt.executor.clients.bedrock.BedrockModels
import io.ktor.serialization.kotlinx.json.json
import io.ktor.server.application.Application
import io.ktor.server.application.install
import io.ktor.server.plugins.contentnegotiation.ContentNegotiation
import io.ktor.server.routing.routing
import kotlinx.serialization.Serializable

@Serializable
data class InvocationRequest(val prompt: String)

@Serializable
data class InvocationResponse(val answer: String)

fun Application.module() {
    install(ContentNegotiation) {
        json()
    }

    install(Koog) {
        llm {
            bedrock()
        }
    }

    routing {
        agentCoreRuntime {
            handle<InvocationRequest, InvocationResponse> { request, context ->
                val sessionId = context.getHeader("X-Amzn-Bedrock-AgentCore-Runtime-Session-Id")
                val answer = aiAgent(
                    input = request.prompt,
                    model = BedrockModels.AmazonNovaMicro,
                )
                InvocationResponse(answer)
            }
        }
    }
}
```
类型化处理器将请求反序列化和响应序列化委托给 Ktor 的 `ContentNegotiation` 插件。宿主应用程序必须为其接受的媒体类型安装转换器，例如用于 JSON 请求和响应的 `json()`。服务器引擎、端口和其他应用程序插件也仍由宿主应用程序控制。

### 处理不同的负载类型

对于非 JSON 负载或多模态响应，请配置统一的 `handler`。它接收一个 `InvocationInput` 和一个 `AgentCoreContext`，并返回一个 `InvocationResult`：
```
routing {
    agentCoreRuntime {
        handler = { input, context ->
            when (input) {
                is InvocationInput.Text -> InvocationResult.Text(
                    aiAgent(input.body, model = BedrockModels.AmazonNovaMicro)
                )
                is InvocationInput.Binary -> InvocationResult.Binary(input.bytes, input.contentType)
                is InvocationInput.Stream -> InvocationResult.Text("Received a streamed request")
                is InvocationInput.Multipart -> InvocationResult.Text("Received multipart data")
            }
        }
    }
}
```
统一处理器支持：

- `InvocationResult.Text`，用于一次性文本输出，并基于 `Accept` 进行内容协商。
- `InvocationResult.Binary`，用于原始图像、音频、视频或文档字节，并带有显式内容类型。
- `InvocationResult.TextStream`，用于将 `Flow<String>` 作为立即刷新的 `text/event-stream` 事件发出。
- `InvocationResult.BinaryStream`，用于原始流式数据块，并带有调用方选择的内容类型。

流式响应会直接写入，不需要 Ktor 的 `SSE` 插件。

### 配置请求处理

`AgentCoreRuntimeConfig` 提供以下选项：

| 选项 | 描述 | 默认值 |
| --- | --- | --- |
| `handler` | 统一处理器，除非注册了类型化的 `handle<I, O>` 处理器，否则使用该处理器。 | 未设置 |
| `binaryStreamThresholdBytes` | 大于此大小的二进制请求体，或没有 `Content-Length` 的二进制请求体，会以 `InvocationInput.Stream` 暴露。 | 1 MiB |
| `maxRequestBytes` | 拒绝声明的 `Content-Length` 超过该限制的请求，并返回 HTTP 413。 | 100 MiB |
| `handlerTimeoutMillis` | 当处理器超过此超时时间时返回 HTTP 504。非正值会禁用超时。 | `0` |
| `pingService` | 用于 `/ping` 端点的自定义健康检查服务。 | 感知任务的默认服务 |
| `taskTracker` | 通过 `AgentCoreContext` 暴露，并由默认健康检查服务使用的跟踪器。 | 新的 `AgentCoreTaskTracker` |

没有 `Content-Length` 头的请求不会预先根据 `maxRequestBytes` 检查；底层服务器引擎的限制仍然适用。

### 监控健康和后台任务

`/ping` 端点返回：

- 当代理没有活动后台任务时，返回 `Healthy` 和 HTTP 200。
- 当 `AgentCoreTaskTracker` 报告有活动工作时，返回 `HealthyBusy` 和 HTTP 200。
- 当健康检查检测到问题时，返回 `Unhealthy` 和 HTTP 503。

在启动长时间运行的后台工作时，使用可从 `AgentCoreContext` 获取的跟踪器。这会让 Runtime 知道代理仍处于活动状态。你可以通过为 `pingService` 分配自定义 `AgentCorePingService` 来替换默认行为。

速率限制也由宿主应用程序控制。全局安装 Ktor 的 `RateLimit` 插件，或将 `agentCoreRuntime` 路由包装在具名的 `rateLimit` 块中，以应用所需策略。

## Amazon Bedrock AgentCore Memory

Koog 通过两种方式与 [Amazon Bedrock AgentCore Memory](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/memory.html) 集成：

- `agents-features-chat-history-aws` 模块将对话历史持久化为 AgentCore 事件。
- `agents-features-longterm-memory-aws` 模块检索由 AgentCore 记忆策略生成的记录，并将其添加到代理提示中。

这两种集成都需要 JVM 17 或更高版本，以及一个 AgentCore 记忆资源。通过标准 AWS SDK 凭证和区域提供程序链配置 AWS 凭证和区域。

### 添加依赖项

将一个或两个 Memory 集成模块添加到你的 Gradle 构建中：
```
dependencies {
    implementation("ai.koog:agents-features-chat-history-aws:$koogVersion")
    implementation("ai.koog:agents-features-longterm-memory-aws:$koogVersion")
}
```
两个模块都公开了其公共 API 所使用的 AWS SDK for Kotlin `BedrockAgentCoreClient`。长期记忆还公开了
`BedrockAgentCoreControlClient`，用于记忆策略发现。

### 持久化对话历史

`AgentcoreChatHistoryProvider` 通过 AgentCore 的 `createEvent` 和
`listEvents` API 实现了 Koog 的 `ChatHistoryProvider`。通过 `ChatMemory` 功能安装它：
```
import ai.koog.agents.chatMemory.feature.ChatMemory
import ai.koog.agents.core.agent.AIAgent
import ai.koog.agents.features.chathistory.aws.AgentcoreChatHistoryProvider
import aws.sdk.kotlin.services.bedrockagentcore.BedrockAgentCoreClient

val agentCoreClient = BedrockAgentCoreClient { region = "us-west-2" }
val chatHistoryProvider = AgentcoreChatHistoryProvider(
    client = agentCoreClient,
    memoryId = "memory-id",
)

val agent = AIAgent(/* ... */) {
    install(ChatMemory) {
        chatHistoryProvider = chatHistoryProvider
    }
}

val result = agent.run(
    agentInput = "Remember that I prefer window seats.",
    conversationId = "user-123:trip-456",
)
```
对话 ID 可以是 `actorId:sessionId` 或仅 `actorId`。当省略会话部分时，除非在其构造函数中设置 `defaultSession`，否则提供程序使用 `default-session`。

该提供程序存储纯文本的 `Message.User` 和 `Message.Assistant` 消息。从 AgentCore 加载的消息在元数据中携带其事件 ID，使提供程序在再次保存完整历史记录时仅存储新消息。系统、工具、推理和非文本内容默认会被跳过；设置 `ignoreUnsupportedValues = false` 可改为拒绝这些内容。使用 `pageSize` 控制 `listEvents` 分页，使用 `totalEventsLimit` 限制加载事件的数量。

### 检索长期记忆

`LongTermMemory` 可以在每次 LLM 请求之前查询一个或多个 AgentCore 记忆策略。`agentcore` DSL 创建复合检索，因此单个块可以组合多种策略类型和命名空间范围：
```
import ai.koog.agents.features.longtermmemory.aws.dsl.agentcore
import ai.koog.agents.longtermmemory.feature.LongTermMemory

val agent = AIAgent(/* ... */) {
    install(LongTermMemory) {
        retrieval {
            agentcore(agentCoreClient, memoryId = "memory-id") {
                semantic(
                    strategyId = "semantic-strategy-id",
                    actorId = "user-123",
                    topK = 5,
                )
                userPreferences(
                    strategyId = "preference-strategy-id",
                    actorId = "user-123",
                    limit = 20,
                )
                summary(
                    strategyId = "summary-strategy-id",
                    actorId = "user-123",
                    sessionId = "trip-456",
                    topK = 3,
                )
            }
        }
    }
}
```
DSL 提供以下辅助工具：

| 辅助工具 | AgentCore 策略 | 命名空间范围 | 检索 |
| --- | --- | --- | --- |
| `semantic` | 语义记忆 | Actor | 相似度搜索 |
| `userPreferences` | 用户偏好 | Actor | 记录列表 |
| `summary` | 摘要 | Actor 和会话 | 相似度搜索 |
| `episodes` | 情景片段 | Actor 和会话 | 相似度搜索 |
| `reflections` | 情景反思 | Actor | 相似度搜索 |
| `episodic` | 片段和反思 | 两种范围 | 复合相似度搜索 |

默认情况下，命名空间遵循 AWS 文档中记录的布局：
`/strategies/{strategyId}/actors/{actorId}/` 用于 actor 范围的记忆，
`/strategies/{strategyId}/actors/{actorId}/sessions/{sessionId}/` 用于会话范围的记忆。如果记忆资源
使用自定义命名空间模板，请在 `agentcore` 块中分配一个 `AgentcoreNamespaceResolver`。

默认的 `AgentcorePromptAugmenter` 会将语义、偏好、片段和反思记录放入系统
消息中。摘要记录会追加到最新的用户消息中。在块中设置 `augmenter` 以使用另一个 Koog
`PromptAugmenter`。

### 发现已配置的记忆策略

当策略 ID 或命名空间模板不应硬编码时，请将 `AgentcoreStrategyDiscovery` 与 AWS
`BedrockAgentCoreControlClient` 一起使用，然后将其结果传递给 `agentcoreDiscovered`。发现 DSL 会配置
为记忆资源返回的所有受支持策略，并允许你覆盖检索限制、分数、过滤器、
命名空间模式，或排除单个策略。当发现的集合包含摘要或情景策略时，需要提供 `sessionId`。

AgentCore 会根据存储的事件异步创建长期记录。因此，由 `ChatMemory` 写入的事件可能
不会立即可用于 `LongTermMemory`。
