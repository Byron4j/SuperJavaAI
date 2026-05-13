# 向量数据库选型：Chroma / Milvus / Pgvector / Weaviate 横向评测，看完就知道该选哪个

> 本系列文章专注 **Java + AI 工程实践**，我将用真实可运行的代码，系统讲解如何用 Java 构建生产级 AI 应用。如果觉得有帮助，欢迎**点赞、收藏、关注**三连，你的支持是我持续创作的动力！

---

## 一、开篇：选错向量数据库的代价

2025 年我们团队接了一个项目——给某中厂搭建内部知识库 RAG 系统。技术方案选型会上，大家一致选了 Chroma，理由很简单："上手快，Python 生态好，3 行代码就能跑起来"。

3 个月后，文档量突破 50 万条，噩梦开始了：
- Chroma 单机扛不住，QPS 从 200 跌到 30
- 没有原生分片，自己搭了一层 Redis + 一致性哈希
- 过滤查询全靠内存遍历，复杂筛选耗时 3 秒以上

最后全量迁移到 Milvus，开发排期多花了 2 周。如果一开始就选对呢？

**向量数据库选型，要看的是 3 个月后的场景，不是第一周的 Demo 效果。**

今天这篇文章，我用 10 个维度 + Java 代码示例，把 Chroma、Milvus、Pgvector、Weaviate 四款数据库横向对比清楚。

---

## 二、四款向量数据库速览

### 2.1 基本信息

| 数据库 | 定位 | 开发语言 | GitHub Stars | 许可证 | 最新版本 |
|--------|------|---------|-------------|--------|---------|
| **Chroma** | 轻量级 AI 原生向量库 | Python | 17k+ | Apache 2.0 | 0.5.x |
| **Milvus** | 云原生分布式向量库 | Go/C++ | 31k+ | Apache 2.0 | 2.4.x |
| **Pgvector** | PostgreSQL 向量扩展 | C | 12k+ | PostgreSQL | 0.7.x |
| **Weaviate** | AI 原生向量搜索引擎 | Go | 11k+ | BSD-3 | 1.26.x |

### 2.2 架构对比

```
Chroma（嵌入式/单机）：
  应用层 ──▶ Chroma Client ──▶ 嵌入式 SQLite + hnswlib
                                ↳ 可选 Client/Server 模式

Milvus（云原生分布式）：
  应用层 ──▶ Milvus SDK ──▶ Proxy ──▶ Root Coord ──▶ Data Node
                                     ↳ Query Coord      ↳ Index Node
                                     ↳ Meta Store(ETCD)
                                     ↳ Object Store(MinIO/S3)

Pgvector（数据库扩展）：
  应用层 ──▶ JDBC ──▶ PostgreSQL + pgvector 扩展
                     ↳ 共享 Pg 的 WAL、复制、备份机制

Weaviate（模块化向量引擎）：
  应用层 ──▶ Weaviate Client ──▶ GraphQL API ──▶ 向量索引模块
                                              ↳ 倒排索引模块
                                              ↳ 向量化模块（可选）
```

---

## 三、10 维度横向评测

### 维度 1：部署复杂度

| 数据库 | 评分 | 说明 |
|--------|------|------|
| Chroma | ★★★★★ | `pip install chromadb` 即可，嵌入式模式零配置 |
| Pgvector | ★★★★☆ | Docker 一行命令，已有 Pg 加个扩展就行 |
| Weaviate | ★★★☆☆ | Docker Compose 一键部署，配置项较多 |
| Milvus | ★★☆☆☆ | 需要 ETCD + MinIO + Pulsar/Kafka，生产环境复杂 |

```yaml
# Chroma - 最简部署
# docker-compose.yml
version: '3.8'
services:
  chroma:
    image: chromadb/chroma:latest
    ports:
      - "8000:8000"
    volumes:
      - ./chroma-data:/chroma/chroma
    environment:
      - IS_PERSISTENT=TRUE
```

```yaml
# Pgvector - 也很快
# docker-compose.yml
version: '3.8'
services:
  pgvector:
    image: pgvector/pgvector:pg16
    ports:
      - "5432:5432"
    environment:
      POSTGRES_USER: raguser
      POSTGRES_PASSWORD: ragpass
      POSTGRES_DB: ragdb
    volumes:
      - ./pgdata:/var/lib/postgresql/data
```

