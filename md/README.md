# Koog 中文 Markdown 文档（按官方侧边栏层级）

说明：

- 这里是按官方文档左侧菜单层级整理的 Markdown 副本。
- `docs-translation/md/` 根目录下的扁平文件仍保留，用于兼容当前静态 HTML 页面中的 Markdown 链接。
- 静态网页在 `docs-translation/site/`，英文源文件在 `docs-translation/_source_en/`。

## 目录

### documentation/a2a

- [A2A 协议](documentation/a2a/037-a2a.md)
- [A2A Client](documentation/a2a/038-a2a__a2a-client.md)
- [A2A 与 Koog 集成](documentation/a2a/039-a2a__a2a-koog-integration.md)
- [A2A Server](documentation/a2a/040-a2a__a2a-server.md)

### documentation/advanced-usage

- [Agent 事件](documentation/advanced-usage/003-agent-events.md)
- [自定义节点实现](documentation/advanced-usage/006-custom-nodes.md)
- [Embeddings](documentation/advanced-usage/010-embeddings.md)
- [检索增强生成（RAG）](documentation/advanced-usage/026-retrieval-augmented-generation.md)
- [序列化](documentation/advanced-usage/027-serialization.md)
- [LLM sessions 和手动历史管理](documentation/advanced-usage/028-sessions.md)
- [流式 API](documentation/advanced-usage/032-streaming-api.md)
- [结构化输出](documentation/advanced-usage/033-structured-output.md)

### documentation/advanced-usage/subgraphs

- [自定义子图](documentation/advanced-usage/subgraphs/008-custom-subgraphs.md)
- [概述](documentation/advanced-usage/subgraphs/034-subgraphs-overview.md)

### documentation/agent-client-protocol

- [Agent Client Protocol](documentation/agent-client-protocol/002-agent-client-protocol.md)

### documentation/agents

- [Agents](documentation/agents/041-agents.md)
- [Basic agents](documentation/agents/042-agents__basic-agents.md)
- [函数式代理](documentation/agents/043-agents__functional-agents.md)
- [基于图的代理](documentation/agents/044-agents__graph-based-agents.md)
- [规划器智能体](documentation/agents/045-agents__planner-agents.md)
- [GOAP agents](documentation/agents/046-agents__planner-agents__goap-agents.md)
- [LLM-based planners](documentation/agents/047-agents__planner-agents__llm-based-planners.md)

### documentation/backend-framework-integrations

- [Ktor 集成：Koog plugin](documentation/backend-framework-integrations/016-ktor-plugin.md)
- [Spring AI 集成](documentation/backend-framework-integrations/030-spring-ai-integration.md)
- [Spring Boot 集成](documentation/backend-framework-integrations/031-spring-boot.md)

### documentation/cloud-platform-integrations

- [Amazon Bedrock AgentCore](documentation/cloud-platform-integrations/004-amazon-bedrock-agentcore.md)

### documentation/content-moderation

- [内容审核](documentation/content-moderation/005-content-moderation.md)

### documentation/features

- [功能](documentation/features/062-features.md)
- [事件处理程序](documentation/features/063-features__agent-event-handlers.md)
- [Agent 持久化](documentation/features/064-features__agent-persistence.md)
- [自定义特性](documentation/features/065-features__custom-features.md)
- [长期记忆](documentation/features/066-features__long-term-memory.md)
- [跟踪](documentation/features/067-features__tracing.md)
- [聊天记忆](documentation/features/068-features__chat-memory.md)
- [构建带记忆的聊天 agent](documentation/features/069-features__chat-memory__chat-agent-with-memory.md)
- [带记忆的聊天后端](documentation/features/070-features__chat-memory__chat-backend-with-memory.md)
- [OpenTelemetry support](documentation/features/071-features__open-telemetry.md)
- [Datadog 导出器](documentation/features/072-features__open-telemetry__opentelemetry-datadog-exporter.md)
- [Langfuse 导出器](documentation/features/073-features__open-telemetry__opentelemetry-langfuse-exporter.md)
- [W&B Weave 导出器](documentation/features/074-features__open-telemetry__opentelemetry-weave-exporter.md)

### documentation/history-compression

- [History compression](documentation/history-compression/013-history-compression.md)

### documentation/llm-parameters

