# 节点之间的数据传递

## 概览

Koog 提供了一种使用 `AIAgentStorage` 存储和传递数据的方式。`AIAgentStorage` 是一个 key-value 存储系统，旨在以类型安全的方式在不同节点甚至子图之间传递数据。

可以通过 agent 节点中可用的 `storage` 属性（`storage: AIAgentStorage`）访问该存储，从而在 AI agent 系统的不同组件之间无缝共享数据。

## 键和值结构

key-value 数据存储结构依赖于 `AIAgentStorageKey` 数据类。有关 `AIAgentStorageKey` 的更多信息，请参见下面的章节。

### AIAgentStorageKey

该存储使用类型化 key 系统，在存储和检索数据时提供类型安全。

`AIAgentStorageKey<T>` 类表示用于标识和访问数据的存储 key。以下是此类的关键特性：

- 泛型类型参数 `T` 指定与此 key 关联的数据类型，从而提供类型安全。
- 每个 key 都有一个 `name` 属性，它是一个字符串标识符，便于识别和调试。
- 每个 key 实例都是唯一的。`name` 不用于判断唯一性，因此可以有多个同名 key。这允许复用现有策略组件，而不会有意外覆盖存储中数据的风险。

## 用法示例

以下章节给出了创建存储 key 并用它存储和检索数据的实际示例。

### 定义表示你的数据的类

存储要传递的数据的第一步，是创建一个表示该数据的类。下面是一个包含基础用户数据的简单类示例：

KotlinJava

```
class UserData(
   val name: String,
   val age: Int
)
```

```
record UserData(
    String name,
    int age
) {}
```

定义完成后，按如下方式使用该类创建存储 key。

### 创建存储 key

为已定义的数据结构创建一个类型化存储 key：

KotlinJava

```
val userDataKey = createStorageKey<UserData>("user-data")
```

```
AIAgentStorageKey<UserData> userDataKey = AIAgentStorage.createStorageKey("user-data", TypeToken.of(UserData.class));
```

`createStorageKey` 函数接收一个用于识别和调试的字符串参数，以及一个表示值类型的 `TypeToken`（在 Java 中；Kotlin 会自动使用 reified generics）。

### 存储数据

若要使用已创建的存储 key 保存数据，请在节点中使用 `storage.set(key: AIAgentStorageKey<T>, value: T)` 方法：

KotlinJava

```
val nodeSaveData by node<Unit, Unit> {
    storage.set(userDataKey, UserData("John", 26))
}
```

```
var nodeSaveData = AIAgentNode.builder("nodeSaveData")
    .withInput(String.class)
    .withOutput(String.class)
    .withAction((input, ctx) -> {
        ctx.getStorage().set(userDataKey, new UserData("John", 26));
        return "";
    })
    .build();
```

### 检索数据

若要检索数据，请在节点中使用 `storage.get` 方法：

KotlinJava

```
val nodeRetrieveData by node<String, Unit> { message ->
    storage.get(userDataKey)?.let { userFromStorage ->
        println("Hello dear $userFromStorage, here's a message for you: $message")
    }
}
```

```
var nodeRetrieveData = AIAgentNode.builder("nodeRetrieveData")
    .withInput(String.class)
    .withOutput(String.class)
    .withAction((message, ctx) -> {
        var userData = ctx.getStorage().get(userDataKey);
        System.out.println("Hello dear %s, here's a message for you: %s".formatted(userData, message));
        return "";
    })
    .build();
```

## API 文档

有关 `AIAgentStorage` 类的完整参考，请参见 [AIAgentStorage](https://api.koog.ai/agents/agents-core/ai.koog.agents.core.agent.entity/-a-i-agent-storage/index.html)。

有关 `AIAgentStorage` 类中可用的各个函数，请参见以下 API 参考：

- [clear](https://api.koog.ai/agents/agents-core/ai.koog.agents.core.agent.entity/-a-i-agent-storage/clear.html)
- [get](https://api.koog.ai/agents/agents-core/ai.koog.agents.core.agent.entity/-a-i-agent-storage/get.html)
- [getValue](https://api.koog.ai/agents/agents-core/ai.koog.agents.core.agent.entity/-a-i-agent-storage/get-value.html)
- [putAll](https://api.koog.ai/agents/agents-core/ai.koog.agents.core.agent.entity/-a-i-agent-storage/put-all.html)
- [remove](https://api.koog.ai/agents/agents-core/ai.koog.agents.core.agent.entity/-a-i-agent-storage/remove.html)
- [set](https://api.koog.ai/agents/agents-core/ai.koog.agents.core.agent.entity/-a-i-agent-storage/set.html)
- [toMap](https://api.koog.ai/agents/agents-core/ai.koog.agents.core.agent.entity/-a-i-agent-storage/to-map.html)

## 其他信息

- `AIAgentStorage` 是线程安全的，它使用 Mutex 来确保并发访问得到妥善处理。
- 检索值时会自动处理类型转换，从而在整个应用中保证类型安全。
- 对于值的非空访问，请使用 `getValue` 方法；如果 key 不存在，该方法会抛出异常。
- 你可以使用 `clear` 方法完全清空存储，它会移除所有已存储的 key-value 对。
