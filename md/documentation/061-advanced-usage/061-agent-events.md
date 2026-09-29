# Agent 事件

Agent 事件是作为 agent 工作流一部分发生的操作或交互。它们包括：

- Agent 生命周期事件
- 策略事件
- 节点执行事件
- LLM 调用事件
- LLM 流式事件
- 工具执行事件

注意：功能事件定义在 agents-core 模块中，位于包 `ai.koog.agents.core.feature.model.events` 下。诸如 `agents-features-trace` 和 `agents-features-event-handler` 等功能会消费这些事件，以处理并转发 agent 执行期间创建的消息。

## 预定义事件类型

Koog 提供了可用于自定义消息处理器的预定义事件类型。预定义事件可根据其关联的实体分为以下几类：

- [Agent 事件](#agent-events)
- [策略事件](#strategy-events)
- [节点事件](#node-events)
- [子图事件](#subgraph-events)
- [LLM 调用事件](#llm-call-events)
- [LLM 流式事件](#llm-streaming-events)
- [工具执行事件](#tool-execution-events)

### Agent 事件

#### AgentStartingEvent

表示 agent 运行的开始。包含以下字段：

| 名称 | 数据类型 | 必需 | 默认值 | 描述 |
| --- | --- | --- | --- | --- |
| `eventId` | String | 是 |  | 事件或事件组的唯一标识符。 |
| `executionInfo` | AgentExecutionInfo | 是 |  | 提供与此事件关联的执行相关的上下文信息。 |
| `agentId` | String | 是 |  | AI agent 的唯一标识符。 |
| `runId` | String | 是 |  | AI agent 运行的唯一标识符。 |

#### AgentCompletedEvent

表示 agent 运行的结束。包含以下字段：

| 名称 | 数据类型 | 必需 | 默认值 | 描述 |
| --- | --- | --- | --- | --- |
| `eventId` | String | 是 |  | 事件或事件组的唯一标识符。 |
| `executionInfo` | AgentExecutionInfo | 是 |  | 提供与此事件关联的执行相关的上下文信息。 |
| `agentId` | String | 是 |  | AI agent 的唯一标识符。 |
| `runId` | String | 是 |  | AI agent 运行的唯一标识符。 |
| `result` | String | 是 |  | agent 运行的结果。如果没有结果，可以为 `null`。 |

#### AgentExecutionFailedEvent

表示 agent 运行期间发生错误。包含以下字段：

| 名称 | 数据类型 | 必需 | 默认值 | 描述 |
| --- | --- | --- | --- | --- |
| `eventId` | String | 是 |  | 事件或事件组的唯一标识符。 |
| `executionInfo` | AgentExecutionInfo | 是 |  | 提供与此事件关联的执行相关的上下文信息。 |
| `agentId` | String | 是 |  | AI agent 的唯一标识符。 |
| `runId` | String | 是 |  | AI agent 运行的唯一标识符。 |
| `error` | AIAgentError | 是 |  | agent 运行期间发生的具体错误。有关更多信息，请参见 [AIAgentError](#aiagenterror)。 |

#### AgentClosingEvent

表示 agent 的关闭或终止。包含以下字段：

| 名称 | 数据类型 | 必需 | 默认值 | 描述 |
| --- | --- | --- | --- | --- |
| `eventId` | String | 是 |  | 事件或事件组的唯一标识符。 |
| `executionInfo` | AgentExecutionInfo | 是 |  | 提供与此事件关联的执行相关的上下文信息。 |
| `agentId` | String | 是 |  | AI agent 的唯一标识符。 |
`AIAgentError` 类提供了有关 agent 运行期间发生的错误的更多详细信息。包含以下字段：

| 名称 | 数据类型 | 必需 | 默认值 | 描述 |
| --- | --- | --- | --- | --- |
| `message` | String | 是 |  | 提供有关特定错误更多详细信息的消息。 |
| `stackTrace` | String | 是 |  | 直到最后执行代码的堆栈记录集合。 |
| `cause` | String | 否 | null | 错误的原因（如果可用）。 |

`AgentExecutionInfo` 类提供有关执行路径的上下文信息，支持跟踪 agent 运行中的嵌套执行上下文。包含以下字段：

| 名称 | 数据类型 | 必需 | 默认值 | 描述 |
| --- | --- | --- | --- | --- |
| `parent` | AgentExecutionInfo | 否 | null | 对父执行上下文的引用。如果为 null，则表示根执行层级。 |
| `partName` | String | 是 |  | 表示当前执行部分或片段名称的字符串。 |

### 策略事件

#### GraphStrategyStartingEvent

表示基于图的策略运行的开始。包含以下字段：

| 名称 | 数据类型 | 必需 | 默认值 | 描述 |
| --- | --- | --- | --- | --- |
| `eventId` | String | 是 |  | 事件或事件组的唯一标识符。 |
| `executionInfo` | AgentExecutionInfo | 是 |  | 提供与此事件关联的执行的上下文信息。 |
| `runId` | String | 是 |  | 策略运行的唯一标识符。 |
| `strategyName` | String | 是 |  | 策略的名称。 |
| `graph` | StrategyEventGraph | 是 |  | 表示策略工作流的图结构。 |

#### FunctionalStrategyStartingEvent

表示函数式策略运行的开始。包含以下字段：

| 名称 | 数据类型 | 必需 | 默认值 | 描述 |
| --- | --- | --- | --- | --- |
| `eventId` | String | 是 |  | 事件或事件组的唯一标识符。 |
| `executionInfo` | AgentExecutionInfo | 是 |  | 提供与此事件关联的执行的上下文信息。 |
| `runId` | String | 是 |  | 策略运行的唯一标识符。 |
| `strategyName` | String | 是 |  | 策略的名称。 |

#### StrategyCompletedEvent

表示策略运行的结束。包含以下字段：

| 名称 | 数据类型 | 必需 | 默认值 | 描述 |
| --- | --- | --- | --- | --- |
| `eventId` | String | 是 |  | 事件或事件组的唯一标识符。 |
| `executionInfo` | AgentExecutionInfo | 是 |  | 提供与此事件关联的执行的上下文信息。 |
| `runId` | String | 是 |  | 策略运行的唯一标识符。 |
| `strategyName` | String | 是 |  | 策略的名称。 |
| `result` | String | 是 |  | 运行的结果。如果没有结果，可以为 `null`。 |

### 节点事件

#### NodeExecutionStartingEvent

表示节点运行的开始。包含以下字段：
| 名称 | 数据类型 | 必需 | 默认值 | 描述 |
| --- | --- | --- | --- | --- |
| `eventId` | String | 是 |  | 事件或事件组的唯一标识符。 |
| `executionInfo` | AgentExecutionInfo | 是 |  | 提供与此事件关联的执行相关的上下文信息。 |
| `runId` | String | 是 |  | 策略运行的唯一标识符。 |
| `nodeName` | String | 是 |  | 其运行已开始的节点名称。 |
| `input` | JsonElement | 否 | null | 节点的输入值。 |

#### NodeExecutionCompletedEvent

表示节点运行的结束。包含以下字段：

| 名称 | 数据类型 | 必需 | 默认值 | 描述 |
| --- | --- | --- | --- | --- |
| `eventId` | String | 是 |  | 事件或事件组的唯一标识符。 |
| `executionInfo` | AgentExecutionInfo | 是 |  | 提供与此事件关联的执行相关的上下文信息。 |
| `runId` | String | 是 |  | 策略运行的唯一标识符。 |
| `nodeName` | String | 是 |  | 其运行已结束的节点名称。 |
| `input` | JsonElement | 否 | null | 节点的输入值。 |
| `output` | JsonElement | 否 | null | 节点产生的输出值。 |

#### NodeExecutionFailedEvent

表示节点运行期间发生的错误。包含以下字段：

| 名称 | 数据类型 | 必需 | 默认值 | 描述 |
| --- | --- | --- | --- | --- |
| `eventId` | String | 是 |  | 事件或事件组的唯一标识符。 |
| `executionInfo` | AgentExecutionInfo | 是 |  | 提供与此事件关联的执行相关的上下文信息。 |
| `runId` | String | 是 |  | 策略运行的唯一标识符。 |
| `nodeName` | String | 是 |  | 发生错误的节点名称。 |
| `input` | JsonElement | 否 | null | 提供给节点的输入数据。 |
| `error` | AIAgentError | 是 |  | 节点运行期间发生的具体错误。有关更多信息，请参阅 [AIAgentError](#aiagenterror)。 |

### 子图事件

#### SubgraphExecutionStartingEvent

表示子图运行的开始。包含以下字段：

| 名称 | 数据类型 | 必需 | 默认值 | 描述 |
| --- | --- | --- | --- | --- |
| `eventId` | String | 是 |  | 事件或事件组的唯一标识符。 |
| `executionInfo` | AgentExecutionInfo | 是 |  | 提供与此事件关联的执行相关的上下文信息。 |
| `runId` | String | 是 |  | 策略运行的唯一标识符。 |
| `subgraphName` | String | 是 |  | 其运行已开始的子图名称。 |
| `input` | JsonElement | 否 | null | 子图的输入值。 |

#### SubgraphExecutionCompletedEvent

表示子图运行的结束。包含以下字段：

| 名称 | 数据类型 | 必需 | 默认值 | 描述 |
| --- | --- | --- | --- | --- |
| `eventId` | String | 是 |  | 事件或事件组的唯一标识符。 |
| `executionInfo` | AgentExecutionInfo | 是 |  | 提供与此事件关联的执行相关的上下文信息。 |
| `runId` | String | 是 |  | 策略运行的唯一标识符。 |
| `subgraphName` | String | 是 |  | 其运行已结束的子图名称。 |
| `input` | JsonElement | 否 | null | 子图的输入值。 |
| `output` | JsonElement | 否 | null | 子图产生的输出值。 |

#### SubgraphExecutionFailedEvent
表示子图运行期间发生的错误。包含以下字段：

| 名称 | 数据类型 | 必需 | 默认值 | 描述 |
| --- | --- | --- | --- | --- |
| `eventId` | String | 是 |  | 事件或事件组的唯一标识符。 |
| `executionInfo` | AgentExecutionInfo | 是 |  | 提供与此事件关联的执行相关的上下文信息。 |
| `runId` | String | 是 |  | 策略运行的唯一标识符。 |
| `subgraphName` | String | 是 |  | 发生错误的子图名称。 |
| `input` | JsonElement | 否 | null | 提供给子图的输入数据。 |
| `error` | AIAgentError | 是 |  | 子图运行期间发生的具体错误。有关更多信息，请参见 [AIAgentError](#aiagenterror)。 |

### LLM 调用事件

#### LLMCallStartingEvent

表示 LLM 调用的开始。包含以下字段：

| 名称 | 数据类型 | 必需 | 默认值 | 描述 |
| --- | --- | --- | --- | --- |
| `eventId` | String | 是 |  | 事件或事件组的唯一标识符。 |
| `executionInfo` | AgentExecutionInfo | 是 |  | 提供与此事件关联的执行相关的上下文信息。 |
| `runId` | String | 是 |  | LLM 运行的唯一标识符。 |
| `prompt` | Prompt | 是 |  | 发送给模型的提示词。有关更多信息，请参见 [Prompt](#prompt)。 |
| `model` | ModelInfo | 是 |  | 模型信息。请参见 [ModelInfo](#modelinfo)。 |
| `tools` | List | 是 |  | 模型可以调用的工具列表。 |

`Prompt` 类表示提示词的数据结构，由消息列表、唯一标识符以及语言模型设置的可选参数组成。包含以下字段：

| 名称 | 数据类型 | 必需 | 默认值 | 描述 |
| --- | --- | --- | --- | --- |
| `messages` | List | 是 |  | 提示词包含的消息列表。 |
| `id` | String | 是 |  | 提示词的唯一标识符。 |
| `params` | LLMParams | 否 | LLMParams() | 控制 LLM 生成内容方式的设置。 |

`ModelInfo` 类表示语言模型的信息，包括其提供方、模型标识符和特性。包含以下字段：

| 名称 | 数据类型 | 必需 | 默认值 | 描述 |
| --- | --- | --- | --- | --- |
| `provider` | String | 是 |  | 提供方标识符（例如 "openai"、"google"、"anthropic"）。 |
| `model` | String | 是 |  | 模型标识符（例如 "gpt-4"、"claude-3"）。 |
| `displayName` | String | 否 | null | 模型的可选人类可读显示名称。 |
| `contextLength` | Long | 否 | null | 模型可以处理的最大 token 数。 |
| `maxOutputTokens` | Long | 否 | null | 模型可以生成的最大 token 数。 |

#### LLMCallCompletedEvent

表示 LLM 调用的结束。包含以下字段：
| 名称 | 数据类型 | 必需 | 默认值 | 描述 |
| --- | --- | --- | --- | --- |
| `eventId` | String | 是 |  | 事件或事件组的唯一标识符。 |
| `executionInfo` | AgentExecutionInfo | 是 |  | 提供与此事件关联的执行相关的上下文信息。 |
| `runId` | String | 是 |  | LLM 运行的唯一标识符。 |
| `prompt` | Prompt | 是 |  | 调用中使用的提示词。 |
| `model` | ModelInfo | 是 |  | 模型信息。参见 [ModelInfo](#modelinfo)。 |
| `responses` | List | 是 |  | 模型返回的一个或多个响应。 |
| `moderationResponse` | ModerationResult | 否 | null | 审核响应（如果有）。 |

#### LLMCallFailedEvent

表示 LLM 调用期间发生错误。包含以下字段：

| 名称 | 数据类型 | 必需 | 默认值 | 描述 |
| --- | --- | --- | --- | --- |
| `eventId` | String | 是 |  | 事件或事件组的唯一标识符。 |
| `executionInfo` | AgentExecutionInfo | 是 |  | 提供与此事件关联的执行相关的上下文信息。 |
| `runId` | String | 是 |  | LLM 运行的唯一标识符。 |
| `prompt` | Prompt | 是 |  | 发送给模型的提示词。 |
| `model` | ModelInfo | 是 |  | 模型信息。参见 [ModelInfo](#modelinfo)。 |
| `tools` | List | 是 |  | 模型可以调用的工具列表。 |
| `error` | AIAgentError | 是 |  | 调用期间发生的具体错误。有关更多信息，请参见 [AIAgentError](#aiagenterror)。 |

### LLM 流式事件

#### LLMStreamingStartingEvent

表示 LLM 流式调用的开始。包含以下字段：

| 名称 | 数据类型 | 必需 | 默认值 | 描述 |
| --- | --- | --- | --- | --- |
| `eventId` | String | 是 |  | 事件或事件组的唯一标识符。 |
| `executionInfo` | AgentExecutionInfo | 是 |  | 提供与此事件关联的执行相关的上下文信息。 |
| `runId` | String | 是 |  | LLM 运行的唯一标识符。 |
| `prompt` | Prompt | 是 |  | 发送给模型的提示词。 |
| `model` | ModelInfo | 是 |  | 模型信息。参见 [ModelInfo](#modelinfo)。 |
| `tools` | List | 是 |  | 模型可以调用的工具列表。 |

#### LLMStreamingFrameReceivedEvent

表示从 LLM 接收到的流式帧。包含以下字段：

| 名称 | 数据类型 | 必需 | 默认值 | 描述 |
| --- | --- | --- | --- | --- |
| `eventId` | String | 是 |  | 事件或事件组的唯一标识符。 |
| `executionInfo` | AgentExecutionInfo | 是 |  | 提供与此事件关联的执行相关的上下文信息。 |
| `runId` | String | 是 |  | LLM 运行的唯一标识符。 |
| `prompt` | Prompt | 是 |  | 发送给模型的提示词。 |
| `model` | ModelInfo | 是 |  | 模型信息。参见 [ModelInfo](#modelinfo)。 |
| `frame` | StreamFrame | 是 |  | 从流中接收到的帧。 |

#### LLMStreamingFailedEvent

表示 LLM 流式调用期间发生错误。包含以下字段：
| 名称 | 数据类型 | 必填 | 默认值 | 描述 |
| --- | --- | --- | --- | --- |
| `eventId` | String | 是 |  | 事件或事件组的唯一标识符。 |
| `executionInfo` | AgentExecutionInfo | 是 |  | 提供与此事件关联的执行相关的上下文信息。 |
| `runId` | String | 是 |  | LLM 运行的唯一标识符。 |
| `prompt` | Prompt | 是 |  | 发送给模型的提示词。 |
| `model` | ModelInfo | 是 |  | 模型信息。参见 [ModelInfo](#modelinfo)。 |
| `error` | AIAgentError | 是 |  | 流式传输期间发生的具体错误。有关更多信息，请参见 [AIAgentError](#aiagenterror)。 |

#### LLMStreamingCompletedEvent

表示 LLM 流式调用结束。包含以下字段：

| 名称 | 数据类型 | 必填 | 默认值 | 描述 |
| --- | --- | --- | --- | --- |
| `eventId` | String | 是 |  | 事件或事件组的唯一标识符。 |
| `executionInfo` | AgentExecutionInfo | 是 |  | 提供与此事件关联的执行相关的上下文信息。 |
| `runId` | String | 是 |  | LLM 运行的唯一标识符。 |
| `prompt` | Prompt | 是 |  | 发送给模型的提示词。 |
| `model` | ModelInfo | 是 |  | 模型信息。参见 [ModelInfo](#modelinfo)。 |
| `tools` | List | 是 |  | 模型可以调用的工具列表。 |

### 工具执行事件

#### ToolCallStartingEvent

表示模型调用工具的事件。包含以下字段：

| 名称 | 数据类型 | 必填 | 默认值 | 描述 |
| --- | --- | --- | --- | --- |
| `eventId` | String | 是 |  | 事件或事件组的唯一标识符。 |
| `executionInfo` | AgentExecutionInfo | 是 |  | 提供与此事件关联的执行相关的上下文信息。 |
| `runId` | String | 是 |  | 策略/智能体运行的唯一标识符。 |
| `toolCallId` | String | 否 | null | 工具调用的标识符（如果可用）。 |
| `toolName` | String | 是 |  | 工具的名称。 |
| `toolArgs` | JsonObject | 是 |  | 提供给工具的参数。 |

#### ToolValidationFailedEvent

表示工具调用期间发生验证错误。包含以下字段：

| 名称 | 数据类型 | 必填 | 默认值 | 描述 |
| --- | --- | --- | --- | --- |
| `eventId` | String | 是 |  | 事件或事件组的唯一标识符。 |
| `executionInfo` | AgentExecutionInfo | 是 |  | 提供与此事件关联的执行相关的上下文信息。 |
| `runId` | String | 是 |  | 策略/智能体运行的唯一标识符。 |
| `toolCallId` | String | 否 | null | 工具调用的标识符（如果可用）。 |
| `toolName` | String | 是 |  | 验证失败的工具的名称。 |
| `toolArgs` | JsonObject | 是 |  | 提供给工具的参数。 |
| `toolDescription` | String | 否 | null | 遇到验证错误的工具的说明。 |
| `message` | String | 否 | null | 描述验证错误的消息。 |
| `error` | AIAgentError | 是 |  | 发生的具体错误。有关更多信息，请参见 [AIAgentError](#aiagenterror)。 |

#### ToolCallFailedEvent

表示执行工具失败。包含以下字段：
| 名称 | 数据类型 | 必填 | 默认值 | 描述 |
| --- | --- | --- | --- | --- |
| `eventId` | String | 是 |  | 事件或事件组的唯一标识符。 |
| `executionInfo` | AgentExecutionInfo | 是 |  | 提供与此事件关联的执行相关的上下文信息。 |
| `runId` | String | 是 |  | 策略/代理运行的唯一标识符。 |
| `toolCallId` | String | 否 | null | 工具调用的标识符（如果可用）。 |
| `toolName` | String | 是 |  | 工具的名称。 |
| `toolArgs` | JsonObject | 是 |  | 提供给工具的参数。 |
| `toolDescription` | String | 否 | null | 失败工具的描述。 |
| `error` | AIAgentError | 是 |  | 尝试调用工具时发生的具体错误。有关更多信息，请参阅 [AIAgentError](#aiagenterror)。 |

#### ToolCallCompletedEvent

表示一次成功的工具调用及其返回结果。包含以下字段：

| 名称 | 数据类型 | 必填 | 默认值 | 描述 |
| --- | --- | --- | --- | --- |
| `eventId` | String | 是 |  | 事件或事件组的唯一标识符。 |
| `executionInfo` | AgentExecutionInfo | 是 |  | 提供与此事件关联的执行相关的上下文信息。 |
| `runId` | String | 是 |  | 运行的唯一标识符。 |
| `toolCallId` | String | 否 | null | 工具调用的标识符。 |
| `toolName` | String | 是 |  | 工具的名称。 |
| `toolArgs` | JsonObject | 是 |  | 提供给工具的参数。 |
| `toolDescription` | String | 否 | null | 已执行工具的描述。 |
| `result` | JsonElement | 否 | null | 工具调用的结果。 |

## 常见问题与故障排除

以下部分包含与 Tracing 功能相关的常见问题与解答。

### 如何仅跟踪代理执行的特定部分？

使用 `messageFilter` 属性来过滤事件。例如，仅跟踪节点执行：

KotlinJava
```
install(Tracing) {
    val fileWriter = TraceFeatureMessageFileWriter.create(outputPath)
    addMessageProcessor(fileWriter)

    // Only trace LLM calls
    fileWriter.setMessageFilter { message ->
        message is LLMCallStartingEvent || message is LLMCallCompletedEvent
    }
}
```

```
.install(Tracing.Feature, config -> {
    var fileWriter = TraceFeatureMessageFileWriter.create(outputPath);
    config.addMessageProcessor(fileWriter);

    // Only trace LLM calls
    fileWriter.setMessageFilter(message ->
        message instanceof LLMCallStartingEvent || message instanceof LLMCallCompletedEvent
    );
})
```
### 我可以使用多个消息处理器吗？

可以，你可以添加多个消息处理器，同时跟踪到不同的目标：

KotlinJava
```
install(Tracing) {
    addMessageProcessor(TraceFeatureMessageLogWriter(logger))
    addMessageProcessor(TraceFeatureMessageFileWriter.create(outputPath))
    addMessageProcessor(TraceFeatureMessageRemoteWriter(connectionConfig))
}
```

```
.install(Tracing.Feature, config -> {
    config.addMessageProcessor(TraceFeatureMessageLogWriter.create(logger));
    config.addMessageProcessor(TraceFeatureMessageFileWriter.create(outputPath));
    config.addMessageProcessor(new TraceFeatureMessageRemoteWriter());
})
```
### 如何创建自定义消息处理器？

实现 `FeatureMessageProcessor` 接口：

KotlinJava
```
class CustomTraceProcessor : FeatureMessageProcessor() {

    override suspend fun processMessage(message: FeatureMessage) {
        // Custom processing logic
        if (message is NodeExecutionStartingEvent) {
            // Process node start event
        } else if (message is LLMCallCompletedEvent) {
            // Process LLM call end event
        } else {
            // Handle other event types
        }
    }

    override suspend fun close() {
        // Close connections if established
    }
}

val agent = AIAgent(
    promptExecutor = simpleOllamaAIExecutor(),
    llmModel = OllamaModels.Meta.LLAMA_3_2,
) {
    install(Tracing) {
        // Use your custom processor
        addMessageProcessor(CustomTraceProcessor())
    }
}
```

```
class CustomTraceProcessor extends FeatureMessageProcessor {

    @Override
    protected void handleMessage(FeatureMessage message) {
        // Custom processing logic
        if (message instanceof NodeExecutionStartingEvent) {
            // Process node start event
        } else if (message instanceof LLMCallCompletedEvent) {
            // Process LLM call end event
        } else {
            // Handle other event types
        }
    }

    @Override
    public void handleClose() {
        // Close connections if established
    }
}

var agent = AIAgent.builder()
    .promptExecutor(PromptExecutor.builder().ollama().build())
    .llmModel(OllamaModels.Meta.LLAMA_3_2)

    .install(Tracing.Feature, config -> {
        // Use your custom processor
        config.addMessageProcessor(new CustomTraceProcessor());
    })
    .build();
```
有关消息处理器可处理的现有事件类型的更多信息，请参阅[预定义事件类型](#predefined-event-types)。
