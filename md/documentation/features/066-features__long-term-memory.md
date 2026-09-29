# 长期记忆

Beta

此特性属于 beta 模块（`1.3.0-beta`）。API 可能会在未来版本中发生变化。
详情请参阅[模块版本控制](../../module-versioning/)。

`LongTermMemory` 特性通过两组独立设置为 Koog AI agent 添加持久化记忆：
- **检索** — 使用来自记忆存储的相关上下文增强 LLM prompt（检索增强生成，Retrieval-Augmented Generation 或 RAG）
- **摄取** — 将对话消息持久化到记忆存储中，以便未来检索

## 快速开始

KotlinJava

```
val myStorage = InMemoryRecordStorage() // or your vector DB adapter

val agent = AIAgent(
    promptExecutor = executor,
    strategy = singleRunStrategy(),
    agentConfig = agentConfig,
    toolRegistry = ToolRegistry.EMPTY
) {
    install(LongTermMemory) {
        retrieval {
            storage = myStorage
            searchStrategy = SimilaritySearchStrategy(topK = 5)
        }
    }
}

agent.run("What did we discuss yesterday?")
```

```
InMemoryRecordStorage myStorage = new InMemoryRecordStorage();

AIAgent agent = AIAgent.builder()
    .promptExecutor(executor)
    .llmModel(OpenAIModels.Chat.GPT4o)
    .systemPrompt("You are a helpful assistant.")
    .install(LongTermMemory.Feature, config -> {
        config.retrieval(
            new LongTermMemory.RetrievalSettingsBuilder()
                .withStorage(myStorage)
                .withSearchStrategy(
                    SearchStrategy.builder().similarity().withTopK(5).build()
                )
                .build()
        );
    })
    .build();

Object result = agent.run("What did we discuss yesterday?");
```

## 仅检索（RAG）

当你已有预填充知识库时，可以使用检索而不使用摄取：

KotlinJava

```
install(LongTermMemory) {
    retrieval {
        storage = myVectorDbStorage
        namespace = "my-collection"  // optional: scope to a specific namespace/collection
        searchStrategy = SimilaritySearchStrategy(topK = 3, similarityThreshold = 0.7)
        promptAugmenter = SystemPromptAugmenter()
    }
}
```

```
var retrievalSettings = new LongTermMemory.RetrievalSettingsBuilder()
    .withStorage(myVectorDbStorage)
    .withSearchStrategy(
        SearchStrategy.builder().similarity().withTopK(3).withSimilarityThreshold(0.7).build()
    )
    .withPromptAugmenter(PromptAugmenter.builder().system().build())
    .build();
```

### Prompt 增强器

| 增强器 | 行为 |
| --- | --- |
| `SystemPromptAugmenter()` | 在 prompt 开头将上下文插入为系统消息（如果没有系统消息，则为空操作） |
| `UserPromptAugmenter()` | 将检索到的上下文作为额外文本部分追加到最后一条用户消息末尾（如果没有用户消息，则为空操作） |
| `PromptAugmenter { prompt, context -> ... }` | 通过 lambda 进行自定义增强 |

### 搜索查询提供者

默认情况下，检索流程使用最后一条用户消息作为搜索查询。你可以通过提供 `SearchQueryProvider` 来自定义此行为：

| 提供者 | 行为 |
| --- | --- |
| `LastUserMessageQueryProvider()` | 使用最后一条用户消息的内容（默认） |
| `SearchQueryProvider { prompt -> ... }` | 通过 lambda 自定义查询推导 |

KotlinJava

```
install(LongTermMemory) {
    retrieval {
        storage = myStorage
        searchQueryProvider = SearchQueryProvider { prompt ->
            // Combine the last two user messages as the search query
            prompt.messages
                .filter { it.role == Message.Role.User }
                .takeLast(2)
                .joinToString(" ") { it.content }
                .ifEmpty { null }
        }
    }
}
```

```
var retrievalSettings = new LongTermMemory.RetrievalSettingsBuilder()
    .withStorage(myStorage)
    .withSearchQueryProvider(prompt -> {
        var userMessages = prompt.getMessages().stream()
            .filter(m -> m.getRole() == Message.Role.User)
            .toList();
        if (userMessages.isEmpty()) return null;
        return userMessages.get(userMessages.size() - 1).getContent();
    })
    .build();
```

### 搜索策略

