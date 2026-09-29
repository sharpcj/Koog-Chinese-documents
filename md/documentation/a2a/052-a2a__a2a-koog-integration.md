# A2A 与 Koog 集成

Beta

此功能是 beta 模块（`1.3.0-beta`）的一部分。API 可能会在未来版本中发生变化。
有关详细信息，请参阅[模块版本管理](../../module-versioning/)。

Koog 提供了与 A2A 协议的无缝集成，使你能够将 Koog agents 暴露为 A2A servers，并将 Koog agents 连接到其他符合 A2A 规范的 agents。

## 依赖项

A2A Koog 集成会根据你的使用场景需要特定的 feature 模块：

### 用于将 Koog Agents 暴露为 A2A Servers

将这些依赖项添加到你的 `build.gradle.kts`：

```
dependencies {
    // Koog A2A server integration feature
    implementation("ai.koog:agents-features-a2a-server:$koogVersion")

    // HTTP JSON-RPC transport
    implementation("ai.koog:a2a-transport-server-jsonrpc-http:$koogVersion")

    // Ktor server engine (choose one that fits your needs)
    implementation("io.ktor:ktor-server-netty:$ktorVersion")
}
```

### 用于将 Koog Agents 连接到 A2A Agents

将这些依赖项添加到你的 `build.gradle.kts`：

```
dependencies {
    // Koog A2A client integration feature
    implementation("ai.koog:agents-features-a2a-client:$koogVersion")

    // HTTP JSON-RPC transport
    implementation("ai.koog:a2a-transport-client-jsonrpc-http:$koogVersion")

    // Ktor client engine (choose one that fits your needs)
    implementation("io.ktor:ktor-client-cio:$ktorVersion")
}
```

## 概览

该集成支持两种主要模式：

1. **将 Koog agents 暴露为 A2A servers** - 让你的 Koog agents 可通过 A2A 协议被发现和访问
2. **将 Koog agents 连接到 A2A agents** - 让你的 Koog agents 与其他符合 A2A 规范的 agents 通信

## 将 Koog Agents 暴露为 A2A Servers

### 定义带有 A2A feature 的 Koog Agent

先定义一个 Koog agent。Agent 的逻辑可以各不相同，下面是一个带 tools 的基础单次运行 agent 示例。
该 agent 接收来自用户的消息，并将其转发给 llm。
如果 llm 响应包含 tool call，agent 会执行该 tool 并将结果转发给 llm。
如果 llm 响应包含 assistant message，agent 会将 assistant message 发送给用户并结束。

在输入接收时，agent 会将包含输入消息的 task submitted 事件发送给 A2A client。
在每次 tool call 时，agent 会将包含 tool call 和结果的 task working 事件发送给 A2A client。
在 assistant message 时，agent 会将包含 assistant message 的 task complete 事件发送给 A2A client。

