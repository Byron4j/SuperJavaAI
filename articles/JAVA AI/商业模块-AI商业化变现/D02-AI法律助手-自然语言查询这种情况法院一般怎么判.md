# AI法律助手：自然语言查询“这种情况法院一般怎么判”

> 输入一段案情描述，30秒给你匹配最相似的判例、法条和判决倾向——法律检索从未如此简单。

---

## 一、行业痛点：法条好查，判例难找

做律师的朋友经常吐槽一件事：查法条太容易了，北大法宝、威科先行、无讼，哪个不能查？但客户真正想问的从来不是“民法典第几条怎么写的”，而是——

**“我这种情况，法院一般怎么判？”**

举个例子。一个创业者的合伙人带着核心团队跳槽去了竞品公司，还带走了部分客户数据。创业者想告他，但不确定胜算多大。他问律师三个问题：

1. 这种情况算不算违反竞业限制？
2. 法院一般支持多少赔偿？
3. 有没有类似的判例可以参考？

律师打开法律数据库，输入关键词“竞业限制”“跳槽”“带走客户”——跳出来3000多条结果，90%不相关，剩下10%需要逐篇阅读摘要，再从中筛选真正有价值的判例。一套操作下来，至少2小时。

**核心矛盾**：传统法律检索是“关键词匹配”，但用户表达案情用的是“自然语言”。这两者之间存在巨大的语义鸿沟。

```
用户表达：          "我合伙人和客户私下签合同，绕过公司"
关键词检索匹配：      "合伙人" AND "客户" AND "合同"
实际相关的判例关键词： "竞业禁止" "忠实义务" "自我交易" "公司机会原则"
                    ↑ 用户根本不知道这些专业术语
```

---

## 二、解决方案：AI判例搜索引擎

我们的产品叫 **CaseInsight**——一个用自然语言查询、用AI匹配相似判例的法律助手。核心能力：

1. **自然语言输入案情**，自动提取法律要素
2. **向量语义检索**，突破关键词限制找到真正相关的判例
3. **LLM分析判决倾向**，告诉你“这种情况法院一般怎么判”
4. **类案推荐**，输出结构化的裁判要旨对比

### 核心流程

```
用户输入案情（自然语言）
        ↓
   【法律要素提取】LLM提取：案由/当事人/事实/诉求
        ↓
   【向量语义检索】Milvus搜索最相似的Top-50判例
        ↓
   【精排重排序】Cross-encoder模型+法律特征重排
        ↓
   【判决倾向分析】LLM对比分析→"支持率约65%"
        ↓
   【结果展示】相似判例列表 + AI分析报告
```

---

## 三、技术架构与实现

### 3.1 判例数据预处理管道

判例数据的质量决定了检索效果。我们用以下管道将原始裁判文书处理为可检索的知识库：

