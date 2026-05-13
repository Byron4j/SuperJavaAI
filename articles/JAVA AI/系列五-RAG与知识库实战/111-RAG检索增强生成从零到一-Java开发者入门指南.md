# RAG（检索增强生成）从零到一：Java 开发者入门指南，一个案例跑通RAG全流程

> 本系列文章专注 **Java + AI 工程实践**，我将用真实可运行的代码，系统讲解如何用 Java 构建生产级 AI 应用。如果觉得有帮助，欢迎**点赞、收藏、关注**三连，你的支持是我持续创作的动力！

---

## 一、开篇

RAG 是 2025-2026 年企业 AI 落地的核心模式。如果你问我："公司想做 AI 应用，该从哪入手？"——我的答案永远是 RAG。

这篇文章不堆概念、不讲虚的。我会用一个 **"公司制度问答系统"** 的真实案例，带你从头到尾跑通 RAG 全流程。看完你就能在代码里把 RAG 用起来。

---

## 二、RAG 到底是什么？一个"开卷考试"的类比

### 2.1 传统 LLM 的困境

你问 ChatGPT："我们公司年假怎么算？"

它大概率回答："作为 AI，我无法访问贵公司的内部制度..." —— 因为它没见过你公司的《考勤管理制度.pdf》。

这就是 LLM 的"知识截止日期"和"私有知识盲区"问题。

### 2.2 RAG 的核心思路

```
RAG = 检索（Retrieval） + 增强（Augmented） + 生成（Generation）
```

用"开卷考试"来类比：

| 对比维度 | 闭卷考试（传统LLM） | 开卷考试（RAG） |
|---------|-------------------|----------------|
| 准备阶段 | 靠脑子里记得的 | 先把课本翻一遍，做好索引 |
| 答题阶段 | 凭记忆，可能记错 | 翻书找到相关内容，对着书回答 |
| 优点 | 快 | 准确、有出处、实时 |
| 缺点 | 知识陈旧、会"幻觉" | 多一步检索，稍慢 |

**RAG 就是在 LLM 回答问题之前，先帮它找到参考资料。**

### 2.3 RAG 的完整流程

```
┌──────────────────────────────────────────────────────┐
│                    离线阶段                              │
│                                                        │
│  文档 ──▶ 文本分割 ──▶ Embedding向量化 ──▶ 向量数据库存储  │
│                                                        │
└──────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────┐
│                    在线阶段                              │
│                                                        │
│  用户问题 ──▶ 向量化 ──▶ 向量检索 ──▶ Top-K文档片段      │
│      │                                       │          │
│      └──── 拼接到Prompt ────────────────────┘          │
│                    │                                    │
│                    ▼                                    │
│              LLM 生成回答                                │
└──────────────────────────────────────────────────────┘
```

---

## 三、项目搭建

### 3.1 技术选型

| 组件 | 技术选型 | 说明 |
|------|---------|------|
| 框架 | Spring Boot 3.2+ | 基础框架 |
| AI 框架 | Spring AI 1.0.0-M5 | 统一 AI 调用接口 |
| LLM | OpenAI / 智谱 / Ollama | 大模型 |
| Embedding | OpenAI text-embedding-3-small | 文本向量化 |
| 向量数据库 | Pgvector (PostgreSQL) | 存储和检索向量 |
| ORM | MyBatis-Plus / Spring JDBC | 数据库操作 |
| 构建工具 | Maven | 依赖管理 |

### 3.2 Maven 依赖

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.3.0</version>
</parent>

<properties>
    <java.version>17</java.version>
    <spring-ai.version>1.0.0-M5</spring-ai.version>
</properties>

