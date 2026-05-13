---
title: Spring AI 多模态接入：图片理解与生成在 Java 中的实现，上传一张图片 AI 告诉你里面有什么
tags: [Spring AI, 多模态, Java, OpenAI, 图片理解, 图片生成, AI]
---

# Spring AI 多模态接入：图片理解与生成在 Java 中的实现

> 上传一张图片，AI 告诉你里面有什么；描述一段场景，AI 帮你画出来 —— Java 开发者也能轻松玩转多模态。

---

## 一、开篇：为什么你需要关注多模态

先来看一个真实场景：

小张是一名电商平台的 Java 开发，Leader 扔过来一个需求：**用户上传商品图片，系统自动识别商品类别、颜色、品牌、适用场景，并生成标准化的商品描述。** 以前这类需求要么靠人工标注（一天审核 5000 张图，团队 10 个人加班到崩溃），要么培训专门的 CV 模型（数据标注+模型训练+部署上线，周期 3 个月起步）。

小张花了一个下午，用 Spring AI 的多模态能力对接 OpenAI 的 GPT-4o，**50 行代码搞定了图片识别 + 描述生成**。演示会上，Leader 沉默了 30 秒，然后问："这玩意儿稳定吗？"

**答案是：非常稳定。** 这就是大模型多模态能力给 Java 开发者带来的革命性变化。

过去我们做图像处理，走的是"专用小模型"路线——目标检测用 YOLO，OCR 用 PaddleOCR，图像分类用 ResNet。每个模型一套 API，调用链路过长，维护成本极高。而现在的多模态大模型（GPT-4o、Claude 3.5、Gemini 2.0）把文本理解、图像理解、图像生成能力大一统到了**一个 API**里，你只需要跟它"聊天"就行。

本文将带你从零开始，用 Spring AI 在 Java 项目中实现**图片理解**和**图片生成**两大核心能力。

---

## 二、什么是多模态？3 分钟快速扫盲

### 2.1 模态（Modality）

简单说，**模态 = 信息的表达形式**。

| 模态类型 | 例子 |
|---------|------|
| 文本 | 聊天消息、文章、代码 |
| 图像 | 照片、截图、设计稿 |
| 音频 | 语音消息、音乐 |
| 视频 | 监控录像、短视频 |

### 2.2 多模态模型 vs 单模态模型

传统 GPT-3.5 是**纯文本模型**，你塞给它一张图片它完全不懂——它只能接收文字的 token。GPT-4o 是**多模态模型**，它不仅理解文字，还能"看"图片、"听"音频，并且可以生成图片。

> 一句话总结：**单模态模型只能处理一种输入，多模态模型能同时理解和生成多种类型的数据。**

### 2.3 Spring AI 的多模态支持

Spring AI 提供了统一的多模态抽象层，屏蔽了底层不同 AI 厂商的 API 差异：

- **图片理解**：通过 `ChatModel` 接口，传入包含图片 URL 或 base64 数据的 `UserMessage`
- **图片生成**：通过 `ImageModel` 接口，传入文本 Prompt，得到生成的图片
- 支持的厂商：OpenAI、Azure OpenAI、Anthropic、Ollama、Stability AI 等

---

## 三、环境准备：5 分钟搭好多模态开发环境

### 3.1 Maven 依赖

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-openai-spring-boot-starter</artifactId>
    <version>1.0.0-M6</version>
</dependency>
```

> 注意：Spring AI 当前最新稳定版是 1.0.0-M6，请根据实际情况选择版本。

### 3.2 配置文件

```yaml
spring:
  ai:
    openai:
      api-key: ${OPENAI_API_KEY}
      base-url: https://api.openai.com
      chat:
        options:
          model: gpt-4o
      image:
        options:
          model: dall-e-3
          size: 1024x1024
          quality: standard
