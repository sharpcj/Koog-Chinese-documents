# 多模态内容

多模态内容是指不同类型的内容，例如文本、图像、音频、视频和文件。
Koog 允许你在 `user` 消息中将图像、音频、视频和文件与文本一起发送给 LLM。
你可以使用 Kotlin 中的相应函数或 Java 中的方法，将它们添加到 `user` 消息中：

- `image()`：附加图像（JPG、PNG、WebP、GIF）。
- `audio()`：附加音频文件（MP3、WAV、FLAC）。
- `video()`：附加视频文件（MP4、AVI、MOV）。
- `file()` / `binaryFile()` / `textFile()`：附加文档（PDF、TXT、MD 等）。

每个函数或方法都支持两种配置附件参数的方式，因此你可以：

- 将 URL 或文件路径传递给函数或方法，并由它自动处理附件参数。对于 `file()`、`binaryFile()` 和 `textFile()`，还必须提供 MIME 类型。
- 创建 `ContentPart` 对象并将其传递给函数或方法，以便自定义控制附件参数。

Note

多模态内容支持因 [LLM provider](../../../llm-providers/) 而异。
请查看提供方文档，了解受支持的内容类型。

### 自动配置的附件

如果将 URL 或文件路径传递给附件函数或方法，Koog 会根据文件扩展名自动构造相应的附件参数。

包含文本消息和自动配置附件列表的 `user` 消息的一般格式如下：

KotlinJava

```
user {
    +"Describe these images:"

    image("https://example.com/test.png")
    image(Path("/path/to/image.png"))

    +"Focus on the main subjects."
}
```

```
ContentPartsBuilder partsBuilder = new ContentPartsBuilder();
partsBuilder.text("Describe these images:");
partsBuilder.image("https://example.com/test.png");
partsBuilder.text("Focus on the main subjects.");

Prompt prompt = Prompt.builder("image_analysis")
    .user(partsBuilder.build())
    .build();
```

在 Kotlin 中，`+` 运算符会将文本内容与附件一起添加到用户消息中。在 Java 中，请使用 `ContentPartsBuilder` 的 `text()` 方法。

### 自定义配置的附件

[`ContentPart`](https://api.koog.ai/prompt/prompt-model/ai.koog.prompt.message/-content-part/index.html) 接口允许你分别配置每个附件的参数。

所有附件都实现 `ContentPart.Attachment` 接口。
你可以为每个附件创建某个具体实现的实例、配置其参数，然后将它传递给 Kotlin 中对应的 `image()`、`audio()`、`video()` 或 `file()` 函数，或 Java 中的方法。

包含文本消息和自定义配置附件列表的 `user` 消息的一般格式如下：

KotlinJava

```
user {
    +"Describe this image"
    image(
        AttachmentSource.Image(
            content = AttachmentContent.URL("https://example.com/capture.png"),
            format = "png",
            mimeType = "image/png",
            fileName = "capture.png"
        )
    )
}
```

```
Prompt prompt = Prompt.builder("custom_image")
    .user(List.of(
        new ContentPart.Text("Describe this image"),
        new ContentPart.Image(
            new AttachmentContent.URL("https://example.com/capture.png"),
            "png",
            "image/png",
            "capture.png"
        )
    ))
    .build();
```

Koog 为每种媒体类型提供了以下专用类，它们实现 `ContentPart.Attachment` 接口：

- [`ContentPart.Image`](api:prompt-model::ai.koog.prompt.message.ContentPart.Image)：图像附件，例如 JPG 或 PNG 文件。
- [`ContentPart.Audio`](api:prompt-model::ai.koog.prompt.message.ContentPart.Audio)：音频附件，例如 MP3 或 WAV 文件。
- [`ContentPart.Video`](api:prompt-model::ai.koog.prompt.message.ContentPart.Video)：视频附件，例如 MP4 或 AVI 文件。
- [`ContentPart.File`](api:prompt-model::ai.koog.prompt.message.ContentPart.File)：文件附件，例如 PDF 或 TXT 文件。

所有 `ContentPart.Attachment` 类型都接受以下参数：

| Name | Data type | Required | Description |
| --- | --- | --- | --- |
| `content` | [AttachmentContent](https://api.koog.ai/prompt/prompt-model/ai.koog.prompt.message/-attachment-content/index.html) | Yes | 所提供文件内容的来源。 |
| `format` | String | Yes | 所提供文件的格式。例如 `png`。 |
| `mimeType` | String | Only for `ContentPart.File` | 所提供文件的 MIME Type。对于 `ContentPart.Image`、`ContentPart.Audio` 和 `ContentPart.Video`，默认为 `<type>/<format>`（例如 `image/png`）。对于 `ContentPart.File`，必须显式提供。 |
| `fileName` | String? | No | 所提供文件的名称，包括扩展名。例如 `screenshot.png`。 |

#### 附件内容

AttachmentContent 接口的实现定义作为输入提供给 LLM 的内容类型和来源：

- [`AttachmentContent.URL`](https://api.koog.ai/prompt/prompt-model/ai.koog.prompt.message/-attachment-content/-u-r-l/index.html) 定义所提供内容的 URL：

  ```
  AttachmentContent.URL("https://example.com/image.png")
  ```
- [`AttachmentContent.Binary.Bytes`](https://api.koog.ai/prompt/prompt-model/ai.koog.prompt.message/-attachment-content/-binary/index.html) 将文件内容定义为字节数组：

  ```
  AttachmentContent.Binary.Bytes(byteArrayOf(/* ... */))
  ```
- [`AttachmentContent.Binary.Base64`](https://api.koog.ai/prompt/prompt-model/ai.koog.prompt.message/-attachment-content/-binary/index.html) 将文件内容定义为包含文件数据的 Base64 编码字符串：

  ```
  AttachmentContent.Binary.Base64("iVBORw0KGgoAAAANS...")
  ```
- [`AttachmentContent.PlainText`](https://api.koog.ai/prompt/prompt-model/ai.koog.prompt.message/-attachment-content/-plain-text/index.html) 将文件内容定义为纯文本（仅适用于 [`ContentPart.File`](api:prompt-model::ai.koog.prompt.message.ContentPart.File)）：

  ```
  AttachmentContent.PlainText("This is the file content.")
  ```

### 混合附件

除了在不同提示或消息中提供不同类型的附件之外，也可以在单个 `user()` 消息中提供多个附件和混合类型的附件：

KotlinJava

```
val prompt = prompt("mixed_content") {
    system("You are a helpful assistant.")

    user {
        +"Compare the image with the document content."
        image(Path("/path/to/image.png"))
        binaryFile(Path("/path/to/page.pdf"), "application/pdf")
        +"Structure the result as a table"
    }
}
```

```
Prompt prompt = Prompt.builder("mixed_content_example")
.system("You are a helpful assistant.")
.user(List.of(
    new ContentPart.Text("Please analyze this image and the attached document."),
    new ContentPart.Image(
        new AttachmentContent.URL("https://example.com/image.png"),
        "png",
        "image/png",
        "image.png"
    ),
    new ContentPart.File(
        new AttachmentContent.URL("https://example.com/document.pdf"),
        "pdf",
        "application/pdf",
        "document.pdf"
    ),
    new ContentPart.Text("Summarize the differences.")
))
.build();
```

## 后续步骤

- 如果你使用单个 LLM 提供方，请通过 [LLM clients](../../llm-clients/) 运行提示。
- 如果你使用多个 LLM 提供方，请通过 [prompt executors](../../prompt-executors/) 运行提示。