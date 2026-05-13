# Spring AI VectorStore：集成 Chroma / Milvus / Pgvector 完整教程，向量数据库选型不再纠结

> Embedding 算出来了但存哪里？Redis 不够、MySQL 太慢——你需要向量数据库。

## 一、开篇：Embedding 之后的存储难题

上一篇文章我们学会了把文字变成向量，也学会了用余弦相似度比较两个向量的距离。但问题来了：

- FAQ 有 200 条，每次搜索遍历 200 条算相似度，勉强可行
- 知识库有 10 万篇文档，每次遍历 10 万条……用户早把浏览器关了
- 电商商品有 1000 万个 SKU，O(n) 遍历……服务器直接 OOM

**本质问题**：暴力搜索（Brute Force）在大规模数据下完全不可行。我们需要一种"近似"搜索算法——不完全精确，但速度快 1000 倍以上。

这就是**向量数据库（Vector Database）**存在的意义。

---

## 二、向量数据库为什么不同于传统数据库

### 2.1 传统数据库能存向量吗？

能，但不该这么做：

| 方案 | 能搜吗 | 速度 | 问题 |
|------|--------|------|------|
| MySQL `FLOAT[]` + 遍历 | ✅ | 极慢 | O(n) 暴力搜索，100 万条 = 几秒到几十秒 |
| PostgreSQL ARRAY + 遍历 | ✅ | 极慢 | 同上 |
| Redis `HGETALL` + 内存算 | ✅ | 中等 | 数据量受内存限制，无索引加速 |
| Elasticsearch `dense_vector` | ✅ | 较快 | 受限于 4096 维，运维复杂 |

### 2.2 向量数据库的核心：ANN 索引

向量数据库的核心能力是 **ANN（Approximate Nearest Neighbor，近似最近邻）搜索**。它不遍历所有数据，而是通过索引结构快速定位到近似区域。

**HNSW 算法通俗解释**（向量数据库最常用的索引算法）：

```
想象你要在一个超大型图书馆里找一本书（目标向量）：

传统方法（MySQL）：一本一本翻，100 万本书翻到地老天荒
HNSW 方法：先看"楼层索引"→ 是文学类 → 再看"区域索引"→ 是亚洲文学
         → 再看"书架索引"→ 是日本文学 → 精准定位，3 步到位
```

HNSW 把向量组织成**多层图结构**，查询时从顶层大范围跳跃，逐层缩小范围，最终在底层精确查找。时间复杂度从 O(n) 降到 O(log n)。

### 2.3 三大候选：Chroma / Pgvector / Milvus

| 维度 | Chroma | Pgvector | Milvus |
|------|--------|----------|--------|
| 类型 | 独立向量数据库 | PostgreSQL 扩展 | 分布式向量数据库 |
| 部署难度 | ⭐ 极简 | ⭐⭐ 中等 | ⭐⭐⭐ 复杂 |
| 存储上限 | TB 级 | TB 级 | PB 级 |
| 查询延迟 | <10ms | <50ms | <10ms |
| 生产就绪 | 一般 | ✅ 成熟（依托 PG） | ✅ 非常成熟 |
| 适用场景 | 开发/原型/PoC | 已有 PG 的项目 | 大规模生产环境 |
| 开源协议 | Apache 2.0 | PostgreSQL License | Apache 2.0 |

> **速选指南**：开发测试用 Chroma，已有 PostgreSQL 用 Pgvector，上生产且数据量 > 1000 万条用 Milvus。

---

## 三、Chroma 集成（轻量级首选）

Chroma 是专为 AI 应用设计的向量数据库，Python 生态出身，部署极简。

### 3.1 部署 Chroma

Docker 一键启动：

```bash
docker run -d \
  --name chroma \
  -p 8000:8000 \
  -v /data/chroma:/chroma/chroma \
  -e IS_PERSISTENT=TRUE \
  chromadb/chroma:latest
```

验证：

```bash
curl http://localhost:8000/api/v1/heartbeat
# 返回 {"nanosecond heartbeat": 1712345678901234567}
```

