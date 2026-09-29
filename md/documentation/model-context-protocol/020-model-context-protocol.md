# 模型上下文协议

Beta

此功能是测试版模块 (`1.3.0-beta`) 的一部分。 API 在未来版本中可能会发生变化。
有关详细信息，请参阅[模块版本控制](../module-versioning/)。

模型上下文协议 (MCP) 是一种标准化协议，允许 AI 代理通过一致的接口与外部工具和服务进行交互。

MCP 将工具和提示公开为 AI 代理可以调用​​的 API 端点。每个工具都有一个特定的名称和一个输入模式，使用 JSON Schema 格式描述其输入和输出。

Koog 框架提供与 MCP 服务器的集成，使您能够将 MCP 工具合并到 Koog 代理中。

要了解有关该协议的更多信息，请参阅[模型上下文协议](https://modelcontextprotocol.io) 文档。

## MCP 服务器

MCP 服务器实现模型上下文协议，并为 AI 代理与工具和服务交互提供标准化方式。

您可以在 [MCP Marketplace](https://mcp.so/) 或 [MCP DockerHub](https://hub.docker.com/u/mcp) 中找到现成的 MCP 服务器。

MCP 服务器支持以下传输协议与代理进行通信：

- Standard 输入/输出 (stdio) 传输协议，用于与作为单独进程运行的 MCP 服务器进行通信。例如，Docker 容器或 CLI 工具。
- 服务器发送的事件 (SSE) 传输协议（可选），用于通过 HTTP 与 MCP 服务器进行通信。

## 与 Koog 集成

Koog 框架使用 [MCP SDK](https://github.com/modelcontextprotocol/kotlin-sdk) 与 MCP 集成，并提供 `agent-mcp` 模块中提供的附加 API 扩展。

此集成允许 Koog 代理执行以下操作：

- 通过各种传输机制（stdio、SSE）连接到 MCP 服务器。
- 从 MCP 服务器检索可用工具。
- 将MCP工具转换为Koog工具界面。
- 在工具注册表中注册转换后的工具。
- 使用 LLM 提供的参数调用 MCP 工具。

### 关键组件

以下是 Koog 中 MCP 集成的主要组件：

|组件|描述 |
| --- | --- |
| [`McpTool`](https://api.koog.ai/agents/agents-mcp/ai.koog.agents.mcp/-mcp-tool/index.html) |充当Koog工具接口和MCP SDK之间的桥梁。 |
| [`McpToolDescriptorParser`](https://api.koog.ai/agents/agents-mcp/ai.koog.agents.mcp/-mcp-tool-descriptor-parser/index.html) |将 MCP 工具定义解析为 Koog 工具描述符格式。 |
| [`McpToolRegistryProvider`](https://api.koog.ai/agents/agents-mcp/ai.koog.agents.mcp/-mcp-tool-registry-provider/index.html) |创建通过各种传输机制(stdio、SSE)连接到 MCP 服务器的 MCP 工具注册表。 |

＃＃ 入门

## 入门

### 1. 设置 MCP 连接

要将MCP与Koog一起使用，您需要建立连接：

1. 启动 MCP 服务器（作为进程、Docker 容器或 Web 服务）。
2. 创建与服务器通信的传输机制。

MCP 服务器支持 stdio 和 SSE 传输机制与代理进行通信，因此您可以使用其中之一进行连接。

#### 连接 stdio

当 MCP 服务器作为单独的进程运行时，使用此协议。以下是使用 stdio 传输设置 MCP 连接的示例：

```
// Start an MCP server (for example, as a process)
val process = ProcessBuilder("path/to/mcp/server").start()

// Create the stdio transport 
val transport = McpToolRegistryProvider.defaultStdioTransport(process)
```

#### 连接 SSE

当 MCP 服务器作为 Web 服务运行时使用此协议。以下是使用 SSE 传输设置 MCP 连接的示例：

```
// Create the SSE transport
val transport = McpToolRegistryProvider.defaultSseTransport("http://localhost:8931")
```

### 2. 创建工具注册表

建立 MCP 连接后，您可以通过以下方式之一使用 MCP 服务器中的工具创建工具注册表：

- 使用提供的传输机制进行通信。例如：

```
// Create a tool registry with tools from the MCP server
val toolRegistry = McpToolRegistryProvider.fromTransport(
    transport = transport,
    serverInfo = McpServerInfo(url = "http://localhost:8931", command = "path/to/mcp/server"),
    name = "my-client",
    version = "1.0.0"
)
```

- 使用连接到 MCP 服务器的 MCP 客户端。例如：

```
// Create a tool registry from an existing MCP client
val toolRegistry = McpToolRegistryProvider.fromClient(
    mcpClient = existingMcpClient,
    serverInfo = McpServerInfo(url = "http://localhost:8931")
)
```

### 3. 与您的代理整合

要将 MCP 工具与 Koog 代理一起使用，您需要向代理注册工具注册表：

```
// Create an agent with the tools
val agent = AIAgent(
    promptExecutor = executor,
    strategy = strategy,
    llmModel = OpenAIModels.Chat.GPT4o,
    toolRegistry = toolRegistry
)

// Run the agent with a task that uses an MCP tool
val result = agent.run("Use the MCP tool to perform a task")
```

## 使用示例

### 谷歌地图 MCP 集成

此示例演示如何使用 MCP 连接到 [Google 地图](https://mcp.so/server/google-maps/modelcontextprotocol) 服务器以获取地理数据：

```
// Start the Docker container with the Google Maps MCP server
val process = ProcessBuilder(
    "docker", "run", "-i",
    "-e", "GOOGLE_MAPS_API_KEY=$googleMapsApiKey",
    "mcp/google-maps"
).start()

// Create the ToolRegistry with tools from the MCP server
val toolRegistry = McpToolRegistryProvider.fromProcess(process = process)

// Create and run the agent
val agent = AIAgent(
    promptExecutor = simpleOpenAIExecutor(openAIApiToken),
    llmModel = OpenAIModels.Chat.GPT4o,
    toolRegistry = toolRegistry,
)
agent.run("Get elevation of the Jetbrains Office in Munich, Germany?")
```

### Playwright MCP 集成

此示例演示如何使用 MCP 连接到 [Playwright](https://mcp.so/server/playwright-mcp/microsoft) 服务器以实现 Web 自动化：

```
// Start the Playwright MCP server
val process = ProcessBuilder(
    "npx", "@playwright/mcp@latest", "--port", "8931"
).start()

// Create the ToolRegistry with tools from the MCP server
val toolRegistry = McpToolRegistryProvider.fromSseUrl("http://localhost:8931")

// Create and run the agent
val agent = AIAgent(
    promptExecutor = simpleOpenAIExecutor(openAIApiToken),
    llmModel = OpenAIModels.Chat.GPT4o,
    toolRegistry = toolRegistry,
)
agent.run("Open a browser, navigate to jetbrains.com, accept all cookies, click AI in toolbar")
```
