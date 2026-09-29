# 自定义子图

## 创建和配置子图

以下部分提供了为智能体工作流创建子图时的代码模板和常见模式。

### 基本子图创建

自定义子图通常使用以下模式创建：

- 具有指定工具选择策略的子图：

KotlinJava
```
strategy<StrategyInput, StrategyOutput>("strategy-name") {
    val subgraphIdentifier by subgraph<Input, Output>(
        name = "subgraph-name",
        toolSelectionStrategy = ToolSelectionStrategy.ALL
    ) {
        // Define nodes and edges for this subgraph
    }

    nodeStart then subgraphIdentifier then nodeFinish
}
```

```
var strategyBuilder = AIAgentGraphStrategy.builder("strategy-name")
    .withInput(String.class)
    .withOutput(String.class);

var subgraphIdentifier = AIAgentSubgraph.builder("subgraph-name")
    .withToolSelectionStrategy(ToolSelectionStrategy.ALL.INSTANCE)
    .withInput(String.class)
    .withOutput(String.class)
    .define(subgraph -> {
        // Define nodes and edges for this subgraph
    })
    .build();

var strategy = strategyBuilder
    .edge(strategyBuilder.nodeStart, subgraphIdentifier)
    .edge(subgraphIdentifier, strategyBuilder.nodeFinish)
    .build();
```
- 具有指定工具列表的子图（来自已定义工具注册表的工具子集）：

KotlinJava
```
strategy<StrategyInput, StrategyOutput>("strategy-name") {
   val subgraphIdentifier by subgraph<Input, Output>(
       name = "subgraph-name",
       tools = listOf(firstTool, secondTool)
   ) {
        // Define nodes and edges for this subgraph
    }
}
```

