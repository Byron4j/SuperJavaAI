# 客服知识库 RAG：FAQ 文档 + 历史工单的双重检索策略，新客服 3 天变老手

> 一个新客服要多久才能独立上岗？1 个月？3 个月？电话客服的离职率高达 50%，因为他们需要记住几百条 FAQ、了解几十个系统、应对各种奇葩问题。"双检索策略"让新客服 3 天就能找到最佳答案。

## 一、客服场景的特殊挑战

客服场景的 RAG 和通用 RAG 有本质区别：

| 维度 | 通用 RAG | 客服 RAG |
|------|---------|---------|
| 数据源 | 单一文档 | FAQ + 历史工单 + SOP |
| 答案形式 | 段落总结 | 带操作指引的标准答案 |
| 准确性要求 | 中等 | 极高（答错就是事故） |
| 延迟要求 | 秒级 | 亚秒级（电话在线等待） |
| 人工介入 | 不需要 | 需要无缝切换 |

最大的挑战是：**FAQ 文档写得很好，但现实中的客户问题往往不在 FAQ 里**——这也是为什么需要"双重检索"。

## 二、双重检索架构

```
┌─────────────────────────────────────────────────────────────────┐
│                    双重检索 RAG 架构                              │
│                                                                 │
│  ┌───────────┐                    ┌──────────────────────────┐  │
│  │ 用户问题   │                    │      融合排序引擎         │  │
│  │ "手机续费  │                    │                          │  │
│  │  后怎么    │                    │  ┌──────────┐──────────┐ │  │
│  │  还停机？" │                    │  │ FAQ 得分  │ 工单得分 │ │  │
│  └─────┬─────┘                    │  │   0.82   │   0.91  │ │  │
│        │                          │  └──────────┘──────────┘ │  │
│        │                          │        │                  │  │
│   ┌────▼────┐  ┌─────────────┐    │   ┌────▼──────────────────┐ │  │
│   │ 查询     │  │ 查询增强     │    │   │ 加权融合               │ │  │
│   │ 改写     │  │ (同义词扩展) │    │   │ score = 0.4*FAQ       │ │  │
│   └────┬────┘  └──────┬──────┘    │   │       + 0.6*Ticket    │ │  │
│        │              │           │   └────────────────────────┘ │  │
│   ┌────▼──────────────▼───────┐   │              │               │  │
│   │      并行检索               │   │         ┌────▼──────┐        │  │
│   │                           │   │         │ Top-K 结果  │        │  │
│   │  ┌────────┐ ┌──────────┐  │   │         └────┬──────┘        │  │
│   │  │FAQ 库  │ │历史工单库 │  │   │              │               │  │
│   │  │        │ │          │  │   │         ┌────▼──────┐        │  │
│   │  │标准答案 │ │10万+工单  │  │   │         │ LLM 整合   │        │  │
│   │  │结构化   │ │非结构化   │  │   │         └────┬──────┘        │  │
│   │  └────────┘ └──────────┘  │   │              │               │  │
│   └───────────────────────────┘   │         ┌────▼──────┐        │  │
│                                   │         │ 答案 > 0.8│        │  │
│                                   │         │ 置信度？   │        │  │
│                                   │         └────┬──────┘        │  │
│                                   │        ┌─────┴─────┐        │  │
│                                   │     ≥0.8│           │<0.8    │  │
│                                   │    ┌────▼──┐   ┌───▼────┐   │  │
│                                   │    │ AI 回复│   │转人工   │   │  │
│                                   │    └───────┘   └────────┘   │  │
└─────────────────────────────────────────────────────────────────┘
```

## 三、FAQ 的结构化处理

### 3.1 FAQ 文档的特点

FAQ 文档通常有如下结构：

```markdown
## Q: 如何办理停机保号？
**适用套餐**: 畅享套餐 / 5G 套餐
**操作路径**: 中国移动 APP > 我的 > 停机保号
**费用**: 5元/月
**注意事项**: 
- 停机期间不收取套餐费
- 最长可停机90天
- 复机后套餐恢复
**关联问题**: 如何复机？停机期间是否影响积分？
```

