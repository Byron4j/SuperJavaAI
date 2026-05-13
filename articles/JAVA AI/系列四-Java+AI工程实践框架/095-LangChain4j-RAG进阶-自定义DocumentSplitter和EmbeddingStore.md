# LangChain4j RAG 进阶：自定义 DocumentSplitter 和 EmbeddingStore，打造专属于你的检索方案

> 默认的递归文本分割器真能应付所有场景？如果你的文档是代码、法律文书、Markdown、或带复杂表格的报告，通用方案会让你吃尽苦头。今天这篇文章手把手带你自定义 LangChain4j RAG 的每一个核心环节。

---

## 一、开篇：为什么默认方案总不够用？

先看三个让人头疼的真实场景：

**场景 A**：你有一批 Java 项目源码，想做代码问答。默认的递归字符分割（`DocumentSplitters.recursive(1000, 200)`）把 `public class OrderService {` 和它的方法体切成两个 chunk——AI 看到半个类定义，直接乱答。

**场景 B**：你的法律文书有严格的段落结构（第 X 条、第 Y 款），默认分割把"第 10 条"的条款切成两半，导致检索时关键词命中了一个"残废"的 chunk。

**场景 C**：你用的是公司自研的向量数据库，接口跟 LangChain4j 内置的 `EmbeddingStore` 完全不兼容。

**结论**：通用方案只能解决 80% 的问题。剩下 20% 的定制化需求，必须靠你自己去扩展。

LangChain4j 的 RAG 体系设计得极为解耦，**每个环节都可以替换**。今天这篇文章，我会带你一步步实现：自定义分割策略、自定义 Embedding 模型、自定义向量存储、元数据管理，最终组装成一个完整的定制化 RAG Pipeline。

---

## 二、RAG Pipeline 架构回顾

在动手之前，先快速回顾 RAG 的核心流程：

```
原始文档 (PDF/Word/Markdown/Code)
    │
    ▼
[1] DocumentLoader ─── 加载文档，拆分段落
    │
    ▼
[2] DocumentSplitter ─ 将长文本切成小块 (chunks)
    │                        │
    ▼                        ▼
[3] TextSegment ─────── 携带元数据的小文本块
    │
    ▼
[4] EmbeddingModel ─── 将文本向量化（text → vector）
    │
    ▼
[5] EmbeddingStore ─── 向量存储与检索
    │
    ▼
[6] Retriever ──────── 检索相关文档片段
    │
    ▼
[7] Prompt Injection ─ 将检索结果注入到 LLM 提示词

用户提问 ──► LLM ──► 回答
```

今天我们要自定义的是 **[2] DocumentSplitter**、**[4] EmbeddingModel**、**[5] EmbeddingStore** 和元数据管理。

---

## 三、自定义 DocumentSplitter：按语义而非字符数切分

### 3.1 DocumentSplitter 接口

```java
public interface DocumentSplitter {
    List<TextSegment> split(Document document);
    List<TextSegment> splitAll(List<Document> documents);
}
```

非常简单——输入一个 `Document`，输出一组 `TextSegment`。你只需要实现 `split` 方法。

### 3.2 代码场景一：Java 代码分割器（按方法/类边界）

