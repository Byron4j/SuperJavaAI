# 第 11 章 · Spring Boot 集成实战

---

> 本章教你用 Java 调用 Transformer 模型。三类方案：调 API、用 ONNX 本地推理、搭 RAG。

---

## 11.1 方案一：调用 OpenAI 兼容 API

这是最常用的方案——任何兼容 OpenAI 协议的大模型都可以用同一个接口调用。

### 11.1.1 Maven 依赖

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>com.theokanning.openai-gpt3-java</groupId>
        <artifactId>service</artifactId>
        <version>0.18.2</version>
    </dependency>
</dependencies>
```

### 11.1.2 Service 实现

```java
@Service
public class TransformerService {

    private final OpenAiService openAiService;

    // application.yml:
    // openai:
    //   api-key: ${OPENAI_API_KEY}
    //   base-url: https://api.openai.com  (或其他兼容服务)
    //   model: gpt-4
    public TransformerService(
            @Value("${openai.api-key}") String apiKey,
            @Value("${openai.base-url}") String baseUrl,
            @Value("${openai.model}") String model) {
        this.openAiService = new OpenAiService(apiKey, Duration.ofSeconds(120));
        // 注意: 如果用自定义 baseUrl，需要设置
    }

    /**
     * 对话补全——对应 Decoder 的自回归生成过程
     * 
     * @param messages  对话历史
     * @param config    生成配置
     * @return 模型回复
     */
    public ChatMessage chat(List<ChatMessage> messages, GenerationConfig config) {
        ChatCompletionRequest request = ChatCompletionRequest.builder()
            .model(config.model())
            .messages(messages)
            .maxTokens(config.maxTokens())       // Decoder 循环上限
            .temperature(config.temperature())   // Softmax 的温度参数
            .topP(config.topP())                 // Nucleus Sampling
            .frequencyPenalty(config.frequencyPenalty())
            .presencePenalty(config.presencePenalty())
            .build();

        ChatCompletionResult result = openAiService.createChatCompletion(request);
        return result.getChoices().get(0).getMessage();
    }

    /**
     * 获取 Embedding——对应 Encoder 的输出（或 Decoder 的中间表示）
     * 
     * @param text 文本
     * @return Embedding 向量 [d_model]（如 text-embedding-ada-002 是 1536 维）
     */
    public float[] embed(String text) {
        EmbeddingRequest request = EmbeddingRequest.builder()
            .model("text-embedding-ada-002")
            .input(List.of(text))
            .build();

        List<Embedding> result = openAiService.createEmbeddings(request).getData();
        return result.get(0).getEmbedding().stream()
            .map(Double::floatValue)
            .toList()
            .toArray(new float[0]);
    }
}

/**
 * 生成配置 —— 封装所有 Transformer 相关的超参数
 */
public record GenerationConfig(
    String model,           // 模型名 (gpt-4, gpt-3.5-turbo, etc.)
    int maxTokens,          // 最大生成 token 数 = Decoder 循环次数
    double temperature,     // Softmax 温度: 0=确定, 1=标准, >1=随机
    double topP,            // Nucleus Sampling 阈值
    double frequencyPenalty, // 惩罚重复生成的 token
    double presencePenalty   // 惩罚已经出现过的 token
) {
    public static GenerationConfig defaultConfig() {
        return new GenerationConfig(
            "gpt-4", 2048, 0.7, 1.0, 0.0, 0.0
        );
    }
}
```

---

## 11.2 方案二：本地 ONNX 推理

当你不想依赖外部 API（数据隐私、延迟、成本），可以用 ONNX Runtime 在本地跑 Transformer。

### 11.2.1 环境准备

```xml
<dependency>
    <groupId>com.microsoft.onnxruntime</groupId>
    <artifactId>onnxruntime</artifactId>
    <version>1.17.1</version>
</dependency>
```

### 11.2.2 Tokenizer 实现

Transformer 需要把文本转成 token IDs，这里用 Hugging Face 的 Tokenizer JSON 格式：

```java
/**
 * 基于 BPE/WordPiece 的 Tokenizer
 * 本质: 查一个巨大的 Map<String, Integer>
 */
