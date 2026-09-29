# Koog 中文 Markdown 文档（按官方侧边栏顺序编号）

说明：

- 这里是按官方文档左侧菜单顺序整理并编号的 Markdown 副本。
- 文件名前缀 `001-`、`002-` 等对应官方导航顺序；官方导航未显示但 sitemap 中存在的页面放在末尾。

## 目录

### documentation/overview

- [概览](documentation/overview/001-index.md)
- [关键特性](documentation/overview/002-key-features.md)
- [版本控制](documentation/overview/003-module-versioning.md)
- [LLM providers](documentation/overview/004-llm-providers.md)
- [术语表](documentation/overview/005-glossary.md)

### documentation/quickstart

- [快速开始](documentation/quickstart/006-quickstart.md)

### documentation/agents

- [Agents](documentation/agents/007-agents.md)
- [Basic agents](documentation/agents/008-agents__basic-agents.md)
- [基于图的代理](documentation/agents/009-agents__graph-based-agents.md)
- [函数式代理](documentation/agents/010-agents__functional-agents.md)
- [规划器智能体](documentation/agents/011-agents__planner-agents.md)
- [LLM-based planners](documentation/agents/012-agents__planner-agents__llm-based-planners.md)
- [GOAP agents](documentation/agents/013-agents__planner-agents__goap-agents.md)

### documentation/prompts

- [Prompts](documentation/prompts/014-prompts.md)
- [创建提示](documentation/prompts/015-prompts__prompt-creation.md)
- [多模态内容](documentation/prompts/016-prompts__prompt-creation__multimodal-content.md)
- [提示缓存控制](documentation/prompts/017-prompts__prompt-creation__cache-control.md)
- [LLM clients](documentation/prompts/018-prompts__llm-clients.md)
- [HTTP clients](documentation/prompts/019-prompts__http-clients.md)
- [Prompt executors](documentation/prompts/020-prompts__prompt-executors.md)
- [处理失败](documentation/prompts/021-prompts__handling-failures.md)
- [LLM 响应缓存](documentation/prompts/022-prompts__llm-response-caching.md)

### documentation/skills

- [Skills 用法](documentation/skills/023-skills.md)

### documentation/strategies

- [预定义节点和组件](documentation/strategies/024-nodes-and-components.md)
- [预定义 agent 策略](documentation/strategies/025-predefined-agent-strategies.md)
- [自定义策略图](documentation/strategies/026-custom-strategy-graphs.md)
- [并行节点执行](documentation/strategies/027-parallel-node-execution.md)
- [节点之间的数据传递](documentation/strategies/028-data-transfer-between-nodes.md)

### documentation/tools

- [概述](documentation/tools/029-tools.md)
- [内置工具](documentation/tools/030-tools__built-in-tools.md)
- [基于注解的工具](documentation/tools/031-tools__annotation-based-tools.md)
- [基于类的工具](documentation/tools/032-tools__class-based-tools.md)
- [ToolDescriptorSchemer](documentation/tools/033-tools__tool-descriptor-schemer.md)

### documentation/features

- [功能](documentation/features/034-features.md)
- [事件处理程序](documentation/features/035-features__agent-event-handlers.md)
- [跟踪](documentation/features/036-features__tracing.md)
- [聊天记忆](documentation/features/037-features__chat-memory.md)
- [构建带记忆的聊天 agent](documentation/features/038-features__chat-memory__chat-agent-with-memory.md)
- [带记忆的聊天后端](documentation/features/039-features__chat-memory__chat-backend-with-memory.md)
- [长期记忆](documentation/features/040-features__long-term-memory.md)
- [Agent 持久化](documentation/features/041-features__agent-persistence.md)
- [OpenTelemetry support](documentation/features/042-features__open-telemetry.md)
- [Datadog 导出器](documentation/features/043-features__open-telemetry__opentelemetry-datadog-exporter.md)
- [Langfuse 导出器](documentation/features/044-features__open-telemetry__opentelemetry-langfuse-exporter.md)
- [W&B Weave 导出器](documentation/features/045-features__open-telemetry__opentelemetry-weave-exporter.md)
- [自定义特性](documentation/features/046-features__custom-features.md)

### documentation/history-compression