```java
@Service
public class CasePreprocessingPipeline {
    
    private final CaseRepository caseRepository;
    private final EmbeddingService embeddingService;
    private final MilvusClient milvusClient;
    
    /**
     * 判例预处理管道
     * 1. 文本清洗 → 2. 要素提取 → 3. 分段 → 4. 向量化 → 5. 入库
     */
    public void processCase(RawJudgment judgment) {
        // Step 1: 文本清洗
        String cleaned = cleanText(judgment.getFullText());
        
        // Step 2: 法律要素提取
        LegalElements elements = extractLegalElements(cleaned);
        
        // Step 3: 多粒度分段（全文/事实部分/裁判理由/判决结果）
        CaseSegments segments = segmentByStructure(cleaned);
        
        // Step 4: 向量化
        // 使用不同embedding模型处理不同部分
        EmbeddingVector caseVector = embeddingService.embed(
            buildEmbeddingText(elements, segments.facts())
        );
        
        // Step 5: 存入向量数据库
        milvusClient.insert(buildMilvusEntity(
            judgment.getCaseId(), caseVector, elements, segments
        ));
        
        // Step 6: 结构化信息存MySQL
        caseRepository.save(buildCaseEntity(judgment, elements, segments));
    }
    
    /**
     * 法律要素提取 - 用LLM从判例中提取结构化信息
     */
    public LegalElements extractLegalElements(String text) {
        return chatClient.prompt()
            .system("""
                你是法律文本分析专家。从裁判文书中提取以下结构化信息：
                
                请以JSON格式返回：
                {
                  "caseType": "案由（如'合同纠纷/劳动争议/知识产权'）",
                  "plaintiff": "原告类型（自然人/公司/政府）",
                  "defendant": "被告类型",
                  "keyFacts": ["关键事实1", "关键事实2"],
                  "legalIssues": ["争议焦点1", "争议焦点2"],
                  "applicableLaws": ["适用法条1", "适用法条2"],
                  "judgmentResult": "判决结果（支持/部分支持/驳回/调解）",
                  "damagesAmount": "赔偿金额（如有）",
                  "courtLevel": "法院层级",
                  "judgmentYear": "判决年份"
                }
                """)
            .user(text)
            .call()
            .entity(LegalElements.class);
    }
    
    @Data
    public static class LegalElements {
        private String caseType;
        private String plaintiff;
        private String defendant;
        private List<String> keyFacts;
        private List<String> legalIssues;
        private List<String> applicableLaws;
        private String judgmentResult;
        private String damagesAmount;
        private String courtLevel;
        private Integer judgmentYear;
    }
    
    @Data
    public static class CaseSegments {
        private String fullText;
        private String facts;        // 事实与理由部分
        private String reasoning;    // 本院认为部分
        private String result;       // 判决结果部分
    }
    
    private CaseSegments segmentByStructure(String text) {
        // 裁判文书有固定结构，按"经审理查明""本院认为""判决如下"等关键词分段
        CaseSegments segments = new CaseSegments();
        segments.setFullText(text);
        
        // 提取事实部分
        Pattern factsPattern = Pattern.compile(
            "(?:经审理查明|原告诉称|被告辩称|经查)(.*?)(?:本院认为)",
            Pattern.DOTALL
        );
        Matcher factsMatcher = factsPattern.matcher(text);
        if (factsMatcher.find()) {
            segments.setFacts(factsMatcher.group(1).trim());
        }
        
        // 提取裁判理由
        Pattern reasoningPattern = Pattern.compile(
            "(?:本院认为)(.*?)(?:判决如下|裁定如下|依照)",
            Pattern.DOTALL
        );
        Matcher reasoningMatcher = reasoningPattern.matcher(text);
        if (reasoningMatcher.find()) {
            segments.setReasoning(reasoningMatcher.group(1).trim());
        }
        
        // 提取判决结果
        Pattern resultPattern = Pattern.compile(
            "(?:判决如下|裁定如下)(.*?)(?:审判长|本判决|$)",
            Pattern.DOTALL
        );
        Matcher resultMatcher = resultPattern.matcher(text);
        if (resultMatcher.find()) {
            segments.setResult(resultMatcher.group(1).trim());
        }
        
        return segments;
    }
    
    private String buildEmbeddingText(LegalElements elements, String facts) {
        return String.format("""
            案由：%s
            关键事实：%s
            争议焦点：%s
            案件描述：%s
            """,
            elements.getCaseType(),
            String.join("；", elements.getKeyFacts()),
            String.join("；", elements.getLegalIssues()),
            facts
        );
    }
}
```

### 3.2 混合检索策略：向量检索 + 关键词检索 + 法律特征过滤

单一的向量检索在专业领域仍有不足。我们采用混合检索：