```java
import dev.langchain4j.data.document.Document;
import dev.langchain4j.data.document.DocumentSplitter;
import dev.langchain4j.data.segment.TextSegment;
import dev.langchain4j.model.Tokenizer;

import java.util.*;
import java.util.regex.*;

/**
 * 专门针对 Java 源码的分割器。
 * 按类、方法、字段的自然边界切分，保证代码语义完整。
 */
public class JavaCodeSplitter implements DocumentSplitter {

    private final int maxLinesPerChunk;
    private final Tokenizer tokenizer;
    private final int maxTokensPerChunk;

    // 匹配 Java 方法/类声明的正则
    private static final Pattern METHOD_PATTERN = Pattern.compile(
        "^\\s*(public|private|protected|static|final|abstract|synchronized|native|\\s)+" +
        "[\\w<>\\[\\],\\s]+\\s+\\w+\\s*\\([^)]*\\)\\s*(throws\\s+[\\w\\s,]+)?\\s*\\{?\\s*$"
    );

    private static final Pattern CLASS_PATTERN = Pattern.compile(
        "^\\s*(public\\s+)?(abstract\\s+)?(final\\s+)?(class|interface|enum|@interface)\\s+\\w+"
    );

    public JavaCodeSplitter(int maxLinesPerChunk,
                            Tokenizer tokenizer,
                            int maxTokensPerChunk) {
        this.maxLinesPerChunk = maxLinesPerChunk;
        this.tokenizer = tokenizer;
        this.maxTokensPerChunk = maxTokensPerChunk;
    }

    @Override
    public List<TextSegment> split(Document document) {
        String text = document.text();
        String[] lines = text.split("\n", -1);

        List<TextSegment> chunks = new ArrayList<>();
        List<String> currentChunk = new ArrayList<>();
        int currentLineCount = 0;

        for (int i = 0; i < lines.length; i++) {
            String line = lines[i];

            // 检测到新的类/方法定义时，考虑是否该切分
            boolean isNewBlock = CLASS_PATTERN.matcher(line).find() ||
                                 METHOD_PATTERN.matcher(line).find();

            if (isNewBlock && currentLineCount >= 5) {
                // 将当前积累的 chunk 保存
                chunks.add(createSegment(document, currentChunk));
                currentChunk = new ArrayList<>();
                currentLineCount = 0;
            }

            currentChunk.add(line);
            currentLineCount++;

            // Token 数超标时切分
            if (currentLineCount >= maxLinesPerChunk) {
                String candidateText = String.join("\n", currentChunk);
                int tokenCount = tokenizer.estimateTokenCountInText(candidateText);
                if (tokenCount > maxTokensPerChunk) {
                    chunks.add(createSegment(document, currentChunk));
                    currentChunk = new ArrayList<>();
                    currentLineCount = 0;
                }
            }
        }

        // 处理最后一个 chunk
        if (!currentChunk.isEmpty()) {
            chunks.add(createSegment(document, currentChunk));
        }

        return chunks;
    }

    private TextSegment createSegment(Document document,
                                       List<String> lines) {
        String text = String.join("\n", lines);
        return TextSegment.from(text,
            Metadata.from("file_name", document.metadata("file_name"),
                          "language", "java",
                          "lines", String.join("-",
                              String.valueOf(document.metadata().getInteger("start_line", 1)),
                              String.valueOf(document.metadata().getInteger("start_line", 1) + lines.size()))
            ));
    }

    @Override
    public List<TextSegment> splitAll(List<Document> documents) {
        return documents.stream()
                .flatMap(doc -> split(doc).stream())
                .toList();
    }
}
```

### 3.3 代码场景二：法律文书分割器（按条款边界）

```java
/**
 * 按法律文书的"第X条、第Y款"结构进行语义分割
 */
public class LegalDocumentSplitter implements DocumentSplitter {

    // 匹配 "第X条" "第X款" "第X章" "第X节" 等中文法律文档结构
    private static final Pattern ARTICLE_PATTERN = Pattern.compile(
        "^[\\s　]*(第[一二三四五六七八九十百千万\\d]+[条章节款项])"
    );

    private final int maxTokensPerChunk;
    private final Tokenizer tokenizer;

    public LegalDocumentSplitter(int maxTokensPerChunk, Tokenizer tokenizer) {
        this.maxTokensPerChunk = maxTokensPerChunk;
        this.tokenizer = tokenizer;
    }

    @Override
    public List<TextSegment> split(Document document) {
        String text = document.text();
        String[] paragraphs = text.split("\n\n+");

        List<TextSegment> chunks = new ArrayList<>();
        StringBuilder currentChunk = new StringBuilder();
        String currentArticle = "序言";

        for (String paragraph : paragraphs) {
            Matcher m = ARTICLE_PATTERN.matcher(paragraph);
            if (m.find()) {
                // 遇到新条款，保存上一个 chunk
                if (currentChunk.length() > 0) {
                    int tokens = tokenizer.estimateTokenCountInText(
                            currentChunk.toString());
                    if (tokens > 0) {
                        chunks.add(createSegment(currentChunk.toString(),
                                document, currentArticle));
                    }
                }
                currentChunk = new StringBuilder();
                currentArticle = m.group(1);
            }

            currentChunk.append(paragraph).append("\n\n");

            int currentTokens = tokenizer.estimateTokenCountInText(
                    currentChunk.toString());

            if (currentTokens > maxTokensPerChunk) {
                // 当前条款太长，强制分割
                chunks.add(createSegment(currentChunk.toString(),
                        document, currentArticle));
                currentChunk = new StringBuilder();
            }
        }

        // 最后一个 chunk
        if (currentChunk.length() > 0) {
            chunks.add(createSegment(currentChunk.toString(),
                    document, currentArticle));
        }

        return chunks;
    }

    private TextSegment createSegment(String text, Document doc,
                                       String article) {
        return TextSegment.from(text,
            Metadata.from(
                "source", doc.metadata("file_name"),
                "article", article,
                "type", "legal_document"
            ));
    }

    @Override
    public List<TextSegment> splitAll(List<Document> documents) {
        return documents.stream()
                .flatMap(doc -> split(doc).stream())
                .toList();
    }
}
```

