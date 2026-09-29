# 版本控制

Koog 遵循 [语义版本控制](https://semver.org/)，格式为 `X.Y.Z`(例如 `1.3.0`)。

该框架是 API 稳定的：一旦公开的 API 发布，如果没有主要版本更新，它就不会被破坏。

## 版本组件

|组件|名称 |格式|意义|
| --- | --- | --- | --- |
| `X` |专业| `X.y.z` |对现有 APIs 的重大更改 |
| `Y` |次要| `x.Y.z` |新的 API 添加和弃用；所有现有的 APIs 继续工作 |
| `Z` |错误修正 | `x.y.Z` |仅修复错误；没有 API 变化 |

### 主要 (`X`)

- 可能会对现有 APIs 进行重大更改。
- 旧的 APIs 可能会被删除。
- 将提供迁移指南。
- 每年最多发布一次。

### 次要 (`Y`)

- 可能会添加新的 APIs。
- 可能会弃用现有的 APIs（并提供替代品），但弃用的 APIs 仍然有效。
- 没有重大更改 - 针对先前次要版本编译的所有代码都将继续编译和工作。
- 每月最多发布一次。

### 错误修复 (`Z`)

- 仅包含错误修复。
- 没有 API 添加、删除或弃用。
- 每周最多发布一次。

## 弃用政策

在次要版本 (`Y`) 中弃用的 APIs 至少在下一个主要版本 (`X`) 之前仍然可用。
弃用警告将指示推荐的替代品。

## 稳定版和 Beta 模块

某些模块被认为是实验性的，并以 `-beta` 版本后缀（例如 `1.3.0-beta`）而不是标准 `X.Y.Z` 发布。由于以下几个原因之一，模块可能处于测试版状态：

- **外部集成** - 底层 LLM 提供程序 API 或外部框架（例如 Spring AI）本身可能不稳定或容易发生频繁或预期的变化。
- **实验功能** - 功能区域仍在探索中，API 形状可能会演变（例如 GOAP 规划策略）。
- **实验协议** — 该模块实现了一个本身尚未稳定的协议（例如，A2A、Kotlin MCP）。

虽然我们已尽一切努力保持 Beta 模块稳定，但在次要版本中可能会发生一些 API 更改。 Beta的变化不会影响任何稳定的模块。

版本 `X.Y.Z` 的稳定模块始终与版本 `X.Y.Z-beta` 的测试版模块兼容（反之亦然）。所有模块均可同步更新。

### 伞形模块

|模块|版本 |内容 |
| --- | --- | --- |
| `koog-agents` | `1.3.0` |所有稳定模块（可传递）——推荐起点 |
| `koog-agents-additions` | `1.3.0-beta` |大多数测试/实验模块（独立外部集成除外）|

### 模块版本

稳定模块 (`1.3.0`)Beta 模块 (`1.3.0-beta`)

|模块|版本 |
| --- | --- |
| `agents` | `1.3.0` |
| `agents-core` | `1.3.0` |
| `agents-features` | `1.3.0` |
| `agents-features-chat-history-jdbc` | `1.3.0` |
| `agents-features-chat-memory-sql` | `1.3.0` |
| `agents-features-event-handler` | `1.3.0` |
| `agents-features-memory` | `1.3.0` |
| `agents-features-opentelemetry` | `1.3.0` |
| `agents-features-persistence-jdbc` | `1.3.0` |
| `agents-features-snapshot` | `1.3.0` |
| `agents-features-sql` | `1.3.0` |
| `agents-features-tokenizer` | `1.3.0` |
| `agents-features-trace` | `1.3.0` |
| `agents-mcp-metadata` | `1.3.0` |
| `agents-test` | `1.3.0` |
| `agents-tools` | `1.3.0` |
| `agents-utils` | `1.3.0` |
| `embeddings` | `1.3.0` |
| `embeddings-base` | `1.3.0` |
| `embeddings-llm` | `1.3.0` |
| `http-client` | `1.3.0` |
| `http-client-core` | `1.3.0` |
| `http-client-java` | `1.3.0` |
| `http-client-ktor` | `1.3.0` |
| `http-client-okhttp` | `1.3.0` |
| `http-client-test` | `1.3.0` |
| `koog-agents` | `1.3.0` |
| `koog-spring-ai` | `1.3.0` |
| `prompt` | `1.3.0` |
| `prompt-cache` | `1.3.0` |
| `prompt-cache-files` | `1.3.0` |
| `prompt-cache-model` | `1.3.0` |
| `prompt-executor` | `1.3.0` |
| `prompt-executor-anthropic-client` | `1.3.0` |
| `prompt-executor-bedrock-client` | `1.3.0` |
| `prompt-executor-cached` | `1.3.0` |
| `prompt-executor-clients` | `1.3.0` |
| `prompt-executor-model` | `1.3.0` |
| `prompt-executor-ollama-client` | `1.3.0` |
| `prompt-executor-openai-client` | `1.3.0` |
| `prompt-executor-openai-client-base` | `1.3.0` |
| `prompt-executor-openrouter-client` | `1.3.0` |
| `prompt-llm` | `1.3.0` |
| `prompt-markdown` | `1.3.0` |
| `prompt-model` | `1.3.0` |
| `prompt-processor` | `1.3.0` |
| `prompt-structure` | `1.3.0` |
| `prompt-tokenizer` | `1.3.0` |
| `prompt-xml` | `1.3.0` |
| `rag-base` | `1.3.0` |
| `serialization` | `1.3.0` |
| `serialization-core` | `1.3.0` |
| `serialization-jackson` | `1.3.0` |
| `serialization-test` | `1.3.0` |
| `test-tck` | `1.3.0` |
| `test-utils` | `1.3.0` |
| `utils` | `1.3.0` |

|模块|版本 |
| --- | --- |
| `a2a-client` | `1.3.0-beta` |
| `a2a-core` | `1.3.0-beta` |
| `a2a-server` | `1.3.0-beta` |
| `a2a-test` | `1.3.0-beta` |
| `a2a-test-server-tck` | `1.3.0-beta` |
| `a2a-transport-client-jsonrpc-http` | `1.3.0-beta` |
| `a2a-transport-core-jsonrpc` | `1.3.0-beta` |
| `a2a-transport-server-jsonrpc-http` | `1.3.0-beta` |
| `agents-ext` | `1.3.0-beta` |
| `agents-features-a2a-client` | `1.3.0-beta` |
| `agents-features-a2a-core` | `1.3.0-beta` |
| `agents-features-a2a-server` | `1.3.0-beta` |
| `agents-features-acp` | `1.3.0-beta` |
| `agents-features-chat-history-aws` | `1.3.0-beta` |
| `agents-features-longterm-memory` | `1.3.0-beta` |
| `agents-features-longterm-memory-aws` | `1.3.0-beta` |
| `agents-mcp` | `1.3.0-beta` |
| `agents-mcp-server` | `1.3.0-beta` |
| `agents-planner` | `1.3.0-beta` |
| `koog-agents-additions` | `1.3.0-beta` |
| `koog-ktor` | `1.3.0-beta` |
| `koog-spring-ai-common` | `1.3.0-beta` |
| `koog-spring-ai-starter-chat-memory` | `1.3.0-beta` |
| `koog-spring-ai-starter-model-chat` | `1.3.0-beta` |
| `koog-spring-ai-starter-model-embedding` | `1.3.0-beta` |
| `koog-spring-ai-starter-vector-store` | `1.3.0-beta` |
| `koog-spring-boot-starter` | `1.3.0-beta` |
| `prompt-cache-redis` | `1.3.0-beta` |
| `prompt-executor-dashscope-client` | `1.3.0-beta` |
| `prompt-executor-deepseek-client` | `1.3.0-beta` |
| `prompt-executor-google-client` | `1.3.0-beta` |
| `prompt-executor-litert-client` | `1.3.0-beta` |
| `prompt-executor-llms-all` | `1.3.0-beta` |
| `prompt-executor-mistralai-client` | `1.3.0-beta` |
| `rag-vector` | `1.3.0-beta` |