```java
@Service
public class HybridSearchService {
    
    private final MilvusClient milvusClient;
    private final EmbeddingService embeddingService;
    private final ChatClient chatClient;
    private final LegalFeatureExtractor featureExtractor;
    
    /**
     * 混合检索：语义 + 关键词 + 法律特征
     * 
     * @param userQuery 用户的自然语言案情描述
     * @param filters   可选的过滤条件（案由/法院层级/年份等）
     */
    public SearchResult hybridSearch(String userQuery, SearchFilters filters) {
        
        // Step 1: 用LLM提取查询中的法律要素
        QueryLegalElements queryElements = extractQueryElements(userQuery);
        
        // Step 2: 向量语义检索（粗排Top-100）
        EmbeddingVector queryVector = embeddingService.embed(userQuery);
        List<SearchHit> semanticHits = milvusClient.search(
            "case_embeddings",
            queryVector,
            100,
            buildMilvusFilter(filters)
        );
        
        // Step 3: 关键词检索（BM25或用Elasticsearch）
        List<SearchHit> keywordHits = elasticsearchService.search(
            "judgments_index",
            buildKeywordQuery(queryElements),
            100
        );
        
        // Step 4: 结果融合（RRF: Reciprocal Rank Fusion）
        List<SearchHit> fusedHits = reciprocalRankFusion(
            semanticHits, keywordHits, 60
        );
        
        // Step 5: 精排（Cross-encoder + 法律特征）
        List<ScoredHit> reranked = rerank(fusedHits, userQuery, queryElements);
        
        // Step 6: LLM分析判决倾向
        JudgmentTendency tendency = analyzeTendency(reranked.subList(0, 10), 
                                                      queryElements);
        
        return new SearchResult(reranked.subList(0, 20), tendency, queryElements);
    }
    
    /**
     * 用户查询的法律要素提取
     */
    private QueryLegalElements extractQueryElements(String query) {
        return chatClient.prompt()
            .system("""
                分析用户描述的法律问题，提取结构化要素：
                {
                  "caseTypes": ["案由1", "案由2"],
                  "parties": {"我方": "角色", "对方": "角色"},
                  "coreFacts": ["核心事实"],
                  "legalQuestions": ["法律问题"],
                  "keywords": ["专业法律关键词"],
                  "expectedResult": "期望结果"
                }
                """)
            .user(query)
            .call()
            .entity(QueryLegalElements.class);
    }
    
    /**
     * RRF融合算法
     * Reciprocal Rank Fusion: score = Σ 1/(k + rank_i)
     */
    private List<SearchHit> reciprocalRankFusion(
            List<SearchHit> list1, List<SearchHit> list2, int topK) {
        
        Map<String, Double> scores = new HashMap<>();
        double k = 60.0;
        
        for (int i = 0; i < list1.size(); i++) {
            String id = list1.get(i).getId();
            scores.merge(id, 1.0 / (k + i + 1), Double::sum);
        }
        
        for (int i = 0; i < list2.size(); i++) {
            String id = list2.get(i).getId();
            scores.merge(id, 1.0 / (k + i + 1), Double::sum);
        }
        
        return scores.entrySet().stream()
            .sorted(Map.Entry.<String, Double>comparingByValue().reversed())
            .limit(topK)
            .map(e -> findHitById(e.getKey(), list1, list2))
            .collect(Collectors.toList());
    }
    
    /**
     * 精排：用LLM做Cross-encoder评估相关性
     */
    private List<ScoredHit> rerank(List<SearchHit> candidates, 
                                    String query, QueryLegalElements elements) {
        
        // 批量评估：每次送10个判例给LLM打分
        List<ScoredHit> scored = new ArrayList<>();
        
        for (int i = 0; i < candidates.size(); i += 10) {
            int end = Math.min(i + 10, candidates.size());
            List<SearchHit> batch = candidates.subList(i, end);
            
            String batchScores = chatClient.prompt()
                .system("""
                    你是专业的法律判例评估专家。评估以下判例与用户案情的相关度。
                    
                    评分标准：
                    - 5分：案情高度相似，法律关系相同，极具参考价值
                    - 4分：法律关系相同，事实要素有部分重叠
                    - 3分：涉及相同法律问题但案情不同
                    - 2分：仅部分相关
                    - 1分：基本不相关
                    
                    请以JSON格式返回：[{"caseId":"xxx", "score":5, "reason":"..."}]
                    """)
                .user("""
                    用户案情：%s
                    待评估判例：
                    %s
                    """.formatted(
                        query,
                        batch.stream()
                            .map(h -> "判例ID: %s\n摘要: %s\n".formatted(
                                h.getId(), h.getSummary()))
                            .collect(Collectors.joining("\n---\n"))
                    ))
                .call()
                .content();
            
            scored.addAll(parseBatchScores(batchScores, batch));
        }
        
        scored.sort(Comparator.comparingInt(ScoredHit::getScore).reversed());
        return scored;
    }
}
```