```

配置解释：
- `gpt-4o`：OpenAI 最新的多模态旗舰模型，支持文本+图像输入，成本比 GPT-4 低 50%
- `dall-e-3`：OpenAI 的图像生成模型，生成的图片质量非常出色
- `api-key`：建议通过环境变量注入，不要硬编码

---

## 四、图片理解：上传一张图，AI 告诉你里面有什么

### 4.1 核心原理

Spring AI 的图片理解并不是单独的一个 API，而是**把图片作为 ChatMessage 的一部分**，和文字 Prompt 一起发给大模型。底层原理是：
1. 将图片编码为 base64 字符串或传入图片 URL
2. 构造一个 `UserMessage`，内容包含文字描述 + 图片引用
3. 调用 `ChatClient`，大模型同时处理文字和图片，返回文字结果

### 4.2 基础示例：识别图片内容

```java
@RestController
@RequestMapping("/api/multimodal")
public class ImageUnderstandingController {

    private final ChatClient chatClient;

    public ImageUnderstandingController(ChatClient.Builder chatClientBuilder) {
        this.chatClient = chatClientBuilder.build();
    }

    /**
     * 通过图片 URL 进行识别
     */
    @PostMapping("/analyze-by-url")
    public ResponseEntity<Map<String, String>> analyzeByUrl(@RequestBody AnalyzeRequest request) {
        String prompt = """
            请仔细分析这张图片，返回以下信息的 JSON 格式：
            1. 图片中有什么物体或场景
            2. 主要的颜色
            3. 图片传达的情绪或氛围
            4. 适用的场景或用途
            """;

        var userMessage = new UserMessage(prompt,
                List.of(new Media(MimeTypeUtils.IMAGE_PNG,
                        request.getImageUrl())));

        String response = chatClient.prompt()
                .messages(userMessage)
                .call()
                .content();

        return ResponseEntity.ok(Map.of("result", response));
    }

    @Data
    public static class AnalyzeRequest {
        private String imageUrl;
    }
}
```

### 4.3 进阶：图片描述转 JSON 结构化输出

实际业务中，AI 返回的纯文本往往不够用。我们需要把它**格式化为结构化 JSON**，方便后续业务处理。Spring AI 提供了 `BeanOutputConverter` 完美解决这个问题：

```java
import org.springframework.ai.converter.BeanOutputConverter;
import org.springframework.ai.converter.StructuredOutputConverter;

// 定义目标数据结构
@JsonClassDescription("图片分析结果")
public record ImageAnalysisResult(
    @JsonProperty(required = true) @JsonPropertyDescription("图片中的主要物体列表")
    List<String> objects,

    @JsonProperty(required = true) @JsonPropertyDescription("图片的主色调")
    String dominantColor,

    @JsonProperty(required = true) @JsonPropertyDescription("图片的风格（写实/插画/动漫等）")
    String style,

    @JsonProperty(required = true) @JsonPropertyDescription("50字以内的图片描述")
    String description,

    @JsonProperty(required = true) @JsonPropertyDescription("商品分类（如服装/电子产品/食品/其他）")
    String category
) {}

@PostMapping("/analyze-structured")
public ImageAnalysisResult analyzeStructured(@RequestParam("file") MultipartFile file) throws IOException {
    // 将上传文件转为 base64
    byte[] bytes = file.getBytes();
    String base64Image = Base64.getEncoder().encodeToString(bytes);

    // 使用 BeanOutputConverter 约束输出格式
    var converter = new BeanOutputConverter<>(ImageAnalysisResult.class);
    String jsonSchema = converter.getJsonSchema();

    var userMessage = new UserMessage("""
        请分析这张图片，严格按照以下 JSON Schema 返回结果，不要添加其他内容。
    
        JSON Schema: %s
        """.formatted(jsonSchema),
        List.of(new Media(MimeTypeUtils.IMAGE_JPEG, base64Image)));

    String response = chatClient.prompt()
            .messages(userMessage)
            .call()
            .content();

    return converter.convert(response);
}
```

这样一来，你可以直接把 `ImageAnalysisResult` 塞进数据库、传给前端、或者触发后续业务流程，全程类型安全。

### 4.4 电商场景实战：商品图片智能识别 Service

```java
@Service
public class ProductImageAnalysisService {

