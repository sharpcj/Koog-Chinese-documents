# GOAP agents

Beta

此功能属于 beta 模块（`1.3.0-beta`）。API 可能会在未来版本中发生变化。
有关详细信息，请参阅[模块版本管理](../../../module-versioning/)。

GOAP 是一种算法式规划方法，它使用 [A\* 搜索](https://en.wikipedia.org/wiki/A*_search_algorithm)寻找满足目标条件、同时最小化总成本的最优动作序列。
不同于使用 LLM 生成计划的 [LLM-based planners](../llm-based-planners/)，GOAP agent 会基于预定义的目标和动作，以算法方式发现动作序列。

GOAP 规划器使用三个主要概念：

- **状态**：表示世界的当前状态。
- **动作**：定义可以执行的事项，包括前置条件、效果（信念）、成本和执行逻辑。
- **目标**：定义目标条件、启发式成本和值函数。

前提条件

确保你的环境和项目满足以下要求：

- JDK 17+
- Kotlin 2.2.0+
- Gradle 8.0+ 或 Maven 3.8+

将 [Koog package](https://central.sonatype.com/artifact/ai.koog/koog-agents/) 添加为依赖：

Gradle (Kotlin)Gradle (Groovy)Maven

build.gradle.kts

```
dependencies {
    // Stable
    implementation("ai.koog:koog-agents:1.3.0")

    // Beta
    implementation("ai.koog:koog-agents-additions:1.3.0-beta")
}
```

build.gradle

```
dependencies {
    // Stable
    implementation 'ai.koog:koog-agents:1.3.0'

    // Beta
    implementation 'ai.koog:koog-agents-additions:1.3.0-beta'
}
```

pom.xml

```
<dependency>
    <!-- Stable -->
    <dependency>
        <groupId>ai.koog</groupId>
        <artifactId>koog-agents-jvm</artifactId>
        <version>1.3.0</version>
    </dependency>

    <!-- Beta -->
    <dependency>
        <groupId>ai.koog</groupId>
        <artifactId>koog-agents-additions-jvm</artifactId>
        <version>1.3.0-beta</version>
    </dependency>
</dependency>
```

从 LLM 提供商获取 API key，或通过 Ollama 运行本地 LLM。
更多信息请参阅 [Quickstart](../quickstart.md)。

本页示例假定你已设置 `OPENAI_API_KEY` 环境变量。

在 Koog 中，你可以通过 DSL 声明式地指定目标和动作来定义 GOAP agent。

要创建 GOAP agent，你需要：

1. 将状态定义为 data class，其中的属性表示与你的目标相关的各个方面。
2. 使用 [goap()](https://api.koog.ai/agents/agents-planner/ai.koog.agents.planner.goap/goap.html) 函数创建 [GOAPPlanner](https://api.koog.ai/agents/agents-planner/ai.koog.agents.planner.goap/-g-o-a-p-planner/index.html) 实例。
   1. 使用 [action()](https://api.koog.ai/agents/agents-planner/ai.koog.agents.planner.goap/-g-o-a-p-planner-builder/action.html) 函数定义带有前置条件和信念的动作。
   2. 使用 [goal()](https://api.koog.ai/agents/agents-planner/ai.koog.agents.planner.goap/-g-o-a-p-planner-builder/goal.html) 函数定义带有完成条件的目标。
3. 使用 [AIAgentPlannerStrategy](https://api.koog.ai/agents/agents-planner/ai.koog.agents.planner/-a-i-agent-planner-strategy/index.html) 包装规划器，并将其传递给 [PlannerAIAgent](https://api.koog.ai/agents/agents-planner/ai.koog.agents.planner/-planner-a-i-agent/index.html) 构造函数。

Note

规划器会选择各个动作及其顺序。
每个动作都包含一个前置条件，该条件必须为真，动作才会被执行；
还包含一个信念，用于定义预测结果。
有关信念的更多信息，请参阅[状态信念与实际执行的比较](#state-beliefs-compared-to-actual-execution)。

在以下示例中，GOAP 负责创建文章的高层规划
（大纲 → 草稿 → 审阅 → 发布），
而 LLM 在每个动作内部执行实际的内容生成。

KotlinJava

```
// Define a state for content creation
data class ContentState(
    val topic: String,
    val hasOutline: Boolean = false,
    val outline: String = "",
    val hasDraft: Boolean = false,
    val draft: String = "",
    val hasReview: Boolean = false,
    val isPublished: Boolean = false
): GoapAgentState<String, String>() {
    override val agentInput = topic
    override fun provideOutput(): String = draft
}

// Create GOAP planner with LLM-powered actions
val planner = goap("content-planner", ::ContentState) {
    // Define actions with preconditions and beliefs
    action(
        name = "Create outline",
        precondition = { state -> !state.hasOutline },
        belief = { state -> state.copy(hasOutline = true, outline = "Outline") },
        cost = { 1.0 }
    ) { ctx, state ->
        // Use LLM to create the outline
        val response = ctx.llm.writeSession {
            appendPrompt {
                user("Create a detailed outline for an article about: ${state.topic}")
            }
            requestLLM()
        }
        state.copy(hasOutline = true, outline = response.parts.filterIsInstance<MessagePart.Text>().joinToString("\n") { it.text })
    }

    action(
        name = "Write draft",
        precondition = { state -> state.hasOutline && !state.hasDraft },
        belief = { state -> state.copy(hasDraft = true, draft = "Draft") },
        cost = { 2.0 }
    ) { ctx, state ->
        // Use LLM to write the draft
        val response = ctx.llm.writeSession {
            appendPrompt {
                user("Write an article based on this outline:\n${state.outline}")
            }
            requestLLM()
        }
        state.copy(hasDraft = true, draft = response.parts.filterIsInstance<MessagePart.Text>().joinToString("\n") { it.text })
    }

    action(
        name = "Review content",
        precondition = { state -> state.hasDraft && !state.hasReview },
        belief = { state -> state.copy(hasReview = true) },
        cost = { 1.0 }
    ) { ctx, state ->
        // Use LLM to review the draft
        val response = ctx.llm.writeSession {
            appendPrompt {
                user("Review this article and suggest improvements:\n${state.draft}")
            }
            requestLLM()
        }
        println("Review feedback: ${response.parts.filterIsInstance<MessagePart.Text>().joinToString("\n") { it.text }}")
        state.copy(hasReview = true)
    }

    action(
        name = "Publish",
        precondition = { state -> state.hasReview && !state.isPublished },
        belief = { state -> state.copy(isPublished = true) },
        cost = { 1.0 }
    ) { ctx, state ->
        println("Publishing article...")
        state.copy(isPublished = true)
    }

    // Define the goal with a completion condition
    goal(
        name = "Published article",
        description = "Complete and publish the article",
        condition = { state -> state.isPublished }
    )
}

// Create and run the agent
val agentConfig = AIAgentConfig(
    prompt = prompt("writer") {
        system("You are a professional content writer.")
    },
    model = OpenAIModels.Chat.GPT4o,
    maxAgentIterations = 20
)

val agent = AIAgent(
    promptExecutor = simpleOpenAIExecutor(System.getenv("OPENAI_API_KEY")),
    strategy = planner,
    agentConfig = agentConfig
)

suspend fun main() {
    val result = agent.run("The Future of AI in Software Development")
    println("Final state: $result")
}
```

```
// Define a state for content creation
static class ContentState extends GoapAgentState<String, String> {
    public String topic;
    public boolean hasOutline = false;
    public String outline = "";
    public boolean hasDraft = false;
    public String draft = "";
    public boolean hasReview = false;
    public boolean isPublished = false;

    public ContentState(String topic) {
        this.topic = topic;
    }

    @Override
    public String getAgentInput() {
        return topic;
    }

    public ContentState copy(boolean hasOutline, String outline, boolean hasDraft,
                             String draft, boolean hasReview, boolean isPublished) {
        ContentState state = new ContentState(topic);
        state.hasOutline = hasOutline;
        state.outline = outline;
        state.hasDraft = hasDraft;
        state.draft = draft;
        state.hasReview = hasReview;
        state.isPublished = isPublished;
        return state;
    }

    @Override
    public String provideOutput() {
        return draft;
    }
}

public static void main(String[] args) {
    var promptExecutor = PromptExecutor.builder()
        .openAI("OPENAI_API_KEY")
        .build();

    var strategy = Planners.goap("content-planner", ContentState::new)
        .action("Create outline", builder -> builder
            .precondition(state -> !state.hasOutline)
            .belief(state -> state.copy(true, "Outline", false, "", false, false))
            .cost(state -> 1.0)
            .execute((context, state) -> {
                String response = context.llm().writeSession(session -> {
                    session.appendPrompt(prompt -> {
                        prompt.user("Create a detailed outline for an article about: " + state.topic);
                        return null;
                    });
                    return session.requestLLM().getParts().stream()
                        .filter(p -> p instanceof MessagePart.Text)
                        .map(p -> ((MessagePart.Text) p).getText())
                        .collect(Collectors.joining());
                });
                return state.copy(true, response, state.hasDraft, state.draft,
                                state.hasReview, state.isPublished);
            })
        )
        .action("Write draft", builder -> builder
            .precondition(state -> state.hasOutline && !state.hasDraft)
            .belief(state -> state.copy(state.hasOutline, state.outline, true, "Draft", false, false))
            .cost(state -> 2.0)
            .execute((context, state) -> {
                String response = context.llm().writeSession(session -> {
                    session.appendPrompt(prompt -> {
                        prompt.user("Write an article based on this outline:\n" + state.outline);
                        return null;
                    });
                    return session.requestLLM().getParts().stream()
                        .filter(p -> p instanceof MessagePart.Text)
                        .map(p -> ((MessagePart.Text) p).getText())
                        .collect(Collectors.joining());
                });
                return state.copy(state.hasOutline, state.outline, true, response,
                                state.hasReview, state.isPublished);
            })
        )
        .action("Review content", builder -> builder
            .precondition(state -> state.hasDraft && !state.hasReview)
            .belief(state -> state.copy(state.hasOutline, state.outline, state.hasDraft,
                                       state.draft, true, false))
            .cost(state -> 1.0)
            .execute((context, state) -> {
                String response = context.llm().writeSession(session -> {
                    session.appendPrompt(prompt -> {
                        prompt.user("Review this article and suggest improvements:\n" + state.draft);
                        return null;
                    });
                    return session.requestLLM().getParts().stream()
                        .filter(p -> p instanceof MessagePart.Text)
                        .map(p -> ((MessagePart.Text) p).getText())
                        .collect(Collectors.joining());
                });
                System.out.println("Review feedback: " + response);
                return state.copy(state.hasOutline, state.outline, state.hasDraft,
                                state.draft, true, state.isPublished);
            })
        )
        .action("Publish", builder -> builder
            .precondition(state -> state.hasReview && !state.isPublished)
            .belief(state -> state.copy(state.hasOutline, state.outline, state.hasDraft,
                                       state.draft, state.hasReview, true))
            .cost(state -> 1.0)
            .execute((context, state) -> {
                System.out.println("Publishing article...");
                return state.copy(state.hasOutline, state.outline, state.hasDraft,
                                state.draft, state.hasReview, true);
            })
        )
        .goal("Published article", builder -> builder
            .description("Complete and publish the article")
            .condition(state -> state.isPublished)
        )
        .build();

    var agent = AIAgent.builder()
        .plannerStrategy(strategy)
        .promptExecutor(promptExecutor)
        .llmModel(OpenAIModels.Chat.GPT4o)
        .systemPrompt("You are a professional content writer.")
        .maxIterations(20)
        .build();

    String result = agent.run("The Future of AI in Software Development");
    System.out.println("Final state: " + result);
}
```

## 自定义成本函数

由于 [A\* 搜索](https://en.wikipedia.org/wiki/A*_search_algorithm)会将成本作为寻找最优动作序列的一个因素，
你可以为动作和目标定义自定义成本函数来引导规划器：

KotlinJava

```
action(
    name = "Expensive operation",
    precondition = { true },
    belief = { state -> state.copy(operationDone = true) },
    cost = { state ->
        // Dynamic cost based on state
        if (state.hasOptimization) 1.0 else 10.0
    }
) { ctx, state ->
    // Execute action
    state.copy(operationDone = true)
}
```

```
.action("Expensive operation", builder -> builder
    .precondition(state -> true)
    .belief(state -> state.copy(true))
    .cost(state -> {
        // Dynamic cost based on state
        return state.hasOptimization ? 1.0 : 10.0;
    })
    .execute((context, state) -> {
        // Execute action
        return state.copy(true);
    })
)
```

## 状态信念与实际执行的比较

GOAP 会区分信念（乐观预测）和实际执行这两个概念：

- **信念**：规划器认为会发生的事情，用于规划。
- **执行**：实际发生的事情，用于真实的状态更新。

这使规划器能够基于预期结果制定计划，同时正确处理实际结果：

KotlinJava

```
action(
    name = "Attempt complex task",
    precondition = { state -> !state.taskComplete },
    belief = { state ->
        // Optimistic belief: task will succeed
        state.copy(taskComplete = true)
    },
    cost = { 5.0 }
) { ctx, state ->
    // Actual execution might fail or have different results
    val success = performComplexTask()
    state.copy(
        taskComplete = success,
        attempts = state.attempts + 1
    )
}
```

```
.action("Attempt complex task", builder -> builder
    .precondition(state -> !state.taskComplete)
    .belief(state -> {
        // Optimistic belief: task will succeed
        return state.copy(true, state.attempts);
    })
    .cost(state -> 5.0)
    .execute((context, state) -> {
        // Actual execution might fail or have different results
        boolean success = performComplexTask();
        return state.copy(success, state.attempts + 1);
    })
)
```