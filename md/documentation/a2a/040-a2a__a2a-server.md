# A2A Server

Beta

此功能是 beta 模块（`1.3.0-beta`）的一部分。API 可能会在未来版本中发生变化。
有关详细信息，请参阅[模块版本管理](../../module-versioning/)。

A2A server 使你能够通过标准化的 A2A（Agent-to-Agent）协议暴露 AI agents。它提供了 [A2A protocol specification](https://a2a-protocol.org/latest/specification/) 的完整实现，处理客户端请求、执行 agent 逻辑、管理复杂的任务生命周期，并支持实时流式响应。

## 依赖项

要在项目中使用 A2A server，请将以下依赖项添加到你的 `build.gradle.kts`：

```
dependencies {
    // Core A2A server library
    implementation("ai.koog:a2a-server:$koogVersion")

    // HTTP JSON-RPC transport (most common)
    implementation("ai.koog:a2a-transport-server-jsonrpc-http:$koogVersion")

    // Ktor server engine (choose one that fits your needs)
    implementation("io.ktor:ktor-server-netty:$ktorVersion")
}
```

## 概览

A2A server 充当 A2A 协议传输层与你的自定义 agent 逻辑之间的桥梁。
它编排整个请求生命周期，同时保持协议合规性并提供可靠的会话管理。

## 核心组件

### A2AServer

实现完整 A2A 协议的主 server 类。它作为中央协调器，负责：

- **验证**传入请求是否符合协议规范
- **管理**并发会话和任务生命周期
- **编排**transport、storage 和业务逻辑层之间的通信
- **处理**所有协议操作：消息发送、任务查询、取消、推送通知

`A2AServer` 接受两个必需参数：

- `AgentExecutor`，定义 agent 的业务逻辑实现
- `AgentCard`，定义 agent 能力和元数据

以及若干可选参数，可用于自定义其 storage 和 transport 行为。

### AgentExecutor

`AgentExecutor` 接口是你实现 agent 核心业务逻辑的地方。
它充当 A2A 协议与你的具体 AI agent 能力之间的桥梁。
要启动 agent 的执行，你必须实现 `execute` 方法，并在其中定义 agent 的逻辑。
要取消 agent，你必须实现 `cancel` 方法。

```
class MyAgentExecutor : AgentExecutor {
    override suspend fun execute(
        context: RequestContext<MessageSendParams>,
        eventProcessor: SessionEventProcessor
    ) {
        // Agent logic here
    }

    override suspend fun cancel(
        context: RequestContext<TaskIdParams>,
        eventProcessor: SessionEventProcessor,
        agentJob: Deferred<Unit>?
    ) {
        // Cancel agent here, optional
    }
}
```

`RequestContext` 提供有关当前请求的丰富信息，包括当前会话的 `contextId` 和 `taskId`、发送的 `message` 以及请求的 `params`。

`SessionEventProcessor` 与客户端通信：

- **`sendMessage(message)`**：发送即时响应（聊天式交互）
- **`sendTaskEvent(event)`**：发送任务相关更新（长时间运行的操作）

```
// For immediate responses (like chatbots)
eventProcessor.sendMessage(
    Message(
        messageId = generateId(),
        role = Role.Agent,
        parts = listOf(TextPart("Here's your answer!")),
        contextId = context.contextId
    )
)

// For task-based operations
eventProcessor.sendTaskEvent(
    TaskStatusUpdateEvent(
        contextId = context.contextId,
        taskId = context.taskId,
        status = TaskStatus(
            state = TaskState.Working,
            message = Message(/* progress update */),
            timestamp = Clock.System.now()
        ),
        final = false  // More updates to come
    )
)
```

### AgentCard

`AgentCard` 充当 agent 的自描述清单。它告诉客户端你的 agent 能做什么、如何与其通信，以及它有哪些安全要求。

```
val agentCard = AgentCard(
    // Basic Identity
    name = "Advanced Recipe Assistant",
    description = "AI agent specialized in cooking advice, recipe generation, and meal planning",
    version = "2.1.0",
    protocolVersion = "0.3.0",

    // Communication Settings
    url = "https://api.example.com/a2a",
    preferredTransport = TransportProtocol.JSONRPC,

    // Optional: Multiple transport support
    additionalInterfaces = listOf(
        AgentInterface("https://api.example.com/a2a", TransportProtocol.JSONRPC),
    ),

    // Capabilities Declaration
    capabilities = AgentCapabilities(
        streaming = true,              // Support real-time responses
        pushNotifications = true,      // Send async notifications
        stateTransitionHistory = true  // Maintain task history
    ),

    // Content Type Support
    defaultInputModes = listOf("text/plain", "text/markdown", "image/jpeg"),
    defaultOutputModes = listOf("text/plain", "text/markdown", "application/json"),

    // Define available security schemes
    securitySchemes = mapOf(
        "bearer" to HTTPAuthSecurityScheme(
            scheme = "Bearer",
            bearerFormat = "JWT",
            description = "JWT token authentication"
        ),
        "api-key" to APIKeySecurityScheme(
            `in` = In.Header,
            name = "X-API-Key",
            description = "API key for service authentication"
        )
    ),

    // Specify security requirements (logical OR of requirements)
    security = listOf(
        mapOf("bearer" to listOf("read", "write")),  // Option 1: JWT with read/write scopes
        mapOf("api-key" to emptyList())              // Option 2: API key
    ),

    // Enable extended card for authenticated users
    supportsAuthenticatedExtendedCard = true,

    // Skills/Capabilities
    skills = listOf(
        AgentSkill(
            id = "recipe-generation",
            name = "Recipe Generation",
            description = "Generate custom recipes based on ingredients, dietary restrictions, and preferences",
            tags = listOf("cooking", "recipes", "nutrition"),
            examples = listOf(
                "Create a vegan pasta recipe with mushrooms",
                "I have chicken, rice, and vegetables. What can I make?"
            )
        ),
        AgentSkill(
            id = "meal-planning",
            name = "Meal Planning",
            description = "Plan weekly meals and generate shopping lists",
            tags = listOf("meal-planning", "nutrition", "shopping")
        )
    ),

    // Optional: Branding
    iconUrl = "https://example.com/agent-icon.png",
    documentationUrl = "https://docs.example.com/recipe-agent",
    provider = AgentProvider(
        organization = "CookingAI Inc.",
        url = "https://cookingai.com"
    )
)
```

### Transport Layer

A2A 本身支持多种用于与客户端通信的 transport protocols。
目前，Koog 提供了基于 HTTP 的 JSON-RPC server transport 实现。

#### HTTP JSON-RPC Transport

```
val transport = HttpJSONRPCServerTransport(server)
transport.start(
    engineFactory = CIO,           // Ktor engine (CIO, Netty, Jetty)
    port = 8080,                   // Server port
    path = "/a2a",                 // API endpoint path
    wait = true                    // Block until server stops
)
```

### Storage

A2A server 使用可插拔的 storage 架构，将不同类型的数据分离开来。
所有 storage 实现都是可选的，并且在开发环境中默认使用内存变体。

- **TaskStorage**：任务生命周期管理 - 存储和管理任务状态、历史记录和 artifacts
- **MessageStorage**：会话历史记录 - 管理 conversation contexts 内的消息历史记录
- **PushNotificationConfigStorage**：Webhook 管理 - 管理异步通知的 webhook 配置

## 快速开始

### 1. 创建 AgentCard

定义 agent 的能力和元数据。

```
val agentCard = AgentCard(
    name = "IO Assistant",
    description = "AI agent specialized in input modification",
    version = "2.1.0",
    protocolVersion = "0.3.0",

    // Communication Settings
    url = "https://api.example.com/a2a",
    preferredTransport = TransportProtocol.JSONRPC,

    // Capabilities Declaration
    capabilities =
        AgentCapabilities(
            streaming = true,              // Support real-time responses
            pushNotifications = true,      // Send async notifications
            stateTransitionHistory = true  // Maintain task history
        ),

    // Content Type Support
    defaultInputModes = listOf("text/plain", "text/markdown", "image/jpeg"),
    defaultOutputModes = listOf("text/plain", "text/markdown", "application/json"),

    // Skills/Capabilities
    skills = listOf(
        AgentSkill(
            id = "echo",
            name = "echo",
            description = "Echoes back user messages",
            tags = listOf("io"),
        )
    )
)
```

### 2. 创建 AgentExecutor

在 executor 中管理和实现 agent 逻辑，处理传入请求并发送响应。

```
class EchoAgentExecutor : AgentExecutor {
    override suspend fun execute(
        context: RequestContext<MessageSendParams>,
        eventProcessor: SessionEventProcessor
    ) {
        val userMessage = context.params.message
        val userText = userMessage.parts
            .filterIsInstance<TextPart>()
            .joinToString(" ") { it.text }

        // Echo the user's message back
        val response = Message(
            messageId = UUID.randomUUID().toString(),
            role = Role.Agent,
            parts = listOf(TextPart("You said: $userText")),
            contextId = context.contextId,
            taskId = context.taskId
        )

        eventProcessor.sendMessage(response)
    }
}
```

### 2. 创建 Server

将 agent executor 和 agent card 传给 server。

```
val server = A2AServer(
    agentExecutor = EchoAgentExecutor(),
    agentCard = agentCard
)
```

### 3. 添加 Transport Layer

创建 transport layer 并启动 server。

```
// HTTP JSON-RPC transport
val transport = HttpJSONRPCServerTransport(server)
transport.start(
    engineFactory = CIO,
    port = 8080,
    path = "/agent",
    wait = true
)
```

## Agent 实现模式

### 简单响应 Agent

如果你的 agent 只需要响应单条消息，可以将其实现为简单 agent。
如果 agent 执行逻辑不复杂且不耗时，也可以使用这种模式。

```
class SimpleAgentExecutor : AgentExecutor {
    override suspend fun execute(
        context: RequestContext<MessageSendParams>,
        eventProcessor: SessionEventProcessor
    ) {
        val response = Message(
            messageId = UUID.randomUUID().toString(),
            role = Role.Agent,
            parts = listOf(TextPart("Hello from agent!")),
            contextId = context.contextId,
            taskId = context.taskId
        )

        eventProcessor.sendMessage(response)
    }
}
```

### 基于任务的 Agent

如果你的 agent 的执行逻辑复杂且需要多个步骤，可以将其实现为基于任务的 agent。
如果 agent 执行逻辑耗时且会挂起，也可以使用这种模式。

```
class TaskAgentExecutor : AgentExecutor {
    override suspend fun execute(
        context: RequestContext<MessageSendParams>,
        eventProcessor: SessionEventProcessor
    ) {
        // Send working status
        eventProcessor.sendTaskEvent(
            TaskStatusUpdateEvent(
                contextId = context.contextId,
                taskId = context.taskId,
                status = TaskStatus(
                    state = TaskState.Working,
                    timestamp = Clock.System.now()
                ),
                final = false
            )
        )

        // Do work...

        // Send completion
        eventProcessor.sendTaskEvent(
            TaskStatusUpdateEvent(
                contextId = context.contextId,
                taskId = context.taskId,
                status = TaskStatus(
                    state = TaskState.Completed,
                    timestamp = Clock.System.now()
                ),
                final = true
            )
        )
    }
}
```