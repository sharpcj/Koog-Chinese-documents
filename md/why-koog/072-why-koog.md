# 为什么选择 Koog

Koog 旨在以 JetBrains 级别的质量解决现实世界中的问题。
它提供先进的 AI 算法、开箱即用的经过验证的技术、Kotlin DSL 以及 Java 流畅 API，还有超越传统框架的强大多平台支持。

它的主要关注点是可靠性——使 AI 智能体能够被放心地用于要求严苛的企业环境。

## 与 Java 和 Kotlin 应用程序集成

Koog 为 Kotlin 开发者提供了专门设计的 Kotlin 领域特定语言（DSL），并为 Java 用户提供了流畅的 Java API。
同一个框架为这两种 JVM 语言都带来了原生的使用体验，确保无缝集成到 Kotlin 和 Java 应用程序中，同时显著提升生产力并改善整体开发者体验。

## 通过 JetBrains 产品进行真实世界验证

Koog 为多个 JetBrains 产品提供支持，包括内部 AI 智能体。
这种真实世界的集成确保 Koog 持续接受测试、改进，并针对实际用例进行验证。
它专注于在实践中行之有效的方法，融合了来自大量反馈和真实产品场景的洞见。
这种集成为 Koog 带来了区别于其他框架的优势。

## 开箱即用的高级解决方案

Koog 包含预构建的、可组合的解决方案，以简化和加速智能体系统的开发，这使其区别于仅提供基本组件的框架：

- **带有领域建模的图工作流。** 将 AI 工作流建模为基于经过验证的领域模型构建的显式图。通过将需求表达为结构化数据类，而不是依赖朴素的提示词，你可以精确控制智能体行为，并显著提高可靠性和可预测性。
- **多种历史压缩策略。** Koog 开箱即用地提供了用于压缩和管理长时间运行对话的高级策略，无需手动试验各种方法。借助由 ML 工程师测试和改进的精细调优提示词、技术和算法，你可以依赖经过验证的方法来提升性能。有关压缩策略的更多详细信息，请参阅[历史压缩](https://docs.koog.ai/history-compression/)。要了解 Koog 在真实场景中如何处理压缩和上下文管理，请查看[这篇文章](https://blog.jetbrains.com/ai/2025/07/when-tool-calling-becomes-an-addiction-debugging-llm-patterns-in-koog/)。
- **高级持久化（持久执行）。** Koog 允许你恢复完整的智能体状态机，而不仅仅是聊天消息。这实现了检查点、故障恢复，甚至能够回退到状态机执行过程中的任意点等功能。
- **所有现代智能体模式，一个框架。** 图工作流、GOAP（目标导向行动规划）和 LLM 规划、多智能体编排——完全支持且完全可组合。构建完全符合你用例需求的智能体。
- **无缝 LLM 切换。** 你可以在任意时刻将对话切换到具有一组新可用工具的不同大语言模型（LLM），而不会丢失现有对话历史。Koog 会自动重写历史并处理不可用的工具，从而实现平滑过渡和自然的交互流程。
- **健壮的重试组件。** Koog 包含重试机制，让你可以包装智能体系统中的任意操作集，并重试它们直到满足可配置的条件。你可以提供反馈并调整每次尝试，以确保可靠的结果。如果 LLM 调用超时、工具未按预期工作或出现网络问题，Koog 可确保你的智能体即使在临时故障期间也能保持韧性并有效运行。有关更多技术细节，请参阅[重试功能](https://docs.koog.ai/history-compression/)。

## 广泛集成、多平台支持、增强的可观测性

Koog 支持在各种平台和环境中开发和部署智能体应用程序：

- **Spring Boot、Spring AI 和 Ktor 集成**。Koog 与广泛使用的企业环境集成。
  - 对于 Spring Boot，Koog 提供开箱即用的 bean 和自动配置的 LLM 客户端，让你轻松开始构建 AI 驱动的工作流。
  - 如果你已经在使用 Spring AI 来实现 LLM 和 RAG 能力，Koog 可以作为编排和智能体框架叠加在其之上。这使你能够利用 Spring AI 的广泛集成，同时受益于 Koog 先进、可靠且高性价比的 AI 工作流。
  - 如果你有 Ktor 服务器，可以将 Koog 作为插件安装，使用配置文件配置提供方，并直接从任意路由调用智能体，而无需手动连接 LLM 客户端。
- **多平台支持**。你可以将智能体应用程序部署到 JVM、JS、WasmJS、Android 和 iOS 目标平台。
- **广泛的 AI 集成**。Koog 与主流 LLM 提供方集成，包括 OpenAI、Anthropic、Google、DeepSeek、Mistral、Alibaba，以及像 Bedrock 这样的企业级 AI 云。它还支持 Ollama 等本地模型。有关可用提供方的完整列表，请参阅 [LLM 提供方](https://docs.koog.ai/llm-providers/)。
- **OpenTelemetry 支持**。Koog 开箱即用地与流行的可观测性提供方集成，如 [W&B Weave](https://wandb.ai/site/weave/)、[Langfuse](https://langfuse.com/) 和 [DataDog](https://www.datadoghq.com/)，用于监控和调试 AI 应用程序。借助原生 OpenTelemetry 支持，你可以使用系统中已在使用的相同工具来追踪、记录和度量你的智能体。要了解更多信息，请参阅 [OpenTelemetry](https://docs.koog.ai/opentelemetry-support/)。

## 与 ML 工程师和产品团队协作

Koog 的一个独特优势是它与 JetBrains ML 工程师和产品团队直接协作。
这确保使用 Koog 构建的功能不仅仅是理论上的，而是基于真实产品需求进行测试和改进的。
这意味着 Koog 融合了：

- **精细调优的提示词和策略**，针对真实世界性能进行了优化。
- **经过验证的工程方法**，通过产品开发发现并验证，例如其独特的历史压缩策略。你可以在[这篇详细文章](https://blog.jetbrains.com/ai/2025/07/when-tool-calling-becomes-an-addiction-debugging-llm-patterns-in-koog/)中了解更多。
- **持续改进**，帮助 Koog 保持高效并适应不断变化的需求。

## 对开发者社区的承诺

Koog 团队坚定致力于构建强大的开发者社区。
通过积极收集和采纳反馈，Koog 不断演进，以有效满足开发者的需求。
我们正在积极扩展对多样化 AI 架构、全面基准测试、详细用例指南和教育资源的支持，以赋能开发者。

## 从哪里开始

- 在[概述](../)中探索 Koog 的能力。
- 通过我们的[快速开始](../quickstart/)指南构建你的第一个 Koog 智能体。
- 在 Koog [发行说明](https://github.com/JetBrains/koog/blob/main/CHANGELOG.md)中查看最新更新。
- 从[示例](https://docs.koog.ai/examples/)中学习。
