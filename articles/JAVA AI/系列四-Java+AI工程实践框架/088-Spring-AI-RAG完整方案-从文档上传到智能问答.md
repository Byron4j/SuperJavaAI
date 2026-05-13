# Spring AI RAG 完整方案：从文档上传到智能问答，一天搭建企业级知识库

> 很多RAG教程只讲原理不讲代码，本文给你一个完整的、可直接运行的Spring AI RAG项目。

---

## 一、RAG 到底是什么

**RAG（Retrieval-Augmented Generation，检索增强生成）** 的核心思想很简单：

```
用户提问 → 从知识库检索相关文档 → 将文档作为上下文注入Prompt → LLM生成回答
```

传统LLM的局限：
- 训练数据有截止日期，不知道最新信息
- 不知道你们公司的内部文档
- 容易"幻觉"——瞎编不存在的内容

RAG解决的就是这个问题——**让LLM基于你的真实文档来回答**。

---

## 二、项目依赖

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-tika-document-reader</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-pgvector-store</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-openai-spring-boot-starter</artifactId>
</dependency>
```

本文使用 **OpenAI Embedding + PostgreSQL pgvector** 作为向量存储，你可以替换为其他Embedding模型和向量数据库。

---

## 三、步骤一：文档读取（PDF/Word/TXT）

### 3.1 支持的文档格式

Spring AI 基于 Apache Tika，支持几乎所有常见格式：

| 格式 | Reader类 | 说明 |
|------|----------|------|
| PDF | `TikaDocumentReader` | 自动提取文本 |
| Word (.docx) | `TikaDocumentReader` | 含表格和图片描述 |
| TXT/Markdown | `TikaDocumentReader` | 纯文本 |
| Excel (.xlsx) | `TikaDocumentReader` | 表格数据 |
| HTML | `TikaDocumentReader` | 网页内容 |
| PPT | `TikaDocumentReader` | 演示文稿 |

### 3.2 读取单个文档

```java
@Service
public class DocumentReaderService {

    public List<Document> readDocument(MultipartFile file) {
        try {
            // Spring AI的TikaDocumentReader自动识别格式
            TikaDocumentReader reader = new TikaDocumentReader(
                new InputStreamResource(file.getInputStream()));
            
            List<Document> documents = reader.get();
            
            log.info("读取文档: {}, 内容长度: {}字符",
                file.getOriginalFilename(),
                documents.stream()
                    .mapToInt(d -> d.getContent().length())
                    .sum());
            
            return documents;
        } catch (IOException e) {
            throw new RuntimeException("文档读取失败", e);
        }
    }
}
```

### 3.3 读取整个目录

```java
@Service
public class BatchDocumentReader {

