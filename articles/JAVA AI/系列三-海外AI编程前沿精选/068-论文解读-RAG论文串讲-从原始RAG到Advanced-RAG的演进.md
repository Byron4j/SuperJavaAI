# [论文解读] RAG 论文串讲：从原始 RAG 到 Advanced RAG 的演进，读完这3篇论文就懂了

> RAG不是一个技术，是一棵不断生长的技术树。从2020到2025，RAG已经演化了4代。读完这3篇关键论文，你就掌握了RAG的发展脉络。

---

## 一、开篇：什么是RAG，为什么要关注它？

**RAG = Retrieval-Augmented Generation（检索增强生成）**。

大语言模型有个天然缺陷：**知识截止日期**。训练数据只到某个时间点，之后发生的事它不可能知道。更致命的是，它有时会"一本正经地胡说八道"——这就是幻觉（Hallucination）。

RAG的思路非常朴素：**让LLM在回答之前，先去外部知识库"查一下"**，然后用查到的资料辅助回答。

这就像你面试时，允许你现场用搜索引擎——你的回答质量和可信度会大幅提升。

---

## 二、第一代：原始RAG（Lewis et al., 2020）

### 2.1 论文核心

**论文**：Lewis, P., et al. (2020). *Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks.* NeurIPS 2020.  
**核心思想**：将检索器（Retriever）和生成器（Generator）端到端联合训练。

### 2.2 架构图解

```
用户问题 ──→ 检索器(DPR) ──→ 相关文档Top-K ──→ 生成器(BART) ──→ 回答
                ↑                                    ↑
         Dense向量检索                         把文档拼到Prompt里
```

### 2.3 关键技术点

**1. Dense Passage Retrieval (DPR)**  
不同于传统的TF-IDF/BM25关键词匹配，DPR用两个BERT分别将问题和文档编码为稠密向量，通过向量相似度检索。本质是**语义匹配**而非**关键词匹配**。

**2. 联合训练**  
检索器和生成器一起训练，而不是各自独立。这样检索器学会"找对生成有帮助的文档"。

### 2.4 Java实现可行性分析

```java
/**
 * 原始RAG的核心流程 —— Java实现分析
 * 
 * 可行性：✅ 完全可以
 * 关键组件：
 *   1. 向量数据库（Milvus/Qdrant/Pinecone 都有Java SDK）
 *   2. Embedding API（OpenAI/Cohere/本地模型都有Java客户端）
 *   3. LLM调用（Spring AI / LangChain4j 已成熟）
 */

public class OriginalRAG {

    private final VectorStore vectorStore;      // 向量数据库客户端
    private final EmbeddingService embedder;     // Embedding服务
    private final LLMClient llmClient;           // LLM API客户端

    /**
     * 离线阶段：建索引
     */
    public void buildIndex(List<Document> documents) {
        for (Document doc : documents) {
            // 1. 把文档切成 Chunk（通常 512-1024 token）
            List<String> chunks = chunkDocument(doc.content, 512);
            
            for (String chunk : chunks) {
                // 2. 用 Embedding 模型把文本变成向量
                float[] vector = embedder.encode(chunk);
                
                // 3. 存入向量数据库
                vectorStore.insert(doc.id, chunk, vector, doc.metadata);
            }
        }
    }

    /**
     * 在线阶段：检索 + 生成
     */
    public String answer(String question) {
        // Step 1: 检索 —— 把问题也编码，找最相似的文档
        float[] questionVector = embedder.encode(question);
        List<SearchResult> topK = vectorStore.search(questionVector, 5); // Top-5
        
        // Step 2: 构建增强Prompt
        String context = topK.stream()
            .map(r -> r.content)
            .collect(Collectors.joining("\n\n"));
        
        String prompt = """
            基于以下参考资料回答问题。如果资料中没有相关信息，
            请明确说"未找到相关信息"，不要编造。
            
            参考资料：
            %s
            
            问题：%s
            回答：""".formatted(context, question);
        
        // Step 3: LLM生成
        return llmClient.complete(prompt);
    }

    // --- 接口定义（使用现有的Java AI库）---
    interface VectorStore {
        void insert(String docId, String text, float[] vector, Map<String, String> meta);
        List<SearchResult> search(float[] queryVector, int topK);
    }
    
    interface EmbeddingService {
        float[] encode(String text);
    }
    
    interface LLMClient {
        String complete(String prompt);
    }
    
    record SearchResult(String docId, String content, double score) {}
    record Document(String id, String content, Map<String, String> metadata) {}
    
    List<String> chunkDocument(String text, int maxTokens) {
        // 按句子边界或固定长度切分
        return List.of(text); // 简化
    }
}
```

