# 结构化输出

## 简介

结构化输出 API 提供了一种确保大型语言模型（LLM）响应符合特定数据结构的方法。
这对于构建可靠的 AI 应用至关重要，因为你需要可预测、格式良好的数据，而不是自由形式的文本。

本页介绍如何使用此 API 来定义数据结构、生成 schema，以及
向 LLM 请求结构化响应。

## 关键组件和概念

结构化输出 API 由几个关键组件组成：

1. **数据结构定义**：使用 kotlinx.serialization 和 LLM 特定注解标注的 Kotlin 数据类。
2. **JSON Schema 生成**：从 Kotlin 数据类生成 JSON schema 的工具。
3. **结构化 LLM 请求**：向 LLM 请求符合已定义结构的响应的方法。
4. **响应处理**：处理和验证结构化响应。

## 定义数据结构

使用结构化输出 API 的第一步是使用 Kotlin 数据类定义数据结构。

### 基本结构

```
@Serializable
@SerialName("WeatherForecast")
@LLMDescription("Weather forecast for a given location")
data class WeatherForecast(
    @property:LLMDescription("Temperature in Celsius")
    val temperature: Int,
    @property:LLMDescription("Weather conditions (e.g., sunny, cloudy, rainy)")
    val conditions: String,
    @property:LLMDescription("Chance of precipitation in percentage")
    val precipitation: Int
)
```

### 关键注解

- `@Serializable`：kotlinx.serialization 与该类配合工作所必需的注解。
- `@SerialName`：指定序列化时使用的名称。
- `@LLMDescription`：为 LLM 提供类的描述。对于字段注解，请使用 `@property:LLMDescription`。

### 支持的功能

该 API 支持广泛的数据结构功能：

#### 嵌套类

```
@Serializable
@SerialName("WeatherForecast")
data class WeatherForecast(
    // Other fields
    @property:LLMDescription("Coordinates of the location")
    val latLon: LatLon
) {
    @Serializable
    @SerialName("LatLon")
    data class LatLon(
        @property:LLMDescription("Latitude of the location")
        val lat: Double,
        @property:LLMDescription("Longitude of the location")
        val lon: Double
    )
}
```

#### 集合（列表和映射）

```
@Serializable
@SerialName("WeatherForecast")
data class WeatherForecast(
    // Other fields
    @property:LLMDescription("List of news articles")
    val news: List<WeatherNews>,
    @property:LLMDescription("Map of weather sources")
    val sources: Map<String, WeatherSource>
)
```

#### 枚举

```
@Serializable
@SerialName("Pollution")
enum class Pollution { Low, Medium, High }
```

#### 使用密封类实现多态

```
@Serializable
@SerialName("WeatherAlert")
sealed class WeatherAlert {
    abstract val severity: Severity
    abstract val message: String

    @Serializable
    @SerialName("Severity")
    enum class Severity { Low, Moderate, Severe, Extreme }

    @Serializable
    @SerialName("StormAlert")
    data class StormAlert(
        override val severity: Severity,
        override val message: String,
        @property:LLMDescription("Wind speed in km/h")
        val windSpeed: Double
    ) : WeatherAlert()

    @Serializable
    @SerialName("FloodAlert")
    data class FloodAlert(
        override val severity: Severity,
        override val message: String,
        @property:LLMDescription("Expected rainfall in mm")
        val expectedRainfall: Double
    ) : WeatherAlert()
}
```

### 提供示例

你可以提供示例来帮助 LLM 理解预期的格式：

```
val exampleForecasts = listOf(
  WeatherForecast(
    news = listOf(WeatherNews(0.0), WeatherNews(5.0)),
    sources = mutableMapOf(
      "openweathermap" to WeatherSource(Url("https://api.openweathermap.org/data/2.5/weather")),
      "googleweather" to WeatherSource(Url("https://weather.google.com"))
    )
    // Other fields
  ),
  WeatherForecast(
    news = listOf(WeatherNews(25.0), WeatherNews(35.0)),
    sources = mutableMapOf(
      "openweathermap" to WeatherSource(Url("https://api.openweathermap.org/data/2.5/weather")),
      "googleweather" to WeatherSource(Url("https://weather.google.com"))
    )
  )
)
```

## 请求结构化响应

