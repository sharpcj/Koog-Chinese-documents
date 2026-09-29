# W&B Weave 导出器

Koog 使用 [OpenTelemetry](https://opentelemetry.io/) 发出智能体跟踪数据；OpenTelemetry 是用于可观测性数据的开放标准。
要将这些跟踪发送到 [W&B Weave](https://wandb.ai/site/weave/)，Koog 内置了 OpenTelemetry 导出器 ——
无需手动埋点。

连接后，Weave 的 [OpenTelemetry 支持](https://weave-docs.wandb.ai/guides/tracking/otel/) 可让你可视化、
分析并调试智能体如何与 LLM、工具和外部 API 交互。

---

## 设置说明

1. 在 <https://wandb.ai> 创建 W&B 账号。
2. 从 <https://wandb.ai/authorize> 获取你的 API key。
3. 在 [W&B Dashboard](https://wandb.ai/home) 中找到你的 entity 名称 —— 对个人账号，它与你的用户名一致；对共享工作区，则为团队/组织名称。
4. 选择项目名称。如果项目尚不存在，发送第一条 trace 时会自动创建。
5. 提供 entity、项目名称和 API key —— 可以作为参数传给 [`addWeaveExporter()`](https://api.koog.ai/agents/agents-features/agents-features-opentelemetry/ai.koog.agents.features.opentelemetry.integration.weave/add-weave-exporter.html)，也可以通过环境变量提供：

```
export WEAVE_API_KEY="<your-api-key>"
export WEAVE_ENTITY="<your-entity>"
export WEAVE_PROJECT_NAME="koog-tracing"
```

## 配置

安装 **OpenTelemetry feature** 并调用 [`addWeaveExporter()`](https://api.koog.ai/agents/agents-features/agents-features-opentelemetry/ai.koog.agents.features.opentelemetry.integration.weave/add-weave-exporter.html) 来启用 Weave 导出。

### 基本示例

KotlinJava

```
fun main() = runBlocking {
    val entity = System.getenv()["WEAVE_ENTITY"] 
        ?: throw IllegalArgumentException("WEAVE_ENTITY is not set")

    val projectName = System.getenv()["WEAVE_PROJECT_NAME"] 
        ?: "koog-tracing"

    val agent = AIAgent(
        promptExecutor = promptExecutor,
        llmModel = OpenAIModels.Chat.GPT4oMini,
        systemPrompt = "You are a code assistant. Provide concise code examples."
    ) {
        install(OpenTelemetry) {
            addWeaveExporter()
        }
    }

    println("Running agent with Weave tracing")

    val result = agent.run("Tell me a joke about programming")
    println("Result: $result\nSee traces on https://wandb.ai/$entity/$projectName/weave/traces")
}
```

```
public static void main(String[] args) {
    var entity = Optional.ofNullable(System.getenv("WEAVE_ENTITY"))
        .filter(env -> !env.isBlank())
        .orElseThrow(() -> new IllegalArgumentException("WEAVE_ENTITY is not set"));

    var projectName = Optional.ofNullable(System.getenv("WEAVE_PROJECT_NAME"))
        .filter(env -> !env.isBlank())
        .orElse("koog-tracing");

    var agent = AIAgent.builder()
        .promptExecutor(promptExecutor)
        .llmModel(OpenAIModels.Chat.GPT4oMini)
        .systemPrompt("You are a helpful assistant.")
        .install(OpenTelemetry.Feature, config ->
            WeaveKt.addWeaveExporter(
                config,
                null,        // weaveOtelBaseUrl: falls back to WEAVE_URL, defaults to https://trace.wandb.ai
                entity,
                projectName  // remaining params (apiKey, timeout) use defaults
            )
        )
        .build();

    System.out.println("Running agent with Weave tracing");

    var result = agent.run("Tell me a joke about programming");
    System.out.println("Result: " + result + "\nSee traces on https://wandb.ai/" + entity + "/" + projectName + "/weave/traces");
}
```

## 会跟踪什么

Weave 导出器会捕获与 Koog 通用 OpenTelemetry 集成相同的活动。
有关捕获的 span 的完整列表，以及如何包含 LLM prompt 和响应内容，请参见[会跟踪什么](../#what-gets-traced)。

在 W&B Weave 中可视化时，trace 如下所示：
![W&B Weave traces](../../../img/opentelemetry-weave-exporter-light.png#only-light)
![W&B Weave traces](../../../img/opentelemetry-weave-exporter-dark.png#only-dark)

有关更多详细信息，请参见官方 [Weave OpenTelemetry 文档](https://weave-docs.wandb.ai/guides/tracking/otel/)。

---

## 故障排除

- **没有 trace**：确认已设置 `WEAVE_API_KEY`、`WEAVE_ENTITY` 和 `WEAVE_PROJECT_NAME`，且你的 W&B 账号有权访问指定的 entity 和项目。
- **身份验证错误**：验证 `WEAVE_API_KEY` 有效，并且具有所选 entity 的写入权限。
- **连接问题**：确认你的环境可以访问 W&B 的 OpenTelemetry 摄取端点。

有关一般故障排除，请参见[故障排除](../#troubleshooting)。