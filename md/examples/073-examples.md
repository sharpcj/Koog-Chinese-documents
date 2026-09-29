# 示例

Koog 框架提供了示例，帮助你了解如何为不同用例实现 AI Agent。
示例同时提供 **Kotlin** 和 **Java** 版本。
它们展示了可适配到你自己应用中的关键特性和模式。

浏览下面的示例，并点击链接在 GitHub 上查看源代码。

KotlinJava

| Example | Description |
| --- | --- |
| [Attachments](Attachments/) | 学习如何在 prompt 中使用结构化 Markdown 和附件。构建包含图片的 prompt，并使用 OpenAI 模型为 Instagram 帖子生成创意内容。 |
| [Banking](Banking/) | 构建一个具备路由能力的综合 AI 银行助手，可通过复杂的基于图的策略处理转账和交易分析。包含领域建模、工具创建和 Agent 组合模式。 |
| [BedrockAgent](BedrockAgent/) | 通过 Koog 框架与 AWS Bedrock 集成创建智能 AI Agent。学习如何定义自定义工具、设置 AWS Bedrock，并构建能够理解用于控制设备的自然语言命令的交互式 Agent。 |
| [Calculator](Calculator/) | 构建一个 calculator Agent，使用加、减、乘、除工具执行算术运算。演示并行工具调用、事件日志记录以及多 executor 支持（OpenAI 和 Ollama）。 |
| [Chess](Chess/) | 构建一个智能国际象棋 Agent，包含复杂领域建模、自定义工具、记忆优化技术和交互式选择。演示高级 Agent strategy、游戏状态管理和人机协作模式。 |
| [GoogleMapsMcp](GoogleMapsMcp/) | 通过 Docker 将 Koog 连接到 Google Maps MCP server。在 Kotlin Notebook 环境中发现工具、对地址进行地理编码，并使用具备真实地理 API 的 AI Agent 获取海拔数据。 |
| [Guesser](Guesser/) | 构建一个数字猜测 Agent，使用工具提出有针对性的问题来实现二分搜索策略。该 Agent 通过策略性提问高效缩小用户数字范围，并演示基于工具的交互模式。 |
| [Langfuse](Langfuse/) | 学习如何使用 OpenTelemetry 将 Koog Agent traces 导出到 Langfuse。设置环境变量、运行 Agent，并在你的 Langfuse 实例中检查 spans 和 traces，以获得全面可观测性。 |
| [MCP](https://github.com/JetBrains/koog/tree/develop/examples/src/main/kotlin/ai/koog/agents/example/mcp) | Model Context Protocol 的集成示例，包含用于地理数据的 GoogleMapsMcpClient 和用于浏览器自动化的 PlaywrightMcpClient。 |
| [Memory](https://github.com/JetBrains/koog/tree/develop/examples/simple-examples/src/main/kotlin/ai/koog/agents/example/memory) | 一个客户支持 Agent，演示 memory system 的使用。该 Agent 使用加密本地存储以及基于 subjects 和 scopes 的正确 memory 组织方式，跟踪用户对话偏好、设备诊断和组织特定信息。 |
| [OpenTelemetry](OpenTelemetry/) | 为 Koog AI Agent 添加基于 OpenTelemetry 的 tracing。学习如何将 spans 输出到控制台用于调试，并将 traces 导出到 OpenTelemetry Collector 以在 Jaeger 中查看。包含 Docker 设置和故障排查指南。 |
| [Planner](https://github.com/JetBrains/koog/tree/develop/examples/simple-examples/src/main/kotlin/ai/koog/agents/example/planner) | 一个任务规划系统，可构建带有并行和顺序执行节点的执行树，为复杂工作流动态构建执行计划。 |
| [PlaywrightMcp](PlaywrightMcp/) | 使用 Playwright MCP 和 Koog 驱动浏览器。启动 Playwright MCP server，通过 SSE 连接，并让 AI Agent 通过自然语言命令自动执行导航、接受 cookie 和 UI 交互等 Web 任务。 |
| [SimpleAPI](https://github.com/JetBrains/koog/tree/develop/examples/simple-examples/src/main/kotlin/ai/koog/agents/example/simpleapi) | 演示 chat Agent 和 basic Agent 的示例，使用简单 API 模式帮助你开始使用 Koog。 |
| [StructuredData](https://github.com/JetBrains/koog/tree/develop/examples/simple-examples/src/main/kotlin/ai/koog/agents/example/structuredoutput) | 演示基于 JSON 的结构化数据输出，包括复杂嵌套类、多态以及天气预报示例，展示如何在 Agent 响应中处理类型化数据。 |
| [SubgraphWithTask](https://github.com/JetBrains/koog/tree/develop/examples/simple-examples/src/main/kotlin/ai/koog/agents/example/subgraphwithtask) | 项目生成工具，展示文件和目录操作，包括使用 subgraph strategy 创建、删除和执行命令。 |
| [Tone](https://github.com/JetBrains/koog/tree/develop/examples/simple-examples/src/main/kotlin/ai/koog/agents/example/tone) | 一个文本语气分析 Agent，使用专门工具识别输入文本中的积极、消极或中性语气，演示情感分析能力。 |
| [UnityMcp](UnityMcp/) | 使用 Unity MCP server 集成，通过 AI Agent 驱动 Unity 游戏开发。通过 stdio 连接到 Unity，发现可用工具，并让 Agent 通过自然语言命令修改场景、放置对象和执行游戏开发任务。 |
| [VaccumAgent](VaccumAgent/) | 使用 Koog 框架实现一个基础反射型 Agent。涵盖简单双单元世界中的环境建模、工具创建以及用于自动清洁任务的 Agent 行为。 |
| [Weave](Weave/) | 学习如何使用 OpenTelemetry (OTLP) 将 Koog Agent traces 发送到 W&B Weave。设置环境变量、运行 Agent，并在 Weave UI 中查看丰富 traces，以进行全面监控和调试。 |
| [A2A](https://github.com/JetBrains/koog/tree/develop/examples/simple-examples/src/main/kotlin/ai/koog/agents/example/a2a) | 演示使用 Koog 框架进行 agent-to-agent (A2A) 通信。展示如何设置 AI Agent 之间的双向通信、启用协作式问题解决，并通过正确的消息路由和协调管理多 Agent 工作流。 |

| Example | Description |
| --- | --- |
| [Calculator](https://github.com/JetBrains/koog/tree/develop/examples/simple-examples-java/src/main/java/ai/koog/agents/example/calculator/Calculator.java) | 构建一个基于图的 calculator Agent。演示使用 `ToolSet` 定义的工具、带类型化边的多节点 graph strategy、条件路由、自动历史压缩、事件处理以及多 executor 支持（OpenAI 和 Ollama）。 |
| [FunctionalAgentChat](https://github.com/JetBrains/koog/tree/develop/examples/simple-examples-java/src/main/java/ai/koog/agents/example/chat/FunctionalAgentChat.java) | 使用 [functional strategy](../agents/functional-agents/) 构建交互式 chat Agent。运行由 Llama 3.2 模型驱动的连续对话循环，接受用户输入直到输入 `/bye`。 |
| [ChatMemoryJdbc](https://github.com/JetBrains/koog/tree/develop/examples/simple-examples-java/src/main/java/ai/koog/agents/example/chatmemory/ChatMemoryJdbcExample.java) | 使用由 JDBC PostgreSQL provider 支持的 [`ChatMemory`](../features/chat-memory/) feature，在 session 之间持久化对话历史。演示创建基于 JDBC 的 chat history provider（24h TTL）、schema migration，以及使用 chat memory 构建 Agent。需要本地或通过 Docker 运行 PostgreSQL。 |
| [FunctionalStrategy](https://github.com/JetBrains/koog/tree/develop/examples/simple-examples-java/src/main/java/ai/koog/agents/example/strategies/functional/FunctionalStrategyExample.java) | 实现带有类型化子任务、按步骤限定工具作用域，以及迭代验证和修复循环的多步骤 functional strategy，以生成经过验证的解决方案。 |
| [GoapStrategy](https://github.com/JetBrains/koog/tree/develop/examples/simple-examples-java/src/main/java/ai/koog/agents/example/strategies/GoapStrategyExample.java) | 使用 GOAP（Goal-Oriented Action Planning）构建基于 planner 的 Agent。定义类型化 belief states、action preconditions 和 goal conditions，以指导迭代式问题解决和自我纠正。 |
| [GraphStrategy](https://github.com/JetBrains/koog/tree/develop/examples/simple-examples-java/src/main/java/ai/koog/agents/example/strategies/GraphStrategyExample.java) | 构建基于图的问题解决 Agent，包含由条件边连接的类型化 subgraphs。使用 LLM-as-a-judge 实现用于验证解决方案的问题解决流水线。 |
| [CustomSubgraph](https://github.com/JetBrains/koog/tree/develop/examples/simple-examples-java/src/main/java/ai/koog/agents/example/subgraphs/CustomSubgraphExample.java) | 构建多 subgraph Agent strategy。实现 3 个顺序连接的 subgraphs，分别处理带 Web search 工具的研究、提纲规划以及撰写最终文章摘要。 |
| [OpenTelemetry](https://github.com/JetBrains/koog/tree/develop/examples/simple-examples-java/src/main/java/ai/koog/agents/example/features/opentelemetry/OpenTelemetryExample.java) | 为 Koog AI Agent 添加基于 OpenTelemetry 的 tracing。配置 console logging exporter 用于本地调试，并配置 OTLP/gRPC exporter 以在 Jaeger 中查看 spans。包含 Docker 设置。 |
| [Langfuse](https://github.com/JetBrains/koog/tree/develop/examples/simple-examples-java/src/main/java/ai/koog/agents/example/features/opentelemetry/langfuse/LangfuseExample.java) | 通过 OpenTelemetry 将 Koog Agent traces 导出到 Langfuse。演示配置 session ID 和 trace tags 等自定义属性，以增强可观测性。 |
| [Weave](https://github.com/JetBrains/koog/tree/develop/examples/simple-examples-java/src/main/java/ai/koog/agents/example/features/opentelemetry/weave/WeaveExample.java) | 使用 OpenTelemetry (OTLP) 将 Koog Agent traces 发送到 W&B Weave。设置环境变量、运行 Agent，并在 Weave UI 中查看丰富 traces。 |
| [PersistenceJdbc](https://github.com/JetBrains/koog/tree/develop/examples/simple-examples-java/src/main/java/ai/koog/agents/example/snapshot/PersistenceJdbcExample.java) | 构建一个带有 `Persistence` feature 且由 JDBC PostgreSQL provider 支持的 Agent。Persistence feature 会在每个节点执行后自动创建检查点，使 Agent 能在重启后从离开的地方继续。需要本地或通过 Docker 运行 PostgreSQL。 |