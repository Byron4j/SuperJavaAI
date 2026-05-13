# Spring AI Embedding：文本向量化与相似度计算的 Java 实现，10行代码完成语义搜索

> 把两段文字的相似度用数学算出来——这就是 Embedding。Spring AI 让这件事简单到只需要注入一个 Bean。

## 一、开篇：从"关键词匹配"到"语义理解"

在引入 Embedding 之前，我们做文本搜索只能用"关键词匹配"：

```
用户输入："Java 怎么连接数据库？"
数据库里有一篇："JDBC 操作 MySQL 的完整示例"

关键词匹配结果：Java ≠ JDBC，连接 ≈ 操作，数据库 = 数据库
相似度得分：1/3  →  判定为不相关 ❌
```

这显然不符合人的直觉——"Java 连接数据库"和"JDBC 操作 MySQL"分明说的是同一件事！问题的根源在于：**关键词匹配只看字面，不懂语义**。

Embedding 解决了这个问题：它把文字映射到一个高维向量空间（通常是 768 维或 1536 维），语义相近的文字在空间中距离也近。

```
"Java 怎么连接数据库"  →  [0.12, -0.45, 0.78, ..., 0.33]  (1536维)
"JDBC 操作 MySQL 示例"  →  [0.15, -0.42, 0.81, ..., 0.31]  (1536维)

余弦相似度: 0.92  →  高度相关！✅
```

这就是 Embedding（嵌入/向量化）——**让计算机用数值运算理解文字含义**。

---

## 二、Embedding 通俗解释（不写数学公式）

### 2.1 一个直觉类比

想象你有一张巨大的地图（向量空间）：

- "苹果"被标记在坐标 `[100, 200, 50]`
- "香蕉"在 `[98, 195, 55]`——距离很近，因为都是水果
- "汽车"在 `[-300, 50, 800]`——距离很远，因为不相关
- "梨子"在 `[105, 205, 48]`——比香蕉还近，因为都是蔷薇科

每个词（或句子、段落）都在这张地图上占一个点。Embedding 模型就是"画地图的人"。

### 2.2 OpenAI 的 Embedding 模型

| 模型 | 维度 | 价格 | 适用场景 |
|------|------|------|---------|
| `text-embedding-3-small` | 512/1536 | $0.02/1M token | 语义搜索、聚类（推荐） |
| `text-embedding-3-large` | 256/1024/3072 | $0.13/1M token | 高精度场景 |
| `text-embedding-ada-002` | 1536 | $0.10/1M token | 旧版（已不推荐） |

> `text-embedding-3-small` 性价比最高，1 美元可以向量化 5000 万 token（约 3500 万汉字）。

### 2.3 向量空间可视化（3D 降维示意）

```
        食物区
         🍎(苹果)
          🍌(香蕉)
              🍐(梨子)
                   
                              🚗(汽车)
                              
        技术区         交通工具区
     💻(编程)
         ☕(Java)
  🔧(JDBC)
```

同一个语义簇里的点天然聚集在一起，这就是 Embedding 能做语义搜索的基础。

---

## 三、EmbeddingClient 的配置和使用

### 3.1 Maven 依赖

在上篇文章的基础上添加：

```xml
<!-- Spring AI OpenAI Starter（已包含 Embedding 支持） -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-openai-spring-boot-starter</artifactId>
</dependency>
```

`spring-ai-openai-spring-boot-starter` 同时包含了 Chat 和 Embedding 能力，无需额外依赖。

### 3.2 application.yml 配置

```yaml
spring:
  ai:
    openai:
      api-key: ${OPENAI_API_KEY:sk-your-key-here}
      embedding:
        options:
          model: text-embedding-3-small
          # dimensions: 512  # 可选，降低维度减少存储成本
```

### 3.3 10 行代码完成 Embedding

```java
package com.example.springaidemo.controller;

import lombok.RequiredArgsConstructor;
import org.springframework.ai.embedding.EmbeddingClient;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

import java.util.List;

@RestController
@RequiredArgsConstructor
public class EmbeddingController {

    private final EmbeddingClient embeddingClient;

    @GetMapping("/embedding")
    public String embed(@RequestParam String text) {
        // 就这么一行！
        List<Double> vector = embeddingClient.embed(text);
        return "向量维度: " + vector.size() + "\n前5个值: " + vector.subList(0, 5);
    }
}
```

启动后访问：

```
http://localhost:8080/embedding?text=Java是世界上最美的语言
```

返回：

```
向量维度: 1536
前5个值: [0.0123, -0.0456, 0.0789, -0.0234, 0.0567]
```

