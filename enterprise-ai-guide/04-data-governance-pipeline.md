# 第 4 章 · 数据治理与流水线

---

> AI 系统的品质上限由数据决定。Prompt 再好，喂进去的是垃圾，出来的就是垃圾。本章覆盖数据准备、RAG 数据管道、Fine-tuning 数据策略。

---

## 4.1 数据金字塔

```
                ┌─────────────┐
                │  Fine-tuning │  ← 最有价值，成本最高
                │   训练数据    │     (10K-100K 高质量问答对)
                ├─────────────┤
                │  评估数据     │  ← 定义"好"的标准
                │  Eval Dataset│     (100-1000 条精心标注)
                ├─────────────┤
                │  提示词示例   │  ← Few-shot 的质量保证
                │  Few-shot    │     (5-20 条精华示例)
                ├─────────────┤
                │  知识库       │  ← RAG 的数据底座
                │  Knowledge   │     (文档、Wiki、代码、合同...)
                ├─────────────┤
                │  企业原始数据  │  ← 最低层，需要清洗/结构化
                │  Raw Data    │     (日志、邮件、工单、数据库...)
                └─────────────┘
```

**核心原则**：每一层数据向上汇聚时，都要经过筛选和标注。顶层的 Fine-tuning 数据质量直接决定模型表现。

---

## 4.2 知识库构建 (RAG 的数据底座)

```java
/**
 * 企业知识库构建流水线
 * 
 * 这是 RAG 系统最重要的基础设施——垃圾进，垃圾出
 */
@Service
public class KnowledgeBasePipeline {

    // ===== Phase 1: 数据采集 =====
    public void collect(DataSource source) {
        switch (source.type()) {
            case CONFLUENCE -> crawlConfluence(source.url());
            case SHAREPOINT -> syncSharePoint(source.path());
            case GITHUB -> cloneAndParse(source.repo());
            case DATABASE -> exportAndFormat(source.connection());
            case PDF -> batchOCR(source.directory());
            case SLACK -> archiveMessages(source.channel());
        }
    }

    // ===== Phase 2: 数据清洗 =====
    public List<CleanDocument> clean(List<RawDocument> rawDocs) {
        return rawDocs.stream()
            .map(doc -> {
                // 1. 去除格式噪音 (HTML 标签、特殊字符)
                String text = stripFormatting(doc.content());

                // 2. 去除页眉页脚、水印、页码
                text = removeHeadersFooters(text);

                // 3. 表格结构化 (表格转 Markdown Table)
                text = tablesToMarkdown(text);

                // 4. 处理编码、全角半角统一
                text = normalizeEncoding(text);

                // 5. 去重 (标题相似度 > 0.95 的文档合并)
                return new CleanDocument(doc.id(), text, doc.metadata());
            })
            .filter(doc -> !doc.content().isBlank())
            .toList();
    }

    // ===== Phase 3: 文档切片 (Chunking) =====
    /**
     * Chunking 策略——RAG 最重要的调优参数
     */
    public List<Chunk> chunk(List<CleanDocument> docs, ChunkingConfig config) {
        return docs.stream()
            .flatMap(doc -> chunkDocument(doc, config).stream())
            .toList();
    }

    private List<Chunk> chunkDocument(CleanDocument doc, ChunkingConfig config) {
        return switch (config.strategy()) {
            case FIXED_SIZE -> fixedSizeChunk(doc, config.chunkSize(), config.overlap());
            case SENTENCE -> sentenceChunk(doc, config.maxSentences());
            case SEMANTIC -> semanticChunk(doc);
            case RECURSIVE -> recursiveChunk(doc, config.chunkSize());
            case DOCUMENT_STRUCTURE -> hierarchyChunk(doc);
        };
    }

    /**
     * 固定大小切片 + 重叠 (最常用)
     * 
     * 参数设置原则:
     *   chunkSize: 256-512 token (短) 或 512-1024 token (长)
     *   overlap:   chunkSize 的 10-20%
     * 
     * 太短 → 丢失上下文，答案碎片化
     * 太长 → 检索精度下降，噪音增加
     */
    private List<Chunk> fixedSizeChunk(CleanDocument doc, int chunkSize, int overlap) {
        List<Chunk> chunks = new ArrayList<>();
        String[] tokens = doc.content().split("\\s+");
        int start = 0;

        while (start < tokens.length) {
            int end = Math.min(start + chunkSize, tokens.length);
            String chunkText = String.join(" ",
                Arrays.copyOfRange(tokens, start, end));
            chunks.add(new Chunk(
                doc.id() + "_" + chunks.size(),
                chunkText,
                doc.metadata()
            ));
            start += (chunkSize - overlap);
        }
        return chunks;
    }

    // ===== Phase 4: 向量化 =====
    public void embed(List<Chunk> chunks, EmbeddingModel model) {
        int batchSize = 100;  // 批量处理
        for (int i = 0; i < chunks.size(); i += batchSize) {
            int end = Math.min(i + batchSize, chunks.size());
            List<Chunk> batch = chunks.subList(i, end);

            List<String> texts = batch.stream().map(Chunk::text).toList();
            List<float[]> embeddings = model.embedBatch(texts);

            // 写入向量数据库
            for (int j = 0; j < batch.size(); j++) {
                vectorDB.insert(
                    batch.get(j).id(),
                    embeddings.get(j),
                    batch.get(j).metadata()
                );
            }
        }
    }

    // ===== Phase 5: 索引构建 =====
    public void buildIndex() {
        // 向量索引 (HNSW / IVF)
        vectorDB.createIndex(IndexType.HNSW, Map.of("M", 16, "efConstruction", 200));

        // 如果支持混合检索，还需要建立倒排索引 (BM25)
        if (vectorDB.supportsHybridSearch()) {
            vectorDB.createKeywordIndex();
        }
    }
}

/**
 * Chunking 策略决策指南
 */
public class ChunkingStrategyGuide {

    public static ChunkingConfig forScenario(Scenario scenario) {
        return switch (scenario) {
            // 简短 FAQ 问答
            case FAQ -> new ChunkingConfig(Strategy.FIXED_SIZE, 256, 50);

            // 技术文档、API 文档
            case TECHNICAL_DOC -> new ChunkingConfig(Strategy.DOCUMENT_STRUCTURE, 512, 100);

            // 法律合同(需要精确边界)
            case LEGAL -> new ChunkingConfig(Strategy.SENTENCE, 5, 0);

            // 长文章、报告
            case LONG_FORM -> new ChunkingConfig(Strategy.SEMANTIC, 1024, 200);

            // 代码
            case CODE -> new ChunkingConfig(Strategy.DOCUMENT_STRUCTURE, 1024, 200);
            // 按函数/类边界切分，而不是按固定字符数
        };
    }
}
```

