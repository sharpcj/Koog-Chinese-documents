# ToolDescriptorSchemer

`ToolDescriptorSchemer` is an extension point that converts a `ToolDescriptor` into a JSON Schema object compatible with specific LLM providers. It can be implemented in both Kotlin and Java.

Key points:

- Location: `ai.koog.agents.core.tools.serialization.ToolDescriptorSchemer`
- Contract: a single function `scheme(toolDescriptor: ToolDescriptor): JsonObject` or `generate(ToolDescriptor toolDescriptor): JsonObject` (Java)
- Implementations provided:
- `OpenAICompatibleToolDescriptorSchemer` — generates schemas compatible with OpenAI‑style function/tool definitions.
- `OllamaToolDescriptorSchemer` — generates schemas compatible with Ollama tool JSON.

```
// Interface
interface ToolDescriptorSchemaGenerator {
fun generate(toolDescriptor: ToolDescriptor): JsonObject
}
```

## Why use it

If you want to provide a custom scheme for existing or new LLM providers in Kotlin or Java, implement this interface to convert Koog’s `ToolDescriptor` into the expected JSON Schema format.

## Implementation example

Below is a minimal custom implementation in both Kotlin and Java that renders only a subset of parameter types to illustrate how to plug into the SPI. Real implementations should cover all `ToolParameterType`s (String, Integer, Float, Boolean, Null, Enum, List, Object, AnyOf).

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

## Using with a client

Typically, you do not need to call a schemer directly. Koog clients accept a list of `ToolDescriptor` objects and apply the correct schemer internally when serializing requests for the provider.

The example below defines a simple tool and passes it to the OpenAI client. The client will use `OpenAICompatibleToolDescriptorSchemer` under the hood to build the JSON schema.

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

If you need direct access to the produced schema (for debugging or for a custom transport), you can instantiate the provider‑specific schemer and serialize the JSON yourself:

KotlinJava

```
val json = Json { prettyPrint = true }
val schema = OpenAICompatibleToolDescriptorSchemaGenerator().generate(getUserTool())
```

```

```