FAQ 的关键特征：**一问一答，答案高度结构化，有明确的操作路径和注意事项**。

### 3.2 FAQ 解析与拆分

```java
import java.util.*;
import java.util.regex.Pattern;
import java.util.regex.Matcher;

public class FAQParser {

    public record FAQEntry(
        String id,
        String question,           // 问题
        List<String> keywords,     // 关键词
        String answer,             // 标准答案
        FaqCategory category,      // 分类
        String applicablePlans,    // 适用套餐/产品
        String operationPath,      // 操作路径
        List<String> notes,        // 注意事项
        List<String> relatedFAQIds, // 关联 FAQ
        int frequency,             // 被查询频率
        String updateTime          // 最后更新时间
    ) {}

    public enum FaqCategory {
        ACCOUNT,      // 账户相关
        BILLING,      // 账单/费用
        PLAN,         // 套餐/产品
        NETWORK,      // 网络/信号
        SERVICE,      // 业务办理
        COMPLAINT,    // 投诉建议
        OTHER
    }

    /**
     * 解析 Markdown 格式的 FAQ 文档
     */
    public List<FAQEntry> parse(String faqMarkdown) {
        List<FAQEntry> entries = new ArrayList<>();

        // 按 ## Q: 分割
        String[] sections = faqMarkdown.split("(?=## Q:)");

        for (String section : sections) {
            if (!section.trim().startsWith("## Q:")) continue;

            FAQEntry entry = new FAQEntry(
                generateId(),
                extractQuestion(section),
                extractKeywords(section),
                extractAnswer(section),
                extractCategory(section),
                extractField(section, "适用套餐"),
                extractField(section, "操作路径"),
                extractNotes(section),
                extractRelatedFaqIds(section),
                0,
                null
            );

            entries.add(entry);
        }

        return entries;
    }

    private String extractQuestion(String section) {
        // 匹配 ## Q: xxx
        Pattern p = Pattern.compile("## Q:\\s*(.+?)(?:\\n|$)");
        Matcher m = p.matcher(section);
        return m.find() ? m.group(1).trim() : "";
    }

    private List<String> extractKeywords(String section) {
        // 从问题和分类中提取关键词
        String question = extractQuestion(section);
        List<String> keywords = new ArrayList<>();

        // 简单的关键词提取（生产环境建议用 NLP 分词）
        String[] words = question.split("[\\s，。？?]+");
        for (String word : words) {
            if (word.length() >= 2) {
                keywords.add(word);
            }
        }

        // 从答案的加粗文本中提取关键词
        Pattern p = Pattern.compile("\\*\\*(.+?)\\*\\*");
        Matcher m = p.matcher(section);
        while (m.find()) {
            keywords.add(m.group(1));
        }

        return keywords;
    }

    private String extractAnswer(String section) {
        // 答案 = 去掉问题标题后的全部内容
        String cleaned = section.replaceFirst("## Q:.+?\\n", "").trim();
        // 结构化整理
        return cleaned;
    }

    private FaqCategory extractCategory(String section) {
        String lower = section.toLowerCase();
        if (lower.contains("账户") || lower.contains("登录") || lower.contains("密码"))
            return FaqCategory.ACCOUNT;
        if (lower.contains("账单") || lower.contains("费用") || lower.contains("扣费"))
            return FaqCategory.BILLING;
        if (lower.contains("套餐") || lower.contains("流量") || lower.contains("通话"))
            return FaqCategory.PLAN;
        if (lower.contains("信号") || lower.contains("网络") || lower.contains("断网"))
            return FaqCategory.NETWORK;
        if (lower.contains("办理") || lower.contains("开通") || lower.contains("取消"))
            return FaqCategory.SERVICE;
        if (lower.contains("投诉") || lower.contains("建议") || lower.contains("反馈"))
            return FaqCategory.COMPLAINT;
        return FaqCategory.OTHER;
    }

    private String extractField(String section, String fieldName) {
        Pattern p = Pattern.compile(Pattern.quote("**" + fieldName + "**") +
            "\\s*[:：]\\s*(.+?)(?:\\n|$)");
        Matcher m = p.matcher(section);
        return m.find() ? m.group(1).trim() : "";
    }

    private List<String> extractNotes(String section) {
        List<String> notes = new ArrayList<>();
        Pattern p = Pattern.compile("\\*\\*注意事项\\*\\*[\\s\\S]*?(?=\\n\\*\\*|$)");
        Matcher m = p.matcher(section);
        if (m.find()) {
            String noteSection = m.group();
            // 提取 - 开头的列表项
            Pattern itemP = Pattern.compile("-\\s*(.+?)(?:\\n|$)");
            Matcher itemM = itemP.matcher(noteSection);
            while (itemM.find()) {
                notes.add(itemM.group(1).trim());
            }
        }
        return notes;
    }

    private List<String> extractRelatedFaqIds(String section) {
        Pattern p = Pattern.compile("\\*\\*关联问题\\*\\*[\\s\\S]*?(?=\\n\\*\\*|$)");
        Matcher m = p.matcher(section);
        if (m.find()) {
            // 提取关联问题，后续通过问题文本匹配 FAQ ID
            return List.of(m.group());
        }
        return List.of();
    }

    private String generateId() {
        return "FAQ-" + UUID.randomUUID().toString().substring(0, 8);
    }
}
```