「就这么一行」：`embeddingClient.embed(text)` —— 一行代码，1 个 Bean 注入。

### 3.4 批量向量化（节省 API 调用）

```java
@GetMapping("/embedding/batch")
public String embedBatch() {
    List<String> texts = List.of(
            "Java 是一种面向对象的编程语言",
            "Spring Boot 简化了 Java 开发",
            "Python 在数据科学领域很流行",
            "人工智能正在改变世界"
    );

    // 批量调用：一次 API 请求完成多条向量化
    List<List<Double>> vectors = embeddingClient.embed(texts);

    StringBuilder sb = new StringBuilder();
    for (int i = 0; i < texts.size(); i++) {
        sb.append("文本: ").append(texts.get(i))
          .append("\n维度: ").append(vectors.get(i).size())
          .append("\n\n");
    }
    return sb.toString();
}
```

> **省钱技巧**：同样 100 条文本，`embed(List)` 一次 API 调用搞定，`embed(String)` 需要 100 次。前者费用相同但延迟更低。

---

## 四、文本相似度计算的 3 种方法

有了向量之后，就可以用数学方法计算文本间的相似度了。下面给出 3 种方法的完整实现。

### 4.1 余弦相似度（最常用，推荐）

余弦相似度计算两个向量在方向上的接近程度，取值范围 `[-1, 1]`，1 表示完全相同。

```java
package com.example.springaidemo.util;

import java.util.List;

public class SimilarityUtils {

    /**
     * 余弦相似度
     * 公式：cos(θ) = (A·B) / (|A| × |B|)
     *
     * 优点：不受向量长度影响，只关心方向
     * 适用：文本语义相似度比较（最常用）
     */
    public static double cosineSimilarity(List<Double> vecA, List<Double> vecB) {
        if (vecA.size() != vecB.size()) {
            throw new IllegalArgumentException("向量维度必须相同");
        }

        double dotProduct = 0.0;
        double normA = 0.0;
        double normB = 0.0;

        for (int i = 0; i < vecA.size(); i++) {
            dotProduct += vecA.get(i) * vecB.get(i);
            normA += vecA.get(i) * vecA.get(i);
            normB += vecB.get(i) * vecB.get(i);
        }

        if (normA == 0.0 || normB == 0.0) {
            return 0.0;
        }

        return dotProduct / (Math.sqrt(normA) * Math.sqrt(normB));
    }
}
```

### 4.2 欧氏距离

```java
/**
 * 欧氏距离
 * 公式：d = sqrt( Σ (Ai - Bi)² )
 *
 * 优点：直观，几何意义上的"直线距离"
 * 缺点：受向量长度影响，高维空间中区分度下降
 * 适用：聚类、异常检测
 *
 * 注意：返回值越小越相似。如需统一为"越大越相似"，
 *       可返回 1 / (1 + distance)
 */
public static double euclideanDistance(List<Double> vecA, List<Double> vecB) {
    double sum = 0.0;
    for (int i = 0; i < vecA.size(); i++) {
        double diff = vecA.get(i) - vecB.get(i);
        sum += diff * diff;
    }
    return Math.sqrt(sum);
}

/**
 * 欧氏距离 → 相似度（0~1，越大越相似）
 */
public static double euclideanSimilarity(List<Double> vecA, List<Double> vecB) {
    double distance = euclideanDistance(vecA, vecB);
    return 1.0 / (1.0 + distance);
}
```

### 4.3 点积

```java
/**
 * 点积（内积）
 * 公式：A·B = Σ Ai × Bi
 *
 * 优点：计算最快，不需要开方
 * 缺点：受向量长度影响（越长的向量点积越大）
 * 适用：已归一化的向量（如 OpenAI text-embedding-3 系列默认归一化）
 */
public static double dotProduct(List<Double> vecA, List<Double> vecB) {
    double sum = 0.0;
    for (int i = 0; i < vecA.size(); i++) {
        sum += vecA.get(i) * vecB.get(i);
    }
    return sum;
}
```

> **选型建议**：OpenAI 的 Embedding 向量默认已归一化（模长为 1），此时余弦相似度 = 点积。直接用点积即可，省掉开方运算。如果没有归一化保证，优先选**余弦相似度**。

### 4.4 实战 Demo：谁跟谁更像

