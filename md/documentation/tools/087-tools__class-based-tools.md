# 基于类的工具

本节介绍专为需要增强灵活性和自定义行为的场景设计的API。
使用Kotlin中的这种方法，您可以完全控制工具，包括其参数、元数据、执行逻辑以及如何注册和调用。在Java中，使用具有基于反射的配准的基于注释的方法创建工具。

这种级别的控制非常适合创建可扩展基本用例的复杂工具，从而实现与座席会话和工作流的无缝集成。

本页介绍如何在Kotlin和Java中实现工具，通过注册表管理工具，调用它们，并在基于节点的代理架构中使用。

備註

API是Kotlin的多平台。Java工具使用基于注释的方法实现，并通过反射进行注册。这使您可以在Kotlin的不同平台上使用相同的工具，而Java提供完整的JVM互操作性。

## 工具实施

Koog框架提供了以下实现工具的方法：

对于Kotlin ：

- 对所有工具使用基类`Tool`。当需要返回非文本结果或需要完全控制工具行为时，应使用此类。
- 使用`SimpleTool`类扩展基本`Tool`类并简化创建返回文本结果的工具。对于以下情况，您应该使用此方法：
工具只需要返回文本。

两种方法都使用相同的核心组件，但在实现和返回的结果上有所不同。

对于Java ：

- 使用基于注释的方法（ `@Tool`和`@LLMDescription` ）和基于反射的配准。这是Java互操作性的推荐方法，因为由于挂起功能限制，不支持从Java中对Kotlin的`Tool`或`SimpleTool`进行子类划分。

## # 工具类（ Kotlin ）