### 3.3 FAQ 的向量化增强

FAQ 的特殊之处在于：问法和写法差异巨大。用户问"手机没信号了怎么办"，FAQ 写的是"移动网络服务异常时的处理指引"。

```java
public class FAQEmbeddingEnhancer {

    /**
     * 将 FAQ 转换为增强的 Embedding 文本
     */
    public static String enhance(FAQParser.FAQEntry faq) {
        StringBuilder enhanced = new StringBuilder();

        // 1. 问题同义词扩展
        enhanced.append("客户问题: ").append(faq.question()).append("\n");

        // 生成问法变体
        List<String> variants = generateQuestionVariants(faq.question());
        for (String variant : variants) {
            enhanced.append("可能问法: ").append(variant).append("\n");
        }

        // 2. 关键词列表
        enhanced.append("关键词: ");
        for (String kw : faq.keywords()) {
            enhanced.append(kw).append(" ");
            // 关键词的同义词
            enhanced.append(expandKeyword(kw)).append(" ");
        }
        enhanced.append("\n");

        // 3. 分类信息
        enhanced.append("问题分类: ").append(faq.category()).append("\n");

        // 4. 适用场景
        if (!faq.applicablePlans().isEmpty()) {
            enhanced.append("适用产品: ").append(faq.applicablePlans()).append("\n");
        }

        // 5. 答案摘要（前 200 字）
        String summary = faq.answer().length() > 200
            ? faq.answer().substring(0, 200) : faq.answer();
        enhanced.append("答案摘要: ").append(summary).append("\n");

        return enhanced.toString();
    }

    /**
     * 生成问题的自然语言变体，提高检索覆盖率
     */
    private static List<String> generateQuestionVariants(String question) {
        List<String> variants = new ArrayList<>();

        // 规则变体（生产环境可用 LLM 批量生成）
        // "怎么办理停机" → ["如何办理停机", "停机怎么操作", "停机办理流程", "停机保号怎么弄"]
        variants.add(question
            .replace("怎么", "如何")
            .replace("为啥", "为什么")
            .replace("啥", "什么"));
        variants.add(question
            .replace("怎么", "怎样")
            .replace("什么", "啥"));
        // 省略主语变体
        if (question.startsWith("我")) {
            variants.add(question.replaceFirst("我", ""));
        }
        if (question.endsWith("？") || question.endsWith("?")) {
            variants.add(question.substring(0, question.length() - 1));
        }

        return variants;
    }

    private static String expandKeyword(String keyword) {
        Map<String, String> synonymMap = Map.ofEntries(
            Map.entry("停机", "暂停 保号 挂失"),
            Map.entry("充值", "缴费 续费 充钱"),
            Map.entry("信号", "网络 网速 断网 无服务"),
            Map.entry("套餐", "资费 月租 流量包"),
            Map.entry("发票", "账单 收据 报销凭证"),
            Map.entry("销户", "退网 注销 取消账户"),
            Map.entry("宽带", "光纤 WIFI 上网"),
            Map.entry("积分", "金币 权益 兑换")
        );
        return synonymMap.getOrDefault(keyword, "");
    }
}
```

