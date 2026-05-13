# 3 个 AI 编程领域必读的 GitHub 开源项目（含项目结构精讲），读源码是快速进阶的秘密

## 开场白：Star 多不代表值得读

每天都有新的 AI 开源项目——LangChain 57k Star、AutoGPT 164k Star、Dify 42k Star…… 但 Star 数不等于学习价值。很多爆火项目是"胶水代码"堆砌，读完除了学会调 API，什么都没得到。

真正值得读源码的项目，应该满足三个标准：

1. **架构设计精良**：模块解耦、扩展点清晰、设计模式运用得当
2. **工程实践扎实**：测试覆盖、代码规范、CI/CD、文档完备
3. **与 Java 关系紧密**：要么是 Java 实现，要么其设计理念可以移植到 Java

基于以上标准，我精选了三个项目。读完它们的源码，你对 AI 编程的认知将上升一个层次。

## 一、LangChain4j：Java 生态的 AI 编排框架

### 1.1 为什么读它

LangChain4j 不是 LangChain 的 Java 翻译版——它从零开始设计，充分考虑了 Java 的类型系统、并发模型和生态现状，是 Java AI 编程的最佳入门源码。

```yaml
基本信息:
  GitHub: https://github.com/langchain4j/langchain4j
  Stars: 4.5k+
  语言: Java 100%
  构建: Maven
  模块数: 30+
  最新版本: 0.32.0
```

### 1.2 项目结构精讲

```
langchain4j/
├── langchain4j-core/          # 核心抽象层
│   ├── src/main/java/dev/langchain4j/
│   │   ├── model/             # ⭐ 模型抽象（ChatModel/EmbeddingModel）
│   │   │   ├── input/         # Prompt 模板系统
│   │   │   ├── output/        # 输出解析器（结构化输出）
│   │   │   └── chat/          # 对话模型抽象
│   │   ├── memory/            # ⭐ 对话记忆机制
│   │   ├── store/             # 嵌入向量存储
│   │   ├── rag/               # ⭐ RAG 检索增强生成
│   │   │   ├── query/         # 查询转换器
│   │   │   ├── retrieval/     # 检索器
│   │   │   └── content/       # 内容整理器
│   │   ├── agent/             # ⭐ Agent 框架
│   │   │   └── tool/          # 工具系统（Java 方法→AI 工具）
│   │   ├── service/           # ⭐ AI Service（最值得看的模块）
│   │   ├── data/              # 用户/助手消息类型
│   │   └── classification/    # 文本分类
│   │
│   └── test/                  # 丰富的测试用例
│
├── langchain4j-open-ai/       # OpenAI 模型集成
├── langchain4j-ollama/        # Ollama 本地模型集成
├── langchain4j-vertex-ai/     # Google Vertex AI
├── langchain4j-qdrant/        # Qdrant 向量数据库
├── langchain4j-pinecone/      # Pinecone 向量数据库
├── langchain4j-milvus/        # Milvus 向量数据库
├── langchain4j-elasticsearch/ # ES 向量搜索
└── ...                         # 其他集成模块
```

### 1.3 核心设计模式：AI Service 的注解驱动魔法

这是 LangChain4j 最精妙的设计——通过 Java 注解将普通接口变成 AI 驱动的服务：

```java
/**
 * LangChain4j AI Service 的核心实现逻辑（简化版）
 * 源码路径：langchain4j-core/src/main/java/dev/langchain4j/service/
 */
public class AiServiceFactory {

    public static <T> T create(Class<T> interfaceClass,
                                ChatModel chatModel) {
        // 1. 分析接口上的注解
        SystemMessage systemAnnotation =
            interfaceClass.getAnnotation(SystemMessage.class);

        // 2. 为每个方法创建动态代理
        return (T) Proxy.newProxyInstance(
            interfaceClass.getClassLoader(),
            new Class[]{interfaceClass},
            (proxy, method, args) -> {
                // 3. 收集方法上的注解信息
                UserMessage userAnnotation =
                    method.getAnnotation(UserMessage.class);
                V vAnnotation =
                    method.getAnnotation(V.class);

                // 4. 构建 Prompt
                PromptTemplate template = PromptTemplate.from(
                    userAnnotation.value());
                Map<String, Object> variables = new HashMap<>();
                Parameter[] params = method.getParameters();
                for (int i = 0; i < params.length; i++) {
                    V paramV = params[i].getAnnotation(V.class);
                    variables.put(paramV.value(), args[i]);
                }
                String prompt = template.apply(variables);

                // 5. 调用 LLM
                String response = chatModel.generate(prompt);

                // 6. 类型转换（如果返回值不是 String）
                return convertResponse(response, method.getReturnType());
            });
    }
}
```

