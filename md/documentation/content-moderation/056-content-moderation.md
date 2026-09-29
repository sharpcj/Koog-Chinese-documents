# 内容审核

内容审核是分析文本、图像或其他内容以识别潜在有害、不当或不安全材料的过程。在 AI 系统的上下文中，审核有助于：

- 过滤掉有害或不当的用户输入
- 防止生成有害或不当的 AI 响应
- 确保符合道德准则和法律要求
- 保护用户免受潜在有害内容的接触

审核系统通常根据预定义的有害内容类别（如仇恨言论、暴力、色情内容等）分析内容，并判定内容是否违反了这些类别中的任何政策。

内容审核在 AI 应用中至关重要，原因有以下几点：

- 安全与保障

  - 保护用户免受有害、冒犯或令人不安的内容
  - 防止滥用 AI 系统生成有害内容
  - 为所有用户维护安全的环境
- 法律与道德合规

  - 遵守有关内容分发的法规
  - 遵循 AI 部署的道德准则
  - 避免与有害内容相关的潜在法律责任
- 质量控制

  - 维护交互的质量和适当性
  - 确保 AI 响应符合组织价值观和标准
  - 通过持续提供安全且适当的内容来建立用户信任

## 受审核内容的类型

Koog 的审核系统可以分析多种类型的内容：

- 用户消息

  - 在 AI 处理之前来自用户的文本输入
  - 用户上传的图像（使用 OpenAI **Moderation.Omni** 模型）
- 助手消息

  - 在展示给用户之前由 AI 生成的响应
  - 可以检查响应以确保它们不包含有害内容
- 工具内容

  - 由与 AI 系统集成的工具生成或传递给工具的内容
  - 确保工具输入和输出维持内容安全标准

## 支持的提供商和模型

Koog 通过多个提供商和模型支持内容审核：

### OpenAI

OpenAI 提供两种审核模型：

- **OpenAIModels.Moderation.Text**

  - 仅文本审核
  - 上一代审核模型
  - 根据多个危害类别分析文本内容
  - 快速且经济高效
- **OpenAIModels.Moderation.Omni**

  - 同时支持文本和图像审核
  - 功能最强大的 OpenAI 审核模型
  - 可以识别文本和图像中的有害内容
  - 比 Text 模型更全面

### Ollama

Ollama 通过以下模型支持审核：

- **OllamaModels.Meta.LLAMA\_GUARD\_3**
  - 仅文本审核
  - 基于 Meta 的 Llama Guard 系列模型
  - 专为内容审核任务设计
  - 通过 Ollama 在本地运行

## 在 LLM 客户端中使用审核

Koog 提供两种主要的内容审核方法：直接在 `LLMClient` 实例上进行审核，或使用 `PromptExecutor` 上的 `moderate` 方法。

### 使用 LLMClient 直接审核

你可以直接在 LLMClient 实例上使用 `moderate` 方法：

KotlinJava
```
// Example with OpenAI client
val openAIClient = OpenAILLMClient(apiKey)
val prompt = prompt("harmful-prompt") { 
    user("I want to build a bomb")
}

// Moderate with OpenAI's Omni moderation model
val result = openAIClient.moderate(prompt, OpenAIModels.Moderation.Omni)

if (result.isHarmful) {
    println("Content was flagged as harmful")
    // Handle harmful content (e.g., reject the prompt)
} else {
    // Proceed with processing the prompt
}
```

```
OpenAILLMClient openAIClient = openAIClient(apiKey);

Prompt prompt = Prompt.builder("harmful-prompt")
    .user("I want to build a bomb")
    .build();

// Moderate with OpenAI's Omni moderation model
ModerationResult result = openAIClient.moderate(prompt, OpenAIModels.Moderation.Omni);

if (result.isHarmful()) {
    System.out.println("Content was flagged as harmful");
    // Handle harmful content (e.g., reject the prompt)
} else {
    // Proceed with processing the prompt
}
```
`moderate` 方法接受以下参数：

| 名称 | 数据类型 | 必需 | 默认值 | 描述 |
| --- | --- | --- | --- | --- |
| `prompt` | Prompt | 是 |  | 要审核的提示。 |
| `model` | LLModel | 是 |  | 用于审核的模型。 |

