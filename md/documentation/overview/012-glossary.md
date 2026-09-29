# 术语表

## Agent

- **Agent**：一个能够与工具交互、处理复杂工作流并与用户沟通的 AI 实体。
- **LLM (Large Language Model)**：为 Agent 能力提供支持的底层 AI 模型。
- **Message**：Agent 系统中的通信单元，表示来自用户、助手或系统的数据。
- **Prompt**：提供给 LLM 的对话历史，由来自用户、助手和系统的消息组成。
- **System prompt**：提供给 Agent 的指令，用于指导其行为、定义其角色，并提供完成任务所需的关键信息。
- **Context**：LLM 交互发生的环境，可访问对话历史和工具。
- **LLM session**：与 LLM 交互的结构化方式，包含对话历史、可用工具以及发起请求的方法。

## Agent 工作流

- **Strategy**：Agent 的已定义工作流，由顺序执行的子图组成。
  Strategy 定义 Agent 如何处理输入、与工具交互并生成输出。
  Strategy graph 由通过边连接的节点组成，边表示节点之间的转换。

### Strategy graphs

- **Graph**：由节点和边连接而成的结构，用于定义 Agent strategy 工作流。
- **Node**：Agent strategy 工作流的基本构建块，表示特定操作或转换。
- **Edge**：Agent graph 中节点之间的连接，用于定义操作流，通常带有条件来指定何时沿该边执行。
- **Conditions**：决定何时沿特定边执行的规则。
- **Subgraph**：Agent strategy 中自包含的处理单元，拥有自己的工具集、上下文和职责。

## Tools

- **Tool**：Agent 可用于执行特定任务或访问外部系统的函数。Agent 知道可用工具及其参数，但不了解其实现细节。
- **Tool call**：LLM 发出的请求，用于使用提供的参数运行特定工具。其作用类似于函数调用。
- **Tool descriptor**：工具元数据，包含工具名称、描述和参数。
- **Tool registry**：Agent 可用工具的列表。Registry 会告知 Agent 有哪些可用工具。
- **Tool result**：运行工具后产生的输出。例如，如果工具是一个方法，结果就是其返回值。

## History compression

- **History compression**：通过应用各种压缩策略来减小对话历史大小以管理 token 使用量的过程。
  要了解更多信息，请参阅 [History compression](../history-compression/)。

## Features

- **Feature**：用于扩展和增强 AI Agent 功能的组件。

### EventHandler feature

- **EventHandler**：一种支持监控并响应各种 Agent 事件的 feature，提供用于跟踪 Agent 生命周期、处理错误和在整个工作流中处理工具调用的钩子。
