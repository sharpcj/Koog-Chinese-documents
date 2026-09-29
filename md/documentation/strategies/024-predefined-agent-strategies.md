# 预定义 agent 策略

为了让 agent 实现更容易，Koog 为常见 agent 用例提供了预定义 agent 策略。
可用的预定义策略如下：

- [Chat agent strategy](#chat-agent-strategy)
- [ReAct strategy](#react-strategy)

## Chat agent strategy

Chat agent strategy 旨在执行聊天交互流程。
它会编排不同阶段、节点和工具之间的交互，以聊天式方式处理用户输入、执行工具并提供响应。

### 概述

Chat agent strategy 实现了一种模式，其中 agent 会：

1. 接收用户输入
2. 使用 LLM 处理输入
3. 调用工具或提供直接响应
4. 处理工具结果并继续对话
5. 如果 LLM 尝试用纯文本响应而不是使用工具，则提供反馈

这种方法会创建一个对话式接口，agent 可以使用工具来满足用户请求。

### 设置和依赖

Koog 中的 Chat agent strategy 通过 `chatAgentStrategy` 函数实现。要在你的 agent 代码中使用该函数，请添加以下依赖导入：

```
ai.koog.agents.ext.agent.chatAgentStrategy
```

要使用该策略，请按以下模式创建 AI agent：

KotlinJava

```
val chatAgent = AIAgent(
    promptExecutor = promptExecutor,
    toolRegistry = toolRegistry,
    llmModel = model,
    // Set chatAgentStrategy as the agent strategy
    strategy = chatAgentStrategy()
)
```

```
AIAgent<String, String> chatAgent = AIAgent.builder()
    .promptExecutor(PromptExecutor.builder().openAI("OPENAI_API_KEY").build())
    .llmModel(OpenAIModels.Chat.O4Mini)
    .toolRegistry(ToolRegistry.builder().build())
    // Set chatAgentStrategy as the agent strategy
    .graphStrategy(AIAgentStrategies.chatAgentStrategy())
    .build();
```

### 何时使用 Chat agent strategy

Chat agent strategy 特别适用于：

- 构建需要使用工具的对话式 agent
- 创建可根据用户请求执行操作的助手
- 实现需要访问外部系统或数据的聊天机器人
- 你希望强制使用工具而不是纯文本响应的场景

### 示例

下面是一个 AI agent 的代码示例，它实现了预定义 Chat agent strategy（`chatAgentStrategy`），并提供了 agent 可使用的工具：

KotlinJava

```
val chatAgent = AIAgent(
    promptExecutor = promptExecutor,
    llmModel = model,
    // Use chatAgentStrategy as the agent strategy
    strategy = chatAgentStrategy(),
    // Add tools the agent can use
    toolRegistry = ToolRegistry {
        tool(searchTool)
        tool(weatherTool)
    }
)

suspend fun main() { 
    // Run the agent with a user query
    val result = chatAgent.run("What's the weather like today and should I bring an umbrella?")
}
```

```
// Add tools the agent can use
ToolRegistry toolRegistry = ToolRegistry.builder()
    .tools(new SearchAndWeatherTools())
    .build();

AIAgent<String, String> chatAgent = AIAgent.builder()
    .promptExecutor(PromptExecutor.builder().openAI("OPENAI_API_KEY").build())
    .llmModel(OpenAIModels.Chat.O4Mini)
    // Use chatAgentStrategy as the agent strategy
    .graphStrategy(AIAgentStrategies.chatAgentStrategy())
    .toolRegistry(toolRegistry)
    .build();

// Run the agent with a user query
String result = chatAgent.run("What's the weather like today and should I bring an umbrella?");
```

## ReAct strategy

ReAct（Reasoning and Acting）strategy 是一种 AI agent 策略，它在推理阶段和执行阶段之间交替，以动态处理任务并向大型语言模型（LLM）请求输出。

### 概述

ReAct strategy 实现了一种模式，其中 agent 会：

1. 对当前状态进行推理并规划下一步
2. 基于该推理采取行动
3. 观察这些行动的结果
4. 重复这一循环

这种方法结合了推理（逐步思考问题）和行动（执行工具以收集信息或执行操作）的优势。

### 流程图

下面是 ReAct strategy 的流程图：

![Koog flow diagram](../img/koog-react-diagram-light.png#only-light)
![Koog flow diagram](../img/koog-react-diagram-dark.png#only-dark)

### 设置和依赖

Koog 中的 ReAct strategy 通过 `reActStrategy` 函数实现。

要使用该策略，请按以下模式创建 AI agent：

KotlinJava

```
val reActAgent = AIAgent(
    promptExecutor = promptExecutor,
    toolRegistry = toolRegistry,
    llmModel = model,
    // Set reActStrategy as the agent strategy
    strategy = reActStrategy(
        // Set optional parameter values
        reasoningInterval = 1,
        name = "react_agent"
    )
)
```

```
AIAgent<String, String> reActAgent = AIAgent.builder()
    .promptExecutor(PromptExecutor.builder().openAI("OPENAI_API_KEY").build())
    .llmModel(OpenAIModels.Chat.O4Mini)
    .toolRegistry(ToolRegistry.builder().build())
    // Set reActStrategy as the agent strategy
    .graphStrategy(AIAgentStrategies.reActStrategy(
        // Set optional parameter values
        1, // reasoningInterval
        "react_agent" // name
    ))
    .build();
```

### 参数

`reActStrategy` 函数接受以下参数：

| Parameter | Type | Default | Description |
| --- | --- | --- | --- |
| `reasoningInterval` | Int | 1 | 指定推理步骤的间隔。必须大于 0。 |
| `name` | String | `re_act` | 策略名称。 |

### 示例用例

下面是 ReAct strategy 与一个简单银行 agent 配合工作的示例：

#### 1. 用户输入

用户发送初始 prompt。例如，它可以是这样的问题：`How much did I spend last month?`。

#### 2. 推理

agent 通过获取用户输入和推理 prompt 来执行初始推理。推理可能如下所示：

```
I need to follow these steps:
1. Get all transactions from last month
2. Filter out deposits (positive amounts)
3. Calculate total spending
```

#### 3. 行动和执行，第 1 阶段

根据 agent 在上一步定义的行动项，它会运行一个工具来获取上个月的所有交易。

在这种情况下，要运行的工具是 `get_transactions`，并带有已定义的 `startDate` 和 `endDate` 参数，这些参数与获取上个月所有交易的请求相匹配：

```
{tool: "get_transactions", args: {startDate: "2025-05-19", endDate: "2025-06-18"}}
```

该工具返回的结果可能如下：

```
[
  {date: "2025-05-25", amount: -100.00, description: "Grocery Store"},
  {date: "2025-05-31", amount: +1000.00, description: "Salary Deposit"},
  {date: "2025-06-10", amount: -500.00, description: "Rent Payment"},
  {date: "2025-06-13", amount: -200.00, description: "Utilities"}
]
```

#### 4. 推理

拿到工具返回的结果后，agent 再次执行推理，以确定流程中的后续步骤：

```
I have the transactions. Now I need to:
1. Remove the salary deposit of +1000.00
2. Sum up the remaining transactions
```

#### 5. 行动和执行，第 2 阶段

根据前一个推理步骤，agent 会调用 `calculate_sum` 工具，对作为工具参数提供的金额求和。由于推理还得出了从交易中移除正金额的行动点，因此作为工具参数提供的金额仅包含负数：

```
{tool: "calculate_sum", args: {amounts: [-100.00, -500.00, -200.00]}}
```

该工具返回最终结果：

```
-800.00
```

#### 6. 最终响应

agent 返回包含计算总和的最终响应（assistant message）：

```
You spent $800.00 last month on groceries, rent, and utilities.
```

### 何时使用 ReAct strategy

ReAct strategy 特别适用于：

- 需要多步推理的复杂任务
- agent 需要先收集信息再提供最终答案的场景
- 适合拆解为更小步骤的问题
- 同时需要分析性思考和工具使用的任务

### 示例

下面是一个 AI agent 的代码示例，它实现了预定义 ReAct strategy（`reActStrategy`），并提供了 agent 可使用的工具：

KotlinJava

```
val bankingAgent = AIAgent(
    promptExecutor = promptExecutor,
    llmModel = model,
    // Use reActStrategy as the agent strategy
    strategy = reActStrategy(
        reasoningInterval = 1,
        name = "banking_agent"
    ),
    // Add tools the agent can use
    toolRegistry = ToolRegistry {
        tool(getTransactions)
        tool(calculateSum)
    }
)

suspend fun main() { 
    // Run the agent with a user query
    val result = bankingAgent.run("How much did I spend last month?")
}
```

```
// Add tools the agent can use
ToolRegistry toolRegistry = ToolRegistry.builder()
    .tools(new BankingTools())
    .build();

AIAgent<String, String> bankingAgent = AIAgent.<String, String>builder()
    .promptExecutor(PromptExecutor.builder().openAI("OPENAI_API_KEY").build())
    .llmModel(OpenAIModels.Chat.O4Mini)
    // Use reActStrategy as the agent strategy
    .graphStrategy(AIAgentStrategies.reActStrategy(1, "banking_agent"))
    .toolRegistry(toolRegistry)
    .build();

// Run the agent with a user query
String result = bankingAgent.run("How much did I spend last month?");
```