    /**
     * 批量读取指定目录下的所有文档
     */
    public List<Document> readDirectory(String directoryPath) {
        try {
            List<Document> allDocs = new ArrayList<>();
            
            Path dir = Paths.get(directoryPath);
            try (DirectoryStream<Path> stream = 
                    Files.newDirectoryStream(dir, "*.*")) {
                for (Path filePath : stream) {
                    // 跳过隐藏文件和临时文件
                    if (filePath.getFileName().toString().startsWith(".")) {
                        continue;
                    }
                    
                    FileSystemResource resource = 
                        new FileSystemResource(filePath);
                    TikaDocumentReader reader = 
                        new TikaDocumentReader(resource);

                    List<Document> docs = reader.get();
                    
                    // 注入元数据：文件名和路径
                    docs.forEach(doc -> {
                        doc.getMetadata().put("source", 
                            filePath.getFileName().toString());
                        doc.getMetadata().put("path", 
                            filePath.toString());
                        doc.getMetadata().put("loaded_at", 
                            LocalDateTime.now().toString());
                    });
                    
                    allDocs.addAll(docs);
                    log.info("已读取: {}", filePath.getFileName());
                }
            }
            
            log.info("共读取 {} 个文档片段", allDocs.size());
            return allDocs;
        } catch (IOException e) {
            throw new RuntimeException("批量读取文档失败", e);
        }
    }
}
```

### 3.4 带元数据过滤的文档读取

```java
public List<Document> readWithMetadata(
        MultipartFile file, 
        Map<String, String> customMetadata) {
    
    TikaDocumentReader reader = new TikaDocumentReader(
        new InputStreamResource(file.getInputStream()));
    
    List<Document> documents = reader.get();
    
    // 为每个文档片段注入自定义元数据
    documents.forEach(doc -> {
        doc.getMetadata().putAll(customMetadata);
        doc.getMetadata().put("filename", 
            file.getOriginalFilename());
        doc.getMetadata().put("file_size", 
            String.valueOf(file.getSize()));
        doc.getMetadata().put("content_type", 
            file.getContentType());
    });
    
    return documents;
}
```

---

## 四、步骤二：文档切割（TokenTextSplitter）

文档太长不能直接Embedding（Embedding模型有token上限），需要切割成合适大小的片段。

### 4.1 TokenTextSplitter 配置

```java
@Configuration
public class TextSplitterConfig {

    @Bean
    public TokenTextSplitter tokenTextSplitter() {
        return new TokenTextSplitter(
            500,    // defaultChunkSize：每个片段默认500个token
            100,    // minChunkSizeChars：最小片段字符数
            50,     // minChunkLengthToEmbed：最小片段token数
            50,     // maxNumChunks：单次最大片段数（0表示不限制）
            true    // keepSeparator：保留分隔符
        );
    }
}
```

### 4.2 带重叠的切割器（推荐）

```java
@Configuration
public class TextSplitterConfig {

    @Bean
    public TokenTextSplitter defaultSplitter() {
        return TokenTextSplitter.builder()
            .withChunkSize(800)           // 每个片段800 token
            .withMinChunkSizeChars(350)    // 最小350字符
            .withMinChunkLengthToEmbed(100) // 至少100token才Embedding
            .withMaxNumChunks(10000)       // 最大10000个片段
            .withKeepSeparator(true)       // 保留段落分隔符
            .build();
    }

    /**
     * 创建带重叠的切分器——适合法律/合同类长文档
     * 重叠可以避免关键信息恰好在片段边界上被切断
     */
    @Bean("overlapSplitter")
    public TokenTextSplitter overlapSplitter() {
        return TokenTextSplitter.builder()
            .withChunkSize(500)
            .withChunkOverlap(100)     // 相邻片段重叠100个token
            .withMinChunkSizeChars(200)
            .withMinChunkLengthToEmbed(50)
            .withMaxNumChunks(10000)
            .withKeepSeparator(true)
            .build();
    }
}
```

### 4.3 按文档类型选择切割策略

```java
@Service
public class SmartDocumentSplitter {

    private final TokenTextSplitter defaultSplitter;
    private final TokenTextSplitter overlapSplitter;

    public List<Document> split(List<Document> documents, 
                                 String contentType) {
        // 合同、法律文档用重叠切割
        if (contentType.contains("pdf") || 
            contentType.contains("application")) {
            return overlapSplitter.apply(documents);
        }
        // 短文本直接用默认切割
        return defaultSplitter.apply(documents);
    }
}
```

---

## 五、步骤三：Embedding 生成 + 存储到VectorDB

### 5.1 配置 pgvector

```yaml
spring:
  ai:
    openai:
      api-key: ${OPENAI_API_KEY}
      embedding:
        options:
          model: text-embedding-3-small  # text-embedding-3-large 效果更好
    vectorstore:
      pgvector:
        host: localhost
        port: 5432
        database: rag_knowledge_base
        username: ${DB_USERNAME}
        password: ${DB_PASSWORD}
        table-name: document_embeddings
        dimensions: 1536  # text-embedding-3-small的向量维度
        index-type: hnsw  # 高性能近似最近邻索引
