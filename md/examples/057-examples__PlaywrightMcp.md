# 使用 Playwright MCP 和 Koog 驱动浏览器

[在 GitHub 上打开](https://github.com/JetBrains/koog/blob/develop/examples/notebooks/PlaywrightMcp.ipynb)
 [下载 .ipynb](https://raw.githubusercontent.com/JetBrains/koog/develop/examples/notebooks/PlaywrightMcp.ipynb)

在本 notebook 中，你将把 Koog agent 连接到 Playwright 的 Model Context Protocol (MCP) server，并让它驱动真实浏览器完成一个任务：打开 jetbrains.com、接受 cookies，并点击工具栏中的 AI section。

我们会保持简单且可复现，重点是一个最小但现实的 agent + tools 设置，你可以发布并复用它。

```
%useLatestDescriptors
%use koog
```

## 前提条件

- 作为环境变量导出的 OpenAI API key：`OPENAI_API_KEY`
- PATH 中可用的 Node.js 和 npx
- Kotlin Jupyter notebook 环境，并可通过 `%use koog` 使用 Koog

提示：以 headful 模式运行 Playwright MCP server，可以观察浏览器自动执行这些步骤。

## 1) 提供你的 OpenAI API key

我们从 `OPENAI_API_KEY` 环境变量读取 API key。这样可以避免把 secret 放进 notebook。

```
// Get the API key from environment variables
val openAIApiToken = System.getenv("OPENAI_API_KEY") ?: error("OPENAI_API_KEY environment variable not set")
```

## 2) 启动 Playwright MCP server

我们将使用 `npx` 在本地启动 Playwright 的 MCP server。默认情况下，它会暴露一个 SSE endpoint，Koog 可以连接到该 endpoint。

```
// Start the Playwright MCP server via npx
val process = ProcessBuilder(
    "npx",
    "@playwright/mcp@latest",
    "--port",
    "8931"
).start()
```

## 3) 从 Koog 连接并运行 agent

我们使用 OpenAI executor 构建一个最小的 Koog `AIAgent`，并通过 SSE 将它的 tool registry 指向 MCP server。然后要求它严格通过工具完成浏览器任务。

```
import kotlinx.coroutines.runBlocking

runBlocking {
    println("Connecting to Playwright MCP server...")
    val toolRegistry = McpToolRegistryProvider.fromTransport(
        transport = McpToolRegistryProvider.defaultSseTransport("http://localhost:8931/sse")
    )
    println("Successfully connected to Playwright MCP server")

    // Create the agent
    val agent = AIAgent(
        executor = simpleOpenAIExecutor(openAIApiToken),
        llmModel = OpenAIModels.Chat.GPT4o,
        toolRegistry = toolRegistry,
    )

    val request = "Open a browser, navigate to jetbrains.com, accept all cookies, click AI in toolbar"
    println("Sending request: $request")

    agent.run(
        request + ". " +
            "You can only call tools. Use the Playwright tools to complete this task."
    )
}
```

## 4) 关闭 MCP 进程

始终在运行结束时清理外部进程。

```
// Shutdown the Playwright MCP process
println("Closing connection to Playwright MCP server")
process.destroy()
```

## 故障排除

- 如果 agent 无法连接，请确保 MCP server 正在 `http://localhost:8931`. 上运行。
- 如果看不到浏览器，请确保已安装 Playwright，并且它能够在你的系统上启动浏览器。
- 如果收到 OpenAI 的认证错误，请仔细检查 `OPENAI_API_KEY` 环境变量。

## 后续步骤

- 尝试不同的网站或流程。MCP server 暴露了一组丰富的 Playwright tools。
- 替换 LLM model，或向 Koog agent 添加更多工具。
- 将此流程集成到你的应用中，或将 notebook 发布为文档。