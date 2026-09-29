# Koog 中的 OpenTelemetry：追踪你的 AI agent

[在 GitHub 上打开](https://github.com/JetBrains/koog/blob/develop/examples/notebooks/OpenTelemetry.ipynb)
 [下载 .ipynb](https://raw.githubusercontent.com/JetBrains/koog/develop/examples/notebooks/OpenTelemetry.ipynb)

本 notebook 演示如何为 Koog AI agent 添加基于 OpenTelemetry 的 tracing。我们将：
- 将 span 发出到控制台，便于快速本地调试。
- 将 span 导出到 OpenTelemetry Collector，并在 Jaeger 中查看。

前提条件：
- 已安装 Docker/Docker Compose
- 环境变量 `OPENAI_API_KEY` 中提供 OpenAI API key

在运行 notebook 之前，启动本地 OpenTelemetry stack（Collector + Jaeger）：

```
./docker-compose up -d
```

agent 运行后，打开 Jaeger UI：
- http://localhost:16686

稍后停止服务：

```
docker-compose down
```

---

```
%useLatestDescriptors
// %use koog
```

```
import ai.koog.agents.core.agent.AIAgent
import ai.koog.agents.features.opentelemetry.feature.OpenTelemetry
import ai.koog.prompt.executor.clients.openai.OpenAIModels
import ai.koog.prompt.executor.llms.all.simpleOpenAIExecutor
import io.opentelemetry.exporter.logging.LoggingSpanExporter
import io.opentelemetry.exporter.otlp.trace.OtlpGrpcSpanExporter
```

## 配置 OpenTelemetry exporter

在下一个单元中，我们会：
- 创建一个 Koog AIAgent
- 安装 OpenTelemetry feature
- 添加两个 span exporter：
- 用于控制台日志的 LoggingSpanExporter
- 到 http://localhost:4317 OTLP gRPC exporter

这与示例说明一致：控制台日志用于本地调试，OTLP 用于在 Jaeger 中查看 trace。

```
val agent = AIAgent(
    executor = simpleOpenAIExecutor(System.getenv("OPENAI_API_KEY")),
    llmModel = OpenAIModels.Chat.GPT4oMini,
    systemPrompt = "You are a code assistant. Provide concise code examples."
) {
    install(OpenTelemetry) {
        // Add a console logger for local debugging
        addSpanExporter(LoggingSpanExporter.create())

        // Send traces to OpenTelemetry collector
        addSpanExporter(
            OtlpGrpcSpanExporter.builder()
                .setEndpoint("http://localhost:4317")
                .build()
        )
    }
}
```

## 运行 agent 并在 Jaeger 中查看 trace

执行下一个单元来触发一个简单的 prompt。你应该会看到：
- 来自 LoggingSpanExporter 的控制台 span 日志
- 导出到本地 OpenTelemetry Collector 的 trace，并可在 http://localhost:16686 的 Jaeger 中查看

提示：运行该单元后，使用 Jaeger 搜索查找最近的 trace。

```
import ai.koog.agents.utils.use
import kotlinx.coroutines.runBlocking

runBlocking {
    agent.use { agent ->
        println("Running agent with OpenTelemetry tracing...")

        val result = agent.run("Tell me a joke about programming")

        "Agent run completed with result: '$result'.\nCheck Jaeger UI at http://localhost:16686 to view traces"
    }
}
```

## 清理和故障排除

完成后：

- 停止服务：

  ```
  docker-compose down
  ```
- 如果在 Jaeger 中看不到 trace：
- 确保 stack 正在运行：`./docker-compose up -d`，并给它几秒钟时间启动。
- 验证端口：
  - Collector (OTLP gRPC): http://localhost:4317
  - Jaeger UI: http://localhost:16686
- 检查 container 日志：`docker-compose logs --tail=200`
- 确认你的 `OPENAI_API_KEY` 已在 notebook 运行环境中设置。
- 确保 exporter 中的 endpoint 与 collector 匹配：`http://localhost:4317`.
- 预期的 span：
- Koog agent 生命周期
- LLM request/response metadata
- 任何工具执行 span（如果你添加了工具）

现在你可以迭代 agent，并观察 tracing pipeline 中的变化。