### 3.3 判决倾向分析

```java
@Service
public class TendencyAnalysisService {
    
    private final ChatClient chatClient;
    
    /**
     * 分析相似判例的判决倾向
     * 回答"这种情况法院一般怎么判"
     */
    public JudgmentTendency analyzeTendency(
            List<ScoredHit> similarCases, 
            QueryLegalElements userQuery) {
        
        // 从相似判例中提取判决信息
        List<CaseJudgmentInfo> judgments = loadJudgmentInfo(similarCases);
        
        // 统计分析
        long supportCount = judgments.stream()
            .filter(j -> j.getResult().contains("支持")).count();
        long partialCount = judgments.stream()
            .filter(j -> j.getResult().contains("部分支持")).count();
        long rejectCount = judgments.stream()
            .filter(j -> j.getResult().contains("驳回")).count();
        
        double supportRate = (double) supportCount / judgments.size();
        
        // 赔偿金额统计（如有）
        DoubleSummaryStatistics damagesStats = judgments.stream()
            .filter(j -> j.getDamagesAmount() > 0)
            .mapToDouble(CaseJudgmentInfo::getDamagesAmount)
            .summaryStatistics();
        
        // LLM综合分析
        String analysis = generateAnalysis(
            userQuery, judgments, supportRate, damagesStats
        );
        
        return JudgmentTendency.builder()
            .supportRate(supportRate)
            .totalCases(judgments.size())
            .supportCount((int) supportCount)
            .partialCount((int) partialCount)
            .rejectCount((int) rejectCount)
            .avgDamages(damagesStats.getCount() > 0 ? 
                        damagesStats.getAverage() : null)
            .keyFactors(extractKeyFactors(judgments))
            .llmAnalysis(analysis)
            .topCases(similarCases.stream().limit(10).toList())
            .build();
    }
    
    private String generateAnalysis(QueryLegalElements query,
                                     List<CaseJudgmentInfo> judgments,
                                     double supportRate,
                                     DoubleSummaryStatistics damagesStats) {
        
        String caseSummary = judgments.stream()
            .map(j -> """
                案号：%s | 结果：%s | 赔偿：%s
                裁判要旨：%s
                """.formatted(j.getCaseNumber(), j.getResult(),
                             j.getDamagesAmount() > 0 ? 
                             j.getDamagesAmount() + "元" : "无",
                             j.getKeyReasoning()))
            .collect(Collectors.joining("\n---\n"));
        
        return chatClient.prompt()
            .system("你是资深法官，擅长分析类案的裁判规律")
            .user("""
                基于以下相似判例，分析该类案件的裁判倾向，回答"法院一般怎么判"：
                
                用户问题：%s
                
                统计概况：
                - 检索到%d个高度相似的判例
                - 支持率：%.0f%%
                - 平均赔偿金额：%s元
                
                相关判例摘要：
                %s
                
                请从以下角度分析：
                1. 法院支持的关键因素是什么？
                2. 法院驳回的常见理由是什么？
                3. 赔偿金额的考量因素有哪些？
                4. 对该案件的胜诉概率和策略建议
                """.formatted(
                    query.getCoreFacts(),
                    judgments.size(),
                    supportRate * 100,
                    damagesStats.getCount() > 0 ? 
                        String.format("%.0f", damagesStats.getAverage()) : "暂无数据",
                    caseSummary
                ))
            .call()
            .content();
    }
}
```

