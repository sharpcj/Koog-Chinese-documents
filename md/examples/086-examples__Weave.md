# Koog agents 的 Weave tracing

[在 GitHub 上打开](https://github.com/JetBrains/koog/blob/develop/examples/notebooks/Weave.ipynb)
 [下载 .ipynb](https://raw.githubusercontent.com/JetBrains/koog/develop/examples/notebooks/Weave.ipynb)

本 notebook 演示如何使用 OpenTelemetry (OTLP) 将 Koog agents 追踪到 W&B Weave。
你将创建一个简单的 Koog `AIAgent`，启用 Weave exporter，运行一个 prompt，并在 Weave UI 中查看
丰富的 trace。

背景信息请参阅 Weave OpenTelemetry 文档：https://weave-docs.wandb.ai/guides/tracking/otel/

## 前提条件

运行示例之前，请确保你拥有：

- Weave/W&B 账号：https://wandb.ai
- 从 https://wandb.ai/authorize 获取的 API key，并作为环境变量暴露：`WEAVE_API_KEY`
- 作为 `WEAVE_ENTITY` 暴露的 Weave entity（团队或用户）名称
- 可在你的 W&B dashboard 中找到它：https://wandb.ai/home
- 作为 `WEAVE_PROJECT_NAME` 暴露的项目名称（如果未设置，此示例使用 `koog-tracing`）
- 用于运行 Koog agent 的 OpenAI API key，作为 `OPENAI_API_KEY` 暴露

示例（macOS/Linux）：

```
export WEAVE_API_KEY=...  # required by Weave
export WEAVE_ENTITY=your-team-or-username
export WEAVE_PROJECT_NAME=koog-tracing
export OPENAI_API_KEY=...
```

## Notebook 设置

我们使用最新的 Kotlin Jupyter descriptors。如果你已经将 Koog 预配置为 `%use` plugin，
可以取消下面这一行的注释。

```
%useLatestDescriptors
//%use koog
```

## 创建 agent 并启用 Weave tracing

我们构造一个最小的 `AIAgent`，并安装带 Weave exporter 的 `OpenTelemetry` feature。
exporter 会使用你的环境配置将 OTLP span 发送到 Weave：
- `WEAVE_API_KEY` — 用于 Weave 认证
- `WEAVE_ENTITY` — trace 归属的团队/用户
- `WEAVE_PROJECT_NAME` — 用于存储 trace 的 Weave 项目

```
import ai.koog.agents.core.agent.AIAgent
import ai.koog.agents.features.opentelemetry.feature.OpenTelemetry
import ai.koog.prompt.executor.clients.openai.OpenAIModels
import ai.koog.prompt.executor.llms.all.simpleOpenAIExecutor

val entity = System.getenv()["WEAVE_ENTITY"] ?: throw IllegalArgumentException("WEAVE_ENTITY is not set")
val projectName = System.getenv()["WEAVE_PROJECT_NAME"] ?: "koog-tracing"

val agent = AIAgent(
    executor = simpleOpenAIExecutor(System.getenv("OPENAI_API_KEY")),
    llmModel = OpenAIModels.Chat.GPT4oMini,
    systemPrompt = "You are a code assistant. Provide concise code examples."
) {
    install(OpenTelemetry) {
        addWeaveExporter(
            weaveEntity = entity,
            weaveProjectName = projectName
        )
    }
}
```

## 运行 agent 并在 Weave 中查看 trace

执行一个简单的 prompt。完成后，打开打印出的链接以在 Weave 中查看 trace。
你应该会看到 agent 运行、model 调用和其他已 instrumented 操作的 span。

```
import kotlinx.coroutines.runBlocking

println("Running agent with Weave tracing")

runBlocking {
    val result = agent.run("Tell me a joke about programming")
    "Result: $result\nSee traces on https://wandb.ai/$entity/$projectName/weave/traces"
}
```

## 故障排除

- 如果看不到 trace，请验证环境中已设置 `WEAVE_API_KEY`、`WEAVE_ENTITY` 和 `WEAVE_PROJECT_NAME`。
- 确保你的网络允许向 Weave 的 OTLP endpoint 发起出站 HTTPS 连接。
- 确认你的 OpenAI key 有效，并且你的账号可以访问所选 model。