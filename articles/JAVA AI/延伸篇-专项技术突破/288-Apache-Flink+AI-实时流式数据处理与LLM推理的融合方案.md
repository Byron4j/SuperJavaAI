# Apache Flink + AI：实时流式数据处理与 LLM 推理的融合方案，流式 AI 从此不同

## 开场白：当实时流遇上大模型

你刷抖音时看到的推荐内容、你在电商平台看到的实时促销弹窗、银行风控系统秒级拦截的欺诈交易——这些场景的背后，都是实时流处理引擎在驱动。

但传统流处理只能做规则匹配和简单统计。当需要对流中的每条数据进行语义理解、情感分析、内容摘要时，LLM 是唯一的选择。

本文教你如何将 Apache Flink 与 LLM 深度融合，构建 **流式 AI 处理管道**——实时接入、实时推理、实时输出，毫秒级延迟。

## 一、为什么是 Flink + AI？

### 1.1 三大流处理框架的真实对比

| 特性 | Apache Flink | Kafka Streams | Spark Streaming |
|------|-------------|---------------|-----------------|
| 处理模型 | 真正的流处理 | 流处理 | 微批处理 |
| 延迟 | 毫秒级 | 毫秒级 | 秒级 |
| 状态管理 | 内置RocksDB | 内置RocksDB | 需外部存储 |
| SQL 支持 | 完善 | 有限 | 完善 |
| AI 扩展性 | AsyncDataStream | 需手写 | 需手写 |
| 背压处理 | 天然支持 | 天然支持 | 微批限制 |

**Flink 是唯一同时具备毫秒级延迟 + 强大状态管理 + 原生异步 I/O 的流处理引擎**，天然适合 LLM 集成。

### 1.2 核心架构

```
┌──────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────┐
│  Kafka   │───▶│ Flink Source │───▶│ Async I/O    │───▶│  Kafka   │
│ (输入流)  │    │   (接入)     │    │ + LLM 推理   │    │ (输出流)  │
└──────────┘    └──────────────┘    └──────┬───────┘    └──────────┘
                                           │
                                    ┌──────▼───────┐
                                    │  LLM 服务     │
                                    │ (GPT-4/本地)  │
                                    └──────────────┘
```

## 二、实战：构建实时评论情感分析管道

### 2.1 场景描述

抖音/微博的实时评论流（Kafka Topic: `comments-raw`）→ Flink 消费 → 经过以下三步处理 → 输出到 Kafka Topic `comments-enriched`：

1. **敏感词过滤**（本地规则引擎）
2. **情感分析**（调用 LLM）
3. **异常检测**（Flink CEP 复杂事件处理）

### 2.2 Maven 依赖

```xml
<properties>
    <flink.version>1.19.0</flink.version>
    <java.version>11</java.version>
</properties>

<dependencies>
    <!-- Flink 核心 -->
    <dependency>
        <groupId>org.apache.flink</groupId>
        <artifactId>flink-streaming-java</artifactId>
        <version>${flink.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.flink</groupId>
        <artifactId>flink-connector-kafka</artifactId>
        <version>3.1.0-1.19</version>
    </dependency>
    <dependency>
        <groupId>org.apache.flink</groupId>
        <artifactId>flink-cep</artifactId>
        <version>${flink.version}</version>
    </dependency>

    <!-- HTTP 客户端（用于 AI API 调用） -->
    <dependency>
        <groupId>com.squareup.okhttp3</groupId>
        <artifactId>okhttp</artifactId>
        <version>4.12.0</version>
    </dependency>

    <!-- JSON -->
    <dependency>
        <groupId>com.fasterxml.jackson.core</groupId>
        <artifactId>jackson-databind</artifactId>
        <version>2.17.0</version>
    </dependency>
</dependencies>
```

### 2.3 核心 Job 代码

