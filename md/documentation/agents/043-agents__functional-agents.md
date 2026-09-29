# 函数式代理

使用函数式代理，你可以将逻辑实现为一个函数，该函数处理用户输入、与 LLM 交互、在必要时调用工具，并生成最终输出。
与[基于图的代理](../graph-based-agents/)相比，
这通常意味着更快的原型开发，但有以下缺点：

- 不易可视化
- 无状态持久化

前提条件

确保你的环境和项目满足以下要求：

- JDK 17+
- Kotlin 2.2.0+
- Gradle 8.0+ 或 Maven 3.8+

将 [Koog 包](https://central.sonatype.com/artifact/ai.koog/koog-agents/)添加为依赖项：

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

本页中的示例假设你正在通过 Ollama 在本地运行 Llama 3.2。

本页介绍如何实现函数式策略，以便为你的代理快速原型开发一些自定义逻辑。

## 创建最小函数式代理

要创建最小函数式代理，请使用与[基础代理](../basic-agents/)相同的 [`AIAgent`](https://api.koog.ai/agents/agents-core/ai.koog.agents.core.agent/-a-i-agent/index.html) 接口，并向其传递一个
[`AIAgentFunctionalStrategy`](https://api.koog.ai/agents/agents-core/ai.koog.agents.core.agent/-a-i-agent-functional-strategy/index.html) 实例。
你可以定义一个函数式策略，它接收输入并返回输出，进行一次 LLM 调用，然后返回
响应中助手消息的内容。

在 Kotlin 中，最方便的方式是使用 `functionalStrategy {...}` DSL 方法。在 Java 中，你可以使用 `AIAgent` 构建器上的 `functionalStrategy` 方法。

KotlinJava

```
val strategy = functionalStrategy<String, String> { input ->
    val response = requestLLM(input)
    response.parts.filterIsInstance<MessagePart.Text>().joinToString("\n") { it.text }
}

val mathAgent = AIAgent(
    promptExecutor = simpleOllamaAIExecutor(),
    llmModel = OllamaModels.Meta.LLAMA_3_2,
    strategy = strategy
)

fun main() = runBlocking {
    val result = mathAgent.run("What is 12 × 9?")
    println(result)
}
```

```
AIAgent<String, String> mathAgent = AIAgent.builder()
    .promptExecutor(SimpleLLMExecutorsKt.simpleOllamaAIExecutor("http://localhost:11434"))
    .llmModel(OllamaModels.Meta.LLAMA_3_2)
    .functionalStrategy("mathStrategy", (AIAgentFunctionalContext context, String input) -> {
        Message.Response response = context.requestLLM(input);
        if (response instanceof Message.Assistant) {
            return ((Message.Assistant) response).getContent();
        }
        return "";
    })
    .build();

String result = mathAgent.run("What is 12 × 9?");
System.out.println(result);
```

代理可以生成以下输出：

```
The answer to 12 × 9 is 108.
```

## 进行顺序 LLM 调用

你可以扩展之前的策略，以进行多次顺序 LLM 调用：

KotlinJava

```
fun Message.Assistant.text(): String =
    parts.filterIsInstance<MessagePart.Text>().joinToString("\n") { it.text }

val strategy = functionalStrategy<String, String> { input ->
    // The first LLM call produces an initial draft based on the user input
    val draft = requestLLM("Draft: $input").text()
    // The second LLM call improves the initial draft
    val improved = requestLLM("Improve and clarify.").text()
    // The final LLM call formats the improved text and returns the result
    requestLLM("Format the result as bold.").text()
}
```

```
AIAgent<String, String> mathAgent = AIAgent.builder()
    .promptExecutor(simpleOllamaAIExecutor("http://localhost:11434"))
    .systemPrompt("You are a precise math assistant.")
    .llmModel(OllamaModels.Meta.LLAMA_3_2)
    .functionalStrategy((AIAgentFunctionalContext context, String input) -> {
        // The first LLM call produces an initial draft based on the user input
        Message.Response draftResponse = context.requestLLM("Draft: " + input);
        String draft = "";
        if (draftResponse instanceof Message.Assistant) {
            draft = ((Message.Assistant) draftResponse).getContent();
        }

        // The second LLM call improves the initial draft
        Message.Response improvedResponse = context.requestLLM("Improve and clarify.");
        String improved = "";
        if (improvedResponse instanceof Message.Assistant) {
            improved = ((Message.Assistant) improvedResponse).getContent();
        }

        // The final LLM call formats the improved text and returns the result
        Message.Response finalResponse = context.requestLLM("Format the result as bold.");
        if (finalResponse instanceof Message.Assistant) {
            return ((Message.Assistant) finalResponse).getContent();
        }
        return "";
    })
    .build();
```

代理可以生成以下输出：

```
To calculate the product of 12 and 9, we multiply these two numbers together.

12 × 9 = **108**
```

## 添加工具

在许多情况下，函数式代理需要完成特定任务，
例如读取和写入数据、调用 API 或执行其他确定性操作。
在 Koog 中，你可以将此类能力公开为[工具](../../tools/)，并让 LLM 决定何时调用它们。

以下是需要执行的操作：

1. 创建一个[基于注解的工具](../../tools/annotation-based-tools/)。
2. 将其添加到工具注册表，并将该注册表传递给代理。
3. 确保代理策略能够识别 LLM 响应中的工具调用，执行请求的工具，
   将其结果发送回 LLM，并重复此过程，直到没有剩余的工具调用。

KotlinJava

```
@LLMDescription("Tools for performing math operations")
class MathTools : ToolSet {
    @Tool
    @LLMDescription("Multiplies two numbers and returns the result")
    fun multiply(a: Int, b: Int): Int {
        // This is not necessary, but it helps to see the tool call in the console output
        println("Multiplying $a and $b...")
        return a * b
    }
}

val toolRegistry = ToolRegistry {
    tool(MathTools()::multiply)
}

val strategy = functionalStrategy<String, String> { input ->
    // Send the user input to the LLM
    var response = requestLLM(input)

    // Only loop while the LLM requests tools
    var toolCalls = response.parts.filterIsInstance<MessagePart.Tool.Call>()
    while (toolCalls.isNotEmpty()) {
        // Execute the tools and return the results
        val results = executeTools(toolCalls)
        // Send the tool results back to the LLM. The LLM may call more tools or return a final output
        response = sendToolResults(results)
        toolCalls = response.parts.filterIsInstance<MessagePart.Tool.Call>()
    }

    // When no tool calls remain, extract and return the assistant message content from the response
    response.parts.filterIsInstance<MessagePart.Text>().joinToString("\n") { it.text }
}

val mathAgentWithTools = AIAgent(
    promptExecutor = simpleOllamaAIExecutor(),
    llmModel = OllamaModels.Meta.LLAMA_3_2,
    toolRegistry = toolRegistry,
    strategy = strategy
)

fun main() = runBlocking {
    val result = mathAgentWithTools.run("Multiply 3 by 4, then multiply the result by 5.")
    println(result)
}
```

```
@LLMDescription(description = "Tools for performing math operations")
public static class MathTools implements ToolSet {
    @Tool
    @LLMDescription(description = "Multiplies two numbers and returns the result")
    public int multiply(int a, int b) {
        // This is not necessary, but it helps to see the tool call in the console output
        System.out.println("Multiplying " + a + " and " + b + "...");
        return a * b;
    }
}

public static void main(String[] args) {
    MathTools mathTools = new MathTools();
    ToolRegistry toolRegistry = ToolRegistry.builder()
        .tools(mathTools)
        .build();

    AIAgent<String, String> mathAgentWithTools = AIAgent.builder()
        .promptExecutor(SimpleLLMExecutorsKt.simpleOllamaAIExecutor("http://localhost:11434"))
        .llmModel(OllamaModels.Meta.LLAMA_3_2)
        .toolRegistry(toolRegistry)
        .functionalStrategy("mathWithTools", (AIAgentFunctionalContext context, String input) -> {
            // Send the user input to the LLM
            List<Message.Response> responses = context.requestLLMMultiple(input);

            // Only loop while the LLM requests tools
            while (context.containsToolCalls(responses)) {
                // Extract tool calls from the response
                List<MessagePart.Tool.Call> pendingCalls = context.extractToolCalls(responses);
                // Execute the tools and return the results
                List<ReceivedToolResult> results = context.executeMultipleTools(pendingCalls, false);
                // Send the tool results back to the LLM
                responses = context.sendMultipleToolResults(results);
            }

            // Extract and return the assistant message content from the response
            Message.Response finalResponse = responses.get(0);
            if (finalResponse instanceof Message.Assistant) {
                return ((Message.Assistant) finalResponse).getContent();
            }
            return "";
        })
        .build();

    String result = mathAgentWithTools.run("Multiply 3 by 4, then multiply the result by 5.");
    System.out.println(result);
}
```

代理可以生成以下输出：

```
Multiplying 3 and 4...
Multiplying 12 and 5...
The result of multiplying 3 by 4 is 12. Multiplying 12 by 5 gives us a final answer of 60.
```

## 后续步骤

- 了解如何创建[基于图的代理](../graph-based-agents/)
