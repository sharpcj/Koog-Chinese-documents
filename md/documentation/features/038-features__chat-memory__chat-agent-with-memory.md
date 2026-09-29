# 构建带记忆的聊天 agent

本指南演示如何创建一个命令行对话聊天应用，
使用 [ChatMemory](../) 特性在多次 agent 交互之间记住之前的消息。

该 CLI 应用执行以下循环：

- 从控制台读取输入
- 如果输入不是 `/bye` 且不为空，则使用用户输入和指定的会话 ID 运行 agent
- agent 首先加载该会话 ID 的先前对话历史，
  并将这些消息与用户输入一起添加到 prompt 中
- agent 执行 LLM 交互
- 在运行结束时、返回响应之前，
  agent 将完整对话历史存储到指定会话 ID 下，
  并将大小限制为最新 20 条消息
- 应用随后打印 agent 的响应

以下是示意图：

```
graph TB
    subgraph agent [Agent with chat memory]
        load[Load chat history]
        save[Save chat history]
        llm([LLM interaction])

        load --> llm --> save
    end

    start((Start))
    read[Read input]
    print[Print response]
    exit((Exit))

    start --> read
    read --"/bye"--> exit
    read --"empty"--> read
    read --"User input"--> agent
    agent --"Agent response"--> print --> read
```

## 代码

前置条件

确保你的环境和项目满足以下要求：

- JDK 17+
- Kotlin 2.2.0+
- Gradle 8.0+ 或 Maven 3.8+

添加主 [Koog agents 包](https://central.sonatype.com/artifact/ai.koog/koog-agents/)
和[聊天记忆特性包](https://mvnrepository.com/artifact/ai.koog/agents-features-memory)
作为依赖：

Gradle (Kotlin)Gradle (Groovy)Maven

build.gradle.kts

```
dependencies {
    implementation("ai.koog:koog-agents:1.3.0")
    implementation("ai.koog:agents-features-memory:1.3.0")
}
```

build.gradle

```
dependencies {
    implementation 'ai.koog:koog-agents:0.7.0'
    implementation 'ai.koog:agents-features-memory:0.7.0'
}
```

pom.xml

```
<dependency>
    <groupId>ai.koog</groupId>
    <artifactId>koog-agents-jvm</artifactId>
    <version>1.3.0</version>
</dependency>
<dependency>
    <groupId>ai.koog</groupId>
    <artifactId>agents-features-memory-jvm</artifactId>
    <version>0.7.0</version>
</dependency>
```

从 LLM 提供商获取 API key，或通过 Ollama 运行本地 LLM。
更多信息请参阅[快速开始](../quickstart.md)。

本页示例假定你已经设置了 `OPENAI_API_KEY` 环境变量。

KotlinJava

```
suspend fun main() {
    val sessionId = "my-conversation"

    simpleOpenAIExecutor(System.getenv("OPENAI_API_KEY")).use { executor ->
        val agent = AIAgent(
            promptExecutor = executor,
            llmModel = OpenAIModels.Chat.GPT5_2,
            systemPrompt = "You are a helpful assistant."
        ) {
            install(ChatMemory) {
                windowSize(20) // keep only the last 20 messages
            }
        }

        while (true) {
            print("You: ")
            val input = readln().trim()
            if (input == "/bye") break
            if (input.isEmpty()) continue

            val reply = agent.run(input, sessionId)
            println("Assistant: $reply\n")
        }
    }
}
```

```
public class ExampleChatAgentOpenAI {
    public static void main(String[] args) {
        String sessionId = "my-conversation";

        try (var executor = simpleOpenAIExecutor(System.getenv("OPENAI_API_KEY"))) {
            AIAgent<String, String> agent = AIAgent.builder()
                    .promptExecutor(executor)
                    .llmModel(OpenAIModels.Chat.GPT5_2)
                    .systemPrompt("You are a helpful assistant.")
                    .install(ChatMemory.Feature, config -> {
                        config.windowSize(20); // keep only the last 20 messages
                    })
                    .build();

            Scanner scanner = new Scanner(System.in);
            while (true) {
                System.out.print("You: ");
                String input = scanner.nextLine().trim();
                if (input.equals("/bye")) break;
                if (input.isEmpty()) continue;

                String reply = agent.run(input, sessionId);
                System.out.println("Assistant: " + reply + "\n");
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

## 实现细节

`agent.run()` 的第二个参数是[会话 ID](../#会话-id)，
用于标识和区分正在进行的对话。
在我们的示例中，它是常量，因为一次只有一个对话。
在真实应用中，例如，你可以为同一用户相关的对话使用单独的唯一 ID。

agent 使用默认的[历史提供者](../#历史提供者)，
它将对话历史存储在内存中。
这意味着应用退出时历史会丢失。
在真实应用中，你应实现自定义历史提供者，
将历史持久化存储到数据库或文件中。

`windowSize(20)` [预处理器](../#预处理器)确保上下文大小受限：
agent 最多只存储最近 20 条消息。
没有它，prompt 大小可能会超过上下文限制。

## 示例会话

```
You: My name is Alice.
Assistant: Nice to meet you, Alice! How can I help you today?

You: What's my favorite color? It's blue.
Assistant: Got it — your favorite color is blue!

You: What's my name?
Assistant: Your name is Alice!
```

尽管每次交互都是一次单独的 agent run，agent 仍然正确回答了 "Your name is Alice!"，
因为 `ChatMemory` 特性在处理第三条消息之前加载了早先的交流。
