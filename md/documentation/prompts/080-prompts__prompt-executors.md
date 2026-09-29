# Prompt executors

Prompt executors 提供了一层更高级的抽象，让你可以管理一个或多个 LLM clients 的生命周期。
你可以通过统一接口使用多个 LLM providers，屏蔽 provider 特定细节，并在它们之间动态切换和回退。

## Executor 类型

Koog 提供三种实现 [`PromptExecutor`](https://api.koog.ai/prompt/prompt-executor/prompt-executor-model/ai.koog.prompt.executor.model/-prompt-executor/index.html) 接口的主要 prompt executors：

| 类型 | 类 | 描述 |
| --- | --- | --- |
| Single-provider | [`SingleLLMPromptExecutor`](https://api.koog.ai/prompt/prompt-executor/prompt-executor-model/ai.koog.prompt.executor.llms/-single-l-l-m-prompt-executor/index.html) | 封装单个 provider 的单个 LLM client。如果你的 agent 只需要在单个 LLM provider 内的模型之间切换，请使用此 executor。 |
| Multi-provider | [`MultiLLMPromptExecutor`](https://api.koog.ai/prompt/prompt-executor/prompt-executor-model/ai.koog.prompt.executor.llms/-multi-l-l-m-prompt-executor/index.html) | 封装多个 LLM clients，并根据 LLM provider 路由调用。当请求的 client 不可用时，它可以选择使用已配置的 fallback provider 和 LLM。如果你的 agent 需要在不同 providers 的 LLM 之间切换，请使用此 executor。 |
| Routing | [`RoutingLLMPromptExecutor`](https://api.koog.ai/prompt/prompt-executor/prompt-executor-model/ai.koog.prompt.executor.llms/-routing-l-l-m-prompt-executor/index.html) | 使用路由策略将对给定 LLM model 的请求分发到多个 client instances。使用此 executor 可避免速率限制、提高吞吐量，并通过负载均衡实现故障转移策略。 |

## 创建 single-provider executor

要为特定 LLM provider 创建 prompt executor，请执行以下操作：

1. 使用相应的 API key 为特定 provider 配置 LLM client。
2. 使用 [`MultiLLMPromptExecutor`](https://api.koog.ai/prompt/prompt-executor/prompt-executor-model/ai.koog.prompt.executor.llms/-multi-l-l-m-prompt-executor/index.html) 创建 prompt executor。

示例如下：

KotlinJava

```
val openAIClient = OpenAILLMClient(System.getenv("OPENAI_API_KEY"))
val promptExecutor = MultiLLMPromptExecutor(openAIClient)
```

```
OpenAILLMClient openAIClient = openAIClient(System.getenv("OPENAI_API_KEY"));
MultiLLMPromptExecutor promptExecutor = new MultiLLMPromptExecutor(openAIClient);
```

## 创建 multi-provider executor

要创建可与多个 LLM providers 配合使用的 prompt executor，请执行以下操作：

1. 使用相应 API keys 为所需 LLM providers 配置 clients。
2. 将配置好的 clients 传入 [`MultiLLMPromptExecutor`](https://api.koog.ai/prompt/prompt-executor/prompt-executor-model/ai.koog.prompt.executor.llms/-multi-l-l-m-prompt-executor/index.html) 类构造函数，以创建带有多个 LLM providers 的 prompt executor。

KotlinJava

```
val openAIClient = OpenAILLMClient(System.getenv("OPENAI_API_KEY"))
val ollamaClient = OllamaClient()

val multiExecutor = MultiLLMPromptExecutor(
    LLMProvider.OpenAI to openAIClient,
    LLMProvider.Ollama to ollamaClient
)
```

```
OpenAILLMClient openAIClient = openAIClient(System.getenv("OPENAI_API_KEY"));
OllamaClient ollamaClient = ollamaClient();

MultiLLMPromptExecutor promptExecutor = new MultiLLMPromptExecutor(openAIClient, ollamaClient);
```

## 创建 routing executor

Experimental API

Routing 能力是实验性的，未来版本中可能会发生变化。
要使用它们，请通过 `@OptIn(ExperimentalRoutingApi::class)` 选择加入。

要创建一个使用路由策略在多个 LLM client instances 之间分发请求的 prompt executor，请执行以下操作：

1. 配置多个 client instances（它们可以属于相同或不同的 LLM providers），并设置相应 API keys。
2. 使用路由策略创建 router，例如 [`RoundRobinRouter`](https://api.koog.ai/prompt/prompt-executor/prompt-executor-model/ai.koog.prompt.executor.llms/-round-robin-router/index.html)。
3. 将 router 传入 [`RoutingLLMPromptExecutor`](https://api.koog.ai/prompt/prompt-executor/prompt-executor-model/ai.koog.prompt.executor.llms/-routing-l-l-m-prompt-executor/index.html) 类构造函数。

这对于避免速率限制、提高吞吐量以及实现故障转移策略很有用。

KotlinJava

```
// Create multiple client instances
val openAI1 = OpenAILLMClient(apiKey = "openai-key-1")
val openAI2 = OpenAILLMClient(apiKey = "openai-key-2")
val anthropic = AnthropicLLMClient(apiKey = "anthropic-key")

// Create router with round-robin strategy
val router = RoundRobinRouter(openAI1, openAI2, anthropic)

// Create routing executor
val routingExecutor = RoutingLLMPromptExecutor(router)
```

```
// Create multiple client instances
OpenAILLMClient openAI1 = openAIClient("openai-key-1");
OpenAILLMClient openAI2 = openAIClient("openai-key-2");
AnthropicLLMClient anthropic = anthropicClient("anthropic-key");

// Create router with round-robin strategy
RoundRobinRouter router = new RoundRobinRouter(openAI1, openAI2, anthropic);

// Create routing executor
RoutingLLMPromptExecutor routingExecutor = new RoutingLLMPromptExecutor(router);
```

当你使用此 executor 执行 prompts 时，对 OpenAI models 的请求会使用 round-robin 策略在 `openAI1` 和 `openAI2` 之间交替。
对 Anthropic models 的请求始终会发送到单个 `anthropic` client，因为 round-robin 会为每个 provider 维护独立计数器。

你也可以通过创建实现 [`LLMClientRouter`](https://api.koog.ai/prompt/prompt-executor/prompt-executor-model/ai.koog.prompt.executor.llms/-l-l-m-client-router/index.html) 接口的类来实现自定义路由策略。

## 预定义 prompt executors

为了更快设置，Koog 为常见 providers 在 Kotlin 和 Java 中提供了开箱即用的 executor 实现。

下表包含**预定义 single-provider executors**，它们返回配置了特定 LLM client 的 `SingleLLMPromptExecutor`。

| LLM provider | Prompt executor | 描述 |
| --- | --- | --- |
| OpenAI | [simpleOpenAIExecutor](https://api.koog.ai/prompt/prompt-executor/prompt-executor-llms-all/ai.koog.prompt.executor.llms.all/simple-open-a-i-executor.html) | 封装运行 OpenAI models prompts 的 `OpenAILLMClient`。 |
| OpenAI | [simpleAzureOpenAIExecutor](https://api.koog.ai/prompt/prompt-executor/prompt-executor-llms-all/ai.koog.prompt.executor.llms.all/simple-azure-open-a-i-executor.html) | 封装配置为使用 Azure OpenAI Service 的 `OpenAILLMClient`。 |
| Anthropic | [simpleAnthropicExecutor](https://api.koog.ai/prompt/prompt-executor/prompt-executor-llms-all/ai.koog.prompt.executor.llms.all/simple-anthropic-executor.html) | 封装运行 Anthropic models prompts 的 `AnthropicLLMClient`。 |
| Google | [simpleGoogleAIExecutor](https://api.koog.ai/prompt/prompt-executor/prompt-executor-llms-all/ai.koog.prompt.executor.llms.all/simple-google-a-i-executor.html) | 封装运行 Google models prompts 的 `GoogleLLMClient`。 |
| OpenRouter | [simpleOpenRouterExecutor](https://api.koog.ai/prompt/prompt-executor/prompt-executor-llms-all/ai.koog.prompt.executor.llms.all/simple-open-router-executor.html) | 封装运行 OpenRouter prompts 的 `OpenRouterLLMClient`。 |
| Amazon Bedrock | [simpleBedrockExecutor](https://api.koog.ai/prompt/prompt-executor/prompt-executor-llms-all/ai.koog.prompt.executor.llms.all/simple-bedrock-executor.html) | 封装运行 AWS Bedrock prompts 的 `BedrockLLMClient`。 |
| Amazon Bedrock | [simpleBedrockExecutorWithBearerToken](https://api.koog.ai/prompt/prompt-executor/prompt-executor-llms-all/ai.koog.prompt.executor.llms.all/simple-bedrock-executor-with-bearer-token.html) | 封装 `BedrockLLMClient`，并使用提供的 Bedrock API key 发送请求。 |
| Mistral | [simpleMistralAIExecutor](https://api.koog.ai/prompt/prompt-executor/prompt-executor-llms-all/ai.koog.prompt.executor.llms.all/simple-mistral-a-i-executor.html) | 封装运行 Mistral models prompts 的 `MistralAILLMClient`。 |
| Ollama | [simpleOllamaAIExecutor](https://api.koog.ai/prompt/prompt-executor/prompt-executor-llms-all/ai.koog.prompt.executor.llms.all/simple-ollama-a-i-executor.html) | 封装运行 Ollama prompts 的 `OllamaClient`。 |

下面是创建预定义 executor 的示例：

KotlinJava

```
// Create an OpenAI executor
val promptExecutor = simpleOpenAIExecutor("OPENAI_API_KEY")
```

```
// Create an OpenAI executor
PromptExecutor openAIExecutor = simpleOpenAIExecutor("OPENAI_API_KEY");
```

## 运行 prompt

要使用 prompt executor 运行 prompt，请执行以下操作：

1. 创建 prompt executor。
2. 使用 `execute()` 方法通过特定 LLM 运行 prompt。

示例如下：

KotlinJava

```
// Create an OpenAI executor
val promptExecutor = simpleOpenAIExecutor("OPENAI_API_KEY")

// Execute a prompt
val response = promptExecutor.execute(
    prompt = prompt("demo") { user("Summarize this.") },
    model = OpenAIModels.Chat.GPT4o
)
```

```
// Create an OpenAI executor
PromptExecutor promptExecutor = simpleOpenAIExecutor("OPENAI_API_KEY");

// Create a prompt
Prompt prompt = Prompt.builder("demo")
    .user("Summarize this.")
    .build();

// Run the prompt
List<Message.Response> response = promptExecutor.execute(prompt, OpenAIModels.Chat.GPT4o);
```

这会使用 `GPT4o` model 运行 prompt 并返回响应。

Note

Prompt executors 提供了使用多种能力运行 prompts 的方法，
例如 streaming、多选生成和内容审核。
由于 prompt executors 封装 LLM clients，每个 executor 都支持相应 client 的能力。
详情请参阅 [LLM clients](../llm-clients/)。

## 在 providers 之间切换

当你使用 `MultiLLMPromptExecutor` 与多个 LLM providers 协作时，可以在它们之间切换。
流程如下：

1. 为你想使用的每个 provider 创建一个 LLM client instance。
2. 创建一个将 LLM providers 映射到 LLM clients 的 `MultiLLMPromptExecutor`。
3. 使用 `execute()` 方法运行 prompt，并将来自相应 client 的 model 作为参数传入。
   prompt executor 会根据 model provider 使用对应 client 来运行 prompt。

下面是在 providers 之间切换的示例：

KotlinJava

```
// Create LLM clients for OpenAI, Anthropic, and Google providers
val openAIClient = OpenAILLMClient("OPENAI_API_KEY")
val anthropicClient = AnthropicLLMClient("ANTHROPIC_API_KEY")
val googleClient = GoogleLLMClient("GOOGLE_API_KEY")

// Create a MultiLLMPromptExecutor that maps LLM providers to LLM clients
val executor = MultiLLMPromptExecutor(
    LLMProvider.OpenAI to openAIClient,
    LLMProvider.Anthropic to anthropicClient,
    LLMProvider.Google to googleClient
)

// Create a prompt
val p = prompt("demo") { user("Summarize this.") }

// Run the prompt with an OpenAI model; the prompt executor automatically switches to the OpenAI client
val openAIResult = executor.execute(p, OpenAIModels.Chat.GPT4o)

// Run the prompt with an Anthropic model; the prompt executor automatically switches to the Anthropic client
val anthropicResult = executor.execute(p, AnthropicModels.Sonnet_4_5)
```

```
// Create LLM clients for OpenAI, Anthropic, and Google providers
OpenAILLMClient openAIClient = openAIClient("OPENAI_API_KEY");
AnthropicLLMClient anthropicClient = anthropicClient("ANTHROPIC_API_KEY");
GoogleLLMClient googleClient = googleClient("GOOGLE_API_KEY");

// Create a MultiLLMPromptExecutor that maps LLM providers to LLM clients
MultiLLMPromptExecutor promptExecutor = new MultiLLMPromptExecutor(
    Map.of(
        LLMProvider.OpenAI, openAIClient,
        LLMProvider.Anthropic, anthropicClient,
        LLMProvider.Google, googleClient
    )
);

// Create a prompt
Prompt prompt = Prompt.builder("demo")
    .user("Summarize this.")
    .build();

// Run the prompt with an OpenAI model; the prompt executor automatically switches to the OpenAI client
List<Message.Response> openAIResult = promptExecutor.execute(prompt, OpenAIModels.Chat.GPT4o);

// Run the prompt with an Anthropic model; the prompt executor automatically switches to the Anthropic client
List<Message.Response> anthropicResult = promptExecutor.execute(prompt, AnthropicModels.Sonnet_4_5);
```

你可以选择配置 fallback LLM provider 和 model，以便在请求的 client 不可用时使用。
详情请参阅 [Configuring fallbacks](#configuring-fallbacks)。

## 配置 fallbacks

Multi-provider 和 routing prompt executors 可以配置为在请求的 LLM client 不可用时使用 fallback LLM provider 和 model。

要配置 fallback 机制，请在创建 `MultiLLMPromptExecutor` 或 `RoutingLLMPromptExecutor` 时传入 fallback 设置：

KotlinJava

```
val openAIClient = OpenAILLMClient(System.getenv("OPENAI_API_KEY"))
val ollamaClient = OllamaClient()

val multiExecutor = MultiLLMPromptExecutor(
    LLMProvider.OpenAI to openAIClient,
    LLMProvider.Ollama to ollamaClient,
    fallback = MultiLLMPromptExecutor.FallbackPromptExecutorSettings(
        fallbackProvider = LLMProvider.Ollama,
        fallbackModel = OllamaModels.Meta.LLAMA_3_2
    )
)
```

```
OpenAILLMClient openAIClient = openAIClient(System.getenv("OPENAI_API_KEY"));
OllamaClient ollamaClient = ollamaClient();

MultiLLMPromptExecutor multiExecutor = new MultiLLMPromptExecutor(
    Map.of(
        LLMProvider.OpenAI, openAIClient,
        LLMProvider.Ollama, ollamaClient
    ),
    new MultiLLMPromptExecutor.FallbackPromptExecutorSettings(
        LLMProvider.Ollama,
        OllamaModels.Meta.LLAMA_3_2
    )
);
```

如果你传入的 model 来自未包含在 `MultiLLMPromptExecutor` 中的 LLM provider，
prompt executor 会使用 fallback model：

KotlinJava

```
// Create a prompt
val p = prompt("demo") { user("Summarize this") }
// If you pass a Google model, the prompt executor will use the fallback model, as the Google client is not included
val response = multiExecutor.execute(p, GoogleModels.Gemini2_5Pro)
```

```
// Create a prompt
Prompt p = Prompt.builder("demo")
    .user("Summarize this")
    .build();

// If you pass a Google model, the prompt executor will use the fallback model, as the Google client is not included
List<Message.Response> response = multiExecutor.execute(p, GoogleModels.Gemini2_5Pro);
```

Note

Fallbacks 仅适用于 `execute()` 和 `executeMultipleChoices()` 方法。