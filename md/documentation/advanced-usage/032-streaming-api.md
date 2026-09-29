# 流式 API

Koog 的 **Streaming API** 让你能够以 `Flow<StreamFrame>`（Kotlin）/ `Flow.Publisher<StreamFrame>`（Java）的形式**增量消费 LLM 输出**。
无需等待完整响应，你的代码可以：

- 在助手文本到达时实时渲染，
- 实时检测**工具调用**并对其做出响应，
- 了解流何时**结束**以及结束原因。

流携带**类型化的帧**，分为两类：

KotlinJava

**增量帧**（增量/部分内容）：

- `StreamFrame.TextDelta(text: String, index: Int?)` — 增量助手文本
- `StreamFrame.ReasoningDelta(text: String?, summary: String?, index: Int?)` — 增量推理文本和摘要
- `StreamFrame.ToolCallDelta(id: String?, name: String?, content: String?, index: Int?)` — 部分工具调用

**完整帧**（完整内容）：

- `StreamFrame.TextComplete(text: String, index: Int?)` — 完整助手文本
- `StreamFrame.ReasoningComplete(content: List<String>, summary: List<String>?, encrypted: String?, index: Int?)` — 完整推理，包含可选摘要和加密内容
- `StreamFrame.ToolCallComplete(id: String?, name: String, content: String, index: Int?)` — 完整工具调用

**结束标记**：

- `StreamFrame.End(finishReason: String?, metaInfo: ResponseMetaInfo)` — 流结束标记，包含响应元数据

**增量帧**（增量/部分内容）：

- `StreamFrame.TextDelta` — 增量助手文本。字段：`getText()`、`getIndex()`。
- `StreamFrame.ReasoningDelta` — 增量推理文本和摘要。字段：`getText()`、`getSummary()`、`getIndex()`。
- `StreamFrame.ToolCallDelta` — 部分工具调用。字段：`getId()`、`getName()`、`getContent()`、`getIndex()`。

**完整帧**（完整内容）：

- `StreamFrame.TextComplete` — 完整助手文本。字段：`getText()`、`getIndex()`。
- `StreamFrame.ReasoningComplete` — 完整推理，包含可选摘要和加密内容。字段：`getText()`（返回 `List<String>`）、`getSummary()`（返回 `List<String>`）、`getEncrypted()`、`getIndex()`。
- `StreamFrame.ToolCallComplete` — 完整工具调用。字段：`getId()`、`getName()`、`getContent()`、`getIndex()`。还提供 `getContentJson()` 和 `getContentJsonResult()` 用于 JSON 解析。

**结束标记**：

- `StreamFrame.End` — 流结束标记。字段：`getFinishReason()`、`getMetaInfo()`。

提供了辅助方法用于提取纯文本、将帧转换为 `Message.Response` 对象，以及安全地**合并分块的工具调用**。

## API 概览

通过流式处理，你可以：

- 在数据到达时进行处理（提升 UI 响应性）
- 实时解析结构化信息（Markdown/JSON 等）
- 在对象完成时发出对象
- 实时触发工具
- 实时访问模型推理（适用于支持的模型）

你可以直接操作**帧**本身，也可以操作从帧派生的**纯文本**。

### 增量帧与完整帧

流式 API 区分两种类型的帧：

- **增量帧**（`DeltaFrame`）— 以分块形式到达的增量/部分内容。这些非常适合在内容流式传输时进行实时显示。例如：`TextDelta`、`ReasoningDelta`、`ToolCallDelta`。
- **完整帧**（`CompleteFrame`）— 在该内容类型的所有增量帧接收完毕后发出的完整内容。这些适用于最终处理和转换为 `Message.Response` 对象。例如：`TextComplete`、`ReasoningComplete`、`ToolCallComplete`。

通常，你会使用增量帧进行 UI 更新，使用完整帧提取最终结构化数据。

---

## 用法

### 直接操作帧

这是最通用的方法：对每种帧类型做出响应。

KotlinJava

