# LLM-based planners

Beta

This feature is part of a beta module (`1.3.0-beta`). The API may change in future releases.
See [module versioning](../../../module-versioning/) for details.

LLM-based planners use LLMs to generate and evaluate plans.
They operate on a string-based state and execute steps through LLM requests.
String-based state means that the agent state is a single string.
At every step, the agent accepts an initial state string and returns the final state string as the result.

Prerequisites

Ensure your environment and project meet the following requirements:

- JDK 17+
- Kotlin 2.2.0+
- Gradle 8.0+ or Maven 3.8+

Add the [Koog package](https://central.sonatype.com/artifact/ai.koog/koog-agents/) as a dependency:

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

Get an API key from an LLM provider or run a local LLM via Ollama.
For more information, see [Quickstart](../quickstart.md).

Examples on this page assume that you have set the `OPENAI_API_KEY` environment variable.

Koog provides two simple planners:

- [SimpleLLMPlanner](https://api.koog.ai/agents/agents-planner/ai.koog.agents.planner.llm/-simple-l-l-m-planner/index.html)
  generates a plan only once at the very beginning and then follows the plan until it is completed.
  To include replanning, extend `SimpleLLMPlanner` and override the `assessPlan` method,
  indicating when the agent should replan.
- [SimpleLLMWithCriticPlanner](https://api.koog.ai/agents/agents-planner/ai.koog.agents.planner.llm/-simple-l-l-m-with-critic-planner/index.html)
  implements the `assessPlan` method that uses an LLM to check the validity of the plan via an LLM request
  and assess whether the agent should replan.

The following example shows how to create a simple planner agent using `SimpleLLMPlanner`:

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

## Next steps

- Learn about [GOAP agents](../goap-agents/)