## 四、历史工单的向量化

### 4.1 历史工单的数据结构

```java
public record TicketRecord(
    String ticketId,        // 工单编号
    String customerQuery,   // 客户原始问题
    String customerContext, // 客户上下文（套餐、账户状态等）
    String agentResponse,   // 客服最终回复
    String resolutionPath,  // 解决路径（操作步骤）
    List<String> tags,      // 标签
    String category,        // 分类
    String satisfaction,    // 满意度评分
    int resolutionTime,     // 解决时长（分钟）
    Timestamp createdAt
) {}

/**
 * 历史工单解析器（从数据库/CSV/Excel 中导入）
 */
public class TicketImporter {

    public List<TicketRecord> importFromDB(String query) {
        // 从工单系统中读取历史工单
        // SELECT * FROM cs_tickets WHERE status = 'RESOLVED'
        return List.of();
    }

    /**
     * 将工单转换为可 Embedding 的文本
     */
    public static String ticketToEmbeddingText(TicketRecord ticket) {
        StringBuilder sb = new StringBuilder();
        sb.append("客户问题: ").append(ticket.customerQuery()).append("\n");

        if (ticket.customerContext() != null && !ticket.customerContext().isEmpty()) {
            sb.append("客户状态: ").append(ticket.customerContext()).append("\n");
        }

        sb.append("客服回复: ").append(ticket.agentResponse()).append("\n");

        if (ticket.resolutionPath() != null && !ticket.resolutionPath().isEmpty()) {
            sb.append("解决步骤: ").append(ticket.resolutionPath()).append("\n");
        }

        sb.append("标签: ").append(String.join(", ", ticket.tags())).append("\n");
        sb.append("分类: ").append(ticket.category()).append("\n");
        sb.append("满意度: ").append(ticket.satisfaction()).append("\n");

        return sb.toString();
    }
}
```

### 4.2 历史工单的过滤与去重

不是所有工单都值得索引：

```java
public class TicketFilter {

    /**
     * 工单质量过滤
     */
    public boolean shouldIndex(TicketRecord ticket) {
        // 过滤条件：
        // 1. 已解决的工单
        // 2. 满意度 >= 3 星
        // 3. 解决时长合理（不是异常值）
        // 4. 内容不为空
        // 5. 去重（和已索引的工单相似度 < 0.95）

        if (ticket.customerQuery() == null || ticket.customerQuery().isBlank()) {
            return false;
        }
        if (ticket.agentResponse() == null || ticket.agentResponse().isBlank()) {
            return false;
        }
        if (ticket.satisfaction() != null) {
            int sat = Integer.parseInt(ticket.satisfaction().replaceAll("[^0-9]", ""));
            if (sat < 3) return false;  // 过滤低满意度工单
        }
        return true;
    }

    /**
     * 相似工单去重：如果两个工单内容高度相似，只保留满意度更高的那个
     */
    public List<TicketRecord> deduplicate(List<TicketRecord> tickets) {
        Map<String, TicketRecord> deduped = new LinkedHashMap<>();

        for (TicketRecord ticket : tickets) {
            String key = normalizeKey(ticket.customerQuery());
            TicketRecord existing = deduped.get(key);
            if (existing == null || compareSatisfaction(ticket, existing) > 0) {
                deduped.put(key, ticket);
            }
        }

        return new ArrayList<>(deduped.values());
    }

    private String normalizeKey(String query) {
        // 去标点、去空格、截取前 50 字
        return query.replaceAll("[\\p{Punct}\\s]", "")
            .substring(0, Math.min(50, query.length()));
    }

    private int compareSatisfaction(TicketRecord a, TicketRecord b) {
        try {
            int sa = Integer.parseInt(a.satisfaction().replaceAll("[^0-9]", ""));
            int sb = Integer.parseInt(b.satisfaction().replaceAll("[^0-9]", ""));
            return Integer.compare(sa, sb);
        } catch (Exception e) {
            return 0;
        }
    }
}
```