**Java可行性结论**：Spring AI + Qdrant/Milvus + OpenAI SDK 完全可以实现。但生产级需要处理：①Chunk策略（不是简单切片）②重复文档去重 ③检索质量评估。

---

## 三、第二代：Advanced RAG —— 三个核心改进方向

原始RAG很简单，但实际使用中暴露了很多问题：
- 检索回来的文档可能不相关（用错了关键词）
- 检索方式单一（向量相似度不是万能的）
- 只检索一次，检索质量不好就完蛋

Advanced RAG围绕**检索前、检索中、检索后**三个阶段做了系统优化：

### 3.1 检索前：Query重写（Query Rewriting）

**问题**：用户问"JDK 21咋样？"，直接用这句话检索，向量相似度未必能找到最好的文档——因为知识库里可能叫"Java 21 新特性"。

**解法**：在检索前先用LLM改写Query，把口语化、模糊的问题改得更适合检索。

```java
/**
 * Query Rewriting —— 检索前的关键优化
 */
public class QueryRewriter {
    
    private final LLMClient llm;
    
    public String rewrite(String userQuery) {
        // 技术1: 让LLM把问题改成更适合检索的形式
        String rewritePrompt = """
            你是一个搜索查询优化专家。将用户的问题改写成更适合
            向量检索的关键词组合。输出3个候选查询，用换行分隔。
            
            用户问题：%s
            候选查询：""".formatted(userQuery);
        
        String rewritten = llm.complete(rewritePrompt);
        
        // 技术2: HyDE (Hypothetical Document Embeddings)
        // 让LLM先"凭空"写一个假设性答案，用这个答案去检索
        // —— 有时答案本身比问题更能匹配知识库中的文档
        String hydePrompt = """
            请根据以下问题，写一个假设性的简短回答（200字左右），
            即使你不知道确切答案，也请基于常识写一个合理的回答。
            
            问题：%s
            假设回答：""".formatted(userQuery);
        
        String hypotheticalAnswer = llm.complete(hydePrompt);
        
        // 用改写后的查询 + 假设答案，混合检索
        return rewritten + " " + hypotheticalAnswer;
    }
}
```

### 3.2 检索中：Hybrid Search（混合检索）

**问题**：纯向量检索在语义泛化好但可能漏掉精确关键词匹配；纯关键词检索（BM25）精确但不会语义泛化。

**解法**：二者结合，取长补短。

```java
/**
 * Hybrid Search：向量检索 + BM25 关键词检索 → 加权融合
 */
public class HybridSearcher {
    
    private final VectorStore vectorStore;   // 语义检索
    private final BM25Index bm25Index;       // 关键词检索
    
    public List<SearchResult> hybridSearch(String query, String rewrittenQuery, int topK) {
        CompletableFuture<List<SearchResult>> denseResults = 
            CompletableFuture.supplyAsync(() -> {
                float[] vec = embedder.encode(rewrittenQuery);
                return vectorStore.search(vec, topK * 2);
            });
        
        CompletableFuture<List<SearchResult>> sparseResults = 
            CompletableFuture.supplyAsync(() -> 
                bm25Index.search(query, topK * 2)
            );
        
        // 等待两边结果
        List<SearchResult> denseList = denseResults.join();
        List<SearchResult> sparseList = sparseResults.join();
        
        // RRF (Reciprocal Rank Fusion) 融合算法
        // 对于每篇文档 d，score = ∑(1 / (k + rank_i(d)))
        // 其中 rank_i 是第 i 个检索器对该文档的排名
        return reciprocalRankFusion(denseList, sparseList, topK);
    }
    
    private List<SearchResult> reciprocalRankFusion(
            List<SearchResult> list1, List<SearchResult> list2, int k) {
        Map<String, Double> scores = new HashMap<>();
        double K = 60.0;  // RRF 常数
        
        for (int i = 0; i < list1.size(); i++) {
            scores.merge(list1.get(i).docId, 1.0 / (K + i + 1), Double::sum);
        }
        for (int i = 0; i < list2.size(); i++) {
            scores.merge(list2.get(i).docId, 1.0 / (K + i + 1), Double::sum);
        }
        
        return scores.entrySet().stream()
            .sorted(Map.Entry.<String, Double>comparingByValue().reversed())
            .limit(k)
            .map(e -> new SearchResult(e.getKey(), "", e.getValue()))
            .collect(Collectors.toList());
    }
    
    interface BM25Index {
        List<SearchResult> search(String query, int topK);
    }
}
```

### 3.3 检索后：Reranker（重排序）

