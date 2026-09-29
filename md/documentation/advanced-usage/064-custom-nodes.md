# 自定义节点实现

本页提供有关如何在 Koog 框架中实现自定义节点的详细说明。
自定义节点允许你通过创建执行特定操作的可复用组件来扩展智能体工作流的功能。

要了解有关图节点是什么、其用法以及现有默认节点的更多信息，请参阅[图节点](../nodes-and-components/)。

## 节点架构概述

在深入实现细节之前，了解 Koog 框架中节点的架构非常重要。节点是智能体工作流的基本构建块，其中每个节点表示工作流中的特定操作或转换。你使用边来连接节点，边定义了节点之间的执行流。

每个节点都有一个 `execute` 方法，该方法接收输入并产生输出，输出随后会传递给工作流中的下一个节点。

## 实现自定义节点

自定义节点实现的范围从对输入数据执行基本逻辑并返回输出的简单实现，到接受参数并在多次运行之间维护状态的更复杂节点实现。

### 基本节点实现

在图中实现自定义节点并定义自己的自定义逻辑的最简单方法是使用以下模式：

KotlinJava
```
val myNode by node<Input, Output>("node_name") { input ->
    // Processing
    returnValue
}
```

```
var myNode = AIAgentNode.builder("node_name")
    .withInput(Input.class)
    .withOutput(Output.class)
    .withAction((input, ctx) -> {
        // Processing
        return returnValue;
    })
    .build();
```
上面的代码表示一个自定义节点 `myNode`，它具有预定义的 `Input` 和 `Output` 类型，以及可选的名字字符串参数（`node_name`）。在 Kotlin 中，你使用 `node` DSL 函数。在 Java 中，你使用 `AIAgentNode.builder()` 模式。

在一个实际示例中，这里有一个简单的节点，它接收字符串输入并返回输入的长度：

KotlinJava
```
val myNode by node<String, Int>("node_name") { input ->
    // Processing
    input.length
}
```

```
var myNode = AIAgentNode.builder("node_name")
    .withInput(String.class)
    .withOutput(Integer.class)
    .withAction((input, ctx) -> {
        // Processing
        return input.length();
    })
    .build();
```
创建自定义节点的另一种方式是将其提取为可复用的函数。在 Kotlin 中，你可以在 `AIAgentSubgraphBuilderBase` 上定义一个扩展函数来调用 `node` 函数。在 Java 中，你可以将节点构建器调用提取到一个辅助方法中。

KotlinJava
```
fun AIAgentSubgraphBuilderBase<*, *>.myCustomNode(
    name: String? = null
): AIAgentNodeDelegate<Input, Output> = node(name) { input ->
    // Custom logic
    input // Return the input as output (pass-through)
}

val myCustomNode by myCustomNode("node_name")
```

```
var myCustomNode = AIAgentNode.builder("node_name")
    .withInput(String.class)
    .withOutput(String.class)
    .withAction((input, ctx) -> {
        // Custom logic
        return input; // Return the input as output (pass-through)
    })
    .build();
```
这会创建一个直通节点，执行一些自定义逻辑，但将输入原样返回作为输出，不做任何修改。

### 带附加参数的节点

你可以创建接受参数以自定义其行为的节点：

KotlinJava
```
    fun AIAgentSubgraphBuilderBase<*, *>.myNodeWithArguments(
    name: String? = null,
    arg1: String,
    arg2: Int
): AIAgentNodeDelegate<Input, Output> = node(name) { input ->
    // Use arg1 and arg2 in your custom logic
    input // Return the input as the output
}

val myCustomNode by myNodeWithArguments("node_name", arg1 = "value1", arg2 = 42)
```

```
String arg1 = "value1";
int arg2 = 42;

var myCustomNode = AIAgentNode.builder("node_name")
    .withInput(String.class)
    .withOutput(String.class)
    .withAction((input, ctx) -> {
        // Use arg1 and arg2 in your custom logic
        return input; // Return the input as the output
    })
    .build();
```
### 参数化节点

你可以使用泛型输入和输出类型参数来定义节点。在 Kotlin 中，你可以使用带有 `reified` 类型参数的 `inline` 函数。在 Java 中，你在构建节点时显式指定类型。

KotlinJava
```
inline fun <reified T> AIAgentSubgraphBuilderBase<*, *>.myParameterizedNode(
    name: String? = null,
): AIAgentNodeDelegate<T, T> = node(name) { input ->
    // Do some additional actions
    // Return the input as the output
    input
}

val strategy = strategy<String, String>("strategy_name") {
    val myCustomNode by myParameterizedNode<String>("node_name")
}
```

```
// In Java, specify the types explicitly when building the node
var myCustomNode = AIAgentNode.builder("node_name")
    .withInput(String.class)
    .withOutput(String.class)
    .withAction((input, ctx) -> {
        // Do some additional actions
        // Return the input as the output
        return input;
    })
    .build();
```
### 有状态节点

如果你的节点需要在多次运行之间维护状态，可以使用闭包变量。在 Kotlin 中，你在外层函数中声明一个变量。在 Java 中，由于 lambda 捕获必须是有效 final 的，你需要使用像 `AtomicInteger` 这样的线程安全包装器。

KotlinJava
```
fun AIAgentSubgraphBuilderBase<*, *>.myStatefulNode(
    name: String? = null
): AIAgentNodeDelegate<Input, Output> {
    var counter = 0

    return node(name) { input ->
        counter++
        println("Node executed $counter times")
        input
    }
}
```

```
// In Java, use AtomicInteger (or similar) since the lambda captures must be effectively final
AtomicInteger counter = new AtomicInteger(0);

var myStatefulNode = AIAgentNode.builder("node_name")
    .withInput(String.class)
    .withOutput(String.class)
    .withAction((input, ctx) -> {
        int count = counter.incrementAndGet();
        System.out.println("Node executed " + count + " times");
        return input;
    })
    .build();
```
## 节点输入和输出类型

