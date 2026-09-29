# 预定义节点和组件

节点是 Koog 框架中智能体工作流的基本构建块。
每个节点代表工作流中的特定操作或转换，它们可以通过边连接以定义执行流程。

通常，节点允许你将复杂逻辑封装到可重用的组件中，这些组件可以轻松集成到
不同的智能体工作流中。本指南将引导你了解可在智能体
策略中使用的现有节点。

每个节点本质上是一个函数（Kotlin）或操作（Java），它接受特定类型的输入并返回特定类型的输出。
```
graph LR
    in:::hidden
    out:::hidden

    subgraph node ["Node"]
        execute(Do stuff)
    end

    in --Input--> execute --Output--> out

    classDef hidden display: none;
```
以下是如何定义一个节点，该节点期望输入一个字符串，并返回字符串的长度（一个整数）作为输出：

KotlinJava
```
val nodeLength by node<String, Int> { input ->
    input.length
}
```

```
var nodeLength = AIAgentNode.builder("nodeLength")
    .withInput(String.class)
    .withOutput(Integer.class)
    .withAction((input, ctx) -> input.length())
    .build();
```
更多信息，请参见 [node()](https://api.koog.ai/agents/agents-core/ai.koog.agents.core.dsl.builder/node.html)（Kotlin）或 [AIAgentNode.builder()](https://api.koog.ai/agents/agents-core/ai.koog.agents.core.agent.entity/-a-i-agent-node/-companion/builder.html)（Java）。

## 实用节点

### 直通节点

一个简单的直通节点，不执行任何操作，并将输入作为输出返回。有关详细信息，请参见 [nodeDoNothing](https://api.koog.ai/agents/agents-core/ai.koog.agents.core.dsl.extension/node-do-nothing.html)（Kotlin）或 [AIAgentNode.doNothing()](https://api.koog.ai/agents/agents-core/ai.koog.agents.core.agent.entity/-a-i-agent-node/-companion/do-nothing.html)（Java）。
```
graph LR
    in:::hidden
    out:::hidden

    subgraph node ["Pass-through node"]
        execute(Do nothing)
    end

    in ---|T| execute --T--> out

    classDef hidden display: none;
```
你可以将此节点用于以下目的：

- 在图中创建一个占位节点。
- 创建一个连接点而不修改数据。

以下是一个示例：

KotlinJava
```
val passthrough by nodeDoNothing<String>("passthrough")

edge(nodeStart forwardTo passthrough)
edge(passthrough forwardTo nodeFinish)
```

```
var passthrough = AIAgentNode.builder("passthrough")
    .withInput(String.class)
    .withOutput(String.class)
    .withAction((input, ctx) -> input)
    .build();

strategy.edge(strategy.nodeStart, passthrough);
strategy.edge(passthrough, strategy.nodeFinish);
```
## LLM 节点

### 提示词准备节点

**一个使用提供的提示词构建器向 LLM 提示词添加消息的节点。
这对于在实际发起 LLM 请求之前修改对话上下文非常有用。** 有关详细信息，请参阅 [nodeAppendPrompt](https://api.koog.ai/agents/agents-core/ai.koog.agents.core.dsl.extension/node-append-prompt.html)（Kotlin）或 [AIAgentNode.appendPrompt()](https://api.koog.ai/agents/agents-core/ai.koog.agents.core.agent.entity/-a-i-agent-node-builder-with-input/append-prompt.html)（Java）。
```
graph LR
    in:::hidden
    out:::hidden

    subgraph node ["Prompt preparation node"]
        execute(Append prompt)
    end

    in ---|T| execute --T--> out

    classDef hidden display: none;
```
你可以将此节点用于以下目的：

- 向提示中添加系统指令。
- 将用户消息插入对话中。
- 为后续 LLM 请求准备上下文。

以下是一个示例：

KotlinJava
```
val firstNode by node<Input, Output> {
    // Transform input to output
}

val secondNode by node<Output, Output> {
    // Transform output to output
}

// Node will get the value of type Output as input from the previous node and path through it to the next node
val setupContext by nodeAppendPrompt<Output>("setupContext") {
    system("You are a helpful assistant specialized in Kotlin programming.")
    user("I need help with Kotlin coroutines.")
}

edge(firstNode forwardTo setupContext)
edge(setupContext forwardTo secondNode)
```

```
var firstNode = AIAgentNode.builder()
    .withInput(Input.class)
    .withOutput(Output.class)
    .withAction((input, ctx) -> {
        // Transform input to output
        return input;
    })
    .build();

var secondNode = AIAgentNode.builder()
    .withInput(Output.class)
    .withOutput(Output.class)
    .withAction((output, ctx) -> {
        // Transform output to output
        return output;
    })
    .build();

var setupContext = AIAgentNode.builder()
    .withInput(Output.class)
    .appendPrompt(prompt -> {
        prompt.system("You are a helpful assistant specialized in Kotlin programming.");
        prompt.user("I need help with Kotlin coroutines.");
    });

strategy.edge(firstNode, setupContext);
strategy.edge(setupContext, secondNode);
```
### 仅工具节点

一个将用户消息追加到 LLM 提示词并获取响应的节点，其中 LLM 只能调用工具。有关详细信息，请参阅 [nodeLLMSendMessageOnlyCallingTools](https://api.koog.ai/agents/agents-core/ai.koog.agents.core.dsl.extension/node-l-l-m-send-message-only-calling-tools.html)（Kotlin）或 [AIAgentNode.llmSendMessageOnlyCallingTools()](https://api.koog.ai/agents/agents-core/ai.koog.agents.core.agent.entity/-a-i-agent-node/-companion/llm-send-message-only-calling-tools.html)（Java）。
```
graph LR
    in:::hidden
    out:::hidden

    subgraph node ["Tool-only node"]
        execute(Request LLM expecting only tool calls)
    end

    in --String--> execute --Message.Response--> out

    classDef hidden display: none;
```
### 强制单工具使用节点

一个将用户消息追加到 LLM 提示词并强制 LLM 使用特定工具的节点。有关详细信息，请参阅 [nodeLLMSendMessageForceOneTool](https://api.koog.ai/agents/agents-core/ai.koog.agents.core.dsl.extension/node-l-l-m-send-message-force-one-tool.html)（Kotlin）或 [AIAgentNode.llmSendMessageForceOneTool()](https://api.koog.ai/agents/agents-core/ai.koog.agents.core.agent.entity/-a-i-agent-node/-companion/llm-send-message-force-one-tool.html)（Java）。
```
graph LR
    in:::hidden
    out:::hidden

    subgraph node ["Forced single tool use node"]
        execute(Request LLM expecting a specific tool call)
    end

    in --String--> execute --Message.Response--> out

    classDef hidden display: none;
```
### LLM 请求节点

该节点将用户消息追加到 LLM 提示中，并获取带有可选工具使用的响应。节点配置决定在处理消息期间是否允许工具调用。有关详细信息，请参阅 [nodeLLMRequest](https://api.koog.ai/agents/agents-core/ai.koog.agents.core.dsl.extension/node-l-l-m-request.html)（Kotlin）或 [AIAgentNode.llmRequest()](https://api.koog.ai/agents/agents-core/ai.koog.agents.core.agent.entity/-a-i-agent-node/-companion/llm-request.html)（Java）。
```
graph LR
    in:::hidden
    out:::hidden

    subgraph node ["LLM request node"]
        execute(Request LLM)
    end

    in --String--> execute --Message.Response--> out

    classDef hidden display: none;
```
你可以将此节点用于以下目的：

- 为当前提示生成 LLM 响应，并控制是否允许 LLM 生成工具调用。

以下是一个示例：

KotlinJava
```
val requestLLM by nodeLLMRequest("requestLLM")
edge(getUserQuestion forwardTo requestLLM)
```

```
var requestLLM = AIAgentNode.llmRequest("requestLLM");

strategy.edge(AIAgentEdge.builder()
    .from(getUserQuestion)
    .to(requestLLM)
    .build());
```
### 具有结构化响应的 LLM 请求节点

一个将用户消息追加到 LLM 提示中，并向 LLM 请求结构化数据且具备错误纠正能力的节点。有关详细信息，请参阅 [nodeLLMRequestStructured](https://api.koog.ai/agents/agents-core/ai.koog.agents.core.dsl.extension/node-l-l-m-request-structured.html)（Kotlin）或 [AIAgentNode.llmRequestStructured()](https://api.koog.ai/agents/agents-core/ai.koog.agents.core.agent.entity/-a-i-agent-node/-companion/llm-request-structured.html)（Java）。
```
graph LR
    in:::hidden
    out:::hidden

    subgraph node ["LLM request node, structured response"]
        execute(Request LLM structured)
    end

    in --String--> execute -- "Result&lt;StructuredResponse&gt;" --> out

    classDef hidden display: none;
```
### 带流式响应的 LLM 请求节点

一个将用户消息追加到 LLM 提示中，并在有或没有流数据转换的情况下流式传输 LLM 响应的节点。有关详细信息，请参阅 [nodeLLMRequestStreaming](https://api.koog.ai/agents/agents-core/ai.koog.agents.core.dsl.extension/node-l-l-m-request-streaming.html)（Kotlin）或 [AIAgentNode.llmRequestStreaming()](https://api.koog.ai/agents/agents-core/ai.koog.agents.core.agent.entity/-a-i-agent-node/-companion/llm-request-streaming.html)（Java）。
```
graph LR
    in:::hidden
    out:::hidden

    subgraph node ["LLM request node, streaming response"]
        execute(Request LLM streaming)
    end

    in --String--> execute --Flow--> out

    classDef hidden display: none;
```
### 支持多个响应的 LLM 请求节点

该节点将用户消息追加到 LLM 提示中，并在启用工具调用的情况下获取多个 LLM 响应。详情请参见 [nodeLLMRequest](https://api.koog.ai/agents/agents-core/ai.koog.agents.core.dsl.extension/node-l-l-m-request.html)（Kotlin）或 [AIAgentNode.llmRequest()](https://api.koog.ai/agents/agents-core/ai.koog.agents.core.agent.entity/-a-i-agent-node/-companion/llm-request.html)（Java）。
```
graph LR
    in:::hidden
    out:::hidden

    subgraph node ["LLM request node, multiple responses"]
        execute(Request LLM expecting multiple responses)
    end

    in --String--> execute -- "List&lt;Message.Response&gt;" --> out

    classDef hidden display: none;
```
你可以将此节点用于以下目的：

- 处理需要多次工具调用的复杂查询。
- 生成多个工具调用。
- 实现需要多个并行操作的工作流。

以下是一个示例：

KotlinJava
```
val requestLLMMultipleTools by nodeLLMRequest()
edge(getComplexUserQuestion forwardTo requestLLMMultipleTools)
```

```
var requestLLMMultipleTools = AIAgentNode.llmRequest("requestLLMMultipleTools");

strategy.edge(AIAgentEdge.builder()
    .from(getComplexUserQuestion)
    .to(requestLLMMultipleTools)
    .build());
```
### 历史压缩节点

一个将当前 LLM 提示（消息历史）压缩为摘要的节点，用简洁的摘要（TL;DR）替换消息。这对于通过压缩历史来减少 token 使用量、管理长对话非常有用。有关详细信息，请参阅 [nodeLLMCompressHistory](https://api.koog.ai/agents/agents-core/ai.koog.agents.core.dsl.extension/node-l-l-m-compress-history.html)（Kotlin）或 [AIAgentNode.llmCompressHistory()](https://api.koog.ai/agents/agents-core/ai.koog.agents.core.agent.entity/-a-i-agent-node/-companion/llm-compress-history.html)（Java）。
```
graph LR
    in:::hidden
    out:::hidden

    subgraph node ["History compression node"]
        execute(Compress current prompt)
    end

    in ---|T| execute --T--> out

    classDef hidden display: none;
```
要了解有关历史压缩的更多信息，请参阅[历史压缩](../history-compression/)。

你可以将此节点用于以下目的：

- 管理长对话以减少 token 使用量。
- 总结对话历史以保持上下文。
- 在长时间运行的代理中实现内存管理。

以下是一个示例：

KotlinJava
```
val compressHistory by nodeLLMCompressHistory<String>(
    "compressHistory",
    strategy = HistoryCompressionStrategy.FromLastNMessages(10),
    preserveMemory = true
)
edge(generateHugeHistory forwardTo compressHistory)
```

```
var compressHistory = AIAgentNode.llmCompressHistory("compressHistory")
    .withInput(String.class)
    .build();

strategy.edge(generateHugeHistory, compressHistory);
```
## 工具节点

### 工具执行节点

一个执行单个工具调用并返回其结果的节点。该节点用于处理 LLM 发起的工具调用。有关详细信息，请参阅 [nodeExecuteTool](https://api.koog.ai/agents/agents-core/ai.koog.agents.core.dsl.extension/node-execute-tool.html)（Kotlin）或 [AIAgentNode.executeTool()](https://api.koog.ai/agents/agents-core/ai.koog.agents.core.agent.entity/-a-i-agent-node/-companion/execute-tool.html)（Java）。
```
graph LR
    in:::hidden
    out:::hidden

    subgraph node ["Tool execution node"]
        execute(Execute tool call)
    end

    in --MessagePart.Tool.Call--> execute --ReceivedToolResult--> out

    classDef hidden display: none;
```
你可以将此节点用于以下目的：

- 执行 LLM 请求的工具。
- 处理响应 LLM 决策的特定操作。
- 将外部功能集成到 agent 工作流中。

以下是一个示例：

KotlinJava
```
val requestLLM by nodeLLMRequest()
val executeTool by nodeExecuteTools()
edge(requestLLM forwardTo executeTool onToolCalls { true })
```

```
var requestLLM = AIAgentNode.llmRequest("requestLLM");
var executeTool = AIAgentNode.executeTools("executeTool");

strategy.edge(AIAgentEdge.builder()
    .from(requestLLM)
    .to(executeTool)
    .onToolCalls()
    .build());
```
### 工具结果后续节点

一个将工具结果添加到提示中并请求 LLM 响应的节点。有关详细信息，请参阅 [nodeLLMSendToolResult](https://api.koog.ai/agents/agents-core/ai.koog.agents.core.dsl.extension/node-l-l-m-send-tool-result.html)（Kotlin）或 [AIAgentNode.llmSendToolResult()](https://api.koog.ai/agents/agents-core/ai.koog.agents.core.agent.entity/-a-i-agent-node/-companion/llm-send-tool-result.html)（Java）。
```
graph LR
    in:::hidden
    out:::hidden

    subgraph node ["Tool result follow-up node"]
        execute(Request LLM)
    end

    in --ReceivedToolResult--> execute --Message.Response--> out

    classDef hidden display: none;
```
你可以将此节点用于以下目的：

- 处理工具执行的结果。
- 基于工具输出生成响应。
- 在工具执行后继续对话。

以下是一个示例：

KotlinJava
```
val executeTool by nodeExecuteTools()
val sendToolResultToLLM by nodeLLMSendToolResults()
edge(executeTool forwardTo sendToolResultToLLM)
```

```
var executeTool = AIAgentNode.executeTools("executeTool");
var sendToolResultToLLM = AIAgentNode.llmSendToolResults("sendToolResultToLLM");

strategy.edge(executeTool, sendToolResultToLLM);
```
### 多工具执行节点

一个执行多个工具调用的节点。这些调用可以选择并行执行。有关详细信息，请参阅 [nodeExecuteMultipleTools](https://api.koog.ai/agents/agents-core/ai.koog.agents.core.dsl.extension/node-execute-multiple-tools.html)（Kotlin）或 [AIAgentNode.executeMultipleTools()](https://api.koog.ai/agents/agents-core/ai.koog.agents.core.agent.entity/-a-i-agent-node/-companion/execute-multiple-tools.html)（Java）。
```
graph LR
    in:::hidden
    out:::hidden

    subgraph node ["Multi-tool execution node"]
        execute(Execute multiple tool calls)
    end

    in -- "List&lt;MessagePart.Tool.Call&gt;" --> execute -- "List&lt;ReceivedToolResult&gt;" --> out

    classDef hidden display: none;
```
你可以将此节点用于以下目的：

- 并行执行多个工具。
- 处理需要多次工具执行的复杂工作流。
- 通过批量调用工具来优化性能。

以下是一个示例：

KotlinJava
```
val requestLLMMultipleTools by nodeLLMRequest()
val executeMultipleTools by nodeExecuteTools(parallel = true)
edge(requestLLMMultipleTools forwardTo executeMultipleTools onToolCalls { true })
```

```
var requestLLMMultipleTools = AIAgentNode.llmRequest("requestLLMMultipleTools");
var executeMultipleTools = AIAgentNode.executeTools("executeMultipleTools");

// Route tool calls from the assistant response to the tool-execution node
strategy.edge(AIAgentEdge.builder()
    .from(requestLLMMultipleTools)
    .to(executeMultipleTools)
    .onToolCalls()
    .build());
```
### 多工具结果后续节点

一个将多个工具结果添加到提示中并获取多个 LLM 响应的节点。有关详细信息，请参阅 [nodeLLMSendMultipleToolResults](https://api.koog.ai/agents/agents-core/ai.koog.agents.core.dsl.extension/node-l-l-m-send-multiple-tool-results.html)（Kotlin）或 [AIAgentNode.llmSendMultipleToolResults()](https://api.koog.ai/agents/agents-core/ai.koog.agents.core.agent.entity/-a-i-agent-node/-companion/llm-send-multiple-tool-results.html)（Java）。
```
graph LR
    in:::hidden
    out:::hidden

    subgraph node ["Multiple tool result follow-up node"]
        execute(Request LLM expecting multiple responses)
    end

    in -- "List&lt;ReceivedToolResult&gt;" --> execute -- "List&lt;Message.Response&gt;" --> out

    classDef hidden display: none;
```
你可以将此节点用于以下目的：

- 处理多个工具执行的结果。
- 生成多个工具调用。
- 实现具有多个并行操作的复杂工作流。

以下是一个示例：

KotlinJava
```
val executeTools by nodeExecuteTools(parallel = true)
val sendToolResultsToLLM by nodeLLMSendToolResults()
edge(executeTools forwardTo sendToolResultsToLLM)
```

```
var executeTools = AIAgentNode.executeTools("executeTools");
var sendToolResultsToLLM = AIAgentNode.llmSendToolResults("sendToolResultsToLLM");

strategy.edge(executeTools, sendToolResultsToLLM);
```
## 节点输出转换

该框架在 Kotlin 中提供了 `transform` 扩展函数，允许你创建节点的转换版本，对其输出应用转换。在 Java 中，你可以通过创建带有显式转换的中间节点来实现相同的结果。当你需要将节点的输出转换为不同的类型或格式，同时保留原始节点的功能时，这非常有用。
```
graph LR
    in:::hidden
    out:::hidden

    subgraph nodeWithTransform [transformed node]
        subgraph node ["node"]
            execute(Do stuff)
        end
        transform
    end

    in --Input--> execute --> transform --Output--> out

    classDef hidden display: none;
```
### 节点转换

在 Kotlin 中，[transform()](https://api.koog.ai/agents/agents-core/ai.koog.agents.core.dsl.builder/-a-i-agent-node-delegate/transform.html) 函数会创建一个新的 `AIAgentNodeDelegate`，它包装原始节点并对其输出应用转换函数。在 Java 中，你需要使用 `AIAgentNode.builder()` 和显式类型参数，手动将节点与转换逻辑组合起来。

KotlinJava
```
inline fun <reified T> AIAgentNodeDelegate<Input, Output>.transform(
    noinline transformation: suspend (Output) -> T
): AIAgentNodeDelegate<Input, T>
```

```
// In Java, you need to manually compose nodes
// with transformation logic using AIAgentNode.builder() and explicit type parameters.
// See the examples below for the Java approach to node transformations.
```
#### 自定义节点转换

将自定义节点的输出转换为不同的数据类型：

KotlinJava
```
val textNode by nodeDoNothing<String>("textNode").transform<Int> { text ->
    text.split(" ").filter { it.isNotBlank() }.size
}

edge(nodeStart forwardTo textNode)
edge(textNode forwardTo nodeFinish)
```

```
var textNode = AIAgentNode.builder("textNode")
    .withInput(String.class)
    .withOutput(Integer.class)
    .withAction((text, ctx) -> {
        String[] words = text.split(" ");
        int count = 0;
        for (String word : words) {
            if (!word.isBlank()) {
                count++;
            }
        }
        return count;
    })
    .build();

strategy.edge(strategy.nodeStart, textNode);
strategy.edge(textNode, strategy.nodeFinish);
```
#### 内置节点转换

转换内置节点的输出，例如 `nodeLLMRequest`（Kotlin）或 `AIAgentNode.llmRequest()`（Java）：

KotlinJava
```
val lengthNode by nodeLLMRequest("llmRequest").transform<Int> { assistantMessage ->
    assistantMessage.parts.filterIsInstance<MessagePart.Text>().joinToString("\n") { it.text }.length
}

edge(nodeStart forwardTo lengthNode)
edge(lengthNode forwardTo nodeFinish)
```

```
var llmRequest = AIAgentNode.llmRequest("llmRequest");
var lengthNode = AIAgentNode.builder("lengthNode")
    .withInput(Message.Assistant.class)
    .withOutput(Integer.class)
    .withAction((assistantMessage, ctx) -> {
        String text = assistantMessage.getParts().stream()
            .filter(p -> p instanceof MessagePart.Text)
            .map(p -> ((MessagePart.Text) p).getText())
            .collect(Collectors.joining());
        return text.length();
    })
    .build();

strategy.edge(AIAgentEdge.builder()
    .from(strategy.nodeStart)
    .to(llmRequest)
    .build());
strategy.edge(llmRequest, lengthNode);
strategy.edge(lengthNode, strategy.nodeFinish);
```
## 预定义子图

该框架提供了预定义子图，用于封装常用模式和流程。这些子图通过自动处理基础节点和边的创建，简化了复杂智能体策略的开发。Kotlin 和 Java 之间的 API 保持一致，Kotlin 使用 DSL 函数，Java 使用构建器方法。

通过使用预定义子图，你可以实现各种常见的流水线。以下是一个示例：

1. 准备数据。
2. 运行任务。
3. 验证任务结果。如果结果不正确，则返回步骤 2，并附带反馈消息以进行调整。

### 任务执行子图

一个使用提供的工具执行特定任务并返回结构化结果的子图。它支持多响应 LLM 交互（助手可能会产生多个响应，并与工具调用交错进行），并允许你控制工具调用的执行方式。在 Kotlin 中，使用 [subgraphWithTask()](https://api.koog.ai/agents/agents-core/ai.koog.agents.ext.agent/subgraph-with-task.html)；在 Java 中，使用 [AIAgentSubgraph.builder().withTask()](https://api.koog.ai/agents/agents-core/ai.koog.agents.core.agent.entity/-typed-a-i-agent-subgraph-builder/with-task.html)。

你可以将此子图用于以下目的：

- 创建处理更大工作流中特定任务的特殊组件。
- 使用清晰的输入和输出接口封装复杂逻辑。
- 配置特定于任务的工具、模型和提示词。
- 通过自动压缩管理对话历史。
- 开发结构化智能体工作流和任务执行流水线。
- 从 LLM 任务执行中生成结构化结果，包括具有多个助手响应和工具调用的流程。

该 API 允许你使用可选参数对执行进行微调：

- [runMode](https://api.koog.ai/agents/agents-core/ai.koog.agents.core.agent.entity/-subgraph-with-task-builder/run-mode.html)：控制任务期间工具调用的执行方式（默认按顺序执行）。当底层模型/执行器支持时，可使用此参数在不同工具执行策略之间切换。
- [assistantResponseRepeatMax](https://api.koog.ai/agents/agents-core/ai.koog.agents.core.agent.entity/-subgraph-with-task-builder/assistant-response-repeat-max.html)：限制在判定任务无法完成之前允许的助手响应数量（如果未提供，则默认为安全的内部限制）。

你可以以文本形式向子图提供任务，根据需要配置 LLM，并提供必要的工具，子图将处理并解决该任务。以下是一个示例：

KotlinJava
```
val processQuery by subgraphWithTask<String, String>(
    tools = listOf(searchTool, calculatorTool, weatherTool),
    llmModel = OpenAIModels.Chat.GPT4o,
    parallelTools = false,
    assistantResponseRepeatMax = 3,
) { userQuery ->
    """
    You are a helpful assistant that can answer questions about various topics.
    Please help with the following query:
    $userQuery
    """
}
```

```
var processQuery = AIAgentSubgraph.builder("processQuery")
    .limitedTools(List.of(searchTool, calculatorTool, weatherTool))
    .withInput(String.class)
    .withOutput(String.class)
    .withTask(userQuery ->
        "You are a helpful assistant that can answer questions about various topics.\n" +
        "Please help with the following query:\n" +
        userQuery)
    .parallelTools(false)
    .assistantResponseRepeatMax(3)
    .build();
```
### 带验证的任务执行子图

`subgraphWithTask` 的一个特殊版本，用于验证任务是否正确执行，并提供所遇到问题的详细信息。此子图适用于需要验证或质量检查的工作流。在 Kotlin 中，使用 [subgraphWithVerification()](https://api.koog.ai/agents/agents-core/ai.koog.agents.ext.agent/subgraph-with-verification.html)，在 Java 中，使用 `AIAgentSubgraph.builder().withVerification()`。

你可以将此子图用于以下目的：

- 验证任务执行的正确性。
- 在工作流中实现质量控制流程。
- 创建可自我验证的组件。
- 生成带有成功/失败状态和详细反馈的结构化验证结果。

该子图确保 LLM 在工作流结束时调用验证工具，以检查任务是否成功完成。它保证此验证作为最后一步执行，并返回一个 [CriticResult](https://api.koog.ai/agents/agents-core/ai.koog.agents.ext.agent/-critic-result/index.html)，指示任务是否成功完成并提供详细反馈。
以下是一个示例：

KotlinJava
```
val verifyCode by subgraphWithVerification<String>(
    tools = listOf(runTestsTool, analyzeTool, readFileTool),
    llmModel = AnthropicModels.Opus_4_6,
    parallelTools = false,
    assistantResponseRepeatMax = 3,
) { codeToVerify ->
    """
    You are a code reviewer. Please verify that the following code meets all requirements:
    1. It compiles without errors
    2. All tests pass
    3. It follows the project's coding standards

    Code to verify:
    $codeToVerify
    """
}
```

```
var verifyCode = AIAgentSubgraph.builder("verifyCode")
    .limitedTools(List.of(runTestsTool, analyzeTool, readFileTool))
    .withInput(String.class)
    .withVerification(codeToVerify ->
        "You are a code reviewer. Please verify that the following code meets all requirements:\n" +
        "1. It compiles without errors\n" +
        "2. All tests pass\n" +
        "3. It follows the project's coding standards\n\n" +
        "Code to verify:\n" +
        codeToVerify)
    .parallelTools(false)
    .assistantResponseRepeatMax(3)
    .build();
```
## 预定义策略和常见策略模式

Koog 提供了组合各种节点的预定义策略。
这些节点通过边连接以定义操作流程，并使用条件指定何时沿每条边执行。

如有需要，你可以将这些策略集成到你的智能体工作流中。

### 单次运行策略

单次运行策略专为非交互式用例设计，在这些用例中，智能体处理一次输入并
返回结果。

当你需要运行不需要复杂逻辑的简单流程时，可以使用此策略。

KotlinJava
```
public fun singleRunStrategy(): AIAgentGraphStrategy<String, String> = strategy("single_run") {
    val nodeCallLLM by nodeLLMRequest("sendInput")
    val nodeExecuteTool by nodeExecuteTools("nodeExecuteTool")
    val nodeSendToolResult by nodeLLMSendToolResults("nodeSendToolResult")

    edge(nodeStart forwardTo nodeCallLLM)
    edge(nodeCallLLM forwardTo nodeExecuteTool onToolCalls { true })
    edge(nodeCallLLM forwardTo nodeFinish onTextMessage { true })
    edge(nodeExecuteTool forwardTo nodeSendToolResult)
    edge(nodeSendToolResult forwardTo nodeFinish onTextMessage { true })
    edge(nodeSendToolResult forwardTo nodeExecuteTool onToolCalls { true })
}
```

```
public static AIAgentGraphStrategy<String, String> singleRunStrategy() {
    var strategy = AIAgentGraphStrategy.builder("single_run")
        .withInput(String.class)
        .withOutput(String.class);

    var nodeCallLLM = AIAgentNode.llmRequest("sendInput");
    var nodeExecuteTool = AIAgentNode.executeTools("nodeExecuteTool");
    var nodeSendToolResult = AIAgentNode.llmSendToolResults("nodeSendToolResult");

    strategy.edge(AIAgentEdge.builder()
        .from(strategy.nodeStart)
        .to(nodeCallLLM)
        .build());

    strategy.edge(AIAgentEdge.builder()
        .from(nodeCallLLM)
        .to(nodeExecuteTool)
        .onToolCalls()
        .build());

    strategy.edge(AIAgentEdge.builder()
        .from(nodeCallLLM)
        .to(strategy.nodeFinish)
        .onTextMessage()
        .build());

    strategy.edge(nodeExecuteTool, nodeSendToolResult);

    strategy.edge(AIAgentEdge.builder()
        .from(nodeSendToolResult)
        .to(strategy.nodeFinish)
        .onTextMessage()
        .build());

    strategy.edge(AIAgentEdge.builder()
        .from(nodeSendToolResult)
        .to(nodeExecuteTool)
        .onToolCalls()
        .build());

    return strategy.build();
}
```
### 基于工具的策略

基于工具的策略专为高度依赖工具执行特定操作的工作流而设计。
它通常根据 LLM 的决策执行工具并处理结果。

KotlinJava
```
fun toolBasedStrategy(name: String, toolRegistry: ToolRegistry): AIAgentGraphStrategy<String, String> {
    return strategy(name) {
        val nodeSendInput by nodeLLMRequest()
        val nodeExecuteTool by nodeExecuteTools()
        val nodeSendToolResult by nodeLLMSendToolResults()

        // Define the flow of the agent
        edge(nodeStart forwardTo nodeSendInput)

        // If the LLM responds with a message, finish
        edge(
            (nodeSendInput forwardTo nodeFinish)
                    onTextMessage { true }
        )

        // If the LLM calls a tool, execute it
        edge(
            (nodeSendInput forwardTo nodeExecuteTool)
                    onToolCalls { true }
        )

        // Send the tool result back to the LLM
        edge(nodeExecuteTool forwardTo nodeSendToolResult)

        // If the LLM calls another tool, execute it
        edge(
            (nodeSendToolResult forwardTo nodeExecuteTool)
                    onToolCalls { true }
        )

        // If the LLM responds with a message, finish
        edge(
            (nodeSendToolResult forwardTo nodeFinish)
                    onTextMessage { true }
        )
    }
}
```

```
public static AIAgentGraphStrategy<String, String> toolBasedStrategy(String name, ToolRegistry toolRegistry) {
    var strategy = AIAgentGraphStrategy.builder(name)
        .withInput(String.class)
        .withOutput(String.class);

    var nodeSendInput = AIAgentNode.llmRequest("nodeSendInput");
    var nodeExecuteTool = AIAgentNode.executeTools("nodeExecuteTool");
    var nodeSendToolResult = AIAgentNode.llmSendToolResults("nodeSendToolResult");

    // Define the flow of the agent
    strategy.edge(AIAgentEdge.builder()
        .from(strategy.nodeStart)
        .to(nodeSendInput)
        .build());

    // If the LLM responds with a message, finish
    strategy.edge(AIAgentEdge.builder()
        .from(nodeSendInput)
        .to(strategy.nodeFinish)
        .onTextMessage()
        .build());

    // If the LLM calls a tool, execute it
    strategy.edge(AIAgentEdge.builder()
        .from(nodeSendInput)
        .to(nodeExecuteTool)
        .onToolCalls(call -> true)
        .build());

    // Send the tool result back to the LLM
    strategy.edge(nodeExecuteTool, nodeSendToolResult);

    // If the LLM calls another tool, execute it
    strategy.edge(AIAgentEdge.builder()
        .from(nodeSendToolResult)
        .to(nodeExecuteTool)
        .onToolCalls()
        .build());

    // If the LLM responds with a message, finish
    strategy.edge(AIAgentEdge.builder()
        .from(nodeSendToolResult)
        .to(strategy.nodeFinish)
        .onTextMessage()
        .build());

    return strategy.build();
}
```
### 流式数据策略

流式数据策略旨在处理来自 LLM 的流式数据。它通常会请求流式数据、处理这些数据，并可能使用处理后的数据调用工具。

KotlinJava
```
val agentStrategy = strategy<String, List<Book>>("library-assistant") {
    // Describe the node containing the output stream parsing
    val getMdOutput by node<String, List<Book>> { booksDescription ->
        val books = mutableListOf<Book>()
        val mdDefinition = markdownBookDefinition()

        llm.writeSession {
            appendPrompt { user(booksDescription) }
            // Initiate the response stream in the form of the definition `mdDefinition`
            val markdownStream = requestLLMStreaming(mdDefinition)
            // Call the parser with the result of the response stream and perform actions with the result
            parseMarkdownStreamToBooks(markdownStream).collect { book ->
                books.add(book)
                println("Parsed Book: ${book.title} by ${book.author}")
            }
        }

        books
    }
    // Describe the agent's graph making sure the node is accessible
    edge(nodeStart forwardTo getMdOutput)
    edge(getMdOutput forwardTo nodeFinish)
}
```

```
var strategy = AIAgentGraphStrategy.builder()
    .withInput(String.class)
    .withOutput(List.class);

var getMdOutput = AIAgentNode.builder()
    .withInput(String.class)
    .<List<Book>>withOutput(TypeToken.of(new TypeCapture<List<Book>>() {}))
    .withAction((booksDescription, ctx) -> {
        var books = new ArrayList<Book>();
        StructureDefinition mdDefinition = markdownBookDefinition();

        ctx.getLlm().writeSession(session -> {
            session.appendPrompt(prompt -> {
                prompt.user(booksDescription);
            });

            // Initiate the response stream in the form of the definition `mdDefinition`
            var markdownStream = session.requestLLMStreaming(mdDefinition);
            // Call the parser with the result of the response stream and perform actions with the result
            parseMarkdownStreamToBooks(markdownStream).subscribe(new Flow.Subscriber<>() {
                @Override
                public void onSubscribe(Flow.Subscription subscription) {
                }

                @Override
                public void onNext(Book book) {
                    books.add(book);
                    System.out.println("Parsed Book: " + book.getTitle() + " by " + book.getAuthor());
                }

                @Override
                public void onError(Throwable throwable) {
                }

                @Override
                public void onComplete() {
                }
            });

            return null;
        });

        return books;
    })
    .build();

strategy.edge(strategy.nodeStart, getMdOutput);
strategy.edge(getMdOutput, strategy.nodeFinish);
```
