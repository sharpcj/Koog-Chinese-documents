# Basic agents

Basic agent 使用预定义 strategy 和简单执行流程，适用于大多数常见使用场景。
它接受字符串输入（问题、请求或任务描述），并将该输入发送给配置的 LLM。
LLM 可能决定调用提供的 tools。
Agent 会执行这些 tools，并将结果发送回 LLM。
这一过程会重复，直到 LLM 不再请求任何 tool calls 并返回字符串响应。
随后 agent 输出该响应。

在 [Graph-based agents](../graph-based-agents/) 中，你可以了解如何重新创建 basic agents 使用的预定义 strategy graph。

先决条件

确保你的环境和项目满足以下要求：

- JDK 17+
- Kotlin 2.2.0+
- Gradle 8.0+ 或 Maven 3.8+

将 [Koog package](https://central.sonatype.com/artifact/ai.koog/koog-agents/) 添加为依赖项：

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

从 LLM provider 获取 API key，或通过 Ollama 运行本地 LLM。
有关更多信息，请参阅[快速开始](../../quickstart/)。

本页示例假设你已设置 `OPENAI_API_KEY` 环境变量。

## 创建最小 agent

要创建最基础的 agent，请实例化 [`AIAgent`](https://api.koog.ai/agents/agents-core/ai.koog.agents.core.agent/-a-i-agent/index.html)，并提供带有[语言模型](../../model-capabilities/#creating-a-model-llmodel-configuration)的 [prompt executor](../../prompts/prompt-executors/)：

KotlinJava

```
val agent = AIAgent(
    promptExecutor = simpleOpenAIExecutor(System.getenv("OPENAI_API_KEY")),
    llmModel = OpenAIModels.Chat.GPT4o
)
```

该 agent 会期望字符串作为输入，并返回字符串作为输出。
要运行该 agent，请使用 `run()` 函数并传入一些用户输入：

```
fun main() = runBlocking {
    val result = agent.run("Hello! How can you help me?")
    println(result)
}
```

```
AIAgent<String, String> agent = AIAgent.builder()
    .promptExecutor(simpleOpenAIExecutor(System.getenv("OPENAI_API_KEY")))
    .llmModel(OpenAIModels.Chat.GPT4o)
    .build();
```

该 agent 期望字符串作为输入，并返回字符串作为输出。
要运行该 agent，请使用 `run()` 方法并传入一些用户输入：

```
String result = agent.run("Hello! How can you help me?");
System.out.println(result);
```

Agent 会返回一个通用回答，例如：

```
I can assist with a wide range of topics and tasks. Here are some examples:

1. **Answering questions**: I can provide information on various subjects, from science and history to entertainment and culture.
2. **Generating text**: I can help with writing tasks, such as suggesting alternative phrases, providing definitions, or even creating entire articles or stories.
3. **Translation**: I can translate text from one language to another, including popular languages such as Spanish, French, German, Chinese, and many more.
4. **Conversation**: I can engage in natural-sounding conversations, using context and understanding to respond to questions and statements.
5. **Brainstorming**: I can help generate ideas for creative projects, such as writing stories, composing music, or coming up with business ideas.
6. **Learning**: I can help with language learning, explaining grammar rules, vocabulary, and pronunciation.
7. **Calculations**: I can perform mathematical calculations, including basic arithmetic, algebra, and more advanced math concepts.

What's on your mind? Do you have a specific question, topic, or task you'd like to tackle?
```

## 添加 system prompt

提供 [system message](../../prompts/prompt-creation/#system-message)，用于定义 agent 的角色以及与任务相关的目的、上下文和指令。

KotlinJava

```
val agent = AIAgent(
    promptExecutor = simpleOpenAIExecutor(System.getenv("YOUR_API_KEY")),
    systemPrompt = "You are an expert in internet memes. Be helpful, friendly, and answer user questions concisely, showing your knowledge of memes.",
    llmModel = OpenAIModels.Chat.GPT4o
)
```

```
AIAgent<String, String> agent = AIAgent.builder()
    .promptExecutor(simpleOpenAIExecutor(System.getenv("OPENAI_API_KEY")))
    .systemPrompt("You are an expert in internet memes. Be helpful, friendly, and answer user questions concisely, showing your knowledge of memes.")
    .llmModel(OpenAIModels.Chat.GPT4o)
    .build();
```

System prompt 中的指令会引导 agent 的响应：

```
I'm here to help you navigate the wild world of internet memes!

What's on your mind? Are you trying to understand a specific meme, need help finding a popular joke, or perhaps want some recommendations for trending memes? Let me know, and I'll do my best to provide you with some LOLs!
```

## 配置 LLM 输出

你可以直接向 agent 构造函数（Kotlin）或通过 builder 方法（Java）提供一些 [LLM 参数](../../llm-parameters/#llm-parameter-reference)，以自定义 LLM 的行为。
例如，使用 `temperature` 参数调整生成响应的随机性：

KotlinJava

```
val agent = AIAgent(
    promptExecutor = simpleOpenAIExecutor(System.getenv("YOUR_API_KEY")),
    systemPrompt = "You are an expert in internet memes. Be helpful, friendly, and answer user questions concisely, showing your knowledge of memes.",
    llmModel = OpenAIModels.Chat.GPT4o,
    temperature = 0.7
)
```

```
AIAgent<String, String> agent = AIAgent.builder()
    .promptExecutor(simpleOpenAIExecutor(System.getenv("OPENAI_API_KEY")))
    .systemPrompt("You are an expert in internet memes. Be helpful, friendly, and answer user questions concisely, showing your knowledge of memes.")
    .llmModel(OpenAIModels.Chat.GPT4o)
    .temperature(0.7)
    .build();
```

以下是不同 temperature 值下的响应示例：

0.40.71.0

```
I'm here to help you navigate the wild world of internet memes! Whether you're looking for explanations, examples, or just want to share a meme with someone, I'm your go-to expert. What's on your mind? Got a specific meme in mind that's got you curious? Or maybe you need some meme-related advice? Fire away!
```

```
I'm here to help you navigate the wild world of internet memes!

What's on your mind? Need help understanding a specific meme, finding a popular joke or trend, or maybe even creating your own meme? Let's get this meme party started!
```

```
I'd be happy to help you navigate the wild world of internet memes!

Whether you're looking for explanations of classic memes, suggestions for new ones to try out, or just want to discuss your favorite meme culture trends, I'm here to assist. What's on your mind?

Do you have a specific question about memes (e.g., "What does this meme mean?"), or are you looking for some meme-related recommendations (e.g., "Can you recommend a funny meme to share with friends?"). Let me know how I can help!
```

## 添加 tools

Agents 可以使用 [tools](../../tools/) 执行特定任务。

首先，通过使用 [`@Tool`](https://api.koog.ai/agents/agents-tools/ai.koog.agents.core.tools.annotations/-tool/index.html) 注解标注函数（Kotlin）或方法（Java）来创建 tool：

KotlinJava

```
@Tool
@LLMDescription("Ask the user a question by sending it to stdout and return the answer from stdin")
fun askUser(
    @LLMDescription("Question from the agent")
    question: String
): String {
    println(question)
    return readln()
}
```

然后，使用 [`ToolRegistry`](https://api.koog.ai/agents/agents-tools/ai.koog.agents.core.tools/-tool-registry/index.html) 使该 tool 可供 agent 使用：

```
val agent = AIAgent(
    promptExecutor = simpleOpenAIExecutor(System.getenv("YOUR_API_KEY")),
    systemPrompt = "You are an expert in internet memes. Be helpful, friendly, and answer user questions concisely, showing your knowledge of memes.",
    llmModel = OpenAIModels.Chat.GPT4o,
    temperature = 0.7,
    toolRegistry = ToolRegistry {
        tool(::askUser)
    }
)
```

在该示例中，`askUser` 是一个 tool，它通过在控制台打印和读取来帮助 agent 与用户保持对话。
如果 agent 决定向用户提问，它可以调用该 tool，该 tool 通过 `println()` 写入 `stdout`，并通过 `readln()` 从 `stdin` 读取。

```
// Create a ToolSet class
class UserConversationTools implements ToolSet {
    @Tool
    @LLMDescription("Ask the user a question by sending it to stdout and return the answer from stdin")
    public String askUser(
        @LLMDescription("Question from the agent")
        String question
    ) {
        System.out.println(question);
        Scanner scanner = new Scanner(System.in);
        return scanner.nextLine();
    }
}
```

然后，使用 [`ToolRegistry`](https://api.koog.ai/agents/agents-tools/ai.koog.agents.core.tools/-tool-registry/index.html) 使该 tool 可供 agent 使用：

```
UserConversationTools askUser = new UserConversationTools();

ToolRegistry toolRegistry = ToolRegistry.builder()
        .tools(askUser)
        .build();

AIAgent<String, String> agent = AIAgent.builder()
    .promptExecutor(simpleOpenAIExecutor(System.getenv("OPENAI_API_KEY")))
    .systemPrompt("You are an expert in internet memes. Be helpful, friendly, and answer user questions concisely, showing your knowledge of memes.")
    .llmModel(OpenAIModels.Chat.GPT4o)
    .temperature(0.7)
    .toolRegistry(toolRegistry)
    .build();
```

在该示例中，`askUser` 是一个 tool，它通过在控制台打印和读取来帮助 agent 与用户保持对话。

下面是与 agent 交互的示例：

```
Agent: Which meme would you like me to explain? Please choose from: Grumpy Cat, Success Kid, or Doge.

User: Explain Doge

Agent:

**Doge**

Doge is a popular internet meme that originated on the imageboard website 4chan in 2013. The meme features a picture of a Shiba Inu dog with a caption written in a playful, exaggerated style.

The typical format of a Doge meme includes:

* A photo of a Shiba Inu dog
* A caption written in all capital letters, using a intentionally simplistic and childlike tone
* Exaggerated or made-up words or phrases, often used to convey a humorous or nonsensical idea

Examples of Doge memes might include:

* "Such wow. Such happy."
* "I had fun today!"
* "Wow, I am good at napping."

The meme is known for its lighthearted and playful tone, and is often used to express excitement, happiness, or silliness. The meme has since become a cultural phenomenon, with countless variations and parodies emerging online.
```

## 调整 agent 迭代次数

为避免无限循环，Koog 允许任何 agent 执行有限数量的步骤（默认 50）。
如果你预计 agent 需要更多步骤（例如 tool calls 和 LLM requests），可以使用 `maxIterations` 参数增加此限制；对于只需要少量步骤的 agents，也可以降低此限制。
例如，这里描述的 simple agent 通常不太可能需要超过 10 步：

KotlinJava

```
val agent = AIAgent(
    promptExecutor = simpleOpenAIExecutor(System.getenv("YOUR_API_KEY")),
    systemPrompt = "You are an expert in internet memes. Be helpful, friendly, and answer user questions concisely, showing your knowledge of memes.",
    llmModel = OpenAIModels.Chat.GPT4o,
    temperature = 0.7,
    toolRegistry = ToolRegistry {
        tool(::askUser)
    },
    maxIterations = 10
)
```

```
// Create a ToolSet class
class UserConversationTools implements ToolSet {
    @Tool
    @LLMDescription("Ask the user a question by sending it to stdout and return the answer from stdin")
    public String askUser(
        @LLMDescription("Question from the agent")
        String question
    ) {
        System.out.println(question);
        Scanner scanner = new Scanner(System.in);
        return scanner.nextLine();
    }
}

// In main method:
UserConversationTools askUser = new UserConversationTools();

ToolRegistry toolRegistry = ToolRegistry.builder()
        .tools(askUser)
        .build();

AIAgent<String, String> agent = AIAgent.builder()
    .promptExecutor(simpleOpenAIExecutor(System.getenv("OPENAI_API_KEY")))
    .systemPrompt("You are an expert in internet memes. Be helpful, friendly, and answer user questions concisely, showing your knowledge of memes.")
    .llmModel(OpenAIModels.Chat.GPT4o)
    .temperature(0.7)
    .toolRegistry(toolRegistry)
    .maxIterations(10)
    .build();
```

提示

除了将 model、temperature、max iterations 和其他参数直接传递给 Kotlin 构造函数或 Java builder，你也可以将它们定义为单独的配置对象并传入。
有关更多信息，请参阅 [Agent configuration](../#agent-configuration)。

## 处理 agent 运行时事件

为辅助测试和调试，并为链式 agent 交互创建 hooks，Koog 提供了 [EventHandler](https://api.koog.ai/agents/agents-features/agents-features-event-handler/ai.koog.agents.features.eventHandler.feature/-event-handler/index.html) feature。

KotlinJava

在 agent 构造函数 lambda 中调用 `handleEvents()` 函数以安装该 feature 并注册 event handlers：

```
val agent = AIAgent(
    promptExecutor = simpleOpenAIExecutor(System.getenv("YOUR_API_KEY")),
    systemPrompt = "You are an expert in internet memes. Be helpful, friendly, and answer user questions concisely, showing your knowledge of memes.",
    llmModel = OpenAIModels.Chat.GPT4o,
    temperature = 0.7,
    toolRegistry = ToolRegistry {
        tool(::askUser)
    },
    maxIterations = 10
){
    handleEvents {
        // Handle tool calls
        onToolCallStarting { eventContext ->
            println("Tool called: ${eventContext.toolName} with args ${eventContext.toolArgs}")
        }
    }
}
```

在 agent builder 上使用 `.install()` 方法，通过 `EventHandler.Feature` 注册 event handlers：

```
// Create a ToolSet class
class UserConversationTools implements ToolSet {
    @Tool
    @LLMDescription("Ask the user a question by sending it to stdout and return the answer from stdin")
    public String askUser(
        @LLMDescription("Question from the agent")
        String question
    ) {
        System.out.println(question);
        Scanner scanner = new Scanner(System.in);
        return scanner.nextLine();
    }
}

// In main method:
UserConversationTools askUser = new UserConversationTools();

ToolRegistry toolRegistry = ToolRegistry.builder()
        .tools(askUser)
        .build();

AIAgent<String, String> agent = AIAgent.builder()
    .promptExecutor(simpleOpenAIExecutor(System.getenv("OPENAI_API_KEY")))
    .systemPrompt("You are an expert in internet memes. Be helpful, friendly, and answer user questions concisely, showing your knowledge of memes.")
    .llmModel(OpenAIModels.Chat.GPT4o)
    .temperature(0.7)
    .toolRegistry(toolRegistry)
    .maxIterations(10)
    .install(EventHandler.Feature, config -> {
        config.onToolCallStarting(eventContext -> {
            System.out.println("Tool called: " + eventContext.getToolName() +
                " with args " + eventContext.getToolArgs());
        });
    })
    .build();
```

当 agent 调用 `askUser` tool 时，现在会输出类似以下内容：

```
Tool called: askUser with args {"question":"Which meme would you like me to explain?"}
```

有关 Koog agent features 的更多信息，请参阅 [Features](../../features/)。

## 后续步骤

- 了解有关构建 [graph-based agents](../graph-based-agents/) 和 [functional agents](../functional-agents/) 的更多信息