---

## 4.3 数据质量检查

```java
/**
 * 数据质量门槛——必须通过的检查
 */
public class DataQualityGate {

    /**
     * 每个 Chunk 必须通过质量检查才能入库
     */
    public QualityReport check(Chunk chunk) {
        List<QualityIssue> issues = new ArrayList<>();

        // 1. 最小长度检查
        if (chunk.text().length() < 50) {
            issues.add(new QualityIssue("TOO_SHORT", "chunk 过短 ({})，可能无法提供有效信息"));
        }

        // 2. 乱码检测
        if (containsGarbledText(chunk.text())) {
            issues.add(new QualityIssue("GARBLED", "包含乱码或不可读字符"));
        }

        // 3. 重复内容检测
        double minHash = computeMinHash(chunk.text());
        if (nearDuplicate(minHash)) {
            issues.add(new QualityIssue("DUPLICATE", "与已有 Chunk 高度重复"));
        }

        // 4. 语言一致性
        String language = detectLanguage(chunk.text());
        if (!allowedLanguages.contains(language)) {
            issues.add(new QualityIssue("LANG", "语言不符合要求: " + language));
        }

        // 5. PII 泄露检测
        if (containsPII(chunk.text())) {
            issues.add(new QualityIssue("PII", "包含个人敏感信息，已自动脱敏"));
            // 自动脱敏后放行 (可配置)
        }

        return new QualityReport(
            issues.isEmpty(),
            issues
        );
    }
}
```

---

## 4.4 Fine-tuning 数据策略

