# 第 3 章 · 技术选型与模型评估

---

> 2025 年已有数百个模型和推理框架。选错一个，可能导致 2-3 个月的无用功。本章提供结构化的选型框架。

---

## 3.1 LLM 选型五维评估法

```java
/**
 * LLM 选型评分卡
 * 每个维度 1-5 分，总分决定推荐度
 */
public class LLMScoreCard {

    public record ModelScore(
        String modelName,
        int capability,        // 能力: 推理、代码、多语言...
        int latency,           // 延迟: 首 Token 时间、生成速度
        int cost,              // 成本: 每 1M Token 价格
        int reliability,       // 可靠性: 可用性 SLA、响应稳定性
        int ecosystem          // 生态: 工具链、社区、文档
    ) {
        public int total() {
            return capability + latency + cost + reliability + ecosystem;
        }
    }

    /**
     * 主流模型评分表 (2025 Q2 视角)
     */
    public static List<ModelScore> scoreAllModels() {
        return List.of(
            // ===== 闭源 (Cloud API) =====
            new ModelScore("GPT-4o",           5, 3, 2, 5, 5),  // 最强综合能力
            new ModelScore("Claude Sonnet 4.5",  5, 3, 2, 5, 4),  // 最强代码/Agent
            new ModelScore("Claude Haiku 4.5",   4, 5, 4, 5, 4),  // 性价比之王
            new ModelScore("Gemini 2.5 Pro",   5, 3, 3, 4, 5),  // 最强上下文长度
            new ModelScore("DeepSeek-V3",      5, 3, 5, 4, 3),  // 开源价格碾压

            // ===== 开源 (Self-hosted) =====
            new ModelScore("DeepSeek-R1",      5, 2, 5, 4, 3),  // 推理最强开源
            new ModelScore("LLaMA 3.1 70B",    4, 3, 4, 4, 5),  // 生态最完善
            new ModelScore("LLaMA 3.1 8B",    3, 5, 5, 4, 5),   // 本地部署性价比
            new ModelScore("Qwen 2.5 72B",     4, 2, 5, 4, 4),  // 中文最强开源
            new ModelScore("Qwen 2.5 7B",      3, 5, 5, 4, 4),  // 轻量中文部署
            new ModelScore("Mistral Large 2",   4, 3, 4, 4, 4),  // 欧洲合规首选
            new ModelScore("Phi-4 14B",         3, 4, 5, 4, 3)   // 边缘推理王者
        );
    }
}
```

---

## 3.2 推理框架选型

```java
/**
 * 自建推理的核心选型维度
 */
public class InferenceFrameworkComparison {

    /**
     * 推理框架对比矩阵 (2025 Q2)
     */
    public static Map<String, FrameworkProfile> compare() {
        return Map.of(
            "vLLM", new FrameworkProfile(
                "vLLM", "Apache 2.0",
                true,    // PagedAttention
                true,    // 连续批处理 (Continuous Batching)
                true,    // 量化支持 (GPTQ/AWQ/FP8)
                true,    // LoRA 热加载
                true,    // OpenAI 兼容 API
                "高",    // 吞吐量
                "中",    // 运维复杂度
                "推荐: 生产环境首选，吞吐最高"
            ),

            "SGLang", new FrameworkProfile(
                "SGLang", "Apache 2.0",
                true,    // RadixAttention (优于 PagedAttention 的前缀缓存)
                true,
                true,
                false,   // LoRA 支持较弱
                true,
                "最高",  // 结构化输出和共享前缀场景下吞吐最高
                "高",
                "推荐: 结构化输出、多轮对话场景"
            ),

            "TensorRT-LLM", new FrameworkProfile(
                "TensorRT-LLM", "Apache 2.0",
                false,   // 自研 KV Cache 管理
                true,    // In-flight Batching
                true,    // FP8/INT4/INT8
                true,
                true,
                "最高",  // NVIDIA GPU 上延迟最低
                "很高",  // 需要构建 TensorRT 引擎，流程复杂
                "推荐: 对延迟有极致要求的场景(NVIDIA GPU)"
            ),

            "llama.cpp", new FrameworkProfile(
                "llama.cpp", "MIT",
                false,
                true,    // 动态批处理
                true,    // GGUF 量化 (Q4/Q5/Q8)
                false,
                true,
                "低",    // CPU 友好但 GPU 效率不如 vLLM
                "低",
                "推荐: CPU 推理、边缘部署、个人开发"
            ),

            "Ollama", new FrameworkProfile(
                "Ollama", "MIT",
                false,
                false,   // 无连续批处理
                true,    // GGUF
                false,
                true,
                "低",
                "极低",  // 一行命令启动
                "推荐: 本地开发、原型验证"
            )
        );
    }
}

/**
 * 推理框架选型决策树
 */
public class InferenceFrameworkDecisionTree {

    public String decide(InferenceRequirements req) {

        // CPU 推理 → llama.cpp / Ollama
        if (req.hardware() == Hardware.CPU) {
            return "llama.cpp (via Ollama for simplicity)";
        }

        // Mac (Apple Silicon) → MLX / llama.cpp
        if (req.hardware() == Hardware.APPLE_SILICON) {
            return "MLX (最高性能) 或 llama.cpp (最方便)";
        }

        // GPU (NVIDIA)
        if (req.useCase() == UseCase.PRODUCTION) {
            if (req.priority() == Priority.THROUGHPUT) {
                return "vLLM (通用) 或 SGLang (共享前缀/结构化输出)";
            }
            if (req.priority() == Priority.LATENCY) {
                return "TensorRT-LLM (NVIDIA GPU)";
            }
        }

        if (req.useCase() == UseCase.DEVELOPMENT) {
            return "Ollama (简单) → vLLM (接近生产的环境)";
        }

        return "vLLM (最通用的选择)";
    }
}
```

