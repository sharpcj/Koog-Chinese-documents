# 测试

## 概述

测试功能为在 Koog 框架中测试 AI 代理管道、子图和工具交互提供了一个全面的框架。它使开发者能够创建受控的测试环境，包括模拟 LLM（大型语言模型）执行器、工具注册表和代理环境。

### 目的

此功能的主要目的是通过以下方式促进基于代理的 AI 功能的测试：

- 模拟 LLM 对特定提示的响应
- 模拟工具调用及其结果
- 测试代理管道子图及其结构
- 验证数据通过代理节点的正确流动
- 为预期行为提供断言

## 配置与初始化

### 设置测试依赖项

在设置测试环境之前，请确保已添加以下依赖项：

```
// build.gradle.kts
dependencies {
   testImplementation("ai.koog:agents-test:LATEST_VERSION")
   testImplementation(kotlin("test"))
}
```

### 模拟 LLM 响应

测试的基本形式涉及模拟 LLM 响应以确保确定性行为。你可以使用 `MockLLMBuilder` 及相关工具来实现。

KotlinJava

```
// Create a mock LLM executor
val mockLLMApi = getMockExecutor {
  // Mock a simple text response
  mockLLMAnswer("Hello!") onRequestContains "Hello"

  // Mock a default response
  mockLLMAnswer("I don't know how to answer that.").asDefaultResponse
}
```

```
import ai.koog.agents.core.tools.ToolRegistry;
import ai.koog.agents.testing.tools.MockExecutor;
import ai.koog.prompt.executor.model.PromptExecutor;

// Create a tool registry (empty)
ToolRegistry toolRegistry = ToolRegistry.builder().build();

// Create a mock LLM executor
PromptExecutor mockLLMApi = MockExecutor.builder()
    .toolRegistry(toolRegistry)
    .mockLLMAnswer("Hello!").onRequestContains("Hello")
    .mockLLMAnswer("I don't know how to answer that.").asDefaultResponse()
    .build();
```

### 模拟工具调用

你可以模拟 LLM 根据输入模式调用特定工具：

KotlinJava

```
// Mock a tool call response
mockLLMToolCall(CreateTool, CreateTool.Args("solve")) onRequestEquals "Solve task"

// Mock tool behavior - simplest form without lambda
mockTool(PositiveToneTool) alwaysReturns "The text has a positive tone."

// Using lambda when you need to perform extra actions
mockTool(NegativeToneTool) alwaysTells {
  // Perform some extra action
  println("Negative tone tool called")

  // Return the result
  "The text has a negative tone."
}

// Mock tool behavior based on specific arguments
mockTool(AnalyzeTool) returns "Detailed analysis" onArguments AnalyzeTool.Args("analyze deeply")

// Mock tool behavior with conditional argument matching
mockTool(SearchTool) returns "Found results" onArgumentsMatching { args ->
  args.query.contains("important")
}
```

```

```

上述示例演示了从简单到复杂的多种模拟工具的方式：

1. `alwaysReturns`：最简单的形式，直接返回一个值，无需 lambda。
2. `alwaysTells`：当你需要执行额外操作时使用 lambda。
3. `returns...onArguments`：为精确匹配的参数返回特定结果。
4. `returns...onArgumentsMatching`：根据自定义参数条件返回结果。

### 启用测试模式

要在代理上启用测试模式，请在 AIAgent 构造函数块中使用 `withTesting()` 函数：

KotlinJava

```
// Create the agent with testing enabled
AIAgent(
    promptExecutor = mockLLMApi,
    toolRegistry = toolRegistry,
    llmModel = llmModel
) {
    // Enable testing mode
    withTesting()
}
```

```

```

## 高级测试

### 测试图结构

在测试详细的节点行为和边连接之前，验证代理图的整体结构非常重要。这包括检查所有必需的节点是否存在，以及在预期的子图中是否正确连接。

测试功能提供了一种全面的方式来测试代理的图结构。这种方法对于具有多个子图和互连节点的复杂代理尤其有价值。

#### 基本结构测试

首先验证代理图的基本结构：

KotlinJava