    private final ChatClient chatClient;

    public ProductImageAnalysisService(ChatClient.Builder builder) {
        this.chatClient = builder.build();
    }

    public ProductInfo analyzeProductImage(String imageUrl) {
        var converter = new BeanOutputConverter<>(ProductInfo.class);

        var userMessage = new UserMessage("""
            你是一个专业的电商商品分析助手。请分析这张商品图片，提取以下信息：
            - 商品名称
            - 品牌（如果可见）
            - 颜色
            - 材质
            - 适用人群（男/女/儿童/通用）
            - 适用场景
            - 生成一段 100 字内的 SEO 友好商品描述
            - 3-5 个标签关键词
    
            请严格按照以下 JSON Schema 返回：
            %s
            """.formatted(converter.getJsonSchema()),
            List.of(new Media(MimeTypeUtils.IMAGE_PNG, imageUrl)));

        String result = chatClient.prompt()
                .messages(userMessage)
                .call()
                .content();

        return converter.convert(result);
    }
}
```

---

## 五、图片生成：说一句话，AI 帮你画出来

### 5.1 Spring AI ImageModel 接口

Spring AI 提供了 `ImageModel` 接口来统一图片生成能力：

```java
public interface ImageModel extends Model<ImagePrompt, ImageResponse> {
    ImageResponse call(ImagePrompt request);
}
```

使用 `ImagePrompt` 构造生成参数（Prompt 文本、尺寸、风格等），返回 `ImageResponse`（包含图片 URL 或 base64 数据）。

### 5.2 基础示例：文字生成图片

```java
@RestController
@RequestMapping("/api/multimodal")
public class ImageGenerationController {

    private final ImageModel imageModel;

    public ImageGenerationController(ImageModel imageModel) {
        this.imageModel = imageModel;
    }

    @PostMapping("/generate-image")
    public ResponseEntity<Map<String, Object>> generateImage(@RequestBody ImageGenRequest request) {
        var imageOptions = OpenAiImageOptions.builder()
                .withModel("dall-e-3")
                .withQuality("hd")           // 高质量
                .withN(1)                    // 生成 1 张
                .withHeight(1024)
                .withWidth(1024)
                .withResponseFormat("url")   // 返回 URL 而非 base64
                .build();

        var imagePrompt = new ImagePrompt(request.getPrompt(), imageOptions);
        ImageResponse response = imageModel.call(imagePrompt);

        String imageUrl = response.getResult().getOutput().getUrl();

        return ResponseEntity.ok(Map.of(
                "url", imageUrl,
                "prompt", request.getPrompt()
        ));
    }

    @Data
    public static class ImageGenRequest {
        private String prompt;
    }
}
```

### 5.3 进阶：用 AI 生成商品展示图

电商场景中，经常需要为商品生成不同场景的展示图。下面是一个完整的示例：

```java
@Service
public class ProductImageGenerationService {

    private final ImageModel imageModel;

    public ProductImageGenerationService(ImageModel imageModel) {
        this.imageModel = imageModel;
    }

    /**
     * 为商品生成不同场景的展示图
     */
    public Map<String, String> generateProductScenes(String productName, String productDescription) {
        Map<String, String> result = new LinkedHashMap<>();

        var scenes = Map.of(
            "白底图", "白色背景，%s 居中摆放，产品摄影，商业摄影，高清，8K",
            "场景图", "%s 摆放在现代简约风格的客厅中，自然光，温馨氛围，高清",
            "使用图", "一位年轻女性正在使用%s，微笑表情，自然光线，生活方式摄影"
        );

        for (var entry : scenes.entrySet()) {
            String prompt = entry.getValue().formatted(productName);

            var imageOptions = OpenAiImageOptions.builder()
                    .withModel("dall-e-3")
                    .withQuality("standard")
                    .withN(1)
                    .withHeight(1024)
                    .withWidth(1024)
                    .build();

            ImageResponse response = imageModel.call(new ImagePrompt(prompt, imageOptions));
            result.put(entry.getKey(), response.getResult().getOutput().getUrl());
        }

        return result;
    }
}
```

---

## 六、整合 Controller：一个完整的 REST API 示例

下面是一个**生产级别的统一多模态 Controller**，整合了图片理解 + 图片生成两大能力：

```java
@RestController
@RequestMapping("/api/v1/multimodal")
@Slf4j
public class MultimodalController {