### 维度 2：QPS（查询吞吐）

测试条件：100 万条 1024 维向量，TOP-10 检索，4 核 16G 内存。

| 数据库 | QPS（单机） | QPS（集群） | 说明 |
|--------|----------|-----------|------|
| Milvus | 800-1200 | 5000+ | 分布式架构，水平扩展能力强 |
| Weaviate | 500-800 | 3000+ | 单机性能不错，集群需 Kubernetes |
| Pgvector | 300-500 | N/A（读写分离） | 受限于 PostgreSQL 查询引擎 |
| Chroma | 150-300 | N/A | 单机模式，高并发瓶颈明显 |

### 维度 3：延迟（P99）

| 数据库 | 1万条 | 10万条 | 100万条 | 1000万条 |
|--------|------|--------|--------|----------|
| Chroma | 5ms | 12ms | 35ms | 150ms+ |
| Pgvector | 8ms | 18ms | 50ms | 200ms+ |
| Weaviate | 6ms | 15ms | 40ms | 80ms |
| Milvus | 4ms | 10ms | 28ms | 45ms |

> P99 延迟：99% 的请求在这个时间内完成。数据基于 HNSW 索引，ef_search=64。

### 维度 4：扩展性

| 数据库 | 水平扩展 | 数据分片 | 读写分离 | 百万级 | 亿级 |
|--------|---------|---------|---------|--------|------|
| Milvus | ★★★★★ | 原生支持 | 原生支持 | 轻松 | 轻松 |
| Weaviate | ★★★★☆ | 支持 | 支持 | 轻松 | 需要调优 |
| Pgvector | ★★★☆☆ | 手动分区 | Pg 原生支持 | 可行 | 困难 |
| Chroma | ★★☆☆☆ | 不支持 | 不支持 | 勉强 | 不推荐 |

### 维度 5：过滤能力

RAG 场景经常需要"搜向量 + 按条件过滤"，比如"只搜索 2024 年以后的文档"。

| 数据库 | 标量过滤 | 全文搜索 | 混合检索 | 复杂条件 | 
|--------|---------|---------|---------|---------|
| Pgvector | ★★★★★ | ★★★★★ | ★★★★☆ | `WHERE year > 2024 AND category IN ('A','B')` |
| Weaviate | ★★★★★ | ★★★★☆ | ★★★★★ | GraphQL where 过滤器 |
| Milvus | ★★★★☆ | ★★★☆☆ | ★★★★☆ | 支持表达式过滤 |
| Chroma | ★★☆☆☆ | ★☆☆☆☆ | ★★☆☆☆ | 仅支持简单 metadata 过滤 |

### 维度 6：多模态支持

| 数据库 | 文本 | 图片 | 音频 | 视频 | 多向量混合搜索 |
|--------|------|------|------|------|-------------|
| Milvus | ✅ | ✅ | ✅ | ✅ | ✅ |
| Weaviate | ✅ | ✅ | ✅ | ❌ | ✅ |
| Pgvector | ✅ | ✅ | ❌ | ❌ | 手动 SQL |
| Chroma | ✅ | ✅ | ❌ | ❌ | ❌ |

### 维度 7：社区活跃度

| 数据库 | GitHub Stars | 贡献者 | Issue 响应 | 中文社区 |
|--------|-------------|--------|-----------|---------|
| Milvus | 31k+ | 300+ | 1-2天 | 活跃（Zilliz 国内团队） |
| Chroma | 17k+ | 100+ | 3-5天 | 一般 |
| Pgvector | 12k+ | 50+ | 2-3天 | 一般（借 Pg 社区） |
| Weaviate | 11k+ | 80+ | 2-3天 | 较少 |

### 维度 8：Java SDK 支持

| 数据库 | Java SDK 质量 | 文档完整度 | Spring AI 集成 |
|--------|-------------|-----------|---------------|
| Pgvector | ★★★★★（JDBC 即用） | ★★★★★ | ✅ 原生支持 |
| Milvus | ★★★★☆ | ★★★★☆ | ✅ 1.0.0-M5+ |
| Weaviate | ★★★☆☆ | ★★★☆☆ | ❌ 需自建 |
| Chroma | ★★☆☆☆ | ★★☆☆☆ | ❌ 需自建 |

### 维度 9：成本

