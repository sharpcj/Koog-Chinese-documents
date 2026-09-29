# Spring Boot 集成

Beta

此功能属于 beta 模块（`1.3.0-beta`）。API 可能会在未来的版本中发生变化。
详情请参阅[模块版本管理](../module-versioning/)。

Koog 通过其自动配置 starter 提供了无缝的 Spring Boot 集成，让你能够以最少的配置轻松地将 AI
代理集成到 Spring Boot 应用程序中。

## 概述

`koog-spring-boot-starter` 会根据你的应用程序属性自动配置 LLM 客户端，并提供
可直接用于依赖注入的 bean。它支持所有主流的 LLM 提供商，包括：

- OpenAI
- Anthropic
- Google
- OpenRouter
- DeepSeek
- Mistral
- Ollama

## 快速开始

### 1. 添加依赖

将 Koog Spring Boot starter 添加到你的 Gradle 构建配置中：

```
dependencies {
    implementation("ai.koog:koog-spring-boot-starter:$koogVersion")
}
```

或者对于 Maven

```
<dependency>
    <groupId>ai.koog</groupId>
    <artifactId>koog-spring-boot-starter</artifactId>
    <version>$koogVersion</version>
</dependency>
```

确保你的 Kotlin 或 Java 项目具有：
- Spring Boot 3（需要 Java 17 或更高版本）
- Kotlin 版本 2.3.10+
- kotlinx-serialization 版本 1.10.0（即 kotlinx-serialization-core-jvm 和 kotlinx-serialization-json-jvm）

### 2. 配置提供商

在 `application.properties` 中配置你偏好的 LLM 提供商：

```
# OpenAI Configuration
ai.koog.openai.enabled=true
ai.koog.openai.api-key=${OPENAI_API_KEY}
ai.koog.openai.base-url=https://api.openai.com
# Anthropic Configuration  
ai.koog.anthropic.enabled=true
ai.koog.anthropic.api-key=${ANTHROPIC_API_KEY}
ai.koog.anthropic.base-url=https://api.anthropic.com
# Google Configuration
ai.koog.google.enabled=true
ai.koog.google.api-key=${GOOGLE_API_KEY}
ai.koog.google.base-url=https://generativelanguage.googleapis.com
# OpenRouter Configuration
ai.koog.openrouter.enabled=true
ai.koog.openrouter.api-key=${OPENROUTER_API_KEY}
ai.koog.openrouter.base-url=https://openrouter.ai
# DeepSeek Configuration
ai.koog.deepseek.enabled=true
ai.koog.deepseek.api-key=${DEEPSEEK_API_KEY}
ai.koog.deepseek.base-url=https://api.deepseek.com
# Mistral Configuration
ai.koog.mistral.enabled=true
ai.koog.mistral.api-key=${MISTRALAI_API_KEY}
ai.koog.mistral.base-url=https://api.mistral.ai
# Ollama Configuration (local - no API key required)
ai.koog.ollama.enabled=true
ai.koog.ollama.base-url=http://127.0.0.1:11434
```

或者使用 YAML 格式（`application.yml`）：

```
ai:
    koog:
        openai:
            enabled: true
            api-key: ${OPENAI_API_KEY}
            base-url: https://api.openai.com
        anthropic:
            enabled: true
            api-key: ${ANTHROPIC_API_KEY}
            base-url: https://api.anthropic.com
        google:
            enabled: true
            api-key: ${GOOGLE_API_KEY}
            base-url: https://generativelanguage.googleapis.com
        openrouter:
            enabled: true
            api-key: ${OPENROUTER_API_KEY}
            base-url: https://openrouter.ai
        deepseek:
            enabled: true
            api-key: ${DEEPSEEK_API_KEY}
            base-url: https://api.deepseek.com
        mistral:
            enabled: true
            api-key: ${MISTRALAI_API_KEY}
            base-url: https://api.mistral.ai
        ollama:
            enabled: true # Set it to `true` explicitly to activate !!!
            base-url: http://127.0.0.1:11434
```

`ai.koog.PROVIDER.api-key` 和 `ai.koog.PROVIDER.enabled` 属性都用于激活提供商。