```
AIAgent(
    // Constructor arguments
    promptExecutor = mockLLMApi,
    toolRegistry = toolRegistry,
    llmModel = llmModel
) {
    testGraph<String, String>("test") {
        val firstSubgraph = assertSubgraphByName<String, String>("first")
        val secondSubgraph = assertSubgraphByName<String, String>("second")

        // Assert subgraph connections
        assertEdges {
            startNode() alwaysGoesTo firstSubgraph
            firstSubgraph alwaysGoesTo secondSubgraph
            secondSubgraph alwaysGoesTo finishNode()
        }

        // Verify the first subgraph
        verifySubgraph(firstSubgraph) {
            val start = startNode()
            val finish = finishNode()

            // Assert nodes by name
            val askLLM = assertNodeByName<String, Message.Assistant>("callLLM")
            val callTool = assertNodeByName<ToolCalls, ReceivedToolResults>("executeTool")

            // Assert node reachability
            assertReachable(start, askLLM)
            assertReachable(askLLM, callTool)
        }
    }
}
```

```

```

### 测试节点行为

节点行为测试让你能够验证代理图中的节点对于给定输入是否产生预期输出。这对于确保代理逻辑在不同场景下正确工作至关重要。

#### 基本节点测试

从对单个节点进行简单的输入和输出验证开始：

KotlinJava

```
assertNodes {

    // Test basic text responses
    askLLM withInput "Hello" outputs assistantMessage("Hello!")

    // Test tool call responses
    askLLM withInput "Solve task" outputs assistantMessage(CreateTool, CreateTool.Args("solve"))
}
```

```

```

上述示例展示了如何测试以下行为：
1. 当 LLM 节点接收到 `Hello` 作为输入时，它响应一条简单的文本消息。
2. 当它接收到 `Solve task` 时，它响应一个工具调用。

#### 测试工具运行节点

你还可以测试运行工具的节点：

KotlinJava

```
assertNodes {
    // Test tool runs with specific arguments
    callTool withInput ToolCalls(listOf(toolCallMessagePart(
        SolveTool,
        SolveTool.Args("solve")
    ))) outputs ReceivedToolResults(listOf(toolResult(SolveTool, SolveTool.Args("solve"), "solved")))
}
```

```

```

这验证了当工具执行节点接收到特定的工具调用签名时，它会产生预期的工具结果。

#### 高级节点测试

对于更复杂的场景，你可以测试具有结构化输入和输出的节点：

KotlinJava

```
assertNodes {
    // Test with different inputs to the same node
    askLLM withInput "Simple query" outputs assistantMessage("Simple response")

    // Test with complex parameters
    askLLM withInput "Complex query with parameters" outputs assistantMessage(
        AnalyzeTool,
        AnalyzeTool.Args(query = "parameters", depth = 3)
    )
}
```

```

```

你还可以测试具有详细结果结构的复杂工具调用场景：

KotlinJava

```
assertNodes {
    // Test a complex tool call with a structured result
    callTool withInput ToolCalls(listOf(toolCallMessagePart(
        AnalyzeTool,
        AnalyzeTool.Args(query = "complex", depth = 5)
    ))) outputs ReceivedToolResults(listOf(toolResult(AnalyzeTool, AnalyzeTool.Args(query = "complex", depth = 5), AnalyzeTool.Result(
        analysis = "Detailed analysis",
        confidence = 0.95,
        metadata = mapOf("source" to "database", "timestamp" to "2023-06-15")
    ))))
}
```

```

```

这些高级测试有助于确保你的节点正确处理复杂的数据结构，这对于复杂的代理行为至关重要。

### 测试边连接

边连接测试允许你验证代理图是否正确地将一个节点的输出路由到适当的下一节点。这确保你的代理根据不同的输出遵循预期的工作流路径。

#### 基本边测试

从简单的边连接测试开始：

KotlinJava

```
assertEdges {
    // Test text message routing
    askLLM withOutput assistantMessage("Hello!") goesTo giveFeedback

    // Test tool call routing
    askLLM withOutput assistantMessage(CreateTool, CreateTool.Args("solve")) goesTo callTool
}
```

```

```

此示例验证以下行为：
1. 当 LLM 节点输出一条简单的文本消息时，流程被定向到 `giveFeedback` 节点。
2. 当它输出一个工具调用时，流程被定向到 `callTool` 节点。