<dependencies>
    <!-- Spring Boot Web -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <!-- Spring AI OpenAI -->
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-openai-spring-boot-starter</artifactId>
        <version>${spring-ai.version}</version>
    </dependency>

    <!-- Spring AI Pgvector -->
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-pgvector-store</artifactId>
        <version>${spring-ai.version}</version>
    </dependency>

    <!-- PostgreSQL -->
    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
    </dependency>

    <!-- Spring JDBC -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-jdbc</artifactId>
    </dependency>

    <!-- Lombok -->
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <optional>true</optional>
    </dependency>

    <!-- PDF 解析 -->
    <dependency>
        <groupId>org.apache.pdfbox</groupId>
        <artifactId>pdfbox</artifactId>
        <version>3.0.1</version>
    </dependency>

    <!-- Markdown 解析 -->
    <dependency>
        <groupId>com.vladsch.flexmark</groupId>
        <artifactId>flexmark-all</artifactId>
        <version>0.64.8</version>
    </dependency>
</dependencies>

<repositories>
    <repository>
        <id>spring-milestones</id>
        <name>Spring Milestones</name>
        <url>https://repo.spring.io/milestone</url>
        <snapshots><enabled>false</enabled></snapshots>
    </repository>
</repositories>
```

### 3.3 配置文件

```yaml
spring:
  application:
    name: rag-demo

  # PostgreSQL + Pgvector
  datasource:
    url: jdbc:postgresql://localhost:5432/rag_demo
    username: postgres
    password: postgres
    driver-class-name: org.postgresql.Driver

  # Spring AI OpenAI 配置
  ai:
    openai:
      api-key: ${OPENAI_API_KEY:your-api-key}
      base-url: https://api.openai.com
      chat:
        enabled: true
        options:
          model: gpt-4o-mini
          temperature: 0.3
      embedding:
        enabled: true
        options:
          model: text-embedding-3-small

    # Pgvector 配置
    vectorstore:
      pgvector:
        host: localhost
        port: 5432
        database: rag_demo
        username: postgres
        password: postgres
        table-name: document_embeddings
        index-type: HNSW
        dimensions: 1536
```

---

## 四、文档处理：从 PDF 到文本块

### 4.1 文档读取与分割

```java
import org.springframework.ai.document.Document;
import org.springframework.ai.reader.ExtractedTextFormatter;
import org.springframework.ai.reader.pdf.PagePdfDocumentReader;
import org.springframework.ai.reader.pdf.config.PdfDocumentReaderConfig;
import org.springframework.ai.transformer.splitter.TokenTextSplitter;
import org.springframework.core.io.ClassPathResource;
import org.springframework.stereotype.Component;

import java.io.File;
import java.util.List;

@Component
public class DocumentProcessor {

    /**
     * 读取PDF文档并分割为文本块
     * @param filePath PDF文件路径
     * @param chunkSize 每个文本块的最大Token数
     * @param overlap 相邻文本块的重叠Token数
     * @return 分割后的文档列表
     */
    public List<Document> processPdf(String filePath, int chunkSize, int overlap) {
        // 1. PDF 读取配置
        PdfDocumentReaderConfig config = PdfDocumentReaderConfig.builder()
            .withPageExtractedTextFormatter(
                ExtractedTextFormatter.builder()
                    .withNumberOfBottomTextLinesToDelete(0)
                    .withNumberOfTopTextLinesToDelete(0)
                    .build())
            .build();

        // 2. 使用 PagePdfDocumentReader 读取
        PagePdfDocumentReader reader = new PagePdfDocumentReader(
            new ClassPathResource(filePath), config);

        // 3. 读取所有页面
        List<Document> pages = reader.get();

        // 4. 使用 TokenTextSplitter 分割为小块
        TokenTextSplitter splitter = new TokenTextSplitter(
            chunkSize,  // 默认800
            overlap,    // 默认100，重叠防止语义断裂
            5,          // 最小块大小
            10000,      // 最大块大小
            true        // 保留块内完整性
        );

        List<Document> chunks = splitter.apply(pages);

        System.out.printf("文档处理完成: 文件=%s, 总页数=%d, 总块数=%d%n",
            filePath, pages.size(), chunks.size());

        return chunks;
    }

