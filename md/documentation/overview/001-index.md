# 概览

Koog 是一个开源的 JetBrains 框架，用于构建专为 JVM 生态系统设计的 AI agent。
它为 Kotlin 和 Java 开发者提供一流的开发体验，包含符合 Kotlin 习惯、类型安全的 Kotlin DSL，以及流畅的 builder 风格 Java API。

Java 开发者可以在 JVM 上通过惯用 API 充分发挥 Koog 的能力；Kotlin 开发者还可以使用 Kotlin Multiplatform 将 agent 部署到 JS、WasmJS、Android 和 iOS 目标平台。

- [**快速开始**](quickstart/)

  ---

  构建并运行你的第一个 AI agent
- [**术语表**](glossary/)

  ---

  了解核心术语
- [**模块版本控制**](module-versioning/)

  ---

  了解稳定模块与 beta 模块以及 API 保证

## Agents

了解[一般意义上的 agent](agents/)，以及如何使用 Koog 创建不同类型的 agent：

- [**基础 agent**](agents/basic-agents/)

  ---

  使用适用于大多数常见用例的预定义策略
- [**函数式 agent**](agents/functional-agents/)

  ---

  在纯 Kotlin 或 Java 中将自定义逻辑定义为 lambda 函数
- [**基于图的 agent**](agents/graph-based-agents/)

  ---

  将自定义工作流实现为策略图
- [**规划 agent**](agents/planner-agents/) beta

  ---

  迭代地构建并执行计划，直到状态符合期望条件

## 核心组件

详细了解 Koog agent 的核心组件：

- [**Prompts**](prompts/)

  ---

  创建、管理并运行驱动 agent 与 LLM 交互的 prompt
- [**Strategies**](predefined-agent-strategies/)

  ---

  将 agent 的预期工作流设计为有向图
- [**Tools**](tools/)

  ---

  让 agent 能够与外部数据源和服务交互
- [**Features**](features/)

  ---

  扩展和增强 AI agent 的功能

## 高级用法

- [**历史压缩**](history-compression/)

  ---

  在长时间运行的对话中，使用高级技术在保留上下文的同时优化 token 使用
- [**Agent 持久化**](features/agent-persistence/)

  ---

  在执行期间的特定点恢复 agent 状态
- [**结构化输出**](structured-output/)

  ---

  生成结构化格式的响应
- [**Streaming API**](streaming-api/)

  ---

  通过流式支持和并行工具调用实时处理响应
- [**知识检索**](embeddings/) beta

  ---

  使用[向量嵌入](embeddings/)和 [RAG](retrieval-augmented-generation/) 在对话之间保留并检索知识
- [**Tracing**](features/tracing/)

  ---

  使用详细且可配置的 tracing 调试和监控 agent 执行
- [**Long Term Memory**](features/long-term-memory/) beta

  ---

  集成向量数据库和记忆提供方，以支持 RAG 和持久记忆。

## 集成

- [**Model Context Protocol (MCP)**](model-context-protocol/) beta

  ---

  在 AI agent 中直接使用 MCP tools
- [**Spring Boot**](spring-boot/) beta

  ---

  将 Koog 添加到你的 Spring 应用
- [**Ktor**](ktor-plugin/) beta

  ---

  将 Koog 与 Ktor 服务器集成
- [**OpenTelemetry**](features/open-telemetry/)

  ---

  使用流行的可观测性工具对你的 agent 进行 trace、log 和度量
- [**A2A Protocol**](a2a/) beta

  ---

  通过共享协议连接 agent 和服务