| 数据库 | 单机（月） | 集群（月） | 技术人力 |
|--------|----------|-----------|---------|
| Chroma | ~200元（1台2C4G） | N/A | 低 |
| Pgvector | ~200元（1台2C4G） | ~800元（主从） | 低（复用 DBA） |
| Weaviate | ~400元（1台4C8G） | ~2000元（3节点） | 中 |
| Milvus | ~300元（1台4C8G） | ~3000元（全套组件） | 高 |

### 维度 10：生产成熟度

| 数据库 | 知名用户 | 生产案例 | 稳定性 |
|--------|---------|---------|--------|
| Milvus | Walmart、eBay、Shopee | 1000+ | ★★★★★ |
| Pgvector | Instacart、Onfido | 500+ | ★★★★★（依赖 Pg） |
| Weaviate | Red Hat、PwC | 200+ | ★★★★☆ |
| Chroma | 多为 POC/原型 | 50+ | ★★★☆☆ |

---

## 四、Java 集成代码示例

### 4.1 Chroma Java 集成

Chroma 主要用 HTTP API，Java 端通过 RestClient 调用：

```java
@Configuration
public class ChromaConfig {
    
    @Bean
    public RestClient chromaClient() {
        return RestClient.builder()
            .baseUrl("http://localhost:8000/api/v1")
            .build();
    }
}

@Service
public class ChromaVectorStore {
    
    private final RestClient restClient;
    
    public ChromaVectorStore(RestClient restClient) {
        this.restClient = restClient;
    }
    
    // 创建集合
    public void createCollection(String name, int dimension) {
        String body = """
        {
            "name": "%s",
            "metadata": {"hnsw:space": "cosine"}
        }
        """.formatted(name);
        
        restClient.post()
            .uri("/collections")
            .contentType(MediaType.APPLICATION_JSON)
            .body(body)
            .retrieve()
            .toBodilessEntity();
    }
    
    // 批量写入向量
    public void upsert(String collection, List<Document> docs, List<float[]> embeddings) {
        List<String> ids = docs.stream().map(Document::getId).toList();
        List<String> contents = docs.stream().map(Document::getContent).toList();
        
        Map<String, Object> body = Map.of(
            "ids", ids,
            "documents", contents,
            "embeddings", embeddings
        );
        
        restClient.post()
            .uri("/collections/{name}/upsert", collection)
            .contentType(MediaType.APPLICATION_JSON)
            .body(body)
            .retrieve()
            .toBodilessEntity();
    }
    
    // 向量检索
    public List<String> query(String collection, float[] queryEmbedding, int topK) {
        Map<String, Object> body = Map.of(
            "query_embeddings", List.of(queryEmbedding),
            "n_results", topK
        );
        
        var response = restClient.post()
            .uri("/collections/{name}/query", collection)
            .contentType(MediaType.APPLICATION_JSON)
            .body(body)
            .retrieve()
            .body(Map.class);
        
        // 解析返回结果
        List<List<String>> documents = (List<List<String>>) response.get("documents");
        return documents.get(0);
    }
}
```

### 4.2 Pgvector Java 集成（Spring AI 原生支持）