### 3.4 代码场景三：Markdown 结构分割器（按标题层级）

```java
/**
 * 按 Markdown 标题层级（# ## ###）进行语义分割
 */
public class MarkdownSplitter implements DocumentSplitter {

    private static final Pattern HEADING_PATTERN =
        Pattern.compile("^(#{1,6})\\s+(.+)", Pattern.MULTILINE);

    private final int maxTokensPerChunk;
    private final Tokenizer tokenizer;

    public MarkdownSplitter(int maxTokensPerChunk, Tokenizer tokenizer) {
        this.maxTokensPerChunk = maxTokensPerChunk;
        this.tokenizer = tokenizer;
    }

    @Override
    public List<TextSegment> split(Document document) {
        String text = document.text();

        // 找到所有标题位置
        List<int[]> headings = new ArrayList<>();
        Matcher m = HEADING_PATTERN.matcher(text);
        while (m.find()) {
            int level = m.group(1).length();    // # 的数量
            String title = m.group(2).trim();
            headings.add(new int[]{m.start(), level, m.end()});
        }

        if (headings.isEmpty()) {
            // 无标题，退化到简单分割
            return simpleSplit(document, text);
        }

        List<TextSegment> chunks = new ArrayList<>();

        for (int i = 0; i < headings.size(); i++) {
            int start = headings.get(i)[0];
            int end = (i + 1 < headings.size())
                    ? headings.get(i + 1)[0]
                    : text.length();
            int level = headings.get(i)[1];
            String title = text.substring(
                    headings.get(i)[2], text.indexOf('\n', headings.get(i)[2])).trim();

            String chunkText = text.substring(start, end).trim();

            // 构建标题路径
            StringBuilder hierarchy = new StringBuilder();
            for (int j = 0; j <= i; j++) {
                if (headings.get(j)[1] <= level) {
                    if (hierarchy.length() > 0) {
                        hierarchy.append(" > ");
                    }
                    String t = text.substring(
                            headings.get(j)[2],
                            text.indexOf('\n', headings.get(j)[2])).trim();
                    hierarchy.append(t);
                }
            }

            // Token 检查：如果超限则二次切分
            int tokens = tokenizer.estimateTokenCountInText(chunkText);
            if (tokens > maxTokensPerChunk) {
                // 按段落二次切
                String[] subChunks = chunkText.split("\n\n+");
                StringBuilder subBuffer = new StringBuilder();
                int subTokens = 0;
                for (String sub : subChunks) {
                    int st = tokenizer.estimateTokenCountInText(sub);
                    if (subTokens + st > maxTokensPerChunk && subBuffer.length() > 0) {
                        chunks.add(buildSegment(subBuffer.toString(),
                                document, title, hierarchy.toString(), level));
                        subBuffer = new StringBuilder();
                        subTokens = 0;
                    }
                    subBuffer.append(sub).append("\n\n");
                    subTokens += st;
                }
                if (subBuffer.length() > 0) {
                    chunks.add(buildSegment(subBuffer.toString(),
                            document, title, hierarchy.toString(), level));
                }
            } else {
                chunks.add(buildSegment(chunkText,
                        document, title, hierarchy.toString(), level));
            }
        }

        return chunks;
    }

    private TextSegment buildSegment(String text, Document doc,
                                      String title, String hierarchy, int level) {
        return TextSegment.from(text,
            Metadata.from(
                "source", doc.metadata("file_name"),
                "title", title,
                "heading_hierarchy", hierarchy,
                "heading_level", String.valueOf(level)
            ));
    }

    private List<TextSegment> simpleSplit(Document document, String text) {
        return DocumentSplitters.recursive(1000, 200).split(document);
    }

    @Override
    public List<TextSegment> splitAll(List<Document> documents) {
        return documents.stream()
                .flatMap(doc -> split(doc).stream())
                .toList();
    }
}
```

