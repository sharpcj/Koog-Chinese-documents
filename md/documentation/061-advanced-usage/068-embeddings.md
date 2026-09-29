# Embeddings

`embeddings` 模块提供用于生成和比较文本与代码嵌入的功能。嵌入是捕获语义含义的向量表示，可用于高效的相似度比较。

## 概览

该模块包含两个主要组件：

1. **embeddings-base**：嵌入的核心接口和数据结构。
2. **embeddings-llm**：使用 Ollama 生成本地嵌入的实现。

## 入门

以下部分包含如何通过下列方式使用嵌入的基础示例：

- 通过 Ollama 使用本地嵌入模型
- 使用 OpenAI 嵌入模型

### 本地嵌入

要通过本地模型使用嵌入功能，需要在系统中安装并运行 Ollama。
有关安装和运行说明，请参阅 [Ollama 官方 GitHub 仓库](https://github.com/ollama/ollama)。

```
fun main() {
    runBlocking {
        // Create an OllamaClient instance
        val client = OllamaClient()
        // Create an embedder
        val embedder = LLMEmbedder(client, OllamaModels.Embeddings.NOMIC_EMBED_TEXT)
        // Create embeddings
        val embedding = embedder.embed("This is the text to embed")
        // Print embeddings to the output
        println(embedding)
    }
}
```

要使用 Ollama 嵌入模型，请确保满足以下前置条件：

- 已安装并运行 [Ollama](https://ollama.com/download)
- 使用以下命令将嵌入模型下载到本地计算机：

  ```
  ollama pull <ollama-model-id>
  ```

  将 `<ollama-model-id>` 替换为具体模型的 Ollama 标识符。有关可用嵌入模型及其标识符的更多信息，请参阅 [Ollama 模型概览](#ollama-models-overview)。

### Ollama 模型概览

下表概述了可用的 Ollama 嵌入模型。

| Model ID | Ollama ID | Parameters | Dimensions | Context Length | Performance | Tradeoffs |
| --- | --- | --- | --- | --- | --- | --- |
| NOMIC\_EMBED\_TEXT | nomic-embed-text | 137M | 768 | 8192 | 面向语义搜索和文本相似度任务的高质量嵌入 | 在质量和效率之间取得平衡 |
| ALL\_MINILM | all-minilm | 33M | 384 | 512 | 推理速度快，适合通用文本嵌入且质量良好 | 模型尺寸更小、上下文长度较短，但效率很高 |
| MULTILINGUAL\_E5 | zylonai/multilingual-e5-large | 300M | 768 | 512 | 在 100 多种语言上表现强劲 | 模型尺寸更大，但提供出色的多语言能力 |
| BGE\_LARGE | bge-large | 335M | 1024 | 512 | 非常适合英文文本检索和语义搜索 | 模型尺寸更大，但提供高质量嵌入 |
| MXBAI\_EMBED\_LARGE | mxbai-embed-large | - | - | - | 文本数据的高维嵌入 | 专为创建高维嵌入而设计 |

有关这些模型的更多信息，请参阅 Ollama 的 [Embedding Models](https://ollama.com/blog/embedding-models) 博文。

### 选择模型

以下是根据需求选择 Ollama 嵌入模型的一些通用建议：

- 对于通用文本嵌入，请使用 `NOMIC_EMBED_TEXT`。
- 对于多语言支持，请使用 `MULTILINGUAL_E5`。
- 对于最高质量（以性能为代价），请使用 `BGE_LARGE`。
- 对于最高效率（以部分质量为代价），请使用 `ALL_MINILM`。
- 对于高维嵌入，请使用 `MXBAI_EMBED_LARGE`。

## OpenAI 嵌入

要使用 OpenAI 嵌入模型创建嵌入，请使用 `OpenAILLMClient` 实例的 `embed` 方法，如下例所示。

```
suspend fun openAIEmbed(text: String) {
    // Get the OpenAI API token from the OPENAI_KEY environment variable
    val token = System.getenv("OPENAI_KEY") ?: error("Environment variable OPENAI_KEY is not set")
    // Create an OpenAILLMClient instance
    val client = OpenAILLMClient(token)
    // Create an embedder
    val embedder = LLMEmbedder(client, OpenAIModels.Embeddings.TextEmbeddingAda002)
    // Create embeddings
    val embedding = embedder.embed(text)
    // Print embeddings to the output
    println(embedding)
}
```

## AWS Bedrock 嵌入

要使用 AWS Bedrock 嵌入模型创建嵌入，请使用 `BedrockLLMClient` 实例和所选模型的 `embed` 方法。示例：

```
suspend fun bedrockEmbed(text: String) {
    // Get AWS credentials from environment/configuration
    val awsAccessKeyId = System.getenv("AWS_ACCESS_KEY_ID") ?: error("AWS_ACCESS_KEY_ID not set")
    val awsSecretAccessKey = System.getenv("AWS_SECRET_ACCESS_KEY") ?: error("AWS_SECRET_ACCESS_KEY not set")
    // (Optional) AWS_SESSION_TOKEN for temporary credentials
    val awsSessionToken = System.getenv("AWS_SESSION_TOKEN")
    // Create a BedrockLLMClient instance
    val client = BedrockLLMClient(
        identityProvider = StaticCredentialsProvider {
            this.accessKeyId = awsAccessKeyId
            this.secretAccessKey = awsSecretAccessKey
            awsSessionToken?.let { this.sessionToken = it }
        },
        settings = BedrockClientSettings()
    )
    // Create an embedder
    val embedder = LLMEmbedder(client, BedrockModels.Embeddings.AmazonTitanEmbedText)
    // Create embeddings
    val embedding = embedder.embed(text)
    // Print embeddings to the output
    println(embedding)
}
```

### 支持的 AWS Bedrock 嵌入模型

| Provider | Model name | Model ID | Input | Output | Dimensions | Context Length | Notes |
| --- | --- | --- | --- | --- | --- | --- | --- |
| Amazon | Titan Embeddings G1 - Text | `amazon.titan-embed-text-v1` | Text | Embedding | 1,536 | 8192 | 25+ 种语言，针对检索、语义相似度和聚类优化；用于搜索时请切分长文档。 |
| Amazon | Titan Text Embeddings V2 | `amazon.titan-embed-text-v2:0` | Text | Embedding | 1,024 | 8192 | 高精度、灵活维度、多语言（100+）；较小维度可节省存储，输出已归一化。 |
| Cohere | Cohere Embed English v3 | `cohere.embed-english-v3` | Text | Embedding | 1,024 | 8192 | 面向搜索、检索和理解文本细微差别的 SOTA 英文文本嵌入。 |
| Cohere | Cohere Embed Multilingual v3 | `cohere.embed-multilingual-v3` | Text | Embedding | 1,024 | 8192 | 多语言嵌入，在跨语言搜索和语义理解方面达到 SOTA。 |

> 有关最新的模型支持，请参阅 [AWS Bedrock 支持的模型文档](https://docs.aws.amazon.com/bedrock/latest/userguide/models-supported.html)。

## 示例

以下示例展示如何使用嵌入将代码与文本或其他代码片段进行比较。

### 代码到文本比较

将代码片段与自然语言描述进行比较，以查找语义匹配：

```
suspend fun compareCodeToText(embedder: Embedder) { // Embedder type
    // Code snippet
    val code = """
        fun factorial(n: Int): Int {
            return if (n <= 1) 1 else n * factorial(n - 1)
        }
    """.trimIndent()

    // Text descriptions
    val description1 = "A recursive function that calculates the factorial of a number"
    val description2 = "A function that sorts an array of integers"

    // Generate embeddings
    val codeEmbedding = embedder.embed(code)
    val desc1Embedding = embedder.embed(description1)
    val desc2Embedding = embedder.embed(description2)

    // Calculate differences (lower value means more similar)
    val diff1 = embedder.diff(codeEmbedding, desc1Embedding)
    val diff2 = embedder.diff(codeEmbedding, desc2Embedding)

    println("Difference between code and description 1: $diff1")
    println("Difference between code and description 2: $diff2")

    // The code should be more similar to description1 than description2
    if (diff1 < diff2) {
        println("The code is more similar to: '$description1'")
    } else {
        println("The code is more similar to: '$description2'")
    }
}
```

### 代码到代码比较

比较代码片段，以不受语法差异影响地查找语义相似性：

```
suspend fun compareCodeToCode(embedder: Embedder) { // Embedder type
    // Two implementations of the same algorithm in different languages
    val kotlinCode = """
        fun fibonacci(n: Int): Int {
            return if (n <= 1) n else fibonacci(n - 1) + fibonacci(n - 2)
        }
    """.trimIndent()

    val pythonCode = """
        def fibonacci(n):
            if n <= 1:
                return n
            else:
                return fibonacci(n-1) + fibonacci(n-2)
    """.trimIndent()

    val javaCode = """
        public static int bubbleSort(int[] arr) {
            int n = arr.length;
            for (int i = 0; i < n-1; i++) {
                for (int j = 0; j < n-i-1; j++) {
                    if (arr[j] > arr[j+1]) {
                        int temp = arr[j];
                        arr[j] = arr[j+1];
                        arr[j+1] = temp;
                    }
                }
            }
            return arr;
        }
    """.trimIndent()

    // Generate embeddings
    val kotlinEmbedding = embedder.embed(kotlinCode)
    val pythonEmbedding = embedder.embed(pythonCode)
    val javaEmbedding = embedder.embed(javaCode)

    // Calculate differences
    val diffKotlinPython = embedder.diff(kotlinEmbedding, pythonEmbedding)
    val diffKotlinJava = embedder.diff(kotlinEmbedding, javaEmbedding)

    println("Difference between Kotlin and Python implementations: $diffKotlinPython")
    println("Difference between Kotlin and Java implementations: $diffKotlinJava")

    // The Kotlin and Python implementations should be more similar
    if (diffKotlinPython < diffKotlinJava) {
        println("The Kotlin code is more similar to the Python code")
    } else {
        println("The Kotlin code is more similar to the Java code")
    }
}
```

## API 文档

有关嵌入相关的完整 API 参考，请参阅以下模块的参考文档：

- [embeddings-base](https://api.koog.ai/embeddings/embeddings-base/ai.koog.embeddings.base/index.html)：提供用于表示和比较文本与代码嵌入的核心接口和数据结构。
- [embeddings-llm](https://api.koog.ai/embeddings/embeddings-llm/index.html)：包含用于处理本地嵌入模型的实现。