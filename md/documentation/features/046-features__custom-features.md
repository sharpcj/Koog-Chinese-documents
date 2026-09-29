# 自定义特性

特性提供了一种在运行时扩展和增强 AI agent 功能的方式。它们被设计为模块化且可组合，
允许你根据需要混合搭配。

除了 Koog 开箱即用的[特性](../)之外，你也可以通过扩展合适的特性接口来实现
自己的特性。
本页使用当前 Koog API 展示构建自定义特性的基本组成部分。

## 特性接口

Koog 提供以下接口，你可以扩展它们来实现自定义特性：

- [AIAgentGraphFeature](https://api.koog.ai/agents/agents-core/ai.koog.agents.core.feature/-a-i-agent-graph-feature/index.html)：表示特定于[已定义工作流的 agent](../../agents/graph-based-agents/)（基于图的 agent）的特性。
- [AIAgentFunctionalFeature](https://api.koog.ai/agents/agents-core/ai.koog.agents.core.feature/-a-i-agent-functional-feature/index.html)：表示可用于[函数式 agent](../../agents/functional-agents/) 的特性。
- [AIAgentPlannerFeature](https://api.koog.ai/agents/agents-planner/ai.koog.agents.planner/-a-i-agent-planner-feature/index.html)：表示特定于[规划器 agent](../../agents/planner-agents/) 的特性类型。

Note

要创建可安装到基于图、函数式和规划器 agent 中的自定义特性，你需要
实现所有接口。

## 实现自定义特性

要实现自定义特性，你需要按以下步骤创建特性结构：

1. 创建特性类。
2. 定义配置类。该配置类是 [FeatureConfig](https://api.koog.ai/agents/agents-core/ai.koog.agents.core.feature.config/-feature-config/index.html) 类的扩展。
3. 创建一个 companion object，用于实现以下接口中的部分或全部：[AIAgentGraphFeature](https://api.koog.ai/agents/agents-core/ai.koog.agents.core.feature/-a-i-agent-graph-feature/index.html)、[AIAgentFunctionalFeature](https://api.koog.ai/agents/agents-core/ai.koog.agents.core.feature/-a-i-agent-functional-feature/index.html)、[AIAgentPlannerFeature](https://api.koog.ai/agents/agents-planner/ai.koog.agents.planner/-a-i-agent-planner-feature/index.html)。
4. 为你的特性提供一个唯一的存储键，用于在 agent 管道中标识和检索该特性。该
   键用于 agent 管道的内部映射，该映射包含一个 agent 的所有已注册特性。运行 agent 时，它需要处理所有已注册特性，并使用该键从此映射中检索特性。
5. 实现必需的方法。

下面的代码示例展示了实现可安装到基于图、函数式和规划器 agent 中的自定义特性的一般模式：

```
class MyFeature(val someProperty: String) {
    class Config : FeatureConfig() {
        var configProperty: String = "default"
    }

    companion object Feature : AIAgentGraphFeature<Config, MyFeature>, AIAgentFunctionalFeature<Config, MyFeature>, AIAgentPlannerFeature<Config, MyFeature> {
        // Unique storage key for retrieval in contexts
        override val key = createStorageKey<MyFeature>("my-feature")
        override fun createInitialConfig(agentConfig: AIAgentConfig): Config = Config()

        // Feature installation for graph-based agents
        override fun install(config: Config, pipeline: AIAgentGraphPipeline) : MyFeature {
            val feature = MyFeature(config.configProperty)

            pipeline.interceptAgentStarting(this) { context ->
                // Event handler implementation
            }
            return feature
        }

        // Feature installation for functional agents
        override fun install(config: Config, pipeline: AIAgentFunctionalPipeline) : MyFeature {
            val feature = MyFeature(config.configProperty)

            pipeline.interceptAgentStarting(this) { context ->
                // Event handler implementation
            }
            return feature
        }

        // Feature installation for planner agents
        override fun install(config: Config, pipeline: AIAgentPlannerPipeline) : MyFeature {
            val feature = MyFeature(config.configProperty)

            pipeline.interceptAgentStarting(this) { context ->
                // Event handler implementation
            }
            return feature
        }
    }
}
```

创建 agent 时，使用 `install` 方法安装你的特性：

```
val agent = AIAgent(
    promptExecutor = simpleOpenAIExecutor(System.getenv("OPENAI_API_KEY")),
    systemPrompt = "You are a helpful assistant. Answer user questions concisely.",
    llmModel = OpenAIModels.Chat.GPT4o
) {
    install(MyFeature) {
        configProperty = "value"
    }
}
```

### 管道拦截器

拦截器表示 agent 生命周期中的各个位置，你可以在这些位置接入 agent 执行管道以
实现自定义逻辑。Koog 包含一系列预定义拦截器，你可以用它们观察各种
事件。

以下是你可以从特性的 `install` 方法注册的拦截器。列出的拦截器按类型分组，适用于基于图、函数式和规划器 agent 管道。为了降低噪音并优化实际特性开发时的成本，
只注册该特性需要的拦截器。

Agent 和环境生命周期：

- `interceptEnvironmentCreated`：在创建 agent 环境时转换该环境。
- `interceptAgentStarting`：在 agent run 开始前调用。
- `interceptAgentCompleted`：在 agent run 成功完成时调用。
- `interceptAgentExecutionFailed`：在 agent run 失败时调用。
- `interceptAgentClosing`：在 agent run 关闭之前调用（清理点）。

策略生命周期：

- `interceptStrategyStarting`：在策略执行开始前调用。
- `interceptStrategyCompleted`：在策略执行成功完成时调用。

LLM 调用生命周期：

- `interceptLLMCallStarting`：在 LLM 调用前调用。
- `interceptLLMCallFailed`：在 LLM 调用失败时调用（底层 prompt executor 或 moderation 调用抛出异常）。
- `interceptLLMCallCompleted`：在 LLM 调用后调用。

LLM 流式传输生命周期：

- `interceptLLMStreamingStarting`：在流式传输开始前调用。
- `interceptLLMStreamingFrameReceived`：对每个接收到的流帧调用。
- `interceptLLMStreamingFailed`：在流式传输失败时调用。
- `interceptLLMStreamingCompleted`：在流式传输完成后调用。

工具调用生命周期：

- `interceptToolCallStarting`：在工具调用前调用。
- `interceptToolValidationFailed`：在工具输入验证失败时调用。
- `interceptToolCallFailed`：在工具执行失败时调用。
- `interceptToolCallCompleted`：在工具完成后调用（带结果）。

#### 基于图的 agent 专用拦截器

以下拦截器仅在 `AIAgentGraphPipeline` 上可用，可让你观察节点和子图生命周期事件。

节点执行生命周期：

- `interceptNodeExecutionStarting`：在节点开始执行前调用。
- `interceptNodeExecutionCompleted`：在节点完成执行后调用。
- `interceptNodeExecutionFailed`：在节点执行出错失败时调用。

子图执行生命周期：

- `interceptSubgraphExecutionStarting`：在子图开始执行前调用。
- `interceptSubgraphExecutionCompleted`：在子图执行完成后调用。
- `interceptSubgraphExecutionFailed`：在子图执行失败时调用。

要让特性处理特定类型的事件，它需要注册相应的管道拦截器。

### 过滤 agent 事件

在 agent 中安装特性时，你可能不想处理该特性中注册的所有事件。要
过滤掉某些事件，可以使用 [FeatureConfig.setEventFilter](https://api.koog.ai/agents/agents-core/ai.koog.agents.core.feature.config/-feature-config/set-event-filter.html) 函数应用过滤器。

以下示例展示如何只允许某个特性的 LLM 调用开始和结束事件：

```
install(MyFeature) {
    setEventFilter { context ->
        context.eventType is AgentLifecycleEventType.LLMCallStarting ||
            context.eventType is AgentLifecycleEventType.LLMCallCompleted
    }
}
```

#### 为特性禁用事件过滤

如果你的特性逻辑依赖完整的 agent 事件结构，事件过滤可能导致意外行为。为
避免这种情况，你需要在实现特性时禁用事件过滤：在特性配置中重写 `setEventFilter`，忽略安装特性时设置的任何自定义过滤器。

依赖处理完整 agent 事件流的特性示例是 [OpenTelemetry](../open-telemetry/)，因为它使用
完整的 agent 事件结构来组合继承式的 span 结构。

以下是为特性禁用事件过滤的示例：

```
class MyFeatureConfig : FeatureConfig() {
    override fun setEventFilter(filter: (AgentLifecycleEventContext) -> Boolean) {
        // Deactivate event filtering for the feature
        throw UnsupportedOperationException("Event filtering is not allowed.")
    }
}
```

## 示例：基本日志特性

以下示例展示如何实现一个记录 agent 生命周期事件的基本日志特性。由于该特性
应可用于基于图、函数式和规划器 agent，因此所有 agent 类型通用的拦截器
在 `installCommon` 方法中实现，以避免代码重复。特定于各个
agent 类型的拦截器在 `installGraphPipeline`、`installFunctionalPipeline` 和 `installPlannerPipeline`
方法中实现。

```
class LoggingFeature(val loggerName: String) {
    class Config : FeatureConfig() {
        var loggerName: String = "agent-logs"
    }

    companion object Feature :
        AIAgentGraphFeature<Config, LoggingFeature>,
        AIAgentFunctionalFeature<Config, LoggingFeature>,
        AIAgentPlannerFeature<Config, LoggingFeature> {

        override val key = createStorageKey<LoggingFeature>("logging-feature")

        override fun createInitialConfig(agentConfig: AIAgentConfig): Config = Config()

        override fun install(config: Config, pipeline: AIAgentGraphPipeline) : LoggingFeature {
            val logging = LoggingFeature(config.loggerName)
            val logger = KotlinLogging.logger(config.loggerName)

            installGraphPipeline(pipeline, logger)

            return logging
        }

        override fun install(config: Config, pipeline: AIAgentFunctionalPipeline) : LoggingFeature {
            val logging = LoggingFeature(config.loggerName)
            val logger = KotlinLogging.logger(config.loggerName)

            installFunctionalPipeline(pipeline, logger)

            return logging
        }

        override fun install(config: Config, pipeline: AIAgentPlannerPipeline) : LoggingFeature {
            val logging = LoggingFeature(config.loggerName)
            val logger = KotlinLogging.logger(config.loggerName)

            installPlannerPipeline(pipeline, logger)

            return logging
        }

        private fun installCommon(
            pipeline: AIAgentPipeline,
            logger: KLogger,
        ) {
            pipeline.interceptAgentStarting(this) { e ->
                logger.info { "Agent starting: runId=${e.runId}" }
            }
            pipeline.interceptStrategyStarting(this) { e ->
                logger.info { "Strategy ${e.strategy.name} starting" }
            }
            pipeline.interceptLLMCallStarting(this) { e ->
                logger.info { "Making LLM call with ${e.tools.size} tools" }
            }
            pipeline.interceptLLMCallCompleted(this) { e ->
                logger.info { "Received response: ${e.response != null}" }
            }
        }

        private fun installGraphPipeline(
            pipeline: AIAgentGraphPipeline,
            logger: KLogger,
        ) {
            installCommon(pipeline, logger)

            pipeline.interceptNodeExecutionStarting(this) { e ->
                logger.info { "Node ${e.node.name} input: ${e.input}" }
            }
            pipeline.interceptNodeExecutionCompleted(this) { e ->
                logger.info { "Node ${e.node.name} output: ${e.output}" }
            }
        }

        private fun installFunctionalPipeline(
            pipeline: AIAgentFunctionalPipeline,
            logger: KLogger
        ) {
            installCommon(pipeline, logger)
        }

        private fun installPlannerPipeline(
            pipeline: AIAgentPlannerPipeline,
            logger: KLogger
        ) {
            installCommon(pipeline, logger)
        }
    }
}
```

以下示例展示如何在 agent 中安装自定义日志特性。该示例展示了基本特性
安装，以及自定义配置属性 `loggerName`，它允许你指定 logger 的名称：

```
val agent = AIAgent(
    promptExecutor = simpleOpenAIExecutor(System.getenv("OPENAI_API_KEY")),
    systemPrompt = "You are a helpful assistant. Answer user questions concisely.",
    llmModel = OpenAIModels.Chat.GPT4o
) {
    install(LoggingFeature) {
        loggerName = "my-custom-logger"
    }
}

agent.run("What is Kotlin?")
```