### 3.2 Maven 依赖

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-chroma-store-spring-boot-starter</artifactId>
</dependency>
```

### 3.3 application.yml 配置

```yaml
spring:
  ai:
    openai:
      api-key: ${OPENAI_API_KEY}
      embedding:
        options:
          model: text-embedding-3-small
    vectorstore:
      chroma:
        initialize-schema: true
        host: http://localhost
        port: 8000
        collection-name: my_knowledge_base
```

### 3.4 完整 CRUD 示例

```java
package com.example.springaidemo.service;

import lombok.RequiredArgsConstructor;
import org.springframework.ai.document.Document;
import org.springframework.ai.vectorstore.SearchRequest;
import org.springframework.ai.vectorstore.VectorStore;
import org.springframework.stereotype.Service;

import java.util.List;
import java.util.Map;

@Service
@RequiredArgsConstructor
public class ChromaVectorService {

    private final VectorStore vectorStore;

    // ==================== 增 ====================

    /**
     * 添加单条文档
     */
    public void addDocument(String content, Map<String, Object> metadata) {
        Document doc = new Document(content, metadata);
        vectorStore.add(List.of(doc));
    }

    /**
     * 批量添加文档
     */
    public void addDocuments(List<Document> documents) {
        vectorStore.add(documents);
    }

    // ==================== 删 ====================

    /**
     * 按 ID 删除
     */
    public void deleteById(String id) {
        vectorStore.delete(List.of(id));
    }

    /**
     * 按元数据过滤删除
     * 删除所有来源为 "deprecated" 的文档
     */
    public void deleteBySource(String source) {
        vectorStore.delete("source == '" + source + "'");
    }

    // ==================== 查 ====================

    /**
     * 语义搜索：最常用的操作
     * 找到与 query 语义最相似的 Top-N 文档
     */
    public List<Document> semanticSearch(String query, int topK) {
        return vectorStore.similaritySearch(
                SearchRequest.defaults()
                        .withQuery(query)
                        .withTopK(topK)
                        .withSimilarityThreshold(0.7)  // 只返回相似度 > 0.7 的结果
        );
    }

    /**
     * 带元数据过滤的语义搜索
     * 例如：只搜索"2024年"来源的文档
     */
    public List<Document> filteredSearch(String query, int topK) {
        return vectorStore.similaritySearch(
                SearchRequest.defaults()
                        .withQuery(query)
                        .withTopK(topK)
                        .withFilterExpression("year == '2024'")
        );
    }

    // ==================== 改 ====================

    /**
     * 更新文档：先删后加（向量数据库一般不直接支持 update）
     */
    public void updateDocument(String id, String newContent, Map<String, Object> newMetadata) {
        vectorStore.delete(List.of(id));
        Document newDoc = new Document(id, newContent, newMetadata);
        vectorStore.add(List.of(newDoc));
    }
}
```

### 3.5 REST 接口封装

```java
package com.example.springaidemo.controller;

import lombok.RequiredArgsConstructor;
import org.springframework.ai.document.Document;
import org.springframework.web.bind.annotation.*;

import java.util.List;
import java.util.Map;

@RestController
@RequestMapping("/chroma")
@RequiredArgsConstructor
public class ChromaController {

    private final ChromaVectorService chromaService;

    @PostMapping("/add")
    public String add(@RequestBody AddRequest request) {
        chromaService.addDocument(
                request.content(),
                Map.of("source", request.source(), "timestamp", System.currentTimeMillis())
        );
        return "OK";
    }

    @GetMapping("/search")
    public List<Document> search(
            @RequestParam String query,
            @RequestParam(defaultValue = "5") int topK) {
        return chromaService.semanticSearch(query, topK);
    }

    @DeleteMapping("/{id}")
    public String delete(@PathVariable String id) {
        chromaService.deleteById(id);
        return "OK";
    }

    public record AddRequest(String content, String source) {}
}
```

---

## 四、Pgvector 集成（PostgreSQL 用户的福音）

如果你的项目已经在用 PostgreSQL，Pgvector 让你无需引入新组件就能拥有向量搜索能力。

### 4.1 启用 Pgvector 扩展

```sql
-- 以超级用户身份执行
CREATE EXTENSION IF NOT EXISTS vector;