---

## 四、自定义 Embedding 模型

### 4.1 为什么需要自定义？

- 公司内部部署了私有的 Embedding 服务（如 text2vec-large-chinese）
- 使用了云厂商的专有 Embedding API（阿里云/华为云）
- 出于安全原因不能把文本发给外部 API

### 4.2 实现自定义 EmbeddingModel

```java
import dev.langchain4j.data.embedding.Embedding;
import dev.langchain4j.data.segment.TextSegment;
import dev.langchain4j.model.embedding.EmbeddingModel;
import dev.langchain4j.model.output.Response;

import java.util.List;

/**
 * 对接公司内部部署的 text2vec-large-chinese 模型
 */
public class InternalEmbeddingModel implements EmbeddingModel {

    private final String apiEndpoint;
    private final HttpClient httpClient;
    private final ObjectMapper objectMapper;

    public InternalEmbeddingModel(String apiEndpoint) {
        this.apiEndpoint = apiEndpoint;
        this.httpClient = HttpClient.newHttpClient();
        this.objectMapper = new ObjectMapper();
    }

    @Override
    public Response<List<Embedding>> embedAll(List<TextSegment> segments) {
        try {
            // 批量提取文本
            List<String> texts = segments.stream()
                    .map(TextSegment::text)
                    .toList();

            // 调用内部 Embedding API
            ObjectNode body = objectMapper.createObjectNode();
            ArrayNode textsArray = body.putArray("texts");
            texts.forEach(textsArray::add);
            body.put("batch_size", segments.size());

            HttpRequest request = HttpRequest.newBuilder()
                    .uri(URI.create(apiEndpoint + "/v1/embeddings"))
                    .header("Content-Type", "application/json")
                    .POST(HttpRequest.BodyPublishers.ofString(body.toString()))
                    .build();

            HttpResponse<String> response =
                    httpClient.send(request, HttpResponse.BodyHandlers.ofString());

            // 解析返回的向量
            JsonNode root = objectMapper.readTree(response.body());
            JsonNode embeddings = root.get("embeddings");

            List<Embedding> results = new ArrayList<>();
            for (int i = 0; i < embeddings.size(); i++) {
                float[] vector = objectMapper.treeToValue(
                        embeddings.get(i), float[].class);
                results.add(Embedding.from(vector));
            }

            return Response.from(results);
        } catch (Exception e) {
            throw new RuntimeException("Embedding 调用失败", e);
        }
    }

    @Override
    public int dimension() {
        // text2vec-large-chinese 输出 1024 维
        return 1024;
    }
}
```

### 4.3 批量处理优化

单条 Embedding 效率低，建议实现批量接口：

```java
public class BatchOptimizedEmbeddingModel implements EmbeddingModel {

    private final EmbeddingModel delegate;
    private final int batchSize;

    public BatchOptimizedEmbeddingModel(EmbeddingModel delegate, int batchSize) {
        this.delegate = delegate;
        this.batchSize = batchSize;
    }

    @Override
    public Response<List<Embedding>> embedAll(List<TextSegment> segments) {
        List<Embedding> allEmbeddings = new ArrayList<>();

        for (int i = 0; i < segments.size(); i += batchSize) {
            int end = Math.min(i + batchSize, segments.size());
            List<TextSegment> batch = segments.subList(i, end);
            Response<List<Embedding>> batchResponse = delegate.embedAll(batch);
            allEmbeddings.addAll(batchResponse.content());
        }

        return Response.from(allEmbeddings);
    }
}
```