    /**
     * 处理纯文本文件
     */
    public List<Document> processText(String content, int chunkSize, int overlap) {
        TokenTextSplitter splitter = new TokenTextSplitter(
            chunkSize, overlap, 5, 10000, true);
        return splitter.apply(List.of(new Document(content)));
    }
}
```

### 4.2 给文档块添加元数据

```java
public List<Document> processPdfWithMetadata(String filePath, int chunkSize, int overlap) {
    List<Document> chunks = processPdf(filePath, chunkSize, overlap);

    // 给每个块添加元数据
    String fileName = filePath.substring(filePath.lastIndexOf('/') + 1);
    for (int i = 0; i < chunks.size(); i++) {
        Document chunk = chunks.get(i);
        Map<String, Object> metadata = new HashMap<>(chunk.getMetadata());
        metadata.put("source_file", fileName);
        metadata.put("chunk_index", i);
        metadata.put("chunk_total", chunks.size());
        metadata.put("processed_at", Instant.now().toString());

        // Spring AI 3.x: 使用 builder 重建
        chunk = Document.builder()
            .withContent(chunk.getContent())
            .withMetadata(metadata)
            .build();
        chunks.set(i, chunk);
    }
    return chunks;
}
```

---

## 五、向量化与存储：Pgvector

### 5.1 Pgvector 环境准备

```sql
-- 安装 pgvector 扩展
CREATE EXTENSION IF NOT EXISTS vector;

-- 创建文档块表
CREATE TABLE IF NOT EXISTS document_embeddings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    content TEXT NOT NULL,
    metadata JSONB,
    embedding vector(1536)  -- OpenAI embedding维度
);

-- 创建 HNSW 索引（加速向量检索）
CREATE INDEX ON document_embeddings 
    USING hnsw (embedding vector_cosine_ops)
    WITH (m = 16, ef_construction = 200);
```

### 5.2 向量化并写入 Pgvector

```java
import org.springframework.ai.embedding.EmbeddingClient;
import org.springframework.ai.embedding.EmbeddingRequest;
import org.springframework.ai.vectorstore.PgVectorStore;
import org.springframework.ai.vectorstore.VectorStore;
import org.springframework.stereotype.Service;

import java.util.List;

@Service
public class EmbeddingService {

    private final EmbeddingClient embeddingClient;
    private final PgVectorStore vectorStore;

    public EmbeddingService(EmbeddingClient embeddingClient, 
                            PgVectorStore vectorStore) {
        this.embeddingClient = embeddingClient;
        this.vectorStore = vectorStore;
    }

    /**
     * 向量化文档块并存储到 Pgvector
     */
    public void embedAndStore(List<Document> documents) {
        // 批量向量化
        List<String> contents = documents.stream()
            .map(Document::getContent)
            .toList();

        // Spring AI 自动处理向量化和存储
        List<org.springframework.ai.document.Document> aiDocs = documents.stream()
            .map(d -> new org.springframework.ai.document.Document(
                d.getContent(), d.getMetadata()))
            .toList();

        // 写入向量数据库
        vectorStore.add(aiDocs);

        System.out.printf("向量化存储完成: 文档块数=%d%n", documents.size());
    }

    /**
     * 检索与问题最相关的Top-K个文档块
     */
    public List<Document> search(String query, int topK) {
        // Spring AI 自动处理查询向量化和相似度搜索
        List<org.springframework.ai.document.Document> results =
            vectorStore.similaritySearch(
                org.springframework.ai.vectorstore.SearchRequest
                    .query(query)
                    .withTopK(topK)
                    .withSimilarityThreshold(0.7)
            );

        // 转换为我们的 Document 对象
        return results.stream()
            .map(d -> new Document(d.getContent(), d.getMetadata()))
            .collect(Collectors.toList());
    }
}
```

---

## 六、构建 RAG 问答引擎

```java
import org.springframework.ai.chat.ChatClient;
import org.springframework.ai.chat.ChatResponse;
import org.springframework.ai.chat.messages.Message;
import org.springframework.ai.chat.messages.UserMessage;
import org.springframework.ai.chat.prompt.Prompt;
import org.springframework.ai.chat.prompt.PromptTemplate;
import org.springframework.stereotype.Service;

