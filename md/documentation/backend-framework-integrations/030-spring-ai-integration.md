# Spring AI 集成

Beta

此功能属于 beta 模块（`1.3.0-beta`）。API 可能会在未来版本中发生变化。
详情请参阅[模块版本管理](../module-versioning/)。

Koog 提供 Spring AI 集成 starters，用于在 Spring AI 的抽象与 Koog agent 框架之间建立桥接。
如果你已经使用 Spring AI 进行模型访问、memory 或向量存储，这些 starters 允许你在现有 Spring AI 配置之上接入 Koog，
而无需替换现有配置。

## 它与 `koog-spring-boot-starter` 有何不同

|  | `koog-spring-boot-starter` | `koog-spring-ai` starters |
| --- | --- | --- |
| **LLM transport** | Koog 自己的 HTTP clients | 委托给 Spring AI beans，例如 `ChatModel` 和 `EmbeddingModel` |
| **Configuration** | 每个 provider 使用 `ai.koog.*` properties | 使用由 Spring AI starters 管理的标准 `spring.ai.*` properties，加上 `koog.spring.ai.*` adapter properties |
| **When to use** | 你希望 Koog 直接管理模型连接 | 你已经使用 Spring AI，并希望在其之上使用 Koog agents、memory 或 RAG |

两种方法彼此独立。
关于直接 Koog starter 方法，请参阅 [Spring Boot Integration](../spring-boot/)。

## 可用 Starters

| Module | Purpose |
| --- | --- |
| `koog-spring-ai-starter-model-chat` | 将 Spring AI `ChatModel` 以及可选的 `ModerationModel` 适配为 Koog `LLMClient` 和 `PromptExecutor` |
| `koog-spring-ai-starter-model-embedding` | 将 Spring AI `EmbeddingModel` 适配为 Koog `LLMEmbeddingProvider` |
| `koog-spring-ai-starter-chat-memory` | 将 Spring AI `ChatMemoryRepository` 适配为 Koog `ChatHistoryProvider` |
| `koog-spring-ai-starter-vector-store` | 将 Spring AI `VectorStore` 适配为 Koog `KoogVectorStore`，用于摄取、搜索和删除 |

每个 starter 都是独立的 Spring Boot starter，拥有自己的自动配置和配置属性。
你可以在同一应用中使用一个 starter，也可以组合多个 starter。

## Dispatcher Types

所有四个 starters 都支持相同的 dispatcher 配置模式：

- **`AUTO`**（默认）：如果可用，则使用 Spring 管理的 `AsyncTaskExecutor`，否则回退到 `Dispatchers.IO`。
- **`IO`**：始终使用 `Dispatchers.IO`。
- **`dispatcher.parallelism`**：当大于 `0` 且 `type=IO` 时，使用 `Dispatchers.IO.limitedParallelism(parallelism)`。

`AUTO` 通常是最简单的选择，尤其是在使用 Spring Boot virtual threads 时。

## Chat Model Starter

### 概述

`koog-spring-ai-starter-model-chat` starter 在 Spring AI 的 chat model 抽象与 Koog agent 框架之间建立桥接。
它会自动配置：

- 一个委托给 Spring AI `ChatModel` 的 Koog `LLMClient`（`SpringAiLLMClient`）
- 一个由所有可用 `LLMClient` beans 组装而成的 `PromptExecutor`（`MultiLLMPromptExecutor`）

工具始终由 Koog agent 框架执行。Spring AI 只接收工具定义和 schemas。

### 添加依赖

将该依赖与任意 Spring AI chat model starter 一起添加：

Gradle (Kotlin DSL)Maven

```
dependencies {
    implementation("ai.koog:koog-agents-jvm:$koogVersion")
    implementation("ai.koog:koog-spring-ai-starter-model-chat:$koogVersion")
    implementation("org.springframework.ai:spring-ai-starter-model-openai")
}
```

