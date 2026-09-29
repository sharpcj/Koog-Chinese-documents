# 跟踪

本页包含 Tracing 特性的详细信息，该特性为 AI agent 提供全面的跟踪能力。

## 特性概览

Tracing 特性是一个强大的监控和调试工具，可捕获 agent run 的详细信息，
包括：

- 策略执行
- LLM 调用
- LLM 流式传输（开始、帧、完成、错误）
- 工具调用
- agent 图中的节点执行

该特性通过拦截 agent 管道中的关键事件，并将其转发给可配置的消息
处理器来工作。这些处理器可以将跟踪信息输出到日志文件或文件系统中的其他类型
文件等各种目标，使开发者能够深入了解 agent 行为并有效排查问题。

### 事件流

1. Tracing 特性拦截 agent 管道中的事件。
2. 根据配置的消息过滤器过滤事件。
3. 过滤后的事件传递给已注册的消息处理器。
4. 消息处理器格式化事件并将其输出到各自的目标。

## 配置和初始化

### 基本设置

要使用 Tracing 特性，你需要：

1. 拥有一个或多个消息处理器（可以使用现有处理器或创建自己的处理器）。
2. 在 agent 中安装 `Tracing`。
3. 配置消息过滤器（可选）。
4. 将消息处理器添加到该特性。

KotlinJava

```
// Defining a logger/file that will be used as a destination of trace messages 
val logger = KotlinLogging.logger { }
val outputPath = Path("/path/to/trace.log")

// Creating an agent
val agent = AIAgent(
    promptExecutor = simpleOllamaAIExecutor(),
    llmModel = OllamaModels.Meta.LLAMA_3_2,
) {
    install(Tracing) {
        // Configure message processors to handle trace events
        addMessageProcessor(TraceFeatureMessageLogWriter(logger))
        addMessageProcessor(TraceFeatureMessageFileWriter.create(outputPath))
    }
}
```

```
// Defining a logger/file that will be used as a destination of trace messages
var logger = LoggerFactory.getLogger("tracing");
var outputPath = Path.of("/path/to/trace.log");

// Creating an agent
var agent = AIAgent.builder()
    .promptExecutor(PromptExecutor.builder().ollama().build())
    .llmModel(OllamaModels.Meta.LLAMA_3_2)

    .install(Tracing.Feature, config -> {
        // Configure message processors to handle trace events
        config.addMessageProcessor(TraceFeatureMessageLogWriter.create(logger));
        config.addMessageProcessor(TraceFeatureMessageFileWriter.create(outputPath));
    })
    .build();
```

### 消息过滤

你可以处理所有现有事件，也可以根据特定条件选择其中一部分。
消息过滤器允许你控制哪些事件会被处理。这对于聚焦 agent run 的特定方面很有用：

KotlinJava

```
val fileWriter = TraceFeatureMessageFileWriter(
    outputPath,
    { path: Path -> SystemFileSystem.sink(path).buffered() }
)

addMessageProcessor(fileWriter)

// Filter for LLM-related events only
fileWriter.setMessageFilter { message ->
    message is LLMCallStartingEvent || message is LLMCallCompletedEvent
}

// Filter for tool-related events only
fileWriter.setMessageFilter { message -> 
    message is ToolCallStartingEvent ||
        message is ToolCallCompletedEvent ||
        message is ToolValidationFailedEvent ||
        message is ToolCallFailedEvent
}

// Filter for node execution events only
fileWriter.setMessageFilter { message -> 
    message is NodeExecutionStartingEvent || message is NodeExecutionCompletedEvent
}
```

```
var fileWriter = TraceFeatureMessageFileWriter.create(
    outputPath,
    path -> { try { return Files.newOutputStream(path); } catch (IOException e) { throw new UncheckedIOException(e); }}
);

config.addMessageProcessor(fileWriter);

// Filter for LLM-related events only
fileWriter.setMessageFilter(message ->
    message instanceof LLMCallStartingEvent || message instanceof LLMCallCompletedEvent
);

// Filter for tool-related events only
fileWriter.setMessageFilter(message ->
    message instanceof ToolCallStartingEvent ||
        message instanceof ToolCallCompletedEvent ||
        message instanceof ToolValidationFailedEvent ||
        message instanceof ToolCallFailedEvent
);

// Filter for node execution events only
fileWriter.setMessageFilter(message ->
    message instanceof NodeExecutionStartingEvent || message instanceof NodeExecutionCompletedEvent
);
```

### 大量跟踪数据

对于具有复杂策略或长时间运行执行的 agent，跟踪事件量可能很大。可以考虑使用以下方法管理事件量：