**精读建议**：重点看 `AiServiceFactory` 和 `AiServiceStreamingResponseHandler`，理解 Java 动态代理如何与 AI 调用深度融合。这是用 Java 元编程能力封装复杂 AI 流程的教科书级范例。

### 1.4 值得学习的工程实践

```java
// 1. Builder 模式的极致运用
ChatModel model = OpenAiChatModel.builder()
    .apiKey(System.getenv("OPENAI_API_KEY"))
    .modelName("gpt-4")
    .temperature(0.7)
    .maxRetries(RetryPolicy.builder()
        .maxRetries(3)
        .delay(Duration.ofSeconds(1))
        .exponentialBackoff(2.0)
        .build())
    .build();

// 2. 工具系统的声明式定义
@Tool("查询指定城市的天气信息")
public String getWeather(
    @P("城市名称") String city,
    @P("日期，格式yyyy-MM-dd") String date) {
    return weatherApi.query(city, date);
}

// 3. 流式响应的回调处理
StreamingChatModel streamingModel = OpenAiStreamingChatModel.builder()
    .apiKey(key)
    .onPartialResponse(partial -> System.out.print(partial))
    .onComplete(total -> System.out.println("\n[总耗时: " + total.duration() + "]"))
    .onError(Throwable::printStackTrace)
    .build();
```

## 二、Spring AI：Spring 生态的 AI 基础设施

### 2.1 为什么读它

Spring AI 是 Spring 官方推出的 AI 集成框架，目前处于 1.0.0-M2 阶段。读它的源码，你能看到 Spring 团队如何将 20 年的企业级框架设计经验应用到 AI 领域。

```yaml
基本信息:
  GitHub: https://github.com/spring-projects/spring-ai
  Stars: 3.5k+
  语言: Java 100%
  构建: Gradle
  模块数: 20+
  最新版本: 1.0.0-M2
```

### 2.2 项目结构精讲

```
spring-ai/
├── spring-ai-commons/                  # 公共抽象层
│   └── src/main/java/org/springframework/ai/
│       ├── chat/                       # ⭐ 对话模型抽象
│       │   ├── model/
│       │   │   ├── ChatModel.java      # 核心接口
│       │   │   ├── StreamingChatModel.java
│       │   │   └── ChatResponse.java
│       │   └── prompt/
│       │       ├── PromptTemplate.java
│       │       └── ChatOptions.java
│       ├── embedding/                  # ⭐ Embedding 抽象
│       │   ├── EmbeddingModel.java
│       │   └── EmbeddingOptions.java
│       ├── vectorstore/               # ⭐ 向量存储抽象
│       │   ├── VectorStore.java
│       │   └── SearchRequest.java
│       ├── document/                   # 文档模型 + ETL
│       │   ├── Document.java
│       │   └── reader/
│       ├── model/                      # 模型结果/状态
│       ├── moderation/                 # 内容审查
│       └── retry/                      # 重试策略
│
├── spring-ai-openai/                   # OpenAI 实现
├── spring-ai-ollama/                   # Ollama 实现
├── spring-ai-azure-openai/            # Azure OpenAI
├── spring-ai-vertex-ai/              # Google Vertex AI
├── spring-ai-transformers/            # Hugging Face 集成
│
├── spring-ai-spring-boot-autoconfigure/# ⭐ 自动配置（最关键）
│   └── src/main/java/.../
│       ├── OpenAiAutoConfiguration.java
│       └── OllamaAutoConfiguration.java
│
├── spring-ai-pgvector/                # PGVector 向量存储
├── spring-ai-qdrant/                  # Qdrant 向量存储
├── spring-ai-milvus/                  # Milvus 向量存储
├── spring-ai-chroma/                  # ChromaDB 向量存储
│
└── spring-ai-spring-boot-starters/    # Starter 汇总
```

### 2.3 最值得学习的代码：VectorStore 的抽象设计