- [LLM 参数](documentation/llm-parameters/017-llm-parameters.md)

### documentation/model-capabilities

- [模型能力](documentation/model-capabilities/019-model-capabilities.md)

### documentation/model-context-protocol

- [模型上下文协议](documentation/model-context-protocol/020-model-context-protocol.md)

### documentation/other

- [Koog on Slack](documentation/other/015-koog-slack-channel.md)

### documentation/overview

- [概览](documentation/overview/001-index.md)
- [术语表](documentation/overview/012-glossary.md)
- [关键特性](documentation/overview/014-key-features.md)
- [LLM providers](documentation/overview/018-llm-providers.md)
- [版本控制](documentation/overview/021-module-versioning.md)

### documentation/prompts

- [Prompts](documentation/prompts/075-prompts.md)
- [处理失败](documentation/prompts/076-prompts__handling-failures.md)
- [HTTP clients](documentation/prompts/077-prompts__http-clients.md)
- [LLM clients](documentation/prompts/078-prompts__llm-clients.md)
- [LLM 响应缓存](documentation/prompts/079-prompts__llm-response-caching.md)
- [Prompt executors](documentation/prompts/080-prompts__prompt-executors.md)
- [创建提示](documentation/prompts/081-prompts__prompt-creation.md)
- [提示缓存控制](documentation/prompts/082-prompts__prompt-creation__cache-control.md)
- [多模态内容](documentation/prompts/083-prompts__prompt-creation__multimodal-content.md)

### documentation/quickstart

- [快速开始](documentation/quickstart/025-quickstart.md)

### documentation/skills

- [Skills 用法](documentation/skills/029-skills.md)

### documentation/strategies

- [自定义策略图](documentation/strategies/007-custom-strategy-graphs.md)
- [节点之间的数据传递](documentation/strategies/009-data-transfer-between-nodes.md)
- [预定义节点和组件](documentation/strategies/022-nodes-and-components.md)
- [并行节点执行](documentation/strategies/023-parallel-node-execution.md)
- [预定义 agent 策略](documentation/strategies/024-predefined-agent-strategies.md)

### documentation/testing

- [测试](documentation/testing/035-testing.md)

### documentation/tools

- [概述](documentation/tools/084-tools.md)
- [基于注解的工具](documentation/tools/085-tools__annotation-based-tools.md)
- [内置工具](documentation/tools/086-tools__built-in-tools.md)
- [基于类的工具](documentation/tools/087-tools__class-based-tools.md)
- [ToolDescriptorSchemer](documentation/tools/088-tools__tool-descriptor-schemer.md)

### examples

- [示例](examples/011-examples.md)
- [Attachments](examples/048-examples__Attachments.md)
- [使用 Koog 构建 AI 银行助手](examples/049-examples__Banking.md)
- [使用 AWS Bedrock 和 Koog 框架构建 AI Agent](examples/050-examples__BedrockAgent.md)
- [使用 Koog 构建可调用工具的计算器 Agent](examples/051-examples__Calculator.md)
- [使用Koog框架构建人工智能棋手](examples/052-examples__Chess.md)
- [在 Kotlin Notebook 中使用 Koog 连接 Google Maps MCP：从零到海拔查询](examples/053-examples__GoogleMapsMcp.md)
- [使用 Koog 构建猜数字 Agent](examples/054-examples__Guesser.md)
- [使用 OpenTelemetry 将 Koog Agent 追踪到 Langfuse](examples/055-examples__Langfuse.md)
- [Koog 中的 OpenTelemetry：追踪你的 AI agent](examples/056-examples__OpenTelemetry.md)
- [使用 Playwright MCP 和 Koog 驱动浏览器](examples/057-examples__PlaywrightMcp.md)
- [Unity + Koog：从 Kotlin Agent 驱动你的游戏](examples/058-examples__UnityMcp.md)
- [构建一个简单的吸尘器 Agent](examples/059-examples__VaccumAgent.md)
- [Koog agents 的 Weave tracing](examples/060-examples__Weave.md)
- [使用 Bright Data 的 The Web MCP 和 Koog 进行网页抓取](examples/061-examples__WebMcpClient.md)

### why-koog

- [为什么选择 Koog](why-koog/036-why-koog.md)
