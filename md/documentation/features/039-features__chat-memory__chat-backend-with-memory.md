# 带记忆的聊天后端

`ChatMemory` 特性的常见模式是一个后端服务，
它代表客户端管理 agent 交互。
每个 HTTP 请求都携带会话 ID，agent 加载匹配的对话历史，
生成并返回响应，然后存储更新后的聊天历史，为下一次交互做好准备。

```
// --- Controller ---

@RestController
class ChatController(private val agentService: ChatAgentService) {
    @PostMapping("/chat")
    suspend fun chat(@RequestBody request: ChatRequest): ChatResponse {
        val reply = agentService.chat(request.sessionId, request.message)
        return ChatResponse(reply)
    }
}

// --- Service ---

@Service
class ChatAgentService(private val executor: SingleLLMPromptExecutor) {
    private val toolRegistry = ToolRegistry {
        // register your tools here
    }

    private val agent = AIAgent(
        promptExecutor = executor,
        llmModel = OpenAIModels.Chat.GPT4oMini,
        systemPrompt = "You are a helpful assistant.",
        toolRegistry = toolRegistry,
    ) {
        install(ChatMemory) {
            chatHistoryProvider = MyDatabaseProvider() // persistent storage
            windowSize(50)
        }
    }

    suspend fun chat(sessionId: String, message: String): String {
        return agent.run(message, sessionId)
    }
}
```

有关使用 Spring Boot 设置 Koog 的完整指南，请参阅
[Spring Boot 集成指南](../../../spring-boot/)。