## 五、双重检索 + 融合排序

### 5.1 并行检索器

```java
public class DualRetriever {

    private final VectorStore faqStore;       // FAQ 向量库
    private final VectorStore ticketStore;    // 历史工单向量库
    private final EmbeddingService embeddingService;

    public record RetrievalResult(
        List<ScoredDocument> faqResults,
        List<ScoredDocument> ticketResults
    ) {}

    public record ScoredDocument(
        String id,
        String content,
        Map<String, Object> metadata,
        double similarity,
        String sourceType  // "FAQ" | "TICKET"
    ) {}

    public DualRetriever(VectorStore faqStore, VectorStore ticketStore,
                          EmbeddingService embeddingService) {
        this.faqStore = faqStore;
        this.ticketStore = ticketStore;
        this.embeddingService = embeddingService;
    }

    /**
     * 并行检索 FAQ 和工单
     */
    public RetrievalResult retrieve(String userQuestion, int topK) {
        // 查询增强
        String enhancedQuery = enhanceQuery(userQuestion);
        List<Float> queryEmbedding = embeddingService.embed(enhancedQuery);

        // 并行检索
        List<VectorDocument> faqDocs = faqStore.search(queryEmbedding, topK);
        List<VectorDocument> ticketDocs = ticketStore.search(queryEmbedding, topK);

        return new RetrievalResult(
            toScoredDocs(faqDocs, "FAQ"),
            toScoredDocs(ticketDocs, "TICKET")
        );
    }

    private String enhanceQuery(String query) {
        // 同义词扩展、口语化转书面化
        return query
            .replace("咋", "怎么")
            .replace("为啥", "为什么")
            .replace("充钱", "充值")
            .replace("上不去网", "无法上网")
            .replace("扣我钱", "扣费");
    }

    private List<ScoredDocument> toScoredDocs(List<VectorDocument> docs, String source) {
        return docs.stream()
            .map(doc -> new ScoredDocument(
                doc.id(), doc.content(), doc.metadata(), doc.similarity(), source))
            .toList();
    }
}
```

### 5.2 融合排序引擎

这是整个系统的核心——如何合理融合两种不同来源的检索结果：