如果提供商支持 API Key（如 OpenAI、Anthropic、Google），则 `ai.koog.PROVIDER.enabled` 默认设置为 `true`。

如果提供商不支持 API Key，如 Ollama，则 `ai.koog.PROVIDER.enabled` 默认设置为 `false`，
并且需要在应用程序配置中显式启用该提供商。

在 Spring Boot starter 中，提供商的基础 URL 已设置为其默认值，但你可以在应用程序中覆盖它。

环境变量

建议使用环境变量来存储 API 密钥，以确保其安全性并避免纳入版本控制。
Spring 配置使用 LLM 提供商众所周知的环境变量。
例如，设置环境变量 `OPENAI_API_KEY` 就足以激活 OpenAI 的 Spring 配置。

| LLM 提供商 | 环境变量 |
| --- | --- |
| Open AI | `OPENAI_API_KEY` |
| Anthropic | `ANTHROPIC_API_KEY` |
| Google | `GOOGLE_API_KEY` |
| OpenRouter | `OPENROUTER_API_KEY` |
| DeepSeek | `DEEPSEEK_API_KEY` |
| Mistral | `MISTRALAI_API_KEY` |

### 3. 在你的项目中使用

以下是在 Spring MVC RestController 中使用自动配置执行器的示例。它需要以下内容：
- spring-boot-starter-web 依赖
- 对于 Kotlin，需要添加 kotlinx-coroutines-core 和 kotlinx-coroutines-reactor 依赖（Java 版本调用阻塞的 `execute` 方法）
- 通过属性启用 Anthropic（ai.koog.anthropic.enabled=true）

KotlinJava

```
import ai.koog.prompt.dsl.prompt
import ai.koog.prompt.executor.clients.anthropic.AnthropicModels
import ai.koog.prompt.executor.model.PromptExecutor
import org.springframework.http.ResponseEntity
import org.springframework.web.bind.annotation.PostMapping
import org.springframework.web.bind.annotation.RequestBody
import org.springframework.web.bind.annotation.RequestMapping
import org.springframework.web.bind.annotation.RestController

@RestController
@RequestMapping("/api/chat")
class ChatController(private val anthropicExecutor: PromptExecutor) {

    @PostMapping
    suspend fun chat(@RequestBody request: ChatRequest): ResponseEntity<ChatResponse> {
        return try {
            val prompt = prompt("chat") {
                system("You are a helpful assistant")
                user(request.message)
            }

            val result = anthropicExecutor.execute(prompt, AnthropicModels.Haiku_4_5)
            ResponseEntity.ok(ChatResponse(result.first().content))
        } catch (e: Exception) {
            ResponseEntity.internalServerError()
                .body(ChatResponse("Error processing request"))
        }
    }
}

data class ChatRequest(val message: String)
data class ChatResponse(val response: String)
```

```
import ai.koog.prompt.Prompt;
import ai.koog.prompt.executor.clients.anthropic.AnthropicModels;
import ai.koog.prompt.executor.model.PromptExecutor;
import ai.koog.prompt.message.Message;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import java.util.List;

@RestController
@RequestMapping("/api/chat")
public class ChatController {
    private final PromptExecutor anthropicExecutor;

    public ChatController(PromptExecutor anthropicExecutor) {
        this.anthropicExecutor = anthropicExecutor;
    }

    @PostMapping
    public ResponseEntity<ChatResponse> chat(@RequestBody ChatRequest request) {
        try {
            Prompt prompt = Prompt.builder("chat")
                    .system("You are a helpful assistant")
                    .user(request.message())
                    .build();

            List<Message.Response> result = anthropicExecutor.execute(prompt, AnthropicModels.Haiku_4_5);
            return ResponseEntity.ok(new ChatResponse(result.get(0).getContent()));
        } catch (Exception e) {
            return ResponseEntity.internalServerError()
                    .body(new ChatResponse("Error processing request"));
        }
    }
}

record ChatRequest(String message) {
}

record ChatResponse(String response) {
}
```