```

### 5.2 数据库初始化

```sql
-- pgvector扩展
CREATE EXTENSION IF NOT EXISTS vector;

-- 文档向量存储表
CREATE TABLE IF NOT EXISTS document_embeddings (
    id          VARCHAR(255) PRIMARY KEY,
    content     TEXT,
    metadata    JSONB,
    embedding   VECTOR(1536)
);

-- HNSW索引（检索速度比IVFFlat快很多）
CREATE INDEX IF NOT EXISTS idx_document_embeddings_embedding 
    ON document_embeddings 
    USING hnsw (embedding vector_cosine_ops);
```

### 5.3 完整的数据入库管道

```java
@Service
public class DocumentIngestionPipeline {

    private final VectorStore vectorStore;
    private final TokenTextSplitter textSplitter;
    private final DocumentReaderService readerService;

    public DocumentIngestionPipeline(VectorStore vectorStore,
                                      TokenTextSplitter textSplitter,
                                      DocumentReaderService readerService) {
        this.vectorStore = vectorStore;
        this.textSplitter = textSplitter;
        this.readerService = readerService;
    }

    /**
     * 完整流程：上传 → 读取 → 切割 → 向量化 → 存储
     */
    public IngestionResult ingestDocument(
            MultipartFile file, 
            Map<String, String> metadata) {
        
        log.info("开始处理文档: {}", file.getOriginalFilename());
        long start = System.currentTimeMillis();

        // Step 1: 读取文档
        List<Document> rawDocs = readerService.readDocument(file);

        // Step 2: 注入自定义元数据
        rawDocs.forEach(doc -> {
            doc.getMetadata().putAll(metadata);
            doc.getMetadata().put("source_file", 
                file.getOriginalFilename());
            doc.getMetadata().put("ingested_at", 
                LocalDateTime.now().toString());
        });

        // Step 3: 切割文档
        List<Document> chunks = textSplitter.apply(rawDocs);
        log.info("文档切割完成: {} → {} 个片段", 
                 rawDocs.size(), chunks.size());

        // Step 4: 向量化并存储到 pgvector
        vectorStore.add(chunks);

        long elapsed = System.currentTimeMillis() - start;
        log.info("文档入库完成: {}, 耗时: {}ms", 
                 file.getOriginalFilename(), elapsed);

        return new IngestionResult(
            file.getOriginalFilename(), 
            chunks.size(), 
            elapsed);
    }

    public record IngestionResult(
        String fileName, int chunkCount, long elapsedMs) {}
}
```

### 5.4 批量导入API

```java
@RestController
@RequestMapping("/api/rag")
public class DocumentIngestionController {

    private final DocumentIngestionPipeline pipeline;

    @PostMapping("/documents/upload")
    public ResponseEntity<IngestionResult> uploadDocument(
            @RequestParam("file") MultipartFile file,
            @RequestParam(value = "category", 
                          defaultValue = "general") String category,
            @RequestParam(value = "department", 
                          required = false) String department) {
        
        Map<String, String> metadata = new HashMap<>();
        metadata.put("category", category);
        metadata.put("department", 
            department != null ? department : "unknown");
        
        IngestionResult result = pipeline.ingestDocument(file, metadata);
        return ResponseEntity.ok(result);
    }

    @PostMapping("/documents/upload-batch")
    public ResponseEntity<List<IngestionResult>> uploadBatch(
            @RequestParam("files") MultipartFile[] files,
            @RequestParam("category") String category) {
        
        Map<String, String> metadata = Map.of("category", category);
        
        List<IngestionResult> results = Arrays.stream(files)
            .map(file -> pipeline.ingestDocument(file, metadata))
            .toList();
        
        return ResponseEntity.ok(results);
    }
}
```

---

## 六、步骤四：检索增强问答

### 6.1 基础检索问答

```java
@Service
public class RagQuestionService {