public class BPETokenizer {
    private final Map<String, Integer> tokenToId;
    private final Map<Integer, String> idToToken;
    private final int vocabSize;
    private final int padTokenId = 0;
    private final int bosTokenId = 1;
    private final int eosTokenId = 2;
    private final int unkTokenId = 3;

    /**
     * 把文本转成 token ID 序列
     * 
     * "I love you" → [1, 45, 2073, 2017, 2]
     *                  ↑                ↑
     *                <s>(BOS)        </s>(EOS)
     */
    public int[] encode(String text) {
        // 简化实现：按空格分词 + 查表
        // 实际 BPE 要复杂得多（子词分割）
        String[] words = text.toLowerCase().split("\\s+");
        List<Integer> tokens = new ArrayList<>();
        tokens.add(bosTokenId);

        for (String word : words) {
            Integer id = tokenToId.get(word);
            tokens.add(id != null ? id : unkTokenId);
        }

        tokens.add(eosTokenId);
        return tokens.stream().mapToInt(Integer::intValue).toArray();
    }

    /**
     * 把 token ID 序列转回文本
     */
    public String decode(int[] tokenIds) {
        StringBuilder sb = new StringBuilder();
        for (int id : tokenIds) {
            if (id == eosTokenId || id == padTokenId || id == bosTokenId) continue;
            String token = idToToken.getOrDefault(id, "<unk>");
            sb.append(token).append(" ");
        }
        return sb.toString().trim();
    }
}
```

### 11.2.3 ONNX 推理引擎

```java
/**
 * 用 ONNX Runtime 跑 Transformer 模型
 * 
 * 前提: 已有 Hugging Face 模型通过 optimum 导出为 ONNX
 * 命令: optimum-cli export onnx --model meta-llama/Llama-2-7b onnx_output/
 */
@Service
public class OnnxTransformerEngine {
    private final OrtEnvironment env;
    private final OrtSession session;
    private final BPETokenizer tokenizer;
    private final KVMultiCache cache;  // 参看第 10 章

    public OnnxTransformerEngine(
            @Value("${onnx.model.path}") String modelPath,
            @Value("${onnx.tokenizer.path}") String tokenizerPath) throws OrtException {
        this.env = OrtEnvironment.getEnvironment();
        this.session = env.createSession(modelPath, new OrtSession.SessionOptions());
        this.tokenizer = new BPETokenizer(tokenizerPath);
        this.cache = new KVMultiCache();
    }

    /**
     * 自回归生成
     */
    public String generate(String prompt, int maxNewTokens) {
        int[] inputIds = tokenizer.encode(prompt);
        int[] generated = new int[inputIds.length + maxNewTokens];
        System.arraycopy(inputIds, 0, generated, 0, inputIds.length);

        int generatedLen = inputIds.length;

        for (int i = inputIds.length; i < inputIds.length + maxNewTokens; i++) {
            // 构建输入（带 KV Cache 只需传最后一个 token）
            long[] inputShape = {1, 1};  // [batch=1, seqLen=1]
            OnnxTensor inputTensor = OnnxTensor.createTensor(
                env, new long[][]{{generated[i - 1]}}, inputShape
            );

            Map<String, OnnxTensor> inputs = Map.of("input_ids", inputTensor);
            // 如果 ONNX 模型导出了 past_key_values 输入，也需要传入

            OrtSession.Result result = session.run(inputs);

            // 取 logits [1, 1, vocabSize] → 最后一个 token 的预测
            float[][][] logits = (float[][][]) result.get("logits").get().getValue();
            float[] lastTokenLogits = logits[0][0];

            // 采样
            int nextToken = sample(softmax(lastTokenLogits));
            generated[i] = nextToken;
            generatedLen++;

            if (nextToken == tokenizer.getEosTokenId()) break;
        }

        return tokenizer.decode(Arrays.copyOf(generated, generatedLen));
    }