```
<dependencies>
    <dependency>
        <groupId>ai.koog</groupId>
        <artifactId>koog-agents-jvm</artifactId>
        <version>${koog.version}</version>
    </dependency>
    <dependency>
        <groupId>ai.koog</groupId>
        <artifactId>koog-spring-ai-starter-model-chat</artifactId>
        <version>${koog.version}</version>
    </dependency>
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-starter-model-openai</artifactId>
    </dependency>
</dependencies>
```

### 可用 providers

该 starter 可与任何由 Spring AI 创建 `ChatModel` 的 provider 配合使用，包括：
Anthropic、Azure OpenAI、Bedrock Converse、DeepSeek、Google GenAI、HuggingFace、MiniMax、
Mistral AI、OCI GenAI、Ollama、OpenAI、Vertex AI 和 ZhiPu AI。

### 配置

通过匹配的 Spring AI starter 配置你的 provider，然后按需添加 Koog properties：

```
# example Spring AI provider configuration
spring.ai.openai.api-key=${OPENAI_API_KEY}

# Koog chat starter defaults
koog.spring.ai.chat.enabled=true
koog.spring.ai.chat.dispatcher.type=AUTO
```

如果你只有一个 `ChatModel` bean，一切都会自动工作。
adapter 会将其包装为 Koog `LLMClient`，并创建可直接使用的 `PromptExecutor`。

### 使用示例

注入 `PromptExecutor` 并用它运行 Koog agent：

KotlinJava

```
import ai.koog.agents.core.agent.AIAgent
import ai.koog.prompt.executor.clients.openai.OpenAIModels
import ai.koog.prompt.executor.model.PromptExecutor
import org.springframework.stereotype.Service

@Service
class MyAgentService(private val promptExecutor: PromptExecutor) {

    suspend fun askAgent(userMessage: String): String {
        val agent = AIAgent(
            promptExecutor = promptExecutor,
            llmModel = OpenAIModels.Chat.GPT5Nano,
            systemPrompt = "You are a helpful assistant."
        )

        return agent.run(userMessage)
    }
}
```

```
import ai.koog.agents.core.agent.AIAgent;
import ai.koog.prompt.executor.clients.openai.OpenAIModels;
import ai.koog.prompt.executor.model.PromptExecutor;
import org.springframework.stereotype.Service;

@Service
public class MyAgentService {
    private final PromptExecutor promptExecutor;

    public MyAgentService(PromptExecutor promptExecutor) {
        this.promptExecutor = promptExecutor;
    }

    public String askAgent(String userMessage) {
        var agent = AIAgent.builder()
                .promptExecutor(promptExecutor)
                .llmModel(OpenAIModels.Chat.GPT5Nano)
                .systemPrompt("You are a helpful assistant.")
                .build();

        return agent.run(userMessage);
    }
}
```

或者提供你自己的 `PromptExecutor` bean，完全覆盖自动配置的 bean。

### 配置属性（`koog.spring.ai.chat`）

| Property | Type | Default | Description |
| --- | --- | --- | --- |
| `enabled` | `Boolean` | `true` | 启用或禁用 chat 自动配置 |
| `chat-model-bean-name` | `String?` | `null` | 当存在多个模型时要使用的 `ChatModel` bean 名称 |
| `moderation-model-bean-name` | `String?` | `null` | 要使用的 `ModerationModel` bean 名称 |
| `provider` | `String?` | `null` | 要暴露的 Koog provider id，而不是从 `ChatModel` 类自动检测 |
| `dispatcher.type` | `AUTO` / `IO` | `AUTO` | 阻塞式模型调用使用的 dispatcher |
| `dispatcher.parallelism` | `Int` | `0` (= unbounded) | `IO` dispatcher 的最大并发数 |

### 多模型上下文

当注册了多个 `ChatModel` 或 `ModerationModel` beans 时，请指定使用哪一个：