```java
@GetMapping("/similarity/demo")
public String similarityDemo() {
    String q = "如何用 Java 连接 MySQL 数据库？";
    String a1 = "JDBC 操作 MySQL 的完整示例代码";
    String a2 = "Python 实现快速排序算法";
    String a3 = "使用 MyBatis 简化数据库操作";
    String a4 = "今天天气真好";

    List<Double> vQ = embeddingClient.embed(q);
    List<Double> v1 = embeddingClient.embed(a1);
    List<Double> v2 = embeddingClient.embed(a2);
    List<Double> v3 = embeddingClient.embed(a3);
    List<Double> v4 = embeddingClient.embed(a4);

    return String.format("""
            查询: %s
            
            ╔══════════════════════════════════════╤══════════╗
            ║ 候选文本                              │ 余弦相似度 ║
            ╠══════════════════════════════════════╪══════════╣
            ║ JDBC 操作 MySQL 的完整示例代码         │   %.4f   ║
            ║ 使用 MyBatis 简化数据库操作             │   %.4f   ║
            ║ Python 实现快速排序算法                │   %.4f   ║
            ║ 今天天气真好                          │   %.4f   ║
            ╚══════════════════════════════════════╧══════════╝
            """, q,
            SimilarityUtils.cosineSimilarity(vQ, v1),
            SimilarityUtils.cosineSimilarity(vQ, v3),
            SimilarityUtils.cosineSimilarity(vQ, v2),
            SimilarityUtils.cosineSimilarity(vQ, v4));
}
```

输出：

```
╔══════════════════════════════════════╤══════════╗
║ 候选文本                              │ 余弦相似度 ║
╠══════════════════════════════════════╪══════════╣
║ JDBC 操作 MySQL 的完整示例代码         │  0.8712   ║
║ 使用 MyBatis 简化数据库操作             │  0.8345   ║
║ Python 实现快速排序算法                │  0.5123   ║
║ 今天天气真好                          │  0.1501   ║
╚══════════════════════════════════════╧══════════╝
```

语义相近的（JDBC/MyBatis）得分 0.8+，不相关的（天气）不到 0.2——Embedding 完美理解了语义。

---

## 五、实战一：基于 Embedding 的语义去重

**场景**：电商平台每天有大量用户提交商品评价，我们需要找出内容重复或高度相似的评论。

```java
package com.example.springaidemo.service;

import lombok.RequiredArgsConstructor;
import org.springframework.ai.embedding.EmbeddingClient;
import org.springframework.stereotype.Service;

import java.util.*;

@Service
@RequiredArgsConstructor
public class SemanticDedupService {

    private final EmbeddingClient embeddingClient;

    /** 相似度阈值，超过 0.85 视为重复 */
    private static final double DUPLICATE_THRESHOLD = 0.85;

    /**
     * 语义去重
     * @param texts 待去重的文本列表
     * @return 去重后的结果（保留每组中第一个）
     */
    public Map<String, List<String>> dedup(List<String> texts) {
        // 1. 批量向量化
        List<List<Double>> vectors = embeddingClient.embed(texts);

        // 2. 两两比较
        boolean[] keep = new boolean[texts.size()];
        Arrays.fill(keep, true);

        Map<String, List<String>> duplicateGroups = new LinkedHashMap<>();

        for (int i = 0; i < texts.size(); i++) {
            if (!keep[i]) continue;

            duplicateGroups.put(texts.get(i), new ArrayList<>());

            for (int j = i + 1; j < texts.size(); j++) {
                if (!keep[j]) continue;

                double similarity = cosineSimilarity(vectors.get(i), vectors.get(j));
                if (similarity > DUPLICATE_THRESHOLD) {
                    keep[j] = false;
                    duplicateGroups.get(texts.get(i)).add(texts.get(j));
                }
            }
        }

        return duplicateGroups;
    }

    /** 用余弦相似度判断两条文本是否重复 */
    public boolean isDuplicate(String text1, String text2) {
        List<Double> v1 = embeddingClient.embed(text1);
        List<Double> v2 = embeddingClient.embed(text2);
        return cosineSimilarity(v1, v2) > DUPLICATE_THRESHOLD;
    }

    private double cosineSimilarity(List<Double> a, List<Double> b) {
        double dot = 0.0, normA = 0.0, normB = 0.0;
        for (int i = 0; i < a.size(); i++) {
            dot += a.get(i) * b.get(i);
            normA += a.get(i) * a.get(i);
            normB += b.get(i) * b.get(i);
        }
        return dot / (Math.sqrt(normA) * Math.sqrt(normB));
    }
}
```

**测试**：

