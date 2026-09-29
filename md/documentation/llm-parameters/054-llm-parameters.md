# LLM 参数

本页提供有关 Koog 智能体框架中 LLM 参数的详细信息。LLM 参数让你能够控制和
自定义语言模型的行为。

## 概述

LLM 参数是配置选项，让你能够微调语言模型生成响应的方式。这些参数
控制响应随机性、长度、格式和工具使用等方面。通过调整这些参数，你可以针对不同的
用例优化模型行为，从创意内容生成到确定性的结构化输出。

在 Koog 中，`LLMParams` 类包含 LLM 参数，并为配置语言
模型行为提供了一致的接口。你可以通过以下方式使用 LLM 参数：

- 创建提示词时：

KotlinJava
```
val prompt = prompt(
    id = "dev-assistant",
    params = LLMParams(
        temperature = 0.7,
        maxTokens = 500
    )
) {
    // Add a system message to set the context
    system("You are a helpful assistant.")

    // Add a user message
    user("Tell me about Kotlin")
}
```

```
Prompt prompt = Prompt.builder("dev-assistant")
    .withParams(new LLMParams(
        0.7,         // temperature
        500,         // maxTokens
        1,           // numberOfChoices
        null,        // speculation
        null,        // schema
        LLMParams.ToolChoice.Auto.INSTANCE, // toolChoice
        null,        // user
        null         // additionalProperties
    ))
    .system("You are a helpful assistant.")
    .user("Tell me about Kotlin")
    .build();
```
有关提示词创建的更多信息，请参阅[提示词](../prompts/prompt-creation/)。

- 创建子图时：

KotlinJava
```
val processQuery by subgraphWithTask<String, String>(
    tools = listOf(searchTool, calculatorTool, weatherTool),
    llmModel = OpenAIModels.Chat.GPT4o,
    llmParams = LLMParams(
        temperature = 0.7,
        maxTokens = 500
    ),
    parallelTools = false,
    assistantResponseRepeatMax = 3,
) { userQuery ->
    """
    You are a helpful assistant that can answer questions about various topics.
    Please help with the following query:
    $userQuery
    """
}
```