    private int sample(float[] probs) {
        // Top-K 采样，K=50
        int k = Math.min(50, probs.length);
        // ... 实现见第 9 章
        return topKSample(probs, k);
    }
}
```

---

## 11.3 方案三：搭建 RAG（检索增强生成）

```java
/**
 * RAG: 把知识库切成 chunk → 用 Encoder 做向量化 → 
 *      用户提问时检索最相关的 chunk → 和问题一起喂给大模型（Decoder）
 */
@Service
public class RAGService {
    private final TransformerService llm;      // Decoder (生成)
    private final EmbeddingStore vectorDB;     // 向量数据库
    private final TextSplitter splitter;       // 文本切割

    /**
     * 索引文档
     */
    public void indexDocument(String document) {
        // Step 1: 切割文档（每个 chunk 500 token，重叠 50）
        List<String> chunks = splitter.split(document, 500, 50);

        // Step 2: 对每个 chunk 计算 Embedding（Encoder 的输出）
        for (int i = 0; i < chunks.size(); i++) {
            float[] embedding = llm.embed(chunks.get(i));  // 1536 维向量
            vectorDB.insert(new DocumentChunk(chunks.get(i), embedding, i));
        }
    }

    /**
     * RAG 查询
     */
    public String query(String question) {
        // Step 1: 把问题也做 Embedding
        float[] questionEmbedding = llm.embed(question);

        // Step 2: 在向量数据库中找到最相似的 K 个 chunk
        List<DocumentChunk> relevantChunks = vectorDB.search(questionEmbedding, 5);
        // 相似度: cosine_similarity(q, chunk) = dot(q, chunk) / (|q|·|chunk|)

        // Step 3: 构建 Prompt
        String context = relevantChunks.stream()
            .map(DocumentChunk::text)
            .collect(Collectors.joining("\n---\n"));

        String prompt = """
            根据以下参考信息回答问题。
            如果参考信息不足以回答，请如实告知。

            参考信息:
            %s

            问题: %s

            回答:""".formatted(context, question);

        // Step 4: 调用 LLM 生成回答
        List<ChatMessage> messages = List.of(
            new ChatMessage("system", "你是一个基于参考信息回答问题的助手。"),
            new ChatMessage("user", prompt)
        );

        ChatMessage response = llm.chat(messages, GenerationConfig.defaultConfig());
        return response.getContent();
    }
}
```

---

## 11.4 方案四：用 Spring AI 简化

Spring AI 已经封装了上述所有 API，一行配置即可接入：

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-openai-spring-boot-starter</artifactId>
</dependency>
```

```yaml
spring:
  ai:
    openai:
      api-key: ${OPENAI_API_KEY}
      chat:
        options:
          model: gpt-4
          temperature: 0.7
          max-tokens: 2048
```

```java
@RestController
public class ChatController {

    private final ChatClient chatClient;

    public ChatController(ChatClient.Builder builder) {
        this.chatClient = builder.build();
    }

    @PostMapping("/chat")
    public String chat(@RequestBody String message) {
        return chatClient.prompt()
            .user(message)
            .call()
            .content();
    }

    // Spring AI 已经内置了 RAG 支持！
    @PostMapping("/rag")
    public String rag(@RequestBody String question) {
        return chatClient.prompt()
            .user(question)
            .advisors(new QuestionAnswerAdvisor(vectorStore))  // RAG 只需一行！
            .call()
            .content();
    }
}
```

---

## 11.5 性能对比：云端 vs 本地

| 维度 | OpenAI API | ONNX 本地 |
|---|---|---|
| **延迟** | 1-5 秒（网络 + 推理） | 0.1-10 秒（取决于硬件和模型大小） |
| **吞吐** | 受 API 限流 | 受本地硬件限制 |
| **成本** | 按 token 付费 | 硬件电费（+ 一次性 GPU 投入） |
| **隐私** | 数据传出 | 完全本地 |
| **可控性** | 低（黑盒） | 高（可调所有参数） |
| **模型选择** | 有限 | 任意开源模型 |
| **运维** | 无 | 需要自己维护 |

---

> **下一章**：[Java 类比速查表](12-java-analogies-cheatsheet.md)