- [History compression](documentation/history-compression/047-history-compression.md)

### documentation/model-context-protocol

- [模型上下文协议](documentation/model-context-protocol/048-model-context-protocol.md)

### documentation/a2a

- [A2A 协议](documentation/a2a/049-a2a.md)
- [A2A Server](documentation/a2a/050-a2a__a2a-server.md)
- [A2A Client](documentation/a2a/051-a2a__a2a-client.md)
- [A2A 与 Koog 集成](documentation/a2a/052-a2a__a2a-koog-integration.md)

### documentation/agent-client-protocol

- [Agent Client Protocol](documentation/agent-client-protocol/053-agent-client-protocol.md)

### documentation/llm-parameters

- [LLM 参数](documentation/llm-parameters/054-llm-parameters.md)

### documentation/model-capabilities

- [模型能力](documentation/model-capabilities/055-model-capabilities.md)

### documentation/content-moderation

- [内容审核](documentation/content-moderation/056-content-moderation.md)

### documentation/backend-framework-integrations

- [Ktor 集成：Koog plugin](documentation/backend-framework-integrations/057-ktor-plugin.md)
- [Spring Boot 集成](documentation/backend-framework-integrations/058-spring-boot.md)
- [Spring AI 集成](documentation/backend-framework-integrations/059-spring-ai-integration.md)

### documentation/cloud-platform-integrations

- [Amazon Bedrock AgentCore](documentation/cloud-platform-integrations/060-amazon-bedrock-agentcore.md)

### documentation/advanced-usage

- [Agent 事件](documentation/advanced-usage/061-agent-events.md)
- [结构化输出](documentation/advanced-usage/062-structured-output.md)
- [流式 API](documentation/advanced-usage/063-streaming-api.md)
- [自定义节点实现](documentation/advanced-usage/064-custom-nodes.md)
- [LLM sessions 和手动历史管理](documentation/advanced-usage/065-sessions.md)

### documentation/advanced-usage/subgraphs

- [概述](documentation/advanced-usage/subgraphs/066-subgraphs-overview.md)
- [自定义子图](documentation/advanced-usage/subgraphs/067-custom-subgraphs.md)

### documentation/advanced-usage

- [Embeddings](documentation/advanced-usage/068-embeddings.md)
- [检索增强生成（RAG）](documentation/advanced-usage/069-retrieval-augmented-generation.md)
- [序列化](documentation/advanced-usage/070-serialization.md)

### documentation/testing

- [测试](documentation/testing/071-testing.md)

### why-koog

- [为什么选择 Koog](why-koog/072-why-koog.md)

### examples

- [示例](examples/073-examples.md)
- [Attachments](examples/074-examples__Attachments.md)
- [使用 Koog 构建 AI 银行助手](examples/075-examples__Banking.md)
- [使用 AWS Bedrock 和 Koog 框架构建 AI Agent](examples/076-examples__BedrockAgent.md)
- [使用 Koog 构建可调用工具的计算器 Agent](examples/077-examples__Calculator.md)
- [使用Koog框架构建人工智能棋手](examples/078-examples__Chess.md)
- [使用 Koog 构建猜数字 Agent](examples/079-examples__Guesser.md)
- [使用 OpenTelemetry 将 Koog Agent 追踪到 Langfuse](examples/080-examples__Langfuse.md)
- [在 Kotlin Notebook 中使用 Koog 连接 Google Maps MCP：从零到海拔查询](examples/081-examples__GoogleMapsMcp.md)
- [使用 Playwright MCP 和 Koog 驱动浏览器](examples/082-examples__PlaywrightMcp.md)
- [Unity + Koog：从 Kotlin Agent 驱动你的游戏](examples/083-examples__UnityMcp.md)
- [Koog 中的 OpenTelemetry：追踪你的 AI agent](examples/084-examples__OpenTelemetry.md)
- [构建一个简单的吸尘器 Agent](examples/085-examples__VaccumAgent.md)
- [Koog agents 的 Weave tracing](examples/086-examples__Weave.md)

### other pages

- [Koog on Slack](documentation/other/087-koog-slack-channel.md)

### examples

- [使用 Bright Data 的 The Web MCP 和 Koog 进行网页抓取](examples/088-examples__WebMcpClient.md)