**问题**：检索结果返回了Top-10，但排序不一定对。也许第5条比第1条更相关。

**解法**：用一个更精细的模型对检索结果重新排序。

```java
/**
 * Reranker：用 Cross-Encoder 对检索结果做精细排序
 * 
 * 原理：向量检索用 Bi-Encoder（独立编码 query 和 doc，速度快）
 *       Reranker 用 Cross-Encoder（把 query+doc 一起输入模型，精度高但慢）
 *       分工：粗筛用 Bi-Encoder → 精排用 Cross-Encoder
 */
public class RerankerService {
    
    private final LLMClient llmClient;
    
    public List<ScoredDoc> rerank(String query, List<SearchResult> candidates, int topN) {
        // 方案1：使用专门的 Rerank 模型（Cohere Rerank / bge-reranker 等）
        // 这些模型计算 query-doc 对的精细相关性分数
        
        List<ScoredDoc> rescored = new ArrayList<>();
        for (SearchResult candidate : candidates) {
            // 简化：用LLM判断文档相关性
            String prompt = """
                判断以下文档与问题的相关性，给出0-100的分数。
                只输出数字，不要其他内容。
                
                问题：%s
                文档：%s
                相关性分数：""".formatted(query, truncate(candidate.content, 500));
            
            String scoreStr = llmClient.complete(prompt);
            double score = Double.parseDouble(scoreStr.trim());
            rescored.add(new ScoredDoc(candidate, score));
        }
        
        // 按新分数排序，取Top-N
        return rescored.stream()
            .sorted(Comparator.comparingDouble(ScoredDoc::score).reversed())
            .limit(topN)
            .collect(Collectors.toList());
    }
    
    record ScoredDoc(SearchResult doc, double score) {}
    String truncate(String text, int len) { 
        return text.length() <= len ? text : text.substring(0, len); 
    }
}
```

**Advanced RAG 完整Pipeline总结**：

```
用户问题
  │
  ├─ Query Rewriting (LLM改写)
  │
  ├─ Hybrid Search (向量 + BM25 并行检索)
  │
  ├─ Reranker (精细重排序)
  │
  └─ Context组装 → LLM生成 → 回答
```

---

## 四、第三代：Self-RAG 和 CRAG —— 让LLM判断是否需要检索

### 4.1 为什么要"判断"？

前面所有RAG变体都默认：**每次都用检索**。

但这有问题：
- 简单问题（"1+1=?"）不需要检索，直接回答更快
- 有些问题检索回来一堆无关文档，反而干扰LLM
- 检索一次不够怎么办？可能需要多轮检索

### 4.2 Self-RAG（Asai et al., 2023）

**论文**：Asai, A., et al. (2023). *Self-RAG: Learning to Retrieve, Generate, and Critique through Self-Reflection.*

**核心创新**：让LLM自己决定是否需要检索、检索了什么、检索结果是否相关、是否需要再次检索。

```
生成Token时带"反思标记"：
  <Retrieve>     —— 我需要检索
  <NoRetrieve>   —— 我不需要检索
  <Relevant>     —— 检索结果相关
  <Irrelevant>   —— 检索结果不相关
  <Supported>    —— 生成内容有据可查
  <Partially>    —— 生成内容部分有据
  <Unsupported>  —— 生成内容没依据（幻觉！）
```

### 4.3 CRAG（Corrective RAG, 2024）

**论文**：Yan, S., et al. (2024). *Corrective Retrieval Augmented Generation.*

**核心创新**：检索后加一个"纠错"步骤。如果检索质量不好，自动触发知识图谱搜索或网页搜索。

