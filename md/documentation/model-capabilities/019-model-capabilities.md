# 模型能力

Koog 提供了一组抽象和实现，用于使用来自各种语言的大型语言模型（LLMs）
LLM 以与提供商无关的方式提供。该集合包括以下类：

- **LLMCapability**：定义LLMs可以支持的各种功能的类层次结构，例如：

- 温度调节以控制响应随机性
- 外部系统交互工具集成
- 用于处理视觉数据的视觉处理
- 向量表示的嵌入生成
- 完成文本生成任务
- 对结构化数据的模式支持（JSON 与 Simple 和 Full 变体）
- 探索性反应的推测
- **LLModel**：代表特定 LLM 及其提供者、唯一标识符和支持的数据类
能力。

这是以统一方式与不同 LLM 提供者交互的基础，允许应用程序工作
使用各种模型，同时抽象出特定于提供商的详细信息。

## LLM 能力

LLM 功能表示大语言模型可以支持的特定特性或功能。在Koog
框架、功能用于定义特定模型可以做什么以及如何配置它。各项能力
表示为 `LLMCapability` 类的子类或数据对象。

配置 LLM 以在应用程序中使用时，您可以通过将其添加到
创建`LLModel`实例时列出`capabilities`。这使得框架能够与模型正确交互
并适当地使用其功能。

### 核心能力

下面的列表包括适用于 Koog 框架中的模型的 LLM 特定核心功能：

- **推测** (`LLMCapability.Speculation`)：让模型生成推测或探索性响应
不同程度的可能性。对于具有更广泛潜力的创造性或假设场景很有用
结果
是所期望的。
- **温度** (`LLMCapability.Temperature`)：允许调整模型的响应随机性或创造力
水平。较高的温度值会产生更多样化的输出，而较低的温度值会导致更集中和更集中的输出。
确定性响应。
- **工具** (`LLMCapability.Tools`)：表示支持外部工具使用或集成。这种能力让
模型运行特定工具或与外部系统交互。
- **工具选择** (`LLMCapability.ToolChoice`)：配置工具调用如何与 LLM 配合使用。根据型号，
它可以配置为：

- 自动选择生成文本或工具调用
- 仅生成工具调用，从不生成文本
- 仅生成文本，从不工具调用
- 强制调用定义的工具中的特定工具
- **多项选择** (`LLMCapability.MultipleChoices`)：让模型生成多个独立的回复选择
到一个提示。

### 媒体处理能力

以下列表代表了一组用于处理图像或音频等媒体内容的功能：

- **Vision** (`LLMCapability.Vision`)：基于视觉的功能类，用于处理、分析和推断见解
从视觉数据。
支持以下类型的可视化数据：

- **图像** (`LLMCapability.Vision.Image`)：处理与图像相关的视觉任务，例如图像分析、识别、
和解释。
- **视频** (`LLMCapability.Vision.Video`)：处理视频数据，包括分析和理解视频
内容。
- **音频** (`LLMCapability.Audio`)：提供与音频相关的功能，例如转录、音频生成或
基于音频的交互。
- **文档** (`LLMCapability.Document`)：支持处理和处理基于文档的输入和输出。

### 文本处理能力

以下功能列表代表文本生成和处理功能：

- **嵌入** (`LLMCapability.Embed`)：让模型从输入文本生成向量嵌入，从而实现相似性
比较、聚类和其他基于向量的分析。
- **完成** (`LLMCapability.Completion`)：包括根据给定的输入上下文生成文本或内容，
例如完成句子、生成建议或生成与输入数据一致的内容。
- **提示缓存** (`LLMCapability.PromptCaching`)：支持提示的缓存功能，可能
改善
重复或类似查询的性能。
- **审核** (`LLMCapability.Moderation`)：让模型分析文本中的潜在有害内容并
根据骚扰、仇恨言论、自残、性内容、暴力等不同类别对其进行分类。

### 架构功能

下面的列表显示了与处理结构化数据相关的功能：