```
/**
 * Create a Koog agent with A2A feature
 */
@OptIn(ExperimentalUuidApi::class)
private fun createAgent(
    context: RequestContext<MessageSendParams>,
    eventProcessor: SessionEventProcessor,
) = AIAgent(
    promptExecutor = MultiLLMPromptExecutor(
        LLMProvider.Google to GoogleLLMClient("api-key")
    ),
    toolRegistry = ToolRegistry {
        // Declare tools here
    },
    strategy = strategy<A2AMessage, Unit>("test") {
        val nodeSetup by node<A2AMessage, Unit> { inputMessage ->
            // Convenience function to transform A2A message into Koog message
            val input = inputMessage.toKoogMessage()
            llm.writeSession {
                appendPrompt {
                    message(input)
                }
            }
            // Send update event to A2A client
            withA2AAgentServer {
                sendTaskUpdate("Request submitted: ${input.content}", TaskState.Submitted)
            }
        }

        // Calling llm
        val nodeLLMRequest by node<Unit, Message> {
            llm.writeSession {
                requestLLM()
            }
        }

        // Executing tool
        val nodeProcessTool by node<MessagePart.Tool.Call, Unit> { toolCall ->
            withA2AAgentServer {
                sendTaskUpdate("Executing tool: ${toolCall.content}", TaskState.Working)
            }

            val toolResult = environment.executeTool(toolCall)

            llm.writeSession {
                appendPrompt {
                    tool {
                        result(toolResult)
                    }
                }
            }
            withA2AAgentServer {
                sendTaskUpdate("Tool result: ${toolResult.content}", TaskState.Working)
            }
        }

        // Sending assistant message
        val nodeProcessAssistant by node<String, Unit> { assistantMessage ->
            withA2AAgentServer {
                sendTaskUpdate(assistantMessage, TaskState.Completed)
            }
        }

        edge(nodeStart forwardTo nodeSetup)
        edge(nodeSetup forwardTo nodeLLMRequest)

        // If a tool call is returned from llm, forward to the tool processing node and then back to llm
        edge(nodeLLMRequest forwardTo nodeProcessTool onToolCall { true })
        edge(nodeProcessTool forwardTo nodeLLMRequest)

        // If an assistant message is returned from llm, forward to the assistant processing node and then to finish
        edge(nodeLLMRequest forwardTo nodeProcessAssistant onAssistantMessage { true })
        edge(nodeProcessAssistant forwardTo nodeFinish)
    },
    agentConfig = AIAgentConfig(
        prompt = prompt("agent") { system("You are a helpful assistant.") },
        model = GoogleModels.Gemini2_5Pro,
        maxAgentIterations = 10
    ),
) {
    install(A2AAgentServer) {
        this.context = context
        this.eventProcessor = eventProcessor
    }
}

/**
 * Convenience function to send task update event to A2A client
 * @param content The message content
 * @param state The task state
 */
@OptIn(ExperimentalUuidApi::class)
private suspend fun A2AAgentServer.sendTaskUpdate(
    content: String,
    state: TaskState,
) {
    val message = A2AMessage(
        messageId = Uuid.random().toString(),
        role = Role.Agent,
        parts = listOf(
            TextPart(content)
        ),
        contextId = context.contextId,
        taskId = context.taskId,
    )

    val task = Task(
        id = context.taskId,
        contextId = context.contextId,
        status = TaskStatus(
            state = state,
            message = message,
            timestamp = Clock.System.now(),
        )
    )
    eventProcessor.sendTaskEvent(task)
}
```

## A2AAgentServer Feature 机制

`A2AAgentServer` 是一个 Koog agent feature，用于实现 Koog agents 与 A2A 协议之间的无缝集成。
`A2AAgentServer` feature 提供对 `RequestContext` 和 `SessionEventProcessor` 实体的访问，这些实体用于在 Koog agent 内部与 A2A client 通信。

要安装该 feature，请在 agent 上调用 `install` 函数，并传入 `A2AAgentServer` feature 以及 `RequestContext` 和 `SessionEventProcessor`：

```
// Install the feature
install(A2AAgentServer) {
    this.context = context
    this.eventProcessor = eventProcessor
}
```

要从 Koog agent strategy 访问这些实体，该 feature 提供了 `withA2AAgentServer` 函数，使 agent nodes 能够在其执行上下文中访问 A2A server 能力。
它会检索已安装的 `A2AAgentServer` feature，并将其作为 action block 的 receiver 提供。

```
// Usage within agent nodes
withA2AAgentServer {
    // 'this' is now A2AAgentServer instance
    eventProcessor.sendTaskUpdate("Processing your request...", TaskState.Working)
}
```

### 启动 A2A Server

运行 server 后，Koog agent 将可通过 A2A 协议被发现和访问。

```
val agentCard = AgentCard(
    name = "Koog Agent",
    url = "http://localhost:9999/koog",
    description = "Simple universal agent powered by Koog",
    version = "1.0.0",
    protocolVersion = "0.3.0",
    preferredTransport = TransportProtocol.JSONRPC,
    capabilities = AgentCapabilities(streaming = true),
    defaultInputModes = listOf("text"),
    defaultOutputModes = listOf("text"),
    skills = listOf(
        AgentSkill(
            id = "koog",
            name = "Koog Agent",
            description = "Universal agent powered by Koog. Supports tool calling.",
            tags = listOf("chat", "tool"),
        )
    )
)
// Server setup
val server = A2AServer(agentExecutor = KoogAgentExecutor(), agentCard = agentCard)
val transport = HttpJSONRPCServerTransport(server)
transport.start(engineFactory = Netty, port = 8080, path = "/chat", wait = true)
```