```java
/**
 * Self-RAG / CRAG 的核心思想：自适应检索
 */
public class AdaptiveRAG {
    
    public String answer(String question) {
        // Step 1: 判断是否需要检索
        boolean needsRetrieval = judgeRetrievalNeed(question);
        
        if (!needsRetrieval) {
            // 不需要检索，直接回答（节省时间+Token）
            return llmClient.complete("直接回答：%s".formatted(question));
        }
        
        // Step 2: 检索
        List<SearchResult> docs = retrieve(question);
        
        // Step 3: 评估检索质量（CRAG的关键步骤）
        RetrievalQuality quality = evaluateRetrieval(question, docs);
        
        switch (quality) {
            case GOOD:
                // 检索质量好，直接用
                return generateWithContext(question, docs);
                
            case MEDIOCRE:
                // 质量一般，尝试改进
                String rewritten = rewriteQuery(question);
                List<SearchResult> moreDocs = retrieve(rewritten);
                docs.addAll(moreDocs);
                return generateWithContext(question, docs);
                
            case POOR:
                // 质量很差，改用 Web Search 或知识图谱
                List<SearchResult> webResults = webSearch(question);
                return generateWithContext(question, webResults);
                
            default:
                return "无法回答该问题";
        }
    }
    
    enum RetrievalQuality { GOOD, MEDIOCRE, POOR }
    
    private boolean judgeRetrievalNeed(String question) {
        // 用一个小模型或规则判断
        // 简单事实类问题（"1+1=?"）→ 不需要
        // 知识类问题（"Java 21 新特性"）→ 需要
        String prompt = """
            判断是否需要检索外部知识来回答问题。只回复 YES 或 NO。
            
            问题：%s
            需要检索？""".formatted(question);
        return "YES".equals(llmClient.complete(prompt).trim());
    }
    
    private RetrievalQuality evaluateRetrieval(String question, List<SearchResult> docs) {
        if (docs.isEmpty()) return RetrievalQuality.POOR;
        
        // 检查检索结果的最高分是否足够高
        double maxScore = docs.stream()
            .mapToDouble(SearchResult::score)
            .max().orElse(0);
        
        if (maxScore > 0.8) return RetrievalQuality.GOOD;
        if (maxScore > 0.5) return RetrievalQuality.MEDIOCRE;
        return RetrievalQuality.POOR;
    }
    
    // ... 其他辅助方法
}
```

---

## 五、RAG演化的全景图

```
2020  (第一代) 原始RAG
      ├─ DPR检索 + BART生成
      └─ 端到端联合训练
      
2022  (第二代) Advanced RAG  
      ├─ Query Rewriting（检索前优化）
      ├─ Hybrid Search（检索中优化）  
      ├─ Reranker（检索后优化）
      └─ 文档摘要（避免Long Context浪费）
      
2023  (第三代) Agentic RAG
      ├─ Self-RAG（LLM自我判断检索必要性）
      ├─ ReAct（推理+行动循环）
      └─ Tool-Augmented RAG（不止检索文档，还能查API/数据库/表格）
      
2024  (第四代) Graph RAG
      ├─ Knowledge Graph + RAG（结构化知识增强）
      ├─ CRAG（纠错式RAG）
      └─ Multi-Agent RAG（多智能体协作检索）
      
2025  (持续演进)
      ├─ Agentic Search（Agent自主决定检索策略）
      ├─ Long-Context RAG（百万Token上下文+检索并存）
      └─ Memory-Augmented RAG（带记忆的持续学习RAG）
```

---

## 六、Java生态如何落地RAG

**成熟的Java框架**：

| 框架 | 定位 | RAG支持 |
|------|------|---------|
| **Spring AI** | Spring官方的AI集成框架 | ✅ 内置RAG，支持10+向量数据库 |
| **LangChain4j** | LangChain的Java移植 | ✅ 完整的RAG Pipeline |
| **Apache Lucene** | 全文检索引擎 | BM25关键词检索 |
| **Qdrant Java SDK** | 向量数据库官方SDK | 原生支持向量检索 |
| **Milvus Java SDK** | 向量数据库官方SDK | 生产级向量检索 |

**推荐技术栈**：
```
Spring Boot 3.x + Spring AI + Qdrant/Milvus + OpenAI/Claude API
```

在生产环境中，Java生态已经完全可以支撑一个完整的RAG系统。

---

## 七、总结与预告

### RAG的进化逻辑

1. **第一代**：证明了"检索+生成"这条路可行
2. **第二代**：围绕"检索质量"做全面优化
3. **第三代**：给LLM自主权——让模型判断什么时候该检索
4. **第四代**：融入知识图谱、多Agent协作

**趋势预判**：RAG不会消失，只会和LLM本身越来越耦合——未来的模型可能天生自带RAG，就像现在的模型自带Attention。

### 下期预告

RAG让LLM学会"查资料再回答"，但这仍然是**被动的**——用户问，AI答。如果让AI**主动思考、自主行动**呢？这就是**ReAct Agent**的核心思想：让LLM像人类一样，在"想"和"做"之间循环迭代，自主完成复杂任务。

**关键词**：#RAG #检索增强生成 #SelfRAG #CRAG #SpringAI #LangChain4j

---

> **参考文献**
> - Lewis, P., et al. (2020). Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks. *NeurIPS 2020*.
> - Asai, A., et al. (2023). Self-RAG: Learning to Retrieve, Generate, and Critique through Self-Reflection.
> - Yan, S., et al. (2024). Corrective Retrieval Augmented Generation.
> - Gao, Y., et al. (2023). Retrieval-Augmented Generation for Large Language Models: A Survey.