---

## 五、自定义 EmbeddingStore：对接任意向量数据库

### 5.1 EmbeddingStore 接口

```java
public interface EmbeddingStore<Embedded> {
    String add(Embedding embedding, TextSegment segment);
    void add(String id, Embedding embedding, TextSegment segment);
    List<String> addAll(List<Embedding> embeddings, List<TextSegment> segments);
    List<EmbeddingMatch<TextSegment>> findRelevant(
        Embedding referenceEmbedding, int maxResults, double minScore);
}
```

### 5.2 PostgreSQL + pgvector 实现

```java
import dev.langchain4j.data.embedding.Embedding;
import dev.langchain4j.data.segment.TextSegment;
import dev.langchain4j.store.embedding.EmbeddingMatch;
import dev.langchain4j.store.embedding.EmbeddingStore;

import java.sql.*;
import java.util.*;

/**
 * 基于 PostgreSQL + pgvector 扩展的自定义向量存储
 *
 * 前置建表SQL：
 * CREATE EXTENSION IF NOT EXISTS vector;
 * CREATE TABLE custom_embeddings (
 *     id        VARCHAR(64) PRIMARY KEY DEFAULT gen_random_uuid(),
 *     content   TEXT,
 *     embedding VECTOR(1536),
 *     metadata  JSONB,
 *     created_at TIMESTAMP DEFAULT NOW()
 * );
 * CREATE INDEX ON custom_embeddings
 *     USING ivfflat (embedding vector_cosine_ops) WITH (lists = 100);
 */
public class PgVectorEmbeddingStore implements EmbeddingStore<TextSegment> {

    private final DataSource dataSource;
    private final int dimension;

    public PgVectorEmbeddingStore(DataSource dataSource, int dimension) {
        this.dataSource = dataSource;
        this.dimension = dimension;
    }

    @Override
    public String add(Embedding embedding, TextSegment segment) {
        return add(UUID.randomUUID().toString(), embedding, segment);
    }

    @Override
    public String add(String id, Embedding embedding, TextSegment segment) {
        addAll(Collections.singletonList(embedding),
               Collections.singletonList(segment));
        return id;
    }

    @Override
    public List<String> addAll(List<Embedding> embeddings,
                                List<TextSegment> segments) {
        String sql = "INSERT INTO custom_embeddings (id, content, embedding, metadata) " +
                     "VALUES (?, ?, ?::vector, ?::jsonb) " +
                     "ON CONFLICT (id) DO NOTHING";

        List<String> ids = new ArrayList<>();

        try (Connection conn = dataSource.getConnection();
             PreparedStatement ps = conn.prepareStatement(sql)) {

            for (int i = 0; i < embeddings.size(); i++) {
                TextSegment segment = segments.get(i);
                Embedding embedding = embeddings.get(i);
                String id = segment.metadata().get("id");

                ps.setString(1, id != null ? id : UUID.randomUUID().toString());
                ps.setString(2, segment.text());
                ps.setString(3, vectorToPgString(embedding.vector()));
                ps.setString(4, metadataToJson(segment.metadata()));
                ps.addBatch();
                ids.add(id);
            }

            ps.executeBatch();
            return ids;
        } catch (SQLException e) {
            throw new RuntimeException("写入向量失败", e);
        }
    }

    @Override
    public List<EmbeddingMatch<TextSegment>> findRelevant(
            Embedding referenceEmbedding, int maxResults, double minScore) {

        // 使用余弦距离查询 (pgvector 的 <=> 操作符)
        String sql = """
            SELECT id, content, metadata,
                   1 - (embedding <=> ?::vector) AS similarity
            FROM custom_embeddings
            WHERE 1 - (embedding <=> ?::vector) >= ?
            ORDER BY embedding <=> ?::vector
            LIMIT ?
            """;

        try (Connection conn = dataSource.getConnection();
             PreparedStatement ps = conn.prepareStatement(sql)) {

            String vectorStr = vectorToPgString(referenceEmbedding.vector());
            ps.setString(1, vectorStr);
            ps.setDouble(2, minScore);
            ps.setString(3, vectorStr);
            ps.setInt(4, maxResults);

            ResultSet rs = ps.executeQuery();
            List<EmbeddingMatch<TextSegment>> results = new ArrayList<>();

            while (rs.next()) {
                double score = rs.getDouble("similarity");
                String content = rs.getString("content");
                String metadataJson = rs.getString("metadata");
                Metadata metadata = jsonToMetadata(metadataJson);

                results.add(new EmbeddingMatch<>(score, id, null,
                        TextSegment.from(content, metadata)));
            }

            return results;
        } catch (SQLException e) {
            throw new RuntimeException("向量检索失败", e);
        }
    }

    // ===== 工具方法 =====

    private String vectorToPgString(float[] vector) {
        StringBuilder sb = new StringBuilder("[");
        for (int i = 0; i < vector.length; i++) {
            if (i > 0) sb.append(",");
            sb.append(vector[i]);
        }
        sb.append("]");
        return sb.toString();
    }

    private String metadataToJson(Metadata metadata) {
        ObjectMapper mapper = new ObjectMapper();
        try {
            Map<String, Object> map = new HashMap<>();
            for (String key : metadata.toMap().keySet()) {
                map.put(key, metadata.toMap().get(key));
            }
            return mapper.writeValueAsString(map);
        } catch (Exception e) {
            return "{}";
        }
    }

    private Metadata jsonToMetadata(String json) {
        ObjectMapper mapper = new ObjectMapper();
        try {
            Map<String, Object> map = mapper.readValue(json,
                    new TypeReference<Map<String, Object>>() {});
            Metadata metadata = new Metadata();
            map.forEach((k, v) -> metadata.put(k, String.valueOf(v)));
            return metadata;
        } catch (Exception e) {
            return new Metadata();
        }
    }
}
```

