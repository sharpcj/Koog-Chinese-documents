# 自定义策略图

策略图是 Koog 框架中智能体工作流的骨干。它们定义了智能体如何处理输入、与工具交互以及生成输出。策略图由通过边连接的节点组成，并通过条件决定执行流程。

创建策略图让你能够根据具体需求定制智能体的行为，无论你是在构建简单的聊天机器人、复杂的数据处理流水线，还是介于两者之间的任何东西。

## 策略图架构

从高层来看，策略图由以下组件组成：

- **策略（Strategy）**：图的顶层容器，使用 `strategy` 函数创建，并通过泛型参数指定输入和输出类型。
- **子图（Subgraphs）**：图中可以拥有自己的一组工具和上下文的部分。
- **节点（Nodes）**：工作流中的单个操作或转换。
- **边（Edges）**：节点之间的连接，用于定义转换条件和转换。

策略图从一个名为 `nodeStart` 的特殊节点开始，并在 `nodeFinish` 结束。
这些节点之间的路径由图中指定的边和条件决定。

## 策略图组件

### 节点

节点是策略图的构建块。每个节点代表一个特定的操作。

Koog 框架提供了预定义节点，同时也允许你使用 `node` 函数创建自定义节点。

有关详细信息，请参阅[预定义节点和组件](../nodes-and-components/)以及[自定义节点](../custom-nodes/)。

### 边

边连接节点并定义策略图中的操作流程。
边使用 `edge` 函数和 `forwardTo` 中缀函数创建：

KotlinJava
```
edge(sourceNode forwardTo targetNode)
```

```
strategy.edge(sourceNode, targetNode);
```
#### 条件

条件决定何时沿策略图中的特定边前进。条件有多种类型，以下是一些常见的类型：

| 条件类型 | 描述 |
| --- | --- |
| onCondition | 一种通用条件，接受一个返回布尔值的 lambda 表达式。 |
| onToolCalls | 当 LLM 调用一个或多个工具时匹配的条件。 |
| onTextMessage | 当 LLM 以文本消息响应时匹配的条件。 |
| onToolNotCalled | 当 LLM 未调用工具时匹配的条件。 |

你可以使用 `transformed` 函数在将输出传递给目标节点之前对其进行转换：

KotlinJava
```
edge(sourceNode forwardTo targetNode 
        onCondition { input -> input.length > 10 }
        transformed { input -> input.uppercase() }
)
```

```
strategy.edge(AIAgentEdge.builder()
    .from(sourceNode)
    .to(targetNode)
    .onCondition(input -> input.length() > 10)
    .transformed(input -> input.toUpperCase())
    .build());
```
### 子图

子图是策略图中独立运行的部分，拥有自己的一套工具和上下文。
策略图可以包含多个子图。每个子图通过使用 `subgraph` 函数来定义：

KotlinJava
```
val strategy = strategy<Input, Output>("strategy-name") {
    val firstSubgraph by subgraph<FirstInput, FirstOutput>("first") {
        // Define nodes and edges for this subgraph
    }
    val secondSubgraph by subgraph<SecondInput, SecondOutput>("second") {
        // Define nodes and edges for this subgraph
    }
}
```

```
var firstSubgraph = AIAgentSubgraph.builder("first")
    .withInput(FirstInput.class)
    .withOutput(FirstOutput.class)
    .define(subgraph -> {
        // Define nodes and edges for this subgraph
    })
    .build();

var secondSubgraph = AIAgentSubgraph.builder("second")
    .withInput(SecondInput.class)
    .withOutput(SecondOutput.class)
    .define(subgraph -> {
        // Define nodes and edges for this subgraph
    })
    .build();
```
子图可以使用工具注册表中的任何工具。
但是，你可以指定该注册表中可在子图中使用的工具子集，并将其作为参数传递给 `subgraph` 函数：

KotlinJava
```
val strategy = strategy<Input, Output>("strategy-name") {
    val firstSubgraph by subgraph<FirstInput, FirstOutput>(
        name = "first",
        tools = listOf(someTool)
    ) {
        // Define nodes and edges for this subgraph
    }
   // Define other subgraphs
}
```

```
var firstSubgraph = AIAgentSubgraph.builder("first")
    .withInput(FirstInput.class)
    .withOutput(FirstOutput.class)
    .limitedTools(someTools)
    .define(subgraph -> {
        // Define nodes and edges for this subgraph
    })
    .build();
```
## 基本策略图创建

基本策略图的运行方式如下：

