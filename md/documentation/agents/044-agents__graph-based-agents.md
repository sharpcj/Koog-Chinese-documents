# 基于图的代理

使用基于图的代理，你可以将行为建模为显式状态机：
图策略的节点表示操作（LLM 调用、工具执行），
边表示节点之间的数据流。

基于图的代理的主要优势是：

- 易于可视化
- 状态持久化
- 可组合的架构

前提条件

确保你的环境和项目满足以下要求：

- JDK 17+
- Kotlin 2.2.0+
- Gradle 8.0+ 或 Maven 3.8+

将 [Koog 包](https://central.sonatype.com/artifact/ai.koog/koog-agents/) 添加为依赖项：

Gradle (Kotlin)Gradle (Groovy)Maven

build.gradle.kts

```
dependencies {
    // Stable
    implementation("ai.koog:koog-agents:1.3.0")

    // Beta
    implementation("ai.koog:koog-agents-additions:1.3.0-beta")
}
```

build.gradle

```
dependencies {
    // Stable
    implementation 'ai.koog:koog-agents:1.3.0'

    // Beta
    implementation 'ai.koog:koog-agents-additions:1.3.0-beta'
}
```

pom.xml

```
<dependency>
    <!-- Stable -->
    <dependency>
        <groupId>ai.koog</groupId>
        <artifactId>koog-agents-jvm</artifactId>
        <version>1.3.0</version>
    </dependency>

    <!-- Beta -->
    <dependency>
        <groupId>ai.koog</groupId>
        <artifactId>koog-agents-additions-jvm</artifactId>
        <version>1.3.0-beta</version>
    </dependency>
</dependency>
```

从 LLM 提供商处获取 API 密钥，或通过 Ollama 运行本地 LLM。
有关更多信息，请参阅[快速入门](../../quickstart/)。

本页中的示例假设你通过 Ollama 在本地运行 Llama 3.2。

本页介绍如何重新创建[基础代理](../basic-agents/)所使用的策略图。
它向 LLM 发送请求，然后要么输出响应（如果 LLM 以助手消息响应），
要么执行工具（如果 LLM 请求工具调用）。
在工具调用的情况下，代理将工具结果发送给 LLM，
然后要么输出响应，要么执行工具。

以下是该策略图的图示：

```
---
config:
  flowchart:
    defaultRenderer: "elk"
---
graph TB
    subgraph nodeStart
        Input
    end

    subgraph nodeFinish
        Output
    end

    subgraph nodeSendInput
        llmRequest(Request LLM)
    end

    subgraph nodeExecuteTool
        executeTool(Execute tool call)
    end

    subgraph nodeSendToolResult
        sendToolResult(Request LLM)
    end

    Input --String--> llmRequest
    llmRequest --Message.Assistant--> onToolCalls{{onToolCalls}}
    llmRequest --Message.Assistant--> onTextMessage{{onTextMessage}}
    onTextMessage --String--> Output
    onToolCalls --ToolCalls--> executeTool --ReceivedToolResults--> sendToolResult
    sendToolResult --Message.Assistant--> onToolCalls
    sendToolResult --Message.Assistant--> onTextMessage
```

## 构建策略图

在 Koog 中，你使用 [`AIAgentGraphStrategyBuilder`](https://api.koog.ai/agents/agents-core/ai.koog.agents.core.dsl.builder/-a-i-agent-graph-strategy-builder/index.html) 实现策略。
就像每个节点都有输入和输出类型一样，
策略整体也定义了某种输入和输出类型。
此示例假设输入和输出类型为字符串，
这意味着实现此策略的代理将期望一个字符串并返回一个字符串。

要创建策略，请使用 [`strategy()`](https://api.koog.ai/agents/agents-core/ai.koog.agents.core.dsl.builder/strategy.html) 函数，以两个泛型作为输入和输出类型，
为策略提供唯一标识符，并定义节点和边。

KotlinJava

```
val calculatorAgentStrategy = strategy<String, String>("Simple calculator") {
    val nodeSendInput by nodeLLMRequest()
    val nodeExecuteTool by nodeExecuteTools()
    val nodeSendToolResult by nodeLLMSendToolResults()

    edge(nodeStart forwardTo nodeSendInput)
    edge(nodeSendInput forwardTo nodeFinish onTextMessage { true })
    edge(nodeSendInput forwardTo nodeExecuteTool onToolCalls { true })
    edge(nodeExecuteTool forwardTo nodeSendToolResult)
    edge(nodeSendToolResult forwardTo nodeFinish onTextMessage { true })
    edge(nodeSendToolResult forwardTo nodeExecuteTool onToolCalls { true })
}
```

```
var calculatorAgentStrategy = AIAgentGraphStrategy.builder("Simple calculator")
    .withInput(String.class)
    .withOutput(String.class);

var nodeSendInput = AIAgentNode.llmRequest("nodeSendInput");
var nodeExecuteTool = AIAgentNode.executeTools("nodeExecuteTool");
var nodeSendToolResult = AIAgentNode.llmSendToolResults("nodeSendToolResult");

calculatorAgentStrategy.edge(AIAgentEdge.builder()
    .from(calculatorAgentStrategy.nodeStart)
    .to(nodeSendInput)
    .build());
calculatorAgentStrategy.edge(AIAgentEdge.builder()
    .from(nodeSendInput)
    .to(calculatorAgentStrategy.nodeFinish)
    .onTextMessage()
    .build());
calculatorAgentStrategy.edge(AIAgentEdge.builder()
    .from(nodeSendInput)
    .to(nodeExecuteTool)
    .onToolCalls()
    .build());
calculatorAgentStrategy.edge(nodeExecuteTool, nodeSendToolResult);
calculatorAgentStrategy.edge(AIAgentEdge.builder()
    .from(nodeSendToolResult)
    .to(calculatorAgentStrategy.nodeFinish)
    .onTextMessage()
    .build());
calculatorAgentStrategy.edge(AIAgentEdge.builder()
    .from(nodeSendToolResult)
    .to(nodeExecuteTool)
    .onToolCalls()
    .build());
```

此示例仅使用[预定义节点](../../nodes-and-components/)，
但你也可以创建[自定义节点](../../custom-nodes/)。

每个策略图都必须有一条从 `nodeStart` 到 `nodeFinish` 的路径，由[边](../../custom-strategy-graphs/#edges)连接。
边可以带有条件，以确定何时沿特定边前进。
边还可以在将前一个节点的输出传递给下一个节点之前对其进行转换。
这对于连接输出和输入类型不匹配的节点是必要的。

在前面的示例中，`onToolCalls { true }` 表示仅当前一个节点返回了
包含至少一个工具调用（`MessagePart.Tool.Call`）的助手消息时，边才会前进。

使用 `onTextMessage { true }` 时，仅当前一个节点返回了
包含文本部分（`MessagePart.Text`）的助手消息时，边才会前进。
此函数还会提取并连接这些部分的文本内容，
从而有效地将 `Message.Assistant` 转换为 `String`，因为 `nodeFinish` 期望一个字符串。

提示

除了 `onTextMessage { true }`，你还可以这样做：

```
onMessageParts(MessagePart.Text::class) transformed { it.joinToString("\n") { part -> part.text } }
```

或者：

```
onCondition { it is Message.Assistant } transformed { (it as Message.Assistant).parts.filterIsInstance<MessagePart.Text>().joinToString("\n") { part -> part.text } }
```

## 创建并运行代理

让我们使用此策略创建一个代理实例并运行它：

KotlinJava

```
val calculatorAgentStrategy = strategy<String, String>("Simple calculator") {
    val nodeSendInput by nodeLLMRequest()
    val nodeExecuteTool by nodeExecuteTools()
    val nodeSendToolResult by nodeLLMSendToolResults()

    edge(nodeStart forwardTo nodeSendInput)
    edge(nodeSendInput forwardTo nodeFinish onTextMessage { true })
    edge(nodeSendInput forwardTo nodeExecuteTool onToolCalls { true })
    edge(nodeExecuteTool forwardTo nodeSendToolResult)
    edge(nodeSendToolResult forwardTo nodeFinish onTextMessage { true })
    edge(nodeSendToolResult forwardTo nodeExecuteTool onToolCalls { true })
}

val mathAgent = AIAgent(
    promptExecutor = simpleOllamaAIExecutor(),
    llmModel = OllamaModels.Meta.LLAMA_3_2,
    strategy = calculatorAgentStrategy
)

fun main() = runBlocking {
    val result = mathAgent.run("Multiply 3 by 4, then multiply the result by 5, then add 10, then add 123.")
    println(result)
}
```

```
var calculatorAgentStrategy = AIAgentGraphStrategy.builder("Simple calculator")
    .withInput(String.class)
    .withOutput(String.class);

var nodeSendInput = AIAgentNode.llmRequest("nodeSendInput");
var nodeExecuteTool = AIAgentNode.executeTools("nodeExecuteTool");
var nodeSendToolResult = AIAgentNode.llmSendToolResults("nodeSendToolResult");

calculatorAgentStrategy.edge(AIAgentEdge.builder()
    .from(calculatorAgentStrategy.nodeStart)
    .to(nodeSendInput)
    .build());
calculatorAgentStrategy.edge(AIAgentEdge.builder()
    .from(nodeSendInput)
    .to(calculatorAgentStrategy.nodeFinish)
    .onTextMessage()
    .build());
calculatorAgentStrategy.edge(AIAgentEdge.builder()
    .from(nodeSendInput)
    .to(nodeExecuteTool)
    .onToolCalls()
    .build());
calculatorAgentStrategy.edge(nodeExecuteTool, nodeSendToolResult);
calculatorAgentStrategy.edge(AIAgentEdge.builder()
    .from(nodeSendToolResult)
    .to(calculatorAgentStrategy.nodeFinish)
    .onTextMessage()
    .build());
calculatorAgentStrategy.edge(AIAgentEdge.builder()
    .from(nodeSendToolResult)
    .to(nodeExecuteTool)
    .onToolCalls()
    .build());

var promptExecutor = PromptExecutor.builder()
    .ollama("http://localhost:11434")
    .build();

AIAgent<String, String> mathAgent = AIAgent.builder()
    .promptExecutor(promptExecutor)
    .llmModel(OllamaModels.Meta.LLAMA_3_2)
    .graphStrategy(calculatorAgentStrategy.build())
    .build();

    String result = mathAgent.run("Multiply 3 by 4, then multiply the result by 5, then add 10, then add 123.", null);
    System.out.println(result);
```

当你运行此代理时，它会返回类似这样的内容：

```
To calculate this, I'll follow the order of operations:

1. Multiply 3 by 4: 3 * 4 = 12
2. Multiply the result by 5: 12 * 5 = 60
3. Add 10: 60 + 10 = 70
4. Add 123: 70 + 123 = 193

The final answer is 193.
```

然而，由于此代理没有任何工具，LLM 永远不会返回工具调用，
而只是生成整个答案。
实际发生的情况如下：

```
---
config:
  flowchart:
    defaultRenderer: "elk"
---
graph LR
    subgraph nodeStart
        Input
    end

    subgraph nodeFinish
        Output
    end

    subgraph nodeSendInput
        llmRequest(Request LLM)
    end

    Input --String--> llmRequest --Message.Assistant--> onTextMessage{{onTextMessage}} --String--> Output
```

尽管在这种情况下是正确的，但答案将取决于底层 LLM 的算术能力。
为了确保计算正确，我们应该为代理提供数学工具。
这样 LLM 就能够决定调用以确定性方式执行计算的工具。

## 添加工具

定义用于执行数学运算的[工具](../../tools/)，并将它们添加到 [ToolRegistry](https://api.koog.ai/agents/agents-tools/ai.koog.agents.core.tools/-tool-registry/index.html)：

KotlinJava

```
@LLMDescription("Tools for performing math operations")
class MathTools : ToolSet {
    @Tool
    @LLMDescription("Adds two numbers and returns the result")
    fun add(a: Int, b: Int): Int {
        // This is not necessary, but it helps to see the tool call in the console output
        println("Adding $a and $b...")
        return a + b
    }
    @Tool
    @LLMDescription("Multiplies two numbers and returns the result")
    fun multiply(a: Int, b: Int): Int {
        // This is not necessary, but it helps to see the tool call in the console output
        println("Multiplying $a and $b...")
        return a * b
    }
}

val toolRegistry = ToolRegistry {
    tools(MathTools())
}
```

```
@LLMDescription("Tools for performing math operations")
public static class MathTools implements ToolSet {
    @Tool
    @LLMDescription("Adds two numbers and returns the result")
    public int add(int a, int b) {
        // This is not necessary, but it helps to see the tool call in the console output
        System.out.println("Adding " + a + " and " + b + "...");
        return a + b;
    }

    @Tool
    @LLMDescription("Multiplies two numbers and returns the result")
    public int multiply(int a, int b) {
        // This is not necessary, but it helps to see the tool call in the console output
        System.out.println("Multiplying " + a + " and " + b + "...");
        return a * b;
    }
}
public static void main(String[] args) {
    ToolRegistry toolRegistry = ToolRegistry.builder()
        .tools(new MathTools())
        .build();
}
```

将工具注册表添加到代理配置中：

KotlinJava

```
val mathAgent = AIAgent(
    promptExecutor = simpleOllamaAIExecutor(),
    llmModel = OllamaModels.Meta.LLAMA_3_2,
    strategy = calculatorAgentStrategy,
    toolRegistry = toolRegistry
)

fun main() = runBlocking {
    val result = mathAgent.run("Multiply 3 by 4, then multiply the result by 5, then add 10, then add 123.")
    println(result)
}
```

```
AIAgent<String, String> mathAgent = AIAgent.builder()
    .promptExecutor(promptExecutor)
    .llmModel(OllamaModels.Meta.LLAMA_3_2)
    .graphStrategy(calculatorAgentStrategy.build())
    .toolRegistry(toolRegistry)
    .build();

String result = mathAgent.run("Multiply 3 by 4, then multiply the result by 5, then add 10, then add 123.", null);
System.out.println(result);
```

现在当你运行代理时，它会返回类似这样的内容：

```
Multiplying 3 and 4...
The output from the first operation was multiplied by 5:
5 * 12 = 60

Then, 10 was added to the result:
60 + 10 = 70

Finally, 123 was added to the result:
70 + 123 = 193
```

根据此输出，代理正确地执行了计算，但它只调用了一次 `multiply` 工具，
而不是为每个操作调用相应的工具。
我们可以通过在系统提示中描述代理的角色并提供使用适当工具的说明来帮助代理。

## 提供系统提示

[系统提示](../../prompts/prompt-creation/#system-message)定义了代理的角色和执行任务的说明。
在我们的示例中，描述代理应如何处理复杂的多步计算非常重要：

KotlinJava

```
val mathAgent = AIAgent(
    promptExecutor = simpleOllamaAIExecutor(),
    llmModel = OllamaModels.Meta.LLAMA_3_2,
    systemPrompt = """
                You are a simple calculator assistant.
                You can add and multiply two numbers using the 'add' and 'multiply' tools.
                When the user provides input, extract the numbers and operations they requested.
                Use the appropriate tool for the first operation, then the next one, and so on, until you calculate the result.
                Always respond with a clear, friendly message showing the calculation and result.
                """.trimIndent(),
    toolRegistry = toolRegistry,
    strategy = calculatorAgentStrategy
)

fun main() = runBlocking {
    val result = mathAgent.run("Multiply 3 by 4, then multiply the result by 5, then add 10, then add 123.")
    println(result)
}
```

```
AIAgent<String, String> mathAgent = AIAgent.builder()
    .promptExecutor(promptExecutor)
    .llmModel(OllamaModels.Meta.LLAMA_3_2)
    .systemPrompt("You are a simple calculator assistant. You can add and multiply two numbers using the 'add' and 'multiply' tools. When the user provides input, extract the numbers and operations they requested. Use the appropriate tool for the first operation, then the next one, and so on, until you calculate the result. Always respond with a clear, friendly message showing the calculation and result.")
    .graphStrategy(calculatorAgentStrategy.build())
    .toolRegistry(toolRegistry)
    .build();

String result = mathAgent.run("Multiply 3 by 4, then multiply the result by 5, then add 10, then add 123.", null);
System.out.println(result);
```

现在当你运行代理时，它会返回类似这样的内容：

```
Multiplying 3 and 4...
Multiplying 12 and 5...
Adding 60 and 10...
Adding 70 and 123...
The final result is: 193
```

如你所见，代理现在会为每个操作正确调用相应的工具，
确保以确定性方式执行计算，而不是冒着产生幻觉结果的风险。

## 后续步骤

- 与[函数式代理](../functional-agents/)和[规划器代理](../planner-agents/)进行比较
- 通过[安装功能](../../features/)增强你的代理
- 使用[结构化输出](../../structured-output/)提高可预测性和可靠性