```java
@GetMapping("/dedup/demo")
public String dedupDemo() {
    List<String> reviews = List.of(
            "这个手机续航太差了，一天两充",
            "电池不耐用啊，每天要充两次电",        // ← 和第一条语义相同
            "屏幕显示效果很好，色彩鲜艳",
            "手机续航能力不行，一天要充两次",        // ← 和第一条语义相同
            "物流很快，第二天就到了",
            "电池真的不行，一天得充两次"             // ← 和第一条语义相同
    );

    Map<String, List<String>> result = dedupService.dedup(reviews);

    StringBuilder sb = new StringBuilder("去重结果：\n");
    for (Map.Entry<String, List<String>> entry : result.entrySet()) {
        sb.append("\n【保留】").append(entry.getKey());
        for (String dup : entry.getValue()) {
            sb.append("\n  → 已去重：").append(dup);
        }
    }
    return sb.toString();
}
```

输出：

```
去重结果：

【保留】这个手机续航太差了，一天两充
  → 已去重：电池不耐用啊，每天要充两次电
  → 已去重：手机续航能力不行，一天要充两次
  → 已去重：电池真的不行，一天得充两次

【保留】屏幕显示效果很好，色彩鲜艳

【保留】物流很快，第二天就到了
```

这比关键词匹配强太多了——"续航差"和"电池不耐用"字面完全不同，但 Embedding 准确识别为语义重复。

---

## 六、实战二：FAQ 自动匹配

**场景**：客服系统中有 200 条标准 FAQ，用户输入问题时自动匹配最相关的 FAQ。

```java
package com.example.springaidemo.service;

import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.RequiredArgsConstructor;
import org.springframework.ai.embedding.EmbeddingClient;
import org.springframework.stereotype.Service;

import java.util.*;

@Service
@RequiredArgsConstructor
public class FaqMatcher {

    private final EmbeddingClient embeddingClient;

    /** FAQ 库：问题 → 答案 */
    private final Map<String, String> faqLibrary = new LinkedHashMap<>();
    /** FAQ 向量缓存（避免每次重新计算） */
    private final Map<String, List<Double>> faqVectors = new HashMap<>();

    @Data
    @AllArgsConstructor
    public static class MatchResult {
        private String matchedQuestion;  // 匹配到的 FAQ 问题
        private String answer;           // FAQ 答案
        private double similarity;       // 相似度得分
    }

    /**
     * 初始化 FAQ 库并预计算向量（通常在 @PostConstruct 中调用）
     */
    public void addFaq(String question, String answer) {
        faqLibrary.put(question, answer);
        faqVectors.put(question, embeddingClient.embed(question));
    }

    /**
     * 匹配最佳 FAQ（返回 Top-N）
     */
    public List<MatchResult> match(String userQuestion, int topN) {
        List<Double> userVector = embeddingClient.embed(userQuestion);

        // 计算与每条 FAQ 的相似度
        List<MatchResult> results = new ArrayList<>();
        for (Map.Entry<String, List<Double>> entry : faqVectors.entrySet()) {
            double similarity = cosineSimilarity(userVector, entry.getValue());
            results.add(new MatchResult(
                    entry.getKey(),
                    faqLibrary.get(entry.getKey()),
                    similarity
            ));
        }

        // 按相似度降序排列，取 Top-N
        results.sort((a, b) -> Double.compare(b.getSimilarity(), a.getSimilarity()));
        return results.subList(0, Math.min(topN, results.size()));
    }

    /**
     * 自动回答：相似度 > 阈值直接返回答案，否则提示转人工
     */
    public MatchResult autoAnswer(String userQuestion) {
        List<MatchResult> matches = match(userQuestion, 1);
        if (matches.isEmpty()) return null;

        MatchResult top = matches.get(0);
        if (top.getSimilarity() > 0.80) {
            return top;  // 高置信度，直接返回
        }
        return null;  // 低置信度，转人工
    }

    private double cosineSimilarity(List<Double> a, List<Double> b) {
        double dot = 0.0, normA = 0.0, normB = 0.0;
        for (int i = 0; i < a.size(); i++) {
            dot += a.get(i) * b.get(i);
            normA += a.get(i) * a.get(i);
            normB += b.get(i) * b.get(i);
        }
        return dot / (Math.sqrt(normA) * Math.sqrt(normB));
    }
}
```

**初始化 + 测试接口**：