- **Schema** (`LLMCapability.Schema`)：与数据交互和相关的结构化模式功能的类
使用特定格式进行编码。
包括对以下格式的支持：
- **JSON** (`LLMCapability.Schema.JSON`)：不同级别的 JSON 模式支持：
- **Basic** (`LLMCapability.Schema.JSON.Basic`)：提供轻量级或基本的 JSON 处理能力。
- **Standard** (`LLMCapability.Schema.JSON.Standard`)：为复杂数据提供全面的 JSON 模式支持
结构。

## 创建模型（LLModel）配置

要以通用的、与提供商无关的方式定义模型，请创建模型配置作为 `LLModel` 的实例
具有以下参数的类：

|名称 |数据类型 |必填|默认 |描述 |
| --- | --- | --- | --- | --- |
| `provider` | LLM 提供者 |是的 |  | LLM的提供商，例如Google或OpenAI。这标识了创建或托管模型的公司或组织。 |
| `id` |字符串|是的 |  | LLM 实例的唯一标识符。这通常代表特定的型号版本或名称。例如，`gpt-4-turbo`、`claude-3-opus`、`llama-3-2`。 |
| `capabilities` |列表<LLM能力> |是的 |  | LLM 支持的功能列表，例如温度调节、工具使用或基于模式的任务。这些功能定义了模型可以做什么以及如何配置模型。 |
| `contextLength` |长|是的 |  | LLM 的上下文长度。这是LLM可以处理的最大代币数量。 |
| `maxOutputTokens` |长|没有 | `null` |提供者可以为 LLM 生成的最大代币数量。 |

### 示例

本节提供了创建具有不同能力的`LLModel`实例的详细示例。

下面的代码代表了具有核心功能的基本 LLM 配置：

KotlinJava

```
val basicModel = LLModel(
    provider = LLMProvider.OpenAI,
    id = "gpt-4-turbo",
    capabilities = listOf(
        LLMCapability.Temperature,
        LLMCapability.Tools,
        LLMCapability.Schema.JSON.Standard
    ),
    contextLength = 128_000
)
```

```
LLModel basicModel = new LLModel(
    LLMProvider.OpenAI,
    "gpt-4-turbo",
    List.of(
        LLMCapability.Temperature.INSTANCE,
        LLMCapability.Tools.INSTANCE,
        LLMCapability.Schema.JSON.Standard.INSTANCE
    ),
    128_000L
);
```

下面的模型配置是具有视觉功能的多模态 LLM：

KotlinJava

```
val visionModel = LLModel(
    provider = LLMProvider.OpenAI,
    id = "gpt-4-vision",
    capabilities = listOf(
        LLMCapability.Temperature,
        LLMCapability.Vision.Image,
        LLMCapability.MultipleChoices
    ),
    contextLength = 1_047_576,
    maxOutputTokens = 32_768
)
```

```
LLModel visionModel = new LLModel(
    LLMProvider.OpenAI,
    "gpt-4-vision",
    List.of(
        LLMCapability.Temperature.INSTANCE,
        LLMCapability.Vision.Image.INSTANCE,
        LLMCapability.MultipleChoices.INSTANCE
    ),
    1_047_576L,
    32_768L
);
```

具有音频处理能力的LLM：

KotlinJava

```
val audioModel = LLModel(
    provider = LLMProvider.Anthropic,
    id = "claude-3-opus",
    capabilities = listOf(
        LLMCapability.Audio,
        LLMCapability.Temperature,
        LLMCapability.PromptCaching
    ),
    contextLength = 200_000
)
```

```
LLModel audioModel = new LLModel(
    LLMProvider.Anthropic,
    "claude-3-opus",
    List.of(
        LLMCapability.Audio.INSTANCE,
        LLMCapability.Temperature.INSTANCE,
        LLMCapability.PromptCaching.INSTANCE
    ),
    200_000L
);
```

除了将模型创建为 `LLModel` 实例并必须指定所有相关参数之外，Koog 还包括
预定义模型及其配置以及受支持功能的集合。
要使用预定义的 Ollama 模型，请按如下方式指定：

KotlinJava

```
val metaModel = OllamaModels.Meta.LLAMA_3_2
```

```
LLModel metaModel = OllamaModels.Meta.LLAMA_3_2;
```

要检查模型是否支持特定功能，请使用 `contains` 方法来检查是否存在
`capabilities` 列表中的功能：

