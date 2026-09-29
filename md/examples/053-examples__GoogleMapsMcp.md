# 在 Kotlin Notebook 中使用 Koog 连接 Google Maps MCP：从零到海拔查询

[在 GitHub 上打开](https://github.com/JetBrains/koog/blob/develop/examples/notebooks/GoogleMapsMcp.ipynb)
 [下载 .ipynb](https://raw.githubusercontent.com/JetBrains/koog/develop/examples/notebooks/GoogleMapsMcp.ipynb)

在这篇简短的博客式演练中，我们会把 Koog 连接到用于 Google Maps 的 Model Context Protocol（MCP）服务器。我们将使用 Docker 启动服务器，发现可用工具，并让 AI agent 对地址进行地理编码并获取其海拔——全部在 Kotlin Notebook 中完成。

完成后，你将得到一个可复现的端到端示例，可以直接放入你的工作流或文档中。

```
%useLatestDescriptors
%use koog
```

## 前提条件

在运行下面的单元格之前，请确保你已经具备：

- 已安装并正在运行 Docker
- 有效的 Google Maps API key，并已作为环境变量导出：`GOOGLE_MAPS_API_KEY`
- OpenAI API key，并已导出为 `OPENAI_API_KEY`

你可以像这样在 shell 中设置它们（macOS/Linux 示例）：

```
export GOOGLE_MAPS_API_KEY="<your-key>"
export OPENAI_API_KEY="<your-openai-key>"
```

```
// Get the API key from environment variables
val googleMapsApiKey = System.getenv("GOOGLE_MAPS_API_KEY") ?: error("GOOGLE_MAPS_API_KEY environment variable not set")
val openAIApiToken = System.getenv("OPENAI_API_KEY") ?: error("OPENAI_API_KEY environment variable not set")
```

## 启动 Google Maps MCP 服务器（Docker）

我们将使用官方的 `mcp/google-maps` 镜像。该容器会通过 MCP 暴露 `maps_geocode` 和 `maps_elevation` 等工具。我们通过环境变量传入 API key，并以附加模式启动它，这样 notebook 就可以通过 stdio 与它通信。

```
// Start the Docker container with the Google Maps MCP server
val process = ProcessBuilder(
    "docker",
    "run",
    "-i",
    "-e",
    "GOOGLE_MAPS_API_KEY=$googleMapsApiKey",
    "mcp/google-maps"
).start()
```

## 通过 McpToolRegistry 发现工具

Koog 可以通过 stdio 连接到 MCP 服务器。这里，我们从正在运行的进程创建一个工具注册表，并打印发现到的工具及其描述符。

```
val toolRegistry = McpToolRegistryProvider.fromTransport(
    transport = McpToolRegistryProvider.defaultStdioTransport(process)
)
toolRegistry.tools.forEach {
    println(it.name)
    println(it.descriptor)
}
```

## 使用 OpenAI 构建 AI Agent

接下来，我们组装一个由 OpenAI executor 和模型支持的简单 agent。该 agent 将能够通过刚刚创建的注册表调用 MCP 服务器暴露的工具。

```
val agent = AIAgent(
    executor = simpleOpenAIExecutor(openAIApiToken),
    llmModel = OpenAIModels.Chat.GPT4o,
    toolRegistry = toolRegistry,
)
```

## 请求海拔：先地理编码，再查询海拔

我们提示 agent 查找 JetBrains 慕尼黑办公室的海拔。该指令明确告诉 agent 只能使用可用工具，并说明此任务优先使用哪些工具。

```
import kotlinx.coroutines.runBlocking

val request = "Get elevation of the Jetbrains Office in Munich, Germany?"
runBlocking {
    agent.run(
        request +
            "You can only call tools. Get it by calling maps_geocode and maps_elevation tools."
    )
}
```

## 清理

完成后，停止 Docker 进程，避免在后台留下正在运行的内容。

```
process.destroy()
```

## 故障排除与后续步骤

- 如果容器启动失败，请检查 Docker 是否正在运行，以及你的 `GOOGLE_MAPS_API_KEY` 是否有效。
- 如果 agent 无法调用工具，请重新运行发现单元格，确保工具注册表已填充。
- 尝试其他提示，例如使用可用的 Google Maps 工具进行路线规划或地点搜索。

接下来，可以考虑组合多个 MCP 服务器（例如用于 Web 自动化的 Playwright + Google Maps），并让 Koog 编排工具使用，以完成更丰富的任务。