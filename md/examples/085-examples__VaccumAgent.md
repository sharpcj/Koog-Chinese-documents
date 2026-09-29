# 构建一个简单的吸尘器 Agent

[在 GitHub 上打开](https://github.com/JetBrains/koog/blob/develop/examples/notebooks/VaccumAgent.ipynb)
 [下载 .ipynb](https://raw.githubusercontent.com/JetBrains/koog/develop/examples/notebooks/VaccumAgent.ipynb)

在本 notebook 中，我们将探索如何使用新的 Kotlin agents framework 实现一个基本的 reflex agent。
我们的示例是经典的“vacuum world”问题——
一个有两个位置的简单环境，每个位置可以是 clean 或 dirty，而 agent 需要清理它们。

首先，让我们理解环境模型：

```
import kotlin.random.Random

/**
 * Represents a simple vacuum world with two locations (A and B).
 *
 * The environment tracks:
 * - The current location of the vacuum agent ('A' or 'B')
 * - The cleanliness status of each location (true = dirty, false = clean)
 */
class VacuumEnv {
    var location: Char = 'A'
        private set

    private val status = mutableMapOf(
        'A' to Random.nextBoolean(),
        'B' to Random.nextBoolean()
    )

    fun percept(): Pair<Char, Boolean> = location to status.getValue(location)

    fun clean(): String {
        status[location] = false
        return "cleaned"
    }

    fun moveLeft(): String {
        location = 'A'
        return "move to A"
    }

    fun moveRight(): String {
        location = 'B'
        return "move to B"
    }

    fun isClean(): Boolean = status.values.all { it }

    fun worldLayout(): String = "${status.keys}"

    override fun toString(): String = "location=$location, dirtyA=${status['A']}, dirtyB=${status['B']}"
}
```

`VacuumEnv` 类对我们的简单世界建模：
- 两个位置由字符 'A' 和 'B' 表示
- 每个位置可以是 clean 或 dirty（随机初始化）
- agent 在任意时刻可以位于其中一个位置
- agent 可以感知自己的当前位置以及该位置是否 dirty
- agent 可以执行操作：移动到指定位置，或清理当前位置

## 为 Vacuum Agent 创建工具

现在，让我们定义 AI agent 用于与环境交互的工具：

```
import ai.koog.agents.core.tools.annotations.LLMDescription
import ai.koog.agents.core.tools.annotations.Tool
import ai.koog.agents.core.tools.reflect.ToolSet

/**
 * Provides tools for the LLM agent to control the vacuum robot.
 * All methods either mutate or read from the VacuumEnv passed to the constructor.
 */
@LLMDescription("Tools for controlling a two-cell vacuum world")
class VacuumTools(private val env: VacuumEnv) : ToolSet {

    @Tool
    @LLMDescription("Returns current location and whether it is dirty")
    fun sense(): String {
        val (loc, dirty) = env.percept()
        return "location=$loc, dirty=$dirty, locations=${env.worldLayout()}"
    }

    @Tool
    @LLMDescription("Cleans the current cell")
    fun clean(): String = env.clean()

    @Tool
    @LLMDescription("Moves the agent to cell A")
    fun moveLeft(): String = env.moveLeft()

    @Tool
    @LLMDescription("Moves the agent to cell B")
    fun moveRight(): String = env.moveRight()
}
```

`VacuumTools` 类在我们的 LLM agent 和环境之间创建一个接口：

- 它实现了 Kotlin AI Agents framework 中的 `ToolSet`
- 每个工具都用 `@Tool` 注解，并带有面向 LLM 的描述
- 这些工具允许 agent 感知环境并执行操作
- 每个方法都会返回一个描述操作结果的字符串

## 设置 Agent

接下来，我们将配置并创建 AI agent：

```
import ai.koog.agents.core.agent.AIAgent
import ai.koog.agents.core.agent.config.AIAgentConfig
import ai.koog.agents.core.tools.ToolRegistry
import ai.koog.agents.core.tools.reflect.asTools
import ai.koog.agents.ext.agent.chatAgentStrategy
import ai.koog.agents.ext.tool.AskUser
import ai.koog.agents.ext.tool.SayToUser
import ai.koog.prompt.dsl.prompt
import ai.koog.prompt.executor.clients.openai.OpenAIModels
import ai.koog.prompt.executor.llms.all.simpleOpenAIExecutor
import ai.koog.prompt.params.LLMParams

val env = VacuumEnv()
val apiToken = System.getenv("OPENAI_API_KEY") ?: error("OPENAI_API_KEY environment variable not set")
val executor = simpleOpenAIExecutor(apiToken = apiToken)

val toolRegistry = ToolRegistry {
    tool(SayToUser)
    tool(AskUser)
    tools(VacuumTools(env).asTools())
}

val systemVacuumPrompt = """
    You are a reflex vacuum-cleaner agent living in a two-cell world labelled A and B.
    Your goal: make both cells clean, using the provided tools.
    First, call sense() to inspect where you are. Then decide: if dirty → clean(); else moveLeft()/moveRight().
    Continue until both cells are clean, then tell the user "done".
    Use sayToUser to inform the user about each step.
""".trimIndent()

val agentConfig = AIAgentConfig(
    prompt = prompt("chat", params = LLMParams(temperature = 1.0)) {
        system(systemVacuumPrompt)
    },
    model = OpenAIModels.Chat.GPT4o,
    maxAgentIterations = 50,
)

val agent = AIAgent(
    promptExecutor = executor,
    strategy = chatAgentStrategy(),
    agentConfig = agentConfig,
    toolRegistry = toolRegistry
)
```

在此设置中：

1. 我们创建环境实例
2. 我们设置到 OpenAI 的 GPT-4o model 的连接
3. 我们注册 agent 可以使用的工具
4. 我们定义 system prompt，为 agent 提供目标和行为规则
5. 我们使用带有 chat strategy 的 `AIAgent` 构造函数创建 agent

## 运行 Agent

最后，让我们运行 agent：

```
import kotlinx.coroutines.runBlocking

runBlocking {
    agent.run("Start cleaning, please")
}
```

```
Agent says: Currently in cell A. It's already clean.
Agent says: Moved to cell B. It's already clean.
```

运行这段代码时：

1. agent 收到开始清理的初始 prompt
2. 它使用工具感知环境并做出决策
3. 它会持续清理，直到两个单元格都 clean
4. 在整个过程中，它会不断告知用户自己正在做什么

```
// Finally we can validate that the work is finished by printing the env state

env
```

```
location=B, dirtyA=false, dirtyB=false
```