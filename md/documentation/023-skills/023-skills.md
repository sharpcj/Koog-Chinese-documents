# Skills 用法

Koog skills 允许 agent 从文件系统中发现可复用的能力包，并通过生成的 prompt 段落将它们暴露给模型。

从高层看，用法包含三部分：

1. 从一个或多个根目录发现 skills。
2. 根据发现的元数据生成 skills prompt 块。
3. 将生成的块添加到 agent 的 `system` prompt 中，并提供 agent 可用于检查文件和执行 skill 脚本的工具。

## 示例：将 skills 添加到 system prompt

```
import ai.koog.agents.core.agent.AIAgent
import ai.koog.agents.core.tools.ToolRegistry
import ai.koog.agents.ext.tool.file.ListDirectoryTool
import ai.koog.agents.ext.tool.file.ReadFileTool
import ai.koog.prompt.executor.clients.openai.OpenAIModels
import ai.koog.prompt.executor.llms.all.simpleOpenAIExecutor
import ai.koog.rag.base.files.JVMFileSystemProvider
import ai.koog.skills.discovery.discoverSkills
import ai.koog.skills.prompt.SkillsPromptFormat
import ai.koog.skills.prompt.generateSkillsPrompt
import kotlinx.coroutines.runBlocking

fun main() = runBlocking {
    val skillsRoot = "/absolute/path/to/skills"
    val discoveredSkills = discoverSkills(JVMFileSystemProvider.ReadOnly, listOf(skillsRoot))
    val generatedSkillsPrompt = generateSkillsPrompt(discoveredSkills, SkillsPromptFormat.XML)

    // Replace with your script execution tool implementation.
    val apiKey = System.getenv("OPENAI_API_KEY")
        ?: error("The API key is not set.")

    val agent = AIAgent(
        promptExecutor = simpleOpenAIExecutor(System.getenv("YOUR_API_KEY")),
        systemPrompt = """
                You are a careful assistant.
                Use the available skills listed below.
                Before using a skill script, disclose the skills by listing and reading files with tools.

                $generatedSkillsPrompt
                """.trimIndent(),
        llmModel = OpenAIModels.Chat.GPT4o,
        toolRegistry = ToolRegistry {
            tool(ListDirectoryTool(JVMFileSystemProvider.ReadOnly))
            tool(ReadFileTool(JVMFileSystemProvider.ReadOnly))
            // Additional tools...
        },
    )
}
```

## 必需组成部分

- `discoverSkills(...)` 会扫描配置的目录并返回发现的 skill 描述符。
- `generateSkillsPrompt(...)` 会把发现的 skills 转换为 prompt 文本（`SkillsPromptFormat.XML` 是常见选择）。
- 生成的文本应嵌入到 agent 的 `system` prompt 中，以便模型能够推理可用 skills。
- 工具注册表必须包含你的工作流所需的工具，通常包括：
- 文件发现/读取工具（用于透明地披露 skill），
- 一个或多个用于执行 skill 脚本的执行工具。

## 行为预期

当存在 skills prompt 且注册了匹配工具时，agent 可以：

- 发现 skill 文件，
- 读取 skill 定义，
- 针对相关任务运行执行工具（例如，使用适当参数运行 python 脚本）。

详情请参阅 [Agent Skills](https://agentskills.io/home) 文档。

## 实用提示

- 将 skills 保存在专用目录中，并在相对根目录可能变化的运行环境中传入绝对路径。
- 当 skills 是静态的时，使用只读文件提供者进行发现（例如 `JVMFileSystemProvider.ReadOnly`）。
- 保持脚本执行工具范围狭窄且类型安全（结构化参数/结果），并校验脚本路径处理。
