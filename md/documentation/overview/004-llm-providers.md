# LLM providers

Koog 可与主流 LLM provider 配合使用，也支持使用 [Ollama](https://ollama.com/) 运行本地模型。
当前支持以下 providers：

| LLM provider | Choose for |
| --- | --- |
| [OpenAI](https://platform.openai.com/docs/overview)（包括 [Azure OpenAI Service](https://azure.microsoft.com/en-us/products/ai-foundry/models/openai)） | 具备广泛能力的高级模型。 |
| [Anthropic](https://www.anthropic.com/) | 长上下文和 prompt caching。 |
| [Google](https://ai.google.dev/) β | 多模态处理（音频、视频）、大上下文。 |
| [DeepSeek](https://www.deepseek.com/) β | 具备成本效益的推理和编码。 |
| [OpenRouter](https://openrouter.ai/) | 通过一次集成访问来自多个 provider 的多个模型，便于灵活使用、provider 比较和统一 API。 |
| [Amazon Bedrock](https://aws.amazon.com/bedrock/) | AWS 原生环境、企业安全与合规、多 provider 访问。 |
| [Mistral](https://mistral.ai/) β | 欧洲数据托管、GDPR 合规。 |
| [Alibaba](https://www.alibabacloud.com/en?_p_lc=1) β（[DashScope](https://dashscope.aliyun.com/) OpenAI-compatible client） | 大上下文和高性价比 Qwen 模型。 |
| [Ollama](https://ollama.com/) | 隐私、本地开发、离线运行，以及无 API 成本。 |

下表展示 Koog 支持的 LLM 能力，以及哪些 providers 的模型提供这些能力。

| LLM capability | OpenAI | Anthropic | Google β | DeepSeek β | OpenRouter | Amazon Bedrock | Mistral β | Alibaba β (DashScope OpenAI-compatible client) | Ollama (local models) |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Supported input | 文本、图像、音频、文档 | 文本、图像、文档[1](#fn:1) | 文本、图像、音频、视频、文档[1](#fn:1) | 文本 | 因模型而异 | 因模型而异 | 文本、图像、文档[1](#fn:1) | 文本、图像、音频、视频[1](#fn:1) | 文本、图像[1](#fn:1) |
| Response streaming | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| Tools | ✓ | ✓ | ✓ | ✓ | ✓ | ✓[1](#fn:1) | ✓ | ✓ | ✓ |
| Tool choice | ✓ | ✓ | ✓ | ✓ | ✓ | ✓[1](#fn:1) | ✓ | ✓ | – |
| Structured output (JSON Schema) | ✓ | ✓[1](#fn:1) | ✓ | ✓ | ✓[1](#fn:1) | – | ✓ | ✓[1](#fn:1) | ✓ |
| Multiple choices | ✓ | – | ✓ | – | ✓[1](#fn:1) | ✓[1](#fn:1) | ✓ | ✓[1](#fn:1) | – |
| Temperature | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| Speculation | ✓[1](#fn:1) | – | – | – | ✓[1](#fn:1) | – | ✓[1](#fn:1) | ✓[1](#fn:1) | – |
| Content moderation | ✓ | – | – | – | – | ✓ | ✓ | – | ✓ |
| Embeddings | ✓ | – | – | – | – | ✓ | ✓ | – | ✓ |
| Prompt caching | ✓[1](#fn:1) | ✓ | – | – | – | – | – | – | – |
| Completion | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| Local execution | – | – | – | – | – | – | – | – | ✓ |

Note

Koog 支持用于创建 AI Agent 的最常用能力。
各 provider 的 LLM 可能还有 Koog 当前不支持的其他功能。
要了解更多信息，请参阅 [Model capabilities](../model-capabilities/)。

## 使用 providers

Koog 允许你在两个层级上使用 LLM providers：

- 使用 **LLM client** 直接与特定 provider 交互。
  每个 client 都实现 `LLMClient` 接口，负责该 provider 的身份验证、
  请求格式化和响应解析。
  详情请参阅 [LLM clients](../prompts/llm-clients/)。
- 使用 **prompt executor** 作为更高层抽象，用于包装一个或多个 LLM clients，
  管理它们的生命周期，并统一跨 provider 的接口。
  它可以在 providers 之间切换，
  并可选择使用相应 client fallback 到已配置的 provider 和 LLM。
  你既可以创建自己的 executor，也可以使用针对特定 provider 的预定义 prompt executor。
  详情请参阅 [Prompt executors](../prompts/prompt-executors/)。

使用 prompt executor 可在一个或多个 LLMClients 之上提供更高层抽象。
它管理 client 生命周期，并暴露跨 providers 的统一接口。
在多 provider 设置中，它可以在 providers 之间路由请求，并在核心请求需要时可选地 fallback 到指定 client。你可以创建自己的 executor，也可以使用预定义 executor——单 provider 和多 provider 选项都可用。

## 后续步骤

- 使用特定 LLM provider [创建并运行 Agent](../quickstart/)。
- 进一步了解 [prompts](../prompts/)。

---

1. 该能力仅受 provider 的部分模型支持。 [↩](#fnref:1 "Jump back to footnote 1 in the text")[↩](#fnref2:1 "Jump back to footnote 1 in the text")[↩](#fnref3:1 "Jump back to footnote 1 in the text")[↩](#fnref4:1 "Jump back to footnote 1 in the text")[↩](#fnref5:1 "Jump back to footnote 1 in the text")[↩](#fnref6:1 "Jump back to footnote 1 in the text")[↩](#fnref7:1 "Jump back to footnote 1 in the text")[↩](#fnref8:1 "Jump back to footnote 1 in the text")[↩](#fnref9:1 "Jump back to footnote 1 in the text")[↩](#fnref10:1 "Jump back to footnote 1 in the text")[↩](#fnref11:1 "Jump back to footnote 1 in the text")[↩](#fnref12:1 "Jump back to footnote 1 in the text")[↩](#fnref13:1 "Jump back to footnote 1 in the text")[↩](#fnref14:1 "Jump back to footnote 1 in the text")[↩](#fnref15:1 "Jump back to footnote 1 in the text")[↩](#fnref16:1 "Jump back to footnote 1 in the text")[↩](#fnref17:1 "Jump back to footnote 1 in the text")[↩](#fnref18:1 "Jump back to footnote 1 in the text")