[`Tool<Args, Result>`](https://api.koog.ai/agents/agents-tools/ai.koog.agents.core.tools/-tool/index.html)抽象类是在Kotlin中创建工具的基类。
它允许您创建接受特定参数类型（ `Args` ）并返回各种类型（ `Result` ）结果的工具。

每个工具由以下组件组成：

组件说明
| --- | --- |
| `Args` |定义工具所需参数的可序列化数据类。|
| `Result` |工具返回的可序列化类型的结果。如果要以自定义格式显示工具结果，请继承[ToolResult.TextSerializable](https://api.koog.ai/agents/agents-tools/ai.koog.agents.core.tools/-tool-result/-text-serializable/index.html)类并实现`textForLLM(): String`方法|
| `argsSerializer` |定义工具参数反序列化的重写变量。另请参阅[argsSerializer](https://api.koog.ai/agents/agents-tools/ai.koog.agents.core.tools/-tool/args-serializer.html)。|
| `resultSerializer` |定义工具结果反序列化的重写变量。另请参阅[resultSerializer](https://api.koog.ai/agents/agents-tools/ai.koog.agents.core.tools/-tool/result-serializer.html)。如果您选择继承[ToolResult.TextSerializable](https://api.koog.ai/agents/agents-tools/ai.koog.agents.core.tools/-tool-result/-text-serializable/index.html) ，请考虑使用`ToolResultUtils.toTextSerializer()` |
| `descriptor` |指定工具元数据的重写变量： - `name` - `description` - `requiredParameters` （默认为空） - `optionalParameters` （默认为空）另请参阅[描述符](https://api.koog.ai/agents/agents-tools/ai.koog.agents.core.tools/-tool/descriptor.html)。|
| `execute()` |实现工具逻辑的函数。它接受`Args`类型的参数并返回`Result`类型的结果。另请参阅execute ()。|

Java Implementation

In Java, instead of subclassing `Tool<Args, Result>`, use annotation-based methods with `@Tool` and `@LLMDescription`. The framework handles serialization and registration automatically through reflection. For more
details, see [Annotation-based methods](#annotation-based-methods-java) below.

Tip

Ensure your tools have clear descriptions and well-defined parameter names to make it easier for the LLM to understand and use them properly. In Kotlin, use the `descriptor` property; in Java, use `@LLMDescription` annotations.

### # Usage example

Here is an example of a custom tool implementation using the `Tool` class that returns a numeric result:

Kotlin

```
// Implement a simple calculator tool that adds two digits
object CalculatorTool : Tool<CalculatorTool.Args, Int>(
    argsType = typeToken<Args>(),
    resultType = typeToken<Int>(),
    name = "calculator",
    description = "A simple calculator that can add two digits (0-9)."
) {

    // Arguments for the calculator tool
    @Serializable
    data class Args(
        @property:LLMDescription("The first digit to add (0-9)")
        val digit1: Int,
        @property:LLMDescription("The second digit to add (0-9)")
        val digit2: Int
    ) {
        init {
            require(digit1 in 0..9) { "digit1 must be a single digit (0-9)" }
            require(digit2 in 0..9) { "digit2 must be a single digit (0-9)" }
        }
    }

    // Function to add two digits
    override suspend fun execute(args: Args): Int = args.digit1 + args.digit2
}
```

After implementing your tool, you need to add it to a tool registry and then use it with an agent. For details, see [Tool registry](../#tool-registry).

For more details, see [API reference](https://api.koog.ai/agents/agents-tools/ai.koog.agents.core.tools/-tool/index.html).

### # Reading the agent context from a tool

Tools that need the agent's full state (LLM context, run id, configuration, storage, ...) extend `AgentContextAwareTool<Args, Result>` instead of `Tool<Args, Result>`. The framework injects the live `AIAgentContext` driving the call, and the tool receives it as a typed parameter rather than reading it out of the argument schema.

Kotlin

```
// A tool that reads the live AIAgentContext driving the call.
object TracingCalculatorTool : AgentContextAwareTool<TracingCalculatorTool.Args, Int>(
    argsType = typeToken<Args>(),
    resultType = typeToken<Int>(),
    name = "tracing_calculator",
    description = "Adds two digits and emits a log line tagged with the agent run id."
) {
    @Serializable
    data class Args(
        @property:LLMDescription("The first digit to add (0-9)")
        val digit1: Int,
        @property:LLMDescription("The second digit to add (0-9)")
        val digit2: Int
    )

    override suspend fun execute(args: Args, context: AIAgentContext): Int {
        val runId = context.runId
        // ... use runId for cross-cutting context (logging, tracing, correlation)
        return args.digit1 + args.digit2
    }
}
```

`AgentContextAwareTool` is dispatched by the framework via a per-call `ToolCallMetadata` side channel that the framework manages on the tool's behalf. Invoking such a tool outside an agent run throws `IllegalStateException` because no `AIAgentContext` was injected; production code should always go through `ContextualAgentEnvironment`, and unit tests can supply the context explicitly via `ToolCallMetadata.of(AgentContextAwareTool.AgentContextKey to context)`.

### # Reading raw per-call metadata

A small number of tools want to read caller- or feature-contributed entries that are *not* the agent context (for example a distributed-tracing span id contributed by an observability feature). These tools extend `ToolBase<Args, Result>` directly, which exposes the full `ToolCallMetadata` bag:

Kotlin

```
object SpanAwareCalculatorTool : ToolBase<SpanAwareCalculatorTool.Args, Int>(
    argsType = typeToken<Args>(),
    resultType = typeToken<Int>(),
    name = "span_aware_calculator",
    description = "Adds two digits, propagating a tracing span id from caller or feature metadata."
) {
    @Serializable
    data class Args(
        @property:LLMDescription("The first digit to add (0-9)")
        val digit1: Int,
        @property:LLMDescription("The second digit to add (0-9)")
        val digit2: Int
    )

    override suspend fun execute(args: Args, metadata: ToolCallMetadata): Int {
        val traceSpanId = metadata["trace.span.id"] as? String
        // ... use traceSpanId for cross-cutting context (logging, tracing, correlation)
        return args.digit1 + args.digit2
    }
}
```

Callers can pass metadata through `SafeTool.execute(args, serializer, metadata)` or directly through `AIAgentEnvironment.executeTool(toolCall, metadata)`. Features can contribute metadata for every tool call during installation by calling `pipeline.provideToolCallMetadata(this) { eventContext -> mapOf(...) }`. Caller-supplied metadata always wins over feature contributions on key collision.

Existing tools that extend `Tool<Args, Result>` and override `execute(args)` continue to work unchanged: the framework dispatches them through the same path and discards any `ToolCallMetadata`. To opt in to metadata, switch to `AgentContextAwareTool` (typed context access) or `ToolBase` (raw bag access).

## # SimpleTool class (Kotlin)

The [`SimpleTool<Args>`](https://api.koog.ai/agents/agents-tools/ai.koog.agents.core.tools/-simple-tool/index.html) abstract class extends `Tool<Args, ToolResult.Text>` and simplifies the creation of tools that return text results.

Each simple tool consists of the following components:

| Component | Description |
| --- | --- |
| `Args` | The serializable data class that defines arguments required for the custom tool. |
| `argsSerializer` | The overridden variable that defines how the arguments for the tool are serialized. See also [argsSerializer](https://api.koog.ai/agents/agents-tools/ai.koog.agents.core.tools/-tool/args-serializer.html). |
| `descriptor` | The overridden variable that specifies tool metadata: - `name` - `description` - `requiredParameters` (empty by default)  - `optionalParameters` (empty by default)  See also [descriptor](https://api.koog.ai/agents/agents-tools/ai.koog.agents.core.tools/-tool/descriptor.html). |
| `doExecute()` | The overridden function that describes the main action performed by the tool. It takes arguments of type `Args` and returns a `String`. See also [doExecute()](https://api.koog.ai/agents/agents-tools/ai.koog.agents.core.tools/-simple-tool/do-execute.html). |

Java Implementation

In Java, the equivalent approach is to use annotation-based methods that return `String`. The framework automatically handles the text result wrapping. For more details, see [Annotation-based methods](#annotation-based-methods-java) below.

Tip

Ensure your tools have clear descriptions and well-defined parameter names to make it easier for the LLM to understand and use them properly. In Kotlin, use the `descriptor` and constructor parameters; in Java, use `@Tool` and `@LLMDescription` annotations.

### # Usage example

Here is an example of a custom tool implementation using `SimpleTool` in Kotlin:

Kotlin

```
// Create a tool that casts a string expression to a double value
object CastToDoubleTool : SimpleTool<CastToDoubleTool.Args>(
    argsType = typeToken<Args>(),
    name = "cast_to_double",
    description = "casts the passed expression to double or returns 0.0 if the expression is not castable"
) {
    // Define tool arguments
    @Serializable
    data class Args(
        @property:LLMDescription("An expression to case to double")
        val expression: String,
        @property:LLMDescription("A comment on how to process the expression")
        val comment: String
    )

    // Function that executes the tool with the provided arguments
    override suspend fun execute(args: Args): String {
        return "Result: ${castToDouble(args.expression)}, " + "the comment was: ${args.comment}"
    }

    // Function to cast a string expression to a double value
    private fun castToDouble(expression: String): Double {
        return expression.toDoubleOrNull() ?: 0.0
    }
}
```

## # Annotation-based methods (Java)

To implement tools in Java, instead of subclassing `Tool` or `SimpleTool`, use annotation-based methods with `@Tool` and
`@LLMDescription`. Koog handles serialization and registration automatically through reflection. To learn more about the
implementation, see Java examples below.

### # Usage examples

This is an example of a tool implementation in Java, equivalent to using the `Tool` class in Kotlin.

Java

```
// Java equivalent: implement the tool as a Java method and register it via ToolRegistry.builder().
// This is the recommended Java interop path instead of subclassing the Kotlin Tool base class.
public final class CalculatorTool {
    private CalculatorTool() {}

    @Tool(customName = "calculator")
    @LLMDescription(description = "A simple calculator that can add two digits (0-9).")
    public static int calculator(
            @LLMDescription(description = "The first digit to add (0-9)") int digit1,
            @LLMDescription(description = "The second digit to add (0-9)") int digit2
    ) {
        if (digit1 < 0 || digit1 > 9) throw new IllegalArgumentException("digit1 must be a single digit (0-9)");
        if (digit2 < 0 || digit2 > 9) throw new IllegalArgumentException("digit2 must be a single digit (0-9)");
        return digit1 + digit2;
    }

    public static ToolRegistry registry() throws NoSuchMethodException {
        return ToolRegistry.builder()
            .tool(CalculatorTool.class.getMethod("calculator", int.class, int.class))
            .build();
    }
}
// Note: Subclassing the Kotlin Tool<TArgs, TResult> and overriding a suspend execute(...) from Java is not supported.
// The Java interop uses reflection-based registration of Java methods as tools.
```

Here is an example of a tool implementation in Java, equivalent to using the `SimpleTool` class in Kotlin. This example
implements a simple tool that returns a text result.

Java

```
// Java equivalent of SimpleTool: provide a Java method and register it as a tool.
public final class CastToDoubleTool {
    private CastToDoubleTool() {}

    @Tool(customName = "cast_to_double")
    @LLMDescription(description = "casts the passed expression to double or returns 0.0 if the expression is not castable")
    public static String castToDouble(
            @LLMDescription(description = "An expression to case to double") String expression,
            @LLMDescription(description = "A comment on how to process the expression") String comment
    ) {
        double value;
        try {
            value = Double.parseDouble(expression);
        } catch (Exception e) {
            value = 0.0;
        }
        return "Result: " + value + ", the comment was: " + comment;
    }

    public static ToolRegistry registry() throws NoSuchMethodException {
        return ToolRegistry.builder()
            .tool(CastToDoubleTool.class.getMethod("castToDouble", String.class, String.class))
            .build();
    }
}
// Note: Extending Kotlin SimpleTool<TArgs> from Java is not required; registering a Java method is the idiomatic approach.
```

## # Sending tool result to LLM in custom format

For Kotlin:

If you are not happy with JSON results sent to LLM (in some cases, LLMs can work better if tool output is structured as Markdown, for instance), you have to follow the following steps:

1. Implement `ToolResult.TextSerializable` interface, and override `textForLLM()` method
2. Override `resultSerializer` using `ToolResultUtils.toTextSerializer<T>()`

For Java:

Return formatted text (such as Markdown) directly as a `String` from your annotated method. The framework handles this automatically.

### # Example

Here is an example showing custom formatted output in both Kotlin and Java:

KotlinJava

```
// A tool that edits file
object EditFile : Tool<EditFile.Args, EditFile.Result>(
    argsType = typeToken<Args>(),
    resultType = typeToken<Result>(),
    name = "edit_file",
    description = "Edits the given file"
) {
    // Define tool arguments
    @Serializable
    public data class Args(
        val path: String,
        val original: String,
        val replacement: String
    )

    @Serializable
    public data class Result(
        private val patchApplyResult: PatchApplyResult
    ) {

        @Serializable
        public sealed interface PatchApplyResult {
            @Serializable
            public data class Success(val updatedContent: String) : PatchApplyResult

            @Serializable
            public sealed class Failure(public val reason: String) : PatchApplyResult
        }

        // Textual output (in Markdown format) that will be visible to the LLM after the tool finishes.
        fun textForLLM(): String = markdown {
            if (patchApplyResult is PatchApplyResult.Success) {
                line {
                    bold("Successfully").text(" edited file (patch applied)")
                }
            } else {
                line {
                    text("File was ")
                        .bold("not")
                        .text(" modified (patch application failed: ${(patchApplyResult as PatchApplyResult.Failure).reason})")
                }
            }
        }

        override fun toString(): String = textForLLM()
    }

    // Function that executes the tool with the provided arguments
    override suspend fun execute(args: Args): Result {
        return TODO("Implement file edit")
    }
}
```

```
import ai.koog.agents.core.tools.ToolRegistry;
import ai.koog.agents.core.tools.annotations.LLMDescription;
import ai.koog.agents.core.tools.annotations.Tool;

// Java equivalent: return Markdown text directly to the LLM from a Java method and register it as a tool.
// This avoids needing a custom serializable Result type (which would require Kotlin serialization support).
public final class EditFile {
    private EditFile() {}

    @Tool(customName = "edit_file")
    @LLMDescription(description = "Edits the given file")
    public static String editFile(
            String path,
            String original,
            String replacement
    ) {
        // TODO: Implement file edit logic; below is a placeholder illustrating Markdown output
        boolean success = false;
        if (success) {
            return "**Successfully** edited file (patch applied)";
        } else {
            return "File was **not** modified (patch application failed: reason)";
        }
    }

    public static ToolRegistry registry() throws NoSuchMethodException {
        return ToolRegistry.builder()
            .tool(EditFile.class.getMethod("editFile", String.class, String.class, String.class))
            .build();
    }
}
// Note: If you need a structured custom Result object from Java, you must expose a Kotlin @Serializable type
// or another serializer-aware type. Returning String works out-of-the-box with Koog's Java interop.
```

After implementing your tool in Kotlin or Java, you need to add it to a tool registry and then use it with an agent.
For details, see [Tool registry](../tools/index#tool-registry).
