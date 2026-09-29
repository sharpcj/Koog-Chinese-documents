# Langfuse 导出器

Koog 使用 [OpenTelemetry](https://opentelemetry.io/) 发出智能体跟踪数据；OpenTelemetry 是用于可观测性数据的开放标准。
要将这些跟踪发送到 [Langfuse](https://langfuse.com/)，Koog 内置了 OpenTelemetry 导出器 ——
无需手动埋点。

连接后，Langfuse 的 [OpenTelemetry 支持](https://langfuse.com/integrations/native/opentelemetry) 可让你可视化、
分析并调试智能体如何与 LLM、工具和外部 API 交互。

---

## 设置说明

1. 使用[设置指南](https://langfuse.com/docs/get-started#create-new-project-in-langfuse)创建 Langfuse 项目。
2. 从 [Organization Settings > API Keys](https://langfuse.com/faq/all/where-are-langfuse-api-keys) 获取你的 `public key` 和 `secret key`。
3. 提供主机、公钥和密钥 —— 可以作为参数传给 [`addLangfuseExporter()`](https://api.koog.ai/agents/agents-features/agents-features-opentelemetry/ai.koog.agents.features.opentelemetry.integration.langfuse/add-langfuse-exporter.html)，也可以通过环境变量提供：

```
   export LANGFUSE_HOST="https://cloud.langfuse.com"
   export LANGFUSE_PUBLIC_KEY="<your-public-key>"
   export LANGFUSE_SECRET_KEY="<your-secret-key>"
```

## 配置

安装 **OpenTelemetry feature** 并调用 [`addLangfuseExporter()`](https://api.koog.ai/agents/agents-features/agents-features-opentelemetry/ai.koog.agents.features.opentelemetry.integration.langfuse/add-langfuse-exporter.html) 来启用 Langfuse 导出。

### 基本示例

KotlinJava

```
fun main() = runBlocking {
    val agent = AIAgent(
        promptExecutor = promptExecutor,
        llmModel = OpenAIModels.Chat.GPT4oMini,
        systemPrompt = "You are a code assistant. Provide concise code examples."
    ) {
        install(OpenTelemetry) {
            addLangfuseExporter()
        }
    }

    println("Running agent with Langfuse tracing")

    val result = agent.run("Tell me a joke about programming")
    println("Result: $result\nSee traces on the Langfuse instance")
}
```

```
public static void main(String[] args) {
    var agent = AIAgent.builder()
        .promptExecutor(promptExecutor)
        .llmModel(OpenAIModels.Chat.GPT4oMini)
        .systemPrompt("You are a code assistant. Provide concise code examples.")
        .install(OpenTelemetry.Feature, config ->
            LangfuseKt.addLangfuseExporter(config)
        )
        .build();

    System.out.println("Running agent with Langfuse tracing");

    var result = agent.run("Tell me a joke about programming");
    System.out.println("Result: " + result + "\nSee traces on the Langfuse instance");
}
```

## 跟踪属性

当 Koog 将智能体活动发送到 Langfuse 时，会以一系列 *span* 的形式发送 —— 它们是单项工作的记录，例如
一次 LLM 调用或一次工具执行。相关 span 会被分组为一个 *trace*，表示从开始到结束的一次完整智能体运行。

[`addLangfuseExporter()`](https://api.koog.ai/agents/agents-features/agents-features-opentelemetry/ai.koog.agents.features.opentelemetry.integration.langfuse/add-langfuse-exporter.html) 接受 `traceAttributes` 参数 —— 这是附加到每个 trace
根部的键值对列表。它们会启用 Langfuse 特定功能，例如会话、环境和标签，从而便于在 Langfuse UI 中筛选和分组 trace。

有关支持属性的完整列表，请参见 [Langfuse OpenTelemetry 文档](https://langfuse.com/integrations/native/opentelemetry#trace-level-attributes)。

常见的可包含属性：

- **Session ID** (`langfuse.session.id`)：将相关 trace 分组，用于聚合指标、成本分析和评分
- **Environment** (`langfuse.environment`)：将生产 trace 与开发和预发布环境隔离
- **Tags** (`langfuse.trace.tags`)：使用功能名称、实验 ID 或客户分群标记 trace（字符串数组）

### 带有会话和标签的示例

KotlinJava

```
fun main() = runBlocking {
    val sessionId = UUID.randomUUID().toString()

    val agent = AIAgent(
        promptExecutor = promptExecutor,
        llmModel = OpenAIModels.Chat.GPT4oMini,
        systemPrompt = "You are a helpful assistant."
    ) {
        install(OpenTelemetry) {
            addLangfuseExporter(
                traceAttributes = listOf(
                    CustomAttribute("langfuse.session.id", sessionId),
                    CustomAttribute("langfuse.trace.tags", listOf("chat", "kotlin", "production"))
                )
            )
        }
    }

    println("Running agent with Langfuse tracing")

    // Multiple runs with the same session ID will be grouped in Langfuse
    agent.run("What is Kotlin?")
    agent.run("Show me a coroutine example")
}
```

注意

目前不支持在 Java 中设置 `traceAttributes`，因为底层 Kotlin 函数带有 [`kotlin.time.Duration`](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.time/-duration/) 参数（一个 value class），这会导致所有重载（包括该参数之后的参数）发生 JVM 名称改编。当需要 `traceAttributes` 时，请使用上面的 Kotlin 示例。

## 会跟踪什么

Langfuse 导出器会捕获与 Koog 通用 OpenTelemetry 集成相同的活动。
它还会捕获 Langfuse 显示 [Agent Graphs](https://langfuse.com/docs/observability/features/agent-graphs) 所需的 span 属性。

有关捕获的 span 的完整列表，以及如何包含 LLM prompt 和响应内容，请参见[会跟踪什么](../#what-gets-traced)。

在 Langfuse 中可视化时，trace 如下所示：
![Langfuse traces](../../../img/opentelemetry-langfuse-exporter-light.png#only-light)
![Langfuse traces](../../../img/opentelemetry-langfuse-exporter-dark.png#only-dark)

有关 Langfuse OpenTelemetry tracing 的更多详细信息，请参见：  
[Langfuse OpenTelemetry 文档](https://langfuse.com/integrations/native/opentelemetry#opentelemetry-endpoint)。

---

## 故障排除

- **没有 trace**：确认已设置 `LANGFUSE_HOST`、`LANGFUSE_PUBLIC_KEY` 和 `LANGFUSE_SECRET_KEY`，且该密钥对属于正确的项目。
- **连接问题**：如果运行自托管 Langfuse，请确认你的环境可以访问 `LANGFUSE_HOST`。

有关一般故障排除，请参见[故障排除](../#troubleshooting)。