```java
public class FusionRanker {

    // 权重配置（可动态调整）
    private double faqWeight = 0.4;      // FAQ 权重
    private double ticketWeight = 0.6;   // 历史工单权重（通常更高，因为更"实战"）

    /**
     * 加权分数融合（Weighted Sum Fusion）
     */
    public List<DualRetriever.ScoredDocument> weightedFusion(
            DualRetriever.RetrievalResult result, int topK) {

        Map<String, DualRetriever.ScoredDocument> merged = new LinkedHashMap<>();

        // 1. 合并两个来源的结果
        for (var faq : result.faqResults()) {
            merged.put(faq.id(), new DualRetriever.ScoredDocument(
                faq.id(), faq.content(), faq.metadata(),
                faq.similarity() * faqWeight, faq.sourceType()
            ));
        }

        for (var ticket : result.ticketResults()) {
            String key = ticket.id();
            if (merged.containsKey(key)) {
                // 如果 FAQ 也有相似结果，取最大分
                double maxScore = Math.max(
                    merged.get(key).similarity(),
                    ticket.similarity() * ticketWeight);
                merged.put(key, new DualRetriever.ScoredDocument(
                    ticket.id(), ticket.content(), ticket.metadata(),
                    maxScore, ticket.sourceType()));
            } else {
                merged.put(key, new DualRetriever.ScoredDocument(
                    ticket.id(), ticket.content(), ticket.metadata(),
                    ticket.similarity() * ticketWeight, ticket.sourceType()));
            }
        }

        // 2. 按分数降序排序
        return merged.values().stream()
            .sorted((a, b) -> Double.compare(b.similarity(), a.similarity()))
            .limit(topK)
            .toList();
    }

    /**
     * 倒数排名融合（Reciprocal Rank Fusion, RRF）
     * 不依赖相似度的绝对值，只依赖排名
     */
    public List<DualRetriever.ScoredDocument> reciprocalRankFusion(
            DualRetriever.RetrievalResult result, int topK, int k) {

        Map<String, Double> scores = new HashMap<>();
        Map<String, DualRetriever.ScoredDocument> docMap = new HashMap<>();

        // FAQ 排名得分
        for (int i = 0; i < result.faqResults().size(); i++) {
            var doc = result.faqResults().get(i);
            double rrf = 1.0 / (k + i + 1);  // RRF 公式
            scores.merge(doc.id(), rrf, Double::sum);
            docMap.putIfAbsent(doc.id(), doc);
        }

        // 工单排名得分
        for (int i = 0; i < result.ticketResults().size(); i++) {
            var doc = result.ticketResults().get(i);
            double rrf = 1.0 / (k + i + 1);
            scores.merge(doc.id(), rrf, Double::sum);
            docMap.putIfAbsent(doc.id(), doc);
        }

        return scores.entrySet().stream()
            .sorted(Map.Entry.<String, Double>comparingByValue().reversed())
            .limit(topK)
            .map(entry -> {
                var doc = docMap.get(entry.getKey());
                return new DualRetriever.ScoredDocument(
                    doc.id(), doc.content(), doc.metadata(),
                    entry.getValue(), doc.sourceType());
            })
            .toList();
    }

    /**
     * 动态权重调整（根据搜索结果质量自适应）
     */
    public void adjustWeights(DualRetriever.RetrievalResult result) {
        double faqAvg = result.faqResults().stream()
            .mapToDouble(DualRetriever.ScoredDocument::similarity).average().orElse(0);
        double ticketAvg = result.ticketResults().stream()
            .mapToDouble(DualRetriever.ScoredDocument::similarity).average().orElse(0);

        // 哪个来源的平均相似度更高，赋予更高权重
        if (faqAvg > 0.7 && ticketAvg < 0.5) {
            this.faqWeight = 0.7;
            this.ticketWeight = 0.3;
        } else if (ticketAvg > 0.7 && faqAvg < 0.5) {
            this.faqWeight = 0.3;
            this.ticketWeight = 0.7;
        } else {
            this.faqWeight = 0.4;
            this.ticketWeight = 0.6;
        }
    }
}
```

## 六、相似工单推荐

当检索到历史工单时，不仅告诉客服"这个问题可能是什么"，还能推荐"之前类似的工单是怎么解决的"：

```java
public class SimilarTicketRecommender {

    /**
     * 根据匹配到的 FAQ，推荐相似的历史工单
     */
    public List<TicketRecord> recommendSimilarTickets(
            String topFAQId, List<DualRetriever.ScoredDocument> ticketResults) {

        // 只返回高分历史工单作为参考
        return ticketResults.stream()
            .filter(t -> t.similarity() >= 0.7)
            .filter(t -> "TICKET".equals(t.sourceType()))
            .map(doc -> {
                // 从工单元数据中提取关键信息
                Map<String, Object> meta = doc.metadata();
                return new TicketRecord(
                    (String) meta.get("ticketId"),
                    doc.content(),
                    (String) meta.get("customerContext"),
                    (String) meta.get("agentResponse"),
                    (String) meta.get("resolutionPath"),
                    List.of(),
                    (String) meta.get("category"),
                    (String) meta.get("satisfaction"),
                    0,
                    null
                );
            })
            .limit(5)
            .toList();
    }
}
```

## 七、人工客服与 AI 的切换机制

### 7.1 置信度评估

