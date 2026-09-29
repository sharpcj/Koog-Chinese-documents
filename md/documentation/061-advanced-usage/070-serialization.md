# 序列化

## 引言

Koog 使用一个轻量、与具体库无关的序列化层，在 JSON 与工具参数和结果之间进行转换。
该层位于 agent 运行时和底层序列化库之间，因此你可以替换序列化库，而无需修改任何工具或 agent 代码。

除工具外，序列化层也会被 **Persistence** 等 agent 功能用于序列化和反序列化节点输入与输出。

默认情况下，Koog 使用 `KotlinxSerializer`（基于 kotlinx-serialization）。
在 JVM 上，你也可以切换到 `JacksonSerializer`（基于 jackson-databind）。

## `JSONSerializer` 接口

`JSONSerializer` 是位于 `serialization-core` 中的核心抽象。
该接口有四个主要方法（编码/解码为字符串和 `JSONElement`），以及两个用于在 `JSONElement` 和字符串之间转换的便捷方法：

- `encodeToString` / `decodeFromString` — 将类型化值序列化为 JSON 字符串，或从 JSON 字符串反序列化。
- `encodeToJSONElement` / `decodeFromJSONElement` — 将类型化值序列化为 `JSONElement` 树，或从 `JSONElement` 树反序列化。
- `encodeJSONElementToString` / `decodeJSONElementFromString` — 在 `JSONElement` 和其字符串形式之间转换。

下面的示例展示所有关键操作：

KotlinJava

```
@Serializable
data class User(val name: String, val age: Int)

val serializer: JSONSerializer = KotlinxSerializer()

// Encode a data class to a JSON string
val json: String = serializer.encodeToString(User("Alice", 30), typeToken<User>())

// Decode a JSON string back to a data class
val user: User = serializer.decodeFromString(json, typeToken<User>())

// Encode to a JSONElement tree
val element: JSONElement = serializer.encodeToJSONElement(user, typeToken<User>())

// Decode from a JSONElement tree
val userFromElement: User = serializer.decodeFromJSONElement(element, typeToken<User>())

// Convert between JSONElement and a raw JSON string
val jsonString = """{"key": "value"}"""
val jsonElement: JSONElement = serializer.decodeJSONElementFromString(jsonString)
val backToString: String = serializer.encodeJSONElementToString(jsonElement)
```

```
// Jackson-serializable class
record User(
    @JsonProperty("name") String name,
    @JsonProperty("age") int age
) {}

var serializer = new JacksonSerializer();

// Encode a data class to a JSON string
String json = serializer.encodeToString(new User("Alice", 30), TypeToken.of(User.class));

// Decode a JSON string back to a data class
User user = serializer.decodeFromString(json, TypeToken.of(User.class));

// Encode to a JSONElement tree
JSONElement element = serializer.encodeToJSONElement(user, TypeToken.of(User.class));

// Decode from a JSONElement tree
User userFromElement = serializer.decodeFromJSONElement(element, TypeToken.of(User.class));

// Convert between JSONElement and a raw JSON string
String jsonString = "{\"key\": \"value\"}";
JSONElement jsonElement = serializer.decodeJSONElementFromString(jsonString);
String backToString = serializer.encodeJSONElementToString(jsonElement);
```

## 类型令牌

`TypeToken` 是 Koog 在运行时传递类型信息的方式。

KotlinJava

```
data class MyClass(val value: String)

// Inline reified — preferred in Kotlin
val tokenReified = typeToken<MyClass>()

// From a KClass (when no reified type parameter is available)
val tokenKClass = typeToken(MyClass::class)

// Generic type — preserves type arguments at runtime
val tokenGeneric = typeToken<List<String>>()
```

```
record MyClass(
    String value
) {}

// Simple class
TypeToken tokenClass = TypeToken.of(MyClass.class);

// Generic type — use TypeCapture to preserve type arguments
TypeToken tokenGeneric = TypeToken.of(new TypeCapture<List<String>>() {});
```

