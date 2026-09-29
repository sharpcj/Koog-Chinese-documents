# Prompts

Prompt 是给大型语言模型（LLM）的指令，用来引导它们生成响应。
它们定义了你与 LLM 交互的内容和结构。
本节介绍如何使用 Koog 创建并运行 prompt。

## 创建 prompt

在 Koog 中，prompt 是 [**Prompt**](https://api.koog.ai/prompt/prompt-model/ai.koog.prompt.dsl/-prompt/index.html)
data class 的实例，具有以下属性：

- `id`：prompt 的唯一标识符。
- `messages`：表示与 LLM 对话的消息列表。
- `params`：可选的 [LLM 配置参数](prompt-creation/#prompt-parameters)（例如 temperature、tool choice 等）。

虽然你可以直接实例化 `Prompt` 类，
但推荐的创建 prompt 方式是使用 [Kotlin DSL](prompt-creation/) 或 Java builder API，
它们提供了一种结构化方式来定义对话。

注意

本页中的 Kotlin 示例使用 Kotlin DSL。Java 示例使用 `Prompt.builder("id")` builder，并显式调用 `system(...)`、`user(...)`、`assistant(...)`、`toolCall(...)`、`toolResult(...)` 以及适用时的 `withOutput(Foo.class)` 等方法。

KotlinJava

```
val myPrompt = prompt("hello-koog") {
    system("You are a helpful assistant.")
    user("What is Koog?")
}
```

```
var myPrompt = Prompt.builder("hello-koog")
    .system("You are a helpful assistant.")
    .user("What is Koog?")
    .build();
```

注意

AI agents 可以接收简单文本 prompt 作为输入。
它们会自动将文本 prompt 转换为 Prompt 对象，并将其发送给 LLM 执行。
这对于[基本智能体](../agents/basic-agents/)很有用，
它只需要运行单个请求，不需要复杂的对话逻辑。

## 运行 prompt

Koog 为针对 LLM 运行 prompt 提供了两个抽象层级：LLM clients 和 prompt executors。
两者都接受 Prompt 对象，并且都可用于直接执行 prompt，而无需 AI agent。
clients 和 executors 的执行流程相同：

```
flowchart TB
    A([Prompt built with Kotlin DSL or Java builder])
    B{LLM client or prompt executor}
    C[LLM provider]
    D([Response to your application])

    A -->|"passed to"| B
    B -->|"sends request"| C
    C -->|"returns response"| B
    B -->|"returns result"| D
```

- [**LLM clients**](llm-clients/)

  ---

  用于与特定 LLM provider 直接交互的低层接口。
  当你使用单个 provider 且不需要高级生命周期管理时使用它们。
- [**Prompt executors**](prompt-executors/)

  ---

  管理一个或多个 LLM clients 生命周期的高级抽象。
  当你需要一个统一 API 来跨多个 providers 运行 prompt，
  并需要在它们之间动态切换和使用 fallback 时使用它们。

## 优化性能和处理失败

Koog 允许你在运行 prompt 时优化性能并处理失败。

- [**LLM 响应缓存**](llm-response-caching/)

  ---

  缓存 LLM 响应，以优化性能并降低重复请求的成本。
- [**处理失败**](handling-failures/)

  ---

  在你的应用中使用内置重试、超时和其他错误处理机制。

## AI agents 中的 prompt

在 Koog 中，AI agents 会在其生命周期中维护并管理 prompt。
虽然 LLM clients 或 executors 用于运行 prompt，但 agents 会处理 prompt 更新流程，确保
对话历史保持相关且一致。

agent 中的 prompt 生命周期通常包括几个阶段：

1. 初始 prompt 设置。
2. 自动 prompt 更新。
3. 上下文窗口管理。
4. 手动 prompt 管理。

### 初始 prompt 设置

当你[初始化 agent](../quickstart/#create-your-first-koog-agent) 时，
可以定义一条[系统消息](prompt-creation/#system-message)来设定 agent 的行为。
然后，当你调用 agent 的 `run()` 方法时，
通常会提供一条初始[用户消息](prompt-creation/#user-messages)作为输入。
这些消息共同构成 agent 的初始 prompt。例如：

KotlinJava

```
// Create an agent
val agent = AIAgent(
    promptExecutor = simpleOpenAIExecutor(apiKey),
    systemPrompt = "You are a helpful assistant.",
    llmModel = OpenAIModels.Chat.GPT4o
)

// Run the agent
val result = agent.run("What is Koog?")
```

```
AIAgent<String, String> agent = AIAgent.builder()
    .promptExecutor(simpleOpenAIExecutor(System.getenv("OPENAI_API_KEY")))
    .systemPrompt("You are a helpful assistant. Answer user questions concisely.")
    .llmModel(OpenAIModels.Chat.GPT4o)
    .build();

var result = agent.run("What is Koog?");
```

在该示例中，agent 会自动将文本 prompt 转换为 Prompt 对象，并将其发送给 prompt executor：

```
flowchart TB
    A([Your application])
    B{{Configured AI agent}}
    C["Text prompt"]
    D["Prompt object"]
    E{{Prompt executor}}
    F[LLM provider]

    A -->|"run() with text"| B
    B -->|"takes"| C
    C -->|"converted to"| D
    D -->|"sent via"| E
    E -->|"calls"| F
    F -->|"responds to"| E
    E -->|"result to"| B
    B -->|"result to"| A
```

对于更高级的配置，你也可以使用 [AIAgentConfig](https://api.koog.ai/agents/agents-core/ai.koog.agents.core.agent.config/-a-i-agent-config/index.html)
来定义 agent 的初始 prompt。

### 自动 prompt 更新

当 agent 运行其策略时，[预定义节点](../nodes-and-components/)会自动更新 prompt。
例如：

- [`nodeLLMRequest`](../nodes-and-components/#nodellmrequest)：向 prompt 追加用户消息并捕获 LLM 响应。
- [`nodeLLMSendToolResult`](../nodes-and-components/#nodellmsendtoolresult)：将工具执行结果追加到对话中。
- [`nodeAppendPrompt`](../nodes-and-components/#nodeappendprompt)：在工作流的任意位置向 prompt 插入特定消息。

### 上下文窗口管理

为避免在长时间运行的交互中超出 LLM 上下文窗口，agents 可以使用
[history compression](../history-compression/) 功能。

### 手动 prompt 管理

对于复杂工作流，你可以使用 [LLM sessions](../sessions/) 手动管理 prompt。
在 agent strategy 或自定义节点中，你可以使用 `llm.writeSession` 访问并更改 `Prompt` 对象。
这让你可以按需添加、移除或重新排序消息。