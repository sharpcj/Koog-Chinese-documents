# LLM 响应缓存

对于使用 prompt executor 运行的重复请求，
你可以在 Kotlin 和 Java 中缓存 LLM 响应，以优化性能并降低成本。
在 Koog 中，所有 prompt executors 都可以通过 `CachedPromptExecutor` 使用缓存，
它是围绕 `PromptExecutor` 的包装器，会添加缓存功能。
它允许你存储之前执行过的 prompts 的响应，并在再次运行相同 prompts 时检索这些响应。

要在 Kotlin 或 Java 中创建缓存的 prompt executor，请执行以下操作：

1. 创建一个你想为其缓存响应的 prompt executor。
2. 通过提供所需缓存和你创建的 prompt executor，创建 `CachedPromptExecutor` 实例。
3. 使用所需 prompt 和 model 运行创建的 `CachedPromptExecutor`。

下面是一个示例：

KotlinJava

```
// Create a prompt executor
val client = OpenAILLMClient(System.getenv("OPENAI_API_KEY"))
val promptExecutor = MultiLLMPromptExecutor(client)

// Create a cached prompt executor
val cachedExecutor = CachedPromptExecutor(
    cache = FilePromptCache(Path("path/to/your/cache/directory")),
    nested = promptExecutor
)

// Run cached prompt executor for the first time
// This will perform an actual LLM request
val firstTime = measureTimeMillis {
    val firstResponse = cachedExecutor.execute(prompt, OpenAIModels.Chat.GPT4o)
    val text = firstResponse.parts.filterIsInstance<MessagePart.Text>().joinToString("\n") { it.text }
    println("First response: $text")
}
println("First execution took: ${firstTime}ms")

// Run cached prompt executor for the second time
// This will return the result immediately from the cache
val secondTime = measureTimeMillis {
    val secondResponse = cachedExecutor.execute(prompt, OpenAIModels.Chat.GPT4o)
    val text = secondResponse.parts.filterIsInstance<MessagePart.Text>().joinToString("\n") { it.text }
    println("Second response: $text")
}
println("Second execution took: ${secondTime}ms")
```

```
// Create a prompt
Prompt prompt = Prompt.builder("test")
        .user("Hello")
        .build();

// Create a prompt executor
OpenAILLMClient client = openAIClient(System.getenv("OPENAI_API_KEY"));
MultiLLMPromptExecutor promptExecutor = new MultiLLMPromptExecutor(client);

// Create a cached prompt executor
FilePromptCache cache = new FilePromptCache(Path.of("path/to/your/cache/directory"), null);
CachedPromptExecutor cachedExecutor = new CachedPromptExecutor(cache, promptExecutor, Clock.System.INSTANCE);

// Run cached prompt executor for the first time
// This will perform an actual LLM request
long start1 = System.nanoTime();
List<Message.Response> firstResponse = cachedExecutor.execute(prompt, OllamaModels.Meta.LLAMA_3_2);
long firstTimeMs = (System.nanoTime() - start1) / 1_000_000L;
System.out.println("First response: " + firstResponse.getFirst().getContent());
System.out.println("First execution took: " + firstTimeMs + "ms");

// Run cached prompt executor for the second time
// This will return the result immediately from the cache
long start2 = System.nanoTime();
List<Message.Response> secondResponse = cachedExecutor.execute(prompt, OllamaModels.Meta.LLAMA_3_2);
long secondTimeMs = (System.nanoTime() - start2) / 1_000_000L;
System.out.println("Second response: " + secondResponse.getFirst().getContent());
System.out.println("Second execution took: " + secondTimeMs + "ms");
```

该示例会产生以下输出：

```
First response: Hello! It seems like we're starting a new conversation. What can I help you with today?
First execution took: 48ms
Second response: Hello! It seems like we're starting a new conversation. What can I help you with today?
Second execution took: 1ms
```

第二次响应从缓存中检索，仅耗时 1ms。

注意

- 如果你在 Kotlin 中调用 `executeStreaming()` 或在 Java 中调用 `executeStreamingWithPublisher()` 并使用缓存的 prompt executor，它会以单个 chunk 生成响应。
- 如果你在 Kotlin 或 Java 中使用缓存的 prompt executor 调用 `moderate()`，它会将请求转发给嵌套的 prompt executor，并且不使用缓存。
- Kotlin 和 Java 都不支持缓存多选项响应（`executeMultipleChoices()`）。