```java
/**
 * Spring AI 的 VectorStore 接口（源码精要）
 * 展示了如何为异构向量数据库设计统一抽象
 */
public interface VectorStore {

    /**
     * 添加文档（自动 Embedding + 存储）
     */
    void add(List<Document> documents);

    /**
     * 语义搜索
     */
    default List<Document> similaritySearch(String query) {
        return similaritySearch(
            SearchRequest.defaults().withQuery(query));
    }

    /**
     * 带过滤条件的语义搜索
     */
    List<Document> similaritySearch(SearchRequest request);

    /**
     * 删除文档
     */
    void delete(List<String> idList);

    /**
     * Builder API 的可选实现
     */
    interface Builder<T extends VectorStore> {
        T build();
        Builder<T> withIndexName(String indexName);
        Builder<T> withEmbeddingModel(EmbeddingModel model);
    }
}

/**
 * SearchRequest：使用 Builder 构建复杂查询（SQL 风格的过滤语法）
 */
public class SearchRequest {

    private String query;
    private int topK = 4;
    private double similarityThreshold = 0.7;
    private FilterExpression filterExpression;

    // 核心亮点：类 SQL 的过滤表达式
    public SearchRequest withFilterExpression(String expression) {
        // 例如："author == '张三' && year >= 2024"
        this.filterExpression = new FilterExpression(expression);
        return this;
    }
}

/**
 * PGVector 的实际实现路径：
 * spring-ai-pgvector/src/main/java/.../PgVectorStore.java
 *
 * 核心 SQL（源码中的 interesting part）：
 *
 * SELECT d.id, d.content, d.metadata,
 *        1 - (e.embedding <=> %s::vector) AS similarity
 * FROM documents d
 * JOIN embeddings e ON d.id = e.document_id
 * WHERE 1 - (e.embedding <=> %s::vector) > %f
 * ORDER BY similarity DESC
 * LIMIT %d
 */
```

### 2.4 Spring AI 的 Advisors 模式

```java
/**
 * Spring AI 的 Advisor 链——类似于 AOP 的拦截器链
 * 但专门为 AI 调用设计
 */
public interface RequestResponseAdvisor {

    ChatResponse advise(
        ChatModel chatModel,
        ChatOptions chatOptions,
        String userText,
        Map<String, Object> advisorContext);

    /**
     * 用于排序，数字越小优先级越高
     */
    default int getOrder() {
        return 0;
    }
}

// Spring 风格的 AI Advisor 用法
@Configuration
public class AIAdvisorConfig {

    @Bean
    public ChatClient chatClient(ChatModel chatModel) {
        return ChatClient.builder(chatModel)
            .defaultAdvisors(
                new RAGRetrievalAugmentationAdvisor(vectorStore),
                new ContentGuardAdvisor(),           // 内容审查
                new LoggingAdvisor(),                // 日志记录
                new TokenCountTrackingAdvisor()      // Token 计数
            )
            .build();
    }
}
```

**精读建议**：重点看 `org.springframework.ai.vectorstore.VectorStore` 接口及其三个实现（PGVector、Qdrant、Milvus），体会 Spring 生态"面向接口编程"的核心哲学在 AI 领域的应用。

## 三、Jlama：纯 Java 实现的 LLM 推理引擎

### 3.1 为什么读它

Jlama 是极其罕见的纯 Java LLM 推理引擎——没有 JNI、没有 Python bridge、没有 native library。它在 JVM 上实现了一个完整的 Transformer 推理栈。读完它的源码，你会真正理解 LLM 的内部运作机制。

```yaml
基本信息:
  GitHub: https://github.com/tjake/Jlama
  Stars: 1k+
  语言: Java 100%
  构建: Maven
  特点: 纯Java、无JNI依赖、Panama Vector API加速
  支持模型: LLaMA/Mistral/Mixtral/Gemma/GPT-2/CodeGen
```

### 3.2 项目结构精讲

```
Jlama/
├── jlama-core/
│   └── src/main/java/com/github/tjake/jlama/
│       ├── model/                          # ⭐ 模型加载与结构定义
│       │   ├── AbstractModel.java          # 模型基类
│       │   ├── LlamaModel.java
│       │   ├── MistralModel.java
│       │   └── GemmaModel.java
│       │
│       ├── tensor/                         # ⭐ 张量计算（整个项目的心脏）
│       │   ├── AbstractTensor.java         # 抽象张量
│       │   ├── FloatTensor.java            # Float32 张量
│       │   ├── Q4ByteBufferTensor.java    # 4-bit 量化张量
│       │   ├── Q8ByteBufferTensor.java    # 8-bit 量化张量
│       │   ├── TensorOperations.java       # 张量操作接口
│       │   ├── Matrix.java                 # 矩阵乘法（用 Vector API）
│       │   └── operations/                 # 各种运算实现
│       │       ├── Gemm.java               # 矩阵乘法核心
│       │       ├── FlashAttention.java     # 注意力计算
│       │       └── RoPE.java               # 旋转位置编码
│       │
│       ├── math/                           # 数学工具
│       │   ├── Float16.java               # IEEE 754 半精度实现
│       │   └── BF16.java                   # Brain Float 16
│       │
│       ├── net/                            # 模型文件下载和解析
│       │   ├── GGUFReader.java            # GGUF 格式解析器
│       │   └── SafeTensorSupport.java     # SafeTensor 格式支持
│       │
│       ├── sampler/                        # 采样策略
│       │   ├── Sampler.java
│       │   ├── TopKSampler.java
│       │   ├── TopPSampler.java
│       │   └── TemperatureSampler.java
│       │
│       └── util/                           # ⭐ Panama Vector API 封装
│           ├── VectorUtil.java
│           └── PanamaUtil.java
│
├── jlama-native/                           # Panama FFI 实验
└── README.md
```