KotlinJava

```
// Check if models support specific capabilities
val supportsTools = basicModel.supports(LLMCapability.Tools) // true
val supportsVideo = visionModel.supports(LLMCapability.Vision.Video) // false

// Check for schema capabilities
val jsonCapability = basicModel.capabilities?.filterIsInstance<LLMCapability.Schema.JSON>()?.firstOrNull()
val hasFullJsonSupport = jsonCapability is LLMCapability.Schema.JSON.Standard // true
```

```
// Check if models support specific capabilities
boolean supportsTools = basicModel.supports(LLMCapability.Tools.INSTANCE); // true
boolean supportsVideo = visionModel.supports(LLMCapability.Vision.Video.INSTANCE); // false

// Check for schema capabilities
LLMCapability jsonCapability = basicModel.getCapabilities().stream()
    .filter(c -> c instanceof LLMCapability.Schema.JSON)
    .map(c -> (LLMCapability.Schema.JSON) c)
    .findFirst()
    .orElse(null);
boolean hasFullJsonSupport = jsonCapability instanceof LLMCapability.Schema.JSON.Standard; // true
```

### LLM 功能（按型号）

此参考显示了不同提供商的每个型号支持哪些 LLM 功能。

在下表中：

- `✓` 表示该型号支持该能力
- `-` 表示该型号不支持该功能
- 对于 JSON Schema、`Full` 或 `Simple` 表示模型支持 JSON Schema 功能的哪个变体

谷歌模型

#### 谷歌模型

|型号|温度| JSON Schema |完成 |多重选择 |工具|工具选择 |愿景（图像）|愿景（视频）|音频|
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Gemini2\_5Pro | ✓ | Full | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| Gemini2\_5Flash | ✓ | Full | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| Gemini2\_5FlashLite | ✓ | Full | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| Gemini2\_0Flash | ✓ | Full | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| Gemini2\_0Flash001 | ✓ | Full | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| Gemini2\_0FlashLite | ✓ | Full | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| Gemini2\_0FlashLite001 | ✓ | Full | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |

OpenAI 模型

#### OpenAI 模型

|型号|温度| JSON Schema |完成 |多重选择 |工具|工具选择 |愿景（图像）|愿景（视频）|音频|猜测|适度 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
|推理.O4Mini | - | Full | ✓ | ✓ | ✓ | ✓ | ✓ | - | - | ✓ | - |
|推理.O3Mini | - | Full | ✓ | ✓ | ✓ | ✓ | - | - | - | ✓ | - |
|推理.O3 | - | Full | ✓ | ✓ | ✓ | ✓ | ✓ | - | - | ✓ | - |
|推理.O1 | - | Full | ✓ | ✓ | ✓ | ✓ | ✓ | - | - | ✓ | - |
|聊天.GPT4o | ✓ | Full | ✓ | ✓ | ✓ | ✓ | ✓ | - | - | ✓ | - |
|聊天.GPT4\_1 | ✓ | Full | ✓ | ✓ | ✓ | ✓ | ✓ | - | - | ✓ | - |
|聊天.GPT5 | ✓ | Full | ✓ | ✓ | ✓ | ✓ | ✓ | - | - | ✓ | - |
|聊天.GPT5Mini | ✓ | Full | ✓ | ✓ | ✓ | ✓ | ✓ | - | - | ✓ | - |
|聊天.GPT5Nano | ✓ | Full | ✓ | ✓ | ✓ | ✓ | ✓ | - | - | ✓ | - |
|音频.GptAudio | ✓ | - | ✓ | - | ✓ | ✓ | - | - | ✓ | - | - |
|音频.GPT4oMiniAudio | ✓ | - | ✓ | - | ✓ | ✓ | - | - | ✓ | - | - |
|音频.GPT4oAudio | ✓ | - | ✓ | - | ✓ | ✓ | - | - | ✓ | - | - |
| CostOptimized.GPT4\_1Nano | ✓ | Full | ✓ | ✓ | ✓ | ✓ | ✓ | - | - | ✓ | - |
| CostOptimized.GPT4\_1Mini | ✓ | Full | ✓ | ✓ | ✓ | ✓ | ✓ | - | - | ✓ | - |
|成本优化.GPT4oMini | ✓ | Full | ✓ | ✓ | ✓ | ✓ | ✓ | - | - | ✓ | - |
|适度.Omni | - | - | - | - | - | - | ✓ | - | - | - | ✓ |

