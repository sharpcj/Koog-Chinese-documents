# Ktor 集成：Koog plugin

Beta

此功能属于 beta 模块（`1.3.0-beta`）。API 可能会在未来版本中发生变化。
详情请参阅 [module versioning](../module-versioning/)。

Koog 可以自然融入你的 Ktor server，使你能够在服务端 AI 应用中同时从两侧使用惯用的 Kotlin API。

只需安装一次 Koog plugin，在 application.conf/YAML 或代码中配置 LLM provider，然后就可以直接在 routes 中调用 Agent。不再需要跨模块连接 LLM clients——你的 routes 只需请求一个 Agent 即可使用。

## 概览

`koog-ktor` 模块为服务端 Agent 开发提供符合惯用法的 Kotlin/Ktor 集成：

- 即插即用的 Ktor plugin：在你的 Application 中使用 `install(Koog)`
- 对 OpenAI、Anthropic、Google、OpenRouter、DeepSeek 和 Ollama 的一等支持
- 通过 YAML/CONF 和/或代码进行集中配置
- 使用 prompt、tools、features 设置 Agent；为 routes 提供简单扩展函数
- 直接使用 LLM（execute、executeStreaming、moderate）
- 仅 JVM 可用的 Model Context Protocol (MCP) tools 集成

## 添加依赖

```
dependencies {
    implementation("ai.koog:koog-ktor:$koogVersion")
}
```

## 快速开始

1) 配置 providers（在 `application.yaml` 或 `application.conf` 中）

在 `koog.<provider>` 下使用嵌套键。plugin 会自动读取它们。

```
# application.yaml (Ktor config)
koog:
  openai:
    apikey: ***
    baseUrl: https://api.openai.com
  anthropic:
    apikey: ${ANTH...KEY}
    baseUrl: https://api.anthropic.com
  google:
    apikey: ***
    baseUrl: https://generativelanguage.googleapis.com
  openrouter:
    apikey: ${OPEN...KEY}
    baseUrl: https://openrouter.ai
  deepseek:
    apikey: ${DEEP...KEY}
    baseUrl: https://api.deepseek.com
  # Ollama is enabled when any koog.ollama.* key exists
  ollama:
    enable: true
    baseUrl: http://localhost:11434
```

可选：配置当请求的 provider 未配置时，direct LLM calls 使用的 fallback。

```
koog:
  llm:
    fallback:
      provider: openai
      # see Model identifiers section below
      model: openai.chat.gpt4_1
```

2) 安装 plugin 并定义 routes

```
fun Application.module() {
    install(Koog) {
        // You can also configure providers programmatically (see below)
    }

    routing {
        route("/ai") {
            post("/chat") {
                val userInput = call.receiveText()
                // Create and run a default single‑run agent using a specific model
                val output = aiAgent(
                    strategy = reActStrategy(),
                    model = OpenAIModels.Chat.GPT4_1,
                    input = userInput
                )
                call.respond(HttpStatusCode.OK, output)
            }
        }
    }
}
```

Notes

- aiAgent 需要一个具体模型（LLModel）——按 route、按用途选择。
- 对于更底层的 LLM 访问，请直接使用 llm()（PromptExecutor）。

## 在 routes 中直接使用 LLM

```
post("/llm-chat") {
    val userInput = call.receiveText()

    val messages = llm().execute(
        prompt("chat") {
            system("You are a helpful assistant that clarifies questions")
            user(userInput)
        },
        GoogleModels.Gemini2_5Pro
    )

    // Join all assistant messages into a single string
    val text = messages.joinToString(separator = "\n") { it.content }
    call.respond(HttpStatusCode.OK, text)
}
```

Streaming

```
get("/stream") {
    val flow = llm().executeStreaming(
        prompt("streaming") { user("Stream this response, please") },
        OpenRouterModels.GPT4o
    )

    // Example: buffer and send as one chunk
    val sb = StringBuilder()
    flow.collect { chunk -> sb.append(chunk) }
    call.respondText(sb.toString())
}
```

Moderation