- 使用特定消息过滤器减少事件数量。
- 实现带缓冲或采样的自定义消息处理器。
- 对日志文件使用文件轮转，防止其过大。

### 依赖图

Tracing 特性具有以下依赖：

```
Tracing
├── AIAgentPipeline (for intercepting events)
├── TraceFeatureConfig
│   └── FeatureConfig
├── Message Processors
│   ├── TraceFeatureMessageLogWriter
│   │   └── FeatureMessageLogWriter
│   ├── TraceFeatureMessageFileWriter
│   │   └── FeatureMessageFileWriter
│   └── TraceFeatureMessageRemoteWriter
│       └── FeatureMessageRemoteWriter
└── Event Types (from ai.koog.agents.core.feature.model)
    ├── AgentStartingEvent
    ├── AgentCompletedEvent
    ├── AgentExecutionFailedEvent
    ├── AgentClosingEvent
    ├── GraphStrategyStartingEvent
    ├── FunctionalStrategyStartingEvent
    ├── StrategyCompletedEvent
    ├── NodeExecutionStartingEvent
    ├── NodeExecutionCompletedEvent
    ├── NodeExecutionFailedEvent
    ├── SubgraphExecutionStartingEvent
    ├── SubgraphExecutionCompletedEvent
    ├── SubgraphExecutionFailedEvent
    ├── LLMCallStartingEvent
    ├── LLMCallCompletedEvent
    ├── LLMCallFailedEvent
    ├── LLMStreamingStartingEvent
    ├── LLMStreamingFrameReceivedEvent
    ├── LLMStreamingFailedEvent
    ├── LLMStreamingCompletedEvent
    ├── ToolCallStartingEvent
    ├── ToolValidationFailedEvent
    ├── ToolCallFailedEvent
    └── ToolCallCompletedEvent
```

## 示例和快速入门

### 到 logger 的基本跟踪

KotlinJava

```
// Create a logger
val logger = KotlinLogging.logger { }

// Create an agent with tracing
val agent = AIAgent(
    promptExecutor = simpleOllamaAIExecutor(),
    llmModel = OllamaModels.Meta.LLAMA_3_2,
) {
    install(Tracing) {
        addMessageProcessor(TraceFeatureMessageLogWriter(logger))
    }
}

// Run the agent
agent.run("Hello, agent!")
```

```
// Create a logger
var logger = LoggerFactory.getLogger("tracing");

// Create an agent with tracing
var agent = AIAgent.builder()
    .promptExecutor(PromptExecutor.builder().ollama().build())
    .llmModel(OllamaModels.Meta.LLAMA_3_2)

    .install(Tracing.Feature, config -> {
        config.addMessageProcessor(TraceFeatureMessageLogWriter.create(logger));
    })
    .build();

// Run the agent
agent.run("Hello, agent!");
```

## 错误处理和边界情况

### 没有消息处理器

如果没有向 Tracing 特性添加消息处理器，会记录一条警告：

```
Tracing Feature. No feature out stream providers are defined. Trace streaming has no target.
```

该特性仍会拦截事件，但它们不会被处理或输出到任何位置。

### 资源管理

消息处理器可能持有需要正确释放的资源（例如文件句柄）。使用 `use` 扩展
函数以确保正确清理：

KotlinJava

```
val writer = TraceFeatureMessageFileWriter(
    outputPath,
    { path: Path -> SystemFileSystem.sink(path).buffered() }
)

// Creating an agent
val agent = AIAgent(
    promptExecutor = simpleOllamaAIExecutor(),
    llmModel = OllamaModels.Meta.LLAMA_3_2,
) {
    install(Tracing) {
        addMessageProcessor(writer)
    }
}

// Run the agent
agent.run(input)

// Writer will be automatically closed when the block exits
```

```
var writer = TraceFeatureMessageFileWriter.create(
    outputPath,
    path -> { try { return Files.newOutputStream(path); } catch (IOException e) { throw new UncheckedIOException(e); }}
);

// Creating an agent
var agent = AIAgent.builder()
    .promptExecutor(PromptExecutor.builder().ollama().build())
    .llmModel(OllamaModels.Meta.LLAMA_3_2)

    .install(Tracing.Feature, config -> {
        config.addMessageProcessor(writer);
    })
    .build();

// Run the agent
agent.run(input);

// Writer will be automatically closed when the block exits
```

### 将特定事件跟踪到文件

KotlinJava