```java
import org.apache.flink.api.common.eventtime.WatermarkStrategy;
import org.apache.flink.cep.CEP;
import org.apache.flink.cep.PatternStream;
import org.apache.flink.cep.functions.PatternProcessFunction;
import org.apache.flink.cep.pattern.Pattern;
import org.apache.flink.streaming.api.datastream.AsyncDataStream;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.functions.async.AsyncFunction;
import org.apache.flink.streaming.api.functions.async.ResultFuture;

import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicInteger;

public class AIStreamingJob {

    private static final OkHttpClient HTTP_CLIENT = new OkHttpClient.Builder()
        .connectTimeout(10, TimeUnit.SECONDS)
        .readTimeout(30, TimeUnit.SECONDS)
        .connectionPool(new ConnectionPool(100, 5, TimeUnit.MINUTES))
        .build();

    private static final ObjectMapper MAPPER = new ObjectMapper();

    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env =
            StreamExecutionEnvironment.getExecutionEnvironment();

        // 1. 从 Kafka 读取原始评论流
        KafkaSource<Comment> source = KafkaSource.<Comment>builder()
            .setBootstrapServers("localhost:9092")
            .setTopics("comments-raw")
            .setGroupId("ai-enrichment-group")
            .setDeserializer(new CommentDeserializer())
            .setStartingOffsets(OffsetsInitializer.latest())
            .build();

        DataStream<Comment> stream = env.fromSource(
            source, WatermarkStrategy.noWatermarks(), "Kafka Source");

        // 2. 敏感词过滤（本地规则，低延迟）
        DataStream<Comment> filtered = stream
            .filter(comment -> !containsSensitiveWords(comment.getContent()))
            .name("SensitiveFilter");

        // 3. 异步调用 LLM 进行情感分析
        DataStream<EnrichedComment> enriched = AsyncDataStream
            .unorderedWait(
                filtered,
                new SentimentAnalysisFunction(),
                30, TimeUnit.SECONDS,    // 超时
                100                       // 最大并发（配合LLM的并发限制）
            )
            .name("SentimentAnalysis");

        // 4. CEP 异常检测：连续3条负面评论 → 触发告警
        Pattern<EnrichedComment> negativePattern = Pattern
            .<EnrichedComment>begin("first")
            .where(c -> "NEGATIVE".equals(c.getSentiment()))
            .next("second")
            .where(c -> "NEGATIVE".equals(c.getSentiment()))
            .next("third")
            .where(c -> "NEGATIVE".equals(c.getSentiment()))
            .within(Time.minutes(5));

        PatternStream<EnrichedComment> patternStream = CEP.pattern(
            enriched.keyBy(EnrichedComment::getTopicId),
            negativePattern);

        DataStream<Alert> alerts = patternStream.process(
            new PatternProcessFunction<>() {
                @Override
                public void processMatch(
                        Map<String, List<EnrichedComment>> match,
                        Context ctx,
                        Collector<Alert> out) {
                    List<EnrichedComment> comments = match.get("first");
                    comments.addAll(match.get("second"));
                    comments.addAll(match.get("third"));
                    out.collect(new Alert("NEGATIVE_SURGE",
                        "Topic " + comments.get(0).getTopicId()
                        + " has 3 consecutive negative comments"));
                }
            });

        // 5. 输出到 Kafka
        enriched.sinkTo(sink("comments-enriched"));
        alerts.sinkTo(sink("comment-alerts"));

        env.execute("AI-Powered Comment Analysis Pipeline");
    }
}
```

### 2.4 异步 LLM 推理函数

```java
public class SentimentAnalysisFunction
        implements AsyncFunction<Comment, EnrichedComment> {

    private static final String LLM_ENDPOINT =
        "http://localhost:8000/v1/chat/completions";

    @Override
    public void asyncInvoke(Comment comment,
                             ResultFuture<EnrichedComment> resultFuture) {

        CompletableFuture.supplyAsync(() -> {
            try {
                String sentiment = analyzeSentiment(comment.getContent());
                return new EnrichedComment(comment, sentiment);
            } catch (Exception e) {
                return new EnrichedComment(comment, "ERROR");
            }
        }).thenAccept(result -> {
            resultFuture.complete(Collections.singleton(result));
        });
    }

    private String analyzeSentiment(String content) throws Exception {
        String prompt = """
            分析以下评论的情感倾向，仅回复 POSITIVE、NEGATIVE 或 NEUTRAL：
            评论：%s
            情感：""".formatted(content);

        String jsonBody = MAPPER.writeValueAsString(Map.of(
            "model", "gpt-3.5-turbo",
            "messages", List.of(Map.of("role", "user", "content", prompt)),
            "max_tokens", 10,
            "temperature", 0
        ));

        Request request = new Request.Builder()
            .url(LLM_ENDPOINT)
            .post(RequestBody.create(jsonBody,
                MediaType.parse("application/json")))
            .header("Authorization", "Bearer " + System.getenv("LLM_API_KEY"))
            .build();

        try (Response response = HTTP_CLIENT.newCall(request).execute()) {
            String body = response.body().string();
            JsonNode node = MAPPER.readTree(body);
            return node.get("choices").get(0)
                .get("message").get("content").asText().strip();
        }
    }
}
```