### 3.4 API接口设计

```java
@RestController
@RequestMapping("/api/v1/case-search")
public class CaseSearchController {
    
    private final HybridSearchService searchService;
    private final TendencyAnalysisService analysisService;
    
    /**
     * 自然语言判例搜索
     */
    @PostMapping("/search")
    public ResponseEntity<SearchResult> search(@RequestBody SearchRequest request) {
        SearchFilters filters = SearchFilters.builder()
            .caseTypes(request.getCaseTypes())
            .courtLevel(request.getCourtLevel())
            .yearRange(request.getYearFrom(), request.getYearTo())
            .region(request.getRegion())
            .build();
        
        SearchResult result = searchService.hybridSearch(
            request.getQuery(), filters
        );
        
        return ResponseEntity.ok(result);
    }
    
    /**
     * 判决倾向分析（直接给结论）
     */
    @PostMapping("/analyze-tendency")
    public ResponseEntity<JudgmentTendency> analyzeTendency(
            @RequestBody TendencyRequest request) {
        
        return ResponseEntity.ok(
            analysisService.analyzeTendency(
                request.getSimilarCaseIds(), 
                request.getQueryElements()
            )
        );
    }
    
    @Data
    public static class SearchRequest {
        @NotBlank private String query;          // 自然语言案情描述
        private List<String> caseTypes;           // 可选：限定案由
        private String courtLevel;                // 可选：法院层级
        private Integer yearFrom;                 // 可选：起始年份
        private Integer yearTo;                   // 可选：结束年份
        private String region;                    // 可选：地域
    }
}
```

---

## 四、技术难点与解决思路

### 难点1：判例数据获取

裁判文书虽然公开，但获取高质量的结构化数据需要下功夫：

```java
@Component
public class JudgmentDataCollector {
    
    /**
     * 从中国裁判文书网API批量获取判例
     * 注意：需要遵守robots协议和使用规范
     */
    public List<RawJudgment> fetchJudgments(JudgmentQuery query) {
        // 方案A：通过官方API（如有）
        // 方案B：与法律数据服务商合作（北大法宝/无讼/威科先行API）
        // 方案C：爬取公开数据（需合规处理）
        
        // 以方案B为例：调用第三方API
        return lawDataProvider.search(
            query.getKeyword(),
            query.getCaseType(),
            query.getPage(),
            query.getPageSize()
        );
    }
    
    /**
     * 增量更新机制：每天自动拉取新增判例
     */
    @Scheduled(cron = "0 0 2 * * ?")  // 每天凌晨2点
    public void incrementalUpdate() {
        LocalDate lastUpdate = getLastUpdateDate();
        List<RawJudgment> newJudgments = fetchRecentJudgments(lastUpdate);
        
        for (RawJudgment judgment : newJudgments) {
            try {
                preprocessingPipeline.processCase(judgment);
            } catch (Exception e) {
                log.error("判例处理失败: {}", judgment.getCaseId(), e);
            }
        }
        
        updateLastUpdateDate(LocalDate.now());
    }
}
```

### 难点2：长文本检索优化

裁判文书通常有几千至几万字，直接用全文做embedding效果不好：