1. 将输入发送给 LLM。
2. 如果 LLM 以消息响应，则结束流程。
3. 如果 LLM 调用工具，则运行该工具。
4. 将工具结果发送回 LLM。
5. 如果 LLM 以消息响应，则结束流程。
6. 如果 LLM 调用另一个工具，则运行该工具，然后从步骤 4 开始重复该过程。

![basic-strategy-graph](../img/basic-strategy-graph.png)

以下是一个基本策略图的示例：

KotlinJava
```
val myStrategy = strategy<String, String>("my-strategy") {
    val nodeCallLLM by nodeLLMRequest()
    val executeToolCall by nodeExecuteTools()
    val sendToolResult by nodeLLMSendToolResults()

    edge(nodeStart forwardTo nodeCallLLM)
    edge(nodeCallLLM forwardTo nodeFinish onTextMessage { true })
    edge(nodeCallLLM forwardTo executeToolCall onToolCalls { true })
    edge(executeToolCall forwardTo sendToolResult)
    edge(sendToolResult forwardTo nodeFinish onTextMessage { true })
    edge(sendToolResult forwardTo executeToolCall onToolCalls { true })
}
```

```
var graph = AIAgentGraphStrategy.builder("single_run")
    .withInput(String.class)
    .withOutput(String.class);

var nodeCallLLM = AIAgentNode.llmRequest("sendInput");
var nodeExecuteTool = AIAgentNode.executeTools("nodeExecuteTool");
var nodeSendToolResult = AIAgentNode.llmSendToolResults("nodeSendToolResult");

graph.edge(AIAgentEdge.builder()
    .from(graph.nodeStart)
    .to(nodeCallLLM)
    .build());

graph.edge(AIAgentEdge.builder()
    .from(nodeCallLLM)
    .to(nodeExecuteTool)
    .onToolCalls()
    .build());

graph.edge(AIAgentEdge.builder()
    .from(nodeCallLLM)
    .to(graph.nodeFinish)
    .onTextMessage()
    .build());

graph.edge(nodeExecuteTool, nodeSendToolResult);

graph.edge(AIAgentEdge.builder()
    .from(nodeSendToolResult)
    .to(graph.nodeFinish)
    .onTextMessage()
    .build());

graph.edge(AIAgentEdge.builder()
    .from(nodeSendToolResult)
    .to(nodeExecuteTool)
    .onToolCalls()
    .build());

var strategy = graph.build();
```
## 可视化策略图

