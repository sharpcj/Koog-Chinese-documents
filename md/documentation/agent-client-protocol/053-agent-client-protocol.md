# Agent Client Protocol

Beta

此功能属于 beta 模块（`1.3.0-beta`）。API 可能会在未来版本中发生变化。
详情请参见[模块版本控制](../module-versioning/)。

Agent Client Protocol (ACP) 是一个开源的标准化协议，使客户端应用能够通过一致的双向接口与 AI agents 通信。
通过在 Koog agent 中实现 ACP，
你可以确保它能轻松集成到任何符合 ACP 的环境中，例如 IDE。

更多信息请参见 [Agent Client Protocol](https://agentclientprotocol.com) 文档。

## 与 Koog 集成

Koog 框架使用 [ACP Kotlin SDK](https://github.com/agentclientprotocol/kotlin-sdk) 并结合额外的 API 扩展来集成 ACP。
该集成提供：

- Koog agent 与符合 ACP 的客户端应用之间的标准化通信
- 工具调用、agent 思考过程和完成状态的自动执行更新
- Koog 的多模态消息格式与 ACP 内容块之间的无缝消息转换
- 将 Koog agent 状态生命周期映射到 ACP session events

Note

由于 [ACP Kotlin SDK](https://github.com/agentclientprotocol/kotlin-sdk) 是 JVM 专用的，
ACP 集成目前仅在 JVM 平台上可用。

### 添加依赖

ACP 支持是一个可选的 [feature](../features/)，默认情况下 Koog 不包含它。
要为你的 Koog agent 实现 ACP，
请添加 [ai.koog:agents-features-acp](https://mvnrepository.com/artifact/ai.koog/agents-features-acp) 依赖，
它自身依赖于 [com.agentclientprotocol:acp](https://mvnrepository.com/artifact/com.agentclientprotocol/acp)。

例如，在 `build.gradle.kts` 中：

```
dependencies {
    implementation("ai.koog:agents-features-acp:$koogVersion")
}
```

### 为 Koog agent 启用 ACP

要将 Koog agent 的内部[事件系统](../agent-events/)与 ACP 协议桥接起来，
请安装 `ai.koog.agents.features.acp.AcpAgent` feature。
安装后，它会监听生命周期事件（例如工具调用或 LLM 响应）
并将它们发送给 ACP client。

```
val agent = AIAgent(
    promptExecutor = simpleOpenAIExecutor(System.getenv("OPENAI_API_KEY")),
    llmModel = OpenAIModels.Chat.GPT4o
) {
    install(AcpAgent) {
        this.sessionId = sessionId
        this.protocol = protocol
        this.eventsProducer = eventsProducer
        this.setDefaultNotifications = true
    }
}
```

关键配置选项：

- **`sessionId`**：标识当前对话 session 的唯一字符串。
- **`protocol`**：用于底层通信的 [`com.agentclientprotocol.protocol.Protocol`](https://github.com/agentclientprotocol/kotlin-sdk/blob/master/acp/src/commonMain/kotlin/com/agentclientprotocol/protocol/Protocol.kt) 实例。
- **`eventsProducer`**：发送 ACP events 的 `kotlinx.coroutines.channels.ProducerScope<Event>`。
  更多信息请参见 [Event streaming](#event-streaming)。
- **`setDefaultNotifications`**：是否为 agent 生命周期事件注册默认通知处理器。
  更多信息请参见 [Handling agent notifications](#handling-agent-notifications)。

该 agent 必须在下一章所述的 ACP session 范围内运行。

### 实现启用 ACP 的 agent

要将你的 Koog agent 连接到 ACP clients，
请实现 [ACP Kotlin SDK](https://github.com/agentclientprotocol/kotlin-sdk) 中的两个核心接口：

- [`AgentSupport`](https://github.com/agentclientprotocol/kotlin-sdk/blob/master/acp/src/commonMain/kotlin/com/agentclientprotocol/agent/AgentSupport.kt)：
  管理 agent 的身份、能力和 session 生命周期（创建或加载 session）。
- [`AgentSession`](https://github.com/agentclientprotocol/kotlin-sdk/blob/master/acp/src/commonMain/kotlin/com/agentclientprotocol/agent/AgentSession.kt)：
  管理单个对话 session，处理 `prompt` 执行并管理取消。

你应该在 `AgentSession` 的 `prompt()` 方法中初始化并运行启用 ACP 的 Koog agent。
示例如下：

AgentSessionAgentSupport

```
class MyAgentSession(
    override val sessionId: SessionId,
    private val promptExecutor: PromptExecutor,
    private val protocol: Protocol,
    private val clock: KoogClock
) : AgentSession {

    private var agentJob: Deferred<Unit>? = null
    private val agentMutex = Mutex()

    override suspend fun prompt(
        content: List<ContentBlock>,
        _meta: JsonElement?
    ): Flow<Event> = channelFlow {
        val agentConfig = AIAgentConfig(
            prompt = prompt("acp") {
                system("You are a helpful assistant.")
            }.appendPrompt(content),
            model = OpenAIModels.Chat.GPT4o,
            maxAgentIterations = 1000
        )

        // Ensure only one agent session runs at a time
        agentMutex.withLock {
            val agent = AIAgent(
                promptExecutor = promptExecutor,
                agentConfig = agentConfig
            ) {
                install(AcpAgent) {
                    this.sessionId = this@MyAgentSession.sessionId.value
                    this.protocol = this@MyAgentSession.protocol
                    this.eventsProducer = this@channelFlow
                    this.setDefaultNotifications = true
                }
            }

            agentJob = async { agent.run("Hello. How can you help me?") }
            agentJob?.await()
        }
    }

    private fun Prompt.appendPrompt(content: List<ContentBlock>): Prompt {
        return withMessages { messages ->
            messages + listOf(content.toKoogMessage(clock))
        }
    }

    override suspend fun cancel() {
        agentJob?.cancel()
    }
}
```

```
class MyAgentSupport(
    private val promptExecutor: PromptExecutor,
    private val clock: KoogClock,
    private val protocol: Protocol,
) : AgentSupport {

    override suspend fun initialize(clientInfo: ClientInfo): AgentInfo {
        return AgentInfo(
            protocolVersion = LATEST_PROTOCOL_VERSION,
            capabilities = AgentCapabilities(
                loadSession = false, // Set to true if you implement session persistence
                promptCapabilities = PromptCapabilities(
                    audio = false,
                    image = false,
                    embeddedContext = true
                )
            )
        )
    }

    @OptIn(ExperimentalUuidApi::class)
    override suspend fun createSession(sessionParameters: SessionCreationParameters): AgentSession {
        val sessionId = SessionId(Uuid.random().toString())
        return MyAgentSession(sessionId, promptExecutor, protocol, clock)
    }

    override suspend fun loadSession(sessionId: SessionId, sessionParameters: SessionCreationParameters): AgentSession {
        throw UnsupportedOperationException("Session loading not implemented")
    }
}
```

## Event streaming

示例中的 `AgentSession` 定义了一个返回 events 的 `channelFlow` 的 `prompt()` 函数。
随后你安装 `AcpAgent` feature，并将 `this@channelFlow` 作为 `eventsProducer`。
这允许从不同协程发送 events。

## 执行同步

示例中的 `AgentSession` 使用 mutex 来同步对 agent 实例的访问，
因为 ACP 不应在前一次 agent 执行完成前触发新的 agent 执行。
为此，创建并运行 agent 的过程会发生在已定义 mutex 的 `withLock` 范围中。

你还会在 `channelFlow` 范围内以 deferred job `agentJob` 的形式异步运行 agent，
以确保 agent 不会被过早取消。

## 处理 ACP client 输入

ACP clients 以 [`ContentBlock`](https://agentclientprotocol.com/protocol/schema#contentblock) 对象列表的形式发送用户输入。
要在 Koog 中处理这些输入，请使用 `List<ContentBlock>.toKoogMessage()` 扩展函数
将 ACP content blocks 转换为 [`Message.User`](https://api.koog.ai/prompt/prompt-model/ai.koog.prompt.message/-message/-user/index.html)，
并将其追加到你的 [agent prompt](../prompts/)。

示例中的 `AgentSession` 定义了一个私有函数，用于扩展 ACP session 中的初始 agent prompt：

```
private fun Prompt.appendPrompt(content: List<ContentBlock>): Prompt {
    return withMessages { messages ->
        messages + listOf(content.toKoogMessage(clock))
    }
}
```

Note

需要一个 `KoogClock` 实例为消息加时间戳。

更多信息请参见 [Converting messages](#converting-messages)。

## 转换消息

`agents-features-acp` 模块提供扩展函数，
用于在 Koog 内部消息类型与 [ACP content blocks](https://agentclientprotocol.com/protocol/content) 之间无缝转换。

从 ACP client 接收输入时，请使用以下函数：

- `List<ContentBlock>.toKoogMessage()` 将 ACP content blocks 列表转换为 [`Message.User`](https://api.koog.ai/prompt/prompt-model/ai.koog.prompt.message/-message/-user/index.html)
- `ContentBlock.toKoogContentPart()` 将单个 ACP content block 转换为 [`ContentPart`](https://api.koog.ai/prompt/prompt-model/ai.koog.prompt.message/-content-part/index.html)

要从 Koog messages 构造 ACP events 或 content blocks，请使用以下函数：

- `Message.Response.toAcpEvents()` 将 [`Message.Response`](https://api.koog.ai/prompt/prompt-model/ai.koog.prompt.message/-message/-response/index.html) 转换为 ACP session update events 列表
- `ContentPart.toAcpContentBlock()` 将 [`ContentPart`](https://api.koog.ai/prompt/prompt-model/ai.koog.prompt.message/-content-part/index.html) 转换为单个 ACP content block

## 处理 agent 通知

默认情况下，`setDefaultNotifications` 设置为 `true`，
启用 ACP 的 agent 会自动处理以下通知：

- **Agent completion**

  当 agent 成功完成时，发送带有 `StopReason.END_TURN` 的 `PromptResponseEvent`
- **Agent execution failures**

  发送带有适当 stop reason 的 `PromptResponseEvent`：

  - 当 agent 超过最大迭代次数时发送 `StopReason.MAX_TURN_REQUESTS`
  - 其他执行失败发送 `StopReason.REFUSAL`
- **LLM responses**

  将 LLM responses 转换并作为 ACP events 发送（text、tool calls、reasoning）
- **Tool call lifecycle**

  报告 tool call 状态变化：

  - tool call 开始时为 `ToolCallStatus.IN_PROGRESS`
  - tool call 成功时为 `ToolCallStatus.COMPLETED`
  - tool call 失败时为 `ToolCallStatus.FAILED`

如果你想自定义通知处理，
请设置 `setDefaultNotifications = false`，并按照规范处理 agent events。

## 发送自定义 events

除了自动通知之外，
你还可以在 agent 执行期间的任何时刻，
在 `withAcpAgent` block 内使用 `sendEvent` 向 ACP client 发送自定义 events。
这适用于进度更新、自定义状态消息或计划更新。

你可以在 `AIAgentContext` 内执行此操作，例如在一个节点中：

```
val plan: Plan = TODO()

val strategy = strategy<Unit, Unit>("my-strategy") {
    val node by node<Unit, Unit> {
        withAcpAgent {
            sendEvent(
                Event.SessionUpdateEvent(
                    SessionUpdate.PlanUpdate(plan.entries)
                )
            )
        }
    }
}
```

你还可以访问底层 `protocol`，向 client 发送自定义请求，例如认证请求：

```
val strategy = strategy<Unit, Unit>("my-strategy") {
    val node by node<Unit, Unit> {
        withAcpAgent {
            protocol.sendRequest(
                AcpMethod.AgentMethods.Authenticate,
                AuthenticateRequest(methodId = AuthMethodId("Google"))
            )
        }
    }
}
```

## 示例

你可以在 Koog repository 的 [/examples](https://github.com/JetBrains/koog/tree/develop/examples/) 下找到可运行的 Koog agents 示例。

### 运行基于控制台的 ACP client

此示例运行一个基于控制台的 ACP client，它会与一个简单的 Koog agent 交互。

1. 打开 [/examples/simple-examples](https://github.com/JetBrains/koog/blob/develop/examples/simple-examples/)。
2. 查看 [README](https://github.com/JetBrains/koog/blob/develop/examples/simple-examples/README.md)，了解如何为 LLM provider 配置 API key。
3. 运行 `runExampleAcpApp` Gradle task。
4. 当 ACP client 在控制台启动后，输入一个发给 agent 的请求，例如：

   ```
   List files in the current directory and create a new file named 'acp-test.txt' with the content 'Hello from ACP!'.
   ```
5. 观察控制台中的 event traces，
   它们会展示 Koog events 如何被转换为 ACP events 并发送给 client。

### 将启用 ACP 的 Koog agent 连接到 JetBrains IDE

此示例演示如何创建启用 ACP 的 agent 并连接到 IntelliJ IDEA。

1. 打开 [/examples/acp-agent](https://github.com/JetBrains/koog/tree/develop/examples/acp-agent)
2. 运行 `installDist` Gradle task。
3. 这应创建 agent 可执行文件：`build/install/acp-agent/bin/acp-agent`
   （Windows 为 `acp-agent.bat`）。
4. 打开 IntelliJ IDEA（或其他 JetBrains IDE）。
5. 转到 **AI Chat** > **Options** > **Add Custom Agent**。
6. 在打开的 `acp.json` 文件中，粘贴以下内容：

   ```
   {
       "agent_servers": {
           "Koog Agent": {
               "command": "/absolute/path/to/acp-agent/build/install/acp-agent/bin/acp-agent",
               "args": [],
               "env": {
                   "OPENAI_API_KEY": "paste-your-api-key-here"
               }
           }
       }
   }
   ```

   配置参数：

   - `agent_servers`：包含一个或多个 agent 配置的对象
   - `Koog Agent`：IDE 的 agent selector 中显示的名称
   - `command`：agent 可执行文件的绝对路径
   - `args`：命令行参数（此 agent 为空）
   - `env`：传递给 agent 进程的环境变量（本例中为 OpenAI API key）
7. 该 agent 应会出现在 **AI Chat** 工具窗口中。

有关向 IDE 添加自定义 agents 的更多信息，
请参见 [AI Assistant documentation](https://www.jetbrains.com/help/ai-assistant/acp.html#add-custom-agent)
以及[这篇博客文章](https://blog.jetbrains.com/ai/2026/02/koog-x-acp-connect-an-agent-to-your-ide-and-more/)。
