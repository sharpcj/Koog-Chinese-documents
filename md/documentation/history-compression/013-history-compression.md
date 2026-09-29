# History compression

AI Agent 会维护一份消息历史，其中包含用户消息、助手响应、工具调用和工具响应。
随着 Agent 执行其 strategy，每次交互都会使这份历史增长。

对于长时间运行的对话，历史可能变得很大并消耗大量 token。
History compression 通过将完整消息列表总结为一条或多条只包含后续 Agent 运行所需重要信息的消息，帮助减少这种消耗。

History compression 解决了 Agent 系统中的关键挑战：

- 优化上下文使用。更聚焦且更小的上下文可提升 LLM 性能，并防止超过 token 限制导致失败。
- 提升性能。压缩历史会减少 LLM 需要处理的消息数量，从而加快响应。
- 增强准确性。聚焦相关信息有助于 LLM 保持专注，并在不受干扰的情况下完成任务。
- 降低成本。减少无关消息可降低 token 使用量，从而降低 API 调用的总体成本。

## 何时压缩历史

History compression 会在 Agent 工作流的特定步骤执行：

- 在 Agent strategy 的逻辑步骤（subgraphs）之间。
- 当上下文变得过长时。

## History compression 实现

在你的 Agent 中实现 history compression 主要有两种方式：

- 在 strategy graph 中
- 在 custom node 中

### Strategy graph 中的 history compression

要在 strategy graph 中压缩历史，需要使用预定义节点，将当前消息历史压缩为简洁摘要：

- **Kotlin**：`nodeLLMCompressHistory`
- **Java**：`AIAgentNode.llmCompressHistory()`

