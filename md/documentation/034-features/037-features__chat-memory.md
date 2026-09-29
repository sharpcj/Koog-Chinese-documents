# 聊天记忆

`ChatMemory` 特性使 AI agent 能够存储对话历史，并在多次运行之间检索它。
安装后，agent 会在每次运行开始时自动加载之前的消息，并在运行完成时
存储更新后的对话，从而实现自然的多轮聊天。

主要能力：

- 按会话 ID 自动加载和存储对话历史
- 通过 `ChatHistoryProvider` 使用可插拔的存储后端
- 内置预处理器，用于限制历史大小和过滤消息
- 支持用于任意消息转换的自定义预处理器

## 添加依赖

聊天记忆是一个可选[特性](../)，Koog 默认不提供。
要为你的 Koog agent 实现聊天记忆，请添加 [`ai.koog:agents-features-memory`](https://mvnrepository.com/artifact/ai.koog/agents-features-memory) 的依赖：

Gradle (Kotlin)Gradle (Groovy)Maven

build.gradle.kts

```
dependencies {
    implementation("ai.koog:agents-features-memory:$koogVersion")
}
```

build.gradle

```
dependencies {
    implementation 'ai.koog:agents-features-memory:$koogVersion'
}
```

pom.xml

```
<dependency>
    <groupId>ai.koog</groupId>
    <artifactId>agents-features-memory-jvm</artifactId>
    <version>$koogVersion</version>
</dependency>
```

Note

`ChatMemory` 特性从 Koog 版本 **0.7.0** 开始可用。

## 启用聊天记忆

创建 agent 时使用 `install()` 方法安装 `ChatMemory`：

KotlinJava

```
val agent = AIAgent(
    promptExecutor = simpleOpenAIExecutor(System.getenv("OPENAI_API_KEY")),
    llmModel = OpenAIModels.Chat.GPT4oMini
) {
    install(ChatMemory)
}
```

```
AIAgent<String, String> agent = AIAgent.builder()
    .promptExecutor(executor)
    .llmModel(OpenAIModels.Chat.GPT4oMini)
    .install(ChatMemory.Feature)
    .build();
```

默认情况下，它使用内存中的[聊天历史提供者](#历史提供者)，且不使用[预处理器](#预处理器)。
可以配置 `ChatMemory` 特性以使用自定义聊天历史提供者和预处理器，例如：

KotlinJava

```
val agent = AIAgent(
    promptExecutor = simpleOpenAIExecutor(System.getenv("OPENAI_API_KEY")),
    llmModel = OpenAIModels.Chat.GPT4oMini
) {
    install(ChatMemory) {
        chatHistoryProvider = MyDatabaseChatHistoryProvider()
        windowSize(20)
        filterMessages { it is Message.User || it is Message.Assistant }
    }
}
```

```
AIAgent<String, String> agent = AIAgent.builder()
    .promptExecutor(executor)
    .llmModel(OpenAIModels.Chat.GPT4oMini)
    .install(ChatMemory.Feature, config -> config
            .chatHistoryProvider(new MyDatabaseChatHistoryProvider())
            .windowSize(20)
            .filterMessages(msg -> msg instanceof Message.User || msg instanceof Message.Assistant))
    .build();
```

## 会话 ID

将会话 ID 作为第二个参数提供给 `agent.run()`。
`ChatMemory` 使用此 ID 存储和加载对话：

```
// First run - the agent saves the chat history at the end
agent.run("What is the capital of France?", "session-1")

// Second run — the agent loads the previous exchange
agent.run("And what about Germany?", "session-1")
```

不同的会话 ID 会产生完全隔离的历史。

## 历史提供者

默认的 `InMemoryChatHistoryProvider` 是线程安全的，但不是持久化的（重启后历史会丢失）。
生产环境中，请实现你自己的 `ChatHistoryProvider`，以持久化存储消息。

```
class MyDatabaseChatHistoryProvider(private val db: Database) : ChatHistoryProvider {
    override suspend fun store(conversationId: String, messages: List<Message>) {
        db.saveMessages(conversationId, messages)
    }

    override suspend fun load(conversationId: String): List<Message> {
        return db.loadMessages(conversationId) ?: emptyList()
    }
}
```

## 预处理器

预处理器会在加载时（agent 看到消息之前）和存储时（保存之前）转换消息列表。
它们会按你添加到 `ChatMemory` 特性配置中的顺序依次运行。

### 内置预处理器

| 配置方法 | 预处理器类 | 行为 |
| --- | --- | --- |
| `windowSize(n)` | `WindowSizePreProcessor` | 只保留最后 `n` 条消息 |
| `filterMessages { ... }` | `FilterMessagesPreProcessor` | 保留与谓词匹配的消息 |

### 预处理器顺序

预处理器按顺序运行，每个输出都会成为下一个输入。
这意味着顺序很重要。

```
// Effect: keep last 10 messages, then filter short ones from those 10
windowSize(10)
filterMessages { it.content.length <= 100 }

// Effect: filter short messages first, then keep last 10 of the survivors
filterMessages { it.content.length <= 100 }
windowSize(10)
```

### 自定义预处理器

要创建自定义预处理器，请实现 `ChatMemoryPreProcessor` 接口：

```
class RedactEmailsPreProcessor : ChatMemoryPreProcessor {
    override fun preprocess(messages: List<Message>): List<Message> {
        return messages.map { message ->
            // Replace email addresses in message content
            Message.User(message.content.replace(Regex("[\\w.]+@[\\w.]+"), "[REDACTED]"))
        }
    }
}
```

然后将其添加到配置中：

```
install(ChatMemory) {
    addPreProcessor(RedactEmailsPreProcessor())
    windowSize(50)
}
```

## 聊天记忆与 agent 持久化

`ChatMemory` 将每次 `agent.run()` 调用视为一个原子且自包含的循环。
agent 在运行前加载聊天历史，并在成功运行后存储它。
如果 agent 在运行期间崩溃，它不会存储当前聊天消息，
这意味着聊天历史会保持为运行前的状态。

[Persistence](../agent-persistence/) 会在运行期间将 agent 的内部执行状态
（图节点、消息历史、输入和输出）捕获为检查点。
如果 agent 崩溃，它可以从最后一个检查点恢复。

|  | ChatMemory | Persistence |
| --- | --- | --- |
| **保存内容** | 对话消息 | 执行状态 |
| **保存时机** | `agent.run()` 完成后 | 每个图节点后或运行期间手动定义的位置 |
| **崩溃行为** | 进行中的运行丢失；先前历史保持完整 | 可从最后一个检查点恢复 |
| **典型用途** | 多轮聊天连续性 | 具有崩溃恢复需求的长时间运行 agent |

如果你的 agent 执行长时间运行任务，而执行中崩溃代价很高，请考虑
同时安装这两个特性：

```
val agent = AIAgent(
    promptExecutor = executor,
    llmModel = OpenAIModels.Chat.GPT4oMini,
    systemPrompt = "You are a helpful assistant.",
) {
    install(ChatMemory) {
        chatHistoryProvider = MyDatabaseProvider()
        windowSize(50)
    }
    install(Persistence) {
        storage = MyPersistenceStorageProvider()
        enableAutomaticPersistence = true
    }
}
```

## 最佳实践

- **始终设置窗口大小**，以防止对话无限增长。
- **谨慎安排预处理器顺序**，因为先过滤再窗口化和先窗口化再过滤会产生不同结果。
- **使用有意义的会话 ID** 进行历史隔离：用户 ID、聊天线程 ID 或 UUID 都很合适。
- **为生产环境实现持久化提供者**，因为默认的 `InMemoryChatHistoryProvider` 会在重启时丢失历史。

## 后续步骤

- 了解如何[构建一个带记忆的简单 CLI 聊天循环](chat-agent-with-memory/)
- 查看[带记忆的聊天端点](chat-backend-with-memory/)示例