```
koog.spring.ai.chat.chat-model-bean-name=openAiChatModel
koog.spring.ai.chat.moderation-model-bean-name=openAiModerationModel
```

如果没有 selector，只有在存在单个候选对象时自动配置才会激活。

### 扩展点

- **`ChatOptionsCustomizer`**：注册一个实现此接口的 Spring bean 来自定义 `ChatOptions`

KotlinJava

```
@Bean
fun chatOptionsCustomizer() = ChatOptionsCustomizer { options, params, model ->
    options
}
```

```
@Bean
public ChatOptionsCustomizer chatOptionsCustomizer() {
    return (options, params, model) -> options;
}
```

- **Custom `LLMClient`**：注册你自己的 `LLMClient` bean。除非你替换名为 `springAiChatModelLLMClient` 的 bean，否则它会与自动配置的 adapter 组合在一起。
- **Custom `PromptExecutor`**：注册你自己的 `PromptExecutor` bean 以覆盖自动配置的 `MultiLLMPromptExecutor`。

## Embedding Model Starter

### 概述

`koog-spring-ai-starter-model-embedding` starter 在 Spring AI 的 embedding model 抽象与 Koog agent 框架之间建立桥接。
它会自动配置：

- 一个委托给 Spring AI `EmbeddingModel` 的 Koog `LLMEmbeddingProvider`（`SpringAiLLMEmbeddingProvider`）

adapter 会把 Koog model id 转发到 Spring AI `EmbeddingOptions` 中，因此支持运行时模型选择的后端可以使用它。

### 添加依赖

将该依赖与任意 Spring AI embedding model starter 一起添加：

Gradle (Kotlin DSL)Maven

```
dependencies {
    implementation("ai.koog:koog-agents-jvm:$koogVersion")
    implementation("ai.koog:koog-spring-ai-starter-model-embedding:$koogVersion")
    implementation("org.springframework.ai:spring-ai-starter-model-openai")
}
```

```
<dependencies>
    <dependency>
        <groupId>ai.koog</groupId>
        <artifactId>koog-agents-jvm</artifactId>
        <version>${koog.version}</version>
    </dependency>
    <dependency>
        <groupId>ai.koog</groupId>
        <artifactId>koog-spring-ai-starter-model-embedding</artifactId>
        <version>${koog.version}</version>
    </dependency>
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-starter-model-openai</artifactId>
    </dependency>
</dependencies>
```

### 可用 providers

该 starter 可与任何由 Spring AI 创建 `EmbeddingModel` 的 provider 配合使用，包括：
Anthropic、Azure OpenAI、Bedrock、Google GenAI、HuggingFace、Mistral AI、OCI GenAI、
Ollama、OpenAI、Transformers、Vertex AI 和 ZhiPu AI。

### 配置

通过 Spring AI 配置你的 embedding provider，然后按需添加 Koog properties：

```
# example Spring AI provider configuration
spring.ai.openai.api-key=${OPENAI_API_KEY}

# Koog embedding starter defaults
koog.spring.ai.embedding.enabled=true
koog.spring.ai.embedding.dispatcher.type=AUTO
```

如果你只有一个 `EmbeddingModel` bean，一切都会自动工作。
adapter 会将其包装为 Koog `LLMEmbeddingProvider`。

### 使用示例

注入 `LLMEmbeddingProvider` 并用它执行 embedding 操作：

KotlinJava

```
import ai.koog.prompt.executor.clients.LLMEmbeddingProvider
import ai.koog.prompt.executor.clients.openai.OpenAIModels
import org.springframework.stereotype.Service

@Service
class MyEmbeddingService(private val embeddingProvider: LLMEmbeddingProvider) {

    suspend fun getEmbedding(text: String): List<Double> {
        return embeddingProvider.embed(
            text,
            OpenAIModels.Embeddings.TextEmbedding3Small
        )
    }
}
```