## `JSONElement` — 与库无关的 JSON 树

`JSONElement` 是 JSON 数据的中立中间表示。
它的存在使 serializers、tools 和 agent 内部不依赖某个特定库中的具体 JSON 类型。

### 层级结构

```
JSONElement
├── JSONObject   – key-value pairs  (entries: Map<String, JSONElement>)
├── JSONArray    – ordered list      (elements: List<JSONElement>)
└── JSONPrimitive
    ├── JSONLiteral  – string, number, or boolean
    └── JSONNull     – JSON null singleton
```

### 与库类型之间的转换

每个序列化集成都提供扩展函数，允许你在 `JSONElement` 和该库自己的动态 JSON 类型之间转换。当你已经有 `JsonElement`、`JsonNode` 等对象，并且想将其传给 Koog（或反向转换）而不经过完整的编码/解码周期时，这会很有用。
下面为每个受支持的库提供示例。

### 构建和读取元素

KotlinJava

```
val obj = JSONObject(
    mapOf(
        "name" to JSONPrimitive("Alice"),
        "age" to JSONPrimitive(30),
        "active" to JSONPrimitive(true),
    )
)

val arr = JSONArray(listOf(JSONPrimitive(1), JSONPrimitive(2), JSONPrimitive(3)))

// Reading values from an object
val nameContent: String = (obj.entries["name"] as JSONPrimitive).content  // "Alice"
val age: Int? = (obj.entries["age"] as JSONPrimitive).intOrNull // 30
```

```
JSONObject obj = new JSONObject(
    Map.of(
        "name", JSONPrimitive.of("Alice"),
        "age", JSONPrimitive.of(30),
        "active", JSONPrimitive.of(true)
    )
);

JSONArray arr = new JSONArray(List.of(JSONPrimitive.of(1), JSONPrimitive.of(2), JSONPrimitive.of(3)));

// Reading values from an object
String nameContent = ((JSONPrimitive) obj.getEntries().get("name")).getContent();  // "Alice"
Integer age = ((JSONPrimitive) obj.getEntries().get("age")).getIntOrNull(); // 30
```

## 支持的 serializers

### `KotlinxSerializer`（默认）

- **Module**：`ai.koog:serialization-core`（随 `ai.koog:agents-core` 传递包含）
- **Backed by**：kotlinx-serialization

Kotlin

```
// Default instance — uses Json.Default
val defaultSerializer = KotlinxSerializer()

// Custom Json configuration
val customSerializer = KotlinxSerializer(
    json = Json {
        ignoreUnknownKeys = true
        prettyPrint = true
    }
)
```

你也可以在 Koog 的 `JSONElement` 和 kotlinx-serialization 的 `JsonElement` 之间转换。

Kotlin

```
val koogJson: JSONElement = JSONObject(
    mapOf(
        "key" to JSONPrimitive("value")
    )
)

// Convert to kotlinx-serialization dynamic JSON instance
val kotlinxJson: JsonElement = koogJson.toKotlinxJsonElement()

// Convert to Koog dynamic JSON instance
val koogJsonConverted: JSONElement = kotlinxJson.toKoogJSONElement()
```

### `JacksonSerializer`（仅 JVM）

- **Module**：`ai.koog:serialization-jackson`（独立依赖）
- **Backed by**：jackson-databind

将依赖添加到你的 `build.gradle.kts`：

```
dependencies {
    implementation("ai.koog:serialization-jackson:<version>")
}
```

然后创建 serializer：

KotlinJava

```
// Default instance — uses a fresh ObjectMapper with JSONElementModule pre-registered
val defaultSerializer = JacksonSerializer()

// Custom ObjectMapper configuration
val customSerializer = JacksonSerializer(
    objectMapper = ObjectMapper().apply {
        configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false)
    }
)
```

