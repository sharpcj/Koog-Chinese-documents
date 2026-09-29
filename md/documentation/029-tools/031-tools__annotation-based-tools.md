# 基于注解的工具

基于注解的工具为 Kotlin 和 Java 提供了一种声明式方式，可将函数和方法作为工具暴露给大型语言模型（LLM）。
通过使用注解，你可以将任意函数或方法转换为 LLM 能理解和使用的工具。

当你需要在 Kotlin 或 Java 中将现有功能暴露给 LLM，而不想手动实现工具描述时，这种方式很有用。

Note

基于注解的工具仅支持 JVM，不适用于其他平台。如需多平台支持，请使用[基于类的工具 API](../class-based-tools/)。

## 关键注解

要开始在项目中使用基于注解的工具，需要了解以下关键注解：

| Annotation | Description |
| --- | --- |
| `@Tool` | 标记应暴露给 LLM 的函数。 |
| `@LLMDescription` | 为工具及其组件提供描述性信息。 |

## @Tool 注解

`@Tool` 注解用于标记应暴露给 LLM 的函数（Kotlin）或方法（Java）。
使用 `@Tool` 注解的函数和方法会通过反射从实现 `ToolSet` 接口的对象中收集。详情请参阅[实现 ToolSet 接口](#1-implement-the-toolset-interface)。

### 定义

```
@Target(AnnotationTarget.FUNCTION)
public annotation class Tool(val customName: String = "")
```

### 参数

| Name | Required | Description |
| --- | --- | --- |
| `customName` | No | 指定工具的自定义名称。如果未提供，则使用函数名。 |

### 用法

要将函数或方法标记为工具，请在实现 `ToolSet` 接口的类中，将 `@Tool` 注解应用到该函数或方法：

KotlinJava

```
class MyToolSet : ToolSet {
    @Tool
    fun myTool(): String {
        // Tool implementation
        return "Result"
    }

    @Tool(customName = "customToolName")
    fun anotherTool(): String {
        // Tool implementation
        return "Result"
    }
}
```

```
public class MyToolSet implements ToolSet {
    @Tool
    public String myTool() {
        // Tool implementation
        return "Result";
    }

    @Tool(customName = "customToolName")
    public String anotherTool() {
        // Tool implementation
        return "Result";
    }
}
```

## @LLMDescription 注解

`@LLMDescription` 注解为代码元素（类、函数、方法、参数等）向 LLM 提供描述性信息。
这有助于 LLM 理解这些元素的用途和用法。

### 定义

```
@Target(
    AnnotationTarget.PROPERTY,
    AnnotationTarget.CLASS,
    AnnotationTarget.TYPE,
    AnnotationTarget.VALUE_PARAMETER,
    AnnotationTarget.FUNCTION
)
public annotation class LLMDescription(val description: String)
```

### 参数

| Name | Required | Description |
| --- | --- | --- |
| `description` | Yes | 描述被注解元素的字符串。 |

### 用法

`@LLMDescription` 注解可以应用于多个层级。例如：

- 函数级别：

KotlinJava

```
@Tool
@LLMDescription("Performs a specific operation and returns the result")
fun myTool(): String {
    // Function implementation
    return "Result"
}
```

```
@Tool
@LLMDescription(description = "Performs a specific operation and returns the result")
public String myTool() {
    // Function implementation
    return "Result";
}
```

- 参数级别：

KotlinJava

```
@Tool
@LLMDescription("Processes input data")
fun processTool(
    @LLMDescription("The input data to process")
    input: String,

    @LLMDescription("Optional configuration parameters")
    config: String = ""
): String {
    // Function implementation
    return "Processed: $input with config: $config"
}
```

```
@Tool
@LLMDescription(description = "Processes input data")
public String processTool(
        @LLMDescription(description = "The input data to process") String input,
        @LLMDescription(description = "Optional configuration parameters") String config
) {
    // Function implementation
    return "Processed: " + input + " with config: " + config;
}
```

## 创建工具

### 1. 实现 ToolSet 接口

创建一个实现 [`ToolSet`](https://api.koog.ai/agents/agents-tools/ai.koog.agents.core.tools.reflect/-tool-set/index.html) 接口的类。
此接口会将你的类标记为工具容器。

KotlinJava

```
class MyFirstToolSet : ToolSet {
    // Tools will go here
}
```

```
public class MyFirstToolSet implements ToolSet {
    // Tools will go here
}
```

### 2. 添加工具函数

向类中添加函数或方法，并使用 `@Tool` 注解它们，以将其暴露为工具：

KotlinJava

```
class MyFirstToolSet : ToolSet {
    @Tool
    fun getWeather(location: String): String {
        // In a real implementation, you would call a weather API
        return "The weather in $location is sunny and 72°F"
    }
}
```

```
public class MyFirstToolSet implements ToolSet {
    @Tool
    public String getWeather(String location) {
        // In a real implementation, you would call a weather API
        return "The weather in " + location + " is sunny and 72°F";
    }
}
```

### 3. 添加描述

添加 `@LLMDescription` 注解，为 LLM 提供上下文：

KotlinJava

```
@LLMDescription("Tools for getting weather information")
class MyFirstToolSet : ToolSet {
    @Tool
    @LLMDescription("Get the current weather for a location")
    fun getWeather(
        @LLMDescription("The city and state/country")
        location: String
    ): String {
        // In a real implementation, you would call a weather API
        return "The weather in $location is sunny and 72°F"
    }
}
```

```
@LLMDescription(description = "Tools for getting weather information")
public class MyFirstToolSet implements ToolSet {
    @Tool
    @LLMDescription(description = "Get the current weather for a location")
    public String getWeather(
            @LLMDescription(description = "The city and state/country") String location
    ) {
        // In a real implementation, you would call a weather API
        return "The weather in " + location + " is sunny and 72°F";
    }
}
```

### 4. 在智能体中使用你的工具

现在可以在智能体中使用你的工具：

KotlinJava

```
fun main() {
    runBlocking {
        // Create your tool set
        val weatherTools = MyFirstToolSet()

        // Create an agent with your tools

        val agent = AIAgent(
            promptExecutor = simpleOpenAIExecutor(apiToken),
            systemPrompt = "Provide weather information for a given location.",
            llmModel = OpenAIModels.Chat.GPT4o,
            toolRegistry = ToolRegistry {
                tools(weatherTools)
            }
        )

        // The agent can now use your weather tools
        agent.run("What's the weather like in New York?")
    }
}
```

```
String apiToken = System.getenv("OPENAI_API_KEY");

// Create your tool set
 MyFirstToolSet weatherTools = new MyFirstToolSet();

ToolRegistry toolRegistry = ToolRegistry.builder()
    .tools(weatherTools)
    .build();

// Create an agent with your tools
AIAgent<String, String> agent = AIAgent.builder()
    .promptExecutor(simpleOpenAIExecutor(System.getenv("OPENAI_API_KEY")))
    .systemPrompt("Provide weather information for a given location.")
    .llmModel(OpenAIModels.Chat.GPT4o)
    .toolRegistry(toolRegistry)
    .build();

// The agent can now use your weather tools
String result = agent.run("What's the weather like in New York?");
System.out.println(result);
```

## 用法示例

下面是一些工具注解的实际示例。

### 基本示例：开关控制器

此示例展示一个用于控制开关的简单工具集：

KotlinJava

```
@LLMDescription("Tools for controlling a switch")
class SwitchTools(val switch: Switch) : ToolSet {
    @Tool
    @LLMDescription("Switches the state of the switch")
    fun switch(
        @LLMDescription("The state to set (true for on, false for off)")
        state: Boolean
    ): String {
        switch.switch(state)
        return "Switched to ${if (state) "on" else "off"}"
    }

    @Tool
    @LLMDescription("Returns the current state of the switch")
    fun switchState(): String {
        return "Switch is ${if (switch.isOn()) "on" else "off"}"
    }
}
```

```
public class Switch {
    private boolean state;

    public Switch(boolean state) {
        this.state = state;
    }

    // "switch" is a reserved keyword in Java, so we use a different method name
    public void setState(boolean state) {
        this.state = state;
    }

    public boolean isOn() {
        return state;
    }
}

@LLMDescription(description = "Tools for controlling a switch")
public class SwitchTools implements ToolSet {
    private final Switch sw;

    public SwitchTools(Switch sw) {
        this.sw = sw;
    }

    @Tool
    @LLMDescription(description = "Switches the state of the switch")
    public String switchStateTo(
            @LLMDescription(description = "The state to set (true for on, false for off)") boolean state
    ) {
        sw.setState(state);
        return "Switched to " + (state ? "on" : "off");
    }

    @Tool
    @LLMDescription(description = "Returns the current state of the switch")
    public String switchState() {
        return "Switch is " + (sw.isOn() ? "on" : "off");
    }
}
```

当 LLM 需要控制开关时，它可以从提供的描述中理解以下信息：

- 工具的目的和功能。
- 使用工具所需的参数。
- 每个参数可接受的值。
- 执行后的预期返回值。

### 高级示例：诊断工具

此示例展示一个用于设备诊断的更复杂工具集：

KotlinJava

```
@LLMDescription("Tools for performing diagnostics and troubleshooting on devices")
class DiagnosticToolSet : ToolSet {
    @Tool
    @LLMDescription("Run diagnostic on a device to check its status and identify any issues")
    fun runDiagnostic(
        @LLMDescription("The ID of the device to diagnose")
        deviceId: String,

        @LLMDescription("Additional information for the diagnostic (optional)")
        additionalInfo: String = ""
    ): String {
        // Implementation
        return "Diagnostic results for device $deviceId"
    }

    @Tool
    @LLMDescription("Analyze an error code to determine its meaning and possible solutions")
    fun analyzeError(
        @LLMDescription("The error code to analyze (e.g., 'E1001')")
        errorCode: String
    ): String {
        // Implementation
        return "Analysis of error code $errorCode"
    }
}
```

```
@LLMDescription(description = "Tools for performing diagnostics and troubleshooting on devices")
public class DiagnosticToolSet implements ToolSet {
    // Convenience overload (not exposed as a tool)
    public String runDiagnostic(String deviceId) {
        return runDiagnostic(deviceId, "");
    }

    @Tool
    @LLMDescription(description = "Run diagnostic on a device to check its status and identify any issues")
    public String runDiagnostic(
            @LLMDescription(description = "The ID of the device to diagnose") String deviceId,
            @LLMDescription(description = "Additional information for the diagnostic (optional)") String additionalInfo
    ) {
        // Implementation
        return "Diagnostic results for device " + deviceId;
    }

    @Tool
    @LLMDescription(description = "Analyze an error code to determine its meaning and possible solutions")
    public String analyzeError(
            @LLMDescription(description = "The error code to analyze (e.g., 'E1001')") String errorCode
    ) {
        // Implementation
        return "Analysis of error code " + errorCode;
    }
}
```

## 最佳实践

- **提供清晰描述**：编写清晰、简洁的描述，说明工具、参数和返回值的目的与行为。
- **描述所有参数**：为所有参数添加 `@LLMDescription`，帮助 LLM 理解每个参数的用途。
- **使用一致命名**：对工具和参数使用一致的命名约定，使其更直观。
- **分组相关工具**：在同一个 `ToolSet` 实现中分组相关工具，并提供类级别描述。
- **返回有信息量的结果**：确保工具返回值能清楚说明操作结果。
- **妥善处理错误**：在工具中包含错误处理，并返回有信息量的错误消息。
- **记录默认值**：当参数有默认值（Kotlin）或重载（Java）时，在描述中说明这一点。
- **保持工具聚焦**：每个工具都应执行一个具体且定义明确的任务，而不是试图做太多事情。

## 常见问题排查

使用工具注解时，你可能会遇到一些常见问题。

### 工具未被识别

如果智能体无法识别你的工具，请检查以下内容：

- 你的类实现了 `ToolSet` 接口。
- 所有工具函数或方法都使用了 `@Tool` 注解。
- 工具函数或方法具有合适的返回类型（为简单起见，建议使用 `String`）。
- 你的工具已正确注册到智能体中。

### 工具描述不清晰

如果 LLM 没有正确使用你的工具，或误解了它们的用途，请尝试以下做法：

- 尽可能使用基本参数类型（Kotlin 中的 `String`、`Boolean`、`Int`，或 Java 中的 `String`、`boolean`、`int`）。
- 在参数描述中清楚说明预期格式。
- 对于复杂类型，考虑使用带特定格式的 `String` 参数，并在工具中解析它们。
- 在参数描述中包含有效输入示例。
- 注意 Java 不支持默认参数。请改用方法重载。

### 参数类型问题

如果 LLM 提供了错误的参数类型，请尝试以下做法：

- 尽可能使用简单参数类型（`String`、`Boolean`、`Int`）。
- 在参数描述中清楚说明预期格式。
- 对于复杂类型，考虑使用带特定格式的 `String` 参数，并在工具中解析它们。
- 在参数描述中包含有效输入示例。

### 性能问题

如果你的工具导致性能问题，请尝试以下做法：

- 保持工具实现轻量。
- 对于资源密集型操作，考虑实现异步处理。
- 在适当时缓存结果。
- 记录工具使用情况，以识别瓶颈。