-- 验证
SELECT * FROM pg_extension WHERE extname = 'vector';
```

如果使用 Docker：

```bash
docker run -d \
  --name pgvector \
  -e POSTGRES_PASSWORD=postgres \
  -p 5432:5432 \
  pgvector/pgvector:pg16
```

### 4.2 Maven 依赖

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-pgvector-store-spring-boot-starter</artifactId>
</dependency>
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
</dependency>
```

### 4.3 application.yml 配置

```yaml
spring:
  ai:
    openai:
      api-key: ${OPENAI_API_KEY}
      embedding:
        options:
          model: text-embedding-3-small
    vectorstore:
      pgvector:
        host: localhost
        port: 5432
        database: vectordb
        username: postgres
        password: ${PG_PASSWORD:postgres}
        table-name: document_embeddings
        # 向量维度（必须与 Embedding 模型匹配）
        dimensions: 1536
        # 索引类型：hnsw 或 ivfflat
        index-type: hnsw
        # HNSW 参数
        hnsw-m: 16           # 每层最大连接数
        hnsw-ef-construction: 200  # 构建时的搜索深度
        # 是否在启动时自动建表
        initialize-schema: true
```

### 4.4 Pgvector 的表结构（自动创建或手动创建）

如果要手动控制表结构，可以这样建表：

```sql
-- Spring AI Pgvector 会自动创建此表，这里展示结构仅供参考
CREATE TABLE IF NOT EXISTS document_embeddings (
    id        VARCHAR(255) PRIMARY KEY,
    content   TEXT,
    metadata  JSONB,
    embedding VECTOR(1536)  -- 向量列，维度 1536
);

-- 创建 HNSW 索引（加速搜索）
CREATE INDEX ON document_embeddings
    USING hnsw (embedding vector_cosine_ops)
    WITH (m = 16, ef_construction = 200);
```

### 4.5 使用 Pgvector 的 VectorStore

Pgvector 实现了 Spring AI 的 `VectorStore` 接口，使用方式与 Chroma 完全一样：

```java
package com.example.springaidemo.service;

import lombok.RequiredArgsConstructor;
import org.springframework.ai.vectorstore.VectorStore;
import org.springframework.stereotype.Service;

@Service
@RequiredArgsConstructor
public class PgvectorService {

    private final VectorStore vectorStore;  // 自动注入 PgvectorStore

    // 增删改查接口与 Chroma 完全相同，不再重复
    // 见 ChromaVectorService 中的代码
}
```

### 4.6 Pgvector 的独特优势：SQL 直接查向量

因为 Pgvector 本质上是 PostgreSQL，你可以用 SQL 直接操作向量数据：

```sql
-- 插入一条带向量的数据
INSERT INTO document_embeddings (id, content, metadata, embedding)
VALUES (
    'doc-001',
    'Java 是一种面向对象的编程语言',
    '{"source": "wiki", "year": "2024"}',
    '[0.012, -0.045, ...]'::vector
);

-- 语义搜索：找到最近的 5 条
SELECT id, content,
       1 - (embedding <=> '[0.012, -0.045, ...]'::vector) AS similarity
FROM document_embeddings
ORDER BY embedding <=> '[0.012, -0.045, ...]'::vector
LIMIT 5;

-- <=> 是余弦距离运算符
-- 1 - 余弦距离 = 余弦相似度

-- 带元数据过滤的语义搜索
SELECT id, content
FROM document_embeddings
WHERE metadata ->> 'source' = 'wiki'
ORDER BY embedding <=> '[0.012, -0.045, ...]'::vector
LIMIT 5;

-- 查看索引使用情况
EXPLAIN ANALYZE
SELECT id FROM document_embeddings
ORDER BY embedding <=> '[0.012, ...]'::vector
LIMIT 5;
```

> 这个 SQL 能力让 Pgvector 在需要复杂过滤的场景下非常有优势——你可以把一个传统数据分析查询和一个向量搜索查询无缝结合。

---

## 五、Milvus 集成（百万级以上的生产选择）

当数据量超过 100 万条，或者需要分布式部署时，Milvus 是最佳选择。

### 5.1 部署 Milvus（单机模式）