```java
@Component
public class LongTextEmbeddingStrategy {
    
    /**
     * 长文本的三种Embedding策略
     */
    public EmbeddingVector embedLongJudgment(CaseSegments segments,
                                              LegalElements elements) {
        
        // 策略1: 结构化摘要embedding（推荐）
        String summary = buildStructuredSummary(elements);
        EmbeddingVector v1 = embeddingService.embed(summary);
        
        // 策略2: 事实部分embedding
        EmbeddingVector v2 = embeddingService.embed(
            truncateText(segments.getFacts(), 2000)
        );
        
        // 策略3: 裁判要旨embedding
        EmbeddingVector v3 = embeddingService.embed(
            truncateText(segments.getReasoning(), 1500)
        );
        
        // 加权融合（结构化摘要权重最高）
        return EmbeddingVector.weightedAverage(
            List.of(v1, v2, v3),
            List.of(0.5, 0.3, 0.2)
        );
    }
    
    private String buildStructuredSummary(LegalElements elements) {
        return String.format("""
            案由：%s
            核心事实：%s
            争议焦点：%s
            适用法律：%s
            判决结果：%s
            """,
            elements.getCaseType(),
            String.join("；", elements.getKeyFacts()),
            String.join("；", elements.getLegalIssues()),
            String.join("；", elements.getApplicableLaws()),
            elements.getJudgmentResult()
        );
    }
}
```

### 难点3：幻觉控制

法律场景对准确性要求极高，绝不能让LLM“编造”法条：

```java
@Component
public class LegalHallucinationGuard {
    
    /**
     * LLM输出校验：确保引用的法条真实存在
     */
    public ValidatedResponse validateLLMOutput(String llmOutput) {
        
        // 提取所有法条引用
        Pattern lawPattern = Pattern.compile(
            "《([^》]+)》第([^条]+)条"
        );
        Matcher matcher = lawPattern.matcher(llmOutput);
        
        List<LawReference> references = new ArrayList<>();
        while (matcher.find()) {
            references.add(new LawReference(
                matcher.group(1), matcher.group(2)
            ));
        }
        
        // 在法条数据库中校验每个引用
        List<LawReference> invalidRefs = new ArrayList<>();
        for (LawReference ref : references) {
            if (!lawDatabaseService.verifyLawExists(
                    ref.getLawName(), ref.getArticle())) {
                invalidRefs.add(ref);
            }
        }
        
        if (!invalidRefs.isEmpty()) {
            log.warn("发现LLM虚构的法条引用: {}", invalidRefs);
            // 标记为需要人工审核
            return ValidatedResponse.needsReview(
                "存在无法验证的法条引用：" + 
                invalidRefs.stream()
                    .map(LawReference::toString)
                    .collect(Collectors.joining(", "))
            );
        }
        
        return ValidatedResponse.valid();
    }
    
    /**
     * 判例信息也必须可溯源
     */
    public boolean verifyCaseReference(String caseNumber) {
        return caseRepository.existsByCaseNumber(caseNumber);
    }
}
```

---

## 五、商业模式设计

### 定价策略

| 版本 | 价格 | 核心功能 |
|------|------|----------|
| **免费版** | ¥0 | 每日3次搜索，基础匹配 |
| **专业版** | ¥199/月 | 无限搜索 + AI判决分析 + 类案对比 |
| **团队版** | ¥999/月 | 5人协作 + 自定义知识库 + 报告导出 |
| **企业版** | 定制 | API接入 + 私有化部署 + 专属数据 |

### 增长飞轮

```
用户搜索 → 产生反馈（有用/无用/收藏）
    ↓
反馈数据 → 优化排序算法 → 更好的搜索结果
    ↓
更好的结果 → 更多用户 → 更多反馈
    ↓
反馈沉淀 → 法律要素标注数据 → 微调专属模型
    ↓
更好的模型 → 更强的竞争壁垒
```

---

## 六、MVP两周计划

| 阶段 | 天数 | 目标 |
|------|------|------|
| 数据准备 | Day 1-3 | 爬取/购买1万份精选判例，完成预处理入库 |
| 核心检索 | Day 4-7 | 向量检索+混合排序+API接口 |
| AI分析 | Day 8-10 | LLM判决倾向分析+幻觉控制 |
| 前端上线 | Day 11-14 | Vue3搜索页+结果页+分析报告页 |

---

> **下一篇预告**：《律所AI提效——内部工具帮律所省了30%的人力成本》，看我们如何帮一家百人所打造内部AI工具箱，从合同审查到法律研究到文书生成，实现全员提效。
