# LLM-based planners

Beta

此功能属于 beta 模块（`1.3.0-beta`）。API 可能会在未来版本中发生变化。
有关详细信息，请参阅[模块版本管理](../../../module-versioning/)。

LLM-based planners 使用 LLM 来生成和评估计划。
它们基于字符串形式的状态运行，并通过 LLM 请求执行步骤。
基于字符串的状态意味着 agent 状态是单个字符串。
在每一步中，agent 接受一个初始状态字符串，并将最终状态字符串作为结果返回。

前提条件

确保你的环境和项目满足以下要求：

- JDK 17+
- Kotlin 2.2.0+
- Gradle 8.0+ 或 Maven 3.8+

将 [Koog package](https://central.sonatype.com/artifact/ai.koog/koog-agents/) 添加为依赖：

Gradle (Kotlin)Gradle (Groovy)Maven

build.gradle.kts

```
dependencies {
    // Stable
    implementation("ai.koog:koog-agents:1.3.0")

    // Beta
    implementation("ai.koog:koog-agents-additions:1.3.0-beta")
}
```

build.gradle

```
dependencies {
    // Stable
    implementation 'ai.koog:koog-agents:1.3.0'

    // Beta
    implementation 'ai.koog:koog-agents-additions:1.3.0-beta'
}
```

pom.xml

```
<dependency>
    <!-- Stable -->
    <dependency>
        <groupId>ai.koog</groupId>
        <artifactId>koog-agents-jvm</artifactId>
        <version>1.3.0</version>
    </dependency>

    <!-- Beta -->
    <dependency>
        <groupId>ai.koog</groupId>
        <artifactId>koog-agents-additions-jvm</artifactId>
        <version>1.3.0-beta</version>
    </dependency>
</dependency>
```

从 LLM 提供商获取 API key，或通过 Ollama 运行本地 LLM。
更多信息请参阅 [Quickstart](../quickstart.md)。

本页示例假定你已设置 `OPENAI_API_KEY` 环境变量。

Koog 提供了两个简单的规划器：

- [SimpleLLMPlanner](https://api.koog.ai/agents/agents-planner/ai.koog.agents.planner.llm/-simple-l-l-m-planner/index.html)
  只在最开始生成一次计划，然后一直遵循该计划直到完成。
  如需包含重新规划，请扩展 `SimpleLLMPlanner` 并重写 `assessPlan` 方法，
  指明 agent 何时应该重新规划。
- [SimpleLLMWithCriticPlanner](https://api.koog.ai/agents/agents-planner/ai.koog.agents.planner.llm/-simple-l-l-m-with-critic-planner/index.html)
  实现了 `assessPlan` 方法，该方法通过 LLM 请求使用 LLM 检查计划的有效性，
  并评估 agent 是否应该重新规划。

以下示例展示如何使用 `SimpleLLMPlanner` 创建一个简单的 planner agent：

KotlinJava

```
// Create the planner
val planner = SimpleLLMPlanner()

// Wrap it in a planner strategy
val strategy = AIAgentPlannerStrategy(
    name = "simple-planner",
    planner = planner
)

// Configure the agent
val agentConfig = AIAgentConfig(
    prompt = prompt("planner") {
        system("You are a helpful planning assistant.")
    },
    model = OpenAIModels.Chat.GPT4o,
    maxAgentIterations = 50
)

// Create the planner agent
val agent = PlannerAIAgent(
    promptExecutor = simpleOpenAIExecutor(System.getenv("OPENAI_API_KEY")),
    strategy = strategy,
    agentConfig = agentConfig
)

suspend fun main() {
    // Run the agent with a task
    val result = agent.run("Create a plan to organize a team meeting")
    println(result)
}
```

```
// Create the planner strategy with LLM-based planner
AIAgentPlannerStrategy<String, String> strategy =
    Planners.llmBased("simple-planner")
        .build();

// Create the OpenAI executor
var promptExecutor = PromptExecutor.builder()
    .openAI("OPENAI_API_KEY")
    .build();

// Create the planner agent using AIAgent builder
AIAgent<String, String> agent = AIAgent.builder()
    .plannerStrategy(strategy)
    .promptExecutor(promptExecutor)
    .llmModel(OpenAIModels.Chat.GPT4o)
    .systemPrompt("You are a helpful planning assistant.")
    .maxIterations(50)
    .build();

// Run the agent with a task
String result = agent.run("Create a plan to organize a team meeting");
System.out.println(result);
```

## 下一步

- 了解 [GOAP agents](../goap-agents/)