节点可以具有不同的输入和输出类型。在 Kotlin 和 Java 中，这些都指定为泛型类型参数：

KotlinJava
```
val stringToIntNode by node<String, Int>("node_name") { input: String ->
    // Processing
    input.toInt() // Convert string to integer
}
```

```
var stringToIntNode = AIAgentNode.builder("node_name")
    .withInput(String.class)
    .withOutput(Integer.class)
    .withAction((input, ctx) -> {
        // Processing
        return Integer.parseInt(input); // Convert string to integer
    })
    .build();
```
Note

输入和输出类型决定了节点如何与工作流中的其他节点连接。只有当源节点的输出类型与目标节点的输入类型兼容时，节点才能连接。

## 最佳实践

在实现自定义节点时，请遵循以下最佳实践：

1. **保持节点专注**：每个节点应执行单一、定义明确的操作。
2. **使用描述性名称**：节点名称应清晰表明其用途。
3. **记录参数**：为所有参数提供清晰的文档。
4. **优雅地处理错误**：实现适当的错误处理，以防止工作流失败。
5. **使节点可复用**：将节点设计为可在不同工作流中复用。
6. **使用类型参数**：在适当的时候使用泛型类型参数，使节点更加灵活。
7. **提供默认值**：在可能的情况下，为参数提供合理的默认值。

## 常见模式

以下各节提供了一些实现自定义节点的常见模式。

### 直通节点

执行某项操作但将输入作为输出返回的节点。

KotlinJava
```
val loggingNode by node<String, String>("node_name") { input ->
    println("Processing input: $input")
    input // Return the input as the output
}
```

```
var loggingNode = AIAgentNode.builder("node_name")
    .withInput(String.class)
    .withOutput(String.class)
    .withAction((input, ctx) -> {
        System.out.println("Processing input: " + input);
        return input; // Return the input as the output
    })
    .build();
```
### 转换节点

对输入数据进行转换并生成修改后输出的节点。

KotlinJava
```
val upperCaseNode by node<String, String>("node_name") { input ->
    println("Processing input: $input")
    input.uppercase() // Transform the input to uppercase
}
```

```
var upperCaseNode = AIAgentNode.builder("node_name")
    .withInput(String.class)
    .withOutput(String.class)
    .withAction((input, ctx) -> {
        System.out.println("Processing input: " + input);
        return input.toUpperCase(); // Transform the input to uppercase
    })
    .build();
```
### LLM 交互节点

与 LLM 交互的节点。在 Kotlin 中，你可以对 LLM 会话进行细粒度控制。在 Java 中，你通常使用预构建的工厂方法，例如 `AIAgentNode.llmRequest()`，它会自动处理提示词构建。

KotlinJava
```
val summarizeTextNode by node<String, String>("node_name") { input ->
    llm.writeSession {
        appendPrompt {
            user("Please summarize the following text: $input")
        }

        val response = requestLLMWithoutTools()
        response.parts.filterIsInstance<MessagePart.Text>().joinToString("\n") { it.text }
    }
}
```

```
// In Java, LLM interaction is handled using pre-built factory nodes.
// AIAgentNode.llmRequest() creates a node that sends the input string as a user
// message to the LLM and returns the response. The prompt text is provided as
// the node's input when it is executed in the graph.
var summarizeTextNode = AIAgentNode.llmRequest("node_name");

// To extract the text content from the LLM response, chain a separate node:
var extractContent = AIAgentNode.builder("extract-content")
    .withInput(Message.Assistant.class)
    .withOutput(String.class)
    .withAction((response, ctx) -> response.getParts().stream()
        .filter(p -> p instanceof MessagePart.Text)
        .map(p -> ((MessagePart.Text) p).getText())
        .collect(Collectors.joining()))
    .build();
```
Note

上面的 Kotlin 示例展示了对 LLM 会话的细粒度控制（自定义提示词构造、显式调用 `requestLLMWithoutTools`）。Java API 提供了更高级的工厂方法，例如 `AIAgentNode.llmRequest()`，它会自动处理提示词构造，其中输入字符串会成为用户消息。对于高级提示词自定义，可以组合多个节点或使用自定义子图。

### 工具运行节点

执行工具的自定义节点。在 Kotlin 中，你可以手动构造工具调用并执行它们。在 Java 中，你通常使用子图，将工具编排委托给 LLM。

KotlinJava
```
val nodeExecuteCustomTool by node<String, String>("node_name") { input ->
    val toolCall = MessagePart.Tool.Call(
        id = UUID.randomUUID().toString(),
        tool = toolName,
        args = Json.encodeToString(ToolArgs(arg1 = input, arg2 = 42)) // Use the input as tool arguments
    )

    val result = environment.executeTool(toolCall)
    result.output
}
```

```
// In Java, direct tool execution (as shown in the Kotlin example) is not available
// through the Java builder API. Instead, use a subgraph that delegates tool calls
// to the LLM, which decides when and how to invoke the tools:
var toolSubgraph = AIAgentSubgraph.builder("tool-subgraph")
    .withInput(String.class)
    .withOutput(String.class)
    .withTask(input -> "Use my_tool with input: " + input)
    .build();
```
Note

Kotlin 示例通过手动构造 `MessagePart.Tool.Call` 并调用 `environment.executeTool()` 来演示底层工具执行。Java API 鼓励使用更高级的方法，即通过 `withTask()` 配合子图，由 LLM 自动编排工具调用。要限制可用的工具，请在 `.withInput()` 之前链式调用 `.limitedTools(List.of(myTool))`。