```
import ai.koog.prompt.executor.clients.LLMEmbeddingProvider;
import ai.koog.prompt.executor.clients.openai.OpenAIModels;
import org.springframework.stereotype.Service;
import java.util.List;

@Service
public class MyEmbeddingService {
    private final LLMEmbeddingProvider embeddingProvider;

    public MyEmbeddingService(LLMEmbeddingProvider embeddingProvider) {
        this.embeddingProvider = embeddingProvider;
    }

    public List<Double> getEmbedding(String text) {
        return embeddingProvider.embed(
                text,
                OpenAIModels.Embeddings.TextEmbedding3Small
        );
    }
}
```

或者提供你自己的 `LLMEmbeddingProvider` bean，完全覆盖自动配置的 adapter。

### 配置属性（`koog.spring.ai.embedding`）

| Property | Type | Default | Description |
| --- | --- | --- | --- |
| `enabled` | `Boolean` | `true` | 启用或禁用 embedding 自动配置 |
| `embedding-model-bean-name` | `String?` | `null` | 当存在多个模型时要使用的 `EmbeddingModel` bean 名称 |
| `dispatcher.type` | `AUTO` / `IO` | `AUTO` | 阻塞式 embedding 调用使用的 dispatcher |
| `dispatcher.parallelism` | `Int` | `0` (= unbounded) | `IO` dispatcher 的最大并发数 |

### 多模型上下文

当注册了多个 `EmbeddingModel` beans 时，请指定使用哪一个：

```
koog.spring.ai.embedding.embedding-model-bean-name=openAiEmbeddingModel
```

如果没有 selector，只有在存在单个候选对象时自动配置才会激活。

### 扩展点

- **Custom `LLMEmbeddingProvider`**：注册你自己的 bean，完全覆盖自动配置的 adapter。

## Chat Memory Starter

### 概述

`koog-spring-ai-starter-chat-memory` starter 在 Spring AI 的 chat memory 抽象与 Koog agent 框架之间建立桥接。
它会自动配置：

- 一个委托给 Spring AI `ChatMemoryRepository` 的 Koog `ChatHistoryProvider`（`SpringAiChatHistoryProvider`）

该 starter 提供纯文本对话持久化，而不是完整的 Koog 执行状态持久化。

### 纯文本契约

只会持久化纯文本 `System`、`User` 和 `Assistant` messages。
以下内容在存储时会被静默丢弃：

- `MessagePart.Tool.Call`
- `MessagePart.Tool.Result`
- `MessagePart.Reasoning`
- 任何带 attachments 的 message

加载时，Spring AI `TOOL` rows 会被静默跳过。
时间戳、token 计数、finish reasons 和自定义 metadata 等元数据不会被保留。

### 添加依赖

将该依赖与 Spring AI chat memory repository 实现一起添加：

Gradle (Kotlin DSL)Maven

```
dependencies {
    implementation("ai.koog:koog-agents-jvm:$koogVersion")
    implementation("ai.koog:koog-spring-ai-starter-chat-memory:$koogVersion")
    implementation("org.springframework.ai:spring-ai-starter-model-chat-memory-repository-jdbc")
}
```

```
<dependencies>
    <dependency>
        <groupId>ai.koog</groupId>
        <artifactId>koog-agents-jvm</artifactId>
        <version>${koog.version}</version>
    </dependency>
    <dependency>
        <groupId>ai.koog</groupId>
        <artifactId>koog-spring-ai-starter-chat-memory</artifactId>
        <version>${koog.version}</version>
    </dependency>
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-starter-model-chat-memory-repository-jdbc</artifactId>
    </dependency>
</dependencies>
```

### 可用 providers

该 starter 可与任何暴露 `ChatMemoryRepository` 的 Spring AI chat memory repository 实现配合使用，
包括基于 JDBC、Redis、Cassandra、Cosmos DB、MongoDB 和 Neo4j 的 repositories。

### 配置

