# 事件处理程序

你可以使用 event handlers 在 agent workflow 期间监控并响应特定事件，用于日志记录、测试、调试以及扩展 agent 行为。

## 功能概览

EventHandler feature 让你可以 hook 到各种 agent 事件。它充当事件委托机制，能够：

- 管理 AI agent 操作的生命周期。
- 提供 hook，用于监控和响应 workflow 的不同阶段。
- 支持错误处理和恢复。
- 便于跟踪工具调用并处理结果。

### 安装和配置

EventHandler feature 通过 `EventHandler` 类与 agent workflow 集成。
该类提供了一种为不同 agent 事件注册 callback 的方式，并且可以作为 feature 安装到 agent 配置中。详情请参阅 [API-reference](https://api.koog.ai/agents/agents-features/agents-features-event-handler/ai.koog.agents.features.eventHandler.feature/-event-handler/index.html)。

要为 agent 安装该 feature 并配置 event handlers，请执行以下操作：

KotlinJava

```
handleEvents {
    // Handle tool calls
    onToolCallStarting { eventContext ->
        println("Tool called: ${eventContext.toolName} with args ${eventContext.toolArgs}")
    }
    // Handle event triggered when the agent completes its execution
    onAgentCompleted { eventContext ->
        println("Agent finished with result: ${eventContext.result}")
    }

    // Other event handlers
}
```

```
AIAgent<String, String> agent = AIAgent.builder()
    .promptExecutor(simpleOllamaAIExecutor("http://localhost:11434"))
    .llmModel(OllamaModels.Meta.LLAMA_3_2)
    .install(EventHandler.Feature, cfg -> {
        // Handle tool calls
        cfg.onToolCallStarting(ctx -> {
            System.out.println("Tool called: " + ctx.getToolName() + " with args " + ctx.getToolArgs());
        });
        // Handle event triggered when the agent completes its execution
        cfg.onAgentCompleted(ctx -> {
            System.out.println("Agent finished with result: " + ctx.getResult());
        });
    })
    .build();
```

有关 event handler 配置的更多详细信息，请参阅 [API-reference](https://api.koog.ai/agents/agents-features/agents-features-event-handler/ai.koog.agents.features.eventHandler.feature/-event-handler-config/index.html)。

创建 agent 时，你也可以使用 `handleEvents` extension function 设置 event handlers。
该函数同样会安装 event handler feature，并为 agent 配置 event handlers。示例如下：

KotlinJava

```
val agent = AIAgent(
    promptExecutor = simpleOllamaAIExecutor(),
    llmModel = OllamaModels.Meta.LLAMA_3_2,
){
    handleEvents {
        // Handle tool calls
        onToolCallStarting { eventContext ->
            println("Tool called: ${eventContext.toolName} with args ${eventContext.toolArgs}")
        }
        // Handle event triggered when the agent completes its execution
        onAgentCompleted { eventContext ->
            println("Agent finished with result: ${eventContext.result}")
        }

        // Other event handlers
    }
}
```

```
AIAgent<String, String> agent = AIAgent.builder()
    .promptExecutor(simpleOllamaAIExecutor("http://localhost:11434"))
    .llmModel(OllamaModels.Meta.LLAMA_3_2)
    .install(EventHandler.Feature, cfg -> {
        // Handle tool calls
        cfg.onToolCallStarting(ctx -> {
            System.out.println("Tool called: " + ctx.getToolName() + " with args " + ctx.getToolArgs());
        });
        // Handle event triggered when the agent completes its execution
        cfg.onAgentCompleted(ctx -> {
            System.out.println("Agent finished with result: " + ctx.getResult());
        });
    })
    .build();
```