```
var strategyBuilder = AIAgentGraphStrategy.builder("strategy-name")
    .withInput(String.class)
    .withOutput(String.class);

var subgraphIdentifier = AIAgentSubgraph.builder("subgraph-name")
    .limitedTools(List.of(firstTool, secondTool))
    .withInput(String.class)
    .withOutput(String.class)
    .define(subgraph -> {
        // Define nodes and edges for this subgraph
    })
    .build();

var strategy = strategyBuilder
    .edge(strategyBuilder.nodeStart, subgraphIdentifier)
    .edge(subgraphIdentifier, strategyBuilder.nodeFinish)
    .build();
```
有关参数和参数值的更多信息，请参见 `subgraph` [API 参考](https://api.koog.ai/agents/agents-core/ai.koog.agents.core.dsl.builder/-a-i-agent-subgraph-builder-base/subgraph.html)。有关工具的更多信息，请参见 [工具](../tools/)。

以下代码示例展示了自定义子图的实际实现：

KotlinJava
```
strategy<String, String>("my-strategy") {
   val mySubgraph by subgraph<String, String>(
      tools = listOf(firstTool, secondTool)
   ) {
        // Define nodes and edges for this subgraph
        val sendInput by nodeLLMRequest()
        val executeToolCall by nodeExecuteTools()
        val sendToolResult by nodeLLMSendToolResults()

        edge(nodeStart forwardTo sendInput)
        edge(sendInput forwardTo executeToolCall onToolCalls { true })
        edge(executeToolCall forwardTo sendToolResult)
        edge(sendToolResult forwardTo nodeFinish onTextMessage { true })
    }
}
```

```
var strategyBuilder = AIAgentGraphStrategy.builder("my-strategy")
        .withInput(String.class)
        .withOutput(String.class);

var sendInput = AIAgentNode.llmRequest(null);
var executeToolCall = AIAgentNode.executeTools(null);
var sendToolResult = AIAgentNode.llmSendToolResults(null);

var mySubgraph = AIAgentSubgraph.builder()
    .limitedTools(List.of(firstTool, secondTool))
    .withInput(String.class)
    .withOutput(String.class)
    .define(subgraph -> {
        // Define nodes and edges for this subgraph
        subgraph
            .edge(AIAgentEdge.builder()
                .from(subgraph.nodeStart)
                .to(sendInput)
                .build()
            )
            .edge(AIAgentEdge.builder()
                .from(sendInput)
                .to(executeToolCall)
                .onToolCalls()
                .build()
            )
            .edge(executeToolCall, sendToolResult)
            .edge(AIAgentEdge.builder()
                .from(sendToolResult)
                .to(subgraph.nodeFinish)
                .onTextMessage()
                .build()
            )
            .build();

    })
    .build();

var strategy = strategyBuilder
    .edge(strategyBuilder.nodeStart, mySubgraph)
    .edge(mySubgraph, strategyBuilder.nodeFinish)
    .build();
```
### 在子图中配置工具

可以通过多种方式为子图配置工具：

- 直接在子图定义中配置：

KotlinJava
```
val mySubgraph by subgraph<String, String>(
   tools = listOf(AskUser)
 ) {
    // Subgraph definition
 }
```

```
var mySubgraph = AIAgentSubgraph.builder()
    .limitedTools(List.of(AskUser.INSTANCE))
    .withInput(String.class)
    .withOutput(String.class)
    .define(subgraph -> {
        // Subgraph definition
    })
    .build();
```
- 从工具注册表：

KotlinJava
```
val mySubgraph by subgraph<String, String>(
    tools = listOf(toolRegistry.getTool("AskUser"))
) {
    // Subgraph definition
}
```

```
var mySubgraph = AIAgentSubgraph.builder()
    .limitedTools(List.of(toolRegistry.getTool("AskUser")))
    .withInput(String.class)
    .withOutput(String.class)
    .define(subgraph -> {
        // Subgraph definition
    })
    .build();
```
- 在执行期间动态地：

KotlinJava
```
// Make a set of tools
this.llm.writeSession {
    tools = tools.filter { it.name in listOf("first_tool_name", "second_tool_name") }
}
```

```
var node = AIAgentNode.builder("node_name")
    .withInput(String.class)
    .withOutput(String.class)
    .withAction((input, ctx) -> {
        // Make a set of tools
        ctx.getLlm().writeSession(session -> {
            session.setTools(session.getTools().stream()
                .filter(t -> List.of("first_tool_name", "second_tool_name").contains(t.getName()))
                .collect(Collectors.toList()));
            return null;
        });
        return input;
    })
    .build();
```
## 高级子图技术

### 多部分策略

复杂的工作流可以分解为多个子图，每个子图处理流程的特定部分：

KotlinJava
```
strategy("complex-workflow") {
   val inputProcessing by subgraph<String, A>(
   ) {
      // Process the initial input
   }

   val reasoning by subgraph<A, B>(
   ) {
      // Perform reasoning based on the processed input
   }

   val toolRun by subgraph<B, C>(
      // Optional subset of tools from the tool registry
      tools = listOf(firstTool, secondTool)
   ) {
      // Run tools based on the reasoning
   }

   val responseGeneration by subgraph<C, String>(
   ) {
      // Generate a response based on the tool results
   }

   nodeStart then inputProcessing then reasoning then toolRun then responseGeneration then nodeFinish

}
```

```
var strategyBuilder = AIAgentGraphStrategy.builder("complex-workflow")
        .withInput(String.class)
        .withOutput(String.class);

var inputProcessing = AIAgentSubgraph.builder()
    .withInput(String.class)
    .withOutput(String.class)
    .define(subgraph -> {
        // Process the initial input
    })
    .build();

var reasoning = AIAgentSubgraph.builder()
    .withInput(String.class)
    .withOutput(String.class)
    .define(subgraph -> {
        // Perform reasoning based on the processed input
    })
    .build();

var toolRun = AIAgentSubgraph.builder()
    // Optional subset of tools from the tool registry
    .limitedTools(List.of(firstTool, secondTool))
    .withInput(String.class)
    .withOutput(String.class)
    .define(subgraph -> {
        // Run tools based on the reasoning
    })
    .build();

var responseGeneration = AIAgentSubgraph.builder()
    .withInput(String.class)
    .withOutput(String.class)
    .define(subgraph -> {
        // Generate a response based on the tool results
    })
    .build();

var strategy = strategyBuilder
    .edge(strategyBuilder.nodeStart, inputProcessing)
    .edge(inputProcessing, reasoning)
    .edge(reasoning, toolRun)
    .edge(toolRun, responseGeneration)
    .edge(responseGeneration, strategyBuilder.nodeFinish)
    .build();
```
## 最佳实践

在使用子图时，请遵循以下最佳实践：

1. **将复杂工作流拆分为子图**：每个子图都应具有清晰、专注的职责。
2. **只传递必要的上下文**：只传递后续子图正常运行所需的信息。
3. **记录子图依赖关系**：清楚地记录每个子图期望从前序子图获得什么，以及它向后序子图提供什么。
4. **单独测试子图**：在将每个子图集成到策略中之前，确保它能在各种输入下正常工作。
5. **考虑 token 用量**：注意 token 用量，尤其是在子图之间传递大量历史记录时。

## 故障排除

### 工具不可用

如果子图中工具不可用：

- 检查工具是否已正确注册到工具注册表中。

### 子图未按定义且预期的顺序运行

如果子图未按定义的顺序执行：

- 检查策略定义，确保子图按正确顺序列出。
- 验证每个子图是否正确将其输出传递给下一个子图。
- 确保你的子图与其余子图相连，并且可从起点（和终点）到达。注意条件边，确保它们覆盖所有可能的继续条件，以免在某个子图或节点中被阻塞。

## 示例

以下示例展示了在真实场景中如何使用子图创建智能体策略。
该代码示例包含三个已定义的子图：`researchSubgraph`、`planSubgraph` 和 `executeSubgraph`，其中每个子图在助手流程中都有明确且不同的用途。

KotlinJava
```
// Define the agent strategy
val strategy = strategy<String, String>("assistant") {

    // A subgraph that includes a tool call
    val researchSubgraph by subgraph<String, String>(
        "research_subgraph",
        tools = listOf(WebSearchTool())
    ) {
        val nodeCallLLM by nodeLLMRequest("call_llm")
        val nodeExecuteTool by nodeExecuteTools()
        val nodeSendToolResult by nodeLLMSendToolResults()

        edge(nodeStart forwardTo nodeCallLLM)
        edge(nodeCallLLM forwardTo nodeExecuteTool onToolCalls { true })
        edge(nodeExecuteTool forwardTo nodeSendToolResult)
        edge(nodeSendToolResult forwardTo nodeExecuteTool onToolCalls { true })
        edge(nodeCallLLM forwardTo nodeFinish onTextMessage { true })
    }

    val planSubgraph by subgraph(
        "plan_subgraph",
        tools = listOf()
    ) {
        val nodeUpdatePrompt by node<String, Unit> { research ->
            llm.writeSession {
                rewritePrompt {
                    prompt("research_prompt") {
                        system(
                            "You are given a problem and some research on how it can be solved." +
                                    "Make step by step a plan on how to solve given task."
                        )
                        user("Research: $research")
                    }
                }
            }
        }
        val nodeCallLLM by nodeLLMRequest("call_llm")

        edge(nodeStart forwardTo nodeUpdatePrompt)
        edge(nodeUpdatePrompt forwardTo nodeCallLLM transformed { "Task: $agentInput" })
        edge(nodeCallLLM forwardTo nodeFinish onTextMessage { true })
    }

    val executeSubgraph by subgraph<String, String>(
        "execute_subgraph",
        tools = listOf(DoAction(), DoAnotherAction()),
    ) {
        val nodeUpdatePrompt by node<String, Unit> { plan ->
            llm.writeSession {
                rewritePrompt {
                    prompt("execute_prompt") {
                        system(
                            "You are given a task and detailed plan how to execute it." +
                                    "Perform execution by calling relevant tools."
                        )
                        user("Execute: $plan")
                        user("Plan: $plan")
                    }
                }
            }
        }
        val nodeCallLLM by nodeLLMRequest("call_llm")
        val nodeExecuteTool by nodeExecuteTools()
        val nodeSendToolResult by nodeLLMSendToolResults()

        edge(nodeStart forwardTo nodeUpdatePrompt)
        edge(nodeUpdatePrompt forwardTo nodeCallLLM transformed { "Task: $agentInput" })
        edge(nodeCallLLM forwardTo nodeExecuteTool onToolCalls { true })
        edge(nodeExecuteTool forwardTo nodeSendToolResult)
        edge(nodeSendToolResult forwardTo nodeExecuteTool onToolCalls { true })
        edge(nodeCallLLM forwardTo nodeFinish onTextMessage { true })
    }

    nodeStart then researchSubgraph then planSubgraph then executeSubgraph then nodeFinish
}
```

```
// Define the agent strategy
var strategyBuilder = AIAgentGraphStrategy.builder("assistant")
    .withInput(String.class)
    .withOutput(String.class);

// A subgraph that includes a tool call
var nodeCallLLM = AIAgentNode.llmRequest(null);
var nodeExecuteTool = AIAgentNode.executeTools(null);
var nodeSendToolResult = AIAgentNode.llmSendToolResults(null);

var researchSubgraph = AIAgentSubgraph.builder("research_subgraph")
    .limitedTools(new WebSearchToolSet())
    .withInput(String.class)
    .withOutput(String.class)
    .define(subgraph -> {
        subgraph
            .edge(AIAgentEdge.builder()
                .from(subgraph.nodeStart)
                .to(nodeCallLLM)
                .build()
            )
            .edge(AIAgentEdge.builder()
                .from(nodeCallLLM)
                .to(nodeExecuteTool)
                .onToolCalls()
                .build()
            )
            .edge(nodeExecuteTool, nodeSendToolResult)
            .edge(AIAgentEdge.builder()
                .from(nodeSendToolResult)
                .to(nodeExecuteTool)
                .onToolCalls()
                .build()
            )
            .edge(AIAgentEdge.builder()
                .from(nodeCallLLM)
                .to(subgraph.nodeFinish)
                .onTextMessage()
                .build()
            )
            .build();
    })
    .build();

var nodeUpdatePrompt = AIAgentNode.builder()
    .withInput(String.class)
    .withOutput(String.class)
    .withAction((research, ctx) -> {
        ctx.getLlm().writeSession(session -> {
            session.setPrompt(Prompt.builder("research_prompt")
                .system(
                    "You are given a problem and some research on how it can be solved." +
                    "Make step by step a plan on how to solve given task."
                )
                .user("Research: " + research)
                .build());
            return null;
        });
        return "Task: " + ctx.getAgentInput();
    })
    .build();
var nodeCallLLMPlan = AIAgentNode.llmRequest(null);

var planSubgraph = AIAgentSubgraph.builder("plan_subgraph")
    .limitedTools(Collections.emptyList())
    .withInput(String.class)
    .withOutput(String.class)
    .define(subgraph -> {
        subgraph
            .edge(subgraph.nodeStart, nodeUpdatePrompt)
            .edge(AIAgentEdge.builder()
                .from(nodeUpdatePrompt)
                .to(nodeCallLLMPlan)
                .build()
            )
            .edge(AIAgentEdge.builder()
                .from(nodeCallLLMPlan)
                .to(subgraph.nodeFinish)
                .onTextMessage()
                .build()
            )
            .build();
    })
    .build();

var nodeUpdatePromptExecute = AIAgentNode.builder()
    .withInput(String.class)
    .withOutput(String.class)
    .withAction((plan, ctx) -> {
        ctx.getLlm().writeSession(session -> {
            session.setPrompt(Prompt.builder("execute_prompt")
                .system(
                    "You are given a task and detailed plan how to execute it." +
                    "Perform execution by calling relevant tools."
                )
                .user("Execute: " + plan)
                .user("Plan: " + plan)
                .build());
            return null;
        });
        return "Task: " + ctx.getAgentInput();
    })
    .build();

var nodeCallLLMExecute = AIAgentNode.llmRequest(null);
var nodeExecuteToolExecute = AIAgentNode.executeTools(null);
var nodeSendToolResultExecute = AIAgentNode.llmSendToolResults(null);

var executeSubgraph = AIAgentSubgraph.builder("execute_subgraph")
    .limitedTools(new ActionToolSet())
    .withInput(String.class)
    .withOutput(String.class)
    .define(subgraph -> {
        subgraph
            .edge(subgraph.nodeStart, nodeUpdatePromptExecute)
            .edge(AIAgentEdge.builder()
                .from(nodeUpdatePromptExecute)
                .to(nodeCallLLMExecute)
                .build()
            )
            .edge(AIAgentEdge.builder()
                .from(nodeCallLLMExecute)
                .to(nodeExecuteToolExecute)
                .onToolCalls()
                .build()
            )
            .edge(nodeExecuteToolExecute, nodeSendToolResultExecute)
            .edge(AIAgentEdge.builder()
                .from(nodeSendToolResultExecute)
                .to(nodeExecuteToolExecute)
                .onToolCalls()
                .build()
            )
            .edge(AIAgentEdge.builder()
                .from(nodeCallLLMExecute)
                .to(subgraph.nodeFinish)
                .onIsInstance(Message.Assistant.class)
                .onTextMessage()
                .build()
            )
            .build();
    })
    .build();

var strategy = strategyBuilder
    .edge(strategyBuilder.nodeStart, researchSubgraph)
    .edge(researchSubgraph, planSubgraph)
    .edge(planSubgraph, executeSubgraph)
    .edge(executeSubgraph, strategyBuilder.nodeFinish)
    .build();
```
