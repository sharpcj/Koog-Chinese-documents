# Attachments

[在 GitHub 上打开](https://github.com/JetBrains/koog/blob/develop/examples/notebooks/Attachments.ipynb)
 [下载 .ipynb](https://raw.githubusercontent.com/JetBrains/koog/develop/examples/notebooks/Attachments.ipynb)

## 设置环境

在深入代码之前，我们先确保 Kotlin Notebook 已准备就绪。
这里我们加载最新的 descriptors 并启用 **Koog** 库，
该库提供了用于处理 AI 模型提供商的清晰 API。

```
// Loads the latest descriptors and activates Koog integration for Kotlin Notebook.
// This makes Koog DSL types and executors available in further cells.
%useLatestDescriptors
%use koog
```

## 配置 API Keys

我们从环境变量读取 API key。这样可以避免将 secrets 写入 notebook 文件，并允许你
切换提供商。你可以设置 `OPENAI_API_KEY`、`ANTHROPIC_API_KEY` 或 `GEMINI_API_KEY`。

```
val apiKey = System.getenv("OPENAI_API_KEY") // or ANTHROPIC_API_KEY, or GEMINI_API_KEY
```

## 创建简单的 OpenAI Executor

executor 封装了认证、base URLs 和正确的默认设置。这里我们使用简单的 OpenAI executor，
但你可以将其替换为 Anthropic 或 Gemini，而无需更改其余代码。

```
// --- Provider selection ---
// For OpenAI-compatible models. Alternatives include:
//   val executor = simpleAnthropicExecutor(System.getenv("ANTHROPIC_API_KEY"))
//   val executor = simpleGeminiExecutor(System.getenv("GEMINI_API_KEY"))
// All executors expose the same high‑level API.
val executor = simpleOpenAIExecutor(apiKey)
```

Koog 的 prompt DSL 允许你添加**结构化 Markdown** 和**附件**。
在此单元格中，我们构建一个 prompt，要求模型生成一张简短的、博客风格的“content card”，并
从本地 `images/` 目录附加两张图片。

```
import ai.koog.prompt.markdown.markdown
import kotlinx.io.files.Path

val prompt = prompt("images-prompt") {
    system("You are professional assistant that can write cool and funny descriptions for Instagram posts.")

    user {
        markdown {
            +"I want to create a new post on Instagram."
            br()
            +"Can you write something creative under my instagram post with the following photos?"
            br()
            h2("Requirements")
            bulleted {
                item("It must be very funny and creative")
                item("It must increase my chance of becoming an ultra-famous blogger!!!!")
                item("It not contain explicit content, harassment or bullying")
                item("It must be a short catching phrase")
                item("You must include relevant hashtags that would increase the visibility of my post")
            }
        }

        attachments {
            image(Path("images/kodee-loving.png"))
            image(Path("images/kodee-electrified.png"))
        }
    }
}
```

## 执行并检查响应

我们针对 `gpt-4.1` 运行 prompt，收集第一条消息并打印其内容。
如果你想使用 streaming，请切换到 Koog 中的 streaming API；如需使用工具，请传入你的工具列表而不是 `emptyList()`。

> 故障排除：
> \* **401/403** — 检查你的 API key/环境变量。
> \* **File not found** — 验证 `images/` 路径。
> \* **Rate limits** — 如有需要，在调用周围添加最小化的 retry/backoff。

```
import kotlinx.coroutines.runBlocking

runBlocking {
    val response = executor.execute(prompt = prompt, model = OpenAIModels.Chat.GPT4_1, tools = emptyList()).first()
    println(response.content)
}
```

```
Caption:
Running on cuteness and extra giggle power! Warning: Side effects may include heart-thief vibes and spontaneous dance parties. 💜🤖💃

Hashtags:  
#ViralVibes #UltraFamousBlogger #CutieAlert #QuirkyContent #InstaFun #SpreadTheLove #DancingIntoFame #RobotLife #InstaFamous #FeedGoals
```

```
runBlocking {
    val response = executor.executeStreaming(prompt = prompt, model = OpenAIModels.Chat.GPT4_1)
    response.collect { print(it) }
}
```

```
Caption:  
Running on good vibes & wi-fi only! 🤖💜 Drop a like if you feel the circuit-joy! #BlogBotInTheWild #HeartDeliveryService #DancingWithWiFi #UltraFamousBlogger #MoreFunThanYourAICat #ViralVibes #InstaFun #BeepBoopFamous
```