在 Koog 中，你可以在三个主要层面使用结构化输出：

1. **提示执行器层面**：使用提示执行器直接进行 LLM 调用
2. **Agent LLM 上下文层面**：在 agent 会话中用于对话上下文
3. **节点层面**：创建具有结构化输出能力的可复用 agent 节点

### 层面 1：提示执行器

提示执行器层面提供了进行结构化 LLM 调用的最直接方式。使用 `executeStructured` 方法进行单个独立请求：

此方法执行提示并确保响应正确结构化，具体方式包括：

- 根据[模型能力](../model-capabilities/)自动选择最佳结构化输出方法
- 在需要时将结构化输出指令注入原始提示
- 在可用时使用原生结构化输出支持
- 可选地在解析失败时通过辅助 LLM 提供自动错误纠正（通过 `fixingParser` 参数）

以下是使用 `executeStructured` 方法的示例：

```
// Define a simple, single-provider prompt executor
val promptExecutor = simpleOpenAIExecutor(System.getenv("OPENAI_KEY"))

// Make an LLM call that returns a structured response
val structuredResponse = promptExecutor.executeStructured<WeatherForecast>(
        // Define the prompt (both system and user messages)
        prompt = prompt("structured-data") {
            system(
                """
                You are a weather forecasting assistant.
                When asked for a weather forecast, provide a realistic but fictional forecast.
                """.trimIndent()
            )
            user(
              "What is the weather forecast for Amsterdam?"
            )
        },
        // Define the main model that will execute the request
        model = OpenAIModels.Chat.GPT4oMini,
        // Optional: provide examples to help the model understand the format
        examples = exampleForecasts,
        // Optional: provide a fixing parser for error correction
        fixingParser = StructureFixingParser(
            model = OpenAIModels.Chat.GPT4o,
            retries = 3
        )
    )
```

`executeStructured` 方法接受以下参数：

| 名称 | 数据类型 | 必需 | 默认值 | 描述 |
| --- | --- | --- | --- | --- |
| `prompt` | Prompt | 是 |  | 要执行的提示。有关更多信息，请参阅[提示](../prompts/)。 |
| `model` | LLModel | 是 |  | 用于执行提示的主模型。 |
| `examples` | List | 否 | `emptyList()` | 可选的示例列表，用于帮助模型理解预期格式。 |
| `fixingParser` | StructureFixingParser? | 否 | `null` | 可选解析器，通过使用辅助 LLM 智能修复解析错误来处理格式错误的响应。提供后，会自动重试失败的解析并进行错误纠正。 |

该方法返回 `Result<StructuredResponse<T>>`，其中包含成功解析的结构化数据或错误。

### 层面 2：Agent LLM 上下文

Agent LLM 上下文层面允许你在 agent 会话中请求结构化响应。这对于构建需要在流程特定点获取结构化数据的对话式 agent 非常有用。

在 `writeSession` 中使用 `requestLLMStructured` 方法进行基于 agent 的交互：

```
val structuredResponse = llm.writeSession {
    requestLLMStructured<WeatherForecast>(
        examples = exampleForecasts,
        fixingParser = StructureFixingParser(
            model = OpenAIModels.Chat.GPT4o,
            retries = 3
        )
    )
}
```

`fixingParser` 参数为格式错误的 JSON 响应提供自动错误纠正。当解析失败时，它会使用辅助 LLM 智能修复响应，最多重试指定次数。

**StructureFixingParser 参数：**
- `model: LLModel` - 用于修复格式错误 JSON 输出的 LLM
- `retries: Int` - 最大修复尝试次数（默认值：3）
- `prompt` - 用于修复过程的可选自定义提示函数（默认为内置修复提示）

修复过程会迭代地将解析错误传递给辅助模型，该模型尝试在保留原始数据并进行最小更改的同时修正 JSON。

#### 与 agent 策略集成

你可以将结构化数据处理集成到 agent 策略中：

```
val agentStrategy = strategy<String, String>("weather-forecast") {
    val setup by nodeLLMRequest()

    val getStructuredForecast by node<Message.Assistant, String> { _ ->
        val structuredResponse = llm.writeSession {
            requestLLMStructured<WeatherForecast>(
                fixingParser = StructureFixingParser(
                    model = OpenAIModels.Chat.GPT4o,
                    retries = 3
                )
            )
        }

        """
        Response structure:
        $structuredResponse
        """.trimIndent()
    }

    edge(nodeStart forwardTo setup)
    edge(setup forwardTo getStructuredForecast)
    edge(getStructuredForecast forwardTo nodeFinish)
}
```

