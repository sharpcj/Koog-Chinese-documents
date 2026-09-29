# 使用 Bright Data 的 The Web MCP 和 Koog 进行网页抓取

[在 GitHub 上打开](https://github.com/JetBrains/koog/blob/develop/examples/bright-data-mcp/)
 [下载 .kt](https://raw.githubusercontent.com/JetBrains/koog/develop/examples/bright-data-mcp/Main.kt)

在本教程中，你将把 Koog agent 连接到 Bright Data 的 Web MCP server，并让它执行网页抓取和数据收集任务。我们将演示如何通过 Model Context Protocol，使用 Bright Data 强大的网页抓取基础设施搜索 Koog.ai 相关信息。

我们会保持简单且可复现，重点是一个最小但现实的 agent + tools 设置，你可以根据自己的网页抓取需求进行调整。

## 前提条件

- 作为环境变量导出的 OpenAI API key：`OPENAI_API_KEY`
- 作为环境变量导出的 Bright Data API token：`BRIGHT_DATA_API_TOKEN`
- PATH 中可用的 Node.js 和 npx
- 带有 Koog 依赖的 Kotlin 开发环境

**提示**：Bright Data MCP server 提供企业级网页抓取工具访问能力，可处理复杂网站、CAPTCHA 和反机器人措施。

## 1) 设置你的 API credentials

我们从环境变量读取两个 API key，以确保 secret 安全且不会进入代码。

```
// Get API keys from environment variables
val openAIApiKey = System.getenv("OPENAI_API_KEY")
    ?: error("OPENAI_API_KEY environment variable is not set")
val brightDataToken = System.getenv("BRIGHT_DATA_API_TOKEN")
    ?: error("BRIGHT_DATA_API_TOKEN environment variable is not set")
```

## 2) 启动 Bright Data 的 The Web MCP server

我们将使用 `npx` 启动 Bright Data 的 MCP server，并使用你的 API token 对其进行配置。该 server 会通过 Model Context Protocol 暴露网页抓取能力。

```
println("Starting Bright Data MCP server...")

// Start the Bright Data MCP server as a separate process
val processBuilder = ProcessBuilder("npx", "@brightdata/mcp")

// Set the API_TOKEN environment variable for the MCP server process
val environment = processBuilder.environment()
environment["API_TOKEN"] = brightDataToken

// Start the process
val process = processBuilder.start()

// Give the process a moment to start
Thread.sleep(2000)
```

## 3) 从 Koog 连接并创建 agent

我们使用 OpenAI executor 构建一个 Koog `AIAgent`，并通过 STDIO transport 将它的 tool registry 连接到 Bright Data MCP server。然后探索可用工具并运行网页抓取任务。

```
println("Creating STDIO transport...")
try {
    // Create the STDIO transport
    val transport = McpToolRegistryProvider.defaultStdioTransport(process)

    println("Creating tool registry...")

    // Create a tool registry with tools from the Bright Data MCP server
    val toolRegistry = McpToolRegistryProvider.fromTransport(
        transport = transport,
        name = "bright-data-client",
        version = "1.0.0"
    )

    // Print available tools (optional - for debugging)
    println("Available tools from Bright Data MCP server:")
    toolRegistry.tools.forEach { tool ->
        println("- ${tool.name}")
    }

    // Create the agent with MCP tools
    val agent = AIAgent(
        executor = simpleOpenAIExecutor(openAIApiKey),
        systemPrompt = "You are a helpful assistant with access to web scraping and data collection tools from Bright Data. You can help users gather information from websites, analyze web data, and provide insights.",
        llmModel = OpenAIModels.Chat.GPT4o,
        temperature = 0.7,
        toolRegistry = toolRegistry,
        maxIterations = 100
    )

    val result = agent.run("Please search for Koog.ai and tell me what is it and who invented it")

    println("\nAgent response:")
    println(result)

} catch (e: Exception) {
    println("Error: ${e.message}")
    e.printStackTrace()
} finally {
    println("Shutting down MCP server...")
    process.destroyForcibly()
}
```

## 4) 完整代码示例

下面是完整的可运行示例，演示如何使用 Bright Data 的 The Web MCP 进行网页抓取：

```
package koog

import ai.koog.agents.core.agent.AIAgent
import ai.koog.agents.mcp.McpToolRegistryProvider
import ai.koog.prompt.executor.clients.openai.OpenAIModels
import ai.koog.prompt.executor.llms.all.simpleOpenAIExecutor
import kotlinx.coroutines.runBlocking

/**
 * The entry point of the program demonstrating AI-driven web scraping and data collection.
 *
 * This function initializes a Bright Data MCP server, sets up tool integration,
 * and defines an AI agent for interacting with web scraping tools. It demonstrates the
 * following key operations:
 *
 * 1. Starts the Bright Data MCP server using a subprocess with proper API token configuration.
 * 2. Configures a registry of tools from the MCP server via STDIO transport communication.
 * 3. Creates an AI agent leveraging OpenAI's GPT-4o model with web scraping capabilities.
 * 4. Runs the agent to perform a specified task (e.g., searching for and analyzing web content
 *    about Koog.ai).
 * 5. Cleans up by shutting down the MCP server process after execution.
 *
 * This function is intended for tutorial purposes, demonstrating how to integrate
 * MCP (Model Context Protocol) servers with AI agents for web data collection and analysis.
 * It requires OPENAI_API_KEY and BRIGHT_DATA_API_TOKEN environment variables to be set.
 */
fun main() = runBlocking {
    // Get API keys from environment variables
    val openAIApiKey = System.getenv("OPENAI_API_KEY")
        ?: error("OPENAI_API_KEY environment variable is not set")
    val brightDataToken = System.getenv("BRIGHT_DATA_API_TOKEN")
        ?: error("BRIGHT_DATA_API_TOKEN environment variable is not set")

    println("Starting Bright Data MCP server...")

    // Start the Bright Data MCP server as a separate process
    val processBuilder = ProcessBuilder("npx", "@brightdata/mcp")

    // Set the API_TOKEN environment variable for the MCP server process
    val environment = processBuilder.environment()
    environment["API_TOKEN"] = brightDataToken

    // Start the process
    val process = processBuilder.start()

    // Give the process a moment to start
    Thread.sleep(2000)

    println("Creating STDIO transport...")

    try {
        // Create the STDIO transport
        val transport = McpToolRegistryProvider.defaultStdioTransport(process)

        println("Creating tool registry...")

        // Create a tool registry with tools from the Bright Data MCP server
        val toolRegistry = McpToolRegistryProvider.fromTransport(
            transport = transport,
            name = "bright-data-client",
            version = "1.0.0"
        )

        // Print available tools (optional - for debugging)
        println("Available tools from Bright Data MCP server:")
        toolRegistry.tools.forEach { tool ->
            println("- ${tool.name}")
        }

        // Create the agent with MCP tools
        val agent = AIAgent(
            executor = simpleOpenAIExecutor(openAIApiKey),
            systemPrompt = "You are a helpful assistant with access to web scraping and data collection tools from Bright Data. You can help users gather information from websites, analyze web data, and provide insights.",
            llmModel = OpenAIModels.Chat.GPT4o,
            temperature = 0.7,
            toolRegistry = toolRegistry,
            maxIterations = 100
        )

        val result = agent.run("Please search for Koog.ai and tell me what is it and who invented it")

        println("\nAgent response:")
        println(result)

    } catch (e: Exception) {
        println("Error: ${e.message}")
        e.printStackTrace()
    } finally {
        println("Shutting down MCP server...")
        process.destroyForcibly()
    }
}
```

## 故障排除

- **连接问题**：如果 agent 无法连接到 MCP server，请确保 Bright Data MCP package 已通过 `npx @brightdata/mcp` 正确安装。
- **API token 错误**：仔细检查你的 `BRIGHT_DATA_API_TOKEN` 是否有效，并且具有网页抓取所需的权限。
- **OpenAI 认证**：验证你的 `OPENAI_API_KEY` 环境变量已正确设置，且 API key 有效。
- **进程超时**：如果 server 启动时间较长，请增加 `Thread.sleep(2000)` 时长。

## 后续步骤

- **探索不同查询**：尝试抓取不同网站，或搜索各种主题。
- **自定义工具集成**：在 Bright Data 的网页抓取能力之外添加你自己的工具。
- **高级抓取**：利用 Bright Data 的高级功能，如住宅代理、CAPTCHA 解决和 JavaScript 渲染。
- **数据处理**：将抓取到的数据与其他 Koog agents 结合，用于分析和洞察。
- **生产部署**：将此模式集成到你的应用中，实现自动化网页数据收集。

## 你学到了什么

本教程演示了如何：
- 设置和配置 Bright Data 的 The Web MCP
- 通过 STDIO transport 将 Koog AI agent 连接到外部 MCP servers
- 使用自然语言指令执行 AI 驱动的网页抓取任务
- 正确处理资源清理和错误管理
- 为生产就绪的网页抓取应用组织代码

Koog 的 AI agent 能力与 Bright Data 的企业级网页抓取基础设施相结合，为自动化数据收集和分析工作流提供了强大的基础。