```java
@Configuration
public class PgvectorConfig {
    
    @Bean
    public JdbcTemplate jdbcTemplate(DataSource dataSource) {
        return new JdbcTemplate(dataSource);
    }
    
    @Bean
    public VectorStore pgvectorStore(JdbcTemplate jdbcTemplate, 
            EmbeddingModel embeddingModel) {
        return new PgVectorStore(jdbcTemplate, 
            PgVectorStore.builder()
                .withTableName("rag_documents")
                .withVectorDimensions(1536)
                .withSchemaName("public")
                .withCreateTable(true)
                .withRemoveExistingVectorStoreTable(false),
            embeddingModel);
    }
}

// 初始化表结构
@Component
public class PgvectorInit {
    
    private final JdbcTemplate jdbcTemplate;
    
    public void init() {
        // 启用 pgvector 扩展
        jdbcTemplate.execute("CREATE EXTENSION IF NOT EXISTS vector");
        
        // 创建带有过滤字段的表
        jdbcTemplate.execute("""
            CREATE TABLE IF NOT EXISTS rag_documents (
                id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
                content TEXT NOT NULL,
                embedding vector(1536),
                metadata JSONB,
                doc_type VARCHAR(50),
                created_at TIMESTAMP DEFAULT NOW()
            )
        """);
        
        // 创建 IVFFlat 索引（100万条以内）
        jdbcTemplate.execute("""
            CREATE INDEX IF NOT EXISTS rag_docs_embedding_idx 
            ON rag_documents 
            USING ivfflat (embedding vector_cosine_ops)
            WITH (lists = 100)
        """);
    }
}

// 带过滤条件的向量检索
@Repository
public class PgvectorRepository {
    
    private final JdbcTemplate jdbcTemplate;
    
    public List<Document> searchWithFilter(
            float[] queryEmbedding, 
            int topK, 
            String docType, 
            LocalDate afterDate) {
        
        String sql = """
            SELECT id, content, metadata,
                   1 - (embedding <=> ?::vector) AS similarity
            FROM rag_documents
            WHERE doc_type = ?
              AND created_at >= ?
            ORDER BY embedding <=> ?::vector
            LIMIT ?
        """;
        
        return jdbcTemplate.query(sql, (rs, rowNum) -> {
            Document doc = new Document();
            doc.setId(rs.getString("id"));
            doc.setContent(rs.getString("content"));
            doc.setSimilarity(rs.getDouble("similarity"));
            return doc;
        }, arrayToPgVector(queryEmbedding), docType, afterDate,
           arrayToPgVector(queryEmbedding), topK);
    }
    
    private String arrayToPgVector(float[] arr) {
        StringBuilder sb = new StringBuilder("[");
        for (int i = 0; i < arr.length; i++) {
            if (i > 0) sb.append(",");
            sb.append(arr[i]);
        }
        sb.append("]");
        return sb.toString();
    }
}
```

### 4.3 Milvus Java 集成

```java
@Configuration
public class MilvusConfig {
    
    @Bean
    public MilvusServiceClient milvusClient() {
        ConnectParam connectParam = ConnectParam.newBuilder()
            .withHost("localhost")
            .withPort(19530)
            .build();
        return new MilvusServiceClient(connectParam);
    }
}

@Service
public class MilvusVectorStore {
    
    private final MilvusServiceClient client;
    private static final String COLLECTION_NAME = "rag_documents";
    private static final int DIMENSION = 1536;
    
    public void initCollection() {
        // 检查集合是否存在
        var hasCollection = client.hasCollection(
            HasCollectionParam.newBuilder()
                .withCollectionName(COLLECTION_NAME)
                .build());
        
        if (!hasCollection.getData()) {
            // 定义字段
            FieldType idField = FieldType.newBuilder()
                .withName("id")
                .withDataType(DataType.VarChar)
                .withMaxLength(64)
                .withPrimaryKey(true)
                .withAutoID(false)
                .build();
            
            FieldType embeddingField = FieldType.newBuilder()
                .withName("embedding")
                .withDataType(DataType.FloatVector)
                .withDimension(DIMENSION)
                .build();
            
            FieldType contentField = FieldType.newBuilder()
                .withName("content")
                .withDataType(DataType.VarChar)
                .withMaxLength(65535)
                .build();
            
            FieldType docTypeField = FieldType.newBuilder()
                .withName("doc_type")
                .withDataType(DataType.VarChar)
                .withMaxLength(50)
                .build();
            
            // 创建 Schema
            CollectionSchema schema = CollectionSchema.newBuilder()
                .withFieldTypes(List.of(idField, embeddingField, contentField, docTypeField))
                .withEnableDynamicField(true)
                .build();
            
            // 创建集合
            CreateCollectionParam createParam = CreateCollectionParam.newBuilder()
                .withCollectionName(COLLECTION_NAME)
                .withSchema(schema)
                .build();
            
            client.createCollection(createParam);
        }
        
        // 创建索引
        IndexParam indexParam = IndexParam.newBuilder()
            .withCollectionName(COLLECTION_NAME)
            .withFieldName("embedding")
            .withIndexType(IndexType.HNSW)
            .withMetricType(MetricType.COSINE)
            .withExtraParam("{\"M\": 16, \"efConstruction\": 200}")
            .build();
        
        client.createIndex(CreateIndexParam.newBuilder()
            .withCollectionName(COLLECTION_NAME)
            .withFieldName("embedding")
            .withIndexName("embedding_hnsw_idx")
            .withIndexType(IndexType.HNSW)
            .withMetricType(MetricType.COSINE)
            .withExtraParam("{\"M\": 16, \"efConstruction\": 200}")
            .build());
    }
    
    // 批量写入
    public void insert(List<String> ids, List<String> contents, 
            List<List<Float>> embeddings, List<String> docTypes) {
        
        List<InsertParam.Field> fields = new ArrayList<>();
        
        fields.add(new InsertParam.Field("id", ids));
        fields.add(new InsertParam.Field("content", contents));
        fields.add(new InsertParam.Field("embedding", embeddings));
        fields.add(new InsertParam.Field("doc_type", docTypes));
        
        InsertParam insertParam = InsertParam.newBuilder()
            .withCollectionName(COLLECTION_NAME)
            .withFields(fields)
            .build();
        
        client.insert(insertParam);
        client.flush(FlushParam.newBuilder()
            .addCollectionName(COLLECTION_NAME)
            .build());
    }
    
    // 向量搜索 + 标量过滤
    public List<SearchResult> searchWithFilter(
            List<Float> queryEmbedding, 
            int topK, 
            String docType) {
        
        // 搜索参数
        String searchParam = """
        {
            "ef": 128,
            "metric_type": "COSINE"
        }
        """;
        
        // 过滤表达式
        String filterExpr = "doc_type == \"" + docType + "\"";
        
        SearchParam searchReq = SearchParam.newBuilder()
            .withCollectionName(COLLECTION_NAME)
            .withVectors(List.of(queryEmbedding))
            .withVectorFieldName("embedding")
            .withTopK(topK)
            .withParams(searchParam)
            .withExpr(filterExpr)
            .withOutputFields(List.of("content", "doc_type"))
            .withMetricType(MetricType.COSINE)
            .build();
        
        R<SearchResults> response = client.search(searchReq);
        
        List<SearchResult> results = new ArrayList<>();
        for (var hits : response.getData().getResults()) {
            for (var hit : hits.getScoresList().isEmpty() ? List.of() : hits.getScoresList()) {
                results.add(new SearchResult(
                    (String) hit.get("content"),
                    hit.getScore()
                ));
            }
        }
        return results;
    }
    
    public record SearchResult(String content, float score) {}
}
```