```
llm.writeSession {
    appendPrompt { user("Tell me a joke, then call a tool with JSON args.") }

    val stream = requestLLMStreaming() // Flow<StreamFrame>

    stream.collect { frame ->
        when (frame) {
            is StreamFrame.TextDelta -> print(frame.text)
            is StreamFrame.ReasoningDelta -> print("[Reasoning] text=${frame.text} summary=${frame.summary}")
            is StreamFrame.ToolCallComplete -> {
                println("\n🔧 Tool call: ${frame.name} args=${frame.content}")
                // Optionally parse lazily:
                // val json = frame.contentJson
            }
            is StreamFrame.End -> println("\n[END] reason=${frame.finishReason}")
            else -> {} // Handle other frame types (TextComplete, ToolCallDelta, etc.)
        }
    }
}
```

```
ctx.getLlm().writeSession(session -> {
    session.appendPrompt(prompt -> {
        prompt.user("Tell me a joke, then call a tool with JSON args.");
        return null;
    });

    Flow.Publisher<StreamFrame> stream = session.requestLLMStreaming();

    stream.subscribe(new Flow.Subscriber<>() {
        @Override
        public void onSubscribe(Flow.Subscription subscription) {
            subscription.request(Long.MAX_VALUE);
        }

        @Override
        public void onNext(StreamFrame frame) {
            if (frame instanceof StreamFrame.TextDelta delta) {
                System.out.print(delta.getText());
            } else if (frame instanceof StreamFrame.ReasoningDelta reasoning) {
                System.out.print("[Reasoning] text=" + reasoning.getText()
                    + " summary=" + reasoning.getSummary());
            } else if (frame instanceof StreamFrame.ToolCallComplete toolCall) {
                System.out.println("\nTool call: " + toolCall.getName()
                    + " args=" + toolCall.getContent());
            } else if (frame instanceof StreamFrame.End end) {
                System.out.println("\n[END] reason=" + end.getFinishReason());
            }
            // Handle other frame types (TextComplete, ToolCallDelta, etc.)
        }

        @Override
        public void onError(Throwable throwable) {
            System.err.println("Stream error: " + throwable.getMessage());
        }

        @Override
        public void onComplete() {
        }
    });

    return null;
});
```

需要注意的是，你可以通过直接操作原始字符串流来解析输出。
这种方法为解析过程提供了更大的灵活性和控制力。

以下是一个原始字符串流，包含输出结构的 Markdown 定义：

KotlinJava

```
fun markdownBookDefinition(): MarkdownStructureDefinition {
    return MarkdownStructureDefinition("name", schema = { /*...*/ })
}

val mdDefinition = markdownBookDefinition()

llm.writeSession {
    val stream = requestLLMStreaming(mdDefinition)
    // Access the raw string chunks directly
    stream.collect { chunk ->
        // Process each chunk of text as it arrives
        println("Received chunk: $chunk") // The chunks together will be structured as a text following the mdDefinition schema
    }
}
```

```
StructureDefinition mdDefinition = markdownBookDefinition();

ctx.getLlm().writeSession(session -> {
    session.appendPrompt(prompt -> {
        prompt.user(input);
    });

    Flow.Publisher<StreamFrame> stream = session.requestLLMStreaming(mdDefinition);

    // Access the raw frames directly
    stream.subscribe(new Flow.Subscriber<>() {
        @Override
        public void onSubscribe(Flow.Subscription subscription) {
            subscription.request(Long.MAX_VALUE);
        }

        @Override
        public void onNext(StreamFrame frame) {
            // Process each frame as it arrives
            System.out.println("Received frame: " + frame);
        }

        @Override
        public void onError(Throwable throwable) {
            System.err.println("Stream error: " + throwable.getMessage());
        }

        @Override
        public void onComplete() {
        }
    });

    return null;
});
```

### 操作推理帧

支持推理的模型（如 Claude Sonnet 4.5 或 GPT-o1）在流式传输期间会发出推理帧。你可以同时访问推理过程及其摘要：

