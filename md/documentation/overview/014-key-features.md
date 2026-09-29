# 关键特性

Koog 的关键特性包括：

- **符合惯用法的 Kotlin 和 Java 支持**：可在类型安全的 Kotlin DSL 与专用、流畅的 Java builder API 之间选择。Java API 旨在让 Java 团队感觉自然，使用标准线程池执行器，而不是暴露协程。
- **可靠性和容错能力**：通过内置重试处理失败，并使用 Agent persistence feature 在执行过程中的特定点恢复 Agent 状态。
- **智能历史压缩**：使用先进的内置历史压缩技术，在长时间运行的对话中优化 token 使用量，同时保持上下文。
- **企业级集成**：利用与 Spring Boot 和 Ktor 等流行 JVM 框架的集成，将 Koog 嵌入到你的应用程序中。
- **通过 OpenTelemetry exporters 实现可观测性**：借助对流行可观测性提供方（W&B Weave、Langfuse）的内置支持，监控和调试应用程序。
- **LLM 切换和无缝历史适配**：可在任意时刻切换到不同的 LLM，而不会丢失现有对话历史，也可在多个 LLM provider 之间重新路由。
- **多平台开发**：对于用 Kotlin 编写的 Agent，可使用 Kotlin Multiplatform 将 Agent 部署到 JVM、JS、WasmJS、Android 和 iOS 目标。
- **Model Context Protocol 集成**：在 AI Agent 中使用 Model Context Protocol (MCP) 工具。
- **知识检索和记忆**：使用向量嵌入和 RAG 在对话之间保留并检索知识。
- **强大的 Streaming API**：通过流式支持和并行工具调用实时处理响应。
- **模块化 feature 系统**：通过可组合架构自定义 Agent 能力。
- **灵活的图工作流**：使用直观的基于图的工作流设计复杂 Agent 行为。
- **自定义工具创建**：通过访问外部系统和 API 的工具增强你的 Agent。
- **全面追踪**：通过详细、可配置的 tracing 调试并监控 Agent 执行。