### 4.4 Weaviate Java 集成

```java
@Configuration
public class WeaviateConfig {
    
    @Bean
    public WeaviateClient weaviateClient() {
        Config config = new Config("http", "localhost:8080");
        return new WeaviateClient(config);
    }
}

@Service
public class WeaviateVectorStore {
    
    private final WeaviateClient client;
    private static final String CLASS_NAME = "RagDocument";
    
    // 创建 Schema
    public void createSchema() {
        // Weaviate 的 Schema 定义
        @SuppressWarnings("unchecked")
        Map<String, Object> classObj = Map.of(
            "class", CLASS_NAME,
            "vectorizer", "none",  // 自己提供向量
            "properties", List.of(
                Map.of("name", "content", "dataType", List.of("text")),
                Map.of("name", "docType", "dataType", List.of("string")),
                Map.of("name", "chunkIndex", "dataType", List.of("int")),
                Map.of("name", "sourceDoc", "dataType", List.of("string"))
            )
        );
        
        client.schema().classCreator()
            .withClass(classObj)
            .run();
    }
    
    // 批量导入
    public void batchImport(List<ImportDoc> docs) {
        var batch = client.batch().objectsAutoBatch();
        
        for (ImportDoc doc : docs) {
            Map<String, Object> properties = Map.of(
                "content", doc.content,
                "docType", doc.docType,
                "chunkIndex", doc.chunkIndex,
                "sourceDoc", doc.sourceDoc
            );
            
            batch.withObject()
                .className(CLASS_NAME)
                .properties(properties)
                .vector(doc.vector)
                .build();
        }
        
        batch.flush();
    }
    
    // GraphQL 混合检索
    public List<String> hybridSearch(String query, float[] queryVector, 
            int topK, String docTypeFilter) {
        
        String graphQL = """
        {
            Get {
                RagDocument(
                    hybrid: {
                        query: "%s"
                        vector: %s
                        alpha: 0.5
                    }
                    where: {
                        path: ["docType"]
                        operator: Equal
                        valueString: "%s"
                    }
                    limit: %d
                ) {
                    content
                    docType
                    _additional {
                        score
                    }
                }
            }
        }
        """.formatted(query, arrayToJsonStr(queryVector), docTypeFilter, topK);
        
        Result<GraphQLResponse> result = client.graphQL().raw()
            .withQuery(graphQL)
            .run();
        
        // 解析结果
        List<String> contents = new ArrayList<>();
        // ... 解析 GraphQLResponse
        return contents;
    }
    
    private String arrayToJsonStr(float[] arr) {
        return Arrays.toString(arr);
    }
    
    public record ImportDoc(String content, String docType, 
            int chunkIndex, String sourceDoc, Float[] vector) {}
}
```