```java
public class ConfidenceEvaluator {

    /**
     * 评估当前回答的置信度
     * 返回 0.0 ~ 1.0
     */
    public double evaluate(List<DualRetriever.ScoredDocument> fusedResults) {
        if (fusedResults.isEmpty()) return 0.0;

        // 1. 最高相似度得分
        double topScore = fusedResults.get(0).similarity();

        // 2. 得分衰减（如果第一名和第二名差距太大，说明答案唯一性高）
        double secondScore = fusedResults.size() > 1
            ? fusedResults.get(1).similarity() : 0;
        double uniqueness = topScore - secondScore;

        // 3. 来源一致性（FAQ 和工单都说同一个答案，置信度更高）
        boolean hasFaq = fusedResults.stream().anyMatch(d -> "FAQ".equals(d.sourceType()));
        boolean hasTicket = fusedResults.stream().anyMatch(d -> "TICKET".equals(d.sourceType()));
        double sourceBonus = (hasFaq && hasTicket) ? 0.1 : 0.0;

        // 4. 综合评分
        double confidence = topScore * 0.6 + uniqueness * 0.3 + sourceBonus;

        return Math.min(1.0, Math.max(0.0, confidence));
    }
}
```

### 7.2 智能路由

```java
public class IntelligentRouter {

    private final DualRetriever retriever;
    private final FusionRanker ranker;
    private final ConfidenceEvaluator evaluator;
    private final LLMClient llmClient;
    private final HumanAgentQueue agentQueue;

    // 阈值配置
    private static final double AUTO_REPLY_THRESHOLD = 0.80;  // 自动回复
    private static final double SUGGEST_THRESHOLD = 0.60;     // 推荐但不自动发
    private static final double ESCALATE_THRESHOLD = 0.40;    // 直接转人工

    public IntelligentRouter(DualRetriever retriever, FusionRanker ranker,
                              ConfidenceEvaluator evaluator, LLMClient llmClient,
                              HumanAgentQueue agentQueue) {
        this.retriever = retriever;
        this.ranker = ranker;
        this.evaluator = evaluator;
        this.llmClient = llmClient;
        this.agentQueue = agentQueue;
    }

    /**
     * 核心路由逻辑
     */
    public RouteResult route(String userQuestion, String customerId) {
        // 1. 双重检索
        var rawResults = retriever.retrieve(userQuestion, 10);

        // 2. 融合排序
        var fusedResults = ranker.weightedFusion(rawResults, 5);

        // 3. 置信度评估
        double confidence = evaluator.evaluate(fusedResults);

        // 4. 根据置信度路由
        if (confidence >= AUTO_REPLY_THRESHOLD) {
            // 高置信度：AI 直接回复
            String answer = generateAnswer(userQuestion, fusedResults);
            return new RouteResult(RouteType.AUTO_REPLY, answer, confidence,
                fusedResults);

        } else if (confidence >= SUGGEST_THRESHOLD) {
            // 中置信度：AI 生成建议，人工确认后发送
            String suggestion = generateAnswer(userQuestion, fusedResults);
            return new RouteResult(RouteType.SUGGEST_FIRST, suggestion, confidence,
                fusedResults);

        } else if (confidence >= ESCALATE_THRESHOLD) {
            // 低置信度：转人工，附带 AI 建议
            String suggestion = generateAnswer(userQuestion, fusedResults);
            HumanAgent agent = agentQueue.assignAgent(customerId);
            return new RouteResult(RouteType.TRANSFER_WITH_SUGGESTION,
                suggestion, confidence, fusedResults, agent);

        } else {
            // 极低置信度：直接转人工
            HumanAgent agent = agentQueue.assignAgent(customerId);
            return new RouteResult(RouteType.DIRECT_TRANSFER, "", confidence,
                fusedResults, agent);
        }
    }

    private String generateAnswer(String question,
                                   List<DualRetriever.ScoredDocument> docs) {
        // 使用第一篇中的 RAG 生成模板
        String context = docs.stream()
            .map(d -> "[来源: " + d.sourceType() + " 相似度: "
                + String.format("%.2f", d.similarity()) + "]\n" + d.content())
            .reduce("", (a, b) -> a + "\n---\n" + b);

        String prompt = """
            你是客服助手。基于以下知识库内容，回答用户问题。
            回答要：专业、简洁、带操作指引。
            
            ## 知识库内容：
            %s
            
            ## 用户问题：
            %s
            """.formatted(context, question);

        return llmClient.chat(prompt);
    }

    public record RouteResult(
        RouteType type,
        String content,
        double confidence,
        List<DualRetriever.ScoredDocument> references,
        HumanAgent agent  // 仅在转人工时有值
    ) {
        public RouteResult(RouteType type, String content, double confidence,
                           List<DualRetriever.ScoredDocument> references) {
            this(type, content, confidence, references, null);
        }
    }

    public enum RouteType {
        AUTO_REPLY,                // AI 直接回复
        SUGGEST_FIRST,             // AI 生成建议，人工确认后发送
        TRANSFER_WITH_SUGGESTION,  // 转人工，附带 AI 建议
        DIRECT_TRANSFER            // 直接转人工
    }

    public interface LLMClient { String chat(String prompt); }
    public interface HumanAgent { String getName(); }
    public interface HumanAgentQueue {
        HumanAgent assignAgent(String customerId);
    }
}
```

