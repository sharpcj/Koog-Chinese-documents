# Unity + Koog：从 Kotlin Agent 驱动你的游戏

[在 GitHub 上打开](https://github.com/JetBrains/koog/blob/develop/examples/notebooks/UnityMcp.ipynb)
 [下载 .ipynb](https://raw.githubusercontent.com/JetBrains/koog/develop/examples/notebooks/UnityMcp.ipynb)

本 notebook 将引导你使用 Model Context Protocol (MCP)，通过 Koog 构建一个熟悉 Unity 的 AI agent。我们会连接到 Unity MCP server、发现工具、使用 LLM 制定计划，并对你打开的场景执行操作。

> 前提条件
> - 一个已安装 Unity-MCP server plugin 的 Unity 项目
> - JDK 17+
> - OPENAI\_API\_KEY 环境变量中的 OpenAI API key

```
%useLatestDescriptors
%use koog
```

```
lateinit var process: Process
```

## 1) 提供你的 OpenAI API key

我们从 `OPENAI_API_KEY` 环境变量读取 API key，这样可以避免把 secret 放进 notebook。

```
val token = System.getenv("OPENAI_API_KEY") ?: error("OPENAI_API_KEY environment variable not set")
val executor = simpleOpenAIExecutor(token)
```

## 2) 配置 Unity agent

我们为 Unity 定义一个紧凑的 system prompt 和 agent 设置。

```
val agentConfig = AIAgentConfig(
    prompt = prompt("cook_agent_system_prompt") {
        system {
            "You are a Unity assistant. You can execute different tasks by interacting with tools from the Unity engine."
        }
    },
    model = OpenAIModels.Chat.GPT4o,
    maxAgentIterations = 1000
)
```

```

```

## 3) 启动 Unity MCP server

我们将从你的 Unity 项目目录启动 Unity MCP server，并通过 stdio 连接。

```
// https://github.com/IvanMurzak/Unity-MCP
val pathToUnityProject = "path/to/unity/project"
val process = ProcessBuilder(
    "$pathToUnityProject/com.ivanmurzak.unity.mcp.server/bin~/Release/net9.0/com.IvanMurzak.Unity.MCP.Server",
    "60606"
).start()
```

## 4) 从 Koog 连接并运行 agent

我们从 Unity MCP server 发现工具，构建一个小型的先计划策略，并运行一个只使用工具来修改你当前打开场景的 agent。

```
import kotlinx.coroutines.runBlocking

runBlocking {
    // Create the ToolRegistry with tools from the MCP server
    val toolRegistry = McpToolRegistryProvider.fromTransport(
        transport = McpToolRegistryProvider.defaultStdioTransport(process)
    )

    toolRegistry.tools.forEach {
        println(it.name)
        println(it.descriptor)
    }

    val strategy = strategy<String, String>("unity_interaction") {
        val nodePlanIngredients by nodeLLMRequest(allowToolCalls = false)
        val interactionWithUnity by subgraphWithTask<String, String>(
            // work with plan
            tools = toolRegistry.tools,
        ) { input ->
            "Start interacting with Unity according to the plan: $input"
        }

        edge(
            nodeStart forwardTo nodePlanIngredients transformed {
                "Create detailed plan for " + agentInput + "" +
                    "using the following tools: ${toolRegistry.tools.joinToString("\n") {
                        it.name + "\ndescription:" + it.descriptor
                    }}"
            }
        )
        edge(nodePlanIngredients forwardTo interactionWithUnity onAssistantMessage { true })
        edge(interactionWithUnity forwardTo nodeFinish)
    }

    val agent = AIAgent(
        promptExecutor = executor,
        strategy = strategy,
        agentConfig = agentConfig,
        toolRegistry = toolRegistry,
        installFeatures = {
            install(Tracing)

            install(EventHandler) {
                onAgentStarting { eventContext ->
                    println("OnAgentStarting first (strategy: ${strategy.name})")
                }

                onAgentStarting { eventContext ->
                    println("OnAgentStarting second (strategy: ${strategy.name})")
                }

                onAgentCompleted { eventContext ->
                    println(
                        "OnAgentCompleted (agent id: ${eventContext.agent.id}, result: ${eventContext.result})"
                    )
                }
            }
        }
    )

    val result = agent.run(
        " extend current opened scene for the towerdefence game. " +
            "Add more placements for the towers, change the path for the enemies"
    )

    result
}
```

## 5) 关闭 MCP 进程

始终在运行结束时清理外部 Unity MCP server 进程。

```
// Shutdown the Unity MCP process
process.destroy()
```