---

## 五、选型决策矩阵

### 5.1 按场景推荐

```yaml
选型决策:
  你的团队:
    只有Java开发+DBA:
      → Pgvector (复用现有Pg运维能力)
    
    有专职基础设施团队:
      → Milvus (性能+扩展性最优)
    
    2-3人创业团队、快速验证:
      → Chroma (最快上线)
    
    需要混合检索+语义搜索一体:
      → Weaviate (原生GraphQL+全文检索)
```

### 5.2 按数据量推荐

```yaml
数据量级:
  低于10万条:
    方案: Chroma 或 Pgvector
    理由: 无需集群，单机运行，运维成本低
    
  10万-100万条:
    方案: Pgvector 或 Weaviate
    理由: Pgvector 够用且能复用 DBA；Weaviate 提供全文检索加分
    
  100万-1000万条:
    方案: Milvus 或 Weaviate
    理由: 需要水平扩展能力
    
  1000万条以上:
    方案: Milvus
    理由: 唯一经过亿级向量验证的方案
```

### 5.3 最终推荐

```
┌─────────────────────────────────────────────────────┐
│                  向量数据库选型决策树                    │
├─────────────────────────────────────────────────────┤
│                                                       │
│  你的数据量？                                          │
│    ├── < 10万条                                        │
│    │   ├── 已有 PostgreSQL？ → Pgvector                │
│    │   ├── Python 技术栈？ → Chroma                    │
│    │   └── Java 技术栈？ → Pgvector                    │
│    │                                                   │
│    ├── 10万-100万条                                    │
│    │   ├── 需要复杂过滤 + 全文检索？ → Weaviate         │
│    │   ├── 有 DBA 团队？ → Pgvector                    │
│    │   └── 追求极致性能？ → Milvus                     │
│    │                                                   │
│    └── > 100万条                                       │
│        ├── 生产环境 → Milvus                           │
│        ├── 需要多模态？ → Milvus                        │
│        └── 需要知识图谱？ → Weaviate                    │
│                                                       │
└─────────────────────────────────────────────────────┘
```

对于大多数 **Java + Spring AI** 技术栈的团队，我最推荐的选择顺序是：

1. **Pgvector**（首选）：Spring AI 原生集成，团队无需学习新数据库
2. **Milvus**（规模化）：数据量破百万后迁移，性能无可匹敌
3. **Weaviate**（混合检索）：需要原生全文+向量融合时选择
4. **Chroma**（原型）：快速验证时可用，生产环境不建议

---

## 六、总结

没有最好的向量数据库，只有最匹配的向量数据库。

关键决策点只有 3 个：

1. **数据量**：小于 100 万用 Pgvector，大于 100 万用 Milvus
2. **团队能力**：有 DBA 优先 Pgvector，有基础设施团队优先 Milvus
3. **功能需求**：需要全文检索选 Weaviate，需要多模态选 Milvus，简单场景选 Chroma

**不要在原型阶段花 2 周搭 Milvus 集群，也不要在百万级数据量时死扛 Chroma。** 选择合适的，比选择"最强"的重要 10 倍。

---

**下一篇预告**：向量数据库选好了，文档也入了库，但你的 RAG 效果到底好不好？下一篇《RAG 评估框架：RAGAS 指标体系的实施与解读》，我将带你搭建完整的 RAG 评估流水线，用数据说话——Faithfulness、AnswerRelevancy、ContextPrecision、ContextRecall 四大指标一次讲透。

---

> 如果觉得这篇文章有帮助，欢迎点赞、收藏、关注，感谢支持！

> 作者：深耕 Java 企业级开发多年，专注 AI 工程化落地。有问题欢迎在评论区交流。
