# 功能

Agent features 提供了一种扩展和增强 AI agents 功能的方式。
通过 features，你可以：

- 为 agents 添加新能力
- 拦截并修改 agent 行为
- 记录并监控 agent 执行
- 在单个 feature 中为同一事件类型注册多个 handler

Koog framework 开箱即用地提供以下 features：

- [事件处理](agent-event-handlers/)

  ---

  在 agent 执行期间监控并响应特定事件
- [追踪](tracing/)

  ---

  捕获 agent 运行的详细信息
- [聊天记忆](chat-memory/)

  ---

  在 agent 运行之间存储和检索 chat message history
- [长期记忆](long-term-memory/)

  ---

  为 AI agents 添加持久内存
- [Agent 持久化](agent-persistence/)

  ---

  在执行期间的特定点保存和恢复 agent 状态
- [OpenTelemetry](open-telemetry/)

  ---

  从你的 agent 生成、收集并导出 telemetry data（trace）

要了解如何实现自己的 features，请参阅 [自定义功能](custom-features/)。