# Agents

AI agents 是能够进行推理、做出决策、与环境交互并采取行动以达成特定目标的自治系统。
在 Koog 中，AI agent 不只是 LLM 的一层封装；它是为 JVM 生态系统设计的结构化、类型安全的状态机。

Koog agents 围绕以下核心概念构建：

- [prompt executor](../prompts/prompt-executors/) 管理并执行 prompts，使 agent 能够与 LLMs 交互以进行推理和决策。
- [strategy](../nodes-and-components/) 定义 agent 的工作流。
  它可以采用有向图、函数或 planner 的形式。
  请参阅 [Agent types](#agent-types)。
- Agent 可以使用 [tools](../tools/) 与外部数据源和服务交互。
- 你可以使用 [features](../features/) 扩展和增强 AI agents 的功能。

提示

有关创建和运行最小 agent 的信息，请参阅[快速开始](../quickstart/)。

## Agent 类型

根据你需要执行的任务，Koog 提供了多种 agent 类型：

- [Basic agents](basic-agents/) 非常适合不需要任何自定义逻辑的简单任务。
  这些 agents 实现了适用于大多数常见使用场景的预定义 strategy。
- [Graph-based agents](graph-based-agents/) 提供对 agent 工作流、状态管理和可视化的完全控制与灵活性。
- [Functional agents](functional-agents/) 使你能够以函数形式快速原型化自定义逻辑，并可访问 agent 的上下文。
- [Planner agents](planner-agents/) 可以通过迭代循环自主规划和执行多步骤任务，直到达到期望的最终状态。

## Agent 配置

Agent 配置定义 agent 的执行参数，包括初始 prompt、语言模型和迭代限制。

提示

有关创建和运行最小 agent 的信息，请参阅[快速开始](../quickstart/)。

对于简单 agents，除了必需的 prompt executor 和语言模型之外，你还可以直接在 agent 构造函数中指定初始 system prompt 和一些其他参数：

KotlinJava

```
val agent = AIAgent(
    promptExecutor = simpleOpenAIExecutor(System.getenv("YOUR_API_KEY")),
    llmModel = OpenAIModels.Chat.GPT4o,
    systemPrompt = "You are a helpful assistant.",
    temperature = 0.7,
    maxIterations = 10
)
```

```
AIAgent<String, String> agent = AIAgent.builder()
    .promptExecutor(simpleOpenAIExecutor(System.getenv("YOUR_API_KEY")))
    .llmModel(OpenAIModels.Chat.GPT4o)
    .systemPrompt("You are a helpful assistant.")
    .temperature(0.7)
    .maxIterations(10)
    .build();
```

或者，你可以创建 [`AIAgentConfig`](https://api.koog.ai/agents/agents-core/ai.koog.agents.core.agent.config/-a-i-agent-config/index.html) 实例，以更细粒度地定义 agent 的行为和参数，然后将其传递给 agent 构造函数。
这使你能够定义包含多条消息、conversation history、LLM 参数以及其他执行参数的复杂 prompts。

KotlinJava

```
val agentConfig = AIAgentConfig(
    prompt = prompt(
        id = "assistant",
        params = LLMParams(
            temperature = 0.7
        )
    ) {
        system("You are a helpful assistant.")
    },
    model = OpenAIModels.Chat.GPT4o,
    maxAgentIterations = 10
)

val agent = AIAgent(
    promptExecutor = simpleOpenAIExecutor(System.getenv("OPENAI_API_KEY")),
    agentConfig = agentConfig
)
```

```
Prompt prompt = Prompt.builder("assistant")
    .system("You are a helpful assistant.")
    .build()
    .withParams(new LLMParams(
        0.7,         // temperature
        null,        // maxTokens
        1,           // numberOfChoices
        null,        // speculation
        null,        // schema
        LLMParams.ToolChoice.Auto.INSTANCE, // toolChoice
        null,        // user
        null         // additionalProperties
    ));

AIAgentConfig agentConfig = AIAgentConfig.builder(OpenAIModels.Chat.GPT4o)
    .prompt(prompt)
    .maxAgentIterations(10)
    .build();

AIAgent<String, String> agent = AIAgent.builder()
    .promptExecutor(simpleOpenAIExecutor(System.getenv("OPENAI_API_KEY")))
    .agentConfig(agentConfig)
    .build();
```

以下是 `AIAgentConfig` 的参数：

- `prompt` 定义初始 [prompt](../prompts/prompt-creation/) 和 [LLM 参数](../llm-parameters/)。
- `model` 指定 agent 交互使用的语言模型。
  你可以使用预定义模型之一，或[创建自定义模型配置](../model-capabilities/#creating-a-model-llmodel-configuration)。
- `maxAgentIterations` 限制 agent 在终止前可以执行的最大步骤数。
  每一步都是 agent 工作流中的一个 [node](../nodes-and-components/)。
- `missingToolsConversionStrategy` 定义 agent 执行期间处理缺失 tools 的策略。
- `responseProcessor` 可用于定义自定义 response processor。
  例如，它可以审核和验证响应内容、更改响应格式或记录响应。