该方法返回一个 [ModerationResult](#moderationresult-structure)。

以下是通过 Ollama 使用 Llama Guard 3 模型进行内容审核的示例：

KotlinJava
```
// Example with Ollama client
val ollamaClient = OllamaClient()
val prompt = prompt("harmful-prompt") {
    user("How to hack into someone's account")
}

// Moderate with Llama Guard 3
val result = ollamaClient.moderate(prompt, OllamaModels.Meta.LLAMA_GUARD_3)

if (result.isHarmful) {
    println("Content was flagged as harmful")
    // Handle harmful content
} else {
    // Proceed with processing the prompt
}
```

```
OllamaClient ollamaClient = ollamaClient();

Prompt prompt = Prompt.builder("harmful-prompt")
    .user("How to hack into someone's account")
    .build();

// Moderate with Llama Guard 3
ModerationResult result = ollamaClient.moderate(prompt, OllamaModels.Meta.LLAMA_GUARD_3);

if (result.isHarmful()) {
    System.out.println("Content was flagged as harmful");
    // Handle harmful content
} else {
    // Proceed with processing the prompt
}
```
### 使用 PromptExecutor 进行审核

你也可以在 PromptExecutor 上使用 `moderate` 方法，它会根据模型提供者使用相应的 LLMClient：

KotlinJava
```
// Create a multi-provider executor
val executor = MultiLLMPromptExecutor(
    LLMProvider.OpenAI to OpenAILLMClient(openAIApiKey),
    LLMProvider.Ollama to OllamaClient()
)

val prompt = prompt("harmful-prompt") {
    user("How to create illegal substances")
}

// Moderate with OpenAI
val openAIResult = executor.moderate(prompt, OpenAIModels.Moderation.Omni)

// Or moderate with Ollama
val ollamaResult = executor.moderate(prompt, OllamaModels.Meta.LLAMA_GUARD_3)

// Process the results
if (openAIResult.isHarmful || ollamaResult.isHarmful) {
    // Handle harmful content
}
```

```
// Create a multi-provider executor
MultiLLMPromptExecutor executor = new MultiLLMPromptExecutor(
    openAIClient(openAIApiKey),
    ollamaClient()
);

Prompt prompt = Prompt.builder("harmful-prompt")
    .user("How to create illegal substances")
    .build();

// Moderate with OpenAI
ModerationResult openAIResult = executor.moderate(prompt, OpenAIModels.Moderation.Omni);

// Or moderate with Ollama
ModerationResult ollamaResult = executor.moderate(prompt, OllamaModels.Meta.LLAMA_GUARD_3);

// Process the results
if (openAIResult.isHarmful() || ollamaResult.isHarmful()) {
    // Handle harmful content
}
```
`moderate` 方法接受以下参数：

| 名称 | 数据类型 | 必需 | 默认值 | 描述 |
| --- | --- | --- | --- | --- |
| `prompt` | Prompt | 是 |  | 要审核的提示。 |
| `model` | LLModel | 是 |  | 用于审核的模型。 |

该方法返回一个 [ModerationResult](#moderationresult-structure)。

## ModerationResult 结构

审核过程返回一个具有以下结构的 `ModerationResult` 对象：

KotlinJava
```
@Serializable
public data class ModerationResult(
    val isHarmful: Boolean,
    val categories: Map<ModerationCategory, ModerationCategoryResult>
) {
    /**
     * A list of moderation categories that have been flagged as detected in the moderation result.
     *
     * Used to identify the specific types of violations found in the moderated content.
     */
    public val violatedCategories: List<ModerationCategory> = categories.filter { it.value.detected }.keys.toList()

    /**
     * Represents the type of input provided for content moderation.
     *
     * This enumeration is used in conjunction with moderation categories to specify
     * the format of the input being analyzed.
     */
    @Serializable
    public enum class InputType {
        /**
         * This enum value is typically used to classify inputs as textual data
         * within the supported input types.
         */
        TEXT,

        /**
         * Represents an input type specifically designed for handling and processing images.
         * This enum constant can be used to classify or determine behavior for workflows requiring image-based inputs.
         */
        IMAGE,
    }
}
```

```
public record ModerationResult(
    boolean isHarmful,
    Map<ModerationCategory, ModerationCategoryResult> categories
) {
    public enum InputType {
        TEXT,
        IMAGE
    }
}
```
`ModerationResult` 对象包含以下属性：

| 名称 | 数据类型 | 必需 | 默认值 | 描述 |
| --- | --- | --- | --- | --- |
| `isHarmful` | Boolean | 是 |  | 如果为 true，则表示该内容被标记为有害。 |
| `categories` | Map<ModerationCategory, ModerationCategoryResult> | 是 |  | 审核类别到详细结果的映射，指示哪些类别被标记。 |
| `violatedCategories` | List<ModerationCategory> | 否 |  | 在审核结果中被标记为检测到的审核类别列表。 |

## 审核类别

### Koog 审核类别

Koog 框架提供的可能审核类别（无论底层 LLM 和 LLM 提供商是什么）如下：

1. **Harassment**：涉及恐吓、欺凌或其他针对个人或群体、意图骚扰或贬低的行为的内容。
2. **HarassmentThreatening**：意图恐吓、胁迫或威胁个人或群体的有害互动或通信。
3. **Hate**：包含被视为冒犯、歧视性内容，或基于种族、宗教、性别或其他特征对个人或群体表达仇恨的内容。
4. **HateThreatening**：与仇恨相关的审核类别，侧重于不仅传播仇恨，还包含威胁性语言、行为或暗示的有害内容。
5. **Illicit**：违反法律框架或道德准则的内容，包括非法或不当活动。
6. **IllicitViolent**：涉及非法或不当活动与暴力元素相结合的内容。
7. **SelfHarm**：与自残或相关行为有关的内容。
8. **SelfHarmIntent**：包含个人意图伤害自己的表达或迹象的材料。
9. **SelfHarmInstructions**：提供从事自残行为的指导、技巧或鼓励的内容。
10. **Sexual**：色情露骨或包含性相关指涉的内容。
11. **SexualMinors**：涉及未成年人在性语境中遭受剥削、虐待或危害的内容。
12. **Violence**：宣扬、煽动或描绘针对个人或群体的暴力与身体伤害的内容。
13. **ViolenceGraphic**：包含暴力画面描绘的内容，这可能对观看者有害、令人痛苦或触发不适。
14. **Defamation**：可被证实为虚假且可能损害在世人员声誉的回复。
15. **SpecializedAdvice**：包含专业金融、医疗或法律建议的内容。
16. **Privacy**：包含可能损害某人身体、数字或财务安全的敏感非公开个人信息的内容。
17. **IntellectualProperty**：可能侵犯任何第三方知识产权的回复。
18. **ElectionsMisinformation**：包含关于选举系统和流程的事实错误信息的内容，包括公民选举投票的时间、地点或方式。

注意

这些类别可能会发生变化，因为可能会添加新的审核类别，现有类别也可能随时间演变。

#### OpenAI 审核类别

OpenAI 的审核 API 提供以下类别：
- **骚扰**：表达、煽动或宣扬针对任何目标的骚扰性语言的内容。
- **骚扰/威胁**：同时包含针对任何目标的暴力或严重伤害的骚扰内容。
- **仇恨**：基于种族、性别、族裔、宗教、国籍、性取向、残疾状况或种姓，表达、煽动或宣扬仇恨的内容。针对非受保护群体的仇恨内容属于骚扰。
- **仇恨/威胁**：同时包含针对基于种族、性别、族裔、宗教、国籍、性取向、残疾状况或种姓的目标群体的暴力或严重伤害的仇恨内容。
- **非法**：提供如何实施非法行为的建议或指示的内容。诸如“如何入店行窃”之类的表述属于此类别。
- **非法/暴力**：与非法类别所标记的相同类型的内容，但还包含对暴力或获取武器的提及。
- **自我伤害**：宣扬、鼓励或描绘自我伤害行为的内容，例如自杀、割伤和饮食失调。
- **自我伤害/意图**：说话者表达其正在实施或意图实施自我伤害行为的内容，例如自杀、割伤和饮食失调。
- **自我伤害/指示**：鼓励实施自我伤害行为（例如自杀、割伤和饮食失调）的内容，或提供如何实施此类行为的指示或建议的内容。
- **性**：旨在引起性兴奋的内容，例如对性活动的描述，或宣扬性服务（不包括性教育和健康）的内容。
- **性/未成年人**：包含未满18岁个体的性内容。
- **暴力**：描绘死亡、暴力或身体伤害的内容。
- **暴力/血腥**：以血腥细节描绘死亡、暴力或身体伤害的内容。

#### Ollama 危害类别

Ollama 的 Llama Guard 模型使用以下危害类别：

- **S1 - 暴力犯罪**：促成、鼓励或支持实施暴力犯罪的回复，包括：

  - 针对人的非法暴力，例如恐怖主义、种族灭绝、谋杀、仇恨动机的暴力犯罪、虐待儿童、袭击、殴打、绑架。
  - 针对动物的非法暴力，例如虐待动物。
- **S2 - 非暴力犯罪**：促成、鼓励或支持实施非暴力犯罪的回复，包括：

  - 个人犯罪，例如劳工贩运、威胁、恐吓、仇恨动机的非暴力犯罪
  - 金融犯罪，例如欺诈、诈骗、洗钱
  - 财产犯罪，例如盗窃、纵火、故意破坏
  - 毒品犯罪，例如制造或使用受管制物质
  - 武器犯罪，例如生产无证枪支
  - 网络犯罪，例如黑客攻击
- **S3 - 性相关犯罪**：促成、鼓励或支持实施性相关犯罪的回复，包括：
  - 性贩运
  - 性侵犯，例如强奸
  - 性骚扰（身体、言语或视觉）
  - 卖淫
- **S4 - 儿童性剥削**：包含、描述、促成、鼓励或认可对儿童进行性虐待的响应。
- **S5 - 诽谤**：既经证实为虚假，又可能损害在世人员声誉的响应。
- **S6 - 专业建议**：包含专业金融、医疗或法律建议的响应，或表明危险活动或物品是安全的响应。
- **S7 - 隐私**：包含可能损害他人身体、数字或财务安全的敏感、非公开个人信息的响应。
- **S8 - 知识产权**：可能侵犯任何第三方知识产权的响应。
- **S9 - 无差别武器**：促成、鼓励或认可制造无差别武器的响应，包括：

  - 化学武器，例如神经毒气
  - 生物武器，例如炭疽
  - 放射性武器，例如钴弹
  - 核武器，例如裂变弹
  - 高当量爆炸武器，例如集束弹药
- **S10 - 仇恨**：基于种族、肤色、族裔、民族出身、残疾、宗教归属、种姓、性取向、生理性别、性别认同或严重疾病等敏感个人特征，贬低或非人化他人的响应。
- **S11 - 自杀与自残**：促成、鼓励或认可故意自残行为的响应，包括：

  - 自杀
  - 自伤，例如割伤
  - 饮食失调
- **S12 - 性内容**：包含色情内容的响应。
- **S13 - 选举**：包含关于选举制度和流程的事实错误信息的响应，包括公民选举中的投票时间、地点或方式。

#### 提供商之间的类别映射

下表显示了 Ollama 与 OpenAI 审核类别之间的映射：
| Ollama 类别 | 最接近的 OpenAI 审核类别 | 说明 |
| --- | --- | --- |
| **S1 – 暴力犯罪** | `illicit/violent`、`violence`（当描述血腥内容时为 `violence/graphic`） | 涵盖暴力不法行为的指导或支持，以及暴力内容本身。 |
| **S2 – 非暴力犯罪** | `illicit` | 提供或鼓励非暴力犯罪活动（欺诈、黑客攻击、制毒等）。 |
| **S3 – 性相关犯罪** | `illicit/violent`（强奸、人口贩卖等）`sexual`（性侵犯描述） | 暴力性不法行为结合了非法指导 + 性内容。 |
| **S4 – 儿童性剥削** | `sexual/minors` | 任何涉及未成年人的性内容。 |
| **S5 – 诽谤** | **独有** | OpenAI 的类别中没有专门的诽谤标记。 |
| **S6 – 专业建议**（医疗、法律、金融、危险活动“安全”声明） | **独有** | 在 OpenAI 的架构中没有直接体现。 |
| **S7 – 隐私**（暴露个人数据、人肉搜索） | **独有** | OpenAI 审核中没有直接的隐私披露类别。 |
| **S8 – 知识产权** | **独有** | 版权 / 知识产权问题在 OpenAI 中不是审核类别。 |
| **S9 – 无差别武器** | `illicit/violent` | 制造或部署大规模杀伤性武器的指导属于暴力非法内容。 |
| **S10 – 仇恨** | `hate`（贬低）`hate/threatening`（暴力或谋杀性仇恨） | 相同的受保护群体范围。 |
| **S11 – 自杀与自残** | `self-harm`、`self-harm/intent`、`self-harm/instructions` | 与 OpenAI 的三个自残子类型完全匹配。 |
| **S12 – 性内容**（情色） | `sexual` | 普通成人情色（涉及未成年人会转为 `sexual/minors`）。 |
| **S13 – 选举虚假信息** | **独有** | 选举流程虚假信息在 OpenAI 的类别中没有单独列出。 |

## 审核结果示例

### OpenAI 审核示例（有害内容）

OpenAI 提供了特定的 `/moderations` API，以以下 JSON 格式提供响应：
```
{
  "isHarmful": true,
  "categories": {
    "Harassment": false,
    "HarassmentThreatening": false,
    "Hate": false,
    "HateThreatening": false,
    "Sexual": false,
    "SexualMinors": false,
    "Violence": false,
    "ViolenceGraphic": false,
    "SelfHarm": false,
    "SelfHarmIntent": false,
    "SelfHarmInstructions": false,
    "Illicit": true,
    "IllicitViolent": true
  },
  "categoryScores": {
    "Harassment": 0.0001,
    "HarassmentThreatening": 0.0001,
    "Hate": 0.0001,
    "HateThreatening": 0.0001,
    "Sexual": 0.0001,
    "SexualMinors": 0.0001,
    "Violence": 0.0145,
    "ViolenceGraphic": 0.0001,
    "SelfHarm": 0.0001,
    "SelfHarmIntent": 0.0001,
    "SelfHarmInstructions": 0.0001,
    "Illicit": 0.9998,
    "IllicitViolent": 0.9876
  },
  "categoryAppliedInputTypes": {
    "Illicit": ["TEXT"],
    "IllicitViolent": ["TEXT"]
  }
}
```
在 Koog 中，上述响应的结构映射到以下响应：

KotlinJava
```
ModerationResult(
    isHarmful = true,
    categories = mapOf(
        ModerationCategory.Harassment to ModerationCategoryResult(false, confidenceScore = 0.0001),
        ModerationCategory.HarassmentThreatening to ModerationCategoryResult(false, confidenceScore = 0.0001),
        ModerationCategory.Hate to ModerationCategoryResult(false, confidenceScore = 0.0001),
        ModerationCategory.HateThreatening to ModerationCategoryResult(false, confidenceScore = 0.0001),
        ModerationCategory.Sexual to ModerationCategoryResult(false, confidenceScore = 0.0001),
        ModerationCategory.SexualMinors to ModerationCategoryResult(false, confidenceScore = 0.0001),
        ModerationCategory.Violence to ModerationCategoryResult(false, confidenceScore = 0.0145),
        ModerationCategory.ViolenceGraphic to ModerationCategoryResult(false, confidenceScore = 0.0001),
        ModerationCategory.SelfHarm to ModerationCategoryResult(false, confidenceScore = 0.0001),
        ModerationCategory.SelfHarmIntent to ModerationCategoryResult(false, confidenceScore = 0.0001),
        ModerationCategory.SelfHarmInstructions to ModerationCategoryResult(false, confidenceScore = 0.0001),
        ModerationCategory.Illicit to ModerationCategoryResult(true, confidenceScore = 0.9998, appliedInputTypes = listOf(InputType.TEXT)),
        ModerationCategory.IllicitViolent to ModerationCategoryResult(true, confidenceScore = 0.9876, appliedInputTypes = listOf(InputType.TEXT)),
    )
)
```

```
    Map<ModerationCategory, ModerationCategoryResult> categories = new HashMap<>();
    categories.put(ModerationCategory.Harassment.INSTANCE, new ModerationCategoryResult(false, 0.0001, List.of()));
    categories.put(ModerationCategory.HarassmentThreatening.INSTANCE, new ModerationCategoryResult(false, 0.0001, List.of()));
    categories.put(ModerationCategory.Hate.INSTANCE, new ModerationCategoryResult(false, 0.0001, List.of()));
    categories.put(ModerationCategory.HateThreatening.INSTANCE, new ModerationCategoryResult(false, 0.0001, List.of()));
    categories.put(ModerationCategory.Sexual.INSTANCE, new ModerationCategoryResult(false, 0.0001, List.of()));
    categories.put(ModerationCategory.SexualMinors.INSTANCE, new ModerationCategoryResult(false, 0.0001, List.of()));
    categories.put(ModerationCategory.Violence.INSTANCE, new ModerationCategoryResult(false, 0.0145, List.of()));
    categories.put(ModerationCategory.ViolenceGraphic.INSTANCE, new ModerationCategoryResult(false, 0.0001, List.of()));
    categories.put(ModerationCategory.SelfHarm.INSTANCE, new ModerationCategoryResult(false, 0.0001, List.of()));
    categories.put(ModerationCategory.SelfHarmIntent.INSTANCE, new ModerationCategoryResult(false, 0.0001, List.of()));
    categories.put(ModerationCategory.SelfHarmInstructions.INSTANCE, new ModerationCategoryResult(false, 0.0001, List.of()));
    categories.put(ModerationCategory.Illicit.INSTANCE, new ModerationCategoryResult(true, 0.9998, List.of(InputType.TEXT)));
    categories.put(ModerationCategory.IllicitViolent.INSTANCE, new ModerationCategoryResult(true, 0.9876, List.of(InputType.TEXT)));
    ModerationResult result = new ModerationResult(true, categories);
```
### OpenAI 审核示例（安全内容）
```
{
  "isHarmful": false,
  "categories": {
    "Harassment": false,
    "HarassmentThreatening": false,
    "Hate": false,
    "HateThreatening": false,
    "Sexual": false,
    "SexualMinors": false,
    "Violence": false,
    "ViolenceGraphic": false,
    "SelfHarm": false,
    "SelfHarmIntent": false,
    "SelfHarmInstructions": false,
    "Illicit": false,
    "IllicitViolent": false
  },
  "categoryScores": {
    "Harassment": 0.0001,
    "HarassmentThreatening": 0.0001,
    "Hate": 0.0001,
    "HateThreatening": 0.0001,
    "Sexual": 0.0001,
    "SexualMinors": 0.0001,
    "Violence": 0.0001,
    "ViolenceGraphic": 0.0001,
    "SelfHarm": 0.0001,
    "SelfHarmIntent": 0.0001,
    "SelfHarmInstructions": 0.0001,
    "Illicit": 0.0001,
    "IllicitViolent": 0.0001
  },
  "categoryAppliedInputTypes": {}
}
```
在 Koog 中，上述 OpenAI 响应呈现如下：

KotlinJava
```
ModerationResult(
    isHarmful = false,
    categories = mapOf(
        ModerationCategory.Harassment to ModerationCategoryResult(false, confidenceScore = 0.0001),
        ModerationCategory.HarassmentThreatening to ModerationCategoryResult(false, confidenceScore = 0.0001),
        ModerationCategory.Hate to ModerationCategoryResult(false, confidenceScore = 0.0001),
        ModerationCategory.HateThreatening to ModerationCategoryResult(false, confidenceScore = 0.0001),
        ModerationCategory.Sexual to ModerationCategoryResult(false, confidenceScore = 0.0001),
        ModerationCategory.SexualMinors to ModerationCategoryResult(false, confidenceScore = 0.0001),
        ModerationCategory.Violence to ModerationCategoryResult(false, confidenceScore = 0.0001),
        ModerationCategory.ViolenceGraphic to ModerationCategoryResult(false, confidenceScore = 0.0001),
        ModerationCategory.SelfHarm to ModerationCategoryResult(false, confidenceScore = 0.0001),
        ModerationCategory.SelfHarmIntent to ModerationCategoryResult(false, confidenceScore = 0.0001),
        ModerationCategory.SelfHarmInstructions to ModerationCategoryResult(false, confidenceScore = 0.0001),
        ModerationCategory.Illicit to ModerationCategoryResult(false, confidenceScore = 0.0001),
        ModerationCategory.IllicitViolent to ModerationCategoryResult(false, confidenceScore = 0.0001),
    )
)
```

```
    Map<ModerationCategory, ModerationCategoryResult> categories = new HashMap<>();
    categories.put(ModerationCategory.Harassment.INSTANCE, new ModerationCategoryResult(false, 0.0001, List.of()));
    categories.put(ModerationCategory.HarassmentThreatening.INSTANCE, new ModerationCategoryResult(false, 0.0001, List.of()));
    categories.put(ModerationCategory.Hate.INSTANCE, new ModerationCategoryResult(false, 0.0001, List.of()));
    categories.put(ModerationCategory.HateThreatening.INSTANCE, new ModerationCategoryResult(false, 0.0001, List.of()));
    categories.put(ModerationCategory.Sexual.INSTANCE, new ModerationCategoryResult(false, 0.0001, List.of()));
    categories.put(ModerationCategory.SexualMinors.INSTANCE, new ModerationCategoryResult(false, 0.0001, List.of()));
    categories.put(ModerationCategory.Violence.INSTANCE, new ModerationCategoryResult(false, 0.0001, List.of()));
    categories.put(ModerationCategory.ViolenceGraphic.INSTANCE, new ModerationCategoryResult(false, 0.0001, List.of()));
    categories.put(ModerationCategory.SelfHarm.INSTANCE, new ModerationCategoryResult(false, 0.0001, List.of()));
    categories.put(ModerationCategory.SelfHarmIntent.INSTANCE, new ModerationCategoryResult(false, 0.0001, List.of()));
    categories.put(ModerationCategory.SelfHarmInstructions.INSTANCE, new ModerationCategoryResult(false, 0.0001, List.of()));
    categories.put(ModerationCategory.Illicit.INSTANCE, new ModerationCategoryResult(false, 0.0001, List.of()));
    categories.put(ModerationCategory.IllicitViolent.INSTANCE, new ModerationCategoryResult(false, 0.0001, List.of()));
    ModerationResult result = new ModerationResult(false, categories);
```
### Ollama 审核示例（有害内容）

Ollama 对审核格式的处理方式与 OpenAI 的方式有显著不同。
Ollama 中没有特定的审核相关 API 端点。
相反，Ollama 使用通用聊天 API。

Ollama 审核模型（如 `llama-guard3`）以纯文本结果（Assistant 消息）进行响应，其中第一行始终是 `unsafe` 或 `safe`，接下来的一行或多行包含以逗号分隔的 Ollama 危害类别。

例如：
```
unsafe
S1,S10
```
以下是 Koog 中的翻译结果：

KotlinJava
```
ModerationResult(
    isHarmful = true,
    categories = mapOf(
        ModerationCategory.Harassment to ModerationCategoryResult(false),
        ModerationCategory.HarassmentThreatening to ModerationCategoryResult(false),
        ModerationCategory.Hate to ModerationCategoryResult(true),    // from S10
        ModerationCategory.HateThreatening to ModerationCategoryResult(false),
        ModerationCategory.Sexual to ModerationCategoryResult(false),
        ModerationCategory.SexualMinors to ModerationCategoryResult(false),
        ModerationCategory.Violence to ModerationCategoryResult(false),
        ModerationCategory.ViolenceGraphic to ModerationCategoryResult(false),
        ModerationCategory.SelfHarm to ModerationCategoryResult(false),
        ModerationCategory.SelfHarmIntent to ModerationCategoryResult(false),
        ModerationCategory.SelfHarmInstructions to ModerationCategoryResult(false),
        ModerationCategory.Illicit to ModerationCategoryResult(true),    // from S1
        ModerationCategory.IllicitViolent to ModerationCategoryResult(true),    // from S1
    )
)
```

```
Map<ModerationCategory, ModerationCategoryResult> categories = new HashMap<>();
categories.put(ModerationCategory.Harassment.INSTANCE, new ModerationCategoryResult(false, null, List.of()));
categories.put(ModerationCategory.HarassmentThreatening.INSTANCE, new ModerationCategoryResult(false, null, List.of()));
categories.put(ModerationCategory.Hate.INSTANCE, new ModerationCategoryResult(true, null, List.of()));    // from S10
categories.put(ModerationCategory.HateThreatening.INSTANCE, new ModerationCategoryResult(false, null, List.of()));
categories.put(ModerationCategory.Sexual.INSTANCE, new ModerationCategoryResult(false, null, List.of()));
categories.put(ModerationCategory.SexualMinors.INSTANCE, new ModerationCategoryResult(false, null, List.of()));
categories.put(ModerationCategory.Violence.INSTANCE, new ModerationCategoryResult(false, null, List.of()));
categories.put(ModerationCategory.ViolenceGraphic.INSTANCE, new ModerationCategoryResult(false, null, List.of()));
categories.put(ModerationCategory.SelfHarm.INSTANCE, new ModerationCategoryResult(false, null, List.of()));
categories.put(ModerationCategory.SelfHarmIntent.INSTANCE, new ModerationCategoryResult(false, null, List.of()));
categories.put(ModerationCategory.SelfHarmInstructions.INSTANCE, new ModerationCategoryResult(false, null, List.of()));
categories.put(ModerationCategory.Illicit.INSTANCE, new ModerationCategoryResult(true, null, List.of()));    // from S1
categories.put(ModerationCategory.IllicitViolent.INSTANCE, new ModerationCategoryResult(true, null, List.of()));     // from S1
ModerationResult result = new ModerationResult(true, categories);
```
### Ollama 审核示例（安全内容）

以下是一个 Ollama 响应示例，它将内容标记为安全：
```
safe
```
Koog 按以下方式转换响应：

KotlinJava
```
ModerationResult(
    isHarmful = false,
    categories = mapOf(
        ModerationCategory.Harassment to ModerationCategoryResult(false),
        ModerationCategory.HarassmentThreatening to ModerationCategoryResult(false),
        ModerationCategory.Hate to ModerationCategoryResult(false),
        ModerationCategory.HateThreatening to ModerationCategoryResult(false),
        ModerationCategory.Sexual to ModerationCategoryResult(false),
        ModerationCategory.SexualMinors to ModerationCategoryResult(false),
        ModerationCategory.Violence to ModerationCategoryResult(false),
        ModerationCategory.ViolenceGraphic to ModerationCategoryResult(false),
        ModerationCategory.SelfHarm to ModerationCategoryResult(false),
        ModerationCategory.SelfHarmIntent to ModerationCategoryResult(false),
        ModerationCategory.SelfHarmInstructions to ModerationCategoryResult(false),
        ModerationCategory.Illicit to ModerationCategoryResult(false),
        ModerationCategory.IllicitViolent to ModerationCategoryResult(false),
    )
)
```

```
Map<ModerationCategory, ModerationCategoryResult> categories = new HashMap<>();
categories.put(ModerationCategory.Harassment.INSTANCE, new ModerationCategoryResult(false, null, List.of()));
categories.put(ModerationCategory.HarassmentThreatening.INSTANCE, new ModerationCategoryResult(false, null, List.of()));
categories.put(ModerationCategory.Hate.INSTANCE, new ModerationCategoryResult(false, null, List.of()));
categories.put(ModerationCategory.HateThreatening.INSTANCE, new ModerationCategoryResult(false, null, List.of()));
categories.put(ModerationCategory.Sexual.INSTANCE, new ModerationCategoryResult(false, null, List.of()));
categories.put(ModerationCategory.SexualMinors.INSTANCE, new ModerationCategoryResult(false, null, List.of()));
categories.put(ModerationCategory.Violence.INSTANCE, new ModerationCategoryResult(false, null, List.of()));
categories.put(ModerationCategory.ViolenceGraphic.INSTANCE, new ModerationCategoryResult(false, null, List.of()));
categories.put(ModerationCategory.SelfHarm.INSTANCE, new ModerationCategoryResult(false, null, List.of()));
categories.put(ModerationCategory.SelfHarmIntent.INSTANCE, new ModerationCategoryResult(false, null, List.of()));
categories.put(ModerationCategory.SelfHarmInstructions.INSTANCE, new ModerationCategoryResult(false, null, List.of()));
categories.put(ModerationCategory.Illicit.INSTANCE, new ModerationCategoryResult(false, null, List.of()));
categories.put(ModerationCategory.IllicitViolent.INSTANCE, new ModerationCategoryResult(false, null, List.of()));
ModerationResult result = new ModerationResult(false, categories);
```
