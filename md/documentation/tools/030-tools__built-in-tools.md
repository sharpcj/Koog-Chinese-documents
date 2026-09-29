# 内置工具

Koog 为 Kotlin 和 Java 提供内置工具，帮助你快速进行原型设计并试验智能体与用户的交互。
这些工具并非用于生产环境。要使用它们，请将 `ai.koog:agents-ext` 添加到依赖项中。
可用的内置工具如下：

| Tool | Name | Description |
| --- | --- | --- |
| SayToUser | `__say_to_user__` | 允许智能体向用户发送消息。它会使用 `Agent says:` 前缀将智能体消息打印到控制台。 |
| AskUser | `__ask_user__` | 允许智能体向用户请求输入。它会将智能体消息打印到控制台，并等待用户响应。 |
| ExitTool | `__exit__` | 允许智能体结束对话并终止会话。 |
| ReadFileTool | `__read_file__` | 读取文本文件，并可选择行范围。使用基于 0 的行索引返回带元数据的格式化内容。 |
| EditFileTool | `__edit_file__` | 在文件中执行一次有针对性的文本替换；也可以创建新文件或完整替换内容。 |
| ListDirectoryTool | `__list_directory__` | 以分层树的形式列出目录内容，并可选择控制深度和进行 glob 过滤。 |
| WriteFileTool | `__write_file__` | 将文本内容写入文件（必要时创建父目录）。 |

## 注册内置工具

与任何其他工具一样，内置工具必须添加到工具注册表中，才能供智能体使用。示例如下：

```
// Create a tool registry with all built-in tools
val toolRegistry = ToolRegistry {
    tool(SayToUser)
    tool(AskUser)
    tool(ExitTool)
    tool(ReadFileTool(JVMFileSystemProvider.ReadOnly))
    tool(ListDirectoryTool(JVMFileSystemProvider.ReadOnly))
    tool(WriteFileTool(JVMFileSystemProvider.ReadWrite))
}

// Pass the registry when creating an agent
val agent = AIAgent(
    promptExecutor = simpleOpenAIExecutor(apiToken),
    systemPrompt = "You are a helpful assistant.",
    llmModel = OpenAIModels.Chat.GPT4o,
    toolRegistry = toolRegistry
)
```

你可以在 Kotlin 和 Java 中，将内置工具与自定义工具组合到同一个注册表中，为智能体创建一组完整的能力。
要了解有关自定义工具的更多信息，请参阅 [Annotation-based tools](../annotation-based-tools/) 和 [Class-based tools](../class-based-tools/)。