    private final ChatClient chatClient;
    private final VectorStore vectorStore;

    public RagQuestionService(ChatClient.Builder builder,
                               VectorStore vectorStore) {
        this.chatClient = builder.build();
        this.vectorStore = vectorStore;
    }

    public String ask(String question) {
        // 1. 用户问题转向量 → 检索相似文档
        List<Document> relevantDocs = vectorStore.similaritySearch(
            SearchRequest.query(question)
                .withTopK(5)           // 取前5个最相关的片段
                .withSimilarityThreshold(0.7)  // 相似度阈值
        );

        if (relevantDocs.isEmpty()) {
            return "抱歉，知识库中没有找到相关信息。";
        }

        // 2. 拼接上下文
        String context = relevantDocs.stream()
            .map(doc -> {
                String source = doc.getMetadata().getOrDefault(
                    "source_file", "unknown");
                return "[来源: " + source + "]\n" + doc.getContent();
            })
            .collect(Collectors.joining("\n\n---\n\n"));

        // 3. 构建System Prompt + 检索到的上下文
        String systemPrompt = """
            你是企业的知识库助手。请根据以下参考文档回答用户问题。
            
            ## 回答规则
            1. 只使用参考文档中的信息回答
            2. 如果参考文档中没有相关信息，请明确告知用户"当前知识库未收录相关信息"
            3. 回答时注明信息来源
            4. 保持专业、准确的语气
            
            ## 参考文档
            %s
            """.formatted(context);

        // 4. 调用LLM生成回答
        return chatClient.prompt()
            .system(systemPrompt)
            .user(question)
            .call()
            .content();
    }
}
```

### 6.2 使用 RetrievalAugmentationAdvisor（更简洁）

```java
@Service
public class AdvancedRagService {

    private final ChatClient chatClient;
    private final VectorStore vectorStore;

    public String askWithAdvisor(String question) {
        // Spring AI提供的检索增强顾问——自动处理检索和上下文注入
        RetrievalAugmentationAdvisor advisor = 
            RetrievalAugmentationAdvisor.builder()
                .vectorStore(vectorStore)
                .topK(5)
                .similarityThreshold(0.7)
                .build();

        return chatClient.prompt()
            .user(question)
            .advisors(advisor)
            .call()
            .content();
    }
}
```

### 6.3 完整问答API

```java
@RestController
@RequestMapping("/api/rag")
public class RagChatController {

    private final AdvancedRagService ragService;

    @PostMapping("/chat")
    public ResponseEntity<Map<String, Object>> chat(
            @RequestBody ChatRequest request) {
        
        String answer = ragService.askWithAdvisor(request.question());
        
        Map<String, Object> response = Map.of(
            "question", request.question(),
            "answer", answer,
            "timestamp", LocalDateTime.now().toString()
        );
        
        return ResponseEntity.ok(response);
    }

    public record ChatRequest(String question) {}
}
```

---

## 七、高级特性：元数据过滤

### 7.1 按部门/类别过滤检索

```java
@Service
public class FilteredRagService {

    private final VectorStore vectorStore;

    /**
     * 只在指定部门的文档中检索
     */
    public List<Document> searchByDepartment(
            String question, String department) {
        
        return vectorStore.similaritySearch(
            SearchRequest.query(question)
                .withTopK(5)
                .withSimilarityThreshold(0.7)
                .withFilterExpression(
                    "department == '" + department + "'")
        );
    }

    /**
     * 多维度过滤：类别 + 时间范围
     */
    public List<Document> searchWithCompositeFilter(
            String question, 
            String category, 
            LocalDateTime after) {
        
        String filterExpr = String.format(
            "category == '%s' && ingested_at >= '%s'",
            category, after.toString());

        return vectorStore.similaritySearch(
            SearchRequest.query(question)
                .withTopK(10)
                .withSimilarityThreshold(0.65)
                .withFilterExpression(filterExpr));
    }