```

```
有关 Koog 中现有子图类型的更多信息，
请参阅[预定义子图](../nodes-and-components/#predefined-subgraphs)。要了解如何创建和实现你自己的
子图，请参阅[自定义子图](../custom-subgraphs/)。

- 在 LLM 写入会话中更新提示词时：

KotlinJava
```
llm.writeSession {
    changeLLMParams(
        LLMParams(
            temperature = 0.7,
            maxTokens = 500
        )
    )
}
```

```

```
有关会话的更多信息，请参阅 [LLM 会话与手动历史记录管理](../sessions/)。

## LLM 参数参考

下表提供了 `LLMParams` 类中包含的 LLM 参数参考，这些参数受 Koog 开箱即用的所有 LLM
提供商支持。
有关某些提供商特有的参数列表，
请参阅[提供商特有参数](#provider-specific-parameters)。

| 参数 | 类型 | 描述 |
| --- | --- | --- |
| `temperature` | Double | 控制输出的随机性。较高的值（如 0.7–1.0）会产生更多样、更有创意的响应，而较低的值会产生更确定、更聚焦的响应。 |
| `maxTokens` | Integer | 响应中要生成的最大 token 数。可用于控制响应长度。 |
| `numberOfChoices` | Integer | 要生成的备选响应数量。必须大于 0。 |
| `speculation` | String | 一个推测性配置字符串，会影响模型行为，旨在提升结果的速度和准确性。仅受某些模型支持，但可能大幅提升速度和准确性。 |
| `schema` | Schema | 定义模型响应格式的结构，支持 JSON 等结构化输出。有关更多信息，请参阅 [Schema](#schema)。 |
| `toolChoice` | ToolChoice | 控制语言模型的工具调用行为。有关更多信息，请参阅[工具选择](#tool-choice)。 |
| `user` | String | 发起请求的用户的标识符，可用于跟踪目的。 |
| `additionalProperties` | Map<String, JsonElement> | 可用于存储某些模型提供商特有的自定义参数的附加属性。 |

有关每个参数的默认值列表，请参阅相应的 LLM 提供商文档：

- [OpenAI Chat](https://platform.openai.com/docs/api-reference/chat/create)
- [OpenAI Responses](https://platform.openai.com/docs/api-reference/responses/create)
- [Google](https://ai.google.dev/api/generate-content#generationconfig) β
- [Anthropic](https://platform.claude.com/docs/en/api/messages/create)
- [Mistral](https://docs.mistral.ai/api/#operation/chatCompletions) β
- [DeepSeek](https://api-docs.deepseek.com/api/create-chat-completion#request) β
- [OpenRouter](https://openrouter.ai/docs/api/reference/parameters)
- Alibaba β ([DashScope](https://www.alibabacloud.com/help/en/model-studio/qwen-api-reference))
- [Ollama](https://docs.ollama.com/api/openai-compatibility)

## Schema

`Schema` 接口定义模型响应格式的结构。
Koog 支持 JSON schema，如下文各节所述。

### JSON schema

JSON schema 允许你从语言模型请求结构化的 JSON 数据。Koog 支持以下两种类型的 JSON
schema：

1) **基础 JSON Schema**（`LLMParams.Schema.JSON.Basic`）：用于基础 JSON 处理能力。此格式
主要侧重于嵌套数据定义，不包含高级 JSON Schema 功能。

KotlinJava
```
// Create parameters with a basic JSON schema
val jsonParams = LLMParams(
    temperature = 0.2,
    schema = LLMParams.Schema.JSON.Basic(
        name = "PersonInfo",
        schema = JsonObject(mapOf(
            "type" to JsonPrimitive("object"),
            "properties" to JsonObject(
                mapOf(
                    "name" to JsonObject(mapOf("type" to JsonPrimitive("string"))),
                    "age" to JsonObject(mapOf("type" to JsonPrimitive("number"))),
                    "skills" to JsonObject(
                        mapOf(
                            "type" to JsonPrimitive("array"),
                            "items" to JsonObject(mapOf("type" to JsonPrimitive("string")))
                        )
                    )
                )
            ),
            "additionalProperties" to JsonPrimitive(false),
            "required" to JsonArray(listOf(JsonPrimitive("name"), JsonPrimitive("age"), JsonPrimitive("skills")))
        ))
    )
)
```

```
// Create parameters with a basic JSON schema
LLMParams jsonParams = new LLMParams(
    0.2,         // temperature
    null,        // maxTokens
    1,           // numberOfChoices
    null,        // speculation
    new LLMParams.Schema.JSON.Basic(
        "PersonInfo",
        new JsonObject(Map.of(
            "type", new JsonPrimitive("object"),
            "properties", new JsonObject(Map.of(
                "name", new JsonObject(Map.of("type", new JsonPrimitive("string"))),
                "age", new JsonObject(Map.of("type", new JsonPrimitive("number"))),
                "skills", new JsonObject(Map.of(
                    "type", new JsonPrimitive("array"),
                    "items", new JsonObject(Map.of("type", new JsonPrimitive("string")))
                ))
            )),
            "additionalProperties", new JsonPrimitive(false),
            "required", new JsonArray(List.of(
                new JsonPrimitive("name"),
                new JsonPrimitive("age"),
                new JsonPrimitive("skills")
            ))
        ))
    ),
    LLMParams.ToolChoice.Auto.INSTANCE, // toolChoice
    null,        // user
    null         // additionalProperties
);
```
2) **标准 JSON Schema**（`LLMParams.Schema.JSON.Standard`）：表示符合 [json-schema.org](https://json-schema.org/) 的标准 JSON schema。此格式是官方 JSON Schema 规范的一个真子集。请注意，不同 LLM 提供商之间的风格可能会有所不同，因为并非所有提供商都支持完整的 JSON schema。

KotlinJava
```
// Create parameters with a standard JSON schema
val standardJsonParams = LLMParams(
    temperature = 0.2,
    schema = LLMParams.Schema.JSON.Standard(
        name = "ProductCatalog",
        schema = JsonObject(mapOf(
            "type" to JsonPrimitive("object"),
            "properties" to JsonObject(mapOf(
                "products" to JsonObject(mapOf(
                    "type" to JsonPrimitive("array"),
                    "items" to JsonObject(mapOf(
                        "type" to JsonPrimitive("object"),
                        "properties" to JsonObject(mapOf(
                            "id" to JsonObject(mapOf("type" to JsonPrimitive("string"))),
                            "name" to JsonObject(mapOf("type" to JsonPrimitive("string"))),
                            "price" to JsonObject(mapOf("type" to JsonPrimitive("number"))),
                            "description" to JsonObject(mapOf("type" to JsonPrimitive("string")))
                        )),
                        "additionalProperties" to JsonPrimitive(false),
                        "required" to JsonArray(listOf(JsonPrimitive("id"), JsonPrimitive("name"), JsonPrimitive("price"), JsonPrimitive("description")))
                    ))
                ))
            )),
            "additionalProperties" to JsonPrimitive(false),
            "required" to JsonArray(listOf(JsonPrimitive("products")))
        ))
    )
)
```

```
// Create parameters with a standard JSON schema
LLMParams standardJsonParams = new LLMParams(
    0.2,         // temperature
    null,        // maxTokens
    1,           // numberOfChoices
    null,        // speculation
    new LLMParams.Schema.JSON.Standard(
        "ProductCatalog",
        new JsonObject(Map.of(
            "type", new JsonPrimitive("object"),
            "properties", new JsonObject(Map.of(
                "products", new JsonObject(Map.of(
                    "type", new JsonPrimitive("array"),
                    "items", new JsonObject(Map.of(
                        "type", new JsonPrimitive("object"),
                        "properties", new JsonObject(Map.of(
                            "id", new JsonObject(Map.of("type", new JsonPrimitive("string"))),
                            "name", new JsonObject(Map.of("type", new JsonPrimitive("string"))),
                            "price", new JsonObject(Map.of("type", new JsonPrimitive("number"))),
                            "description", new JsonObject(Map.of("type", new JsonPrimitive("string")))
                        )),
                        "additionalProperties", new JsonPrimitive(false),
                        "required", new JsonArray(List.of(
                            new JsonPrimitive("id"),
                            new JsonPrimitive("name"),
                            new JsonPrimitive("price"),
                            new JsonPrimitive("description")
                        ))
                    ))
                ))
            )),
            "additionalProperties", new JsonPrimitive(false),
            "required", new JsonArray(List.of(new JsonPrimitive("products")))
        ))
    ),
    LLMParams.ToolChoice.Auto.INSTANCE, // toolChoice
    null,        // user
    null         // additionalProperties
);
```
## 工具选择

`ToolChoice` 类控制语言模型如何使用工具。它提供以下选项：

- `LLMParams.ToolChoice.Named`：语言模型调用指定的工具。接受 `name` 字符串参数，该参数
  表示要调用的工具名称。
- `LLMParams.ToolChoice.All`：语言模型调用所有工具。
- `LLMParams.ToolChoice.None`：语言模型不调用工具，仅生成文本。
- `LLMParams.ToolChoice.Auto`：语言模型自动决定是否调用工具以及调用哪个工具。
- `LLMParams.ToolChoice.Required`：语言模型至少调用一个工具。

以下是使用 `LLMParams.ToolChoice.Named` 类调用特定工具的示例：

KotlinJava
```
val specificToolParams = LLMParams(
    toolChoice = LLMParams.ToolChoice.Named(name = "calculator")
)
```

```
LLMParams specificToolParams = new LLMParams(
    null,        // temperature
    null,        // maxTokens
    1,           // numberOfChoices
    null,        // speculation
    null,        // schema
    new LLMParams.ToolChoice.Named("calculator"), // toolChoice
    null,        // user
    null         // additionalProperties
);
```
## 提供商特定参数

Koog 支持为某些 LLM 提供商设置提供商特定参数。这些参数扩展了基础 `LLMParams` 类，并添加了提供商特定的功能。以下类包含各提供商特定的参数：

- `OpenAIChatParams`：特定于 OpenAI Chat Completions API 的参数。
- `OpenAIResponsesParams`：特定于 OpenAI Responses API 的参数。
- `GoogleParams`：特定于 Google 模型的参数。
- `AnthropicParams`：特定于 Anthropic 模型的参数。
- `MistralAIParams`：特定于 Mistral 模型的参数。
- `DeepSeekParams`：特定于 DeepSeek 模型的参数。
- `OpenRouterParams`：特定于 OpenRouter 模型的参数。
- `DashscopeParams`：特定于阿里巴巴模型的参数。
- `OllamaParams`：特定于 Ollama 模型的参数。

以下是 Koog 中提供商特定参数的完整参考：

OpenAI ChatOpenAI ResponsesGoogleAnthropicMistralDeepSeekOpenRouterAlibaba (DashScope)Ollama
| 参数 | 类型 | 描述 |
| --- | --- | --- |
| `audio` | OpenAIAudioConfig | 使用支持音频的模型时的音频输出配置。有关更多信息，请参阅 [OpenAIAudioConfig](api:prompt-executor-openai-client-base::ai.koog.prompt.executor.clients.openai.base.models.OpenAIAudioConfig) 的 API 文档。 |
| `frequencyPenalty` | Double | 对频繁出现的 token 进行惩罚以减少重复。较高的 `frequencyPenalty` 值会导致更大的措辞变化并减少重复。取值范围为 -2.0 到 2.0。 |
| `logprobs` | Boolean | 如果为 `true`，则包含输出 token 的对数概率。 |
| `parallelToolCalls` | Boolean | 如果为 `true`，多个工具调用可以并行运行。特别适用于自定义节点或代理策略之外的 LLM 交互。 |
| `presencePenalty` | Double | 防止模型重复使用已包含在输出中的 token。较高的值鼓励引入新的 token 和主题。取值范围为 -2.0 到 2.0。 |
| `promptCacheKey` | String | 用于提示缓存的稳定缓存键。OpenAI 使用它来缓存相似请求的响应。 |
| `reasoningEffort` | ReasoningEffort | 指定模型将使用的推理努力程度。有关更多信息和可用值，请参阅 [ReasoningEffort](api:prompt-executor-openai-client-base::ai.koog.prompt.executor.clients.openai.base.models.ReasoningEffort) 的 API 文档。 |
| `safetyIdentifier` | String | 一个稳定且唯一的用户标识符，可用于检测违反 OpenAI 政策的用户。 |
| `serviceTier` | ServiceTier | OpenAI 处理层级选择，让您可以在性能与成本之间进行优先级排序。有关更多信息，请参阅 [ServiceTier](api:prompt-executor-openai-client-base::ai.koog.prompt.executor.clients.openai.base.models.ServiceTier) 的 API 文档。 |
| `stop` | List<String> | 当模型遇到其中任何一个字符串时，会发出停止生成内容的信号。例如，要让模型在产生两个换行符时停止生成内容，请将停止序列指定为 `stop = listOf("/n/n")`。 |
| `store` | Boolean | 如果为 `true`，提供方可能会存储输出以供后续检索。 |
| `topLogprobs` | Integer | 每个位置最可能的 token 数量。取值范围为 0–20。需要将 `logprobs` 参数设置为 `true`。 |
| `topP` | Double | 也称为核采样。通过将概率值最高的 token 添加到子集中来创建下一个 token 的子集，直到它们的概率之和达到指定的 `topP` 值。取值大于 0.0 且小于或等于 1.0。 |
| `webSearchOptions` | OpenAIWebSearchOptions | 配置网络搜索工具的使用（如果支持）。有关更多信息，请参阅 [OpenAIWebSearchOptions](api:prompt-executor-openai-client-base::ai.koog.prompt.executor.clients.openai.base.models.OpenAIWebSearchOptions) 的 API 文档。 |
| 参数 | 类型 | 描述 |
| --- | --- | --- |
| `background` | Boolean | 在后台运行响应。 |
| `include` | List<OpenAIInclude> | 要包含在模型响应中的额外数据，例如网络搜索工具调用的来源或文件搜索工具调用的搜索结果。有关详细参考信息，请参阅 Koog API 参考中的 [OpenAIInclude](api:prompt-executor-openai-client::ai.koog.prompt.executor.clients.openai.models.OpenAIInclude)。要了解有关 `include` 参数的更多信息，请参阅 [OpenAI 的文档](https://platform.openai.com/docs/api-reference/responses/create#responses-create-include)。 |
| `logprobs` | Boolean | 如果为 `true`，则包含输出 token 的对数概率。 |
| `maxToolCalls` | Integer | 此响应中允许的内置工具调用总次数上限。取值等于或大于 `0`。 |
| `parallelToolCalls` | Boolean | 如果为 `true`，则可以并行运行多个工具调用。特别适用于自定义节点或代理策略之外的 LLM 交互。 |
| `promptCacheKey` | String | 用于提示缓存的稳定缓存键。OpenAI 使用它来缓存相似请求的响应。 |
| `reasoning` | ReasoningConfig | 支持推理的模型的推理配置。有关更多信息，请参阅 [ReasoningConfig](api:prompt-executor-openai-client::ai.koog.prompt.executor.clients.openai.models.ReasoningConfig) 的 API 文档。 |
| `safetyIdentifier` | String | 一个稳定且唯一的用户标识符，可用于检测违反 OpenAI 政策的用户。 |
| `serviceTier` | ServiceTier | OpenAI 处理层级选择，让您可以在性能与成本之间进行优先级取舍。有关更多信息，请参阅 [ServiceTier](api:prompt-executor-openai-client-base::ai.koog.prompt.executor.clients.openai.base.models.ServiceTier) 的 API 文档。 |
| `store` | Boolean | 如果为 `true`，提供方可能会存储输出以供后续检索。 |
| `topLogprobs` | Integer | 每个位置最可能的 token 数量。取值范围为 0–20。需要将 `logprobs` 参数设置为 `true`。 |
| `topP` | Double | 也称为核采样。通过将概率值最高的 token 添加到子集中，直到其概率之和达到指定的 `topP` 值，从而创建下一个 token 的子集。取值大于 0.0 且小于或等于 1.0。 |
| `truncation` | Truncation | 接近上下文窗口时的截断策略。有关更多信息，请参阅 [Truncation](api:prompt-executor-openai-client::ai.koog.prompt.executor.clients.openai.models.Truncation) 的 API 文档。 |

| 参数 | 类型 | 描述 |
| --- | --- | --- |
| `thinkingConfig` | GoogleThinkingConfig | 控制模型是否应公开其思维链以及可用于思维链的 token 数量。有关更多信息，请参阅 [GoogleThinkingConfig](api:prompt-executor-google-client::ai.koog.prompt.executor.clients.google.models.GoogleThinkingConfig) 的 API 参考。 |
| `topK` | Integer | 生成输出时要考虑的最可能的 token 数量。取值大于或等于 0（可能适用特定于提供方的最小值）。 |
| `topP` | Double | 也称为核采样。通过将概率值最高的 token 添加到子集中，直到其概率之和达到指定的 `topP` 值，从而创建下一个 token 的子集。取值大于 0.0 且小于或等于 1.0。 |
| 参数 | 类型 | 描述 |
| --- | --- | --- |
| `container` | String | 用于跨请求复用的容器标识符。Anthropic 的代码执行工具使用容器来提供安全且容器化的代码执行环境。通过提供先前响应中的容器标识符，你可以在多个请求之间复用容器，从而在请求之间保留已创建的文件。有关更多信息，请参阅 Anthropic 文档中的 [Containers](https://platform.claude.com/docs/en/agents-and-tools/tool-use/code-execution-tool#containers)。 |
| `mcpServers` | List<AnthropicMCPServerURLDefinition> | 请求中要使用的 MCP 服务器定义。最多支持 20 个服务器。有关更多信息，请参阅 [AnthropicMCPServerURLDefinition](api:prompt-executor-anthropic-client::ai.koog.prompt.executor.clients.anthropic.models.AnthropicMCPServerURLDefinition) 的 API 参考。 |
| `serviceTier` | ServiceTier | OpenAI 处理层级选择，可让你在性能与成本之间进行权衡。有关更多信息，请参阅 [ServiceTier](api:prompt-executor-openai-client-base::ai.koog.prompt.executor.clients.openai.base.models.ServiceTier) 的 API 文档。 |
| `stopSequences` | List<String> | 导致模型停止生成内容的自定义文本序列。如果匹配，响应中 `stop_reason` 的值为 `stop_sequence`。 |
| `thinking` | AnthropicThinking | 用于激活 Claude 扩展思考的配置。激活后，响应还会包含思考内容块。有关更多信息，请参阅 [AnthropicThinking](api:prompt-executor-anthropic-client::ai.koog.prompt.executor.clients.anthropic.models.AnthropicThinking) 的 API 参考。 |
| `topK` | Integer | 生成输出时要考虑的前 K 个 token 的数量。取值大于或等于 0（可能适用特定于提供商的最小值）。 |
| `topP` | Double | 也称为核采样。通过将概率值最高的 token 添加到子集中，直到其概率之和达到指定的 `topP` 值，从而创建下一个 token 的子集。取值大于 0.0 且小于或等于 1.0。 |
| 参数 | 类型 | 描述 |
| --- | --- | --- |
| `frequencyPenalty` | Double | 对频繁出现的 token 进行惩罚以减少重复。较高的 `frequencyPenalty` 值会导致措辞变化更大并减少重复。取值范围为 -2.0 到 2.0。 |
| `parallelToolCalls` | Boolean | 如果为 `true`，多个工具调用可以并行运行。特别适用于自定义节点或代理策略之外的 LLM 交互。 |
| `presencePenalty` | Double | 防止模型重复使用已包含在输出中的 token。较高的值会鼓励引入新的 token 和主题。取值范围为 -2.0 到 2.0。 |
| `promptMode` | String | 允许你在推理模式和无系统提示之间切换。当设置为 `reasoning` 时，将使用推理模型的默认系统提示。有关更多信息，请参阅 Mistral 的 [Reasoning](https://docs.mistral.ai/capabilities/reasoning) 文档。 |
| `randomSeed` | Integer | 用于随机采样的种子。如果设置，使用相同参数和相同种子值的不同调用将生成确定性结果。 |
| `safePrompt` | Boolean | 指定是否在所有对话之前注入安全提示。安全提示用于实施护栏并防止有害内容。有关更多信息，请参阅 Mistral 的 [Moderation & Guardarailing](https://docs.mistral.ai/capabilities/guardrailing) 文档。 |
| `stop` | List<String> | 当模型遇到其中任何一个字符串时，会发出停止生成内容的信号。例如，要让模型在产生两个换行符时停止生成内容，请将停止序列指定为 `stop = listOf("/n/n")`。 |
| `topP` | Double | 也称为核采样。通过将概率值最高的 token 添加到子集中来创建下一个 token 的子集，直到它们的概率之和达到指定的 `topP` 值。取值范围为大于 0.0 且小于或等于 1.0。 |

| 参数 | 类型 | 描述 |
| --- | --- | --- |
| `frequencyPenalty` | Double | 对频繁出现的 token 进行惩罚以减少重复。较高的 `frequencyPenalty` 值会导致措辞变化更大并减少重复。取值范围为 -2.0 到 2.0。 |
| `logprobs` | Boolean | 如果为 `true`，则包含输出 token 的对数概率。 |
| `presencePenalty` | Double | 防止模型重复使用已包含在输出中的 token。较高的值会鼓励引入新的 token 和主题。取值范围为 -2.0 到 2.0。 |
| `stop` | List<String> | 当模型遇到其中任何一个字符串时，会发出停止生成内容的信号。例如，要让模型在产生两个换行符时停止生成内容，请将停止序列指定为 `stop = listOf("/n/n")`。 |
| `topLogprobs` | Integer | 每个位置最可能的顶部 token 数量。取值范围为 0–20。需要将 `logprobs` 参数设置为 `true`。 |
| `topP` | Double | 也称为核采样。通过将概率值最高的 token 添加到子集中来创建下一个 token 的子集，直到它们的概率之和达到指定的 `topP` 值。取值范围为大于 0.0 且小于或等于 1.0。 |
| 参数 | 类型 | 描述 |
| --- | --- | --- |
| `frequencyPenalty` | Double | 对频繁出现的 token 进行惩罚以减少重复。`frequencyPenalty` 值越高，措辞变化越大，重复越少。取值范围为 -2.0 到 2.0。 |
| `logprobs` | Boolean | 如果为 `true`，则包含输出 token 的对数概率。 |
| `minP` | Double | 过滤掉相对于最可能 token 的相对概率低于指定 `minP` 值的 token。取值范围为 0.0–0.1。 |
| `models` | List<String> | 请求允许使用的模型列表。 |
| `presencePenalty` | Double | 防止模型重复使用已出现在输出中的 token。值越高，越鼓励引入新的 token 和主题。取值范围为 -2.0 到 2.0。 |
| `provider` | ProviderPreferences | 包含一系列参数，可让你显式控制 OpenRouter 如何选择使用哪个 LLM 提供商。有关更多信息，请参阅 [ProviderPreferences](api:prompt-executor-openrouter-client::ai.koog.prompt.executor.clients.openrouter.models.ProviderPreferences) 的 API 文档。 |
| `repetitionPenalty` | Double | 对 token 重复进行惩罚。已出现在输出中的 token 的下一个 token 概率会除以 `repetitionPenalty` 的值，这使得它们在 `repetitionPenalty > 1` 时更不容易再次出现。取值范围为大于 0.0 且小于或等于 2.0。 |
| `route` | String | 要使用的请求路由策略。 |
| `stop` | List<String> | 当模型遇到其中任意一个字符串时，会发出停止生成内容的信号。例如，要让模型在产生两个换行符时停止生成内容，可将停止序列指定为 `stop = listOf("/n/n")`。 |
| `topA` | Double | 根据模型置信度动态调整采样窗口。如果模型有把握（存在占主导的高概率下一个 token），则将采样窗口限制在少数几个顶级 token 内。如果置信度较低（有许多概率相近的 token），则在采样窗口中保留更多 token。取值范围为 0.0–0.1（含边界）。值越高表示动态适应性越强。 |
| `topK` | Integer | 生成输出时要考虑的前几个 token 的数量。取值范围为大于或等于 0（可能因提供商而异存在最小值限制）。 |
| `topLogprobs` | Integer | 每个位置最可能的前几个 token 的数量。取值范围为 0–20。需要将 `logprobs` 参数设置为 `true`。 |
| `topP` | Double | 也称为核采样。通过将概率值最高的 token 添加到子集中，直到它们的概率之和达到指定的 `topP` 值，从而创建下一个 token 的子集。取值范围为大于 0.0 且小于或等于 1.0。 |
| `transforms` | List<String> | 上下文转换列表。定义当上下文超出模型 token 限制时如何转换上下文。默认转换为 `middle-out`，即从提示词的中间进行截断。使用空列表表示不进行任何转换。有关更多信息，请参阅 OpenRouter 文档中的 [Message Transforms](https://openrouter.ai/docs/guides/features/message-transforms)。 |
| 参数 | 类型 | 描述 |
| --- | --- | --- |
| `enableSearch` | Boolean | 指定是否启用网络搜索功能。有关更多信息，请参阅阿里云的 [Web 搜索](https://www.alibabacloud.com/help/en/model-studio/web-search?spm=a2c63.p38356.0.i14) 文档。 |
| `enableThinking` | Boolean | 指定在使用混合思考模型时是否启用思考模式。有关更多信息，请参阅阿里云关于[深度思考](https://www.alibabacloud.com/help/en/model-studio/deep-thinking?spm=a2c63.p38356.0.i11)的文档。 |
| `frequencyPenalty` | Double | 对频繁出现的 token 进行惩罚以减少重复。`frequencyPenalty` 值越高，措辞变化越大，重复越少。取值范围为 -2.0 到 2.0。 |
| `logprobs` | Boolean | 如果为 `true`，则包含输出 token 的对数概率。 |
| `parallelToolCalls` | Boolean | 如果为 `true`，则可以并行运行多个工具调用。特别适用于自定义节点或代理策略之外的 LLM 交互。 |
| `presencePenalty` | Double | 防止模型重复使用已包含在输出中的 token。值越高，越鼓励引入新的 token 和主题。取值范围为 -2.0 到 2.0。 |
| `stop` | List<String> | 当模型遇到其中任意一个字符串时，会发出停止生成内容的信号。例如，要让模型在产生两个换行符时停止生成内容，可将停止序列指定为 `stop = listOf("/n/n")`。 |
| `topLogprobs` | Integer | 每个位置最可能的 top token 数量。取值范围为 0–20。需要将 `logprobs` 参数设置为 `true`。 |
| `topP` | Double | 也称为核采样。通过将概率值最高的 token 添加到子集中，直到它们的概率之和达到指定的 `topP` 值，从而创建下一个 token 的子集。取值大于 0.0 且小于或等于 1.0。 |

| 参数 | 类型 | 描述 |
| --- | --- | --- |
| `think` | Boolean | 用于激活 Ollama 扩展思考的配置。激活后，响应还会包含思考内容块。有关更多信息，请参阅 [Ollama 思考](https://docs.ollama.com/capabilities/thinking#enable-thinking-in-api-calls)的 API 参考。 |

以下示例展示了使用提供商特定的 `OpenRouterParams` 类定义的 OpenRouter LLM 参数：

KotlinJava
```
val openRouterParams = OpenRouterParams(
    temperature = 0.7,
    maxTokens = 500,
    frequencyPenalty = 0.5,
    presencePenalty = 0.5,
    topP = 0.9,
    topK = 40,
    repetitionPenalty = 1.1,
    models = listOf("anthropic/claude-3-opus", "anthropic/claude-3-sonnet"),
    transforms = listOf("middle-out")
)
```

```
OpenRouterParams openRouterParams = new OpenRouterParams(
    0.7,         // temperature
    500,         // maxTokens
    1,           // numberOfChoices
    null,        // speculation
    null,        // schema
    null,        // toolChoice
    null,        // user
    null,        // additionalProperties
    0.5,         // frequencyPenalty
    null,        // logprobs
    null,        // minP
    Arrays.asList("anthropic/claude-3-opus", "anthropic/claude-3-sonnet"), // models
    0.5,         // presencePenalty
    null,        // provider
    1.1,         // repetitionPenalty
    null,        // route
    null,        // stop
    null,        // topA
    40,          // topK
    null,        // topLogprobs
    0.9,         // topP
    Arrays.asList("middle-out") // transforms
);
```
## 使用示例

### 基本用法

KotlinJava
```
// A basic set of parameters with limited length
val basicParams = LLMParams(
    temperature = 0.7,
    maxTokens = 150,
    toolChoice = LLMParams.ToolChoice.Auto
)
```

```
// A basic set of parameters with limited length
LLMParams basicParams = new LLMParams(
    0.7,         // temperature
    150,         // maxTokens
    1,           // numberOfChoices
    null,        // speculation
    null,        // schema
    LLMParams.ToolChoice.Auto.INSTANCE, // toolChoice
    null,        // user
    null         // additionalProperties
);
```
### 推理控制

你通过控制模型推理的特定于提供程序的参数来实现推理控制。
使用 OpenAI Chat API 和支持推理的模型时，使用 `reasoningEffort` 参数
来控制模型在提供响应之前生成多少推理 token：

KotlinJava
```
val openAIReasoningEffortParams = OpenAIChatParams(
    reasoningEffort = ReasoningEffort.MEDIUM
)
```

```
OpenAIChatParams openAIReasoningEffortParams = new OpenAIChatParams(
    null,        // temperature
    null,        // maxTokens
    1,           // numberOfChoices
    null,        // speculation
    null,        // schema
    null,        // toolChoice
    null,        // user
    null,        // additionalProperties
    null,        // audio
    null,        // frequencyPenalty
    null,        // logprobs
    null,        // parallelToolCalls
    null,        // presencePenalty
    null,        // promptCacheKey
    ReasoningEffort.MEDIUM, // reasoningEffort
    null,        // safetyIdentifier
    null,        // serviceTier
    null,        // stop
    null,        // store
    null,        // topLogprobs
    null,        // topP
    null         // webSearchOptions
);
```
此外，在无状态模式下使用 OpenAI Responses API 时，你会保留推理项的加密历史记录，并在每一轮对话中将其发送给模型。加密在 OpenAI 侧完成，你需要通过在请求中将 `include` 参数设置为 `reasoning.encrypted_content` 来请求加密的推理令牌。然后，你可以在接下来的对话轮次中将加密的推理令牌传回给模型。

KotlinJava
```
val openAIStatelessReasoningParams = OpenAIResponsesParams(
    include = listOf(OpenAIInclude.REASONING_ENCRYPTED_CONTENT)
)
```

```
OpenAIResponsesParams openAIStatelessReasoningParams = new OpenAIResponsesParams(
    null,        // temperature
    null,        // maxTokens
    1,           // numberOfChoices
    null,        // speculation
    null,        // schema
    null,        // toolChoice
    null,        // user
    null,        // additionalProperties
    null,        // background
    Arrays.asList(OpenAIInclude.REASONING_ENCRYPTED_CONTENT), // include
    null,        // logprobs
    null,        // maxToolCalls
    null,        // parallelToolCalls
    null,        // promptCacheKey
    null,        // reasoning
    null,        // safetyIdentifier
    null,        // serviceTier
    null,        // store
    null,        // topLogprobs
    null,        // topP
    null         // truncation
);
```
### 自定义参数

要添加可能特定于提供方且 Koog 开箱即不支持的自定义参数，请使用
`additionalProperties` 属性，如下例所示。

KotlinJava
```
// Add custom parameters for specific model providers
val customParams = LLMParams(
    additionalProperties = additionalPropertiesOf(
        "top_p" to 0.95,
        "frequency_penalty" to 0.5,
        "presence_penalty" to 0.5
    )
)
```

```
// Add custom parameters for specific model providers
LLMParams customParams = new LLMParams(
    null,        // temperature
    null,        // maxTokens
    1,           // numberOfChoices
    null,        // speculation
    null,        // schema
    null,        // toolChoice
    null,        // user
    AdditionalPropertiesKt.additionalPropertiesOf(
        "top_p", 0.95,
        "frequency_penalty", 0.5,
        "presence_penalty", 0.5
    )
);
```
### 设置和覆盖参数

下面的代码示例展示了如何定义一组你可能主要想使用的 LLM 参数，
然后通过部分覆盖原始集合中的值并向其中添加新值来创建另一组参数。
这让你可以定义大多数请求通用的参数，同时也能添加更具体的参数组合，
而无需重复通用参数。

KotlinJava
```
// Define default parameters
val defaultParams = LLMParams(
    temperature = 0.7,
    maxTokens = 150,
    toolChoice = LLMParams.ToolChoice.Auto
)

// Create parameters with some overrides, using defaults for the rest
val overrideParams = LLMParams(
    temperature = 0.2,
    numberOfChoices = 3
).default(defaultParams)
```

```
// Define default parameters
LLMParams defaultParams = new LLMParams(
    0.7,         // temperature
    150,         // maxTokens
    1,           // numberOfChoices
    null,        // speculation
    null,        // schema
    LLMParams.ToolChoice.Auto.INSTANCE, // toolChoice
    null,        // user
    null         // additionalProperties
);

// Create parameters with some overrides, using defaults for the rest
LLMParams overrideParams = new LLMParams(
    0.2,         // temperature
    null,        // maxTokens
    3,           // numberOfChoices
    null,        // speculation
    null,        // schema
    null,        // toolChoice
    null,        // user
    null         // additionalProperties
).applyDefaults(defaultParams);
```
生成的 `overrideParams` 集合中的值等同于以下内容：

KotlinJava
```
val overrideParams = LLMParams(
    temperature = 0.2,
    maxTokens = 150,
    toolChoice = LLMParams.ToolChoice.Auto,
    numberOfChoices = 3
)
```

```
LLMParams overrideParams = new LLMParams(
    0.2,         // temperature
    150,         // maxTokens
    3,           // numberOfChoices
    null,        // speculation
    null,        // schema
    LLMParams.ToolChoice.Auto.INSTANCE, // toolChoice
    null,        // user
    null         // additionalProperties
);
```