人择模型

#### 人类模型

|型号|温度| JSON Schema |完成 |工具|工具选择 |愿景（图像）|
| --- | --- | --- | --- | --- | --- | --- |
|寓言\_5 | ✓ | Full | ✓ | ✓ | ✓ | ✓ |
|作品\_4\_6 | ✓ | Full | ✓ | ✓ | ✓ | ✓ |
|作品\_4\_5 | ✓ | Full | ✓ | ✓ | ✓ | ✓ |
|作品\_4\_1 | ✓ | - | ✓ | ✓ | ✓ | ✓ |
|作品\_4 | ✓ | - | ✓ | ✓ | ✓ | ✓ |
|十四行诗\_4\_6 | ✓ | Full | ✓ | ✓ | ✓ | ✓ |
|十四行诗\_4\_5 | ✓ | Full | ✓ | ✓ | ✓ | ✓ |
|十四行诗\_4 | ✓ | - | ✓ | ✓ | ✓ | ✓ |
|俳句\_4\_5 | ✓ | Full | ✓ | ✓ | ✓ | ✓ |
|俳句\_3 | ✓ | - | ✓ | ✓ | ✓ | ✓ |

奥拉马型号

#### 奥拉玛模型

##### 元模型

|型号|温度| JSON Schema |工具|适度 |
| --- | --- | --- | --- | --- |
|骆驼\_3\_2\_3B | ✓ | Simple | ✓ | - |
|骆驼\_3\_2 | ✓ | Simple | ✓ | - |
|骆驼\_4 | ✓ | Simple | ✓ | - |
|骆驼\_GUARD\_3 | - | - | - | ✓ |

##### 阿里巴巴模型

|型号|温度| JSON Schema |工具|
| --- | --- | --- | --- |
| QWEN\_2\_5\_05B | ✓ | Simple | ✓ |
| QWEN\_3\_06B | ✓ | Simple | ✓ |
| QWQ | ✓ | Simple | ✓ |
| QWEN\_CODER\_2\_5\_32B | ✓ | Simple | ✓ |

##### Groq 模型

|型号|温度| JSON Schema |工具|
| --- | --- | --- | --- |
|骆驼\_3\_GROK\_工具\_使用\_8B | ✓ | Full | ✓ |
|骆驼\_3\_GROK\_工具\_使用\_70B | ✓ | Full | ✓ |

##### 花岗岩模型

|型号|温度| JSON Schema |工具|愿景（图像）|
| --- | --- | --- | --- | --- |
|花岗岩\_3\_2\_VISION | ✓ | Simple | ✓ | ✓ |

DeepSeek 模型

#### DeepSeek 模型

|型号|温度| JSON Schema |完成 |猜测|工具|工具选择 |愿景（图像）|
| --- | --- | --- | --- | --- | --- | --- | --- |
| DeepSeekChat | ✓ | Full | ✓ | - | ✓ | ✓ | - |
| DeepSeekReasoner | ✓ | Full | ✓ | - | ✓ | ✓ | - |

开放路由器型号

#### OpenRouter 型号