Spring Framework 通过 bean 名称（`anthropicExecutor`）注入了 Anthropic 的执行器，
但你也可以使用 `@Qualifier` 注解注入多个 `PromptExecutor` bean（参见下方的“多个 bean 错误”）。

## 高级用法

### LLM 提供商回退

配置多个 LLM 提供商后，你可以通过 `MultiLLMPromptExecutor` 向多个 LLM 发送请求：

KotlinJava

```
import ai.koog.prompt.dsl.prompt
import ai.koog.prompt.executor.clients.anthropic.AnthropicModels.Haiku_4_5
import ai.koog.prompt.executor.clients.openai.OpenAIModels.Chat.GPT4oMini
import ai.koog.prompt.executor.clients.openrouter.OpenRouterModels.Claude3Haiku
import ai.koog.prompt.executor.llms.MultiLLMPromptExecutor
import org.slf4j.Logger
import org.slf4j.LoggerFactory
import org.springframework.stereotype.Service

@Service
class RobustAIService(private val multiLLMPromptExecutor: MultiLLMPromptExecutor) {

    private val llms = listOf(GPT4oMini, Haiku_4_5, Claude3Haiku)

    suspend fun generateWithFallback(input: String): String {
        val prompt = prompt("robust") {
            system("You are a helpful AI assistant")
            user(input)
        }

        for (llm in llms) {
            try {
                val result = multiLLMPromptExecutor.execute(prompt, llm)
                return result.first().content
            } catch (e: Exception) {
                logger.warn("{} executor failed, trying next: {}", llm.id, e.message)
            }
        }

        throw IllegalStateException("All AI providers failed")
    }

    companion object {
        private val logger = LoggerFactory.getLogger(RobustAIService::class.java)
    }
}
```

```
import ai.koog.prompt.Prompt;
import ai.koog.prompt.executor.clients.anthropic.AnthropicModels;
import ai.koog.prompt.executor.clients.openai.OpenAIModels;
import ai.koog.prompt.executor.clients.openrouter.OpenRouterModels;
import ai.koog.prompt.executor.llms.MultiLLMPromptExecutor;
import ai.koog.prompt.llm.LLModel;
import ai.koog.prompt.message.Message;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Service;

import java.util.List;

@Service
public class RobustAIService {
    private static final Logger logger = LoggerFactory.getLogger(RobustAIService.class);

    private final List<LLModel> llms = List.of(OpenAIModels.Chat.GPT4oMini, AnthropicModels.Haiku_4_5, OpenRouterModels.Claude3Haiku);

    private final MultiLLMPromptExecutor multiLLMPromptExecutor;

    public RobustAIService(MultiLLMPromptExecutor multiLLMPromptExecutor) {
        this.multiLLMPromptExecutor = multiLLMPromptExecutor;
    }

    public String generateWithFallback(String input) {
        Prompt prompt = Prompt.builder("robust")
            .system("You are a helpful AI assistant")
            .user(input)
            .build();

        for (LLModel llm : llms) {
            try {
                List<Message.Response> result = multiLLMPromptExecutor.execute(prompt, llm);
                return result.get(0).getContent();
            } catch (Exception e) {
                logger.warn("{} executor failed, trying next: {}", llm.getId(), e.getMessage());
            }
        }

        throw new IllegalStateException("All AI providers failed");
    }
}
```

你也可以注册自己的 `MultiLLMPromptExecutor` bean 并向其传递 `FallbackPromptExecutorSettings`。
要覆盖自动配置的 bean，你可以使用 `@Primary` 注解。

## 配置参考

### 可用属性