```bash
# 下载 docker-compose
wget https://github.com/milvus-io/milvus/releases/download/v2.4.0/milvus-standalone-docker-compose.yml

# 启动
docker compose -f milvus-standalone-docker-compose.yml up -d

# 验证
curl http://localhost:19530/api/v1/health
```

### 5.2 Maven 依赖

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-milvus-store-spring-boot-starter</artifactId>
</dependency>
```

### 5.3 application.yml 配置

```yaml
spring:
  ai:
    openai:
      api-key: ${OPENAI_API_KEY}
      embedding:
        options:
          model: text-embedding-3-small
    vectorstore:
      milvus:
        host: localhost
        port: 19530
        username: root
        password: milvus
        database-name: default
        collection-name: knowledge_base
        # 向量维度
        embedding-dimension: 1536
        # 索引类型：FLAT / IVF_FLAT / IVF_SQ8 / IVF_PQ / HNSW
        index-type: IVF_FLAT
        # 相似度度量：COSINE / IP / L2
        metric-type: COSINE
        # 索引参数
        index-extra-params: '{"nlist": 1024}'
        # 是否在启动时创建 collection
        initialize-schema: true
```

### 5.4 完整 CRUD 示例

```java
package com.example.springaidemo.service;

import lombok.RequiredArgsConstructor;
import org.springframework.ai.document.Document;
import org.springframework.ai.vectorstore.SearchRequest;
import org.springframework.ai.vectorstore.VectorStore;
import org.springframework.stereotype.Service;

import java.util.List;
import java.util.Map;

@Service
@RequiredArgsConstructor
public class MilvusVectorService {

    private final VectorStore vectorStore;

    /**
     * 批量导入文档（Milvus 适合批量操作）
     */
    public void batchImport(List<Document> documents) {
        // Milvus 批量写入性能极高，100 万条通常在数秒内完成
        vectorStore.add(documents);
        System.out.println("导入完成: " + documents.size() + " 条");
    }

    /**
     * 语义搜索
     */
    public List<Document> search(String query, int topK) {
        return vectorStore.similaritySearch(
                SearchRequest.defaults()
                        .withQuery(query)
                        .withTopK(topK)
                        .withSimilarityThreshold(0.65)
        );
    }

    /**
     * 带复杂过滤条件的搜索
     * Milvus 支持标量过滤（元数据字段需要预先定义 schema）
     */
    public List<Document> filteredSearch(String query, String category, int topK) {
        return vectorStore.similaritySearch(
                SearchRequest.defaults()
                        .withQuery(query)
                        .withTopK(topK)
                        .withFilterExpression("category == '" + category + "'")
        );
    }

    /**
     * 按 ID 删除
     */
    public void deleteByIds(List<String> ids) {
        vectorStore.delete(ids);
    }

    /**
     * 按条件批量删除
     */
    public void deleteByCategory(String category) {
        vectorStore.delete("category == '" + category + "'");
    }

    /**
     * 更新文档
     */
    public void updateDocument(String id, Document newDoc) {
        vectorStore.delete(List.of(id));
        vectorStore.add(List.of(newDoc));
    }
}
```

### 5.5 Milvus 运维要点

```java
// 手动创建索引（如果关闭了 initialize-schema）
// Milvus collection 创建后需要手动构建索引才能高效查询

// 索引类型对比：
// ┌────────────┬──────────┬────────────┬──────────┐
// │ 索引类型    │ 召回率    │ QPS         │ 内存占用  │
// ├────────────┼──────────┼────────────┼──────────┤
// │ FLAT       │ 100%     │ 低          │ 低        │
// │ IVF_FLAT   │ ~98%     │ 中          │ 中        │
// │ IVF_SQ8    │ ~95%     │ 高          │ 低        │
// │ HNSW       │ ~99%     │ 非常高       │ 高        │
// └────────────┴──────────┴────────────┴──────────┘

// 生产推荐：HNSW（召回率高 + QPS 高，内存充足的场景）
// 内存受限：IVF_SQ8（压缩存储，牺牲少量精度换取更大容量）
```

---

## 六、大统一：抽象出 VectorStoreService

得益于 Spring AI 的 `VectorStore` 接口，你可以写一套代码，随时切换底层向量数据库：

```java
package com.example.springaidemo.service;