通常，除了 Spring AI repository 设置外，不需要额外配置：

```
# Koog chat-memory starter defaults
koog.spring.ai.chat-memory.enabled=true
koog.spring.ai.chat-memory.dispatcher.type=AUTO
```

如果你只有一个 `ChatMemoryRepository` bean，一切都会自动工作。
adapter 会将其包装为 Koog `ChatHistoryProvider`。

### 使用示例

使用自动配置的 `ChatHistoryProvider` 在你的 agent 上安装 `ChatMemory` 功能：

Kotlin

```
import ai.koog.agents.chatMemory.feature.ChatMemory
import ai.koog.agents.chatMemory.feature.ChatHistoryProvider
import ai.koog.agents.core.agent.AIAgent
import ai.koog.prompt.executor.clients.openai.OpenAIModels
import ai.koog.prompt.executor.model.PromptExecutor
import org.springframework.stereotype.Service

@Service
class MyAgentService(
    private val promptExecutor: PromptExecutor,
    private val chatStorage: ChatHistoryProvider,
) {

    suspend fun askAgent(userMessage: String, sessionId: String): String {
        val agent = AIAgent(
            promptExecutor = promptExecutor,
            llmModel = OpenAIModels.Chat.GPT5Nano,
            systemPrompt = "You are a helpful assistant.",
        ) {
            install(ChatMemory) {
                chatHistoryProvider = chatStorage
            }
        }

        return agent.run(userMessage, sessionId)
    }
}
```

### 配置属性（`koog.spring.ai.chat-memory`）

| Property | Type | Default | Description |
| --- | --- | --- | --- |
| `enabled` | `Boolean` | `true` | 启用或禁用 chat-memory 自动配置 |
| `chat-memory-repository-bean-name` | `String?` | `null` | 当存在多个 repositories 时要使用的 `ChatMemoryRepository` bean 名称 |
| `dispatcher.type` | `AUTO` / `IO` | `AUTO` | 阻塞式 repository 调用使用的 dispatcher |
| `dispatcher.parallelism` | `Int` | `0` (= unbounded) | `IO` dispatcher 的最大并发数 |

### 多 repository 上下文

当注册了多个 `ChatMemoryRepository` beans 时，请指定使用哪一个：

```
koog.spring.ai.chat-memory.chat-memory-repository-bean-name=jdbcChatMemoryRepository
```

如果没有 selector，只有在存在单个候选对象时自动配置才会激活。

### 当前限制

- 仅持久化文本对话历史
- 工具调用、工具结果、reasoning messages 和 attachments 不会被持久化
- Spring AI `TOOL` messages 在加载时会被跳过
- message metadata 无法在往返过程中保留

## Vector Store Starter

### 概述

`koog-spring-ai-starter-vector-store` starter 在 Spring AI vector-store 抽象与 Koog 的 RAG 存储接口之间建立桥接。
它会自动配置：

- 一个作为 Koog `KoogVectorStore` 暴露的 `SpringAiKoogVectorStore` adapter

`KoogVectorStore` 组合了：

- `WriteStorage<TextDocument>`
- `SearchStorage<TextDocument, SimilaritySearchRequest>`
- `FilteringDeletionStorage`

示例通常使用 `DocumentWithMetadata` 作为具体文档类型。

### 添加依赖

将该依赖与 Spring AI vector-store starter 一起添加：

Gradle (Kotlin DSL)Maven

```
dependencies {
    implementation("ai.koog:koog-spring-ai-starter-vector-store:$koogVersion")
    implementation("org.springframework.ai:spring-ai-starter-vector-store-pgvector")
}
```

```
<dependencies>
    <dependency>
        <groupId>ai.koog</groupId>
        <artifactId>koog-spring-ai-starter-vector-store</artifactId>
        <version>${koog.version}</version>
    </dependency>
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-starter-vector-store-pgvector</artifactId>
    </dependency>
</dependencies>
```

### 可用 providers

