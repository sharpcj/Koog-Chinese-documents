# LLM clients

LLM clients 专为与 LLM providers 直接交互而设计。
每个 client 都实现 [`LLMClient`](https://api.koog.ai/prompt/prompt-executor/prompt-executor-clients/ai.koog.prompt.executor.clients/-l-l-m-client/index.html) 接口，该接口提供执行 prompts 和流式响应的方法。

当你使用单个 LLM provider 且不需要高级生命周期管理时，可以使用 LLM client。
如果需要管理多个 LLM providers，请使用 [prompt executor](../prompt-executors/)。

下表展示了可用的 LLM clients 及其能力。

| LLM provider | LLMClient | 工具调用 | 流式传输 | 多选项 | Embeddings | Moderation | 模型列表 | 说明 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| [OpenAI](https://platform.openai.com/docs/overview) | [OpenAILLMClient](https://api.koog.ai/prompt/prompt-executor/prompt-executor-clients/prompt-executor-openai-client/ai.koog.prompt.executor.clients.openai/-open-a-i-l-l-m-client/index.html) | ✓ | ✓ | ✓ | ✓ | ✓[1](#fn:1) | ✓ |  |
| [Anthropic](https://www.anthropic.com/) | [AnthropicLLMClient](https://api.koog.ai/prompt/prompt-executor/prompt-executor-clients/prompt-executor-anthropic-client/ai.koog.prompt.executor.clients.anthropic/-anthropic-l-l-m-client/index.html) | ✓ | ✓ | - | - | - | - | - |
| [Google](https://ai.google.dev/) β | [GoogleLLMClient](https://api.koog.ai/prompt/prompt-executor/prompt-executor-clients/prompt-executor-google-client/ai.koog.prompt.executor.clients.google/-google-l-l-m-client/index.html) | ✓ | ✓ | ✓ | ✓ | - | ✓ | - |
| [DeepSeek](https://www.deepseek.com/) β | [DeepSeekLLMClient](https://api.koog.ai/prompt/prompt-executor/prompt-executor-clients/prompt-executor-deepseek-client/ai.koog.prompt.executor.clients.deepseek/-deep-seek-l-l-m-client/index.html) | ✓ | ✓ | ✓ | - | - | ✓ | OpenAI-compatible chat client。 |
| [OpenRouter](https://openrouter.ai/) | [OpenRouterLLMClient](https://api.koog.ai/prompt/prompt-executor/prompt-executor-clients/prompt-executor-openrouter-client/ai.koog.prompt.executor.clients.openrouter/-open-router-l-l-m-client/index.html) | ✓ | ✓ | ✓ | - | - | ✓ | OpenAI-compatible router client。 |
| [Amazon Bedrock](https://aws.amazon.com/bedrock/) | [BedrockLLMClient](https://api.koog.ai/prompt/prompt-executor/prompt-executor-clients/prompt-executor-bedrock-client/ai.koog.prompt.executor.clients.bedrock/-bedrock-l-l-m-client/index.html) | ✓ | ✓ | - | ✓ | ✓[2](#fn:2) | - | 仅 JVM 的 AWS SDK client，支持多个模型系列。 |
| [Mistral](https://mistral.ai/) β | [MistralAILLMClient](https://api.koog.ai/prompt/prompt-executor/prompt-executor-clients/prompt-executor-mistralai-client/ai.koog.prompt.executor.clients.mistralai/-mistral-a-i-l-l-m-client/index.html) | ✓ | ✓ | ✓ | ✓ | ✓[3](#fn:3) | ✓ | OpenAI-compatible client。 |
| [Alibaba](https://www.alibabacloud.com/en?_p_lc=1) β | [DashScopeLLMClient](https://api.koog.ai/prompt/prompt-executor/prompt-executor-clients/prompt-executor-dashscope-client/ai.koog.prompt.executor.clients.dashscope/-dashscope-l-l-m-client/index.html) | ✓ | ✓ | ✓ | - | - | ✓ | OpenAI-compatible client，暴露 provider-specific 参数（`enableSearch`、`parallelToolCalls`、`enableThinking`）。 |
| [Ollama](https://ollama.com/) | [OllamaClient](https://api.koog.ai/prompt/prompt-executor/prompt-executor-clients/prompt-executor-ollama-client/ai.koog.prompt.executor.ollama.client/-ollama-client/index.html) | ✓ | ✓ | - | ✓ | ✓ | - | 带有模型管理 API 的本地服务器 client。 |

## 运行 prompt

要使用 LLM client 运行 prompt，请执行以下操作：

1. 创建一个 LLM client，用于处理你的应用与 LLM providers 之间的连接。
2. 使用 prompt 和 LLM 作为参数调用 `execute()` 方法。

下面是一个使用 `OpenAILLMClient` 运行 prompts 的示例：

KotlinJava

```
fun main() = runBlocking {
    // Create an OpenAI client
    val apiKey = System.getenv("OPENAI_API_KEY")
    val client = OpenAILLMClient(apiKey)

    // Create a prompt
    val prompt = prompt("prompt_name", LLMParams()) {
        // Add a system message to set the context
        system("You are a helpful assistant.")

        // Add a user message
        user("Tell me about Kotlin")

        // You can also add assistant messages for few-shot examples
        assistant("Kotlin is a modern programming language...")

        // Add another user message
        user("What are its key features?")
    }

    // Run the prompt
    val response = client.execute(prompt, OpenAIModels.Chat.GPT4o)
    // Print the response
    println(response)
}
```

```
// Create an OpenAI client
String apiKey = System.getenv("OPENAI_API_KEY");
OpenAILLMClient client = openAIClient(apiKey);

// Create a prompt
Prompt prompt = Prompt.builder("prompt_name")
    // Add a system message to set the context
    .system("You are a helpful assistant.")

    // Add a user message
    .user("Tell me about Kotlin")

    // You can also add assistant messages for few-shot examples
    .assistant("Kotlin is a modern programming language...")

    // Add another user message
    .user("What are its key features?")
    .build();

// Run the prompt
List<Message.Response> response = client.execute(prompt, OpenAIModels.Chat.GPT4o, Collections.emptyList());
// Print the response
System.out.println(response);

client.close();
```

## 流式响应

注意

适用于所有 LLM clients。

当你需要在响应生成时处理响应，可以在 Kotlin 中使用 `executeStreaming()` 方法，或在 Java 中使用
`executeStreamingWithPublisher()` 来流式传输模型输出。

流式 API 提供不同的 frame 类型：

- **Delta frames** (`TextDelta`, `ReasoningDelta`, `ToolCallDelta`) —— 以分块形式到达的增量内容
- **Complete frames** (`TextComplete`, `ReasoningComplete`, `ToolCallComplete`) —— 收到所有 delta 后的完整内容
- **End frame** (`End`) —— 通过 finish reason 标记流完成

对于支持 reasoning 的模型（例如 Claude Sonnet 4.5 或 GPT-o1），流式传输期间会发出 reasoning frames。
有关处理 frames 的更多详细信息，请参见 [Streaming API 文档](../../streaming-api/)。

KotlinJava

```
// Set up the OpenAI client with your API key
val token = System.getenv("OPENAI_API_KEY")
val client = OpenAILLMClient(token)

val response = client.executeStreaming(
    prompt = prompt("stream_demo") { user("Stream this response in short chunks.") },
    model = OpenAIModels.Chat.GPT4_1
)

response.collect { frame ->
    when (frame) {
        is StreamFrame.TextDelta -> print(frame.text)
        is StreamFrame.ReasoningDelta -> print("[Reasoning] ${frame.text}")
        is StreamFrame.ToolCallComplete -> println("\nTool call: ${frame.name}")
        is StreamFrame.End -> println("\n[done] Reason: ${frame.finishReason}")
        else -> {} // Handle other frame types if needed
    }
}
```

```
// Set up the OpenAI client with your API key
String token = System.getenv("OPENAI_API_KEY");
OpenAILLMClient client = openAIClient(token);

Prompt prompt = Prompt.builder("stream_demo")
            .user("Stream this response in short chunks.")
            .build();

Publisher<StreamFrame> response = client.executeStreamingWithPublisher(prompt, OpenAIModels.Chat.GPT4_1);

// Subscribe to the Publisher to consume frames
response.subscribe(new Subscriber<StreamFrame>() {
    private Subscription subscription;

    @Override
    public void onSubscribe(Subscription s) {
        this.subscription = s;
        s.request(Long.MAX_VALUE);
    }

    @Override
    public void onNext(StreamFrame frame) {
        switch (frame) {
            case StreamFrame.TextDelta delta ->
                    System.out.print(delta.getText());
            case StreamFrame.ReasoningDelta reasoning ->
                    System.out.print("[Reasoning] " + reasoning.getText());
            case StreamFrame.ToolCallComplete toolCall ->
                    System.out.println("\nTool call: " + toolCall.getName());
            case StreamFrame.End end ->
                    System.out.println("\n[done] Reason: " + end.getFinishReason());
            default -> {} // Handle other frame types
        }
    }

    @Override
    public void onError(Throwable t) {
        t.printStackTrace();
    }

    @Override
    public void onComplete() { }
});
```

## 多选项

注意

适用于除 `GoogleLLMClient`、`BedrockLLMClient` 和 `OllamaClient` 之外的所有 LLM clients

你可以使用 `executeMultipleChoices()` 方法在单次调用中请求模型生成多个备选响应。
它还要求在正在执行的 prompt 中指定 [`numberOfChoices`](../prompt-creation/#prompt-parameters) LLM 参数。

KotlinJava

```
fun main() = runBlocking {
    val apiKey = System.getenv("OPENAI_API_KEY")
    val client = OpenAILLMClient(apiKey)

    val choices = client.executeMultipleChoices(
        prompt = prompt("n_best", params = LLMParams(numberOfChoices = 3)) {
            system("You are a creative assistant.")
            user("Give me three different opening lines for a story.")
        },
        model = OpenAIModels.Chat.GPT4o
    )

    choices.forEachIndexed { i, choice ->
        val text = choice.parts.filterIsInstance<MessagePart.Text>().joinToString(" ") { it.text }
        println("Line #${i + 1}: $text")
    }
}
```

```
String apiKey = System.getenv("OPENAI_API_KEY");
OpenAILLMClient client = openAIClient(apiKey);

// Configure parameters (LLMParams constructor requires all 8 arguments in Java)
LLMParams params = new LLMParams(
    null, // temperature
    null, // maxTokens
    3,    // numberOfChoices
    null, // speculation
    null, // schema
    null, // toolChoice
    null, // user
    null  // additionalProperties
);

Prompt prompt = Prompt.builder("n_best")
    .system("You are a creative assistant.")
    .user("Give me three different opening lines for a story.")
    .build()
    .withParams(params);

// LLMChoice is a type alias for List<Message.Response>
List<List<Message.Response>> choices = client.executeMultipleChoices(
    prompt, 
    OpenAIModels.Chat.GPT4o
);

for (int i = 0; i < choices.size(); i++) {
    List<Message.Response> choice = choices.get(i);
    StringBuilder text = new StringBuilder();
    for (Message.Response msg : choice) {
        text.append(msg.getContent()).append(" ");
    }
    System.out.println("Line #" + (i + 1) + ": " + text.toString().trim());
}
```

## 列出可用模型

注意

适用于除 `AnthropicLLMClient`、`BedrockLLMClient` 和 `OllamaClient` 之外的所有 LLM clients。

要获取 LLM client 支持的可用模型 ID 列表，请使用 `models()` 方法：

KotlinJava

```
fun main() = runBlocking {
    val apiKey = System.getenv("OPENAI_API_KEY")
    val client = OpenAILLMClient(apiKey)

    val models: List<LLModel> = client.models()
    models.forEach { println(it.id) }
}
```

```
String apiKey = System.getenv("OPENAI_API_KEY");
OpenAILLMClient client = openAIClient(apiKey);

List<LLModel> models = client.models();
for (LLModel model : models) {
    System.out.println(model.getId());
}
```

## Embeddings

注意

适用于 `OpenAILLMClient`、`GoogleLLMClient`、`BedrockLLMClient`、`MistralAILLMClient` 和 `OllamaClient`。

你可以使用 `embed()` 方法将文本转换为 embedding 向量。
选择一个 embedding 模型，并将你的文本传给此方法：

```
fun main() = runBlocking {
    val apiKey = System.getenv("OPENAI_API_KEY")
    val client = OpenAILLMClient(apiKey)

    val embedding = client.embed(
        text = "This is a sample text for embedding",
        model = OpenAIModels.Embeddings.TextEmbedding3Large
    )

    println("Embedding size: ${embedding.size}")
}
```

## Moderation

注意

适用于以下 LLM clients：`OpenAILLMClient`、`BedrockLLMClient`、`MistralAILLMClient`、`OllamaClient`。

你可以将 `moderate()` 方法与 moderation 模型一起使用，检查 prompt 是否包含不适当内容：

KotlinJava

```
fun main() = runBlocking {
    val apiKey = System.getenv("OPENAI_API_KEY")
    val client = OpenAILLMClient(apiKey)

    val result = client.moderate(
        prompt = prompt("moderation") {
            user("This is a test message that may contain offensive content.")
        },
        model = OpenAIModels.Moderation.Omni
    )

    println(result)
}
```

```
String apiKey = System.getenv("OPENAI_API_KEY");
OpenAILLMClient client = openAIClient(apiKey);

Prompt prompt = Prompt.builder("moderation")
    .user("This is a test message that may contain offensive content.")
    .build();

ModerationResult result = client.moderate(prompt, OpenAIModels.Moderation.Omni);
System.out.println(result);
```

## 与 prompt executors 集成

[Prompt executors](../prompt-executors/) 会包装 LLM clients，并提供额外功能，例如路由、fallback，以及跨 providers 的统一使用。
它们推荐用于生产环境，因为在使用多个 providers 时可提供灵活性。

---

1. 支持通过 OpenAI Moderation API 进行 moderation。 [↩](#fnref:1 "Jump back to footnote 1 in the text")
2. Moderation 需要 Guardrails 配置。 [↩](#fnref:2 "Jump back to footnote 2 in the text")
3. 支持通过 Mistral `v1/moderations` endpoint 进行 moderation。 [↩](#fnref:3 "Jump back to footnote 3 in the text")