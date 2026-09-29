# 创建提示

Koog 提供了一种结构化方式来创建提示，并可控制消息类型、消息顺序和内容：

- 对于 **Kotlin** 用户，通过类型安全的 Kotlin DSL。
- 对于 **Java** 用户，通过流式 builder API。

## 基本结构

Kotlin 中的 `prompt()` 函数或 Java 中的 `Prompt.builder()` 会创建一个带有唯一 ID 和消息列表的 Prompt 对象：

KotlinJava

```
val prompt = prompt("unique_prompt_id") {
    // List of messages
}
```

```
Prompt prompt = Prompt.builder("unique_prompt_id")
    // List of messages
    .build();
```

## 消息类型

Kotlin DSL 和 Java builder API 支持以下消息类型，每种类型都对应对话中的特定角色：

- **System message**：向 LLM 提供上下文、指令和约束，定义其行为。
- **User message**：表示用户输入。
- **Assistant message**：表示 LLM 响应，可用于 few-shot 学习或继续对话。
- **Tool message**：表示工具调用及其结果。

KotlinJava

```
val prompt = prompt("unique_prompt_id") {
    // Add a system message to set the context
    system("You are a helpful assistant with access to tools.")
    // Add a user message
    user("What is 5 + 3 ?")
    // Add an assistant message
    assistant("The result is 8.")
}
```

```
Prompt prompt = Prompt.builder("unique_prompt_id")
    // Add a system message to set the context
    .system("You are a helpful assistant with access to tools.")
    // Add a user message
    .user("What is 5 + 3 ?")
    // Add an assistant message
    .assistant("The result is 8.")
    .build();
```

### System message

系统消息定义 LLM 的行为，并为整个对话设置上下文。
它可以指定模型的角色和语气，为响应提供准则和约束，并提供响应示例。

要创建系统消息，请将字符串作为参数传递给 Kotlin 的 `system()` 函数或 Java 方法：

KotlinJava

```
val prompt = prompt("system_message") {
    system("You are a helpful assistant that explains technical concepts.")
}
```

```
Prompt prompt = Prompt.builder("system_message")
    .system("You are a helpful assistant that explains technical concepts.")
    .build();
```

### User messages

用户消息表示来自用户的输入。
要创建用户消息，请将字符串作为参数传递给 Kotlin 的 `user()` 函数或 Java 方法：

KotlinJava

```
val prompt = prompt("user_message") {
    system("You are a helpful assistant.")
    user("What is Koog?")
}
```

```
Prompt prompt = Prompt.builder("user_message")
    .system("You are a helpful assistant.")
    .user("What is Koog?")
    .build();
```

大多数用户消息包含纯文本，但也可以包含多模态内容，例如图像、音频、视频和文档。
有关详情和示例，请参阅 [Multimodal content](multimodal-content/)。

### Assistant messages

助手消息表示 LLM 响应，可用于未来类似交互中的 few-shot 学习、继续对话，或展示预期的输出结构。

要创建助手消息，请将字符串作为参数传递给 Kotlin 的 `assistant()` 函数或 Java 方法：

KotlinJava

```
val prompt = prompt("article_review") {
    system("Evaluate the article.")

    // Example 1
    user("The article is clear and easy to understand.")
    assistant("positive")

    // Example 2
    user("The article is hard to read but it's clear and useful.")
    assistant("neutral")

    // Example 3
    user("The article is confusing and misleading.")
    assistant("negative")

    // New input to classify
    user("The article is interesting and helpful.")
}
```

```
Prompt prompt = Prompt.builder("article_review")
    .system("Evaluate the article.")

    // Example 1
    .user("The article is clear and easy to understand.")
    .assistant("positive")

    // Example 2
    .user("The article is hard to read but it's clear and useful.")
    .assistant("neutral")

    // Example 3
    .user("The article is confusing and misleading.")
    .assistant("negative")

    // New input to classify
    .user("The article is interesting and helpful.")
    .build();
```

### Tool messages

工具消息表示工具调用及其结果，可用于预填充工具调用历史。

Tip

LLM 会在执行过程中生成工具调用。
预填充工具调用有助于 few-shot 学习，或演示工具的预期使用方式。

要创建工具消息，请在 Kotlin 中调用 `tool()` 函数，或在 Java 中调用 `toolCall()` 和 `toolResult()` 方法：

KotlinJava

```
val prompt = prompt("calculator_example") {
    system("You are a helpful assistant with access to tools.")
    user("What is 5 + 3?")
    // Tool call
    toolCall(
        id = "calculator_tool_id",
        tool = "calculator",
        args = """{"operation": "add", "a": 5, "b": 3}"""
    )
    // Tool result
    toolResult(
        id = "calculator_tool_id",
        tool = "calculator",
        output = "8"
    )

    // LLM response based on tool result
    assistant("The result of 5 + 3 is 8.")
    user("What is 4 + 5?")
}
```

