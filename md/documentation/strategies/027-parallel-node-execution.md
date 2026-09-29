# 并行节点执行

## 概述

并行节点执行允许你同时运行多个 AI agent 节点，从而提升性能并支持复杂工作流。此功能在你需要执行以下操作时特别有用：

- 通过不同模型或方法同时处理同一输入
- 并行执行多个相互独立的操作
- 实现竞争式评估模式：生成多个解决方案后再进行比较

## 关键组件

Koog 中的并行节点执行由下述方法和数据结构组成。

### 方法

- `parallel()`：并行执行多个节点并收集它们的结果。

### 数据结构

- `ParallelResult`：表示一次并行节点执行完成后的结果。
- `NodeExecutionResult`：包含某个节点执行的输出和上下文。

## 基本用法

### 并行运行节点

要启动节点的并行执行，请按以下格式使用 `parallel` 方法：

```
val nodeName by parallel<Input, Output>(
   firstNode, secondNode, thirdNode /* Add more nodes if needed */
) {
   // Merge strategy goes here, for example: 
   selectByMax { it.length }
}
```

下面是一个实际示例：并行运行三个节点，并选择长度最大的结果：

```
val calc by parallel<String, Int>(
   nodeCalcTokens, nodeCalcSymbols, nodeCalcWords,
) {
   selectByMax { it }
}
```

以上代码会并行运行 `nodeCalcTokens`、`nodeCalcSymbols` 和 `nodeCalcWords` 节点，并返回值最大的结果。

### 合并策略

并行执行节点后，你需要指定如何合并结果。Koog 提供以下合并策略：

- `selectBy()`：根据谓词函数选择一个结果。
- `selectByMax()`：根据比较函数选择具有最大值的结果。
- `selectByIndex()`：根据选择函数返回的索引选择一个结果。
- `fold()`：使用操作函数将结果折叠为单个值。

#### selectBy

根据谓词函数选择一个结果：

```
val nodeSelectJoke by parallel<String, String>(
   nodeOpenAI, nodeAnthropicSonnet, nodeAnthropicOpus,
) {
   selectBy { it.contains("programmer") }
}
```

这会选择第一个包含单词 "programmer" 的笑话。

#### selectByMax

根据比较函数选择具有最大值的结果：

```
val nodeLongestJoke by parallel<String, String>(
   nodeOpenAI, nodeAnthropicSonnet, nodeAnthropicOpus,
) {
   selectByMax { it.length }
}
```

这会选择长度最大的笑话。

#### selectByIndex

根据选择函数返回的索引选择一个结果：

```
val nodeBestJoke by parallel<String, String>(
   nodeOpenAI, nodeAnthropicSonnet, nodeAnthropicOpus,
) {
   selectByIndex { jokes ->
      // Use another LLM to determine the best joke
      llm.writeSession {
         model = OpenAIModels.Chat.GPT4o
         appendPrompt {
            system("You are a comedy critic. Select the best joke.")
            user("Here are three jokes: ${jokes.joinToString("\n\n")}")
         }
         val response = requestLLMStructured<JokeRating>()
         response.getOrNull()!!.data.bestJokeIndex
      }
   }
}
```

这会使用另一次 LLM 调用来确定最佳笑话的索引。

#### fold

使用操作函数将结果折叠为单个值：

```
val nodeAllJokes by parallel<String, String>(
   nodeOpenAI, nodeAnthropicSonnet, nodeAnthropicOpus,
) {
   fold("Jokes:\n") { result, joke -> "$result\n$joke" }
}
```

这会将所有笑话合并为一个字符串。

## 示例：最佳笑话 agent

下面是一个完整示例，它使用并行执行从不同 LLM 模型生成笑话，并选择最佳笑话：

```
val strategy = strategy("best-joke") {
   // Define nodes for different LLM models
   val nodeOpenAI by node<String, String> { topic ->
      llm.writeSession {
         model = OpenAIModels.Chat.GPT4o
         appendPrompt {
            system("You are a comedian. Generate a funny joke about the given topic.")
            user("Tell me a joke about $topic.")
         }
         val response = requestLLMWithoutTools()
         response.parts.filterIsInstance<MessagePart.Text>().joinToString("\n") { it.text }
      }
   }

   val nodeAnthropicSonnet by node<String, String> { topic ->
      llm.writeSession {
         model = AnthropicModels.Sonnet_4_5
         appendPrompt {
            system("You are a comedian. Generate a funny joke about the given topic.")
            user("Tell me a joke about $topic.")
         }
         val response = requestLLMWithoutTools()
         response.parts.filterIsInstance<MessagePart.Text>().joinToString("\n") { it.text }
      }
   }

   val nodeAnthropicOpus by node<String, String> { topic ->
      llm.writeSession {
         model = AnthropicModels.Opus_4_6
         appendPrompt {
            system("You are a comedian. Generate a funny joke about the given topic.")
            user("Tell me a joke about $topic.")
         }
         val response = requestLLMWithoutTools()
         response.parts.filterIsInstance<MessagePart.Text>().joinToString("\n") { it.text }
      }
   }

   // Execute joke generation in parallel and select the best joke
   val nodeGenerateBestJoke by parallel(
      nodeOpenAI, nodeAnthropicSonnet, nodeAnthropicOpus,
   ) {
      selectByIndex { jokes ->
         // Another LLM (e.g., GPT4o) would find the funniest joke:
         llm.writeSession {
            model = OpenAIModels.Chat.GPT4o
            appendPrompt {
               prompt("best-joke-selector") {
                  system("You are a comedy critic. Give a critique for the given joke.")
                  user(
                     """
                            Here are three jokes about the same topic:

                            ${jokes.mapIndexed { index, joke -> "Joke $index:\n$joke" }.joinToString("\n\n")}

                            Select the best joke and explain why it's the best.
                            """.trimIndent()
                  )
               }
            }

            val response = requestLLMStructured<JokeRating>()
            val bestJoke = response.getOrNull()!!.data
            bestJoke.bestJokeIndex
         }
      }
   }

   // Connect the nodes
   nodeStart then nodeGenerateBestJoke then nodeFinish
}
```

## 最佳实践

1. **考虑资源约束**：并行执行节点时要注意资源使用，尤其是在同时发起多个 LLM API 调用时。
2. **上下文管理**：每次并行执行都会创建一个分叉上下文。合并结果时，需要选择保留哪个上下文，或如何组合来自不同执行的上下文。
3. **针对你的用例优化**：

   - 对于竞争式评估（如笑话示例），使用 `selectByIndex` 选择最佳结果
   - 对于查找最大值，使用 `selectByMax`
   - 对于基于条件的过滤，使用 `selectBy`
   - 对于聚合，使用 `fold` 将所有结果组合成复合输出

## 性能注意事项

并行执行可以显著提升吞吐量，但也会带来一些开销：

- 每个并行节点都会创建一个新的协程
- 上下文分叉和合并会增加一些计算成本
- 大量并行执行时可能会发生资源争用

为获得最佳性能，请并行化满足以下条件的操作：

- 彼此独立
- 执行时间较长
- 不共享可变状态
