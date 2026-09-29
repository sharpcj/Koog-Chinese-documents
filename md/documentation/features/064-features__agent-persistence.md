# Agent 持久化

Agent Persistence 是 Koog 框架中为 AI agent 提供检查点功能的特性。
它允许你在执行过程中的特定位置保存和恢复 agent 的状态，从而支持以下能力：

- 从特定位置恢复 agent 执行
- 回滚到以前的状态
- 跨会话持久化 agent 状态

## 核心概念

### 检查点

检查点会捕获 agent 在执行中特定时刻的完整状态，包括：

- 消息历史（用户、系统、助手和工具之间的所有交互）
- 最后一个成功执行的节点
- 该节点的输出数据
- 选定的 LLM
- 通用 LLM 参数
- 选定的工具
- `AIAgentStorage` 内容（执行期间存储的键值数据）
- 创建时间戳

检查点由唯一 ID 标识，并与特定 agent 关联。

### `AIAgentStorage` 持久化

创建检查点时，框架会序列化 `AIAgentStorage` 中当前保存的所有值，并将其包含在检查点中。
恢复时，这些值会被反序列化，并以检查点创建时的原样提供给恢复后的 agent。

**只有可序列化的值会被持久化。**
使用的序列化器是在 `AIAgentConfig` 中通过其 `serializer` 属性配置的序列化器。
无法由该序列化器编码的值会被静默跳过，并且不会出现在恢复后的存储中。

写入检查点时，不可序列化的值会被静默丢弃。
如果从检查点恢复后某个值缺失，请确认其类型可由已配置的序列化器序列化。

更多信息请参阅[序列化](../../serialization/)。

## 安装

要使用 Agent Persistence 特性，请将其添加到 agent 配置中：

KotlinJava

```
val agent = AIAgent(
    promptExecutor = executor,
    llmModel = OllamaModels.Meta.LLAMA_3_2,
) {
    install(Persistence) {
        // Use in-memory storage for snapshots
        storage = InMemoryPersistenceStorageProvider()
    }
}
```

```
AIAgent<String, String> agent = AIAgent.builder()
    .promptExecutor(executor)
    .llmModel(OllamaModels.Meta.LLAMA_3_2)
    .install(Persistence.Feature, cfg -> {
        // Use in-memory storage for snapshots
        cfg.setStorage(new InMemoryPersistenceStorageProvider());
    })
.build();
```

## 配置选项

Agent Persistence 特性有三个主要配置选项：

- **存储提供者**：用于保存和检索检查点的提供者。
- **连续持久化**：每个节点运行后自动创建检查点。
- **回滚策略**：决定回滚到检查点时将恢复哪种状态。

### 存储提供者

设置用于保存和检索检查点的存储提供者：

KotlinJava

```
install(Persistence) {
    storage = InMemoryPersistenceStorageProvider()
}
```

```
AIAgent<String, String> agent = AIAgent.builder()
    .promptExecutor(executor)
    .llmModel(OllamaModels.Meta.LLAMA_3_2)
    .install(Persistence.Feature, cfg -> {
        cfg.setStorage(new InMemoryPersistenceStorageProvider());
    })
    .build();
```

框架包含以下内置提供者：

- `InMemoryPersistenceStorageProvider`：将检查点存储在内存中（应用重启后丢失）。
- `FilePersistenceStorageProvider`：将检查点持久化到文件系统。
- `NoPersistenceStorageProvider`：不存储检查点的空操作实现。这是默认提供者。