## 三、Flink 状态管理：跨消息的内存与上下文

LLM 推理往往需要上下文。比如分析一个用户的连续评论，需要知道他刚刚说过什么。

### 3.1 使用 KeyedState 维护用户会话上下文

```java
import org.apache.flink.api.common.state.ValueState;
import org.apache.flink.api.common.state.ValueStateDescriptor;
import org.apache.flink.configuration.Configuration;
import org.apache.flink.streaming.api.functions.async.RichAsyncFunction;

public class ContextAwareLLMFunction
        extends RichAsyncFunction<Comment, EnrichedComment> {

    private transient ValueState<ConversationContext> contextState;

    @Override
    public void open(Configuration parameters) {
        ValueStateDescriptor<ConversationContext> descriptor =
            new ValueStateDescriptor<>(
                "conversation-context",
                ConversationContext.class);
        contextState = getRuntimeContext().getState(descriptor);
    }

    @Override
    public void asyncInvoke(Comment comment,
                             ResultFuture<EnrichedComment> resultFuture) {

        CompletableFuture.supplyAsync(() -> {
            try {
                ConversationContext ctx = contextState.value();
                if (ctx == null) {
                    ctx = new ConversationContext(comment.getUserId());
                }

                // 把历史上下文注入 prompt
                String enhancedPrompt = buildPromptWithContext(
                    comment.getContent(), ctx.getHistory());

                String response = callLLM(enhancedPrompt);

                // 更新状态
                ctx.addMessage("user", comment.getContent());
                ctx.addMessage("assistant", response);
                contextState.update(ctx);

                return new EnrichedComment(comment, response);

            } catch (Exception e) {
                return new EnrichedComment(comment, "ERROR: " + e.getMessage());
            }
        }).thenAccept(result -> {
            resultFuture.complete(Collections.singleton(result));
        });
    }
}
```

### 3.2 使用 TTL 防止状态无限增长

```java
StateTtlConfig ttlConfig = StateTtlConfig
    .newBuilder(Time.hours(24))
    .setUpdateType(StateTtlConfig.UpdateType.OnCreateAndWrite)
    .setStateVisibility(StateTtlConfig.StateVisibility.NeverReturnExpired)
    .build();

ValueStateDescriptor<ConversationContext> descriptor =
    new ValueStateDescriptor<>("context", ConversationContext.class);
descriptor.enableTimeToLive(ttlConfig);
```

## 四、Flink SQL + AI：声明式流式 AI

如果不想写 Java 代码，Flink SQL 也能整合 AI：

```sql
-- 1. 定义 Kafka Source 表
CREATE TABLE comments_raw (
    user_id    STRING,
    content    STRING,
    timestamp  TIMESTAMP(3),
    WATERMARK FOR timestamp AS timestamp - INTERVAL '5' SECOND
) WITH (
    'connector' = 'kafka',
    'topic' = 'comments-raw',
    'properties.bootstrap.servers' = 'localhost:9092',
    'format' = 'json'
);

-- 2. 定义 UDF：调用 LLM 的情感分析函数
CREATE FUNCTION sentiment_analysis
    AS 'com.example.udf.SentimentAnalysisUDF';

-- 3. 流式 SQL + AI
SELECT
    user_id,
    content,
    sentiment_analysis(content) AS sentiment,
    timestamp
FROM comments_raw
WHERE sentiment_analysis(content) <> 'ERROR';
```

UDF 实现：

```java
public class SentimentAnalysisUDF extends ScalarFunction {

    private static final OkHttpClient client = new OkHttpClient();

    public String eval(String content) {
        // ... 与之前相同的 LLM 调用逻辑
        return callLLM(content);
    }
}
```

## 五、生产级优化策略

### 5.1 LLM 调用的批处理优化

单条调用 LLM 效率低，可以微批处理（Micro-batching）：

```java
public class MicroBatcherLLMFunction
        extends ProcessWindowFunction<Comment, EnrichedComment, String, TimeWindow> {

    @Override
    public void process(String key, Context context,
                        Iterable<Comment> elements,
                        Collector<EnrichedComment> out) {

        List<Comment> batch = new ArrayList<>();
        elements.forEach(batch::add);

        // 一次 LLM 调用处理多条评论（批量情感分析）
        List<String> results = batchAnalyzeWithLLM(batch);

        for (int i = 0; i < batch.size(); i++) {
            out.collect(new EnrichedComment(batch.get(i), results.get(i)));
        }
    }
}
```