#### 测试条件路由

你可以基于输出内容测试更复杂的路由逻辑：

KotlinJava

```
assertEdges {
    // Different text responses can route to different nodes
    askLLM withOutput assistantMessage("Need more information") goesTo askForInfo
    askLLM withOutput assistantMessage("Ready to proceed") goesTo processRequest
}
```

```

```

#### 高级边测试

对于复杂的代理，你可以基于工具结果中的结构化数据测试条件路由：

KotlinJava

```
assertEdges {
    // Test routing based on tool result content
    callTool withOutput ReceivedToolResults(listOf(toolResult(
        AnalyzeTool,
        AnalyzeTool.Args(query = "parameters", depth = 3),
        AnalyzeTool.Result(analysis = "Needs more processing", confidence = 0.5)
    ))) goesTo processResult
}
```

```

```

你还可以基于不同的结果属性测试复杂的决策路径：

KotlinJava

```
assertEdges {
    // Route to different nodes based on confidence level
    callTool withOutput ReceivedToolResults(listOf(toolResult(
        AnalyzeTool,
        AnalyzeTool.Args(query = "parameters", depth = 3),
        AnalyzeTool.Result(analysis = "Complete", confidence = 0.9)
    ))) goesTo finish

    callTool withOutput ReceivedToolResults(listOf(toolResult(
        AnalyzeTool,
        AnalyzeTool.Args(query = "parameters", depth = 3),
        AnalyzeTool.Result(analysis = "Uncertain", confidence = 0.3)
    ))) goesTo verifyResult
}
```

```

```

这些高级边测试有助于确保你的代理根据节点输出的内容和结构做出正确的决策，这对于创建智能的、上下文感知的工作流至关重要。

## 完整测试示例

以下是一个演示完整测试场景的用户故事：

你正在开发一个语气分析代理，它分析文本的语气并提供反馈。该代理使用工具来检测积极、消极和中性语气。

以下是如何测试此代理：

KotlinJava

```
@Test
fun testToneAgent() = runTest {
    // Create a list to track tool calls
    var toolCalls = mutableListOf<String>()
    var result: String? = null

    // Create a tool registry
    val toolRegistry = ToolRegistry {
        // A special tool, required with this type of agent
        tool(SayToUser)

        with(ToneTools) {
            tools()
        }
    }

    // Create an event handler
    val eventHandler = EventHandler {
        onToolCallStarting { tool, args ->
            println("[DEBUG_LOG] Tool called: tool ${tool.name}, args $args")
            toolCalls.add(tool.name)
        }

        handleError {
            println("[DEBUG_LOG] An error occurred: ${it.message}\n${it.stackTraceToString()}")
            true
        }

        handleResult {
            println("[DEBUG_LOG] Result: $it")
            result = it
        }
    }

    val positiveText = "I love this product!"
    val negativeText = "Awful service, hate the app."
    val defaultText = "I don't know how to answer this question."

    val positiveResponse = "The text has a positive tone."
    val negativeResponse = "The text has a negative tone."
    val neutralResponse = "The text has a neutral tone."

    val mockLLMApi = getMockExecutor(toolRegistry, eventHandler) {
        // Set up LLM responses for different input texts
        mockLLMToolCall(NeutralToneTool, ToneTool.Args(defaultText)) onRequestEquals defaultText
        mockLLMToolCall(PositiveToneTool, ToneTool.Args(positiveText)) onRequestEquals positiveText
        mockLLMToolCall(NegativeToneTool, ToneTool.Args(negativeText)) onRequestEquals negativeText

        // Mock the behavior where the LLM responds with just tool responses when the tools return results
        mockLLMAnswer(positiveResponse) onRequestContains positiveResponse
        mockLLMAnswer(negativeResponse) onRequestContains negativeResponse
        mockLLMAnswer(neutralResponse) onRequestContains neutralResponse

        mockLLMAnswer(defaultText).asDefaultResponse

        // Tool mocks
        mockTool(PositiveToneTool) alwaysTells {
            toolCalls += "Positive tone tool called"
            positiveResponse
        }
        mockTool(NegativeToneTool) alwaysTells {
            toolCalls += "Negative tone tool called"
            negativeResponse
        }
        mockTool(NeutralToneTool) alwaysTells {
            toolCalls += "Neutral tone tool called"
            neutralResponse
        }
    }

    // Create a strategy
    val strategy = toneStrategy("tone_analysis")

    // Create an agent configuration
    val agentConfig = AIAgentConfig(
        prompt = prompt("test-agent") {
            system(
                """
                You are an question answering agent with access to the tone analysis tools.
                You need to answer 1 question with the best of your ability.
                Be as concise as possible in your answers.
                DO NOT ANSWER ANY QUESTIONS THAT ARE BESIDES PERFORMING TONE ANALYSIS!
                DO NOT HALLUCINATE!
            """.trimIndent()
            )
        },
        model = mockk<LLModel>(relaxed = true),
        maxAgentIterations = 10
    )

    // Create an agent with testing enabled
    val agent = AIAgent(
        promptExecutor = mockLLMApi,
        toolRegistry = toolRegistry,
        strategy = strategy,
        eventHandler = eventHandler,
        agentConfig = agentConfig,
    ) {
        withTesting()
    }

    // Test the positive text
    agent.run(positiveText)
    assertEquals("The text has a positive tone.", result, "Positive tone result should match")
    assertEquals(1, toolCalls.size, "One tool is expected to be called")

    // Test the negative text
    agent.run(negativeText)
    assertEquals("The text has a negative tone.", result, "Negative tone result should match")
    assertEquals(2, toolCalls.size, "Two tools are expected to be called")

    //Test the neutral text
    agent.run(defaultText)
    assertEquals("The text has a neutral tone.", result, "Neutral tone result should match")
    assertEquals(3, toolCalls.size, "Three tools are expected to be called")
}
```