KotlinJava

```
llm.writeSession {
    appendPrompt { user("Solve this complex problem: ...") }

    val stream = requestLLMStreaming()
    val reasoningSteps = mutableListOf<String>()
    val summarySteps = mutableListOf<String>()

    stream.collect { frame ->
        when (frame) {
            is StreamFrame.ReasoningDelta -> {
                frame.text?.let { 
                    reasoningSteps.add(it)
                    print(frame.text) // Display reasoning as it arrives
                }
                frame.summary?.let {
                    summarySteps.add(it)
                    print(frame.summary) // Display reasoning summary as it arrives
                }
            }
            is StreamFrame.ReasoningComplete -> {
                // Access complete reasoning
                println("\nComplete reasoning: ${frame.content.joinToString("")}")
                println("Summary: ${frame.summary?.joinToString("") ?: "N/A"}")
            }
            is StreamFrame.TextDelta -> print(frame.text)
            is StreamFrame.End -> println("\n[END]")
            else -> {}
        }
    }
}
```

```
ctx.getLlm().writeSession(session -> {
    session.appendPrompt(prompt -> {
        prompt.user("Solve this complex problem: ...");
        return null;
    });

    Flow.Publisher<StreamFrame> stream = session.requestLLMStreaming();
    List<String> reasoningSteps = new ArrayList<>();
    List<String> summarySteps = new ArrayList<>();

    stream.subscribe(new Flow.Subscriber<StreamFrame>() {
        @Override
        public void onSubscribe(Flow.Subscription subscription) {
            subscription.request(Long.MAX_VALUE);
        }

        @Override
        public void onNext(StreamFrame frame) {
            if (frame instanceof StreamFrame.ReasoningDelta reasoning) {
                if (reasoning.getText() != null) {
                    reasoningSteps.add(reasoning.getText());
                    System.out.print(reasoning.getText());
                }
                if (reasoning.getSummary() != null) {
                    summarySteps.add(reasoning.getSummary());
                    System.out.print(reasoning.getSummary());
                }
            } else if (frame instanceof StreamFrame.ReasoningComplete complete) {
                // Access complete reasoning
                System.out.println("\nComplete reasoning: "
                    + String.join("", complete.getContent()));
                System.out.println("Summary: "
                    + (complete.getSummary() != null
                        ? String.join("", complete.getSummary()) : "N/A"));
            } else if (frame instanceof StreamFrame.TextDelta delta) {
                System.out.print(delta.getText());
            } else if (frame instanceof StreamFrame.End) {
                System.out.println("\n[END]");
            }
        }

        @Override
        public void onError(Throwable throwable) { }

        @Override
        public void onComplete() { }
    });

    return null;
});
```

### 操作原始文本流（派生）

如果你已有期望 `Flow<String>` 的流式解析器，
可以通过 `filterTextOnly()` 派生文本块，或使用 `collectText()` 收集它们。

KotlinJava

```
llm.writeSession {
    val frames = requestLLMStreaming()

    // Stream text chunks as they come:
    frames.filterTextOnly().collect { chunk -> print(chunk) }

    // Or, gather all text into one String after End:
    val fullText = frames.collectText()
    println("\n---\n$fullText")
}
```

```
ctx.getLlm().writeSession(session -> {
    Flow.Publisher<StreamFrame> frames = session.requestLLMStreaming();

    // Stream text chunks as they come (equivalent of filterTextOnly):
    StringBuilder fullText = new StringBuilder();
    frames.subscribe(new Flow.Subscriber<>() {
        @Override
        public void onSubscribe(Flow.Subscription subscription) {
            subscription.request(Long.MAX_VALUE);
        }

        @Override
        public void onNext(StreamFrame frame) {
            if (frame instanceof StreamFrame.TextDelta delta) {
                System.out.print(delta.getText());
                fullText.append(delta.getText());
            }
        }

        @Override
        public void onError(Throwable throwable) { }

        @Override
        public void onComplete() {
            // fullText now contains all text (equivalent of collectText)
            System.out.println("\n---\n" + fullText);
        }
    });

    return null;
});
```