有关更多信息和具体示例，请参阅 [History compression node](../nodes-and-components/#history-compression-node)。

根据你决定在哪个步骤执行压缩，可以使用以下场景：

- 当历史变得过长时压缩历史：在 edge 条件中检查消息数量，并添加 history compression node。要检查历史长度，请执行以下操作：
- **Kotlin**：定义一个 helper extension。
- **Java**：在 `.onCondition()` 中使用 inline lambda expressions。

KotlinJava

```
// Define that the history is too long if there are more than 100 messages
private suspend fun AIAgentContext.historyIsTooLong(): Boolean = llm.readSession { prompt.messages.size > 100 }

val strategy = strategy<String, String>("execute-with-history-compression") {
    val callLLM by nodeLLMRequest()
    val executeTool by nodeExecuteTools()
    val sendToolResult by nodeLLMSendToolResults()

    // Compress the LLM history and keep the current ReceivedToolResults for the next node
    val compressHistory by nodeLLMCompressHistory<ReceivedToolResults>()

    edge(nodeStart forwardTo callLLM)
    edge(callLLM forwardTo nodeFinish onTextMessage { true })
    edge(callLLM forwardTo executeTool onToolCalls { true })

    // Compress history after executing any tool if the history is too long 
    edge(executeTool forwardTo compressHistory onCondition { historyIsTooLong() })
    edge(compressHistory forwardTo sendToolResult)
    // Otherwise, proceed to the next LLM request
    edge(executeTool forwardTo sendToolResult onCondition { !historyIsTooLong() })

    edge(sendToolResult forwardTo executeTool onToolCalls { true })
    edge(sendToolResult forwardTo nodeFinish onTextMessage { true })
}
```

```
var graph = AIAgentGraphStrategy.builder("execute-with-history-compression")
    .withInput(String.class)
    .withOutput(String.class);

var callLLM = AIAgentNode.llmRequest(null);
var executeTool = AIAgentNode.executeTools(null);
var sendToolResult = AIAgentNode.llmSendToolResults(null);

// Compress the LLM history; the carried ReceivedToolResults flows into the next node.
var compressHistory = AIAgentNode
    .llmCompressHistory("compressHistory")
    .withInput(ReceivedToolResults.class)
    .build();

// Edge from start to callLLM
graph.edge(AIAgentEdge.builder()
    .from(graph.nodeStart)
    .to(callLLM)
    .build());

// Edge from callLLM to finish on text response
graph.edge(AIAgentEdge.builder()
    .from(callLLM)
    .to(graph.nodeFinish)
    .onTextMessage()
    .build());

// Edge from callLLM to executeTool on tool call
graph.edge(AIAgentEdge.builder()
    .from(callLLM)
    .to(executeTool)
    .onToolCalls()
    .build());

// Compress history after executing any tool if the history is too long
graph.edge(AIAgentEdge.builder()
    .from(executeTool)
    .to(compressHistory)
    .onCondition((message, ctx) ->
        ctx.getLlm().readSession(session ->
            session.getPrompt().getMessages().size() > 100
        )
    )
    .build());

graph.edge(compressHistory, sendToolResult);

// Otherwise, proceed to the next LLM request
graph.edge(AIAgentEdge.builder()
    .from(executeTool)
    .to(sendToolResult)
    .onCondition((message, ctx) ->
        ctx.getLlm().readSession(session ->
            session.getPrompt().getMessages().size() <= 100
        )
    )
    .build());

// Edge from sendToolResult to executeTool on tool call
graph.edge(AIAgentEdge.builder()
    .from(sendToolResult)
    .to(executeTool)
    .onToolCalls()
    .build());

// Edge from sendToolResult to finish on text response
graph.edge(AIAgentEdge.builder()
    .from(sendToolResult)
    .to(graph.nodeFinish)
    .onTextMessage()
    .build());
```

在此示例中，strategy 会在每次工具调用后检查历史是否过长。
历史会在将工具结果发送回 LLM 之前被压缩。这样可以防止长对话期间上下文不断增长。

- 要在 strategy 的逻辑步骤（subgraphs）之间压缩历史，可以按如下方式实现你的 strategy：

KotlinJava

```
val strategy = strategy<String, String>("execute-with-history-compression") {
    val collectInformation by subgraph<String, String> {
        // Some steps to collect the information
    }
    val compressHistory by nodeLLMCompressHistory<String>()
    val makeTheDecision by subgraph<String, String> {
        // Some steps to make the decision based on the current compressed history and collected information
    }

    nodeStart then collectInformation then compressHistory then makeTheDecision
}
```

```
var graph = AIAgentGraphStrategy.builder("execute-with-history-compression")
    .withInput(String.class)
    .withOutput(String.class);

// Subgraph to collect information
var collectInformation = AIAgentSubgraph.builder("collectInformation")
    .withInput(String.class)
    .withOutput(String.class)
    .limitedTools(Collections.emptyList())
    .withTask(input -> "Collect information based on: " + input)
    .build();

// Compress history after collecting information
var compressHistory = AIAgentNode
    .llmCompressHistory("compressHistory")
    .withInput(String.class)
    .build();

// Subgraph to make decision based on compressed history
var makeTheDecision = AIAgentSubgraph.builder("makeTheDecision")
    .withInput(String.class)
    .withOutput(String.class)
    .limitedTools(Collections.emptyList())
    .withTask(input -> "Make a decision based on the information")
    .build();

// Build the flow: start -> collectInformation -> compressHistory -> makeTheDecision -> finish
graph.edge(graph.nodeStart, collectInformation);
graph.edge(collectInformation, compressHistory);
graph.edge(compressHistory, makeTheDecision);
graph.edge(makeTheDecision, graph.nodeFinish);
```

在此示例中，历史会在完成信息收集阶段后、进入决策阶段之前被压缩。

### Custom node 中的 history compression

如果你正在实现 custom node，可以使用 `replaceHistoryWithTLDR()` 函数（Kotlin）压缩历史，如下所示：

Kotlin

```
llm.writeSession {
    replaceHistoryWithTLDR()
}
```

这种方式让你能够根据具体需求，在 custom node 逻辑中的任意位置更灵活地实现压缩。

要了解 custom nodes 的更多信息，请参阅 [Custom nodes](../custom-nodes/)。

## History compression strategies

你可以使用可选的 `strategy` 参数自定义压缩过程：

- **Kotlin**：将 strategy 传给 `nodeLLMCompressHistory(strategy=...)` 或 `replaceHistoryWithTLDR(strategy=...)`。
- **Java**：使用 `.compressionStrategy()` builder 方法。

框架提供了多个内置策略。

### WholeHistory (Default)

默认策略会将整个历史压缩为一条 TLDR 消息，用于总结到目前为止已完成的内容。
该策略适用于大多数通用场景：你希望在减少 token 使用量的同时，保持对整个对话上下文的感知。

可以按如下方式使用：

- 在 strategy graph 中：

KotlinJava

```
val compressHistory by nodeLLMCompressHistory<ProcessedInput>(
    strategy = HistoryCompressionStrategy.WholeHistory
)
```

```
// Using WholeHistory strategy in a compression node
var compressHistory = AIAgentNode
    .llmCompressHistory("compressHistory")
    .withInput(String.class)
    .compressionStrategy(HistoryCompressionStrategy.WholeHistory)
    .build();

// Note: This example only shows the node creation.
// You would need to add edges and other nodes to complete the graph.
```

- 在 custom node 中：

KotlinJava

```
llm.writeSession {
    replaceHistoryWithTLDR(strategy = HistoryCompressionStrategy.WholeHistory)
}
```

```
ctx.getLlm().writeSession(session -> {
    session.replaceHistoryWithTLDR(HistoryCompressionStrategy.WholeHistory);
    return null;
});
```

### FromLastNMessages

该策略只将最后 `n` 条消息压缩为一条 TLDR 消息，并完全丢弃更早的消息。
当只有 Agent 最新完成的事项（或最新发现的事实、最新上下文）与解决问题相关时，这很有用。

可以按如下方式使用：

- 在 strategy graph 中：

KotlinJava

```
val compressHistory by nodeLLMCompressHistory<ProcessedInput>(
    strategy = HistoryCompressionStrategy.FromLastNMessages(5)
)
```

```
// Using FromLastNMessages strategy to compress only the last 5 messages
var compressHistory = AIAgentNode
    .llmCompressHistory("compressHistory")
    .withInput(String.class)
    .compressionStrategy(HistoryCompressionStrategy.FromLastNMessages(5))
    .build();

// Note: This example only shows the node creation.
// You would need to add edges and other nodes to complete the graph.
```

- 在 custom node 中：

KotlinJava

```
llm.writeSession {
    replaceHistoryWithTLDR(strategy = HistoryCompressionStrategy.FromLastNMessages(5))
}
```

```
ctx.getLlm().writeSession(session -> {
    session.replaceHistoryWithTLDR(HistoryCompressionStrategy.FromLastNMessages(5));
    return null;
});
```

### Chunked

该策略会将完整消息历史拆分为固定大小的 chunks，并将每个 chunk 独立压缩为一条 TLDR 消息。
当你不仅需要了解到目前为止已完成内容的简洁 TLDR，还希望跟踪整体进展，并且某些较早信息也可能重要时，这很有用。

可以按如下方式使用：

- 在 strategy graph 中：

KotlinJava

```
val compressHistory by nodeLLMCompressHistory<ProcessedInput>(
    strategy = HistoryCompressionStrategy.Chunked(10)
)
```

```
// Using Chunked strategy to compress history in chunks of 10 messages
var compressHistory = AIAgentNode
    .llmCompressHistory("compressHistory")
    .withInput(String.class)
    .compressionStrategy(HistoryCompressionStrategy.Chunked(10))
    .build();

// Note: This example only shows the node creation.
// You would need to add edges and other nodes to complete the graph.
```

- 在 custom node 中：

KotlinJava

```
llm.writeSession {
    replaceHistoryWithTLDR(strategy = HistoryCompressionStrategy.Chunked(10))
}
```

```
ctx.getLlm().writeSession(session -> {
    session.replaceHistoryWithTLDR(HistoryCompressionStrategy.Chunked(10));
    return null;
});
```

### FactRetrievalHistoryCompressionStrategy

该策略会在历史中搜索与所提供概念列表相关的特定事实，并检索这些事实。
它会将整个历史改为仅包含这些事实，并将其作为后续 LLM 请求的上下文。
当你知道哪些确切事实会有助于 LLM 更好地执行任务时，这很有用。

可以按如下方式使用：

- 在 strategy graph 中：

KotlinJava

```
val compressHistory by nodeLLMCompressHistory<ProcessedInput>(
    strategy = FactRetrievalHistoryCompressionStrategy(
        Concept(
            keyword = "user_preferences",
            // Description to the LLM -- what specifically to search for
            description = "User's preferences for the recommendation system, including the preferred conversation style, theme in the application, etc.",
            // LLM would search for multiple relevant facts related to this concept:
            factType = FactType.MULTIPLE
        ),
        Concept(
            keyword = "product_details",
            // Description to the LLM -- what specifically to search for
            description = "Brief details about products in the catalog the user has been checking",
            // LLM would search for multiple relevant facts related to this concept:
            factType = FactType.MULTIPLE
        ),
        Concept(
            keyword = "issue_solved",
            // Description to the LLM -- what specifically to search for
            description = "Was the initial user's issue resolved?",
            // LLM would search for a single answer to the question:
            factType = FactType.SINGLE
        )
    )
)
```

```
// Using FactRetrievalHistoryCompressionStrategy strategy to extract specific facts
var compressHistory = AIAgentNode
    .llmCompressHistory("compressHistory")
    .withInput(ReceivedToolResult.class)
    .compressionStrategy(new FactRetrievalHistoryCompressionStrategy(
        new Concept(
            "user_preferences",
            "User's preferences for the recommendation system, including the preferred conversation style, theme in the application, etc.",
            FactType.MULTIPLE
        ),
        new Concept(
            "product_details",
            "Brief details about products in the catalog the user has been checking",
            FactType.MULTIPLE
        ),
        new Concept(
            "issue_solved",
            "Was the initial user's issue resolved?",
            FactType.SINGLE
        )
    ))
    .build();

    // Note: This example only shows the node creation.
    // You would need to add edges and other nodes to complete the graph.
```

- 在 custom node 中：

KotlinJava

```
llm.writeSession {
    replaceHistoryWithTLDR(
        strategy = FactRetrievalHistoryCompressionStrategy(
            Concept(
                keyword = "user_preferences", 
                // Description to the LLM -- what specifically to search for
                description = "User's preferences for the recommendation system, including the preferred conversation style, theme in the application, etc.",
                // LLM would search for multiple relevant facts related to this concept:
                factType = FactType.MULTIPLE
            ),
            Concept(
                keyword = "product_details",
                // Description to the LLM -- what specifically to search for
                description = "Brief details about products in the catalog the user has been checking",
                // LLM would search for multiple relevant facts related to this concept:
                factType = FactType.MULTIPLE
            ),
            Concept(
                keyword = "issue_solved",
                // Description to the LLM -- what specifically to search for
                description = "Was the initial user's issue resolved?",
                // LLM would search for a single answer to the question:
                factType = FactType.SINGLE
            )
        )
    )
}
```

```
ctx.getLlm().writeSession(session -> {
    session.replaceHistoryWithTLDR(new FactRetrievalHistoryCompressionStrategy(
            new Concept(
                "user_preferences", 
                // Description to the LLM -- what specifically to search for
                "User's preferences for the recommendation system, including the preferred conversation style, theme in the application, etc.",
                // LLM would search for multiple relevant facts related to this concept:
                FactType.MULTIPLE
            ),
            new Concept(
                "product_details",
                // Description to the LLM -- what specifically to search for
                "Brief details about products in the catalog the user has been checking",
                // LLM would search for multiple relevant facts related to this concept:
                FactType.MULTIPLE
            ),
            new Concept(
                "issue_solved",
                // Description to the LLM -- what specifically to search for
                "Was the initial user's issue resolved?",
                // LLM would search for a single answer to the question:
                FactType.SINGLE
            )
        ));
    return null;
});
```

## Custom history compression strategy implementation

Warning

Custom history compression strategies 仅在 Kotlin 中可用。

你可以通过扩展 `HistoryCompressionStrategy` 抽象类并实现 `compress` 方法来创建自己的 history compression strategy。

示例如下：

Kotlin

```
class MyCustomCompressionStrategy : HistoryCompressionStrategy() {
    override suspend fun compress(
        llmSession: AIAgentLLMWriteSession,
        memoryMessages: List<Message>
    ) {
        // 1. Process the current history in llmSession.prompt.messages
        // 2. Create new compressed messages
        // 3. Update the prompt with the compressed messages

        // Save original messages to preserve them
        val originalMessages = llmSession.prompt.messages

        // Example implementation:
        val importantMessages = llmSession.prompt.messages
            .filterIsInstance<Message.Assistant>()
            .filter { message ->
                // Your custom filtering logic
                message.parts.filterIsInstance<MessagePart.Text>().any { it.text.contains("important") }
            }

        // Note: you can also make LLM requests using the `llmSession` and ask the LLM to do some job for you using, for example, `llmSession.requestLLMWithoutTools()`
        // Or you can change the current model: `llmSession.model = AnthropicModels.Opus_4_6` and ask some other LLM model -- but don't forget to change it back after

        // Compose the prompt with the filtered messages
        val compressedMessages = composeMessageHistory(
            originalMessages,
            importantMessages,
            memoryMessages
        )
    }
}
```

在此示例中，自定义策略会筛选包含单词 "important" 的消息，并只在压缩后的历史中保留这些消息。

然后可以按如下方式使用：

- 在 strategy graph 中：

Kotlin

```
val compressHistory by nodeLLMCompressHistory<ProcessedInput>(
    strategy = MyCustomCompressionStrategy()
)
```

- 在 custom node 中：

Kotlin

```
llm.writeSession {
    replaceHistoryWithTLDR(strategy = MyCustomCompressionStrategy())
}
```

## 压缩期间的 memory preservation

所有 history compression 方法都支持 memory preservation，用于决定压缩期间是否应保留 memory-related messages。在 Kotlin 中，使用 `preserveMemory` 参数。在 Java 中，使用 `.preserveMemory()` builder 方法。
这些消息包含从 memory 检索到的事实，或表明 memory feature 未启用。

要启用 memory preservation：

- **Kotlin**：使用 `preserveMemory` 参数。
- **Java**：使用 `.preserveMemory()` builder 方法。
- 在 strategy graph 中：

KotlinJava

```
val compressHistory by nodeLLMCompressHistory<ProcessedInput>(
    strategy = HistoryCompressionStrategy.WholeHistory,
    preserveMemory = true
)
```

```
// Using WholeHistory strategy with preserveMemory=true
var compressHistory = AIAgentNode
    .llmCompressHistory("compressHistory")
    .withInput(String.class)
    .compressionStrategy(HistoryCompressionStrategy.WholeHistory)
    .preserveMemory(true)
    .build();

// Note: This example only shows the node creation.
// You would need to add edges and other nodes to complete the graph.
```

- 在 custom node 中：

KotlinJava

```
llm.writeSession {
    replaceHistoryWithTLDR(
        strategy = HistoryCompressionStrategy.WholeHistory,
        preserveMemory = true
    )
}
```

```
ctx.getLlm().writeSession(session -> {
    session.replaceHistoryWithTLDR(
        /** strategy */ HistoryCompressionStrategy.WholeHistory,
        /** preserveMemory */ true
    );
    return null;
});
```