```

```

对于具有多个子图的更复杂代理，你还可以测试图结构：

KotlinJava

```
@Test
fun testMultiSubgraphAgentStructure() = runTest {
    val strategy = strategy("test") {
        val firstSubgraph by subgraph(
            "first",
            tools = listOf(DummyTool, CreateTool, SolveTool)
        ) {
            val callLLM by nodeLLMRequest(allowToolCalls = false)
            val executeTool by nodeExecuteTools()
            val sendToolResult by nodeLLMSendToolResults()
            val giveFeedback by node<String, String> { input ->
                llm.writeSession {
                    appendPrompt {
                        user("Call tools! Don't chat!")
                    }
                }
                input
            }

            edge(nodeStart forwardTo callLLM)
            edge(callLLM forwardTo executeTool onToolCalls { true })
            edge(callLLM forwardTo giveFeedback onTextMessage { true })
            edge(giveFeedback forwardTo giveFeedback transformed { it })
            edge(executeTool forwardTo nodeFinish transformed { it.toolResults.first().output })
        }

        val secondSubgraph by subgraph<String, String>("second") {
            edge(nodeStart forwardTo nodeFinish)
        }

        edge(nodeStart forwardTo firstSubgraph)
        edge(firstSubgraph forwardTo secondSubgraph)
        edge(secondSubgraph forwardTo nodeFinish)
    }

    val toolRegistry = ToolRegistry {
        tool(DummyTool)
        tool(CreateTool)
        tool(SolveTool)
    }

    val mockLLMApi = getMockExecutor(toolRegistry) {
        mockLLMAnswer("Hello!") onRequestContains "Hello"
        mockLLMToolCall(CreateTool, CreateTool.Args("solve")) onRequestEquals "Solve task"
    }

    val basePrompt = prompt("test") {}

    AIAgent(
        toolRegistry = toolRegistry,
        strategy = strategy,
        eventHandler = EventHandler {},
        agentConfig = AIAgentConfig(prompt = basePrompt, model = OpenAIModels.Chat.GPT4o, maxAgentIterations = 100),
        promptExecutor = mockLLMApi,
    ) {
        testGraph("test") {
            val firstSubgraph = assertSubgraphByName<String, String>("first")
            val secondSubgraph = assertSubgraphByName<String, String>("second")

            assertEdges {
                startNode() alwaysGoesTo firstSubgraph
                firstSubgraph alwaysGoesTo secondSubgraph
                secondSubgraph alwaysGoesTo finishNode()
            }

            verifySubgraph(firstSubgraph) {
                val start = startNode()
                val finish = finishNode()

                val askLLM = assertNodeByName<String, Message.Assistant>("callLLM")
                val callTool = assertNodeByName<ToolCalls, ReceivedToolResults>("executeTool")
                val giveFeedback = assertNodeByName<Any?, Any?>("giveFeedback")

                assertReachable(start, askLLM)
                assertReachable(askLLM, callTool)

                assertNodes {
                    askLLM withInput "Hello" outputs assistantMessage("Hello!")
                    askLLM withInput "Solve task" outputs assistantMessage(CreateTool, CreateTool.Args("solve"))

                    callTool withInput ToolCalls(listOf(toolCallMessagePart(
                        SolveTool,
                        SolveTool.Args("solve")
                    ))) outputs ReceivedToolResults(listOf(toolResult(SolveTool, SolveTool.Args("solve"), "solved")))

                    callTool withInput ToolCalls(listOf(toolCallMessagePart(
                        CreateTool,
                        CreateTool.Args("solve")
                    ))) outputs ReceivedToolResults(listOf(toolResult(CreateTool, CreateTool.Args("solve"), "created")))
                }

                assertEdges {
                    askLLM withOutput assistantMessage("Hello!") goesTo giveFeedback
                    askLLM withOutput assistantMessage(CreateTool, CreateTool.Args("solve")) goesTo callTool
                }
            }
        }
    }
}
```

