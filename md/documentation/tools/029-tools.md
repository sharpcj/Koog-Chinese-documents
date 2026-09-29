# 概述

智能体使用工具来执行特定任务或访问外部系统。

## 工具工作流

Koog 框架为在 Kotlin 和 Java 中使用工具提供以下工作流：

1. 创建自定义工具或使用一个内置工具。
2. 将工具添加到工具注册表。
3. 将工具注册表传递给智能体。
4. 通过智能体使用该工具。

### 可用的工具类型

Koog 框架中有三种工具类型：

- 内置工具，提供智能体与用户交互以及对话管理功能。详情请参阅 [Built-in tools](built-in-tools/)。
- 基于注解的自定义工具，可将函数作为工具暴露给 LLM。详情请参阅 [Annotation-based tools](annotation-based-tools/)。
- 自定义工具，可控制工具参数、元数据、执行逻辑，以及工具的注册和调用方式。详情请参阅 [Class-based
  tools](class-based-tools/)。

### 工具注册表

在智能体中使用工具之前，必须先将它添加到工具注册表。
工具注册表管理智能体可用的所有工具。

工具注册表的主要功能：

- 组织工具。
- 支持合并多个工具注册表。
- 提供按名称或类型检索工具的方法。

要了解更多信息，请参阅 [ToolRegistry](https://api.koog.ai/agents/agents-tools/ai.koog.agents.core.tools/-tool-registry/index.html)。

以下示例展示如何创建工具注册表并将工具添加到其中：

KotlinJava

```
val toolRegistry = ToolRegistry {
    tools(myTool)
}
```

```
// Create an instance of your ToolSet
MyToolSet myTool = new MyToolSet();

// Build the ToolRegistry and register tools from the ToolSet
ToolRegistry toolRegistry = ToolRegistry.builder()
    .tools(myTool)
    .build();
```

要合并多个工具注册表，请执行以下操作：

KotlinJava

```
val firstToolRegistry = ToolRegistry {
    tools(firstSampleTool)
}

val secondToolRegistry = ToolRegistry {
    tools(secondSampleTool)
}

val newRegistry = firstToolRegistry + secondToolRegistry
```

```
// Create instances of your ToolSets
FirstToolSet firstSampleTool = new FirstToolSet();
SecondToolSet secondSampleTool = new SecondToolSet();

// Build separate tool registries
ToolRegistry firstToolRegistry = ToolRegistry.builder()
    .tools(firstSampleTool)
    .build();

ToolRegistry secondToolRegistry = ToolRegistry.builder()
    .tools(secondSampleTool)
    .build();

ToolRegistry newRegistry = firstToolRegistry.plus(secondToolRegistry);
```

### 将工具传递给智能体

要让智能体使用某个工具，在创建智能体时需要提供包含该工具的工具注册表作为参数：

KotlinJava

```
// Agent initialization
val agent = AIAgent(
    promptExecutor = simpleOpenAIExecutor(System.getenv("OPENAI_API_KEY")),
    systemPrompt = "You are a helpful assistant with strong mathematical skills.",
    llmModel = OpenAIModels.Chat.GPT4o,
    // Pass your tool registry to the agent
    toolRegistry = toolRegistry
)
```

```
AIAgent<String, String> agent = AIAgent.builder()
    .promptExecutor(simpleOpenAIExecutor(System.getenv("OPENAI_API_KEY")))
    .systemPrompt("You are a helpful assistant with strong mathematical skills.")
    .llmModel(OpenAIModels.Chat.GPT4o)
    .toolRegistry(ToolRegistry.builder()
        .tools(secondSampleTool)
        .build()
    )
    .build();
```

### 调用工具

在智能体代码中调用工具有多种方式。推荐的方法是使用智能体上下文中提供的方法，而不是直接调用工具，因为这样可以确保在智能体环境中正确处理工具操作。

Tip

请确保已在工具中实现适当的[错误处理](../features/agent-event-handlers/)，以防止智能体失败。

工具在由 `AIAgentLLMWriteSession` 表示的特定会话上下文中调用。
它提供了多个调用工具的方法，使你可以：

- 使用给定参数调用工具。
- 按工具名称和给定参数调用工具。
- 按提供的工具类和参数调用工具。
- 使用给定参数调用指定类型的工具。
- 调用返回原始字符串结果的工具。

更多详情请参阅 [AIAgentLLMWriteSession](https://api.koog.ai/agents/agents-core/ai.koog.agents.core.agent.session/-a-i-agent-l-l-m-write-session/index.html) 的 API 参考。

#### 并行工具调用

你也可以使用 `toParallelToolCallsRaw` 扩展并行调用工具。例如：

KotlinJava

```
@Serializable
data class Book(
    val title: String,
    val author: String,
    val description: String
)

class BookTool() : SimpleTool<Book>(
    argsType = typeToken<Book>(),
    name = NAME,
    description = "A tool to parse book information from Markdown"
) {
    companion object {
        const val NAME = "book"
    }

    override suspend fun execute(args: Book): String {
        println("${args.title} by ${args.author}:\n ${args.description}")
        return "Done"
    }
}

val strategy = strategy<Unit, Unit>("strategy-name") {

    /*...*/
    val myNode by node<Unit, Unit> { _ ->
        llm.writeSession {
            flow {
                emit(Book("Book 1", "Author 1", "Description 1"))
            }.toParallelToolCallsRaw(BookTool::class).collect()
        }
    }
}
```

```

```

#### 从节点调用工具

构建带节点的智能体工作流时，可以使用特殊节点来调用工具：

- **nodeExecuteTool**：调用单个工具调用并返回其结果。详情请参阅 [API-reference](https://api.koog.ai/agents/agents-core/ai.koog.agents.core.dsl.extension/node-execute-tool.html)。
- **nodeExecuteSingleTool**：使用提供的参数调用特定工具。详情请参阅 [API-reference](https://api.koog.ai/agents/agents-core/ai.koog.agents.core.dsl.extension/node-execute-single-tool.html)。
- **nodeExecuteMultipleTools**：执行多个工具调用并返回其结果。详情请参阅 [API-reference](https://api.koog.ai/agents/agents-core/ai.koog.agents.core.dsl.extension/node-execute-multiple-tools.html)。
- **nodeLLMSendToolResult**：将工具结果发送给 LLM 并获取响应。详情请参阅 [API-reference](https://api.koog.ai/agents/agents-core/ai.koog.agents.core.dsl.extension/node-l-l-m-send-tool-result.html)。
- **nodeLLMSendMultipleToolResults**：将多个工具结果发送给 LLM。详情请参阅 [API-reference](https://api.koog.ai/agents/agents-core/ai.koog.agents.core.dsl.extension/node-l-l-m-send-multiple-tool-results.html)。

## 将智能体用作工具

该框架提供了将任意 AI 智能体转换为工具的能力，使其可供其他智能体使用。
此功能可让你创建分层智能体架构，在其中专门化智能体可以被更高层的编排智能体作为工具调用。

### 将智能体转换为工具

要将智能体转换为工具，请使用 `AIAgentService` 和 `createAgentTool()` 扩展函数：

KotlinJava

```
// Create a specialized agent service, responsible for creating financial analysis agents.
val analysisAgentService = AIAgentService(
    promptExecutor = simpleOpenAIExecutor(apiKey),
    llmModel = OpenAIModels.Chat.GPT4o,
    systemPrompt = "You are a financial analysis specialist.",
    toolRegistry = analysisToolRegistry
)

// Create a tool that would run financial analysis agent once called.
val analysisAgentTool = analysisAgentService.createAgentTool(
    agentName = "analyzeTransactions",
    agentDescription = "Performs financial transaction analysis",
    inputDescription = "Transaction analysis request",
    inputType = typeToken<String>(),
)
```

```

```

### 在其他智能体中使用智能体工具

转换为工具后，你可以将该智能体工具添加到另一个智能体的工具注册表中：

KotlinJava

```
// Create a coordinator agent that can use specialized agents as tools
val coordinatorAgent = AIAgent(
    promptExecutor = simpleOpenAIExecutor(apiKey),
    llmModel = OpenAIModels.Chat.GPT4o,
    systemPrompt = "You coordinate different specialized services.",
    toolRegistry = ToolRegistry {
        tool(analysisAgentTool)
        // Add other tools as needed
    }
)
```

```

```

### 智能体工具执行

当调用智能体工具时：

1. 根据输入描述符反序列化参数。
2. 使用反序列化后的输入执行被包装的智能体。
3. 将智能体的输出序列化，并作为工具结果返回。

### 将智能体用作工具的优势

- **模块化**：将复杂工作流拆分为专门化智能体。
- **可重用性**：在多个协调智能体中使用同一个专门化智能体。
- **关注点分离**：每个智能体都可以专注于自己的特定领域。