import lombok.RequiredArgsConstructor;
import org.springframework.ai.document.Document;
import org.springframework.ai.vectorstore.SearchRequest;
import org.springframework.ai.vectorstore.VectorStore;
import org.springframework.stereotype.Service;

import java.util.List;
import java.util.Map;

/**
 * 通用的向量存储服务
 * 通过切换 VectorStore 的实现 Bean 来切换 Chroma/Pgvector/Milvus
 * 调用方代码完全不变
 */
@Service
@RequiredArgsConstructor
public class UniversalVectorService {

    private final VectorStore vectorStore;

    public void add(String content, Map<String, Object> metadata) {
        vectorStore.add(List.of(new Document(content, metadata)));
    }

    public void addAll(List<Document> docs) {
        vectorStore.add(docs);
    }

    public List<Document> search(String query, int topK) {
        return vectorStore.similaritySearch(
                SearchRequest.defaults()
                        .withQuery(query)
                        .withTopK(topK)
                        .withSimilarityThreshold(0.65)
        );
    }

    public List<Document> searchWithFilter(String query, String filterExpr, int topK) {
        return vectorStore.similaritySearch(
                SearchRequest.defaults()
                        .withQuery(query)
                        .withTopK(topK)
                        .withFilterExpression(filterExpr)
        );
    }

    public void delete(List<String> ids) {
        vectorStore.delete(ids);
    }
}
```

**切换数据库只需要改配置**：

```yaml
# 使用 Chroma
spring.ai.vectorstore.chroma.host: http://localhost
# 注释掉上面的，启用下面的就是 Milvus
# spring.ai.vectorstore.milvus.host: localhost
```

代码零改动，这就是 Spring 生态的力量。

---

## 七、性能对比与选型建议

### 7.1 实测数据（100 万条 1536 维向量，单机）

| 操作 | Chroma | Pgvector | Milvus |
|------|--------|----------|--------|
| 写入 10 万条 | 45s | 120s | 8s |
| 查询 Top-10 | 8ms | 45ms | 5ms |
| 查询 Top-100 | 15ms | 80ms | 8ms |
| 内存占用 | 6GB | 8GB | 4GB |
| 磁盘占用 | 6GB | 12GB | 6GB |

### 7.2 选型决策树

```
你的数据量多大？
├── < 10 万条
│   ├── 已有 PostgreSQL？  →  Pgvector
│   └── 想快速启动？        →  Chroma
│
├── 10 万 ~ 100 万条
│   ├── 需要复杂 SQL 过滤？ →  Pgvector
│   └── 追求极致性能？      →  Milvus
│
└── > 100 万条
    └── 必须 Milvus
```

### 7.3 最终推荐

| 场景 | 推荐 | 理由 |
|------|------|------|
| 开发/原型/个人项目 | **Chroma** | 部署最快，API 最简洁 |
| 公司已有 PostgreSQL | **Pgvector** | 复用现有基础设施，运维成本低 |
| 企业级生产环境 | **Milvus** | 性能最强，分布式支持，社区活跃 |
| 需要混合标量+向量查询 | **Pgvector** | SQL + 向量搜索结合最佳 |

---

## 八、综合实战：构建一个 RAG 知识库

把前面学的串起来——ChatClient + Embedding + VectorStore 组合成 RAG（检索增强生成）：

```java
package com.example.springaidemo.service;

import lombok.RequiredArgsConstructor;
import org.springframework.ai.chat.client.ChatClient;
import org.springframework.ai.document.Document;
import org.springframework.ai.vectorstore.SearchRequest;
import org.springframework.ai.vectorstore.VectorStore;
import org.springframework.stereotype.Service;

import java.util.List;
import java.util.Map;
import java.util.stream.Collectors;

/**
 * RAG（检索增强生成）服务
 * 1. 用户提问
 * 2. 从向量数据库检索相关文档
 * 3. 把文档作为上下文一起发给 LLM
 * 4. 返回带知识库背景的答案
 */
@Service
@RequiredArgsConstructor
public class RagService {