import java.util.List;
import java.util.Map;
import java.util.stream.Collectors;

@Service
public class RagService {

    private final ChatClient chatClient;
    private final EmbeddingService embeddingService;
    private final DocumentProcessor documentProcessor;

    public RagService(ChatClient chatClient, 
                      EmbeddingService embeddingService,
                      DocumentProcessor documentProcessor) {
        this.chatClient = chatClient;
        this.embeddingService = embeddingService;
        this.documentProcessor = documentProcessor;
    }

    /**
     * 文档导入：处理 → 向量化 → 存储
     */
    public void importDocument(String filePath) {
        List<Document> chunks = documentProcessor.processPdfWithMetadata(filePath, 800, 100);
        embeddingService.embedAndStore(chunks);
    }

    /**
     * RAG 问答
     */
    public String ask(String question) {
        // 1. 检索相关文档
        List<Document> relevantDocs = embeddingService.search(question, 5);

        // 2. 拼接上下文
        String context = buildContext(relevantDocs);

        // 3. 构建 Prompt
        String prompt = buildPrompt(question, context);

        // 4. 调用 LLM 生成回答
        ChatResponse response = chatClient.call(new Prompt(List.of(
            new UserMessage(prompt)
        )));

        return response.getResult().getOutput().getContent();
    }

    /**
     * 构建上下文
     */
    private String buildContext(List<Document> documents) {
        StringBuilder sb = new StringBuilder();
        for (int i = 0; i < documents.size(); i++) {
            Document doc = documents.get(i);
            sb.append(String.format("【参考资料%d】%s\n\n", i + 1, doc.getContent()));
        }
        return sb.toString();
    }

    /**
     * 构建带上下文的 Prompt
     */
    private String buildPrompt(String question, String context) {
        return String.format("""
            你是一个公司制度问答助手。请严格根据以下参考资料回答用户问题。
            
            如果参考资料中不包含相关信息，请明确告知用户"这个问题在现有制度中没有找到相关规定"，
            不要编造任何信息。
            
            【参考资料】
            %s
            
            【用户问题】
            %s
            
            【回答要求】
            1. 在回答末尾注明引用的参考资料来源
            2. 回答要简洁、准确、有条理
            3. 如果制度中有多条相关规定，请列出所有相关条文
            """, context, question);
    }
}
```

---

## 七、效果演示

### 7.1 准备测试数据

假设我们有 `公司考勤管理制度.pdf`，内容包含：

```
第三章 年假管理
第12条：员工入职满1年后，享有5天带薪年假。
第13条：员工入职满10年后，享有10天带薪年假。
第14条：员工入职满20年后，享有15天带薪年假。
第15条：年假需提前3个工作日申请，经部门负责人审批。

第四章 加班管理
第20条：工作日加班按1.5倍工资计算。
第21条：休息日加班按2倍工资计算。
第22条：法定节假日加班按3倍工资计算。
```

### 7.2 控制器

```java
@RestController
@RequestMapping("/api/rag")
public class RagController {

    @Autowired
    private RagService ragService;

    @PostMapping("/import")
    public ResponseEntity<String> importDocument(@RequestParam String filePath) {
        ragService.importDocument(filePath);
        return ResponseEntity.ok("文档导入成功: " + filePath);
    }

    @PostMapping("/ask")
    public ResponseEntity<Map<String, Object>> ask(@RequestParam String question) {
        String answer = ragService.ask(question);
        
        Map<String, Object> result = new HashMap<>();
        result.put("question", question);
        result.put("answer", answer);
        result.put("timestamp", Instant.now());
        
        return ResponseEntity.ok(result);
    }
}
```

### 7.3 效果测试

```
# 启动后测试
curl -X POST "http://localhost:8080/api/rag/import?filePath=data/考勤管理制度.pdf"
# 输出：文档导入成功