    /**
     * 排除特定来源的文档
     */
    public List<Document> searchExcludingSource(
            String question, String excludeSource) {
        
        return vectorStore.similaritySearch(
            SearchRequest.query(question)
                .withTopK(5)
                .withFilterExpression(
                    "source_file != '" + excludeSource + "'"));
    }
}
```

### 7.2 带过滤的问答

```java
public String askWithFilter(String question, 
                             String department, 
                             String category) {
    // 仅检索指定部门和类别的文档
    String filterExpr = String.format(
        "department == '%s' && category == '%s'", 
        department, category);

    List<Document> docs = vectorStore.similaritySearch(
        SearchRequest.query(question)
            .withTopK(5)
            .withSimilarityThreshold(0.7)
            .withFilterExpression(filterExpr));

    if (docs.isEmpty()) {
        return "在" + department + "部门的" + category + 
               "分类中没有找到相关信息。";
    }

    // 构建回答...（同上）
    return buildAnswer(question, docs);
}
```

---

## 八、高级特性：相似度阈值调优

### 8.1 相似度阈值策略

```java
public enum SimilarityStrategy {
    STRICT(0.85, "严格匹配——只返回高度相关的文档"),
    STANDARD(0.70, "标准匹配——平衡准确率和召回率"),
    RELAXED(0.55, "宽松匹配——尽可能多地召回文档"),
    ADAPTIVE(0.0, "自适应——根据文档数量动态调整");

    private final double defaultThreshold;
    private final String description;

    SimilarityStrategy(double t, String d) {
        this.defaultThreshold = t;
        this.description = d;
    }
}
```

### 8.2 自适应阈值实现

```java
@Service
public class AdaptiveRagService {

    private final VectorStore vectorStore;

    public List<Document> adaptiveSearch(String question) {
        // 先用宽松策略检索
        List<Document> initial = vectorStore.similaritySearch(
            SearchRequest.query(question)
                .withTopK(20)
                .withSimilarityThreshold(0.5));

        if (initial.size() >= 10) {
            // 文档较多，提高阈值，取最相关的
            return initial.stream()
                .filter(doc -> {
                    Double score = (Double) doc.getMetadata()
                        .get("similarity_score");
                    return score != null && score >= 0.75;
                })
                .limit(5)
                .toList();
        } else if (initial.size() >= 3) {
            // 文档适中，直接使用
            return initial.stream().limit(5).toList();
        } else {
            // 文档太少，降低阈值再试一次
            return vectorStore.similaritySearch(
                SearchRequest.query(question)
                    .withTopK(5)
                    .withSimilarityThreshold(0.3));
        }
    }
}
```

### 8.3 配置中心化

```yaml
spring:
  ai:
    rag:
      retrieval:
        top-k: 5
        similarity-threshold: 0.7
      chunk:
        size: 800
        overlap: 100
      embedding:
        model: text-embedding-3-small
        batch-size: 20
```

```java
@ConfigurationProperties(prefix = "spring.ai.rag")
@Configuration
public class RagProperties {

    private final Retrieval retrieval = new Retrieval();
    private final Chunk chunk = new Chunk();
    private final Embedding embedding = new Embedding();

    // getters and setters...

    public static class Retrieval {
        private int topK = 5;
        private double similarityThreshold = 0.7;
        // getters and setters...
    }

    public static class Chunk {
        private int size = 800;
        private int overlap = 100;
        // getters and setters...
    }

    public static class Embedding {
        private String model = "text-embedding-3-small";
        private int batchSize = 20;
        // getters and setters...
    }
}
```

---

## 九、性能优化

### 9.1 连接池优化

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
      connection-timeout: 30000
      idle-timeout: 600000
      max-lifetime: 1800000
```

### 9.2 异步Embedding（批量处理）