```

```

## API 参考

有关测试功能的完整 API 参考，请参阅 [agents-test](https://api.koog.ai/agents/agents-test/index.html) 模块的参考文档。

## 常见问题与故障排除

#### 如何模拟特定的工具响应？

使用 `MockLLMBuilder` 中的 `mockTool` 方法：

KotlinJava

```
val mockExecutor = getMockExecutor {
    mockTool(myTool) alwaysReturns myResult

    // Or with conditions
    mockTool(myTool) returns myResult onArguments myArgs
}
```

```

```

#### 如何测试复杂的图结构？

使用子图断言、`verifySubgraph` 和节点引用：

KotlinJava

```
testGraph<Unit, String>("test") {
    val mySubgraph = assertSubgraphByName<Unit, String>("mySubgraph")

    verifySubgraph(mySubgraph) {
        // Get references to nodes
        val nodeA = assertNodeByName<Unit, String>("nodeA")
        val nodeB = assertNodeByName<String, String>("nodeB")

        // Assert reachability
        assertReachable(nodeA, nodeB)

        // Assert edge connections
        assertEdges {
            nodeA.withOutput("result") goesTo nodeB
        }
    }
}
```

```

```

#### 如何根据输入模拟不同的 LLM 响应？

使用模式匹配方法：

KotlinJava

```
getMockExecutor {
    mockLLMAnswer("Response A") onRequestContains "topic A"
    mockLLMAnswer("Response B") onRequestContains "topic B"
    mockLLMAnswer("Exact response") onRequestEquals "exact question"
    mockLLMAnswer("Conditional response") onCondition { it.contains("keyword") && it.length > 10 }
}
```

```
import ai.koog.agents.testing.tools.MockExecutor;
import ai.koog.prompt.executor.model.PromptExecutor;

PromptExecutor promptExecutor = MockExecutor.builder()
    .mockLLMAnswer("Response A").onRequestContains("topic A")
    .mockLLMAnswer("Response B").onRequestContains("topic B")
    .mockLLMAnswer("Exact response").onRequestEquals("exact question")
    .mockLLMAnswer("Conditional response").onCondition(s -> s.contains("keyword") && s.length() > 10)
    .build();
```

### 故障排除

#### 模拟执行器总是返回默认响应

检查你的模式匹配是否正确。模式区分大小写，必须与指定的完全匹配。

#### 工具调用未被拦截

确保：

1. 工具注册表已正确设置。
2. 工具名称完全匹配。
3. 工具操作已正确配置。

#### 图断言失败

1. 验证节点名称是否正确。
2. 检查图结构是否符合你的预期。
3. 使用 `startNode()` 和 `finishNode()` 方法获取正确的入口和出口点。