```
val fileWriter = TraceFeatureMessageFileWriter(
    outputPath,
    { path: Path -> SystemFileSystem.sink(path).buffered() }
)

// Creating an agent
val agent = AIAgent(
    promptExecutor = simpleOllamaAIExecutor(),
    llmModel = OllamaModels.Meta.LLAMA_3_2,
) {
    install(Tracing) {
        addMessageProcessor(fileWriter)

        // Only trace LLM calls
        fileWriter.setMessageFilter { message ->
            message is LLMCallStartingEvent || message is LLMCallCompletedEvent
        }
    }
}
```

```
var fileWriter = TraceFeatureMessageFileWriter.create(
    outputPath,
    path -> { try { return Files.newOutputStream(path); } catch (IOException e) { throw new UncheckedIOException(e); }}
);

// Creating an agent
var agent = AIAgent.builder()
    .promptExecutor(PromptExecutor.builder().ollama().build())
    .llmModel(OllamaModels.Meta.LLAMA_3_2)

    .install(Tracing.Feature, config -> {
        config.addMessageProcessor(fileWriter);

        // Only trace LLM calls
        fileWriter.setMessageFilter(message ->
            message instanceof LLMCallStartingEvent || message instanceof LLMCallCompletedEvent
        );
    })
    .build();
```

### 将特定事件跟踪到远程端点

当你需要通过网络发送事件数据时，可以使用到远程端点的跟踪。启动后，到
远程端点的跟踪会在指定端口号启动一个轻量服务器，并通过 Kotlin Server-Sent Events
(SSE) 发送事件。

KotlinJava

```
val connectionConfig = DefaultServerConnectionConfig(host = host, port = port)
val writer = TraceFeatureMessageRemoteWriter(connectionConfig)

// Creating an agent
val agent = AIAgent(
    promptExecutor = simpleOllamaAIExecutor(),
    llmModel = OllamaModels.Meta.LLAMA_3_2,
) {
    install(Tracing) {
        addMessageProcessor(writer)
    }
}

// Run the agent
agent.run(input)

// Writer will be automatically closed when the block exits
```

```
var connectionConfig = new DefaultServerConnectionConfig(host, port);
var writer = new TraceFeatureMessageRemoteWriter(connectionConfig);

// Creating an agent
var agent = AIAgent.builder()
    .promptExecutor(PromptExecutor.builder().ollama().build())
    .llmModel(OllamaModels.Meta.LLAMA_3_2)

    .install(Tracing.Feature, config -> {
        config.addMessageProcessor(writer);
    })
    .build();

// Run the agent
agent.run(input);

// Writer will be automatically closed when the block exits
```

在客户端，可以使用 `FeatureMessageRemoteClient` 接收事件并对其反序列化。

Kotlin

```
val clientConfig = DefaultClientConnectionConfig(host = host, port = port, protocol = URLProtocol.HTTP)
val agentEvents = mutableListOf<DefinedFeatureEvent>()

val clientJob = launch {
    FeatureMessageRemoteClient(connectionConfig = clientConfig, scope = this).use { client ->
        val collectEventsJob = launch {
            client.receivedMessages.consumeAsFlow().collect { event ->
                // Collect events from server
                agentEvents.add(event as DefinedFeatureEvent)

                // Stop collecting events on agent finished
                if (event is AgentCompletedEvent) {
                    cancel()
                }
            }
        }
        client.connect()
        collectEventsJob.join()
        client.healthCheck()
    }
}

listOf(clientJob).joinAll()
```

## API 文档

Tracing 特性采用模块化架构，包含以下关键组件：

1. [Tracing](https://api.koog.ai/agents/agents-features/agents-features-trace/ai.koog.agents.features.tracing.feature/-tracing/index.html)：拦截 agent 管道中事件的主特性类。
2. [TraceFeatureConfig](https://api.koog.ai/agents/agents-features/agents-features-trace/ai.koog.agents.features.tracing.feature/-trace-feature-config/index.html)：用于自定义特性行为的配置类。
3. 消息处理器：处理并输出跟踪事件的组件：
   - [TraceFeatureMessageLogWriter](https://api.koog.ai/agents/agents-features/agents-features-trace/ai.koog.agents.features.tracing.writer/-trace-feature-message-log-writer/index.html)：将跟踪事件写入 logger。
   - [TraceFeatureMessageFileWriter](https://api.koog.ai/agents/agents-features/agents-features-trace/ai.koog.agents.features.tracing.writer/-trace-feature-message-file-writer/index.html)：将跟踪事件写入文件。
   - [TraceFeatureMessageRemoteWriter](https://api.koog.ai/agents/agents-features/agents-features-trace/ai.koog.agents.features.tracing.writer/-trace-feature-message-remote-writer/index.html)：将跟踪事件发送到远程服务器。
