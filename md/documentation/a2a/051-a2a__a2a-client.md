# A2A Client

Beta

此功能是 beta 模块（`1.3.0-beta`）的一部分。API 可能会在未来版本中发生变化。
有关详细信息，请参阅[模块版本管理](../../module-versioning/)。

A2A client 使你能够通过网络与符合 A2A 规范的 agents 通信。
它提供了 [A2A protocol specification](https://a2a-protocol.org/latest/specification/) 的完整实现，处理 agent 发现、消息交换、任务管理以及实时流式响应。

## 依赖项

要在项目中使用 A2A client，请将以下依赖项添加到你的 `build.gradle.kts`：

```
dependencies {
    // Core A2A client library
    implementation("ai.koog:a2a-client:$koogVersion")

    // HTTP JSON-RPC transport (most common)
    implementation("ai.koog:a2a-transport-client-jsonrpc-http:$koogVersion")

    // Ktor client engine (choose one that fits your needs)
    implementation("io.ktor:ktor-client-cio:$ktorVersion")
}
```

## 概览

A2A client 充当你的应用程序与符合 A2A 规范的 agents 之间的桥梁。
它编排整个通信生命周期，同时保持协议合规性并提供可靠的会话管理。

## 核心组件

### A2AClient

实现完整 A2A 协议的主 client 类。它作为中央协调器，负责：

- **管理**通过可插拔 resolvers 进行的连接和 agent 发现
- **编排**消息交换和任务操作，并自动保持协议合规
- **处理**agents 支持时的流式响应和实时通信
- **提供**全面的错误处理和 fallback 机制，以构建可靠的应用程序

`A2AClient` 接受两个必需参数：

- `ClientTransport`，用于处理网络通信层
- `AgentCardResolver`，用于处理 agent 发现和元数据检索

`A2AClient` 接口提供了若干用于与 A2A agents 交互的关键方法：

- `connect` 方法 - 连接到 agent 并检索其能力，发现该 agent 能做什么并缓存 AgentCard
- `sendMessage` 方法 - 向 agent 发送消息，并为简单的请求-响应模式接收单个响应
- `sendMessageStreaming` 方法 - 发送支持流式传输的消息以获取实时响应，返回包含部分消息和任务更新等事件的 Flow
- `getTask` 方法 - 查询特定任务的状态和详细信息
- `cancelTask` 方法 - 如果 agent 支持取消，则取消正在运行的任务
- `cachedAgentCard` 方法 - 不发起网络请求而获取缓存的 agent card；如果尚未调用 connect，则返回 null

### ClientTransport

`ClientTransport` 接口处理底层网络通信，而 A2A client 管理协议逻辑。
它抽象了传输层特定细节，使你能够无缝使用不同协议。

#### HTTP JSON-RPC Transport

A2A agents 最常用的 transport：

```
val transport = HttpJSONRPCClientTransport(
    url = "https://agent.example.com/a2a",        // Agent endpoint URL
    httpClient = HttpClient(CIO) {                // Optional: custom HTTP client
        install(ContentNegotiation) {
            json()
        }
        install(HttpTimeout) {
            requestTimeoutMillis = 30000
        }
    }
)
```

### AgentCardResolver

`AgentCardResolver` 接口检索 agent 元数据和能力。
它支持从多种来源发现 agent，并支持缓存策略以获得最佳性能。

#### URL Agent Card Resolver

按照 A2A 约定从 HTTP endpoints 获取 agent cards：

```
val agentCardResolver = UrlAgentCardResolver(
    baseUrl = "https://agent.example.com",           // Base URL of the agent service
    path = "/.well-known/agent-card.json",           // Standard agent card location
    httpClient = HttpClient(CIO),                    // Optional: custom HTTP client
)
```

## 快速开始

### 1. 创建 Client

定义 transport 和 agent card resolver，并创建 client。

```
// HTTP JSON-RPC transport
val transport = HttpJSONRPCClientTransport(
    url = "https://agent.example.com/a2a"
)

// Agent card resolver
val agentCardResolver = UrlAgentCardResolver(
    baseUrl = "https://agent.example.com",
    path = "/.well-known/agent-card.json"
)

// Create client
val client = A2AClient(transport, agentCardResolver)
```

### 2. 连接并发现

连接到 agent 并检索其 card。
拥有 agent 的 card 后，你可以查询其能力并执行其他操作，例如检查它是否支持流式传输。

```
// Connect and retrieve agent capabilities
client.connect()
val agentCard = client.cachedAgentCard()

println("Connected to: ${agentCard.name}")
println("Supports streaming: ${agentCard.capabilities.streaming}")
```

### 3. 发送消息

向 agent 发送消息并接收单个响应。
如果 agent 直接响应，响应可以是消息；如果 agent 正在执行任务，响应也可以是任务事件。

```
val message = Message(
    messageId = UUID.randomUUID().toString(),
    role = Role.User,
    parts = listOf(TextPart("Hello, agent!")),
    contextId = "conversation-1"
)

val request = Request(data = MessageSendParams(message))
val response = client.sendMessage(request)

// Handle response
when (val event = response.data) {
    is Message -> {
        val text = event.parts
            .filterIsInstance<TextPart>()
            .joinToString { it.text }
        print(text) // Stream partial responses
    }
    is TaskEvent -> {
        if (event.final) {
            println("\nTask completed")
        }
    }
}
```

### 4. 发送流式消息

A2A client 支持用于实时通信的流式响应。
它不会接收单个响应，而是返回包含消息和任务更新等事件的 `Flow`。

```
// Check if agent supports streaming
if (client.cachedAgentCard()?.capabilities?.streaming == true) {
    client.sendMessageStreaming(request).collect { response ->
        when (val event = response.data) {
            is Message -> {
                val text = event.parts
                    .filterIsInstance<TextPart>()
                    .joinToString { it.text }
                print(text) // Stream partial responses
            }
            is TaskStatusUpdateEvent -> {
                if (event.final) {
                    println("\nTask completed")
                }
            }
        }
    }
} else {
    // Fallback to non-streaming
    val response = client.sendMessage(request)
    // Handle single response
}
```

### 5. 管理任务

A2A Client 提供了通过查询 server 任务状态和取消任务来控制任务的方法。

```
// Query task status
val taskRequest = Request(data = TaskQueryParams(taskId = "task-123"))
val taskResponse = client.getTask(taskRequest)
val task = taskResponse.data

println("Task state: ${task.status.state}")

// Cancel running task
if (task.status.state == TaskState.Working) {
    val cancelRequest = Request(data = TaskIdParams(taskId = "task-123"))
    val cancelledTask = client.cancelTask(cancelRequest).data
    println("Task cancelled: ${cancelledTask.status.state}")
}
```