### 5.3 Elasticsearch 实现（适合全文+向量混合检索）

```java
/**
 * 基于 Elasticsearch 8.x 的向量存储，
 * 支持 dense_vector + 文本 BM25 混合检索
 */
public class ElasticsearchEmbeddingStore implements EmbeddingStore<TextSegment> {

    private final RestHighLevelClient client;
    private final String indexName;
    private final int dimension;

    // 索引 Mapping 创建
    public static final String INDEX_MAPPING = """
        {
          "mappings": {
            "properties": {
              "content":    { "type": "text" },
              "embedding":  { "type": "dense_vector", "dims": %d,
                              "index": true, "similarity": "cosine" },
              "metadata":   { "type": "object", "enabled": false },
              "created_at": { "type": "date" }
            }
          }
        }
        """.formatted(dimension);

    // ... 实现 add/addAll/findRelevant 方法，思路同上略
}
```

---

## 六、元数据管理：让你的 Chunk 自带"身份证"

元数据（Metadata）是 RAG 中被严重低估的能力。默认分割出来的 chunk 只是一段裸文本，你根本不知道它来自哪个文件、第几页、哪个章节。**元数据就是 chunk 的身份证**。

### 6.1 元数据的核心字段

```java
public class RAGMetadataKeys {
    public static final String FILE_NAME   = "file_name";
    public static final String FILE_PATH   = "file_path";
    public static final String PAGE_NUMBER = "page_number";
    public static final String CHUNK_INDEX = "chunk_index";
    public static final String TOTAL_CHUNKS = "total_chunks";
    public static final String AUTHOR      = "author";
    public static final String CREATED_AT  = "created_at";
    public static final String DOC_TYPE    = "doc_type";
    public static final String LANGUAGE    = "language";
    public static final String CATEGORY    = "category";
}
```

### 6.2 元数据在检索时的作用

```java
public class MetadataAwareRetriever {

    private final EmbeddingStore<TextSegment> store;
    private final EmbeddingModel embeddingModel;

    /**
     * 检索 + 元数据过滤
     */
    public List<TextSegment> retrieve(String query,
                                       Map<String, String> filterConditions,
                                       int maxResults) {

        // 构建带过滤条件的 EmbeddingStore 扩展
        // 这里演示思路：先向量检索，再根据元数据过滤

        Embedding queryEmbedding = embeddingModel.embed(query).content();
        List<EmbeddingMatch<TextSegment>> matches =
                store.findRelevant(queryEmbedding, maxResults * 2, 0.7);

        return matches.stream()
                .map(EmbeddingMatch::embedded)
                .filter(segment -> {
                    // 元数据过滤逻辑
                    for (Map.Entry<String, String> cond : filterConditions.entrySet()) {
                        String actualValue = segment.metadata().get(cond.getKey());
                        if (actualValue == null ||
                            !actualValue.equals(cond.getValue())) {
                            return false; // 不满足过滤条件
                        }
                    }
                    return true;
                })
                .limit(maxResults)
                .toList();
    }
}
```