该 starter 可与任何暴露 `VectorStore` 的 Spring AI 实现配合使用，包括 PgVector、
Azure AI Search、Cassandra、Chroma、Elasticsearch、Milvus、MongoDB Atlas、Neo4j、OpenSearch、
Oracle、Pinecone、Qdrant、Redis、Typesense 和 Weaviate。

### 配置

通常，除了 Spring AI vector-store 设置外，不需要额外 Koog 配置：

```
# Koog vector-store starter defaults
koog.spring.ai.vectorstore.enabled=true
koog.spring.ai.vectorstore.dispatcher.type=AUTO
```

如果你只有一个 `VectorStore` bean，一切都会自动工作。
adapter 会将其包装为 Koog `KoogVectorStore`。

### 使用示例

直接将 `KoogVectorStore` 注入到你的 Spring 组件中：

Kotlin

```
import ai.koog.rag.base.TextDocument
import ai.koog.rag.base.storage.search.SearchResult
import ai.koog.rag.base.storage.search.SimilaritySearchRequest
import ai.koog.spring.ai.vectorstore.DocumentWithMetadata
import ai.koog.spring.ai.vectorstore.KoogVectorStore
import org.springframework.stereotype.Service

@Service
class MyKnowledgeBase(
    private val vectorStore: KoogVectorStore,
) {

    suspend fun ingest(text: String): List<String> {
        return vectorStore.add(
            listOf(
                DocumentWithMetadata(
                    content = text,
                    metadata = mapOf("source" to "user")
                )
            )
        )
    }

    suspend fun search(query: String): List<SearchResult<TextDocument>> {
        return vectorStore.search(
            SimilaritySearchRequest(queryText = query, limit = 5)
        )
    }

    suspend fun remove(ids: List<String>) {
        vectorStore.delete(ids)
    }
}
```

### 配置属性（`koog.spring.ai.vectorstore`）

| Property | Type | Default | Description |
| --- | --- | --- | --- |
| `enabled` | `Boolean` | `true` | 启用或禁用 vector-store 自动配置 |
| `vector-store-bean-name` | `String?` | `null` | 当存在多个 stores 时要使用的 `VectorStore` bean 名称 |
| `dispatcher.type` | `AUTO` / `IO` | `AUTO` | 阻塞式 vector-store 调用使用的 dispatcher |
| `dispatcher.parallelism` | `Int` | `0` (= unbounded) | `IO` dispatcher 的最大并发数 |

### 多 store 上下文

当注册了多个 `VectorStore` beans 时，请指定使用哪一个：

```
koog.spring.ai.vectorstore.vector-store-bean-name=pgVectorStore
```

如果没有 selector，只有在存在单个候选对象时自动配置才会激活。

### 当前限制

- Spring AI 的 `VectorStore` 契约仅暴露相似性搜索
- Update 实现为先 `delete(ids)` 再 `add(documents)`，因此它不是事务性的
- 由于 Spring AI 没有可移植的按 id 读取 API，因此未实现 `LookupStorage`
- `delete(ids)` 会原样返回输入 ids；Spring AI 不会确认实际删除了哪些文档
- `delete(filterExpression)` 返回空列表；Spring AI 不返回匹配文档的 ids
- 未实现 namespace scoping
- Metadata values 必须是 `String`、`Number` 或 `Boolean` 等 primitive values

## 后续步骤

- 了解 [basic agents](../agents/basic-agents/) 以构建最小 AI 工作流
- 探索 [graph-based agents](../agents/graph-based-agents/) 以了解高级用例
- 参阅 [tools overview](../tools/) 以扩展 agent 的能力
- 阅读 [retrieval-augmented generation](../retrieval-augmented-generation/) 以了解 RAG 概念
- 查看 [examples](../examples/) 了解真实世界实现
- 阅读 [Spring Boot Integration](../spring-boot/) 指南，了解直接 Koog starter 方法