## 八、完整的客服对话流程

```java
public class CustomerServiceRAGSystem {

    private final FAQParser faqParser;
    private final FAQEmbeddingEnhancer faqEnhancer;
    private final TicketImporter ticketImporter;
    private final TicketFilter ticketFilter;
    private final DualRetriever dualRetriever;
    private final FusionRanker fusionRanker;
    private final ConfidenceEvaluator confidenceEvaluator;
    private final IntelligentRouter router;

    /**
     * 系统初始化：索引 FAQ 和工单
     */
    public void initialize(String faqContent, String ticketDBUrl) {
        // 1. 解析并索引 FAQ
        List<FAQParser.FAQEntry> faqs = faqParser.parse(faqContent);
        System.out.println("已解析 FAQ: " + faqs.size() + " 条");

        for (FAQParser.FAQEntry faq : faqs) {
            String enhanced = faqEnhancer.enhance(faq);
            List<Float> embedding = embeddingService.embed(enhanced);
            faqStore.insert(new VectorDocument(
                faq.id(), embedding, faq.answer(),
                Map.of("question", faq.question(), "category", faq.category().name())
            ));
        }

        // 2. 导入并索引历史工单
        List<TicketRecord> tickets = ticketImporter.importFromDB(ticketDBUrl);
        System.out.println("已导入工单: " + tickets.size() + " 条");

        List<TicketRecord> validTickets = tickets.stream()
            .filter(ticketFilter::shouldIndex)
            .toList();
        System.out.println("有效工单: " + validTickets.size() + " 条");

        for (TicketRecord ticket : validTickets) {
            String text = TicketImporter.ticketToEmbeddingText(ticket);
            List<Float> embedding = embeddingService.embed(text);
            ticketStore.insert(new VectorDocument(
                ticket.ticketId(), embedding, text,
                Map.of("ticketId", ticket.ticketId(),
                    "satisfaction", ticket.satisfaction(),
                    "category", ticket.category())
            ));
        }

        System.out.println("系统初始化完成！");
    }

    /**
     * 处理客户消息
     */
    public IntelligentRouter.RouteResult handleMessage(String customerId, String message) {
        long start = System.currentTimeMillis();
        var result = router.route(message, customerId);
        long elapsed = System.currentTimeMillis() - start;
        System.out.printf("路由决策: %s, 置信度: %.2f, 耗时: %dms\n",
            result.type(), result.confidence(), elapsed);
        return result;
    }

    // 依赖注入
    private final VectorStore faqStore;
    private final VectorStore ticketStore;
    private final EmbeddingService embeddingService;

    public interface EmbeddingService { List<Float> embed(String text); }
}
```

## 九、总结

客服知识库 RAG 的四个核心设计：

1. **双通道检索**：FAQ 负责标准答案，历史工单负责实战经验
2. **融合排序**：加权融合或 RRF 算法，综合两种检索源的得分
3. **置信度路由**：根据置信度自动在 AI 回复和人工客服之间切换
4. **同义词扩展**：生成问题变体、关键词扩展，弥合"写法"和"问法"的鸿沟

这套系统让新客服在回答问题时，背后有一个"老手智囊团"——10 万条历史工单的经验随时待命。

> 下一篇预告：**法律文书 RAG：长文本合同的条款级检索与风险提示**——我们将挑战最复杂的文本类型：法务合同。条款级切割、风险自动标注、多合同条款对比，AI 法律顾问的幕后技术全揭秘！