### 5.2 本地小模型 + 远程大模型的分级策略

```
                  ┌──────────────┐
                  │  入站评论流   │
                  └──────┬───────┘
                         │
                  ┌──────▼───────┐
                  │ 本地规则引擎  │ (敏感词过滤, 0.01ms)
                  └──────┬───────┘
                         │
                  ┌──────▼───────┐
                  │ 本地小模型    │ (FastText/ONNX, 5ms)
                  │ 置信度>0.9?  │
                  └──┬───────┬───┘
                     │       │
              YES ◄──┘       └──► NO (低置信度)
              │                    │
        ┌─────▼─────┐      ┌──────▼──────┐
        │ 输出结果   │      │ 远程GPT-4   │ (200ms)
        └───────────┘      └─────────────┘
```

```java
public class TieredInferenceFunction
        extends RichAsyncFunction<Comment, EnrichedComment> {

    private ONNXModel localModel;
    private LLMClient remoteLLM;

    @Override
    public void open(Configuration parameters) {
        localModel = new ONNXModel("sentiment-model.onnx");
        remoteLLM = new LLMClient();
    }

    @Override
    public void asyncInvoke(Comment comment,
                             ResultFuture<EnrichedComment> resultFuture) {
        // 先用本地小模型快速推断
        ClassificationResult localResult = localModel.predict(comment.getContent());

        if (localResult.getConfidence() > 0.9) {
            resultFuture.complete(Collections.singleton(
                new EnrichedComment(comment, localResult.getLabel())));
            return;
        }

        // 低置信度才调用远程大模型
        CompletableFuture.supplyAsync(() -> {
            String result = remoteLLM.analyzeSentiment(comment.getContent());
            return new EnrichedComment(comment, result);
        }).thenAccept(r ->
            resultFuture.complete(Collections.singleton(r)));
    }
}
```

### 5.3 反压与限流

Flink 天然支持反压，但 LLM 服务有严格的 QPS 限制。需要主动控制：

```java
// Flink 配置
env.getConfig().setAutoWatermarkInterval(200);
env.setParallelism(4);

// 配合 Guava RateLimiter
RateLimiter rateLimiter = RateLimiter.create(50.0); // 50 QPS

private String callLLMWithRateLimit(String prompt) {
    rateLimiter.acquire();  // 阻塞直到获取令牌
    return doActualLLMCall(prompt);
}
```

## 六、独特观点：流式 AI 的三层架构

我认为未来的流式 AI 系统应采用以下三层架构：

```
┌─────────────────────────────────────────────┐
│              第一层：边缘推理 (Edge)          │
│  本地 ONNX/TensorFlow Lite 模型              │
│  延迟 < 10ms, 处理 80% 的简单场景            │
├─────────────────────────────────────────────┤
│              第二层：区域推理 (Fog)           │
│  Flink + 中等规模模型 (7B-13B 参数)          │
│  延迟 < 200ms, 处理 15% 的复杂场景           │
├─────────────────────────────────────────────┤
│              第三层：云端推理 (Cloud)          │
│  GPT-4/Claude 等顶级大模型                   │
│  延迟 < 2s, 处理 5% 的超复杂/高价值场景       │
└─────────────────────────────────────────────┘

代价 = P1 × 0.01 + P2 × 0.5 + P3 × 2.0  (按权重)
     = 0.008 + 0.075 + 0.10 = 0.183 元/条
     
vs 全部用GPT-4 = 2.0 元/条，节省 91% 成本！
```

## 七、总结

Apache Flink 为 LLM 推理提供了一个完美的流式"底座"：

- **AsyncDataStream** 让 AI 调用不阻塞流处理管道
- **状态管理** 让跨消息的 LLM 上下文成为可能
- **CEP 复杂事件处理** 让 AI 结果与事件模式深度联动
- **分层推理** 可以用 10% 的成本覆盖 95% 的场景

AI 不只是离线批处理的事后分析，它应该像血液一样在实时数据管道中流动。

---

**本文是「Java + AI 编程从入门到精通」310 篇系列的第 288 篇。全系列覆盖：**

- 基础篇（1-50）：Java 基础 + AI 概念入门
- 框架篇（51-120）：Spring AI、LangChain4j、Semantic Kernel
- 进阶篇（121-200）：RAG、Agent、多模态、微调
- 实战篇（201-250）：企业级 AI 应用实战
- 延伸篇（251-310）：专项技术突破、性能优化、架构设计

**系列全部文章持续更新中，关注获取最新内容。**