curl -X POST "http://localhost:8080/api/rag/ask?question=入职5年有多少天年假？"

# LLM回答：
# 根据《公司考勤管理制度》第12条规定：员工入职满1年后，享有5天带薪年假。
# 入职5年属于"入职满1年但未满10年"的范畴，因此享有5天带薪年假。
# 【参考来源：《公司考勤管理制度》第三章第12条】
```

### 7.4 对比：不带RAG vs 带RAG

```
不带RAG（直接问LLM）：
Q: 入职5年有多少天年假？
A: 我无法访问贵公司的内部制度，无法回答此问题。

带RAG：
Q: 入职5年有多少天年假？
A: 根据《公司考勤管理制度》第三章第12条规定，入职满1年后享有5天年假。
   入职5年属于"满1年但未满10年"范围，因此享有5天带薪年假。
   【参考来源：《公司考勤管理制度》第三章第12条】
```

**差距一目了然。**

---

## 八、RAG 效果优化的 3 个进阶技巧

### 8.1 技巧一：混合检索（Hybrid Search）

纯向量检索的局限：长尾词、专有名词等稀疏特征容易被忽略。

```java
@Service
public class HybridSearchService {

    @Autowired
    private PgVectorStore vectorStore;

    @Autowired
    private JdbcTemplate jdbcTemplate;

    /**
     * 混合检索 = 向量检索 + 关键词检索
     */
    public List<Document> hybridSearch(String query, int topK) {
        // 1. 向量检索
        List<Document> vectorResults = vectorStore.similaritySearch(
            SearchRequest.query(query).withTopK(topK * 2));

        // 2. 关键词检索（使用 PostgreSQL 全文搜索）
        String sql = """
            SELECT content, metadata, 
                   ts_rank(to_tsvector('chinese', content), 
                           plainto_tsquery('chinese', ?)) AS rank
            FROM document_embeddings
            WHERE to_tsvector('chinese', content) @@ plainto_tsquery('chinese', ?)
            ORDER BY rank DESC
            LIMIT ?
            """;
        List<Document> keywordResults = jdbcTemplate.query(sql, 
            (rs, rowNum) -> new Document(rs.getString("content"), /* metadata */ null),
            query, query, topK * 2);

        // 3. RRF（Reciprocal Rank Fusion）融合
        return rrfFusion(vectorResults, keywordResults, topK);
    }

    private List<Document> rrfFusion(List<Document> vectorDocs, 
                                      List<Document> keywordDocs, int topK) {
        Map<String, Double> scores = new HashMap<>();

        for (int i = 0; i < vectorDocs.size(); i++) {
            String key = vectorDocs.get(i).getContent().substring(0, Math.min(50, 
                vectorDocs.get(i).getContent().length()));
            scores.merge(key, 1.0 / (60 + i + 1), Double::sum);
        }
        for (int i = 0; i < keywordDocs.size(); i++) {
            String key = keywordDocs.get(i).getContent().substring(0, Math.min(50, 
                keywordDocs.get(i).getContent().length()));
            scores.merge(key, 1.0 / (60 + i + 1), Double::sum);
        }

        // 按分数排序，返回 TopK
        return scores.entrySet().stream()
            .sorted(Map.Entry.<String, Double>comparingByValue().reversed())
            .limit(topK)
            .map(e -> new Document(e.getKey(), null))
            .collect(Collectors.toList());
    }
}
```

### 8.2 技巧二：重排序（Re-ranker）

```java
/**
 * 利用 LLM 对检索结果进行重排序
 */