|型号|温度| JSON Schema |完成 |猜测|工具|工具选择 |愿景（图像）|
| --- | --- | --- | --- | --- | --- | --- | --- |
| Phi4 推理 | ✓ | Full | ✓ | ✓ | ✓ | ✓ | - |
|克劳德3Opus | ✓ | Full | ✓ | ✓ | ✓ | ✓ | ✓ |
|克劳德第三十四行诗| ✓ | Full | ✓ | ✓ | ✓ | ✓ | ✓ |
|克劳德俳句 | ✓ | Full | ✓ | ✓ | ✓ | ✓ | ✓ |
|克劳德3\_5十四行诗| ✓ | Full | ✓ | ✓ | ✓ | ✓ | ✓ |
|克劳德3\_7十四行诗| ✓ | Full | ✓ | ✓ | ✓ | ✓ | ✓ |
|克劳德十四行诗| ✓ | Full | ✓ | ✓ | ✓ | ✓ | ✓ |
|克劳德4\_1作品 | ✓ | Full | ✓ | ✓ | ✓ | ✓ | ✓ |
| GPT4oMini | ✓ | Full | ✓ | ✓ | ✓ | ✓ | ✓ |
| GPT5 | ✓ | Full | ✓ | ✓ | ✓ | ✓ | - |
| GPT5迷你 | ✓ | Full | ✓ | ✓ | ✓ | ✓ | - |
| GPT5纳米| ✓ | Full | ✓ | ✓ | ✓ | ✓ | - |
| GPT\_OSS\_120b | ✓ | Full | ✓ | ✓ | ✓ | ✓ | - |
| GPT4 | ✓ | Full | ✓ | ✓ | ✓ | ✓ | - |
| GPT4o | ✓ | Full | ✓ | ✓ | ✓ | ✓ | ✓ |
| GPT4涡轮 | ✓ | Full | ✓ | ✓ | ✓ | ✓ | ✓ |
| GPT35涡轮| ✓ | Full | ✓ | ✓ | ✓ | ✓ | - |
|骆驼3 | ✓ | Full | ✓ | ✓ | ✓ | ✓ | - |
| Llama3 指导 | ✓ | Full | ✓ | ✓ | ✓ | ✓ | - |
|米斯特拉尔7B | ✓ | Full | ✓ | ✓ | ✓ | ✓ | - |
|混合8x7B | ✓ | Full | ✓ | ✓ | ✓ | ✓ | - |
|克劳德3Vision十四行诗| ✓ | Full | ✓ | ✓ | ✓ | ✓ | ✓ |
|克劳德3Vision作品 | ✓ | Full | ✓ | ✓ | ✓ | ✓ | ✓ |
| Claude3Vision俳句 | ✓ | Full | ✓ | ✓ | ✓ | ✓ | ✓ |
| DeepSeekV30324 | ✓ | Full | ✓ | ✓ | ✓ | ✓ | - |
| Gemini2\_5FlashLite | ✓ | Full | ✓ | ✓ | ✓ | ✓ | ✓ |
| Gemini2\_5Flash | ✓ | Full | ✓ | ✓ | ✓ | ✓ | ✓ |
| Gemini2\_5Pro | ✓ | Full | ✓ | ✓ | ✓ | ✓ | ✓ |

基岩模型

#### 基岩模型

Bedrock 模型可通过 AWS Bedrock 访问并使用 InvokeModel 或 Converse API。
标有 **(C)** 的型号仅适用于 Converse，并且需要 `BedrockAPIMethod.Converse`。

##### 人类克劳德（来自基岩）

|型号|温度| JSON Schema |完成 |工具|工具选择 |愿景（图像）|文件|
| --- | --- | --- | --- | --- | --- | --- | --- |
|人类克劳德寓言5 | ✓ | Full | ✓ | ✓ | ✓ | ✓ | ✓ |
|人类克劳德47作品| ✓ | - | ✓ | ✓ | ✓ | ✓ | ✓ |
|人类克劳德46作品 | ✓ | - | ✓ | ✓ | ✓ | ✓ | ✓ |
|人类克劳德45作品| ✓ | - | ✓ | ✓ | ✓ | ✓ | ✓ |
|人类克劳德41作品 | ✓ | - | ✓ | ✓ | ✓ | ✓ | ✓ |
|人类Claude4Opus | ✓ | - | ✓ | ✓ | ✓ | ✓ | ✓ |
|人类克劳德4\_6十四行诗| ✓ | - | ✓ | ✓ | ✓ | ✓ | ✓ |
|人类克劳德4\_5十四行诗| ✓ | - | ✓ | ✓ | ✓ | ✓ | ✓ |
|人类克劳德十四行诗 | ✓ | - | ✓ | ✓ | ✓ | ✓ | ✓ |
| AnthropicClaude4\_5俳句 | ✓ | - | ✓ | ✓ | ✓ | ✓ | ✓ |
|人类克劳德3俳句 | ✓ | - | ✓ | ✓ | ✓ | ✓ | ✓ |