| 策略 | 行为 |
| --- | --- |
| `SimilaritySearchStrategy()` | 向量相似度语义搜索 — **默认** |
| `query -> new SimilaritySearchRequest(query, 20, 0, 0.0, null)` | 通过 lambda 自定义搜索 |

## 仅摄取

使用摄取而不使用检索，以随着时间构建记忆存储：

KotlinJava

```
install(LongTermMemory) {
    ingestion {
        storage = myVectorDbStorage
        namespace = "my-collection"  // optional: scope to a specific namespace/collection
        documentExtractor = MessagePassingDocumentExtractor(
            messageRolesToExtract = setOf(Message.Role.User, Message.Role.Assistant)
        )
    }
}
```

```
var ingestionSettings = new LongTermMemory.IngestionSettingsBuilder()
    .withStorage(myVectorDbStorage)
    .withDocumentExtractor(
        DocumentExtractor.builder()
            .filtering()
            .withExtractRoles(new HashSet<>(Arrays.asList(Message.Role.User, Message.Role.Assistant)))
            .build()
    )
    .build();
```

摄取会在 agent run 完成时运行一次：最终累积的会话 prompt/history 会作为单个批次传给已配置的 `documentExtractor`。

## 禁用自动行为

默认情况下，检索和摄取会自动运行（检索在每次 LLM 调用前运行；摄取在 agent 完成时运行一次）。你可以禁用自动行为，同时仍可在策略节点中访问已配置的存储和策略：

KotlinJava

```
install(LongTermMemory) {
    retrieval {
        storage = myStorage
        enableAutomaticRetrieval = false  // no automatic prompt augmentation
    }
    ingestion {
        storage = myStorage
        enableAutomaticIngestion = false  // no automatic message persistence
    }
}
```

```
config.retrieval(
    new LongTermMemory.RetrievalSettingsBuilder()
        .withStorage(myStorage)
        .withEnableAutomaticRetrieval(false)
        .build()
);
config.ingestion(
    new LongTermMemory.IngestionSettingsBuilder()
        .withStorage(myStorage)
        .withEnableAutomaticIngestion(false)
        .build()
);
```

这为你提供三种清晰模式：

1. **全自动**（默认）：安装特性并配置存储 — 检索和摄取会自动工作。
2. **仅手动**：设置 `enableAutomaticRetrieval = false` / `enableAutomaticIngestion = false`，并在图策略节点中使用存储和策略。
3. **混合**：将自动摄取与手动检索组合使用（或反之）。

## 从策略节点访问长期记忆

在策略节点内部使用 `withLongTermMemory { }` 来直接搜索或添加记录：

```
val myNode by node<String, Unit> {
    withLongTermMemory {
        // Manually add records
        val record = MemoryRecord(content = "important fact")
        ingestionStorage?.add(listOf(record), namespace = "my-namespace")

        // Manually search
        val request = SimilaritySearchRequest(queryText = input, limit = 5)
        val results = retrievalStorage?.search(request, namespace = "my-namespace")
    }
}
```

使用 `longTermMemory()` 直接获取特性实例：

```
val myNode by node<String, Unit> {
    val memory = longTermMemory()
    val storage = memory.ingestionStorage
}
```

## 自定义文档提取器

实现 `DocumentExtractor` 来控制消息在存储前如何转换：

```
val summarizingExtractor = DocumentExtractor { messages ->
    messages
        .filter { it.role == Message.Role.Assistant }
        .map { MemoryRecord(content = summarize(it.content)) }
}

install(LongTermMemory) {
    ingestion {
        storage = myStorage
        documentExtractor = summarizingExtractor
    }
}
```

## 实现自定义存储

实现 `SearchStorage` 和/或 `WriteStorage` 以连接到你的向量数据库：

```
class MyVectorDbStorage : SearchStorage<TextDocument, SearchRequest>, WriteStorage<TextDocument> {
    override suspend fun search(
        request: SearchRequest, namespace: String?
    ): List<SearchResult<TextDocument>> {
        // Query your vector DB
    }

    override suspend fun add(
        records: List<TextDocument>, namespace: String?
    ): List<String> {
        // Upsert into your vector DB and return the IDs of added records
    }
}
```

测试时，可以使用内置的 `InMemoryRecordStorage`，它会将记录保存在内存中。它同时支持 `KeywordSearchRequest`（实现为不区分大小写的子字符串匹配）和 `SimilaritySearchRequest`（实现为基于不区分大小写词集的 Jaccard 系数）；不使用向量嵌入。