public List<Document> rerank(String question, List<Document> candidates, int topK) {
    // 使用 LLM 评估每个候选文档与问题的相关性
    PromptTemplate template = new PromptTemplate("""
        请评估以下文档与用户问题的相关性，给出0-10的分数。
        
        用户问题：{question}
        文档内容：{document}
        
        只返回分数数字，不要解释：
        """);

    return candidates.stream()
        .map(doc -> {
            Prompt prompt = template.create(Map.of(
                "question", question,
                "document", doc.getContent().substring(0, Math.min(2000, 
                    doc.getContent().length()))
            ));
            String scoreStr = chatClient.call(prompt).getResult().getOutput().getContent();
            double score = Double.parseDouble(scoreStr.trim());
            return Map.entry(doc, score);
        })
        .sorted(Map.Entry.<Document, Double>comparingByValue().reversed())
        .limit(topK)
        .map(Map.Entry::getKey)
        .collect(Collectors.toList());
}
```

### 8.3 技巧三：文档分块策略调优

```java
public enum ChunkStrategy {
    /**
     * 固定大小分块：简单，但可能截断句子
     */
    FIXED_SIZE(800, 100),

    /**
     * 语义分块：按段落/标题分割，保留语义完整性
     */
    SEMANTIC(1200, 200),

    /**
     * 问题导向分块：提前生成QA对，按问题索引
     */
    QA_BASED(300, 50);

    final int chunkSize;
    final int overlap;

    ChunkStrategy(int chunkSize, int overlap) {
        this.chunkSize = chunkSize;
        this.overlap = overlap;
    }
}
```

---

## 九、RAG 效果评估

```java
@Component
public class RagEvaluator {

    /**
     * 自动评估 RAG 回答质量
     */
    public EvaluationResult evaluate(RagService ragService, 
                                      List<QAPair> testCases) {
        int total = testCases.size();
        int answerCorrect = 0;      // 答案正确
        int grounded = 0;           // 有引用来源
        int hallucination = 0;      // 幻觉（编造信息）

        for (QAPair pair : testCases) {
            String answer = ragService.ask(pair.question());

            // 检查是否有引用来源
            if (answer.contains("参考来源") || answer.contains("根据")) {
                grounded++;
            }

            // 检查是否包含预期答案
            if (pair.expectedKeywords().stream().anyMatch(answer::contains)) {
                answerCorrect++;
            }

            // 检查幻觉（简单判断：是否有"没有找到""无法"等）
            if (answer.contains("没有找到") && !pair.isUnanswerable()) {
                hallucination++;
            }
        }

        return new EvaluationResult(
            (double) answerCorrect / total,
            (double) grounded / total,
            (double) hallucination / total
        );
    }

    record QAPair(String question, List<String> expectedKeywords, 
                  boolean isUnanswerable) {}
    record EvaluationResult(double accuracy, double groundedRate, 
                            double hallucinationRate) {}
}
```

---

## 十、总结

这篇文章带你从零开始，完整搭建了一个基于 **Spring Boot + Spring AI + Pgvector** 的 RAG 系统。

回顾核心流程：
```
文档(PDF) → 文本分割 → Embedding向量化 → Pgvector存储
                                                    ↓
用户问题 → 向量化 → 相似度检索 → Top-K文档 → Prompt拼接 → LLM生成回答
```

三个关键点请记住：
1. **分块策略决定了检索效果的下限**（太小丢语义，太大丢精度）
2. **Embedding 模型的质量决定了检索效果的上限**（下一篇详细讲）
3. **Prompt 模板设计决定了生成回答的质量**（明确要求"引用来源"可大幅降低幻觉）

---

**下一篇预告**：RAG 效果好坏，60% 取决于 Embedding 模型的选择。下一篇《Embedding 模型选型指南：OpenAI / 智谱 / 本地模型的效果与成本对比》，我会给出主流模型的实测数据对比表，帮你选对最适合的 Embedding 模型。敬请期待！

---

> 如果觉得这篇文章有帮助，欢迎点赞、收藏、关注，感谢支持！

> 作者：深耕 Java 企业级开发多年，专注 AI 工程化落地。有问题欢迎在评论区交流。