### 6.3 元数据驱动的混合检索

```java
/**
 * 关键词 BM25 + 向量相似度 + 元数据过滤 的混合检索
 */
public class HybridRetriever {

    private final EmbeddingStore<TextSegment> vectorStore;
    private final EmbeddingModel embeddingModel;

    public String retrieveContext(String query,
                                   Map<String, String> metadataFilter) {

        // 1. 向量检索
        Embedding queryEmb = embeddingModel.embed(query).content();
        List<EmbeddingMatch<TextSegment>> candidates =
                vectorStore.findRelevant(queryEmb, 20, 0.65);

        // 2. 关键词 BM25 加分（复用 Elasticsearch）
        // candidates = rerankByKeywords(candidates, query);

        // 3. 元数据过滤
        List<TextSegment> filtered = candidates.stream()
                .map(EmbeddingMatch::embedded)
                .filter(s -> matchMetadata(s.metadata(), metadataFilter))
                .limit(5)
                .toList();

        // 4. 拼接上下文
        StringBuilder context = new StringBuilder();
        for (int i = 0; i < filtered.size(); i++) {
            TextSegment seg = filtered.get(i);
            context.append(String.format("[来源: %s, 章节: %s]\n%s\n\n",
                    seg.metadata().get("file_name"),
                    seg.metadata().getOrDefault("heading_hierarchy", "未知"),
                    seg.text()));
        }

        return context.toString();
    }

    private boolean matchMetadata(Metadata metadata,
                                   Map<String, String> filter) {
        if (filter == null || filter.isEmpty()) return true;
        return filter.entrySet().stream().allMatch(e ->
                e.getValue().equals(metadata.get(e.getKey())));
    }
}
```

---

## 七、完整自定义 RAG Pipeline 组装

现在把所有自定义组件串起来：