##### 亚马逊新星

|型号|温度| JSON Schema |完成 |工具|工具选择 |愿景（图像）|文件|
| --- | --- | --- | --- | --- | --- | --- | --- |
|亚马逊NovaMicro | ✓ | - | ✓ | ✓ | - | - | - |
|亚马逊NovaLite | ✓ | - | ✓ | ✓ | - | - | - |
|亚马逊NovaPro | ✓ | - | ✓ | ✓ | - | - | - |
|亚马逊NovaPremier | ✓ | - | ✓ | ✓ | - | - | - |

##### Meta Llama（来自基岩）

|型号|温度| JSON Schema |完成 |工具|工具选择 |愿景（图像）|文件|
| --- | --- | --- | --- | --- | --- | --- | --- |
| MetaLlama3\_3\_70B说明 | ✓ | - | ✓ | ✓ | ✓ | - | - |
| MetaLlama3\_2\_90B说明 | ✓ | - | ✓ | ✓ | ✓ | ✓ | ✓ |
| MetaLlama3\_2\_11B指令 | ✓ | - | ✓ | ✓ | ✓ | ✓ | ✓ |
| MetaLlama3\_2\_3B指令 | ✓ | - | ✓ | - | - | - | - |
| MetaLlama3\_2\_1B指令 | ✓ | - | ✓ | - | - | - | - |
| MetaLlama3\_1\_405B说明 | ✓ | - | ✓ | - | - | - | - |
| MetaLlama3\_1\_70B说明 | ✓ | - | ✓ | - | - | - | - |
| MetaLlama3\_1\_8B指令 | ✓ | - | ✓ | - | - | - | - |
| MetaLlama3\_0\_70B说明 | ✓ | - | ✓ | - | - | - | - |
| MetaLlama3\_0\_8B指令 | ✓ | - | ✓ | - | - | - | - |

##### Moonshot Kimi（仅限 Converse）

|型号|温度| JSON Schema |完成 |工具|工具选择 |愿景（图像）|文件|
| --- | --- | --- | --- | --- | --- | --- | --- |
| MoonshotKimiK2\_5 **(C)** | ✓ | - | ✓ | ✓ | ✓ | ✓ | - |
| MoonshotKimiK2Thinking **(C)** | ✓ | - | ✓ | ✓ | ✓ | - | - |

##### MiniMax（仅限 Converse）

|型号|温度| JSON Schema |完成 |工具|工具选择 |愿景（图像）|文件|
| --- | --- | --- | --- | --- | --- | --- | --- |
| MiniMaxM2\_5 **(C)** | ✓ | - | ✓ | ✓ | ✓ | - | - |

##### OpenAI GPT-OSS（仅限 Converse）

|型号|温度| JSON Schema |完成 |工具|工具选择 |愿景（图像）|文件|
| --- | --- | --- | --- | --- | --- | --- | --- |
| OpenAIGptOss120B **(C)** | ✓ | Full | ✓ | ✓ | ✓ | - | - |
| OpenAIGptOss20B **(C)** | ✓ | Full | ✓ | ✓ | ✓ | - | - |

##### Google Gemma 3（仅限 Converse）

|型号|温度| JSON Schema |完成 |工具|工具选择 |愿景（图像）|文件|
| --- | --- | --- | --- | --- | --- | --- | --- |
| GoogleGemma3\_27BIt **(C)** | ✓ | Full | ✓ | ✓ | ✓ | ✓ | ✓ |
| GoogleGemma3\_12BIt **(C)** | ✓ | Full | ✓ | ✓ | ✓ | ✓ | ✓ |
| GoogleGemma3\_4BIt **(C)** | ✓ | - | ✓ | ✓ | ✓ | ✓ | - |

##### 嵌入模型

|型号|嵌入 |
| --- | --- |
| CohereEmbedV4 | ✓ |
| Cohere嵌入英语V3 | ✓ |
| CohereEmbedMultilingualV3 | CohereEmbedMultilingualV3 | CohereEmbedMultilingualV3 | CohereEmbedMultilingualV3 | ✓ |
|亚马逊TitanEmbedTextV2 | ✓ |
|亚马逊泰坦嵌入文本 | ✓ |