    private final ChatClient chatClient;
    private final ImageModel imageModel;

    public MultimodalController(ChatClient.Builder chatClientBuilder, ImageModel imageModel) {
        this.chatClient = chatClientBuilder.build();
        this.imageModel = imageModel;
    }

    /**
     * 图片理解 —— 通过 URL
     */
    @PostMapping("/understanding/url")
    public ResponseEntity<Map<String, String>> understandByUrl(@RequestBody @Valid ImageUrlRequest request) {
        log.info("分析图片 URL: {}", request.getImageUrl());

        var userMessage = new UserMessage(request.getPrompt(),
                List.of(new Media(MimeTypeUtils.IMAGE_PNG, request.getImageUrl())));

        String result = chatClient.prompt()
                .messages(userMessage)
                .call()
                .content();

        return ResponseEntity.ok(Map.of("result", result));
    }

    /**
     * 图片理解 —— 上传本地文件
     */
    @PostMapping("/understanding/upload")
    public ResponseEntity<Map<String, String>> understandByUpload(
            @RequestParam("file") MultipartFile file,
            @RequestParam(defaultValue = "请描述这张图片的内容") String prompt) throws IOException {

        String base64 = Base64.getEncoder().encodeToString(file.getBytes());
        var userMessage = new UserMessage(prompt,
                List.of(new Media(MimeTypeUtils.IMAGE_JPEG, base64)));

        String result = chatClient.prompt()
                .messages(userMessage)
                .call()
                .content();

        return ResponseEntity.ok(Map.of("result", result));
    }

    /**
     * 图片生成
     */
    @PostMapping("/generation")
    public ResponseEntity<Map<String, Object>> generate(@RequestBody @Valid ImageGenRequest request) {
        var options = OpenAiImageOptions.builder()
                .withModel("dall-e-3")
                .withQuality(request.getQuality())
                .withN(1)
                .withHeight(request.getHeight())
                .withWidth(request.getWidth())
                .build();

        ImageResponse response = imageModel.call(new ImagePrompt(request.getPrompt(), options));
        String url = response.getResult().getOutput().getUrl();

        log.info("图片生成完成: {}", url);
        return ResponseEntity.ok(Map.of("url", url, "prompt", request.getPrompt()));
    }

    // --- DTOs ---

    @Data
    public static class ImageUrlRequest {
        @NotBlank private String imageUrl;
        @NotBlank private String prompt;
    }

    @Data
    public static class ImageGenRequest {
        @NotBlank private String prompt;
        private String quality = "standard";
        private int width = 1024;
        private int height = 1024;
    }
}
```

---

## 七、多模态对接国产模型

除了 OpenAI，Spring AI 也支持对接国产多模态模型。以通义千问（DashScope）为例：

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-dashscope-spring-boot-starter</artifactId>
    <version>1.0.0-M6</version>
</dependency>
```

```yaml
spring:
  ai:
    dashscope:
      api-key: ${DASHSCOPE_API_KEY}
      chat:
        options:
          model: qwen-vl-max
```

```java
// 代码完全一致，只是底层模型换成了通义千问的多模态版本
var userMessage = new UserMessage("描述这张图片",
        List.of(new Media(MimeTypeUtils.IMAGE_PNG, imageUrl)));

String result = chatClient.prompt()
        .messages(userMessage)
        .call()
        .content();
```