### 3.3 核心代码精读：Flash Attention 的 Java 实现

```java
/**
 * Jlama 中的 Flash Attention 实现（简化精要版）
 * 源码路径：jlama-core/src/.../tensor/operations/FlashAttention.java
 *
 * 这段代码的价值：用纯 Java + Vector API 实现了
 * 一整套 Transformer 推理时的注意力计算流程
 */
public class FlashAttention {

    public static AbstractTensor compute(
            AbstractTensor query,    // [nHeads, seqLen, headDim]
            AbstractTensor key,      // [nKvHeads, seqLen, headDim]
            AbstractTensor value,    // [nKvHeads, seqLen, headDim]
            AbstractTensor mask,     // [1, 1, seqLen, seqLen] (causal mask)
            boolean useCache) {

        // ===== Part 1: Q·K^T 注意力分数矩阵 =====
        AbstractTensor attnWeights = query.dotProduct(
            key,  // 批量矩阵乘法：O(nHeads × seqLen × seqLen)
            false, true  // Q不转置，K转置
        );

        // ===== Part 2: 缩放 =====
        float scale = (float) (1.0 / Math.sqrt(query.dims().last()));
        attnWeights = attnWeights.scale(scale);

        // ===== Part 3: 应用 Causal Mask =====
        attnWeights = attnWeights.accumulate(mask);

        // ===== Part 4: Softmax 沿最后一维 =====
        AbstractTensor attnProbs = attnWeights.softmax();

        // 如果使用了 KV Cache，复用历史计算结果
        if (useCache) {
            // 只计算新 token 与历史+当前 KV 的注意力
            // 时间复杂度从 O(n²) 降到 O(n)
        }

        // ===== Part 5: Attention × Value =====
        AbstractTensor output = attnProbs.dotProduct(
            value,
            false, false
        );

        return output;
    }
}

/**
 * 矩阵乘法的 Vector API 加速实现
 * 源码路径：jlama-core/src/.../tensor/operations/Gemm.java
 */
public class Gemm {

    private static final VectorSpecies<Float> SPECIES =
        FloatVector.SPECIES_PREFERRED;

    public static void gemm(
            float[] a, float[] b, float[] c,
            int m, int n, int k,
            boolean transposeA, boolean transposeB) {

        for (int i = 0; i < m; i++) {
            for (int j = 0; j < n; j++) {
                float sum = 0;
                int limit = SPECIES.loopBound(k);
                int l = 0;

                // Vector API 向量化计算
                for (; l < limit; l += SPECIES.length()) {
                    FloatVector va = FloatVector.fromArray(
                        SPECIES, a, index(i, l, k, transposeA));
                    FloatVector vb = FloatVector.fromArray(
                        SPECIES, b, index(l, j, n, transposeB));
                    sum += va.mul(vb).reduceLanes(
                        VectorOperators.ADD);
                }

                // 处理剩余元素
                for (; l < k; l++) {
                    sum += a[index(i, l, k, transposeA)]
                         * b[index(l, j, n, transposeB)];
                }

                c[index(i, j, n, false)] = sum;
            }
        }
    }
}
```

### 3.4 Panama Vector API：Java 的 SIMD 加速

Jlama 最惊艳的部分是其对 Java Panama Vector API 的运用。这个 API 在 JDK 16+ 中孵化，允许 Java 直接使用 CPU 的 SIMD 指令：