在 JVM 上，你可以为策略图生成 [Mermaid 状态图](https://mermaid.js.org/syntax/stateDiagram.html)。

对于上一个示例中创建的图，你可以运行：

KotlinJava
```
val mermaidDiagram: String = myStrategy.asMermaidDiagram()

println(mermaidDiagram)
```

```
var mermaidDiagram = MermaidDiagramGenerator.INSTANCE.generate(myStrategy);
System.out.println(mermaidDiagram);
```
输出将是：
```
---
title: my-strategy
---
stateDiagram
    state "nodeCallLLM" as nodeCallLLM
    state "executeToolCall" as executeToolCall
    state "sendToolResult" as sendToolResult

    [*] --> nodeCallLLM
    nodeCallLLM --> [*] : transformed
    nodeCallLLM --> executeToolCall : onCondition
    executeToolCall --> sendToolResult
    sendToolResult --> [*] : transformed
    sendToolResult --> executeToolCall : onCondition
```
## 高级策略技术

### 历史压缩

对于长时间运行的对话，历史记录可能会变得很大并消耗大量 token。要了解如何压缩历史记录，请参阅[历史压缩](../history-compression/)。

### 并行工具执行

对于需要并行执行多个工具的工作流，你可以使用带有 `parallel = true` 的 `nodeExecuteTools` 节点：
```
val executeMultipleTools by nodeExecuteTools(parallel = true)
val processMultipleResults by nodeLLMSendToolResults()

edge(someNode forwardTo executeMultipleTools)
edge(executeMultipleTools forwardTo processMultipleResults)
```
你也可以使用 `toParallelToolCallsRaw` 扩展函数来处理流式数据：
```
parseMarkdownStreamToBooks(markdownStream).toParallelToolCallsRaw(BookTool::class).collect()
```
要了解更多信息，请参阅[工具](../tools/#parallel-tool-calls)。

### 并行节点执行

并行节点执行让你可以同时运行多个节点，从而提升性能并支持复杂的工作流。

要启动并行节点运行，请使用 `parallel` 方法：
```
val calc by parallel<String, Int>(
    nodeCalcTokens, nodeCalcSymbols, nodeCalcWords,
) {
    selectByMax { it }
}
```
上面的代码创建了一个名为 `calc` 的节点，它并行运行 `nodeCalcTokens`、`nodeCalcSymbols` 和 `nodeCalcWords` 节点，并将结果作为 `AsyncParallelResult` 的实例返回。

有关并行节点执行的更多信息及详细参考，请参见[并行节点执行](../parallel-node-execution/)。

### 条件分支

对于需要根据特定条件选择不同路径的复杂工作流，你可以使用条件分支：
```
val branchA by node<String, String> { input ->
    // Logic for branch A
    "Branch A: $input"
}

val branchB by node<String, String> { input ->
    // Logic for branch B
    "Branch B: $input"
}

edge(
    (someNode forwardTo branchA)
            onCondition { input -> input.contains("A") }
)
edge(
    (someNode forwardTo branchB)
            onCondition { input -> input.contains("B") }
)
```
## 最佳实践

创建自定义策略图时，请遵循以下最佳实践：

- 保持简单。从简单的图开始，根据需要再增加复杂度。
- 为节点和边赋予描述性名称，使图更易于理解。
- 处理所有可能的路径和边界情况。
- 使用各种输入测试你的图，确保其行为符合预期。
- 记录图的目的和行为，以供将来参考。
- 使用预定义策略或常见模式作为起点。
- 对于长时间运行的对话，使用历史压缩来减少 token 使用量。
- 使用子图来组织你的图并管理工具访问权限。

## 使用示例

### 语气分析策略

语气分析策略是一个很好的示例，展示了包含历史压缩的基于工具的策略：
```
fun toneStrategy(name: String, toolRegistry: ToolRegistry): AIAgentGraphStrategy<String, String> {
    return strategy(name) {
        val nodeSendInput by nodeLLMRequest()
        val nodeExecuteTool by nodeExecuteTools()
        val nodeSendToolResult by nodeLLMSendToolResults()
        val nodeCompressHistory by nodeLLMCompressHistory<ReceivedToolResults>()

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

        // If the history gets too large, compress it
        edge(
            (nodeExecuteTool forwardTo nodeCompressHistory)
                    onCondition { _ -> llm.readSession { prompt.messages.size > 100 } }
        )

        edge(nodeCompressHistory forwardTo nodeSendToolResult)

        // Otherwise, send the tool result directly
        edge(
            (nodeExecuteTool forwardTo nodeSendToolResult)
                    onCondition { _ -> llm.readSession { prompt.messages.size <= 100 } }
        )

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
该策略执行以下操作：

1. 将输入发送给 LLM。
2. 如果 LLM 返回一条消息，该策略结束流程。
3. 如果 LLM 调用工具，该策略运行该工具。
4. 如果历史记录过大（超过 100 条消息），该策略会在发送工具结果之前对其进行压缩。
5. 否则，该策略直接发送工具结果。
6. 如果 LLM 调用另一个工具，该策略运行它。
7. 如果 LLM 返回一条消息，该策略结束流程。

## 故障排除

在创建自定义策略图时，你可能会遇到一些常见问题。以下是一些故障排除提示：

### 图无法到达 finish 节点

如果你的图无法到达 finish 节点，请检查以下内容：

- 从起始节点出发的所有路径最终都通向 finish 节点。
- 你的条件限制性不会过强，以免阻止边的执行。
- 图中不存在没有退出条件的循环。

### 工具调用未运行

如果工具调用未运行，请检查以下内容：

- 工具已正确注册到工具注册表中。
- 从 LLM 节点到工具执行节点的边具有正确的条件（`onToolCall { true }`）。

### 历史记录变得过大

如果你的历史记录变得过大并消耗过多 token，请考虑以下做法：

- 添加一个历史记录压缩节点。
- 使用条件检查历史记录的大小，并在其变得过大时进行压缩。
- 使用更激进的压缩策略（例如，使用较小 N 值的 `FromLastNMessages`）。

### 图出现意外行为

如果你的图出现了意外的分支，请检查以下内容：

- 你的条件定义正确。
- 条件按预期顺序求值（边按定义顺序检查）。
- 你没有意外地用更通用的条件覆盖了其他条件。

### 出现性能问题

如果你的图存在性能问题，请考虑以下做法：

- 通过移除不必要的节点和边来简化图。
- 对独立操作使用并行工具执行。
- 压缩历史记录。
- 使用更高效的节点和操作。
