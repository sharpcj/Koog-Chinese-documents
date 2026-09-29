# 检索增强生成（RAG）

Beta

此功能属于 beta 模块（`1.3.0-beta`）。API 可能会在未来版本中发生变化。
详情请参阅[模块版本管理](../module-versioning/)。

Koog 提供了检索增强生成（RAG）的构建块：嵌入文本、存储已嵌入的文档，以及为查询检索最相关的结果。

本页重点介绍当前 `rag` 模块中可用的内容以及如何使用它。

## Koog 当前提供的能力

当前 RAG 支持拆分为两个模块：

- `rag-base`：用于检索、存储、搜索请求、过滤以及文件/文档提供者的通用抽象
- `rag-vector`：将文档嵌入与向量存储结合起来的本地实现

## 使用 EmbeddingStorage 嵌入和检索文档

最完整的开箱即用 RAG 流程使用 `rag-vector` 模块中的 `EmbeddingStorage`。它将 `DocumentEmbedder`（把文档转换为向量）与 `VectorStorageBackend`（持久化向量）结合在一起。

步骤如下：

1. 创建一个由嵌入模型（Ollama 或 OpenAI）支持的 `Embedder`。
2. 创建一个读取文件内容并委托给 embedder 的 `JVMTextDocumentEmbedder`。
3. 使用内存或基于文件的后端创建 `EmbeddingStorage`。
4. 使用 `add()` 添加文档。
5. 使用 `search(SimilaritySearchRequest(...))` 搜索。

KotlinJava

```
// 1. Create an embedder backed by a local Ollama model
val embedder = LLMEmbedder(
    client = OllamaClient(),
    model = OllamaModels.Embeddings.NOMIC_EMBED_TEXT
)

// 2. Create a JVM document embedder that reads files and embeds their text
val documentEmbedder = JVMTextDocumentEmbedder(embedder)

// 3. Create an EmbeddingStorage with an in-memory backend
val storage = EmbeddingStorage(
    embedder = documentEmbedder,
    storage = InMemoryVectorStorageBackend()
)

// 4. Add documents to the storage
storage.add(
    listOf(
        Path.of("./docs/faq.txt"),
        Path.of("./docs/pricing.txt"),
        Path.of("./docs/getting-started.txt")
    )
)

// 5. Search for the most relevant documents
val results = storage.search(
    SimilaritySearchRequest(
        queryText = "How do I reset my password?",
        limit = 3,
        minScore = 0.5
    )
)

results.forEach { result ->
    println("${result.document} (score: ${result.score.value})")
}
```

```

```

## 将相关性搜索作为 agent 工具提供（agentic RAG）

你可以把 RAG 存储暴露为 agent 按需调用的工具，而不是预先把所有检索到的文档注入 prompt。这样 agent 可以自行控制何时搜索以及搜索什么。

下面的示例将一个 `SearchStorage`（`EmbeddingStorage` 实现的基础搜索接口）包装在使用 `@Tool` 和 `@LLMDescription` 注解的函数中，然后将它注册到 `ToolRegistry` 以供 agent 使用。

KotlinJava

```
// Define a tool that searches the RAG storage
@Tool
@LLMDescription("Search the knowledge base for documents relevant to a query. Returns the content of the most relevant documents.")
suspend fun searchKnowledgeBase(
    @LLMDescription("The search query describing what information you need")
    query: String,
    @LLMDescription("Maximum number of documents to return")
    count: Int
): String {
    val results = ragStorage.search(
        SimilaritySearchRequest(
            queryText = query,
            limit = count,
            minScore = 0.5
        )
    )

    if (results.isEmpty()) {
        return "No relevant documents found for: $query"
    }

    val response = StringBuilder("Found ${results.size} relevant documents:\n\n")
    results.forEachIndexed { index, result ->
        val content = Files.readString(result.document)
        response.append("Document ${index + 1}: ${result.document.fileName}")
        response.append(" (score: ${"%.2f".format(result.score.value)})\n")
        response.append("Content: $content\n\n")
    }
    return response.toString()
}

fun main() {
    runBlocking {
        // Register the search tool and create an agent
        val tools = ToolRegistry {
            tool(::searchKnowledgeBase.asTool())
        }

        val agent = AIAgent(
            toolRegistry = tools,
            promptExecutor = simpleOpenAIExecutor(apiKey),
            llmModel = OpenAIModels.Chat.GPT4o
        )

        val response = agent.run("What is your refund policy?")
        println("Agent response: $response")
    }
}
```

```

```

通过这种方法，agent 会根据用户查询自行决定何时调用搜索工具。当 agent 处理多样化请求且只有部分请求需要知识库查找时，这会很有用。

## 可用实现

### 向量存储后端

- `InMemoryVectorStorageBackend`：将向量存储在内存中；适合测试和原型
- `FileVectorStorageBackend`：将向量持久化到磁盘，以便重启后继续使用
- `JVMFileVectorStorageBackend`：使用 `java.nio.file.Path` 的 JVM 专用文件后端

### 文档嵌入器

- `TextDocumentEmbedder`：按文档和路径类型参数化的通用文档转文本嵌入器
- `JVMTextDocumentEmbedder`：从 `java.nio.file.Path` 读取文件的 JVM 专用嵌入器

### 组合式存储实现

- `EmbeddingStorage`：将任意 `DocumentEmbedder` 与任意 `VectorStorageBackend` 组合起来
- `InMemoryDocumentEmbeddingStorage`：`EmbeddingStorage` + `InMemoryVectorStorageBackend` 的便捷快捷方式
- `FileDocumentEmbeddingStorage`：`EmbeddingStorage` + `FileVectorStorageBackend` 的便捷快捷方式
- `JVMFileDocumentEmbeddingStorage`：基于 JVM 文件的嵌入存储
- `TextFileDocumentEmbeddingStorage`：用于文本文档的文件存储
- `JVMFileEmbeddingStorage`：用于文本文档的 JVM 文件存储

## 当前限制

内置流程适用于本地和参考实现，但它还不是完整的生产级 RAG 平台。

重要限制：

- 内置实现仅支持相似性搜索
- `rag` 模块中没有内置分块流水线
- 富元数据的生产记录建模仍然有限
- 当前 `rag` 模块未提供生产向量数据库集成（Pinecone、Weaviate、pgvector、Milvus）

如果你正在构建自定义后端，请从 `rag-base` 抽象开始，并实现你自己的存储适配器。

## 如何选择起点

如果满足以下情况，请使用 `rag-vector`：

- 你想构建本地 RAG 原型
- 你想使用简单的参考实现
- 你想在 Koog 内部试验嵌入和检索流程

如果满足以下情况，请使用 `rag-base`：

- 你正在构建自己的存储后端
- 你想集成外部向量数据库
- 你想在其他 Koog 模块中复用这些抽象

## 另请参阅

- [Embeddings](../embeddings/)