### 在事件处理器中监听流事件

你可以在[智能体事件处理器](../features/agent-event-handlers/)中监听流事件。

KotlinJava

```
handleEvents {
    onToolCallStarting { context ->
        println("\n🔧 Using ${context.toolName} with ${context.toolArgs}... ")
    }

    onLLMStreamingFrameReceived { context ->
        when (val frame = context.streamFrame) {
            is StreamFrame.TextDelta -> print(frame.text)
            is StreamFrame.ReasoningDelta -> print("[Reasoning] text=${frame.text} summary=${frame.summary}")
            else -> {} // Handle other frame types if needed
        }
    }

    onLLMStreamingFailed { context ->
        println("❌ Error: ${context.error}")
    }

    onLLMStreamingCompleted {
        println("🏁 Done")
    }
}
```

```
.install(EventHandler.Feature, config -> {
    config.onToolCallStarting(ctx -> {
        System.out.println("\nUsing " + ctx.getToolName() + " with " + ctx.getToolArgs() + "... ");
    });

    config.onLLMStreamingFrameReceived(ctx -> {
        StreamFrame frame = ctx.getStreamFrame();
        if (frame instanceof StreamFrame.TextDelta delta) {
            System.out.print(delta.getText());
        } else if (frame instanceof StreamFrame.ReasoningDelta reasoning) {
            System.out.print("[Reasoning] text=" + reasoning.getText()
                + " summary=" + reasoning.getSummary());
        }
    });

    config.onLLMStreamingFailed(ctx -> {
        System.out.println("Error: " + ctx.getError());
    });

    config.onLLMStreamingCompleted(ctx -> {
        System.out.println("Done");
    });
})
```

### 将帧转换为 `Message.Response`

你可以将收集到的帧列表转换为标准消息对象：

- `toAssistantMessageOrNull()` — 从文本帧中提取 `Message.Assistant`
- `toReasoningMessageOrNull()` — 从推理帧中提取 `MessagePart.Reasoning`
- `toToolCallMessages()` — 从工具调用帧中提取 `MessagePart.Tool.Call`
- `toMessageResponses()` — 将所有完整帧转换为对应的 `Message.Response` 对象

## 示例

### 流式传输中的结构化数据（Markdown 示例）

虽然可以操作原始字符串流，
但操作[结构化数据](../structured-output/)通常更为方便。

结构化数据方法包含以下关键组件：

1. **MarkdownStructureDefinition**：一个帮助你以 Markdown 格式定义结构化数据模式和示例的类。
2. **markdownStreamingParser**：一个创建解析器的函数，该解析器处理 Markdown 块流并发出事件。

以下各节提供了与处理结构化数据流相关的分步说明和代码示例。

#### 1. 定义数据结构

首先，定义一个数据类来表示你的结构化数据：

KotlinJava

```
@Serializable
data class Book(
    val title: String,
    val author: String,
    val description: String
)
```

```
// TODO not yet supported in Java
```

#### 2. 定义 Markdown 结构

使用 `MarkdownStructureDefinition` 类创建一个定义，指定数据在 Markdown 中的结构：

KotlinJava

```
fun markdownBookDefinition(): MarkdownStructureDefinition {
    return MarkdownStructureDefinition("bookList", schema = {
        markdown {
            header(1, "title")
            bulleted {
                item("author")
                item("description")
            }
        }
    }, examples = {
        markdown {
            header(1, "The Great Gatsby")
            bulleted {
                item("F. Scott Fitzgerald")
                item("A novel set in the Jazz Age that tells the story of Jay Gatsby's unrequited love for Daisy Buchanan.")
            }
        }
    })
}
```

```
// TODO not yet supported in Java
```

#### 3. 为数据结构创建解析器

`markdownStreamingParser` 为不同的 Markdown 元素提供了多个处理器：

KotlinJava