| 属性 | 描述 | Bean 条件 | 默认值 |
| --- | --- | --- | --- |
| `ai.koog.openai.api-key` | OpenAI API 密钥 | `openAIExecutor` bean 必需 | - |
| `ai.koog.openai.base-url` | OpenAI 基础 URL | 可选 | `https://api.openai.com` |
| `ai.koog.anthropic.api-key` | Anthropic API 密钥 | `anthropicExecutor` bean 必需 | - |
| `ai.koog.anthropic.base-url` | Anthropic 基础 URL | 可选 | `https://api.anthropic.com` |
| `ai.koog.google.api-key` | Google API 密钥 | `googleExecutor` bean 必需 | - |
| `ai.koog.google.base-url` | Google 基础 URL | 可选 | `https://generativelanguage.googleapis.com` |
| `ai.koog.openrouter.api-key` | OpenRouter API 密钥 | `openRouterExecutor` bean 必需 | - |
| `ai.koog.openrouter.base-url` | OpenRouter 基础 URL | 可选 | `https://openrouter.ai` |
| `ai.koog.deepseek.api-key` | DeepSeek API 密钥 | `deepSeekExecutor` bean 必需 | - |
| `ai.koog.deepseek.base-url` | DeepSeek 基础 URL | 可选 | `https://api.deepseek.com` |
| `ai.koog.mistral.api-key` | Mistral API 密钥 | `mistralAIExecutor` bean 必需 | - |
| `ai.koog.mistral.base-url` | Mistral 基础 URL | 可选 | `https://api.mistral.ai` |
| `ai.koog.ollama.base-url` | Ollama 基础 URL | 可选 | `http://127.0.0.1:11434` |

### Bean 名称

自动配置会创建以下 bean（在已配置的情况下）：

- `openAIExecutor` - OpenAI 执行器（需要 `ai.koog.openai.api-key`）
- `anthropicExecutor` - Anthropic 执行器（需要 `ai.koog.anthropic.api-key`）
- `googleExecutor` - Google 执行器（需要 `ai.koog.google.api-key`）
- `openRouterExecutor` - OpenRouter 执行器（需要 `ai.koog.openrouter.api-key`）
- `deepSeekExecutor` - DeepSeek 执行器（需要 `ai.koog.deepseek.api-key`）
- `mistralAIExecutor` - Mistral AI 执行器（需要 `ai.koog.mistral.api-key`）
- `ollamaExecutor` - Ollama 执行器（需要 `ai.koog.ollama.enabled=true`）
- `multiLLMPromptExecutor` - MultiLLMPromptExecutor

## 故障排除

### 常见问题

**错误：没有可用的 'PromptExecutor' 类型的合格 bean**

**解决方案：** 确保你已在属性文件中配置了至少一个提供商。

**错误：存在多个 'PromptExecutor' 类型的合格 bean**

**解决方案：** 使用 `@Qualifier` 指定你想要的 bean：

KotlinJava

```
@Service
class MyService(
    @Qualifier("openAIExecutor") private val openAIExecutor: PromptExecutor,
    @Qualifier("anthropicExecutor") private val anthropicExecutor: PromptExecutor
) {
    // ...
}
```

```
@Service
public class MyService {
    private final PromptExecutor openAIExecutor;
    private final PromptExecutor anthropicExecutor;

    public MyService(@Qualifier("openAIExecutor") PromptExecutor openAIExecutor,
                     @Qualifier("anthropicExecutor") PromptExecutor anthropicExecutor) {
        this.openAIExecutor = openAIExecutor;
        this.anthropicExecutor = anthropicExecutor;
    }
    // ...
}
```

**错误：需要 API 密钥但未提供**

**解决方案：** 检查你的环境变量是否已正确设置并且可被 Spring Boot 应用程序访问。

## 最佳实践

1. **环境变量**：始终使用环境变量存储 API 密钥
2. **可空注入**：使用可空类型来处理未配置提供商的情况
3. **回退逻辑**：在使用多个提供商时实现回退机制
4. **错误处理**：在生产代码中始终将执行器调用包裹在 try-catch 块中
5. **测试**：在测试中使用 mock 以避免进行实际的 API 调用
6. **配置验证**：在使用执行器之前检查它们是否可用

## 后续步骤

- 了解[基础代理](../agents/basic-agents/)以构建最小的 AI 工作流
- 探索[基于图的代理](../agents/graph-based-agents/)以应对高级用例
- 查看[工具概述](../tools/)以扩展代理的能力
- 查看[示例](../examples/)了解真实世界的实现
- 阅读[术语表](../glossary/)以更好地理解该框架