```
Prompt prompt = Prompt.builder("calculator_example")
    .system("You are a helpful assistant with access to tools.")
    .user("What is 5 + 3?")
    // Tool call
    .toolCall("calculator_tool_id", "calculator", "{\"operation\": \"add\", \"a\": 5, \"b\": 3}")
    // Tool result
    .toolResult("calculator_tool_id", "calculator", "8")
    // LLM response based on tool result    
    .assistant("The result of 5 + 3 is 8.")
    .user("What is 4 + 5?")
    .build();
```

## 文本消息构建器

Warning

文本消息构建器仅在 Kotlin 中可用。

构建 `system()`、`user()` 或 `assistant()` 消息时，可以使用辅助 [text-building-functions](https://api.koog.ai/prompt/prompt-model/ai.koog.prompt.text/-text-content-builder/index.html) 进行富文本格式化。

Kotlin

```
val prompt = prompt("text_example") {
    user {
        +"Review the following code snippet:"
        +"fun greet(name: String) = println(\"Hello, \$name!\")"

        // Paragraph break
        br()
        text("Please include in your explanation:")

        // Indent content
        padding("  ") {
            +"1. What the function does."
            +"2. How string interpolation works."
        }
    }
}
```

你还可以使用 [Markdown](https://api.koog.ai/prompt/prompt-markdown/ai.koog.prompt.markdown/markdown.html) 和 [XML](https://api.koog.ai/prompt/prompt-xml/ai.koog.prompt.xml/xml.html) 构建器，以相应格式添加内容。

Kotlin

```
val prompt = prompt("markdown_xml_example") {
    // A user message in Markdown format
    user {
        markdown {
            h2("Evaluate the article using the following criteria:")
            bulleted {
                item { +"Clarity and readability" }
                item { +"Accuracy of information" }
                item { +"Usefulness to the reader" }
            }
        }
    }
    // An assistant message in XML format
    assistant {
        xml {
            xmlDeclaration()
            tag("review") {
                tag("clarity") { text("positive") }
                tag("accuracy") { text("neutral") }
                tag("usefulness") { text("positive") }
            }
        }
    }
}
```

Tip

可以将文本构建函数与 XML 和 Markdown 构建器混合使用。

## 提示参数

可以通过配置控制 LLM 行为的参数来自定义提示。

KotlinJava

```
val prompt = prompt(
    id = "custom_params",
    params = LLMParams(
        temperature = 0.7,
        numberOfChoices = 1,
        toolChoice = LLMParams.ToolChoice.Auto
    )
) {
    system("You are a creative writing assistant.")
    user("Write a song about winter.")
}
```

```
// Create params first
LLMParams params = new LLMParams(
    0.7,                    // temperature
    null,                   // maxTokens
    1,                      // numberOfChoices
    null,                   // speculation
    null,                   // schema
    LLMParams.ToolChoice.Auto.INSTANCE, // toolChoice
    null,                   // user
    null                    // additionalProperties
);

Prompt prompt = Prompt.builder("custom_params")
    .system("You are a creative writing assistant.")
    .user("Write a song about winter.")
    .build();

// Apply params to the built prompt
prompt = prompt.withParams(params);
```

支持以下参数：

- `temperature`：控制模型响应中的随机性。
- `toolChoice`：控制模型的工具调用行为。
- `numberOfChoices`：请求多个备选响应。
- `schema`：定义模型响应格式的结构。
- `maxTokens`：限制响应中的 token 数量。
- `speculation`：提供关于预期响应格式的提示（仅受特定模型支持）。

更多信息请参阅 [LLM parameters](../../llm-parameters/)。

## 扩展现有提示

你可以通过在 Kotlin 中调用 `prompt()` 函数，或在 Java 中调用 `Prompt.builder()` 并传入现有提示作为参数来扩展现有提示：

KotlinJava

```
val basePrompt = prompt("base") {
    system("You are a helpful assistant.")
    user("Hello!")
    assistant("Hi! How can I help you?")
}

val extendedPrompt = prompt(basePrompt) {
    user("What's the weather like?")
}
```

```
Prompt basePrompt = Prompt.builder("base")
    .system("You are a helpful assistant.")
    .user("Hello!")
    .assistant("Hi! How can I help you?")
    .build();

Prompt extendedPrompt = Prompt.builder(String.valueOf(basePrompt))
    .user("What's the weather like?")
    .build();
```

这会创建一个新提示，其中包含 `basePrompt` 的所有消息和新的用户消息。

## 后续步骤

- 了解如何使用[多模态内容](multimodal-content/)。
- 如果你使用单个 LLM 提供方，请通过 [LLM clients](../llm-clients/) 运行提示。
- 如果你使用多个 LLM 提供方，请通过 [prompt executors](../prompt-executors/) 运行提示。
- 了解如何通过 [cache control](cache-control/) 使用 llm cache。