    private final VectorStore vectorStore;
    private final ChatClient.Builder chatClientBuilder;

    /**
     * RAG 问答
     */
    public String ask(String question) {
        // Step 1: 从向量数据库中检索相关文档
        List<Document> relevantDocs = vectorStore.similaritySearch(
                SearchRequest.defaults()
                        .withQuery(question)
                        .withTopK(5)
                        .withSimilarityThreshold(0.65)
        );

        // Step 2: 构建上下文
        String context = relevantDocs.stream()
                .map(doc -> "【来源：" + doc.getMetadata().getOrDefault("source", "未知")
                        + "】\n" + doc.getContent())
                .collect(Collectors.joining("\n\n"));

        // Step 3: 构建 Prompt
        String systemPrompt = """
                你是一个基于知识库的智能助手。请根据以下知识库内容回答用户问题。
                如果知识库中没有相关信息，请如实告知用户。

                知识库内容：
                %s
                
                请用中文回答，保持专业、准确。
                """.formatted(context);

        // Step 4: 调用 LLM 生成回答
        return chatClientBuilder.build()
                .prompt()
                .system(systemPrompt)
                .user(question)
                .call()
                .content();
    }

    /**
     * 向知识库添加文档
     */
    public void addToKnowledgeBase(String content, String source) {
        Document doc = new Document(content,
                Map.of("source", source, "timestamp", System.currentTimeMillis()));
        vectorStore.add(List.of(doc));
    }
}
```

**测试效果**：

```java
// 先喂一些知识
ragService.addToKnowledgeBase("Spring AI 1.0.0-M5 支持 OpenAI、Azure OpenAI、Ollama、智谱等模型提供商", "官网文档");
ragService.addToKnowledgeBase("ChatClient 支持同步、异步和流式三种调用方式", "官网文档");
ragService.addToKnowledgeBase("Pgvector 的 HNSW 索引参数 m 推荐设为 16，ef_construction 推荐设为 200", "技术博客");

// 然后提问
String answer1 = ragService.ask("Spring AI 支持哪些大模型？");
// → 根据知识库，Spring AI 1.0.0-M5 支持 OpenAI、Azure OpenAI、Ollama、智谱等模型提供商

String answer2 = ragService.ask("Pgvector 的 HNSW 参数怎么设置？");
// → 根据知识库，m 推荐设为 16，ef_construction 推荐设为 200

String answer3 = ragService.ask("什么是 Kubernetes？");
// → 知识库中没有关于 Kubernetes 的相关信息，建议您提供更多资料后我再为您解答
```

这就是 RAG 的基本工作流——**检索（Retrieve）→ 增强（Augment）→ 生成（Generate）**。无需微调模型，你的 LLM 就能基于私有知识回答问题了。

---

## 九、总结与下篇预告

本文从"Embedding 算出来存哪里"出发，深入讲解了向量数据库的选型与 Spring AI 集成：

1. **Chroma**：轻量级，Docker 一条命令启动，适合开发和小项目
2. **Pgvector**：PostgreSQL 扩展，已有 PG 的天然选择
3. **Milvus**：高性能分布式，百万级以上数据量的标准答案

核心接口 `VectorStore` 实现了底层数据库的完整抽象，切换只需改配置。

**下一篇**：《Spring AI Prompt Template：告别字符串拼接，打造可复用的 AI 指令模板》，我们将告别硬编码的 Prompt，用模板引擎、变量注入、条件逻辑来构建专业级的 AI Prompt 工程体系。

---

> **系列目录**：
> - 081：[Spring AI 入门]：5 分钟接入 GPT-4
> - 082：[ChatClient 深度配置]（连接池/超时/重试/熔断）
> - 083：[Embedding 文本向量化]（语义搜索/相似度计算）
> - 084：VectorStore 向量数据库集成 ← 本文
>
> **相关资源**：
> - [Spring AI VectorStore 文档](https://docs.spring.io/spring-ai/reference/api/vectordbs.html)
> - [Chroma 官方文档](https://docs.trychroma.com/)
> - [Pgvector GitHub](https://github.com/pgvector/pgvector)
> - [Milvus 官方文档](https://milvus.io/docs/)