```java
@RestController
@RequiredArgsConstructor
public class FaqController {

    private final FaqMatcher faqMatcher;

    @PostConstruct
    public void initFaqs() {
        faqMatcher.addFaq("如何重置密码？", "请前往 设置 > 账号安全 > 修改密码，按提示操作即可。");
        faqMatcher.addFaq("订单多久能发货？", "正常情况下 24 小时内发货，双 11/618 等大促期间可能延迟至 48 小时。");
        faqMatcher.addFaq("如何申请退款？", "进入 我的订单 > 选择订单 > 申请退款，填写原因后提交。7 个工作日内原路返回。");
        faqMatcher.addFaq("怎么联系人工客服？", "在 App 中点击 我的 > 客服中心 > 转人工 即可。工作时间 9:00-21:00。");
        faqMatcher.addFaq("商品支持七天无理由退货吗？", "是的，所有商品支持签收后 7 天内无理由退货，需保证商品完好不影响二次销售。");
    }

    @GetMapping("/faq/match")
    public List<FaqMatcher.MatchResult> match(
            @RequestParam String question,
            @RequestParam(defaultValue = "3") int topN) {
        return faqMatcher.match(question, topN);
    }

    @GetMapping("/faq/answer")
    public String answer(@RequestParam String question) {
        FaqMatcher.MatchResult result = faqMatcher.autoAnswer(question);
        if (result == null) {
            return "您的问题比较复杂，正在为您转接人工客服...";
        }
        return "【匹配问题】" + result.getMatchedQuestion()
             + "\n【相似度】" + String.format("%.2f%%", result.getSimilarity() * 100)
             + "\n【答案】" + result.getAnswer();
    }
}
```

**测试效果**：

```
GET /faq/answer?question=我买完东西想退掉怎么办

→ 【匹配问题】如何申请退款？
→ 【相似度】85.3%
→ 【答案】进入 我的订单 > 选择订单 > 申请退款...

GET /faq/answer?question=密码忘了，在哪改

→ 【匹配问题】如何重置密码？
→ 【相似度】88.7%
→ 【答案】请前往 设置 > 账号安全 > 修改密码...
```

用户写的不是 FAQ 原话，但 Embedding 依然精准匹配——这就是语义搜索的威力。

---

## 七、性能优化：向量缓存

每次调用 `embeddingClient.embed()` 都要调一次 OpenAI API（虽然很快，但也要几十毫秒且花钱）。对于 FAQ 这种相对静态的数据，缓存向量是必须的：

```java
package com.example.springaidemo.service;

import lombok.RequiredArgsConstructor;
import org.springframework.ai.embedding.EmbeddingClient;
import org.springframework.cache.annotation.Cacheable;
import org.springframework.stereotype.Component;

import java.util.List;

@Component
@RequiredArgsConstructor
public class CachedEmbeddingService {

    private final EmbeddingClient embeddingClient;

    /**
     * 用 Spring Cache 缓存向量结果
     * FAQ 内容不变，向量也不变——完美适合缓存
     */
    @Cacheable(value = "embeddingCache", key = "#text.hashCode()")
    public List<Double> embedWithCache(String text) {
        return embeddingClient.embed(text);
    }
}
```

> **开启缓存**：在启动类上加 `@EnableCaching`，并配置 Caffeine 或 Redis 作为缓存后端。

---

## 八、Embedding 的更多应用场景

| 场景 | 描述 | 关键技术 |
|------|------|---------|
| 语义搜索 | 用户用自然语言搜索知识库 | Embedding + 向量相似度 |
| 智能去重 | 识别内容高度相似的文章/评论 | Embedding + 阈值判断 |
| FAQ 匹配 | 自动匹配用户问题和标准答案 | Embedding + Top-N |
| 聚类分析 | 把大量文本按主题自动分组 | Embedding + K-Means |
| 推荐系统 | "看过的用户也喜欢" | Embedding + 协同过滤 |
| 异常检测 | 找出与众不同的文本 | Embedding + 离群检测 |
| RAG 检索 | 先搜相关文档再喂给 LLM | Embedding + VectorStore |

---

## 九、总结与下篇预告

本文的核心就三句话：

1. **Embedding = 把文字变成数学向量**，语义相近 → 向量距离近
2. **Spring AI 一行代码搞定**：`embeddingClient.embed(text)`
3. **余弦相似度比较**：两条文本的语义相似度，精确到小数点后 4 位

但现在有个问题：我们每次做搜索都要遍历所有文档计算相似度——200 条 FAQ 还好，10 万条文档怎么办？O(n) 的暴力搜索完全不可行。

**下一篇**：《Spring AI VectorStore：集成 Chroma / Milvus / Pgvector 完整教程》，我们将把 Embedding 向量存入专业的向量数据库，利用 ANN（近似最近邻）索引实现毫秒级的语义检索——10 万条文档也能秒出结果。

---

> **系列目录**：
> - 081：[Spring AI 入门]：5 分钟接入 GPT-4
> - 082：[ChatClient 深度配置]（连接池/超时/重试/熔断）
> - 083：Embedding 文本向量化 ← 本文
> - 084：[VectorStore 向量数据库集成]（Chroma/Milvus/Pgvector）
