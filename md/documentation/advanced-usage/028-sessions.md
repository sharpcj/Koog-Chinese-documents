# LLM sessions 和手动历史管理

本页提供有关 LLM sessions 的详细信息，包括如何使用读写 sessions、管理对话历史，以及向语言模型发起请求。

## 引言

LLM sessions 是一个基础概念，它为与语言模型（LLMs）交互提供了结构化方式。
它们管理对话历史、处理对 LLM 的请求，并为运行工具和处理响应提供一致接口。

## 理解 LLM sessions

LLM session 表示与语言模型交互的上下文。它封装了：

- 对话历史（prompt）
- 可用工具
- 向 LLM 发起请求的方法
- 更新对话历史的方法
- 运行工具的方法

Sessions 由 `AIAgentLLMContext` 类管理，该类提供用于创建读写 sessions 的方法。

### Session 类型

Koog 框架提供两种 session：

1. **Write Sessions**（`AIAgentLLMWriteSession`）：允许修改 prompt 和工具、发起 LLM 请求并运行工具。在 write session 中所做的更改会持久回写到 LLM 上下文。
2. **Read Sessions**（`AIAgentLLMReadSession`）：提供对 prompt 和工具的只读访问。它们适用于在不进行更改的情况下检查当前状态。

关键区别在于 write sessions 可以修改对话历史，而 read sessions 不能。

### Session 生命周期

Sessions 具有明确的生命周期：

1. **创建**：创建 session，例如使用 `llm.writeSession { ... }` 或 `llm.readSession { ... }`。
2. **活动阶段**：lambda 块执行期间，session 处于活动状态。
3. **终止**：lambda 块完成时，session 会自动关闭。

Sessions 实现了 `AutoCloseable` 接口，确保即使发生异常也能被正确清理。

## 使用 LLM sessions

### 创建 sessions

Sessions 通过 `AIAgentLLMContext` 类的方法创建：

KotlinJava

```
// Creating a write session
llm.writeSession {
    // Session code here
}

// Creating a read session
llm.readSession {
    // Session code here
}
```

```
// Creating a write session
ctx.getLlm().writeSession(session -> {
    // Session code here
    return null;
});

// Creating a read session
ctx.getLlm().readSession(session -> {
    // Session code here
    return null;
});
```

这些函数接受一个 lambda 块，该块在 session 上下文中运行。块完成后，session 会自动关闭。

### Session 作用域和线程安全

Sessions 使用读写锁来确保线程安全：

- 多个 read sessions 可以同时处于活动状态。
- 一次只能有一个 write session 处于活动状态。
- write session 会阻塞所有其他 sessions（包括 read 和 write）。

这可确保对话历史不会被并发修改破坏。

### 访问 session 属性

在 session 内，你可以访问 prompt 和工具：

KotlinJava

```
llm.readSession {
    val messageCount = prompt.messages.size
    val availableTools = tools.map { it.name }
}
```

```
ctx.getLlm().readSession(session -> {
    int messageCount = session.getPrompt().getMessages().size();
    var availableTools = session.getTools().stream().map(tool -> tool.getName()).toList();
    return null;
});
```

在 write session 中，你还可以修改这些属性：

KotlinJava

```
llm.writeSession {
    // Modify the prompt
    appendPrompt {
        user("New user message")
    }

    // Modify the tools
    tools = newTools
}
```

```
ctx.getLlm().writeSession(session -> {
    // Modify the prompt
    session.appendPrompt(promptBuilder -> {
        promptBuilder.user("New user message");
        return null;
    });

    // Modify the tools
    session.setTools(newTools);
    return null;
});
```

