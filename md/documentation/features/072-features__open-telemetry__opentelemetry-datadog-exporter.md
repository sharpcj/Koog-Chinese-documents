# Datadog 导出器

Koog 使用 [OpenTelemetry](https://opentelemetry.io/) 发出 agent trace；OpenTelemetry 是用于可观测性数据的开放标准。
为了将这些 trace 发送到 [Datadog](https://www.datadoghq.com/)，Koog 内置了一个 OpenTelemetry 导出器 —
无需手动埋点。

连接后，Datadog 的 [OpenTelemetry 支持](https://docs.datadoghq.com/opentelemetry/)允许你可视化、
分析和调试 agent 如何与 LLM、工具和外部 API 交互。

---

## 设置说明

1. 在 <https://www.datadoghq.com/> 创建 Datadog 账户
2. 从 [Organization Settings > API Keys](https://app.datadoghq.com/organization-settings/api-keys) 获取你的 API key
3. 提供你的 API key — 可以作为参数传给 [`addDatadogExporter()`](https://api.koog.ai/agents/agents-features/agents-features-opentelemetry/ai.koog.agents.features.opentelemetry.integration.datadog/add-datadog-exporter.html)，也可以通过环境变量提供：

   ```
   export DD_API_KEY="<your-api-key>"
   ```
4. （可选）如果要使用 US1（`datadoghq.com`）以外的 Datadog 区域，请将站点作为参数传给 [`addDatadogExporter()`](https://api.koog.ai/agents/agents-features/agents-features-opentelemetry/ai.koog.agents.features.opentelemetry.integration.datadog/add-datadog-exporter.html)，或设置环境变量：

   ```
   export DD_SITE="datadoghq.eu"
   ```

   支持的站点：

| 站点 | 区域 |
| --- | --- |
| `datadoghq.com` | US1（默认） |
| `datadoghq.eu` | EU1 |
| `us3.datadoghq.com` | US3 |
| `us5.datadoghq.com` | US5 |
| `ap1.datadoghq.com` | AP1（日本） |

## 配置

要启用 Datadog 导出，请安装 **OpenTelemetry 特性**并调用 [`addDatadogExporter()`](https://api.koog.ai/agents/agents-features/agents-features-opentelemetry/ai.koog.agents.features.opentelemetry.integration.datadog/add-datadog-exporter.html)。

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
            addDatadogExporter()
        }
    }

    println("Running agent with Datadog tracing")

    val result = agent.run("Tell me a joke about programming")
    println("Result: $result\nSee traces in Datadog LLM Observability")
}
```

```
public static void main(String[] args) {
    var agent = AIAgent.builder()
        .promptExecutor(promptExecutor)
        .llmModel(OpenAIModels.Chat.GPT4oMini)
        .systemPrompt("You are a code assistant. Provide concise code examples.")
        .install(OpenTelemetry.Feature, config ->
            DatadogKt.addDatadogExporter(config)
        )
        .build();

    System.out.println("Running agent with Datadog tracing");

    var result = agent.run("Tell me a joke about programming");
    System.out.println("Result: " + result + "\nSee traces in Datadog LLM Observability");
}
```

## Trace 属性

当 Koog 将 agent 活动发送到 Datadog 时，它会以一系列 *span* 的形式发送 — 即单个工作记录，例如
一次 LLM 调用或一次工具执行。相关 span 会组合成一个 *trace*，表示从开始到结束的一次完整 agent run。

[`addDatadogExporter()`](https://api.koog.ai/agents/agents-features/agents-features-opentelemetry/ai.koog.agents.features.opentelemetry.integration.datadog/add-datadog-exporter.html) 接受 `resourceAttributes` 参数 — 一个描述
发出 trace 的应用的键值对映射。这些属性会附加到每个 span 上，使你能够在
Datadog 中按环境或版本等属性轻松过滤和分组 trace。

常见的推荐属性：

- **env**：环境名称（例如 `production`、`staging` 或 `development`）
- **service.name**：你的服务或应用名称
- **version**：应用版本，便于比较不同部署之间的行为

### 带 trace 属性的示例

KotlinJava

```
fun main() = runBlocking {
    val agent = AIAgent(
        promptExecutor = promptExecutor,
        llmModel = OpenAIModels.Chat.GPT4oMini,
        systemPrompt = "You are a helpful assistant."
    ) {
        install(OpenTelemetry) {
            addDatadogExporter(
                url = "datadoghq.eu",  // Use EU region
                resourceAttributes = mapOf(
                    "env" to "production",
                    "service.name" to "my-agent",
                    "version" to "1.0.0"
                )
            )
        }
    }

    println("Running agent with Datadog tracing")

    agent.run("What is Kotlin?")
}
```

```
public static void main(String[] args) {
    var agent = AIAgent.builder()
        .promptExecutor(promptExecutor)
        .systemPrompt("You are a helpful assistant.")
        .llmModel(OpenAIModels.Chat.GPT4oMini)
        .install(OpenTelemetry.Feature, config ->
            DatadogKt.addDatadogExporter(
                config,
                null,                            // datadogApiKey: *** DD_API_KEY env var
                "datadoghq.eu"                   // url: use EU region
            ))
        .build();

    System.out.println("Running agent with Datadog tracing");

    agent.run("What is Kotlin?");
}
```

Note

目前不支持从 Java 设置 `resourceAttributes`，因为底层 Kotlin 函数带有 [`kotlin.time.Duration`](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.time/-duration/) 参数（值类），会导致其后包含参数的所有重载发生 JVM 名称改编。需要 `resourceAttributes` 时，请使用上面的 Kotlin 示例。

## 发送到多个后端

要同时将 trace 发送到 Datadog 和另一个后端，请通过
[`addDatadogExporter()`](https://api.koog.ai/agents/agents-features/agents-features-opentelemetry/ai.koog.agents.features.opentelemetry.integration.datadog/add-datadog-exporter.html)
注册 Datadog，并通过
[`addSpanExporter()`](https://api.koog.ai/agents/agents-features/agents-features-opentelemetry/ai.koog.agents.features.opentelemetry.feature/-open-telemetry-config/add-span-exporter.html)
添加第二个导出器。
每次调用都会注册一个独立的批量 span processor，因此两个后端会并行
导出：

Kotlin

```
install(OpenTelemetry) {
    addDatadogExporter()
    addSpanExporter(
        OtlpHttpSpanExporter.builder()
            .setEndpoint("http://localhost:4318/v1/traces")
            .build()
    )
}
```

## 跟踪哪些内容

Datadog 导出器捕获的活动与 Koog 通用 OpenTelemetry 集成相同。
有关捕获 span 的完整列表以及如何包含 LLM prompt 和响应内容，请参阅[跟踪哪些内容](../#跟踪哪些内容)。

有关 Datadog OpenTelemetry 支持的更多详情，请参阅 [Datadog OTLP API Intake](https://docs.datadoghq.com/opentelemetry/guide/otlp_api/)。

---

## 故障排除

- **没有 trace**：确认已正确设置 `DD_API_KEY` 和 `DD_SITE`（请参阅[设置说明](#设置说明)）。
- **认证错误**：确认你的 key 在 [Organization Settings > API Keys](https://app.datadoghq.com/organization-settings/api-keys) 中处于活动状态。
- **连接问题**：确认你的环境可以访问 `https://otlp.<DD_SITE>/v1/traces` — 例如 US1 对应 `https://otlp.datadoghq.com/v1/traces`

一般故障排除请参阅[故障排除](../#故障排除)。