---

## 3.3 向量数据库选型

```java
/**
 * RAG 的核心组件：向量数据库
 */
public class VectorDBComparison {

    public static record VectorDBProfile(
        String name,
        String bestFor,           // 最适合场景
        int maxVectors,           // 规模上限
        boolean distributed,      // 是否支持分布式
        boolean hybridSearch,     // 混合检索(向量+关键词)
        String filtering,         // 标量过滤能力
        String consistency,       // 一致性模型
        int complexity            // 运维复杂度 1-5
    ) {}

    public static List<VectorDBProfile> compare() {
        return List.of(
            // ===== 专用向量数据库 =====
            new VectorDBProfile("Milvus", "大规模生产 (>10M 向量)",
                Integer.MAX_VALUE, true, true, "强大", "最终/强一致", 4),

            new VectorDBProfile("Qdrant", "中小规模生产 (1M-10M 向量)",
                1_000_000_000, true, true, "强大", "最终一致", 2),

            new VectorDBProfile("Weaviate", "多模态 + GraphQL 接口",
                1_000_000_000, true, true, "强大", "最终一致", 3),

            new VectorDBProfile("Pinecone", "Serverless/托管，不想运维",
                10_000_000_000, true, true, "中等", "最终一致", 1),

            // ===== 数据库内置向量能力 =====
            new VectorDBProfile("pgvector (PostgreSQL)", "已有 PG 架构，向量 < 10M",
                10_000_000, false, true, "强大(SQL)", "ACID", 1),

            new VectorDBProfile("Elasticsearch", "已有 ES 架构，需要全文+向量",
                1_000_000_000, true, true, "强大(ES查询)", "最终一致", 3),

            new VectorDBProfile("Redis Stack", "低延迟 (< 10ms)，实时场景",
                1_000_000_000, true, false, "有限", "最终一致", 1),

            // ===== 轻量级 =====
            new VectorDBProfile("Chroma", "原型开发 / PoC",
                1_000_000, false, false, "基础", "最终一致", 1),

            new VectorDBProfile("FAISS", "嵌入现有 Python 应用，无需独立服务",
                Integer.MAX_VALUE, false, false, "无", "无(无服务)", 1)
        );
    }
}
```

---

## 3.4 模型能力评估体系

### 3.4.1 评估维度