这就是 Spring AI 最大的价值——**一次编写，切换模型零代码改动**。

---

## 八、性能优化与最佳实践

### 8.1 图片压缩

多模态大模型的 Token 消耗与图片大小成正比。一张 10MB 的原图直接上传会消耗大量 Token。建议在上传前做压缩处理：

```java
public String compressAndEncode(MultipartFile file, int maxWidth, int maxHeight) throws IOException {
    BufferedImage original = ImageIO.read(file.getInputStream());
    int width = Math.min(original.getWidth(), maxWidth);
    int height = Math.min(original.getHeight(), maxHeight);

    BufferedImage resized = new BufferedImage(width, height, BufferedImage.TYPE_INT_RGB);
    Graphics2D g = resized.createGraphics();
    g.drawImage(original.getScaledInstance(width, height, Image.SCALE_SMOOTH), 0, 0, null);
    g.dispose();

    ByteArrayOutputStream bos = new ByteArrayOutputStream();
    ImageIO.write(resized, "jpg", bos);
    return Base64.getEncoder().encodeToString(bos.toByteArray());
}
```

### 8.2 缓存策略

相同图片的识别结果建议缓存，避免重复调用：

```java
@Cacheable(value = "image-analysis", key = "#imageHash")
public ImageAnalysisResult analyzeWithCache(String imageUrl, String imageHash) {
    // ... 调用 AI
}
```

### 8.3 异步处理

图片生成通常需要几秒到十几秒，建议使用异步方式避免阻塞 HTTP 线程：

```java
@Async
public CompletableFuture<String> generateAsync(String prompt) {
    ImageResponse response = imageModel.call(new ImagePrompt(prompt, options));
    return CompletableFuture.completedFuture(response.getResult().getOutput().getUrl());
}
```

---

## 九、常见问题 FAQ

**Q1: 多模态模型能识别图片中的文字吗？**
A: 能。GPT-4o、Claude 3.5 的多模态能力已经内置了 OCR 功能，你不需要额外对接 OCR 服务。直接问"提取图片中的文字"就行。

**Q2: 能否识别视频？**
A: Spring AI 当前版本主要支持图片（单帧）输入。视频处理建议先将视频抽帧，然后逐帧传给多模态模型分析。

**Q3: 图片生成怎么控制风格？**
A: 在 Prompt 中明确描述即可，例如"宫崎骏动漫风格""水彩画风""极简扁平化设计"。DALL-E 3 对风格控制的理解非常精准。

**Q4: 国内的模型能用吗？**
A: 完全能。Spring AI 已经适配了通义千问、百度文心一言、智谱 GLM-4V 等多模态模型，接入方式与 OpenAI 几乎一致。

---

## 十、总结

今天我们完整覆盖了 Spring AI 多模态接入的方方面面：

1. **多模态概念**：一张图讲清楚单模态 vs 多模态的本质区别
2. **图片理解**：从基础 URL 识别到结构化 JSON 输出，再到电商场景实战
3. **图片生成**：从单图生成到批量场景图生成
4. **生产级 Controller**：可直接复用的完整 REST API
5. **国产模型适配**：一行代码替换，从 OpenAI 切换到通义千问
6. **性能优化**：图片压缩、缓存、异步处理三件套

核心思想就一句话：**大模型时代，Java 开发者不需要再分散精力学 CV/NLP 专用模型，一个 ChatClient 搞定全部多模态需求。**

---

**下一篇预告**：《Spring AI + Spring Cloud：构建 AI 微服务集群的最佳实践》—— 当 AI 能力从单体走向分布式，如何用 Spring Cloud 全家桶构建高可用、可扩展的 AI 微服务体系？敬请期待！

---

> 作者：IT 老熊
> 标签：Spring AI, 多模态, Java, 图片理解, 图片生成
> 原文首发：CSDN 技术社区
