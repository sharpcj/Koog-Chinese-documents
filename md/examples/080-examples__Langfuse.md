# 使用 OpenTelemetry 将 Koog Agent 追踪到 Langfuse

[在 GitHub 上打开](https://github.com/JetBrains/koog/blob/develop/examples/notebooks/Langfuse.ipynb)
 [下载 .ipynb](https://raw.githubusercontent.com/JetBrains/koog/develop/examples/notebooks/Langfuse.ipynb)

本 notebook 展示如何使用 OpenTelemetry 将 Koog agent 的 trace 导出到你的 Langfuse 实例。你将设置环境变量、运行一个简单的 agent，然后在 Langfuse 中检查 span 和 trace。

## 你将学到什么

- Koog 如何与 OpenTelemetry 集成来发出 trace
- 如何通过环境变量配置 Langfuse exporter
- 如何运行 agent 并在 Langfuse 中查看它的 trace

## 前提条件

- 一个 Langfuse 项目（host URL、public key、secret key）
- 用于 LLM executor 的 OpenAI API key
- 在 shell 中设置的环境变量：

```
export OPENAI_API_KEY=sk-...
export LANGFUSE_HOST=https://cloud.langfuse.com # or your self-hosted URL
export LANGFUSE_PUBLIC_KEY=pk_...
export LANGFUSE_SECRET_KEY=sk_...
```

```
%useLatestDescriptors
//%use koog
```

```
import ai.koog.agents.core.agent.AIAgent
import ai.koog.agents.features.opentelemetry.feature.OpenTelemetry
import ai.koog.prompt.executor.clients.openai.OpenAIModels
import ai.koog.prompt.executor.llms.all.simpleOpenAIExecutor

/**
 * Example of Koog agents tracing to [Langfuse](https://langfuse.com/)
 *
 * Agent traces are exported to:
 * - Langfuse OTLP endpoint instance using [OtlpHttpSpanExporter]
 *
 * To run this example:
 *  1. Set up a Langfuse project and credentials as described [here](https://langfuse.com/docs/get-started#create-new-project-in-langfuse)
 *  2. Get Langfuse credentials as described [here](https://langfuse.com/faq/all/where-are-langfuse-api-keys)
 *  3. Set `LANGFUSE_HOST`, `LANGFUSE_PUBLIC_KEY`, and `LANGFUSE_SECRET_KEY` environment variables
 *
 * @see <a href="https://langfuse.com/docs/opentelemetry/get-started#opentelemetry-endpoint">Langfuse OpenTelemetry Docs</a>
 */
val agent = AIAgent(
    executor = simpleOpenAIExecutor(System.getenv("OPENAI_API_KEY")),
    llmModel = OpenAIModels.Chat.GPT4oMini,
    systemPrompt = "You are a code assistant. Provide concise code examples."
) {
    install(OpenTelemetry) {
        addLangfuseExporter()
    }
}
```

## 配置 agent 和 Langfuse exporter

在下一个单元中，我们会：

- 创建一个使用 OpenAI 作为 LLM executor 的 AIAgent
- 安装 OpenTelemetry feature 并添加 Langfuse exporter
- 依赖环境变量进行 Langfuse 配置

在底层，Koog 会为 agent 生命周期、LLM 调用和工具执行（如果有）发出 span。Langfuse exporter 会通过 OpenTelemetry endpoint 将这些 span 发送到你的 Langfuse 实例。

```
import kotlinx.coroutines.runBlocking

println("Running agent with Langfuse tracing")

runBlocking {
    val result = agent.run("Tell me a joke about programming")
    "Result: $result\nSee traces on the Langfuse instance"
}
```

## 运行 agent 并查看 trace

执行下一个单元来触发一个简单的 prompt。这会生成 span，并将它们导出到你的 Langfuse 项目。

### 在 Langfuse 中查看的位置

1. 打开你的 Langfuse dashboard 并选择你的项目
2. 进入 Traces/Spans 视图
3. 查找你运行此单元时附近的最新条目
4. 下钻到 span 以查看：
5. Agent 生命周期事件
6. LLM request/response metadata
7. 错误（如果有）

### 故障排除

- 没有显示 trace？
- 仔细检查 LANGFUSE\_HOST、LANGFUSE\_PUBLIC\_KEY、LANGFUSE\_SECRET\_KEY
- 确保你的网络允许向 Langfuse endpoint 发起出站 HTTPS 连接
- 验证你的 Langfuse 项目处于活动状态，并且 key 属于正确的项目
- 认证错误
- 在 Langfuse 中重新生成 key 并更新环境变量
- OpenAI 问题
- 确认 OPENAI\_API\_KEY 已设置且有效