```
markdownStreamingParser {
    // Handle level 1 headings (level ranges from 1 to 6)
    onHeader(1) { headerText -> }
    // Handle bullet points
    onBullet { bulletText -> }
    // Handle code blocks
    onCodeBlock { codeBlockContent -> }
    // Handle lines matching a regex pattern
    onLineMatching(Regex("pattern")) { line -> }
    // Handle the end of the stream
    onFinishStream { remainingText -> }
}
```

```
// TODO not yet supported in Java
```

使用已定义的处理器，你可以实现一个函数，通过 `markdownStreamingParser` 函数解析 Markdown 流并发出数据对象。

KotlinJava

```
fun parseMarkdownStreamToBooks(markdownStream: Flow<StreamFrame>): Flow<Book> {
   return flow {
      markdownStreamingParser {
         var currentBookTitle = ""
         val bulletPoints = mutableListOf<String>()

         // Handle the event of receiving the Markdown header in the response stream
         onHeader(1) { headerText ->
            // If there was a previous book, emit it
            if (currentBookTitle.isNotEmpty() && bulletPoints.isNotEmpty()) {
               val author = bulletPoints.getOrNull(0) ?: ""
               val description = bulletPoints.getOrNull(1) ?: ""
               emit(Book(currentBookTitle, author, description))
            }

            currentBookTitle = headerText
            bulletPoints.clear()
         }

         // Handle the event of receiving the Markdown bullets list in the response stream
         onBullet { bulletText ->
            bulletPoints.add(bulletText)
         }

         // Handle the end of the response stream
         onFinishStream {
            // Emit the last book, if present
            if (currentBookTitle.isNotEmpty() && bulletPoints.isNotEmpty()) {
               val author = bulletPoints.getOrNull(0) ?: ""
               val description = bulletPoints.getOrNull(1) ?: ""
               emit(Book(currentBookTitle, author, description))
            }
         }
      }.parseStream(markdownStream.filterTextOnly())
   }
}
```

```
// TODO not yet supported in Java
```

#### 4. 在智能体策略中使用解析器

KotlinJava

```
val agentStrategy = strategy<String, List<Book>>("library-assistant") {
   // Describe the node containing the output stream parsing
   val getMdOutput by node<String, List<Book>> { booksDescription ->
      val books = mutableListOf<Book>()
      val mdDefinition = markdownBookDefinition()

      llm.writeSession {
         appendPrompt { user(booksDescription) }
         // Initiate the response stream in the form of the definition `mdDefinition`
         val markdownStream = requestLLMStreaming(mdDefinition)
         // Call the parser with the result of the response stream and perform actions with the result
         parseMarkdownStreamToBooks(markdownStream).collect { book ->
            books.add(book)
            println("Parsed Book: ${book.title} by ${book.author}")
         }
      }

      books
   }
   // Describe the agent's graph making sure the node is accessible
   edge(nodeStart forwardTo getMdOutput)
   edge(getMdOutput forwardTo nodeFinish)
}
```

```
// TODO not yet supported in Java
```

### 高级用法：结合工具进行流式处理

你还可以将 Streaming API 与工具结合使用，在数据到达时进行处理。
以下各节提供了关于如何定义工具并将其与流式数据结合使用的简要分步指南。

### 1. 为数据结构定义工具

KotlinJava

```
@Serializable
data class Book(
   val title: String,
   val author: String,
   val description: String
)

class BookTool(): SimpleTool<Book>(
    argsType = typeToken<Book>(),
    name = NAME,
    description = "A tool to parse book information from Markdown"
) {

    companion object { const val NAME = "book" }

    override suspend fun execute(args: Book): String {
        println("${args.title} by ${args.author}:\n ${args.description}")
        return "Done"
    }
}
```

```
class BookTool implements ToolSet {
    @Tool
    @LLMDescription("A tool to parse book information from Markdown")
    public String book(
        @LLMDescription("Title of the book") String title,
        @LLMDescription("Author of the book") String author,
        @LLMDescription("Description of the book") String description
    ) {
        System.out.println(title + " by " + author + ":\n " + description);
        return "Done";
    }
}
```