```java
/**
 * Fine-tuning 数据——最昂贵的 AI 资产
 */
public class FinetuningDataStrategy {

    /**
     * Fine-tuning 数据的 5 个来源
     */
    public List<FinetuningDataSource> identifySources() {
        return List.of(
            // 来源 1: 生产日志 (最丰富)
            // 收集已上线的 LLM 交互日志，筛选高质量对话
            new FinetuningDataSource("生产日志", 5_000, quality -> quality >= 4),

            // 来源 2: 专家标注 (最精准但最贵)
            // 让领域专家 (律师、医生、资深工程师) 编写标准问答对
            new FinetuningDataSource("专家标注", 500, quality -> quality >= 5),

            // 来源 3: 合成数据 (最灵活)
            // 用强模型 (GPT-4) 生成训练数据，再人工审核
            new FinetuningDataSource("GPT-4 合成", 2_000, quality -> quality >= 4),

            // 来源 4: 历史数据迁移 (最高效)
            // 原有系统的优质问答、工单处理记录
            new FinetuningDataSource("历史工单", 10_000, quality -> quality >= 3),

            // 来源 5: 用户反馈 (最真实)
            // 用户点赞的回答、修正后的回复
            new FinetuningDataSource("用户反馈", 1_000, quality -> quality >= 4)
        );
    }

    /**
     * Fine-tuning 数据格式
     */
    public record FinetuningExample(
        List<Message> messages  // 多轮对话
    ) {
        public String toChatML() {
            StringBuilder sb = new StringBuilder();
            for (Message msg : messages) {
                sb.append("<|im_start|>").append(msg.role()).append("\n");
                sb.append(msg.content()).append("\n");
                sb.append("<|im_end|>\n");
            }
            return sb.toString();
        }
    }

    // 示例: 客服 Fine-tuning 数据
    private static FinetuningExample customerServiceExample() {
        return new FinetuningExample(List.of(
            new Message("system", """
                你是电商客服助手。退货政策:
                - 7天无理由退货
                - 电子产品需原包装完好
                - 生鲜食品不支持退货
                """),
            new Message("user", "我买的手机用了 3 天想退，可以吗？"),
            new Message("assistant", """
                可以退货的！我们支持 7 天无理由退货。
                不过需要确认:
                1. 原包装和配件是否完整？
                2. 手机没有人为损坏吧？
                
                如果都满足，我帮您生成退货单！
                """)
        ));
    }

    /**
     * 数据质量红线
     */
    public static class FinetuningDataQC {

        // 必须检查项
        public List<String> mandatoryChecks() {
            return List.of(
                "回复长度 > 10 字",
                "不包含 PII",
                "不包含有害内容",
                "与 System Prompt 一致 (不违反给定规则)",
                "不存在逻辑矛盾 (不能上一句说可以退货，下一句说不可以)",
                "格式正确 (ChatML 格式无误)"
            );
        }

        // 推荐检查项
        public List<String> recommendedChecks() {
            return List.of(
                "回复有逻辑推理过程 (不是简单的是/否)",
                "不包含幻觉 (确认所有陈述有事实依据)",
                "风格一致 (不会一会儿正式一会儿口语化)",
                "多样性 (同类问题有不同表达方式)"
            );
        }
    }
}
```

---

## 4.5 数据管道自动化

```java
/**
 * 企业级 AI 数据管道的 CI/CD
 * 
 * 知识库不是一次性构建的，需要持续更新
 */
@Component
public class DataPipelineScheduler {

    // ===== 全量重建: 每周日凌晨 =====
    @Scheduled(cron = "0 0 2 * * 0")
    public void weeklyFullRebuild() {
        log.info("开始全量重建知识库...");
        pipeline.fullRebuild();
        log.info("全量重建完成");
    }

    // ===== 增量更新: 每小时 =====
    @Scheduled(cron = "0 0 * * * ?")
    public void hourlyIncrementalUpdate() {
        // 检测数据源变化
        List<DataChange> changes = dataSource.detectChanges(lastSyncTime);

        if (changes.isEmpty()) return;

        // 增量处理变化的文档
        for (DataChange change : changes) {
            switch (change.type()) {
                case CREATED, UPDATED -> pipeline.processNew(change.document());
                case DELETED -> pipeline.remove(change.documentId());
            }
        }

        lastSyncTime = Instant.now();
    }

    // ===== 质量监控 =====
    @Scheduled(cron = "0 0 9 * * ?")
    public void dailyQualityReport() {
        DataQualityMetrics metrics = qualityChecker.calculateMetrics();
        // 告警: 质量指标低于阈值
        if (metrics.avgRelevanceScore() < 0.7) {
            alertService.send("知识库质量下降",
                "平均相关性分数: %.2f, 阈值: 0.7".formatted(metrics.avgRelevanceScore()));
        }
    }
}
```

---

> **下一章**：[安全合规与风险管控](05-security-compliance-risk.md)