```java
@Service
public class AsyncEmbeddingPipeline {

    private final VectorStore vectorStore;
    private final TokenTextSplitter textSplitter;

    @Async
    public CompletableFuture<Void> ingestAsync(
            MultipartFile file, Map<String, String> metadata) {
        
        return CompletableFuture.runAsync(() -> {
            List<Document> rawDocs = readerService.readDocument(file);
            rawDocs.forEach(d -> d.getMetadata().putAll(metadata));
            List<Document> chunks = textSplitter.apply(rawDocs);
            
            // 分批Embedding，避免一次性请求过大
            int batchSize = 20;
            for (int i = 0; i < chunks.size(); i += batchSize) {
                int end = Math.min(i + batchSize, chunks.size());
                List<Document> batch = chunks.subList(i, end);
                vectorStore.add(batch);
                log.debug("已处理批次: {}/{}", end, chunks.size());
            }
        });
    }
}
```

### 9.3 缓存热门问答

```java
@Service
public class CachedRagService {

    private final CacheManager cacheManager;
    private final RagQuestionService ragService;

    @Cacheable(value = "rag_answers", key = "#question.hashCode()")
    @Cacheable
    public String ask(String question) {
        return ragService.ask(question);
    }

    @CacheEvict(value = "rag_answers", allEntries = true)
    public void clearCache() {
        log.info("知识库更新，清理问答缓存");
    }
}
```

---

## 十、完整项目结构

```
src/main/java/com/example/rag/
├── config/
│   ├── RagProperties.java          # RAG配置属性
│   ├── TextSplitterConfig.java     # 文档切割器配置
│   └── VectorStoreConfig.java      # 向量数据库配置
├── reader/
│   ├── DocumentReaderService.java  # 文档读取服务
│   └── BatchDocumentReader.java    # 批量读取
├── splitter/
│   └── SmartDocumentSplitter.java  # 智能切割
├── pipeline/
│   ├── DocumentIngestionPipeline.java  # 数据入库管道
│   └── AsyncEmbeddingPipeline.java     # 异步入库
├── retrieval/
│   ├── RagQuestionService.java     # 检索问答
│   ├── FilteredRagService.java     # 带元数据过滤检索
│   └── AdaptiveRagService.java     # 自适应阈值检索
├── controller/
│   ├── DocumentIngestionController.java  # 文档上传API
│   └── RagChatController.java            # 问答API
└── model/
    └── IngestionResult.java
```

---

## 十一、小结

一个完整的Spring AI RAG系统包含四大核心步骤：

```
文档上传 → TikaDocumentReader读取 → TokenTextSplitter切割 → 
OpenAI Embedding向量化 → pgvector存储 → 相似度检索 → LLM生成回答
```

| 步骤 | 关键组件 | 核心配置 |
|------|---------|---------|
| 文档读取 | `TikaDocumentReader` | 支持PDF/Word/TXT/HTML等所有格式 |
| 文档切割 | `TokenTextSplitter` | chunkSize=800, overlap=100 |
| 向量化存储 | `VectorStore` + pgvector | dimensions=1536, index=HNSW |
| 检索增强 | `RetrievalAugmentationAdvisor` | topK=5, threshold=0.7 |
| 过滤检索 | `FilterExpression` | 按部门/类别/时间过滤 |

通过本文的方案，你可以在一天内搭建一个企业级知识库——支持任意格式文档上传、智能分段、向量检索、自然语言问答，还带有元数据过滤和相似度阈值调优。

当你的AI能基于内部文档回答问题，下一步就是要让它"看得懂"图片、听懂语音——这就是**多模态**要解决的问题。

---

> **下一篇预告**：Spring AI 多模态接入——让AI看懂图片、分析视频、理解音频。图文并茂的AI能力实战！

---

*如果觉得有帮助，欢迎点赞收藏关注，你的支持是我持续输出的动力！*

---