```
post("/moderated-chat") {
    val userInput = call.receiveText()

    val moderation = llm().moderate(
        prompt("moderation") { user(userInput) },
        OpenAIModels.Moderation.Omni
    )

    if (moderation.isHarmful) {
        call.respond(HttpStatusCode.BadRequest, "Harmful content detected")
        return@post
    }

    val output = aiAgent(
        strategy = reActStrategy(),
        model = OpenAIModels.Chat.GPT4_1,
        input = userInput
    )
    call.respond(HttpStatusCode.OK, output)
}
```

## 程序化配置（在代码中）

所有 providers 和 Agent 行为都可以通过 install(Koog) {} 配置。

```
install(Koog) {
    llm {
        openAI(apiKey = System.getenv("OPENAI_API_KEY") ?: "") {
            baseUrl = "https://api.openai.com"
            timeouts { // Default values shown below
                requestTimeout = 15.minutes
                connectTimeout = 60.seconds
                socketTimeout = 15.minutes
            }
        }
        anthropic(apiKey = System.getenv("ANTHROPIC_API_KEY") ?: "")
        google(apiKey = System.getenv("GOOGLE_API_KEY") ?: "")
        openRouter(apiKey = System.getenv("OPENROUTER_API_KEY") ?: "")
        deepSeek(apiKey = System.getenv("DEEPSEEK_API_KEY") ?: "")
        ollama { baseUrl = "http://localhost:11434" }

        // Optional fallback used by PromptExecutor when a provider isn’t configured
        fallback {
            provider = LLMProvider.OpenAI
            model = OpenAIModels.Chat.GPT4_1
        }
    }

    agentConfig {
        // Provide a reusable base prompt for your agents
        prompt(name = "agent") {
            system("You are a helpful server‑side agent")
        }

        // Limit runaway tools/loops
        maxAgentIterations = 10

        // Register tools available to agents by default
        registerTools {
            // tool(::yourTool) // see Tools Overview for details
        }

        // Install agent features (tracing, etc.)
        // install(OpenTelemetry) { /* ... */ }
    }
}
```

## 配置中的模型标识符（fallback）

在 YAML/CONF 中配置 llm.fallback 时，请使用以下标识符格式：

- OpenAI: openai.chat.gpt4\_1, openai.reasoning.o3, openai.costoptimized.gpt4\_1mini, openai.audio.gpt4oaudio, openai.moderation.omni
- Anthropic: anthropic.fable\_5, anthropic.sonnet\_4\_5, anthropic.opus\_4, anthropic.haiku\_4\_5
- Google: google.gemini2\_5pro, google.gemini2\_0flash001
- OpenRouter: openrouter.gpt4o, openrouter.gpt4, openrouter.claude3sonnet
- DeepSeek: deepseek.deepseek-v4-flash, deepseek.deepseek-v4-pro, deepseek.deepseek-chat, deepseek.deepseek-reasoner
- Ollama: ollama.meta.llama3.2, ollama.alibaba.qwq:32b, ollama.groq.llama3-grok-tool-use:8b

Note

- 对于 OpenAI，必须包含类别（chat、reasoning、costoptimized、audio、embeddings、moderation）。
- 对于 Ollama，同时支持 ollama.model 和 ollama..。

## MCP tools（仅 JVM）

在 JVM 上，你可以将来自 MCP server 的 tools 添加到 Agent tool registry：

```
install(Koog) {
    agentConfig {
        mcp {
            // Register via SSE
            sse("https://your-mcp-server.com/sse")

            // Or register via spawned process (stdio transport)
            // process(Runtime.getRuntime().exec("your-mcp-binary ..."))

            // Or from an existing MCP client instance
            // client(existingMcpClient)
        }
    }
}
```

## 为什么选择 Koog + Ktor？

- Kotlin 优先、类型安全的服务端 Agent 开发
- 集中配置，route 代码清晰且易于测试
- 可为每个 route 使用合适的模型，或为 direct LLM calls 自动 fallback
- 生产就绪特性：tools、moderation、streaming 和 tracing