### 层面 3：节点层面

节点层面为 agent 工作流中的结构化输出提供了最高级别的抽象。使用 `nodeLLMRequestStructured` 创建处理结构化数据的可复用 agent 节点。

这会创建一个 agent 节点，它：
- 接受 `String` 输入（用户消息）
- 将消息追加到 LLM 提示中
- 向 LLM 请求结构化输出
- 返回 `Result<StructuredResponse<MyStruct>>`

#### 节点层面示例

```
val agentStrategy = strategy<Unit, String>("weather-forecast") {
    val setup by node<Unit, String> { _ ->
        "Please provide a weather forecast for Amsterdam"
    }

    // Create a structured output node using delegate syntax
    val getWeatherForecast by nodeLLMRequestStructured<WeatherForecast>(
        name = "forecast-node",
        examples = exampleForecasts,
        fixingParser = StructureFixingParser(
            model = OpenAIModels.Chat.GPT4o,
            retries = 3
        )
    )

    val processResult by node<Result<StructuredResponse<WeatherForecast>>, String> { result ->
        when {
            result.isSuccess -> {
                val forecast = result.getOrNull()?.data
                "Weather forecast: $forecast"
            }
            result.isFailure -> {
                "Failed to get structured forecast: ${result.exceptionOrNull()?.message}"
            }
            else -> "Unknown result state"
        }
    }

    edge(nodeStart forwardTo setup)
    edge(setup forwardTo getWeatherForecast)
    edge(getWeatherForecast forwardTo processResult)
    edge(processResult forwardTo nodeFinish)
}
```

#### 完整代码示例

以下是使用结构化输出 API 的完整示例：

```
// Note: Import statements are omitted for brevity
@Serializable
@SerialName("SimpleWeatherForecast")
@LLMDescription("Simple weather forecast for a location")
data class SimpleWeatherForecast(
    @property:LLMDescription("Location name")
    val location: String,
    @property:LLMDescription("Temperature in Celsius")
    val temperature: Int,
    @property:LLMDescription("Weather conditions (e.g., sunny, cloudy, rainy)")
    val conditions: String
)

val token = System.getenv("OPENAI_KEY") ?: error("Environment variable OPENAI_KEY is not set")

fun main(): Unit = runBlocking {
    // Create sample forecasts
    val exampleForecasts = listOf(
        SimpleWeatherForecast(
            location = "New York",
            temperature = 25,
            conditions = "Sunny"
        ),
        SimpleWeatherForecast(
            location = "London",
            temperature = 18,
            conditions = "Cloudy"
        )
    )

    // Generate JSON Schema
    val forecastStructure = JsonStructure.create<SimpleWeatherForecast>(
        schemaGenerator = BasicJsonSchemaGenerator.Default,
        examples = exampleForecasts
    )

    // Define the agent strategy
    val agentStrategy = strategy<String, String>("weather-forecast") {
        val setup by nodeLLMRequest()

        val getStructuredForecast by node<Message.Assistant, String> { _ ->
            val structuredResponse = llm.writeSession {
                requestLLMStructured<SimpleWeatherForecast>()
            }

            """
            Response structure:
            $structuredResponse
            """.trimIndent()
        }

        edge(nodeStart forwardTo setup)
        edge(setup forwardTo getStructuredForecast)
        edge(getStructuredForecast forwardTo nodeFinish)
    }

    // Configure and run the agent
    val agentConfig = AIAgentConfig(
        prompt = prompt("weather-forecast-prompt") {
            system(
                """
                You are a weather forecasting assistant.
                When asked for a weather forecast, provide a realistic but fictional forecast.
                """.trimIndent()
            )
        },
        model = OpenAIModels.Chat.GPT4o,
        maxAgentIterations = 5
    )

    val runner = AIAgent(
        promptExecutor = simpleOpenAIExecutor(token),
        toolRegistry = ToolRegistry.EMPTY,
        strategy = agentStrategy,
        agentConfig = agentConfig
    )

    runner.run("Get weather forecast for Paris")
}
```

