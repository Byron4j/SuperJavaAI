# Spring AI 输出解析：BeanOutputParser 自动映射为 Java POJO，AI的文本回答秒变结构化数据

> AI返回了一段文本，你要手动解析JSON？太土了——Spring AI的OutputParser直接帮你映射成POJO。

---

## 一、没有OutputParser 的日子

先感受一下"手动解析"的痛苦。假设你要让AI分析一段用户评论的情感：

```java
// 原始做法：让AI返回JSON，然后手工解析
String prompt = "分析以下评论的情感，返回JSON: {sentiment: 'positive/negative/neutral', score: 0-10}";
String aiResponse = chatClient.call(prompt);

// AI返回的内容可能是：
// "好的，分析结果如下：{"sentiment": "positive", "score": 8}"
// 或者
// "根据分析，情感为正面\n{\"sentiment\": \"positive\", \"score\": 8}"
// 甚至
// "```json\n{\"sentiment\": \"positive\", \"score\": 8}\n```"

// 你需要写一堆代码来处理各种格式：
String json = extractJsonFromResponse(aiResponse);  // 自己写正则
SentimentResult result = objectMapper.readValue(json, SentimentResult.class);
```

三个问题：
1. **格式不可控**：AI可能返回纯JSON、markdown包裹的JSON、带说明文字的JSON
2. **解析代码脆弱**：正则提取JSON容易漏掉边界情况
3. **维护成本高**：每加一个AI返回类型就要写一套提取+解析逻辑

**Spring AI 的 OutputParser 直接解决了这些问题。**

---

## 二、BeanOutputParser——让AI乖乖返回你的POJO

### 2.1 定义目标POJO

```java
public record SentimentResult(
    @JsonProperty("sentiment") String sentiment,
    @JsonProperty("score") int score,
    @JsonProperty("keywords") List<String> keywords,
    @JsonProperty("summary") String summary
) {}
```

### 2.2 使用 BeanOutputParser

```java
@Service
public class SentimentAnalysisService {

    private final ChatClient chatClient;

    public SentimentAnalysisService(ChatClient.Builder builder) {
        this.chatClient = builder.build();
    }

    public SentimentResult analyze(String review) {
        // 创建OutputParser，指定目标类型
        BeanOutputParser<SentimentResult> parser = 
            BeanOutputParser.newInstance(SentimentResult.class);

        String userMessage = "分析以下用户评论的情感：\n" + review;

        String aiContent = chatClient.prompt()
            .user(userMessage)
            .call()
            .content();

        // 一键解析！
        SentimentResult result = parser.parse(aiContent);
        return result;
    }
}
```

就这么简单——**指定目标类 → 调用AI → 解析 → 拿到POJO对象**。

### 2.3 底层原理

BeanOutputParser 做了什么？

1. **自动生成JSON Schema**：根据你POJO的字段类型自动生成JSON Schema
2. **嵌入到Prompt**：在system message中追加输出格式约束，强制AI按Schema输出
3. **JSON提取**：从AI的回复中智能提取JSON（自动处理markdown代码块包裹）
4. **反序列化**：Jackson将JSON反序列化为POJO

实际上发送给AI的Prompt大致长这样：

```
## 输出格式要求
请严格按照以下JSON Schema格式输出，不要输出任何其他内容（不要包含markdown代码块标记）：

{
  "sentiment": "positive/negative/neutral",
  "score": 0-10,
  "keywords": ["关键词列表"],
  "summary": "一句话总结"
}
```

---

## 三、实战：AI分析用户评论的完整示例

### 3.1 定义复杂POJO

```java
package com.example.ai.model;

import com.fasterxml.jackson.annotation.JsonProperty;
import java.util.List;

public record ProductReviewAnalysis(
    @JsonProperty("overall_sentiment") String overallSentiment,
    @JsonProperty("sentiment_score") double sentimentScore,
    @JsonProperty("key_highlights") List<String> keyHighlights,
    @JsonProperty("pain_points") List<String> painPoints,
    @JsonProperty("product_quality") QualityAssessment productQuality,
    @JsonProperty("delivery_experience") QualityAssessment deliveryExperience,
    @JsonProperty("improvement_suggestions") List<String> improvementSuggestions,
    @JsonProperty("reply_draft") String replyDraft
) {
    public record QualityAssessment(
        @JsonProperty("rating") int rating,
        @JsonProperty("comment") String comment
    ) {}
}
```

### 3.2 Service 实现

```java
@RestController
@RequestMapping("/api/review")
public class ReviewAnalysisController {

    private final ChatClient chatClient;

    public ReviewAnalysisController(ChatClient.Builder builder) {
        this.chatClient = builder.build();
    }

    @PostMapping("/analyze")
    public ProductReviewAnalysis analyze(@RequestBody String review) {
        BeanOutputParser<ProductReviewAnalysis> parser = 
            BeanOutputParser.newInstance(ProductReviewAnalysis.class);

        String systemPrompt = """
            你是一个电商评论分析专家。
            请仔细分析用户的产品评论，从以下维度进行剖析：
            1. 整体情感倾向（正面/负面/中性）
            2. 用户提到的亮点
            3. 用户反映的痛点
            4. 产品质量评估
            5. 物流体验评估
            6. 改进建议
            7. 生成回复草稿（200字以内，针对用户的问题点进行回应）
            """;

        String aiContent = chatClient.prompt()
            .system(systemPrompt)
            .user(review)
            .call()
            .content();

        return parser.parse(aiContent);
    }
}
```

### 3.3 调用效果

```java
// 输入评论
String review = """
    在你们家买了个蓝牙耳机，音质确实不错，低音很震撼。
    但是包装太糟糕了，盒子都压扁了，耳机仓还有划痕。
    客服态度也一般，问了好几次才回复。
    总体来说产品还行，服务需要提升。
    """;

// 解析结果
ProductReviewAnalysis result = controller.analyze(review);
System.out.println(result.overallSentiment());   // "中性"
System.out.println(result.sentimentScore());     // 6.5
System.out.println(result.keyHighlights());      // ["音质好", "低音震撼"]
System.out.println(result.painPoints());         // ["包装破损", "有划痕", "客服响应慢"]
System.out.println(result.productQuality().rating()); // 7
System.out.println(result.deliveryExperience().rating()); // 3
```

---

## 四、ListOutputParser——让AI返回列表

有时候你需要AI返回一个列表，比如"推荐5款产品"。

### 4.1 基础用法

```java
@Service
public class ProductRecommendService {

    private final ChatClient chatClient;

    public List<String> recommendNames(String userPreference) {
        ListOutputParser listParser = new ListOutputParser();

        String aiContent = chatClient.prompt()
            .user("推荐5款适合{0}的产品名称，每行一个，不需要序号".replace(
                "{0}", userPreference))
            .call()
            .content();

        return listParser.parse(aiContent);
    }
}

// 调用
List<String> headphones = service.recommendNames("跑步用的运动耳机");
// 返回: ["韶音OpenRun Pro", "JBL Endurance Peak 3", 
//        "Beats Fit Pro", "Sony WF-1000XM5", "Bose Sport Earbuds"]
```

### 4.2 返回结构化对象列表

```java
public record ProductRecommendation(
    @JsonProperty("name") String name,
    @JsonProperty("price") Double price,
    @JsonProperty("reason") String reason
) {}

@Service
public class StructuredListService {

    private final ChatClient chatClient;

    public List<ProductRecommendation> recommend(String category) {
        BeanOutputParser<List<ProductRecommendation>> parser = 
            BeanOutputParser.newListInstance(ProductRecommendation.class);

        String prompt = String.format(
            "为用户推荐5款%s类产品，每款说明名称、价格和推荐理由", category);

        String aiContent = chatClient.prompt()
            .user(prompt)
            .call()
            .content();

        return parser.parse(aiContent);
    }
}
```

### 4.3 列表解析的容错

```java
public List<String> robustParse(String aiContent) {
    try {
        return new ListOutputParser().parse(aiContent);
    } catch (Exception e) {
        // 尝试按换行分割
        return Arrays.stream(aiContent.split("\n"))
            .map(String::trim)
            .filter(line -> !line.isEmpty())
            .map(line -> line.replaceAll("^\\d+[\\.\\)、]\\s*", ""))
            .collect(Collectors.toList());
    }
}
```

---

## 五、MapOutputParser——让AI返回Key-Value

```java
@Service
public class KeywordExtractService {

    private final ChatClient chatClient;

    public Map<String, String> extractKeywords(String text) {
        MapOutputParser mapParser = new MapOutputParser();

        String prompt = """
            从以下文本中提取关键信息，以Key-Value格式返回：
            - 作者：...
            - 产品类型：...
            - 核心卖点：...
            - 目标人群：...
            
            文本内容：
            %s
            """.formatted(text);

        String aiContent = chatClient.prompt()
            .user(prompt)
            .call()
            .content();

        return mapParser.parse(aiContent);
    }

    // 带泛型的Map解析
    public Map<String, Object> extractWithTypedValues(String text) {
        BeanOutputParser<Map<String, Object>> parser = 
            BeanOutputParser.newMapInstance();

        String aiContent = chatClient.prompt()
            .user("分析这段产品描述，提取规格参数：" + text)
            .call()
            .content();

        return parser.parse(aiContent);
    }
}
```

---

## 六、自定义 OutputParser——处理复杂嵌套结构

当内置Parser不满足需求时，自定义Parser：

### 6.1 定义复杂模型

```java
// 一个复杂的合同分析结果
public record ContractAnalysis(
    String contractType,
    LocalDate effectiveDate,
    LocalDate expirationDate,
    List<Party> parties,
    List<Clause> keyClauses,
    RiskAssessment riskAssessment,
    List<Obligation> obligations
) {
    public record Party(String name, String role, String contactInfo) {}
    public record Clause(String title, String summary, String risk) {}
    public record RiskAssessment(String level, List<String> concerns, 
                                  String recommendation) {}
    public record Obligation(String party, String description, 
                              LocalDate deadline) {}
}
```

### 6.2 自定义Parser

```java
@Component
public class ContractAnalysisParser implements OutputParser<ContractAnalysis> {

    private final ObjectMapper objectMapper;

    public ContractAnalysisParser(ObjectMapper objectMapper) {
        this.objectMapper = objectMapper;
    }

    @Override
    public ContractAnalysis parse(String aiResponse) {
        try {
            // 1. 提取JSON（处理各种包裹情况）
            String json = extractJson(aiResponse);
            
            // 2. 反序列化
            return objectMapper.readValue(json, ContractAnalysis.class);
            
        } catch (Exception e) {
            throw new OutputParseException(
                "合同分析结果解析失败，原始输出: " + aiResponse, e);
        }
    }

    @Override
    public String getFormat() {
        return """
            请严格按照以下JSON格式输出（不要带markdown代码块标记）：
            {
              "contractType": "合同类型",
              "effectiveDate": "yyyy-MM-dd",
              "expirationDate": "yyyy-MM-dd",
              "parties": [{"name": "", "role": "", "contactInfo": ""}],
              "keyClauses": [{"title": "", "summary": "", "risk": ""}],
              "riskAssessment": {
                "level": "高/中/低",
                "concerns": ["风险点"],
                "recommendation": "建议"
              },
              "obligations": [{"party": "", "description": "", 
                               "deadline": "yyyy-MM-dd"}]
            }
            """;
    }

    private String extractJson(String text) {
        // 处理markdown代码块
        text = text.replaceAll("```json\\s*", "")
                   .replaceAll("```\\s*", "")
                   .trim();
        
        // 找到JSON的起止位置
        int start = text.indexOf('{');
        int end = text.lastIndexOf('}');
        
        if (start >= 0 && end > start) {
            return text.substring(start, end + 1);
        }
        throw new OutputParseException("无法从回复中提取JSON");
    }
}
```

### 6.3 使用自定义Parser

```java
@Service
public class ContractAnalysisService {

    private final ChatClient chatClient;
    private final ContractAnalysisParser parser;

    public ContractAnalysis analyze(String contractText) {
        String aiContent = chatClient.prompt()
            .system("你是一个专业的合同审查律师。请仔细分析以下合同内容。")
            .system(parser.getFormat())  // 注入输出格式约束
            .user(contractText)
            .call()
            .content();

        return parser.parse(aiContent);
    }
}
```

---

## 七、解析失败的容错处理

### 7.1 策略一：Fallback 降级

```java
@Component
public class ResilientOutputParser<T> {

    private final BeanOutputParser<T> primaryParser;
    private final ObjectMapper objectMapper;
    private final Class<T> targetType;

    public ResilientOutputParser(Class<T> targetType, ObjectMapper om) {
        this.primaryParser = BeanOutputParser.newInstance(targetType);
        this.objectMapper = om;
        this.targetType = targetType;
    }

    public T parse(String aiResponse) {
        try {
            // 策略1：直接解析
            return primaryParser.parse(aiResponse);
        } catch (Exception e1) {
            log.warn("Primitive parse failed, trying fallback strategies", e1);
            
            try {
                // 策略2：清理后重试
                String cleaned = cleanResponse(aiResponse);
                return primaryParser.parse(cleaned);
            } catch (Exception e2) {
                log.warn("Cleaned parse failed, trying raw JSON extraction", e2);
                
                try {
                    // 策略3：纯JSON提取
                    String rawJson = extractJsonBrutally(aiResponse);
                    return objectMapper.readValue(rawJson, targetType);
                } catch (Exception e3) {
                    log.error("All parsing strategies failed", e3);
                    return createFallback(aiResponse);
                }
            }
        }
    }

    private String cleanResponse(String text) {
        // 移除markdown代码块标记
        text = text.replaceAll("```json\\s*", "")
                   .replaceAll("```\\s*", "");
        // 移除开头的中文说明
        text = text.replaceAll("^[^{]*\\{", "{");
        // 移除结尾的补充说明
        int lastBrace = text.lastIndexOf('}');
        if (lastBrace > 0) {
            text = text.substring(0, lastBrace + 1);
        }
        return text.trim();
    }

    private String extractJsonBrutally(String text) {
        int start = text.indexOf('{');
        int end = text.lastIndexOf('}');
        if (start < 0 || end <= start) {
            throw new OutputParseException("No JSON found in response");
        }
        return text.substring(start, end + 1);
    }

    private T createFallback(String rawResponse) {
        // 创建一个部分填充的对象
        T fallback = instantiate(targetType);
        // 尝试设置通用字段
        try {
            Field field = targetType.getDeclaredField("rawResponse");
            field.setAccessible(true);
            field.set(fallback, rawResponse);
        } catch (Exception ignored) {}
        return fallback;
    }
}
```

### 7.2 策略二：重试机制

```java
@Component
public class RetryingOutputParser<T> {

    private final ChatClient chatClient;
    private final BeanOutputParser<T> parser;
    private final int maxRetries;

    public RetryingOutputParser(ChatClient.Builder builder, 
                                 Class<T> targetType,
                                 @Value("${ai.parser.max-retries:3}") int maxRetries) {
        this.chatClient = builder.build();
        this.parser = BeanOutputParser.newInstance(targetType);
        this.maxRetries = maxRetries;
    }

    public T parseWithRetry(String userMessage) {
        String lastResponse = null;
        
        for (int attempt = 0; attempt < maxRetries; attempt++) {
            try {
                String prompt = attempt == 0 ? userMessage :
                    buildRetryPrompt(userMessage, lastResponse);
                
                String aiContent = chatClient.prompt()
                    .user(prompt)
                    .call()
                    .content();
                lastResponse = aiContent;

                return parser.parse(aiContent);
                
            } catch (Exception e) {
                log.warn("Parse attempt {} failed: {}", 
                         attempt + 1, e.getMessage());
                
                if (attempt == maxRetries - 1) {
                    throw new OutputParseException(
                        "After " + maxRetries + " attempts, parsing still failed. " +
                        "Last response: " + lastResponse, e);
                }
            }
        }
        throw new IllegalStateException("Unreachable");
    }

    private String buildRetryPrompt(String originalPrompt, String lastResponse) {
        return originalPrompt + """
            
            你之前的输出格式不正确。
            请严格按照JSON格式重新输出，不要包含任何解释文字。
            只返回合法的JSON对象。
            """;
    }
}
```

### 7.3 策略三：流式解析的容错

```java
@Component
public class StreamingOutputParser {

    private final ObjectMapper objectMapper;

    /**
     * 流式解析——适配SSE场景
     */
    public <T> T parseStreaming(Flux<String> tokenStream, Class<T> targetType) {
        StringBuilder buffer = new StringBuilder();
        int braceCount = 0;
        boolean started = false;

        return tokenStream
            .doOnNext(token -> {
                if (token.contains("{")) started = true;
                if (started) {
                    buffer.append(token);
                    braceCount += countChar(token, '{');
                    braceCount -= countChar(token, '}');
                }
            })
            .takeUntil(token -> started && braceCount == 0 && 
                              buffer.toString().contains("}"))
            .then(Mono.fromCallable(() -> {
                String json = buffer.toString();
                // 清理非JSON前缀
                int start = json.indexOf('{');
                if (start > 0) json = json.substring(start);
                return objectMapper.readValue(json, targetType);
            }))
            .block();
    }

    private int countChar(String s, char c) {
        return (int) s.chars().filter(ch -> ch == c).count();
    }
}
```

---

## 八、高性能使用——集中管理所有Parser

```java
@Configuration
public class OutputParserConfig {

    @Bean
    public BeanOutputParser<SentimentResult> sentimentParser() {
        return BeanOutputParser.newInstance(SentimentResult.class);
    }

    @Bean
    public BeanOutputParser<ProductReviewAnalysis> reviewParser() {
        return BeanOutputParser.newInstance(ProductReviewAnalysis.class);
    }

    @Bean
    public BeanOutputParser<List<ProductRecommendation>> recommendListParser() {
        return BeanOutputParser.newListInstance(ProductRecommendation.class);
    }

    @Bean
    public MapOutputParser mapParser() {
        return new MapOutputParser();
    }

    @Bean
    public ListOutputParser listParser() {
        return new ListOutputParser();
    }
}
```

### 8.1 统一注入使用

```java
@Service
public class AIServiceFacade {

    private final ChatClient chatClient;
    private final BeanOutputParser<SentimentResult> sentimentParser;
    private final BeanOutputParser<ProductReviewAnalysis> reviewParser;
    private final MapOutputParser mapParser;
    private final ListOutputParser listParser;

    // 构造函数注入所有Parser

    public SentimentResult analyzeSentiment(String text) {
        String content = chatClient.prompt().user(text).call().content();
        return sentimentParser.parse(content);
    }

    public ProductReviewAnalysis analyzeReview(String review) {
        String content = chatClient.prompt().user(review).call().content();
        return reviewParser.parse(content);
    }

    public Map<String, String> extractKeywords(String text) {
        String content = chatClient.prompt().user(text).call().content();
        return mapParser.parse(content);
    }
}
```

---

## 九、不同模型对输出格式支持的对比

| 模型 | JSON输出稳定性 | 备注 |
|------|---------------|------|
| GPT-4o | ★★★★★ | 几乎总能正确输出JSON |
| GPT-4o-mini | ★★★★☆ | 偶尔需要重试 |
| Claude 3.5 Sonnet | ★★★★★ | JSON输出稳定，对Schema理解好 |
| DeepSeek V3 | ★★★★☆ | 测试中表现良好 |
| 通义千问 | ★★★★☆ | 中文Prompt下输出更稳定 |
| 本地Ollama模型 | ★★★☆☆ | 小模型可能不稳定，需加强容错 |

---

## 十、最佳实践总结

```java
// 1. 给你的POJO加上清晰的Jackson注解
public record AnalysisResult(
    @JsonProperty("sentiment") 
    @JsonPropertyDescription("情感倾向：positive/negative/neutral")
    String sentiment,
    
    @JsonProperty("score") 
    @JsonPropertyDescription("情感强度分数，0-10")
    int score,
    
    @JsonProperty("keywords")
    @JsonPropertyDescription("关键主题词列表")
    List<String> keywords
) {}
```

> **注**：`@JsonPropertyDescription` 来自 `com.fasterxml.jackson.annotation`，会被BeanOutputParser提取到JSON Schema中展示给AI。

核心原则：
1. **POJO字段要有描述性的命名和注解**——AI是"看到"这些信息来决定输出什么的
2. **区分场景选Parser**：单对象用BeanOutputParser，列表用ListOutputParser，KV用MapOutputParser
3. **永远不要信任AI的输出**——务必加容错和降级
4. **重试是一种有效的容错手段**——告诉AI"你上次格式不对"，它真的会改正
5. **Parser集中管理**——统一注册为Bean，方便复用

---

## 十一、小结

OutputParser 帮你把AI从"文本生成器"变成了"数据源"——从此以后，AI的输出不再是需要人工解析的字符串，而是可以直接被你的业务代码消费的POJO对象。

**核心收益**：
- 告别手写JSON提取正则
- AI的输出直接被Jackson反序列化为强类型POJO
- 嵌套结构自动处理
- 多种Parser覆盖不同场景
- 容错机制保证生产可用

当你能让AI返回结构化数据，下一步就是让它基于海量文档来回答问题——这就是 **RAG（检索增强生成）**，我们下一篇见。

---

> **下一篇预告**：Spring AI RAG完整方案——从文档上传到智能问答，一天搭建企业级知识库。从PDF读取到向量检索，全链路代码实战！

---

*如果觉得有帮助，欢迎点赞收藏关注，你的支持是我持续输出的动力！*

---