更多信息，请参阅 [AIAgentLLMReadSession](https://api.koog.ai/agents/agents-core/ai.koog.agents.core.agent.session/-a-i-agent-l-l-m-read-session/index.html) 和 [AIAgentLLMWriteSession](https://api.koog.ai/agents/agents-core/ai.koog.agents.core.agent.session/-a-i-agent-l-l-m-write-session/index.html) 的详细 API 参考。

## 发起 LLM 请求

### 基本请求方法

最常用的 LLM 请求方法包括：

1. `requestLLM()`：使用当前 prompt 和工具向 LLM 发起请求，返回响应。
2. `requestLLMWithoutTools()`：使用当前 prompt 但不带任何工具向 LLM 发起请求，返回单个响应。
3. `requestLLMForceOneTool()`：使用当前 prompt 和工具向 LLM 发起请求，并强制使用一个工具。
4. `requestLLMOnlyCallingTools()`：发起一个应仅通过使用工具来处理的 LLM 请求。

示例：

KotlinJava

```
llm.writeSession {
    // Make a request with tools enabled
    val response = requestLLM()

    // Make a request without tools
    val responseWithoutTools = requestLLMWithoutTools()
}
```

```
ctx.getLlm().writeSession(session -> {
    // Make a request with tools enabled
    var response = session.requestLLM();

    // Make a request without tools
    var responseWithoutTools = session.requestLLMWithoutTools();

    return null;
});
```

### 请求如何工作

LLM 请求会在你显式调用某个请求方法时发起。需要理解的关键点如下：

1. **显式调用**：只有在你调用 `requestLLM()`、`requestLLMWithoutTools()` 等方法时，请求才会发生。
2. **立即执行**：调用请求方法时，请求会立即发起，并且该方法会阻塞直到收到响应。
3. **自动历史更新**：在 write session 中，响应会自动添加到对话历史中。

### 带工具的请求方法

在启用工具的情况下发起请求时，LLM 可能会返回工具调用而不是文本响应。请求方法会透明地处理这一点：

KotlinJava

```
llm.writeSession {
    val response = requestLLM()

    // The response might contain tool calls and/or text
    val toolCalls = response.parts.filterIsInstance<MessagePart.Tool.Call>()
    if (toolCalls.isNotEmpty()) {
        // Handle tool calls
    } else {
        // Handle text response
    }
}
```

```
ctx.getLlm().writeSession(session -> {
    var response = session.requestLLM();

    // The response parts might contain a tool call or text content
    boolean hasToolCall = response.getParts().stream()
        .anyMatch(p -> p instanceof MessagePart.Tool.Call);
    if (hasToolCall) {
        // Handle tool call
    } else {
        // Handle text response
    }
    return null;
});
```

实践中，你通常不需要手动检查响应类型，因为 agent graph 会自动处理这种路由。

### 结构化和流式请求

对于更高级的用例，提供了结构化和流式请求方法：

1. `requestLLMStructured()`：请求 LLM 以特定结构化格式提供响应。
2. `requestLLMStructuredOneShot()`：类似于 `requestLLMStructured()`，但没有重试或修正。
3. `requestLLMStreaming()`：向 LLM 发起流式请求，返回响应块的 flow。你可以在 [Streaming API](../streaming-api/) 页面了解更多关于 streaming 的内容。

示例：

KotlinJava

```
llm.writeSession {
    // Make a structured request
    val structuredResponse = requestLLMStructured<JokeRating>()

    // Make a streaming request
    val responseStream = requestLLMStreaming()
    responseStream.collect { chunk ->
        // Process each chunk as it arrives
    }
}
```

```
ctx.getLlm().writeSession(session -> {
    // Make a non-tool request
    var responseWithoutTools = session.requestLLMWithoutTools();

    // Make a streaming request
    var responseStream = session.requestLLMStreaming();
    // Process chunks from Flow.Publisher<StreamFrame>
    return null;
});
```

## 管理对话历史

### 更新 prompt

在 write session 中，你可以使用 `appendPrompt` 方法向 prompt（对话历史）添加消息：

KotlinJava

```
llm.writeSession {
    appendPrompt {
        // Add a system message
        system("You are a helpful assistant.")

        // Add a user message
        user("Hello, can you help me with a coding question?")

        // Add an assistant message
        assistant("Of course! What's your question?")

        // Add a tool result
        toolResult(myToolResult)
    }
}
```

```
ctx.getLlm().writeSession(session -> {
    session.appendPrompt(promptBuilder -> {
        // Add a system message
        promptBuilder.system("You are a helpful assistant.");

        // Add a user message
        promptBuilder.user("Hello, can you help me with a coding question?");

        // Add an assistant message
        promptBuilder.assistant("Of course! What's your question?");

        // Add follow-up context after tool execution
        promptBuilder.assistant("Tool execution completed successfully.");
        return null;
    });
    return null;
});
```

你也可以通过为 `prompt` 属性分配一个新的 `Prompt` 对象来完全重写 prompt：

KotlinJava

```
llm.writeSession {
    // Create a new prompt based on the old one
    prompt = prompt.copy(messages = filteredMessages)
}
```

```
ctx.getLlm().writeSession(session -> {
    var oldPrompt = session.getPrompt();
    // Rebuild and replace the prompt (manual rewrite approach in Java)
    session.setPrompt(
        Prompt.builder(oldPrompt.getId())
            .user("Retained summary of previous conversation")
            .build()
    );
    return null;
});
```

### 响应时自动更新历史

当你在 write session 中发起 LLM 请求时，响应会自动添加到对话历史：

KotlinJava

```
llm.writeSession {
    // Add a user message
    appendPrompt {
        user("What's the capital of France?")
    }

    // Make a request - the response is automatically added to the history
    val response = requestLLM()

    // The prompt now includes both the user message and the model's response
}
```

```
ctx.getLlm().writeSession(session -> {
    // Add a user message
    session.appendPrompt(promptBuilder -> {
        promptBuilder.user("What's the capital of France?");
        return null;
    });

    // Make a request - the response is automatically added to the history
    var response = session.requestLLM();

    // The prompt now includes both the user message and the model's response
    return null;
});
```

这种自动历史更新是 write sessions 的关键功能，可确保对话自然流动。

### 历史压缩

对于长时间运行的对话，历史可能会变得很大并消耗大量 tokens。平台提供了压缩历史的方法：

KotlinJava

```
llm.writeSession {
    // Compress the history using a TLDR approach
    replaceHistoryWithTLDR(HistoryCompressionStrategy.WholeHistory, preserveMemory = true)
}
```

```
// Use the dedicated Java node for history compression.
var compressHistory = AIAgentNode.llmCompressHistory("compressHistory");
```

你也可以在 strategy graph 中使用 `nodeLLMCompressHistory` 节点，在特定点压缩历史。

有关历史压缩和压缩策略的更多信息，请参阅 [History compression](../history-compression/)。

## 在 sessions 中运行工具

### 调用工具

Write sessions 提供多种调用工具的方法：

1. `callTool(tool, args)`：通过引用调用工具。
2. `callTool(toolName, args)`：通过名称调用工具。
3. `callTool(toolClass, args)`：通过类调用工具。
4. `callToolRaw(toolName, args)`：通过名称调用工具并返回原始字符串结果。

示例：

KotlinJava

```
llm.writeSession {
    // Call a tool by reference
    val result = callTool(myTool, myArgs)

    // Call a tool by name
    val result2 = callTool("myToolName", myArgs)

    // Call a tool by class
    val result3 = callTool(MyTool::class, myArgs)

    // Call a tool and get the raw result
    val rawResult = callToolRaw("myToolName", myArgs)
}
```

```
// Java uses dedicated tool-execution nodes in graph strategies.
var executeTool = AIAgentNode.executeTools("executeTool");
var sendToolResult = AIAgentNode.llmSendToolResults("sendToolResult");
```

### 并行工具运行

要并行运行多个工具，write sessions 提供了 `Flow` 上的扩展函数：

KotlinJava

```
llm.writeSession {
    // Run tools in parallel
    parseDataToArgs(data).toParallelToolCalls(MyTool::class).collect { result ->
        // Process each result
    }

    // Run tools in parallel and get raw results
    parseDataToArgs(data).toParallelToolCallsRaw(MyTool::class).collect { rawResult ->
        // Process each raw result
    }
}
```

```
// Java equivalent for multi-tool execution: use the executeTools node.
var executeMultipleTools = AIAgentNode.executeTools("executeMultipleTools");
```

这有助于高效处理大量数据。

## 最佳实践

使用 LLM sessions 时，请遵循以下最佳实践：

1. **使用正确的 session 类型**：当你需要修改对话历史时使用 write sessions；只需要读取时使用 read sessions。
2. **保持 sessions 简短**：Sessions 应聚焦于特定任务，并尽快关闭以释放资源。
3. **处理异常**：确保在 sessions 内处理异常，防止资源泄漏。
4. **管理历史大小**：对于长时间运行的对话，使用历史压缩来减少 token 使用。
5. **优先使用高级抽象**：尽可能使用基于节点的 API。例如，使用 `nodeLLMRequest`，而不是直接使用 sessions。
6. **注意线程安全**：记住 write sessions 会阻塞其他 sessions，因此应尽量缩短写操作。
7. **对复杂数据使用结构化请求**：当你需要 LLM 返回结构化数据时，使用 `requestLLMStructured`，而不是解析自由格式文本。
8. **对长响应使用 streaming**：对于长响应，使用 `requestLLMStreaming` 在响应到达时逐块处理。

## 故障排除

### Session already closed

如果你看到类似 `Cannot use session after it was closed` 的错误，说明你正在 session 的 lambda 块完成后使用该 session。请确保所有 session 操作都在 session 块内执行。

### History too large

如果你的历史变得过大并消耗过多 tokens，请使用历史压缩技术：

KotlinJava

```
llm.writeSession {
    replaceHistoryWithTLDR(HistoryCompressionStrategy.FromLastNMessages(10), preserveMemory = true)
}
```

```
// Compress recent history with the Java compression node.
var compressHistory = AIAgentNode.llmCompressHistory("compressHistory");
```

更多信息，请参阅 [History compression](../history-compression/)

### Tool not found

如果看到找不到工具的错误，请检查：

- 工具是否已正确注册到工具注册表中。
- 你是否使用了正确的工具名称或类。

## API 文档

更多信息，请参阅完整的 [AIAgentLLMSession](https://api.koog.ai/agents/agents-core/ai.koog.agents.core.agent.session/-a-i-agent-l-l-m-session/index.html) 和 [AIAgentLLMContext](https://api.koog.ai/agents/agents-core/ai.koog.agents.core.agent.context/-a-i-agent-l-l-m-context/index.html) 参考。
