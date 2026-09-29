# 使用 Koog 构建猜数字 Agent

[在 GitHub 上打开](https://github.com/JetBrains/koog/blob/develop/examples/notebooks/Guesser.ipynb)
 [下载 .ipynb](https://raw.githubusercontent.com/JetBrains/koog/develop/examples/notebooks/Guesser.ipynb)

让我们构建一个小巧但有趣的 agent，用来猜测你心里想的数字。我们将依靠 Koog 的工具调用来提出有针对性的问题，并使用经典的二分搜索策略逐步收敛。最终得到的是一个符合 Kotlin 习惯的 Kotlin Notebook，可以直接放入文档中。

我们会保持代码最小化、流程透明：几个很小的工具、一个紧凑的提示，以及一个交互式 CLI 循环。

## 设置

此 notebook 假设：
- 你正在带有 Koog 的 Kotlin Notebook 中运行。
- 已设置环境变量 `OPENAI_API_KEY`。agent 会通过 `simpleOpenAIExecutor(System.getenv("OPENAI_API_KEY"))` 使用它。

加载 Koog kernel：

```
%useLatestDescriptors
%use koog
```

## 工具：提出有针对性的问题

工具是 LLM 可以调用的小型、描述清晰的函数。我们将提供三个工具：
- `lessThan(value)`：“你的数字是否小于 value？”
- `greaterThan(value)`：“你的数字是否大于 value？”
- `proposeNumber(value)`：“你的数字是否等于 value？”（在范围足够小时使用）

每个工具都返回一个简单的 "YES"/"NO" 字符串。辅助函数 `ask` 实现了一个最小化的 Y/n 循环并验证输入。通过 `@LLMDescription` 给出的描述可以帮助模型正确选择工具。

```
import ai.koog.agents.core.tools.annotations.Tool

class GuesserTool : ToolSet {

    @Tool
    @LLMDescription("Asks the user if his number is STRICTLY less than a given value.")
    fun lessThan(
        @LLMDescription("A value to compare the guessed number with.") value: Int
    ): String = ask("Is your number less than $value?", value)

    @Tool
    @LLMDescription("Asks the user if his number is STRICTLY greater than a given value.")
    fun greaterThan(
        @LLMDescription("A value to compare the guessed number with.") value: Int
    ): String = ask("Is your number greater than $value?", value)

    @Tool
    @LLMDescription("Asks the user if his number is EXACTLY equal to the given number. Only use this tool once you've narrowed down your answer.")
    fun proposeNumber(
        @LLMDescription("A value to compare the guessed number with.") value: Int
    ): String = ask("Is your number equal to $value?", value)

    fun ask(question: String, value: Int): String {
        print("$question [Y/n]: ")
        val input = readln()
        println(input)

        return when (input.lowercase()) {
            "", "y", "yes" -> "YES"
            "n", "no" -> "NO"
            else -> {
                println("Invalid input! Please, try again.")
                ask(question, value)
            }
        }
    }
}
```

## Tool Registry

将你的工具暴露给 agent。我们还添加了内置的 `SayToUser` 工具，以便 agent 可以直接向用户展示消息。

```
val toolRegistry = ToolRegistry {
    tool(SayToUser)
    tools(GuesserTool())
}
```

## Agent 配置

我们只需要一个简短、以工具为中心的 system prompt。我们会建议采用二分搜索策略，并保持 `temperature = 0.0`，以获得稳定、确定性的行为。这里我们使用 OpenAI 的推理模型 `GPT4oMini` 来进行清晰的规划。

```
val agent = AIAgent(
    executor = simpleOpenAIExecutor(System.getenv("OPENAI_API_KEY")),
    llmModel = OpenAIModels.Chat.GPT4oMini,
    systemPrompt = """
            You are a number guessing agent. Your goal is to guess a number that the user is thinking of.

            Follow these steps:
            1. Start by asking the user to think of a number between 1 and 100.
            2. Use the less_than and greater_than tools to narrow down the range.
                a. If it's neither greater nor smaller, use the propose_number tool.
            3. Once you're confident about the number, use the propose_number tool to check if your guess is correct.
            4. If your guess is correct, congratulate the user. If not, continue guessing.

            Be efficient with your guessing strategy. A binary search approach works well.
        """.trimIndent(),
    temperature = 0.0,
    toolRegistry = toolRegistry
)
```

## 运行它

- 想一个 1 到 100 之间的数字。
- 输入 `start` 开始。
- 用 `Y`/`Enter` 表示是，或用 `n` 表示否，来回答 agent 的问题。agent 应该能在约 7 步内锁定你的数字。

```
import kotlinx.coroutines.runBlocking

println("Number Guessing Game started!")
println("Think of a number between 1 and 100, and I'll try to guess it.")
println("Type 'start' to begin the game.")

val initialMessage = readln()
runBlocking {
    agent.run(initialMessage)
}
```

## 工作原理

- agent 读取 system prompt 并规划二分搜索。
- 在每次迭代中，它会调用你的一个工具：`lessThan`、`greaterThan`，或在确定时调用 `proposeNumber`。
- 辅助函数 `ask` 收集你的 Y/n 输入，并向模型返回干净的 "YES"/"NO" 信号。
- 当它得到确认后，会通过 `SayToUser` 祝贺你。

## 扩展它

- 通过调整 system prompt 更改范围（例如 1..1000）。
- 添加 `between(low, high)` 工具，以进一步减少调用次数。
- 替换模型或 executor（例如使用 Ollama executor 和本地模型），同时保持相同的工具。
- 将猜测或结果持久化到存储中，用于分析。

## 故障排除

- 缺少 key：确保已在你的环境中设置 `OPENAI_API_KEY`。
- 找不到 kernel：确保 `%useLatestDescriptors` 和 `%use koog` 已成功执行。
- 未调用工具：确认 `ToolRegistry` 包含 `GuesserTool()`，并且 prompt 中的名称与你的工具函数匹配。