```java
/**
 * 企业级 LLM 评估体系
 * 
 * 不能盲目相信公开 Benchmark (可能有数据污染)
 * 必须建立自己的评估集
 */
public class EnterpriseEvalFramework {

    /**
     * 五层评估金字塔
     */
    public enum EvalLayers {
        // L1: 功能正确性 → 模型能不能正确完成任务
        FUNCTIONAL_CORRECTNESS,

        // L2: 输出质量 → 回答好不好 (准确性、完整性、相关性)
        OUTPUT_QUALITY,

        // L3: 安全与合规 → 不输出有害内容，不泄露数据
        SAFETY_COMPLIANCE,

        // L4: 性能 → 延迟、吞吐
        PERFORMANCE,

        // L5: 成本效益 → Token 效率、单位成本产出
        COST_EFFECTIVENESS
    }

    /**
     * L1: 功能正确性评估
     */
    public FunctionalEvalResult evaluateFunctionality(LLM model) {
        // ===== 收集业务场景的测试用例 =====
        List<TestCase> testCases = loadGoldenDataset();

        int pass = 0, fail = 0;

        for (TestCase tc : testCases) {
            String output = model.generate(tc.prompt());

            // 方式 1: 精确匹配 (分类、提取类)
            if (tc.expectExact()) {
                if (output.equals(tc.expectedOutput())) pass++;
                else fail++;
            }

            // 方式 2: LLM-as-Judge (生成类)
            else {
                String judgePrompt = """
                    你是一个严格的评估裁判。
                    
                    参考答案: %s
                    模型输出: %s
                    
                    请判定模型输出是否正确 (YES/NO)。
                    只有完全正确才给 YES。
                    """.formatted(tc.expectedOutput(), output);

                String verdict = evaluatorLLM.generate(judgePrompt);
                if (verdict.contains("YES")) pass++;
                else fail++;
            }
        }

        return new FunctionalEvalResult(pass, fail);
    }

    /**
     * L2: 输出质量评估 (多维度)
     */
    public QualityEvalResult evaluateQuality(LLM model) {
        List<QualityDimension> dimensions = List.of(
            QualityDimension.ACCURACY,       // 准确性
            QualityDimension.COMPLETENESS,   // 完整性
            QualityDimension.RELEVANCE,      // 相关性
            QualityDimension.COHERENCE,      // 连贯性
            QualityDimension.CONCISENESS     // 简洁性
        );

        Map<QualityDimension, Double> scores = new HashMap<>();

        for (var dim : dimensions) {
            double score = evaluateDimension(model, dim);
            scores.put(dim, score);
        }

        return new QualityEvalResult(scores);
    }

    /**
     * L5: 成本效率评估
     */
    public CostEfficiencyEval evaluateEfficiency(LLM model) {
        // 统计完成相同任务消耗的 Token 数
        double totalTokens = 0;
        double totalCorrectOutputs = 0;

        for (TestCase tc : loadEfficiencyDataset()) {
            String output = model.generate(tc.prompt());
            int tokensUsed = tokenCounter.count(tc.prompt() + output);
            boolean correct = isCorrect(output, tc.expectedOutput());

            totalTokens += tokensUsed;
            if (correct) totalCorrectOutputs++;
        }

        double costPerCorrect = (totalTokens / 1_000_000) * model.pricePer1M()
                                  / totalCorrectOutputs;

        return new CostEfficiencyEval(costPerCorrect);
    }
}
```

### 3.4.2 LLM-as-Judge 评分模板

```java
/**
 * 用 LLM 当裁判评测另一个 LLM
 * 
 * 这是当前业界主流做法 (OpenAI、Anthropic 都用)
 */
public class LLMJudge {

    /**
     * 评分: 1-5 分制
     */
    public static String buildJudgePrompt(
            String userQuery, String modelOutput, String rubric) {

        return """
            你是一个客观、严格的 AI 输出评分裁判。
            
            ## 评分标准 (1-5 分):
            %s
            
            ## 用户问题:
            %s
            
            ## 模型回答:
            %s
            
            ## 请输出 JSON 评分 (不要包含其他文字):
            {
              "score": <1-5>,
              "reasoning": "<评分理由，不超过100字>",
              "issues": ["<问题1>", "<问题2>"]  // 如有问题，否则空数组
            }
            """.formatted(rubric, userQuery, modelOutput);
    }

    /**
     * 多裁判一致性评估 (降低单个 Judge 的偏差)
     */
    public double multiJudgeScore(String query, String output) {
        // 用 3 个不同模型的 Judge 投票
        List<LLM> judges = List.of(
            LLM.GPT_4,      // 裁判 1: GPT-4
            LLM.CLAUDE,     // 裁判 2: Claude
            LLM.GEMINI      // 裁判 3: Gemini
        );

        List<Integer> scores = judges.parallelStream()
            .map(judge -> judge.evaluate(query, output))
            .toList();

        // 取中位数(避免极端值)
        return median(scores);
    }
}
```

---

## 3.5 模型评估清单

```yaml
企业 LLM 评估必做事项:

自动评估 (每次 Prompt 变更时):
  - [ ] Golden Dataset 测试集 (> 100 条真实业务场景)
  - [ ] 精确匹配任务: 分类、提取、结构化输出
  - [ ] LLM-as-Judge: 生成类任务的质量评分
  - [ ] 安全性测试: 越狱攻击、注入攻击、有害内容
  - [ ] 性能测试: P50/P95/P99 延迟、吞吐量

人工评估 (重要变更时):
  - [ ] 盲测对比 (A/B 测试): 让人工评审员比较不同模型/Prompt 的输出
  - [ ] 边界用例: 测试极端输入 (空输入、超长输入、多语言混合)
  - [ ] 行业专业知识: 领域专家验证特定场景

持续监控 (生产环境):
  - [ ] 用户满意度: 👍/👎 比例
  - [ ] 幻觉检测: 引用来源是否真的存在
  - [ ] 内容漂移: 模型行为是否随时间改变
  - [ ] Token 效率: 回答是否越来越啰嗦
```

---

> **下一章**：[数据治理与流水线](04-data-governance-pipeline.md)