```
// Default instance — uses a fresh ObjectMapper with JSONElementModule pre-registered
var defaultSerializer = new JacksonSerializer();

// Custom ObjectMapper configuration
ObjectMapper objectMapper = new ObjectMapper();
objectMapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
var customSerializer = new JacksonSerializer(objectMapper);
```

Note

`JacksonSerializer` 会自动在其使用的 `ObjectMapper` 上注册 `JSONElementModule`，以便正确序列化/反序列化 `JSONElement` 类型。

你也可以在 Koog 的 `JSONElement` 和 Jackson 的 `JsonNode` 之间转换。

KotlinJava

```
val koogJson: JSONElement = JSONObject(
    mapOf(
        "key" to JSONPrimitive("value")
    )
)

// Convert to Jackson dynamic JSON instance
val jacksonJson: JsonNode = koogJson.toJacksonJsonNode()

// Convert to Koog dynamic JSON instance
val koogJsonConverted: JSONElement = jacksonJson.toKoogJSONElement()
```

```
JSONElement koogJson = new JSONObject(
    Map.of(
        "key", JSONPrimitive.of("value")
    )
);

// Convert to Jackson dynamic JSON instance
JsonNode jacksonJson = JacksonJSONElementMappers.toJacksonJsonNode(koogJson);

// Convert to Koog dynamic JSON instance
JSONElement koogJsonConverted = JacksonJSONElementMappers.toKoogJSONElement(jacksonJson);
```

## 在 `AIAgentConfig` 中配置 serializer

KotlinJava

构造 `AIAgentConfig` 时传入 `serializer` 参数。
如果省略它，将使用 `KotlinxSerializer`。

```
val agentConfig = AIAgentConfig(
    prompt = prompt("assistant") {
        system("You are a helpful assistant.")
    },
    model = OpenAIModels.Chat.GPT4o,
    maxAgentIterations = 10,
    serializer = JacksonSerializer()
)
```

构造 `AIAgentConfig` 时传入 `serializer` 参数。
如果省略它，将使用 `JacksonSerializer`。

```
var agentConfig = AIAgentConfig.builder()
    .model(OpenAIModels.Chat.GPT4o)
    .prompt(
        Prompt.builder("assistant")
            .system("You are a helpful assistant")
            .build()
    )
    .maxAgentIterations(10)
    .serializer(new JacksonSerializer())
    .build();
```

## 工具如何与 serializer 交互

agent 运行时会自动调用每个 `Tool` 实例上的以下方法。
在正常使用中，你不需要自行调用它们。

- **`decodeArgs(rawArgs, serializer)`**（JSON → TArgs）— 将来自 LLM 的原始 JSON 参数反序列化为工具的类型化 args 类。
- **`encodeArgs(args, serializer)`**（TArgs → JSON）— 将类型化 args 序列化回 JSON（由某些 agent 功能使用）。
- **`decodeResult(rawResult, serializer)`**（JSON → TResult）— 反序列化已存储的 JSON 结果。
- **`encodeResult(result, serializer)`**（TResult → JSON）— 将工具结果序列化为 JSON。
- **`encodeResultToString(result, serializer)`**（TResult → String）— 将工具结果序列化为发送给 LLM 的字符串。
  默认委托给 `encodeResult`。可以重写以自定义面向 LLM 的结果格式。

这些方法在 `Tool` 上是 `open` 的，因此如果你需要为特定工具自定义序列化行为，可以重写它们。

## 功能如何使用 serializer

序列化层并不限于工具——某些 agent 功能也依赖它。

例如，**Persistence** 使用 `AIAgentConfig` 中配置的 `JSONSerializer`，在创建检查点和恢复 agent 状态时序列化和反序列化节点输入与输出。这意味着流经持久化节点的任何类型都必须能被配置的 `JSONSerializer` 序列化。

有关检查点创建和恢复的详细信息，请参阅 [Agent Persistence](../features/agent-persistence/)。