## 将 Koog Agents 连接到 A2A Agents

### 创建 A2A Client 并连接到 A2A Server

```
val transport = HttpJSONRPCClientTransport(url = "http://localhost:9999/koog")
val agentCardResolver =
    UrlAgentCardResolver(baseUrl = "http://localhost:9999", path = "/koog")
val client = A2AClient(transport = transport, agentCardResolver = agentCardResolver)

val agentId = "koog"
client.connect()
```

### 创建 Koog Agent 并将 A2A Client 添加到 A2AAgentClient Feature

要从你的 Koog Agent 连接到 A2A agent，可以使用 A2AAgentClient feature，它提供用于连接 A2A agents 的 client API。
Client 的原理与 server 相同：你安装该 feature，并传入 `A2AAgentClient` feature 以及 `RequestContext` 和 `SessionEventProcessor`。

```
val agent = AIAgent(
    promptExecutor = MultiLLMPromptExecutor(
        LLMProvider.Google to GoogleLLMClient("api-key")
    ),
    toolRegistry = ToolRegistry {
        // declare tools here
    },
    strategy = strategy<String, Unit>("test") {

        val nodeCheckStreaming by nodeA2AClientGetAgentCard().transform { it.capabilities.streaming }

        val nodeA2ASendMessageStreaming by nodeA2AClientSendMessageStreaming()
        val nodeA2ASendMessage by nodeA2AClientSendMessage()

        val nodeProcessStreaming by node<Flow<Response<Event>>, Unit> {
            it.collect { response ->
                when (response.data) {
                    is Task -> {
                        // Process task
                    }

                    is A2AMessage -> {
                        // Process message
                    }

                    is TaskStatusUpdateEvent -> {
                        // Process task status update
                    }

                    is TaskArtifactUpdateEvent -> {
                        // Process task artifact update
                    }
                }
            }
        }

        val nodeProcessEvent by node<CommunicationEvent, Unit> { event ->
            when (event) {
                is Task -> {
                    // Process task
                }

                is A2AMessage -> {
                    // Process message
                }
            }
        }

        // If streaming is supported, send a message, process response and finish
        edge(nodeStart forwardTo nodeCheckStreaming transformed { agentId })
        edge(
            nodeCheckStreaming forwardTo nodeA2ASendMessageStreaming
                onCondition { it == true } transformed { buildA2ARequest(agentId) }
        )
        edge(nodeA2ASendMessageStreaming forwardTo nodeProcessStreaming)
        edge(nodeProcessStreaming forwardTo nodeFinish)

        // If streaming is not supported, send a message, process response and finish
        edge(
            nodeCheckStreaming forwardTo nodeA2ASendMessage
                onCondition { it == false } transformed { buildA2ARequest(agentId) }
        )
        edge(nodeA2ASendMessage forwardTo nodeProcessEvent)
        edge(nodeProcessEvent forwardTo nodeFinish)

        // If streaming is not supported, send a message, process response and finish
        edge(nodeCheckStreaming forwardTo nodeFinish onCondition { it == null }
            transformed { println("Failed to get agents card") }
        )

    },
    agentConfig = AIAgentConfig(
        prompt = prompt("agent") { system("You are a helpful assistant.") },
        model = GoogleModels.Gemini2_5Pro,
        maxAgentIterations = 10
    ),
) {
    install(A2AAgentClient) {
        this.a2aClients = mapOf(agentId to client)
    }
}

@OptIn(ExperimentalUuidApi::class)
private fun AIAgentGraphContextBase.buildA2ARequest(agentId: String): A2AClientRequest<MessageSendParams> =
    A2AClientRequest(
        agentId = agentId,
        callContext = ClientCallContext.Default,
        params = MessageSendParams(
            message = A2AMessage(
                messageId = Uuid.random().toString(),
                role = Role.User,
                parts = listOf(
                    TextPart(agentInput as String)
                )
            )
        )
    )
```