```java
/**
 * Jlama 使用 Panama Vector API 加速张量计算
 * 精要理解：不用 C++/CUDA，纯 Java 实现向量化计算
 */
public class VectorUtil {

    private static final VectorSpecies<Float> FSP = FloatVector.SPECIES_PREFERRED;

    /**
     * 向量化点积：v1·v2
     */
    public static float dot(float[] v1, float[] v2) {
        float sum = 0;
        int i = 0;
        int bound = FSP.loopBound(v1.length);

        for (; i < bound; i += FSP.length()) {
            FloatVector a = FloatVector.fromArray(FSP, v1, i);
            FloatVector b = FloatVector.fromArray(FSP, v2, i);
            sum += a.mul(b).reduceLanes(VectorOperators.ADD);
        }

        for (; i < v1.length; i++) {
            sum += v1[i] * v2[i];
        }
        return sum;
    }

    /**
     * 向量化 Softmax 加速
     */
    public static void softmax(float[] x, int offset, int length) {
        // 1. 向量化找最大值（防止指数溢出）
        float max = x[offset];
        for (int i = offset; i < offset + length; i++) {
            max = Math.max(max, x[i]);
        }

        // 2. 向量化 exp + sum
        float sum = 0;
        int i = offset;
        int bound = offset + FSP.loopBound(length);

        FloatVector maxVec = FloatVector.broadcast(FSP, max);
        for (; i < bound; i += FSP.length()) {
            FloatVector v = FloatVector.fromArray(FSP, x, i);
            // 在向量寄存器中计算 exp(x - max) 的近似值
            FloatVector expv = expApproximation(v.sub(maxVec));
            expv.intoArray(x, i);
            sum += expv.reduceLanes(VectorOperators.ADD);
        }

        // 3. 归一化
        FloatVector sumVec = FloatVector.broadcast(FSP, sum);
        for (i = offset; i < bound; i += FSP.length()) {
            FloatVector v = FloatVector.fromArray(FSP, x, i);
            v.div(sumVec).intoArray(x, i);
        }
    }
}
```

**精读建议**：从 `AbstractModel.generate()` 为主线，追踪一个 token 从输入到输出的全链路——Tokenizer → Embedding → Transformer Blocks → LM Head → Sampler。这是理解 LLM 内部机制的最佳方式。

## 四、三个项目的对比与学习路径

```
            ┌────────────────────────────────────────┐
            │          AI 编程能力进阶路径            │
            ├────────────────────────────────────────┤
            │                                        │
            │  入门级 ──▶ LangChain4j                  │
            │  学会：如何用Java调用AI、如何封装AI流程   │
            │  时间：1周，精读 core 和 open-ai 模块    │
            │                                        │
            │  中级   ──▶ Spring AI                     │
            │  学会：企业级AI架构设计、Spring生态整合    │
            │  时间：1周，重点看 autoconfigure 和       │
            │        vectorstore 抽象                 │
            │                                        │
            │  高级   ──▶ Jlama                         │
            │  学会：LLM内部原理、Panama Vector API、   │
            │        量化技术、GGUF格式                 │
            │  时间：2周，从 AbstractModel.generate()   │
            │        入口追踪全链路                     │
            │                                        │
            └────────────────────────────────────────┘
```

## 五、独特观点：读源码的三层境界

**第一层：看代码做什么（What）**
你看到 LangChain4j 用注解驱动 AI Service，学会了用 `@SystemMessage` 和 `@UserMessage`。这一层让你能"用"框架。

**第二层：看怎么做的（How）**
你看到 `Proxy.newProxyInstance` 创建动态代理，`@Tool` 注解通过反射转为 OpenAI Function Calling 的 JSON Schema。这一层让你能"模仿"设计。

**第三层：看为什么这么做（Why）**
你理解了为什么要用 Builder 模式而不是构造函数——因为 AI 模型的参数多达 20+ 个，且未来还会增加；理解了为什么 ChatModel 和 EmbeddingModel 要分开抽象——因为它们的输入输出模型完全不同（文本 vs 向量），时序要求也不同（同步 vs 可批量）。这一层让你能"创造"设计。

**多数人停在第一层，优秀的人进入第二层，真正的大师在第三层思考。**

## 六、总结

| 项目 | 学习重点 | 核心收获 |
|------|---------|---------|
| LangChain4j | 注解驱动 AI Service、工具系统、RAG 管道 | Java 元编程 + AI 编排模式 |
| Spring AI | VectorStore 抽象、Advisors 链、自动配置 | Spring 生态的 AI 集成哲学 |
| Jlama | Transformer 推理、Vector API、量化 | LLM 底层原理 + Java 高性能计算 |

读完这三个项目的源码，你不仅学会了用 Java 写 AI 应用，更学会了像顶级框架作者一样思考。

---

**本文是「Java + AI 编程从入门到精通」310 篇系列的第 293 篇。全系列覆盖：**

- 基础篇（1-50）：Java 基础 + AI 概念入门
- 框架篇（51-120）：Spring AI、LangChain4j、Semantic Kernel
- 进阶篇（121-200）：RAG、Agent、多模态、微调
- 实战篇（201-250）：企业级 AI 应用实战
- 延伸篇（251-310）：专项技术突破、性能优化、架构设计

**系列全部文章持续更新中，关注获取最新内容。**