### 2. 将工具与流式数据结合使用

KotlinJava

```
val agentStrategy = strategy<String, Unit>("library-assistant") {
   val getMdOutput by node<String, Unit> { input ->
      val mdDefinition = markdownBookDefinition()

      llm.writeSession {
         appendPrompt { user(input) }
         val markdownStream = requestLLMStreaming(mdDefinition)

         parseMarkdownStreamToBooks(markdownStream).collect { book ->
            callToolRaw(BookTool.NAME, book)
            /* Other possible options:
                callTool(BookTool::class, book)
                callTool<BookTool>(book)
                findTool(BookTool::class).execute(book)
            */
         }

         // We can make parallel tool calls
         parseMarkdownStreamToBooks(markdownStream).toParallelToolCallsRaw(toolClass=BookTool::class).collect {
            println("Tool call result: $it")
         }
      }
   }

   edge(nodeStart forwardTo getMdOutput)
   edge(getMdOutput forwardTo nodeFinish)
 }
```

```
var strategy = AIAgentGraphStrategy.builder("library-assistant")
    .withInput(String.class)
    .withOutput(Void.class);

var getMdOutput = AIAgentNode.builder("getMdOutput")
    .withInput(String.class)
    .withOutput(Void.class)
    .withAction((input, ctx) -> {
        StructureDefinition mdDefinition = markdownBookDefinition();

        ctx.getLlm().writeSession(session -> {
            session.appendPrompt(prompt -> {
                prompt.user(input);
                return null;
            });

            Flow.Publisher<StreamFrame> markdownStream = session.requestLLMStreaming(mdDefinition);

            // Process streamed frames and invoke tools on ToolCallComplete frames
            markdownStream.subscribe(new Flow.Subscriber<StreamFrame>() {
                @Override
                public void onSubscribe(Flow.Subscription subscription) {
                    subscription.request(Long.MAX_VALUE);
                }

                @Override
                public void onNext(StreamFrame frame) {
                    if (frame instanceof StreamFrame.ToolCallComplete toolCall) {
                        System.out.println("Tool call: " + toolCall.getName()
                            + " args=" + toolCall.getContent());
                    }
                }

                @Override
                public void onError(Throwable throwable) { }

                @Override
                public void onComplete() { }
            });

            return null;
        });

        return null;
    })
    .build();

strategy.edge(strategy.nodeStart, getMdOutput);
strategy.edge(getMdOutput, strategy.nodeFinish);
```

### 3. 在智能体配置中注册工具

KotlinJava

```
val toolRegistry = ToolRegistry {
    tool(BookTool())
}

val runner = AIAgent(
    promptExecutor = simpleOpenAIExecutor("OPENAI_API_KEY"),
    llmModel = OpenAIModels.Chat.GPT4o,
    toolRegistry = toolRegistry
)
```

```
ToolRegistry toolRegistry = ToolRegistry.builder()
    .tools(new BookTool())
    .build();

AIAgent<String, String> runner = AIAgent.<String, String>builder()
    .promptExecutor(PromptExecutor.builder().openAI("OPENAI_API_KEY").build())
    .llmModel(OpenAIModels.Chat.GPT4o)
    .toolRegistry(toolRegistry)
    .build();
```

## 最佳实践

1. **定义清晰的结构**：为你的数据创建清晰明确的 Markdown 结构。
2. **提供良好的示例**：在 `MarkdownStructureDefinition` 中包含全面的示例以指导 LLM。
3. **处理不完整数据**：在解析流中的数据时，始终检查 null 或空值。
4. **清理资源**：使用 `onFinishStream` 处理器清理资源并处理任何剩余数据。
5. **处理错误**：为格式错误的 Markdown 或意外数据实现适当的错误处理。
6. **测试**：使用各种输入场景测试你的解析器，包括部分块和格式错误的输入。
7. **并行处理**：对于独立的数据项，考虑使用并行工具调用以获得更好的性能。
