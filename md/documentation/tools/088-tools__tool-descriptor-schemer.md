# ToolDescriptorSchemer

`ToolDescriptorSchemer` 是一个扩展点，用于将 `ToolDescriptor` 转换为与特定 LLM 提供方兼容的 JSON Schema 对象。它可以在 Kotlin 和 Java 中实现。

要点：

- 位置：`ai.koog.agents.core.tools.serialization.ToolDescriptorSchemer`
- 契约：单个函数 `scheme(toolDescriptor: ToolDescriptor): JsonObject`，或 `generate(ToolDescriptor toolDescriptor): JsonObject`（Java）
- 已提供的实现：
- `OpenAICompatibleToolDescriptorSchemer` — 生成与 OpenAI 风格的函数/工具定义兼容的 schema。
- `OllamaToolDescriptorSchemer` — 生成与 Ollama 工具 JSON 兼容的 schema。

```
// Interface
interface ToolDescriptorSchemaGenerator {
fun generate(toolDescriptor: ToolDescriptor): JsonObject
}
```

## 为什么使用它

如果你想在 Kotlin 或 Java 中为现有或新的 LLM 提供方提供自定义 scheme，请实现此接口，将 Koog 的 `ToolDescriptor` 转换为预期的 JSON Schema 格式。

## 实现示例

下面是在 Kotlin 和 Java 中的最小自定义实现，它只渲染一部分参数类型，用于说明如何接入 SPI。实际实现应覆盖所有 `ToolParameterType`（String、Integer、Float、Boolean、Null、Enum、List、Object、AnyOf）。

KotlinJava

```
class MinimalSchemer : ToolDescriptorSchemaGenerator {
    override fun generate(toolDescriptor: ToolDescriptor): JsonObject = buildJsonObject {
        put("type", "object")
        putJsonObject("properties") {
            (toolDescriptor.requiredParameters + toolDescriptor.optionalParameters).forEach { p ->
                put(p.name, buildJsonObject {
                    put("description", p.description)
                    when (val t = p.type) {
                        ToolParameterType.String -> put("type", "string")
                        ToolParameterType.Integer -> put("type", "integer")
                        is ToolParameterType.Enum -> {
                            put("type", "string")
                            putJsonArray("enum") { t.entries.forEach { add(JsonPrimitive(it)) } }
                        }
                        else -> put("type", "string") // fallback for brevity
                    }
                })
            }
        }
        putJsonArray("required") { toolDescriptor.requiredParameters.forEach { add(JsonPrimitive(it.name)) } }
    }
}
```

```
public static class MinimalSchemer extends OpenAICompatibleToolDescriptorSchemaGenerator {
    @Override
    public JsonObject generate(ToolDescriptor toolDescriptor) {
        Map<String, JsonElement> root = new LinkedHashMap<>();
        root.put("type", JsonPrimitive("object"));

        // properties
        Map<String, JsonElement> props = new LinkedHashMap<>();
        for (ToolParameterDescriptor p : concat(toolDescriptor.getRequiredParameters(), toolDescriptor.getOptionalParameters())) {
            Map<String, JsonElement> prop = new LinkedHashMap<>();
            prop.put("description", JsonPrimitive(p.getDescription()));

            ToolParameterType t = p.getType();
            if (t == ToolParameterType.String.INSTANCE) {
                prop.put("type", JsonPrimitive("string"));
            } else if (t == ToolParameterType.Integer.INSTANCE) {
                prop.put("type", JsonPrimitive("integer"));
            } else if (t instanceof ToolParameterType.Enum) {
                prop.put("type", JsonPrimitive("string"));
                String[] entries = ((ToolParameterType.Enum) t).getEntries();
                List<JsonElement> enumVals = new ArrayList<>();
                for (String e : entries) enumVals.add(JsonPrimitive(e));
                prop.put("enum", new JsonArray(enumVals));
            } else {
                prop.put("type", JsonPrimitive("string")); // fallback for brevity
            }

            props.put(p.getName(), new JsonObject(prop));
        }
        root.put("properties", new JsonObject(props));

        // required array
        List<JsonElement> required = new ArrayList<>();
        for (ToolParameterDescriptor p : toolDescriptor.getRequiredParameters()) {
            required.add(JsonPrimitive(p.getName()));
        }
        root.put("required", new JsonArray(required));

        return new JsonObject(root);
    }

    private static List<ToolParameterDescriptor> concat(List<ToolParameterDescriptor> a, List<ToolParameterDescriptor> b) {
        List<ToolParameterDescriptor> res = new ArrayList<>(a.size() + b.size());
        res.addAll(a);
        res.addAll(b);
        return res;
    }
}
```

## 与客户端一起使用

通常，你不需要直接调用 schemer。Koog 客户端接受 `ToolDescriptor` 对象列表，并在为提供方序列化请求时在内部应用正确的 schemer。

下面的示例定义了一个简单工具，并将它传递给 OpenAI 客户端。客户端会在底层使用 `OpenAICompatibleToolDescriptorSchemer` 构建 JSON schema。

KotlinJava

```
val client = OpenAILLMClient(apiKey = System.getenv("OPENAI_API_KEY"), toolsConverter = MinimalSchemer())

val getUserTool = ToolDescriptor(
    name = "get_user",
    description = "Returns user profile by id",
    requiredParameters = listOf(
        ToolParameterDescriptor(
            name = "id",
            description = "User id",
            type = ToolParameterType.String
        )
    )
)

val prompt = Prompt.build(id = "p1") { user("Hello") }
val responses = runBlocking {
    client.execute(
        prompt = prompt,
        model = OpenAIModels.Chat.GPT4o,
        tools = listOf(getUserTool)
    )
}
```

```
// Custom schemer extending the OpenAI-compatible one is Kotlin-only in the docs; for Java example we reuse MinimalSchemer from above.
OpenAILLMClient client = openAIClient(System.getenv("OPENAI_API_KEY"), new OpenAIClientSettings(), null, null, new OpenAICompatibleToolDescriptorSchemaGenerator());

ToolDescriptor getUserTool = new ToolDescriptor(
    "get_user",
    "Returns user profile by id",
    Collections.singletonList(new ToolParameterDescriptor(
        "id",
        "User id",
        ToolParameterType.String.INSTANCE
    )),
    Collections.emptyList()
);

Prompt prompt = Prompt.builder("p1")
    .user("Hello")
    .build();

List<Message.Response> responses = client.execute(prompt, OpenAIModels.Chat.GPT4o, java.util.List.of(getUserTool));
```

如果你需要直接访问生成的 schema（用于调试或自定义传输），可以实例化特定于提供方的 schemer，并自行序列化 JSON：

KotlinJava

```
val json = Json { prettyPrint = true }
val schema = OpenAICompatibleToolDescriptorSchemaGenerator().generate(getUserTool())
```

```

```