```java
@SpringBootApplication
public class CustomRAGApplication {

    public static void main(String[] args) {
        SpringApplication.run(CustomRAGApplication.class, args);
    }

    @Bean
    public EmbeddingModel embeddingModel(
            @Value("${embedding.api.endpoint}") String endpoint) {
        return new BatchOptimizedEmbeddingModel(
                new InternalEmbeddingModel(endpoint), 50);
    }

    @Bean
    public EmbeddingStore<TextSegment> embeddingStore(DataSource ds) {
        return new PgVectorEmbeddingStore(ds, 1536);
    }

    @Bean
    public DocumentSplitter documentSplitter(
            EmbeddingModel embeddingModel) {
        // 根据文档类型选择不同分割器
        return document -> {
            String docType = document.metadata().get("doc_type");
            if ("java".equals(docType)) {
                return new JavaCodeSplitter(80,
                        new OpenAiTokenizer("gpt-4o-mini"), 1500)
                        .split(document);
            } else if ("legal".equals(docType)) {
                return new LegalDocumentSplitter(800,
                        new OpenAiTokenizer("gpt-4o-mini"))
                        .split(document);
            } else if ("markdown".equals(docType)) {
                return new MarkdownSplitter(1000,
                        new OpenAiTokenizer("gpt-4o-mini"))
                        .split(document);
            } else {
                return DocumentSplitters.recursive(800, 100)
                        .split(document);
            }
        };
    }

    @Bean
    public Assistant ragAssistant(OpenAiChatModel chatModel,
                                   EmbeddingModel embeddingModel,
                                   EmbeddingStore<TextSegment> embeddingStore,
                                   DocumentSplitter splitter) {

        // 构建完整的 RAG pipeline
        return AiServices.builder(Assistant.class)
                .chatLanguageModel(chatModel)
                .contentRetriever(query -> {
                    // 1. 向量检索
                    Embedding queryEmb = embeddingModel.embed(query).content();
                    List<EmbeddingMatch<TextSegment>> matches =
                            embeddingStore.findRelevant(queryEmb, 5, 0.7);

                    // 2. 格式化检索结果
                    StringBuilder context = new StringBuilder();
                    for (int i = 0; i < matches.size(); i++) {
                        TextSegment seg = matches.get(i).embedded();
                        double score = matches.get(i).score();
                        context.append(String.format(
                            "[#%d | 相似度:%.2f | 来源:%s | 标题:%s]\n%s\n\n",
                            i + 1, score,
                            seg.metadata().get("source"),
                            seg.metadata().getOrDefault("title", "-"),
                            seg.text()
                        ));
                    }

                    return Content.from(context.toString());
                })
                .build();
    }

    /**
     * 文档摄入接口：加载文档 → 分割 → 向量化 → 存入存储
     */
    @Bean
    public CommandLineRunner ingestDocuments(
            DocumentSplitter splitter,
            EmbeddingModel embeddingModel,
            EmbeddingStore<TextSegment> store,
            @Value("${documents.path}") String documentsPath) {

        return args -> {
            Path path = Paths.get(documentsPath);
            List<Path> files = Files.list(path)
                    .filter(Files::isRegularFile)
                    .toList();

            for (Path file : files) {
                String content = Files.readString(file);
                String docType = detectDocType(file.getFileName().toString());

                Document doc = Document.from(content,
                        Metadata.from(
                                "file_name", file.getFileName().toString(),
                                "file_path", file.toString(),
                                "doc_type", docType
                        ));

                // 分割
                List<TextSegment> segments = splitter.split(doc);

                // 向量化
                List<Embedding> embeddings = embeddingModel.embedAll(
                        segments.stream()
                                .map(TextSegment::from)
                                .toList()
                ).content();

                // 存入存储
                store.addAll(embeddings, segments);

                System.out.printf("已摄入: %s → %d chunks\n",
                        file.getFileName(), segments.size());
            }

            System.out.println("文档摄入完成！共处理 " + files.size() + " 个文件");
        };
    }

    private String detectDocType(String fileName) {
        if (fileName.endsWith(".java")) return "java";
        if (fileName.endsWith(".md")) return "markdown";
        if (fileName.contains("法律") || fileName.contains("合同")) return "legal";
        return "text";
    }

    public interface Assistant {
        String chat(String message);
    }
}
```

---

## 八、性能优化建议

### 8.1 分割阶段

- **预处理压缩**：移除多余空白、注释、页眉页脚，减少噪音
- **适度 overlap**：代码分块 overlap 设为 3-5 行，自然语言 overlap 设为 10-20%
- **控制 chunk 大小**：代码建议 500-1500 Token，文档建议 800-2000 Token

### 8.2 Embedding 阶段

- **批量发送**：一次发送 20-50 个文本块，减少 HTTP 往返
- **并行处理**：大文档用 `parallelStream()` 并发处理
- **缓存 Embedding**：相同文本块的向量只算一次

### 8.3 检索阶段

- **索引优化**：pgvector 用 IVFFlat/HNSW 索引，ES 用 HNSW
- **混合检索**：向量 + BM25 关键词的组合效果远好于纯向量
- **元数据预过滤**：先按文档类型/日期范围缩小候选集，再做向量检索

---

## 九、写在最后

RAG 不是"调一个默认 API 就能搞定"的事情。真正落地的 RAG 系统，90% 的精力都花在**数据预处理、分割策略、检索优化**上。LangChain4j 提供了一个优雅的可插拔架构，让你能像搭积木一样替换每一个环节。

默认方案是你的起点，自定义方案才是你的竞争力。

---

**下一篇预告**：最后一篇我们聊聊 **LangChain4j 与 Quarkus 集成**——如何在云原生环境中打造一个启动只需 0.5 秒的 AI 微服务。GraalVM Native Image + LangChain4j + Quarkus，这是一套让 Spring Boot 望尘莫及的组合拳。

> 觉得有用就点个赞，评论区聊聊你的 RAG 踩坑经历！