你也可以通过实现 `PersistenceStorageProvider` 接口来实现自定义存储提供者。
更多信息请参阅[自定义存储提供者](#自定义存储提供者)。

### 连续持久化

连续持久化意味着每个节点运行后都会自动创建一个检查点。
要禁用连续持久化，请使用以下代码：

KotlinJava

```
install(Persistence) {
    enableAutomaticPersistence = false
}
```

```
AIAgent<String, String> agent = AIAgent.builder()
    .promptExecutor(executor)
    .llmModel(OllamaModels.Meta.LLAMA_3_2)
    .install(Persistence.Feature, cfg -> {
        cfg.setEnableAutomaticPersistence(true);
    })
    .build();
```

如果禁用了连续持久化，你仍然可以手动创建检查点。

## 基本用法

### 创建检查点

要了解如何在 agent 执行中的特定位置创建检查点，请参阅以下代码示例：

KotlinJava

```
suspend fun example(context: AIAgentContext) {
    // Create a checkpoint with the current state
    val checkpoint = context.persistence().createCheckpointAfterNode(
        agentContext = context,
        nodePath = context.executionInfo.path(),
        lastOutput = outputData,
        lastOutputType = outputType,
        checkpointId = context.runId,
        version = 0L
    )

    // The checkpoint ID can be stored for later use
    val checkpointId = checkpoint?.checkpointId
}
```

```
// PersistenceKt.persistence() is the Java-accessible form of the Kotlin extension function
Persistence persistence = PersistenceKt.persistence(context);

// Create a checkpoint with the current state
AgentCheckpointData checkpoint = persistence.createCheckpointAfterNode(
    context,
    context.getExecutionInfo().path(),
    outputData,
    TypeToken.of(String.class),
    0L,
    context.getRunId()
);

// The checkpoint ID can be stored for later use
String checkpointId = checkpoint != null ? checkpoint.getCheckpointId() : null;
```

### 从检查点恢复

要从特定检查点恢复 agent 状态，请参阅以下代码示例：

KotlinJava

```
suspend fun example(context: AIAgentContext, checkpointId: String) {
    // Roll back to a specific checkpoint
    context.persistence().rollbackToCheckpoint(checkpointId, context)

    // Or roll back to the latest checkpoint
    context.persistence().rollbackToLatestCheckpoint(context)
}
```

```
Persistence persistence = PersistenceKt.persistence(context);

// Roll back to a specific checkpoint
persistence.rollbackToCheckpoint(checkpointId, context);

// Or roll back to the latest checkpoint
persistence.rollbackToLatestCheckpoint(context);
```

#### 回滚工具产生的所有副作用

某些工具产生副作用是很常见的。具体来说，当你在后端运行 agent 时，
某些工具很可能会执行一些数据库事务。这会让 agent 回到过去的状态变得更困难。

假设你有一个 `createUser` 工具，它会在数据库中创建一个新用户。而你的 agent 随时间产生了多个工具调用：

```
tool call: createUser "Alex"

->>>> checkpoint-1 <<<<-

tool call: createUser "Daniel"
tool call: createUser "Maria"
```

现在你想回滚到一个检查点。仅恢复 agent 的状态（包括消息历史和策略图节点）并不足以
达到检查点之前的真实世界状态。你还应该恢复工具调用产生的副作用。在我们的示例中，
这意味着从数据库中移除 `Maria` 和 `Daniel`。

在 Koog Persistence 中，你可以通过向 `Persistence` 特性配置提供 `RollbackToolRegistry` 来实现：

KotlinJava

```
install(Persistence) {
    enableAutomaticPersistence = true
    rollbackToolRegistry = RollbackToolRegistry {
        // For every `createUser` tool call there will be a `removeUser` invocation in the reverse order 
        // when rolling back to the desired execution point.
        // Note: `removeUser` tool should take the same exact arguments as `createUser`. 
        // It's the developer's responsibility to make sure that `removeUser` invocation rolls back all side-effects of `createUser`:
        registerRollback(::createUser, ::removeUser)
    }
}
```

```
AIAgent<String, String> agent = AIAgent.builder()
    .promptExecutor(executor)
    .llmModel(OllamaModels.Meta.LLAMA_3_2)
    .install(Persistence.Feature, cfg -> {
        cfg.setEnableAutomaticPersistence(true);
        cfg.setRollbackToolRegistry(
            RollbackToolRegistry.builder()
                // For every tool in UserToolSet there will be a corresponding rollback tool
                // in UserRollbackToolSet, invoked in reverse order when rolling back.
                // UserRollbackToolSet methods must be annotated with @Reverts to link
                // them to the corresponding tools in UserToolSet.
                .registerRollbacks(new UserToolSet(), new UserRollbackToolSet())
                .build()
        );
    })
    .build();
```

### 使用扩展函数

Agent Persistence 特性提供了用于处理检查点的便捷扩展函数：

KotlinJava

```
suspend fun example(context: AIAgentContext) {
    // Access the checkpoint feature
    val checkpointFeature = context.persistence()

    // Or perform an action with the checkpoint feature
    context.withPersistence { ctx ->
        // 'this' is the checkpoint feature
        createCheckpointAfterNode(
            agentContext = ctx,
            nodePath = ctx.executionInfo.path(),
            lastOutput = outputData,
            lastOutputType = outputType,
            checkpointId = ctx.runId,
            version = 0L
        )
    }
}
```

```
// Access the persistence feature via PersistenceKt (the Kotlin extension function)
Persistence persistence = PersistenceKt.persistence(context);

// Use the persistence feature directly to create a checkpoint
persistence.createCheckpointAfterNode(
    context,
    context.getExecutionInfo().path(),
    outputData,
    TypeToken.of(String.class),
    0L,
    context.getRunId()
);
```

## 高级用法

### 自定义存储提供者

你可以通过实现 `PersistenceStorageProvider` 接口来实现自定义存储提供者：

KotlinJava

```
class MyCustomStorageProvider<MyFilterType> : PersistenceStorageProvider<MyFilterType> {
    override suspend fun getCheckpoints(sessionId: String, filter: MyFilterType?): List<AgentCheckpointData> {
        TODO("Not yet implemented")
    }

    override suspend fun saveCheckpoint(sessionId: String, agentCheckpointData: AgentCheckpointData) {
        TODO("Not yet implemented")
    }

    override suspend fun getLatestCheckpoint(sessionId: String, filter: MyFilterType?): AgentCheckpointData? {
        TODO("Not yet implemented")
    }
}
```

```
class MyCustomStorageProvider extends AsyncPersistenceStorageProvider<Object> {
    @Override
    public CompletableFuture<List<AgentCheckpointData>> getCheckpointsAsync(
            String agentId, Object filter) {
        throw new UnsupportedOperationException("Not yet implemented");
    }

    @Override
    public CompletableFuture<Boolean> saveCheckpointAsync(
            String agentId, AgentCheckpointData checkpointData) {
        throw new UnsupportedOperationException("Not yet implemented");
    }

    @Override
    public CompletableFuture<AgentCheckpointData> getLatestCheckpointAsync(
            String agentId, Object filter) {
        throw new UnsupportedOperationException("Not yet implemented");
    }
}
```

要在特性配置中使用你的自定义提供者，请在为 agent 配置 Agent Persistence
特性时将其设置为存储。

KotlinJava

```
install(Persistence) {
    storage = MyCustomStorageProvider<Any>()
}
```

```
AIAgent<String, String> agent = AIAgent.builder()
    .promptExecutor(executor)
    .llmModel(OllamaModels.Meta.LLAMA_3_2)
    .install(Persistence.Feature, cfg -> {
        cfg.setStorage(new MyCustomStorageProvider());
    })
    .build();
```

### 设置执行点

对于高级控制，你可以直接设置 agent 的执行点：

KotlinJava

```
suspend fun example(context: AIAgentContext) {
    // You can set the execution point after some node and provide an output from the node:
    context.persistence().setExecutionPointAfterNode(
        agentContext = context,
        nodePath = context.executionInfo.path(),
        messageHistory = customMessageHistory,
        output = customOutput
    )
}
```

```
Persistence persistence = PersistenceKt.persistence(context);

// Set the execution point after a node and provide an output from the node:
persistence.setExecutionPointAfterNode(
    context,
    context.getExecutionInfo().path(),
    customMessageHistory,
    customOutput
);
```

这允许你对 agent 状态进行比仅从检查点恢复更细粒度的控制。