## 高级用法

上述示例演示了简化 API，它会根据模型能力自动选择最佳结构化输出方法。
如果需要对结构化输出过程进行更多控制，你可以使用高级 API，手动创建 schema 并进行提供商特定的配置。

### 手动创建 schema 和配置

你可以使用 `JsonStructure.create` 显式创建 schema，并通过 `StructuredOutput` 类手动配置结构化输出行为，而不是依赖自动 schema 生成。

关键区别在于，你不再传递 `examples` 和 `fixingParser` 等简单参数，而是创建一个 `StructuredRequestConfig` 对象，它允许对以下方面进行细粒度控制：

- **Schema 生成**：选择特定的生成器（Standard、Basic 或提供商特定）
- **输出模式**：原生结构化输出支持 vs 手动提示
- **提供商映射**：为不同 LLM 提供商使用不同配置
- **回退策略**：当提供商特定配置不可用时的默认行为

```
// Create different schema structures with different generators
val genericStructure = JsonStructure.create<WeatherForecast>(
    schemaGenerator = StandardJsonSchemaGenerator,
    examples = exampleForecasts
)

val openAiStructure = JsonStructure.create<WeatherForecast>(
    schemaGenerator = OpenAIBasicJsonSchemaGenerator,
    examples = exampleForecasts
)

val anthropicStructure = JsonStructure.create<WeatherForecast>(
    schemaGenerator = AnthropicBasicJsonSchemaGenerator,
    examples = exampleForecasts
)

val promptExecutor = simpleOpenAIExecutor(System.getenv("OPENAI_KEY"))

// The advanced API uses StructuredRequestConfig instead of simple parameters
val structuredResponse = promptExecutor.executeStructured(
    prompt = prompt("structured-data") {
        system("You are a weather forecasting assistant.")
        user("What is the weather forecast for Amsterdam?")
    },
    model = OpenAIModels.Chat.GPT4oMini,
    config = StructuredRequestConfig(
        byProvider = mapOf(
            LLMProvider.OpenAI to StructuredRequest.Native(openAiStructure),
            LLMProvider.Anthropic to StructuredRequest.Native(anthropicStructure),
        ),
        default = StructuredRequest.Manual(genericStructure)
    ),
    fixingParser = StructureFixingParser(
        model = AnthropicModels.Haiku_4_5,
        retries = 2
    )
)
```

### Schema 生成器

根据你的需求，可以使用不同的 schema 生成器：

- **StandardJsonSchemaGenerator**：完整的 JSON Schema，支持多态、定义和递归引用
- **BasicJsonSchemaGenerator**：简化 schema，不支持多态，兼容更多模型
- **提供商特定生成器**：为特定 LLM 提供商（OpenAI、Anthropic、Google 等）优化的 schema

### 在所有层面中使用

高级配置在 API 的所有三个层面中一致工作。方法名称保持不变，只是参数从简单参数变为更高级的 `StructuredRequestConfig`：

- **提示执行器**：`executeStructured(prompt, model, config: StructuredRequestConfig<T>)`
- **Agent LLM 上下文**：`requestLLMStructured(config: StructuredRequestConfig<T>)`
- **节点层面**：`nodeLLMRequestStructured(config: StructuredRequestConfig<T>)`

对于大多数用例，推荐使用简化 API（仅使用 `examples` 和 `fixingParser` 参数），而高级 API 在需要时提供额外的控制。

## 最佳实践

1. **使用清晰的描述**：使用 `@LLMDescription` 注解提供清晰详细的描述，帮助 LLM 理解预期数据。
2. **提供示例**：包含有效数据结构的示例来指导 LLM。
3. **优雅地处理错误**：实现适当的错误处理，以应对 LLM 可能无法生成有效结构的情况。
4. **使用适当的 schema 类型**：根据你的需求和你所使用的 LLM 的能力，选择合适的 schema 格式和类型。
5. **使用不同模型进行测试**：不同的 LLM 遵循结构化格式的能力可能不同，因此如果可能，请使用多个模型进行测试。
6. **从简单开始**：从简单的结构开始，根据需要逐步增加复杂性。
7. **谨慎使用多态**：虽然 API 支持使用密封类实现多态，但请注意 LLM 可能更难正确处理它。
