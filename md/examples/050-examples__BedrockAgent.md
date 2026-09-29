# 使用 AWS Bedrock 和 Koog 框架构建 AI Agent

[在 GitHub 上打开](https://github.com/JetBrains/koog/blob/develop/examples/notebooks/BedrockAgent.ipynb)
 [下载 .ipynb](https://raw.githubusercontent.com/JetBrains/koog/develop/examples/notebooks/BedrockAgent.ipynb)

欢迎阅读这份关于使用 Koog 框架并集成 AWS Bedrock 来创建智能 AI agent 的完整指南。在这个 notebook 中，我们将逐步构建一个可工作的 agent，它能通过自然语言命令控制一个简单的开关设备。

## 你将学到什么

- 如何使用 Kotlin 注解为 AI agent 定义自定义工具
- 为基于 LLM 的 agent 设置 AWS Bedrock 集成
- 创建工具注册表并将其连接到 agent
- 构建能够理解并执行命令的交互式 agent

## 前提条件

- 具有适当权限的 AWS Bedrock 访问权限
- 已配置 AWS 凭证（access key 和 secret key）
- 对 Kotlin 协程有基本了解

让我们开始构建第一个由 Bedrock 驱动的 AI agent！

```
%useLatestDescriptors
// %use koog
```

```
import ai.koog.agents.core.tools.annotations.LLMDescription
import ai.koog.agents.core.tools.annotations.Tool
import ai.koog.agents.core.tools.reflect.ToolSet

// Simple state-holding device that our agent will control
class Switch {
    private var state: Boolean = false

    fun switch(on: Boolean) {
        state = on
    }

    fun isOn(): Boolean {
        return state
    }
}

/**
 * ToolSet implementation that exposes switch operations to the AI agent.
 *
 * Key concepts:
 * - @Tool annotation marks methods as callable by the agent
 * - @LLMDescription provides natural language descriptions for the LLM
 * - ToolSet interface allows grouping related tools together
 */
class SwitchTools(val switch: Switch) : ToolSet {

    @Tool
    @LLMDescription("Switches the state of the switch to on or off")
    fun switchState(state: Boolean): String {
        switch.switch(state)
        return "Switch turned ${if (state) "on" else "off"} successfully"
    }

    @Tool
    @LLMDescription("Returns the current state of the switch (on or off)")
    fun getCurrentState(): String {
        return "Switch is currently ${if (switch.isOn()) "on" else "off"}"
    }
}
```

```
import ai.koog.agents.core.tools.ToolRegistry
import ai.koog.agents.core.tools.reflect.asTools

// Create our switch instance
val switch = Switch()

// Build the tool registry with our switch tools
val toolRegistry = ToolRegistry {
    // Convert our ToolSet to individual tools and register them
    tools(SwitchTools(switch).asTools())
}

println("✅ Tool registry created with ${toolRegistry.tools.size} tools:")
toolRegistry.tools.forEach { tool ->
    println("  - ${tool.name}")
}
```

```
✅ Tool registry created with 2 tools:
  - getCurrentState
  - switchState
```

```
import ai.koog.prompt.executor.clients.bedrock.BedrockClientSettings
import ai.koog.prompt.executor.clients.bedrock.BedrockRegions

val region = BedrockRegions.US_WEST_2.regionCode
val maxRetries = 3

// Configure Bedrock client settings
val bedrockSettings = BedrockClientSettings(
    region = region, // Choose your preferred AWS region
    maxRetries = maxRetries // Number of retry attempts for failed requests
)

println("🌐 Bedrock configured for region: $region")
println("🔄 Max retries set to: $maxRetries")
```

```
🌐 Bedrock configured for region: us-west-2
🔄 Max retries set to: 3
```

```
import ai.koog.prompt.executor.llms.all.simpleBedrockExecutor

// Create the Bedrock LLM executor with credentials from environment
val executor = simpleBedrockExecutor(
    awsAccessKeyId = System.getenv("AWS_BEDROCK_ACCESS_KEY")
        ?: throw IllegalStateException("AWS_BEDROCK_ACCESS_KEY environment variable not set"),
    awsSecretAccessKey = System.getenv("AWS_BEDROCK_SECRET_ACCESS_KEY")
        ?: throw IllegalStateException("AWS_BEDROCK_SECRET_ACCESS_KEY environment variable not set"),
    settings = bedrockSettings
)

println("🔐 Bedrock executor initialized successfully")
println("💡 Pro tip: Set AWS_BEDROCK_ACCESS_KEY and AWS_BEDROCK_SECRET_ACCESS_KEY environment variables")
```

```
🔐 Bedrock executor initialized successfully
💡 Pro tip: Set AWS_BEDROCK_ACCESS_KEY and AWS_BEDROCK_SECRET_ACCESS_KEY environment variables
```

```
import ai.koog.agents.core.agent.AIAgent
import ai.koog.prompt.executor.clients.bedrock.BedrockModels

val agent = AIAgent(
    executor = executor,
    llmModel = BedrockModels.AnthropicClaude35SonnetV2, // State-of-the-art reasoning model
    systemPrompt = """
        You are a helpful assistant that controls a switch device.

        You can:
        - Turn the switch on or off when requested
        - Check the current state of the switch
        - Explain what you're doing

        Always be clear about the switch's current state and confirm actions taken.
    """.trimIndent(),
    temperature = 0.1, // Low temperature for consistent, focused responses
    toolRegistry = toolRegistry
)

println("🤖 AI Agent created successfully!")
println("📋 System prompt configured")
println("🛠️  Tools available: ${toolRegistry.tools.size}")
println("🎯 Model: ${BedrockModels.AnthropicClaude35SonnetV2}")
println("🌡️  Temperature: 0.1 (focused responses)")
```

```
🤖 AI Agent created successfully!
📋 System prompt configured
🛠️  Tools available: 2
🎯 Model: LLModel(provider=Bedrock, id=us.anthropic.claude-3-5-sonnet-20241022-v2:0, capabilities=[Temperature, Tools, ToolChoice, Image, Document, Completion], contextLength=200000, maxOutputTokens=8192)
🌡️  Temperature: 0.1 (focused responses)
```

```
import kotlinx.coroutines.runBlocking

println("🎉 Bedrock Agent with Switch Tools - Ready to Go!")
println("💬 You can ask me to:")
println("   • Turn the switch on/off")
println("   • Check the current switch state")
println("   • Ask questions about the switch")
println()
println("💡 Example: 'Please turn on the switch' or 'What's the current state?'")
println("📝 Type your request:")

val input = readln()
println("\n🤖 Processing your request...")

runBlocking {
    val response = agent.run(input)
    println("\n✨ Agent response:")
    println(response)
}
```

```
🎉 Bedrock Agent with Switch Tools - Ready to Go!
💬 You can ask me to:
   • Turn the switch on/off
   • Check the current switch state
   • Ask questions about the switch

💡 Example: 'Please turn on the switch' or 'What's the current state?'
📝 Type your request:

The execution was interrupted
```

## 刚才发生了什么？🎯

当你运行 agent 时，幕后会发生如下过程：

1. **自然语言处理**：你的输入通过 Bedrock 发送给 Claude 3.5 Sonnet
2. **意图识别**：模型理解你想对开关执行什么操作
3. **工具选择**：agent 根据你的请求决定调用哪些工具
4. **动作执行**：在你的开关对象上调用相应的工具方法
5. **响应生成**：agent 用自然语言说明发生了什么

这展示了 Koog 框架的核心能力——在自然语言理解和程序化动作之间实现无缝集成。

## 后续步骤与扩展

准备进一步探索了吗？下面是一些可以尝试的想法：

### 🔧 增强工具

```
@Tool
@LLMDescription("Sets a timer to automatically turn off the switch after specified seconds")
fun setAutoOffTimer(seconds: Int): String

@Tool
@LLMDescription("Gets the switch usage statistics and history")
fun getUsageStats(): String
```

### 🌐 多设备

```
class HomeAutomationTools : ToolSet {
    @Tool fun controlLight(room: String, on: Boolean): String
    @Tool fun setThermostat(temperature: Double): String
    @Tool fun lockDoor(doorName: String): String
}
```

### 🧠 记忆与上下文

```
val agent = AIAgent(
    executor = executor,
    // ... other config
    features = listOf(
        MemoryFeature(), // Remember past interactions
        LoggingFeature()  // Track all actions
    )
)
```

### 🔄 高级工作流

```
// Multi-step workflows with conditional logic
@Tool
@LLMDescription("Executes evening routine: dims lights, locks doors, sets thermostat")
fun eveningRoutine(): String
```

## 关键要点

✅ **工具就是函数**：任何 Kotlin 函数都可以成为 agent 能力
✅ **注解决定行为**：@Tool 和 @LLMDescription 让函数可被发现
✅ **ToolSets 组织能力**：将相关工具按逻辑分组
✅ **Registries 是工具箱**：ToolRegistry 包含所有可用的 agent 能力
✅ **Agents 编排一切**：AIAgent 将 LLM 智能与工具组合在一起

Koog 框架让构建复杂 AI agent 变得非常直接，这些 agent 能理解自然语言并执行现实世界动作。从简单场景开始，然后按需添加更多工具和功能来扩展 agent 的能力。

**祝你构建 agent 愉快！** 🚀

## 测试 Agent

现在看看我们的 agent 实际运行效果！agent 现在可以理解自然语言请求，并使用我们提供的工具来控制开关。

**试试这些命令：**
- "Turn on the switch"
- "What's the current state?"
- "Switch it off please"
- "Is the switch on or off?"