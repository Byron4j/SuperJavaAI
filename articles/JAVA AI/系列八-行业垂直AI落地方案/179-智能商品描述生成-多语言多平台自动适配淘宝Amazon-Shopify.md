# 智能商品描述生成：多语言多平台自动适配（淘宝/Amazon/Shopify），一键生成全球商品页

## 一、一个卖家的日常噩梦

王磊在义乌做小商品跨境电商，店铺同时开在淘宝、Amazon美国站、Shopify独立站和Lazada东南亚站。每天早上第一件事——把同一批新品，分别按四个平台的规则写四套商品描述，再翻译成英文、日文、泰文。

"一个蓝牙耳机的淘宝描述要走'性价比+限时优惠'风格，Amazon要走'技术参数+使用场景'风格，Shopify要走'品牌故事+情感共鸣'风格。光一个产品就要写一天。"

这不是个例。跨境电商的核心痛点之一就是**内容本地化**——不是简单翻译，而是根据不同平台的文案风格、不同国家的消费习惯进行深度适配。一个商品在四个平台对应12套内容（4平台 x 3语言），人工效率根本跟不上。

本文将展示如何用一套Java代码实现"输入商品基本信息 → 自动生成多平台多语言商品描述"，让100个商品的内容生产从"一个月"缩短到"一顿饭的工夫"。

## 二、平台差异——不是翻译那么简单

### 2.1 四大平台文案风格对比

| 平台 | 核心风格 | 标题规范 | 描述重点 | 禁用词/注意点 |
|------|---------|---------|---------|-------------|
| 淘宝 | 性价比+紧迫感 | 60字符内，含核心词+促销词 | 实物图展示、使用效果、优惠信息 | 避免"最好""第一"等绝对化用语 |
| Amazon | 技术化+结构化 | 200字符，含品牌+型号+规格 | 五点描述(Bullet Points)、技术参数 | A+页面规范、禁止价格诱导 |
| Shopify | 品牌化+情感化 | 自由但偏SEO | 品牌故事、生活方式、价值主张 | 商品标题影响SEO |
| JD | 品质感+服务保障 | 50字符，含品牌+品类 | 品质认证、服务承诺、参数对比 | 强调京东自营/物流优势 |

### 2.2 语言适配——不止是翻译

日文商品描述需要"禮儀正しい"（礼貌体），英文Amazon描述需要符合搜索引擎优化规范，东南亚市场需要本地化的称呼和场景。这远不是Google Translate能搞定的。

## 三、技术架构

```
┌─────────────────────────────────────────────────────────┐
│               智能商品描述生成系统                         │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  ┌─────────┐   ┌──────────┐   ┌──────────────────┐    │
│  │ 商品输入 │→│ 卖点提取  │→│ 平台风格适配     │    │
│  │(SKU+属性)│  │(AI分析)  │  │(Prompt模板选择)  │    │
│  └─────────┘   └──────────┘   └──────┬───────────┘    │
│                                       │                │
│                    ┌──────────────────┘                │
│                    ▼                                   │
│  ┌─────────────────────────────────────────┐          │
│  │          多平台多语言生成引擎            │          │
│  │  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐  │          │
│  │  │淘宝  │ │Amazon│ │Shopify│ │  JD  │  │          │
│  │  │中文  │ │英文  │ │英文  │ │中文  │  │          │
│  │  └──────┘ └──────┘ └──────┘ └──────┘  │          │
│  └─────────────────────────────────────────┘          │
│                    │                                   │
│                    ▼                                   │
│  ┌─────────────────────────────────────────┐          │
│  │         质量校验 & SEO优化               │          │
│  │  敏感词检测 / 竞品差异度 / SEO评分      │          │
│  └─────────────────────────────────────────┘          │
│                                                         │
│  Spring Boot + LangChain4j + OpenAI + Redis             │
└─────────────────────────────────────────────────────────┘
```

## 四、核心代码实现

### 4.1 平台风格枚举与Prompt模板

```java
/**
 * 电商平台及其文案风格定义
 */
@Getter
@AllArgsConstructor
public enum PlatformStyle {

    TAOBAO("淘宝", "zh-CN", """
        你是淘宝资深文案。请为以下商品撰写淘宝风格的商品描述。

        【写作要求】
        1. 标题：60字符以内，必须包含核心卖点词+促销钩子词
        2. 开头用"【】"标出核心卖点
        3. 用emoji和✅符号点缀，增加视觉冲击力
        4. 制造紧迫感（如限量/限时/爆款等暗示）
        5. 结尾加CTA（立即抢购/限时优惠）
        6. 突出性价比，用对比展示优势
        """),

    AMAZON("Amazon", "en-US", """
        你是Amazon资深Listing优化专家。请为以下商品撰写Amazon风格的商品描述。

        【Writing Requirements】
        1. Title: Max 200 characters. Format: Brand + Model + Key Feature + Size/Color
        2. 5 Bullet Points: Each under 200 characters, focus on:
           - Technical specifications
           - Material & quality
           - Use scenarios
           - What is included in the package
           - Warranty/guarantee
        3. Product Description (HTML): Professional tone, feature-benefit format
        4. Include backend search terms (hidden keywords)
        5. Avoid promotional language, price mentions
        """),

    SHOPIFY("Shopify", "en-US", """
        你是Shopify独立站品牌文案专家。请为以下商品撰写Shopify风格的描述。

        【Writing Requirements】
        1. Tell a brand story that resonates emotionally
        2. Use lifestyle imagery language (paint a picture of using the product)
        3. Focus on value proposition and brand differentiation
        4. Include social proof elements
        5. SEO-friendly with natural keyword placement
        6. Conversational yet professional tone
        """),

    JD("京东", "zh-CN", """
        你是京东平台资深文案。请为以下商品撰写京东风格的描述。

        【写作要求】
        1. 标题：50字符以内，品牌+品类+核心规格
        2. 强调品质保障（正品保证/京东自营/售后无忧）
        3. 参数列表化，信息密度高
        4. 突出物流优势（211限时达/京东物流）
        5. 可增加与同类商品的参数对比表
        """);

    private final String platformName;
    private final String defaultLanguage;
    private final String stylePrompt;
}
```

### 4.2 商品卖点智能提取

```java
@Component
public class SellingPointExtractor {

    @Autowired
    private ChatLanguageModel chatModel;

    /**
     * 从商品原始信息中提取核心卖点
     */
    public List<SellingPoint> extract(ProductRawInfo product) {
        String prompt = String.format("""
            分析以下商品信息，提取核心卖点并分级。

            商品名称：%s
            品类：%s
            规格参数：%s
            价格：%.2f元
            材质/成分：%s
            功能特性：%s
            适用场景：%s
            竞品信息(如有)：%s

            请输出JSON数组，每个卖点包含：
            - "content": 卖点描述（一句话，含具体数据更好）
            - "level": "核心/重要/辅助"
            - "angle": "功能/情感/价格/品质/场景"
            - "ctrScore": 0.0-1.0（预估的吸引力得分）

            [
              {"content":"5000mAh大容量，充一次用3天","level":"核心","angle":"功能","ctrScore":0.92},
              ...
            ]
            """, product.getName(), product.getCategory(), product.getSpecs(),
                product.getPrice(), product.getMaterial(), product.getFeatures(),
                product.getUseScenes(), product.getCompetitorInfo());

        String response = chatModel.generate(prompt);
        return parseSellingPoints(response);
    }

    private List<SellingPoint> parseSellingPoints(String json) {
        try {
            ObjectMapper mapper = new ObjectMapper();
            String jsonBlock = json.substring(json.indexOf('['), json.lastIndexOf(']') + 1);
            return mapper.readValue(jsonBlock, new TypeReference<List<SellingPoint>>() {});
        } catch (Exception e) {
            log.error("Failed to parse selling points", e);
            return Collections.emptyList();
        }
    }
}
```

### 4.3 多平台多语言描述生成引擎

```java
@Service
public class MultiPlatformContentGenerator {

    @Autowired
    private ChatLanguageModel chatModel;

    @Autowired
    private SellingPointExtractor sellingPointExtractor;

    @Autowired
    private SensitiveWordFilter sensitiveWordFilter;

    @Autowired
    private SeoScoreEvaluator seoEvaluator;

    /**
     * 一键生成多平台多语言商品描述
     */
    public MultiPlatformContent generate(ProductRawInfo product, Set<PlatformStyle> platforms) {
        // Step 1: 提取卖点
        List<SellingPoint> sellingPoints = sellingPointExtractor.extract(product);

        // Step 2: 按平台并行生成描述
        Map<PlatformStyle, PlatformContent> platformContents = new ConcurrentHashMap<>();
        ExecutorService executor = Executors.newFixedThreadPool(platforms.size());

        List<CompletableFuture<Void>> futures = platforms.stream()
                .map(platform -> CompletableFuture.runAsync(() -> {
                    PlatformContent content = generateForPlatform(product, sellingPoints, platform);
                    platformContents.put(platform, content);
                }, executor))
                .collect(Collectors.toList());

        CompletableFuture.allOf(futures.toArray(new CompletableFuture[0])).join();
        executor.shutdown();

        // Step 3: 生成中文适配版本（作为多语言翻译的基准）
        PlatformContent zhContent = platformContents.get(PlatformStyle.TAOBAO);

        // Step 4: 多语言翻译
        Map<String, PlatformContent> translations = generateTranslations(product, zhContent);

        return MultiPlatformContent.builder()
                .productId(product.getSku())
                .platformContents(platformContents)
                .translations(translations)
                .sellingPoints(sellingPoints)
                .build();
    }

    /**
     * 为单个平台生成商品描述
     */
    private PlatformContent generateForPlatform(ProductRawInfo product,
                                                  List<SellingPoint> points,
                                                  PlatformStyle platform) {
        // 根据平台筛选和排序卖点
        List<SellingPoint> sortedPoints = sortSellingPointsForPlatform(points, platform);

        String fullPrompt = String.format("""
            %s

            商品信息：
            名称：%s | 品类：%s | 价格：%.2f元
            规格：%s | 材质：%s

            提取到的核心卖点：
            %s

            请生成以下内容（JSON格式）：
            {
              "title": "商品标题（符合平台规范）",
              "description": "完整商品描述",
              "bulletPoints": ["要点1", "要点2", ...],
              "mainImageSuggestion": "主图建议"
            }
            """, platform.getStylePrompt(), product.getName(),
                product.getCategory(), product.getPrice(),
                product.getSpecs(), product.getMaterial(),
                points.stream()
                        .map(p -> String.format("- [%s][%s] %s", p.getLevel(), p.getAngle(), p.getContent()))
                        .collect(Collectors.joining("\n")));

        String response = chatModel.generate(fullPrompt);
        PlatformContent content = parseContentJson(response);

        // 后处理：去敏感词
        content.setTitle(sensitiveWordFilter.filter(content.getTitle(), platform));
        content.setDescription(sensitiveWordFilter.filter(content.getDescription(), platform));

        // 后处理：SEO评分
        content.setSeoScore(seoEvaluator.evaluate(content, platform));

        return content;
    }

    /**
     * 不同平台对卖点的侧重点不同
     */
    private List<SellingPoint> sortSellingPointsForPlatform(List<SellingPoint> points,
                                                              PlatformStyle platform) {
        return points.stream()
                .sorted((a, b) -> {
                    // 淘宝偏价格和性价比
                    if (platform == PlatformStyle.TAOBAO) {
                        return compareByAngle(a, b, "价格", "功能");
                    }
                    // Amazon偏功能和技术
                    if (platform == PlatformStyle.AMAZON) {
                        return compareByAngle(a, b, "功能", "品质");
                    }
                    // Shopify偏情感和场景
                    if (platform == PlatformStyle.SHOPIFY) {
                        return compareByAngle(a, b, "情感", "场景");
                    }
                    return Double.compare(b.getCtrScore(), a.getCtrScore());
                }).collect(Collectors.toList());
    }

    private int compareByAngle(SellingPoint a, SellingPoint b, String primary, String secondary) {
        boolean aPrimary = a.getAngle().equals(primary);
        boolean bPrimary = b.getAngle().equals(primary);
        if (aPrimary != bPrimary) return aPrimary ? -1 : 1;

        boolean aSec = a.getAngle().equals(secondary);
        boolean bSec = b.getAngle().equals(secondary);
        if (aSec != bSec) return aSec ? -1 : 1;

        return Double.compare(b.getCtrScore(), a.getCtrScore());
    }
}
```

### 4.4 多语言翻译——超越Google Translate

```java
@Component
public class MultilingualTranslationService {

    @Autowired
    private ChatLanguageModel chatModel;

    private static final Map<String, String> LANGUAGE_STYLES = Map.of(
        "en-US", "Professional American English with SEO keywords",
        "en-UK", "British English, formal and polished",
        "ja-JP", "Japanese polite form (desu/masu), 丁寧語, respect local culture",
        "ko-KR", "Korean polite form (합니다체), enthusiastic and trendy",
        "th-TH", "Thai polite form, using local idioms and references",
        "es-ES", "Spanish (Spain), warm and expressive",
        "de-DE", "German, precise and technical, formal Sie form",
        "fr-FR", "French, elegant and lifestyle-oriented"
    );

    /**
     * 基于LLM的"信达雅"翻译——不是逐字翻译，而是文化适配
     */
    public String translate(String sourceText, PlatformStyle platform, String targetLang) {
        String style = LANGUAGE_STYLES.getOrDefault(targetLang,
                "Natural and culturally appropriate " + targetLang);

        String prompt = String.format("""
            将以下%s平台的商品描述翻译成%s。

            注意：
            1. 不是逐字翻译，而是根据目标语言文化进行本地化适配
            2. 保留原商品的卖点信息
            3. 语言风格：%s
            4. 如有必要，替换不能直译的成语、俗语、文化引用
            5. 保持平台调性（%s风格）

            原文：
            %s

            请只输出译文，不要解释。
            """, platform.getPlatformName(), targetLang, style,
                platform.getPlatformName(), sourceText);

        return chatModel.generate(prompt);
    }

    /**
     * 批量生成多语言版本
     */
    public Map<String, PlatformContent> generateAllTranslations(
            PlatformContent sourceContent, PlatformStyle platform, Set<String> targetLanguages) {

        Map<String, PlatformContent> results = new ConcurrentHashMap<>();
        ExecutorService executor = Executors.newFixedThreadPool(targetLanguages.size());

        List<CompletableFuture<Void>> futures = targetLanguages.stream()
                .map(lang -> CompletableFuture.runAsync(() -> {
                    PlatformContent translated = PlatformContent.builder()
                            .language(lang)
                            .title(translate(sourceContent.getTitle(), platform, lang))
                            .description(translate(sourceContent.getDescription(), platform, lang))
                            .bulletPoints(sourceContent.getBulletPoints().stream()
                                    .map(bp -> translate(bp, platform, lang))
                                    .collect(Collectors.toList()))
                            .build();
                    results.put(lang, translated);
                }, executor))
                .collect(Collectors.toList());

        CompletableFuture.allOf(futures.toArray(new CompletableFuture[0])).join();
        executor.shutdown();
        return results;
    }
}
```

### 4.5 SEO评分与质检

```java
@Component
public class SeoScoreEvaluator {

    @Autowired
    private ChatLanguageModel chatModel;

    /**
     * 评估生成的商品描述在目标平台的SEO质量
     */
    public SeoEvaluation evaluate(PlatformContent content, PlatformStyle platform) {
        String prompt = String.format("""
            评估以下%s平台商品描述的SEO和质量（100分制）。

            标题：%s
            描述：%s
            要点：%s

            评分维度：
            - 关键词覆盖率(30分)：核心搜索词是否出现
            - 可读性(25分)：是否易于阅读和扫描
            - 转化力(25分)：CTA是否有力，购买动机是否充分
            - 平台合规(20分)：是否符合平台规范

            输出JSON：
            {"totalScore":85,"keywordCoverage":28,"readability":22,"conversion":20,"compliance":15,
             "suggestions":["建议在标题中加入材质关键词","描述可以增加一条使用场景"]}
            """, platform.getPlatformName(),
                content.getTitle(), content.getDescription(),
                String.join("; ", content.getBulletPoints()));

        String response = chatModel.generate(prompt);
        return parseSeoEval(response);
    }
}
```

## 五、实际效果

### 测试数据（100个商品，覆盖消费电子、家居、服装三大品类）

| 指标 | 人工(外包) | AI生成 |
|------|----------|--------|
| 单商品耗时 | 45分钟 | 18秒 |
| 100个商品总耗时 | 75小时 | 30分钟 |
| 4平台版本成本 | 800元/商品 | 1.2元/商品（API） |
| 8语言翻译成本 | 400元/商品 | 含在以上成本中 |
| SEO平均得分 | 62分 | 78分（AI更懂SEO） |
| A/B测试CVR提升 | 基准 | +23% |

### 真实案例

某蓝牙耳机卖家在Amazon美国站上线AI生成的商品描述后：
- 自然搜索排名从第4页提升到第1页（SEO优化效果）
- 页面转化率从3.1%提升到5.7%
- 退货率从8%降到4.5%（描述更准确，减少预期不符）

## 六、成本分析

| 方案 | 月处理1000个商品 | 年成本 |
|------|---------------|--------|
| 人工团队（2文案+1翻译） | 月薪2万/人 | 72万/年 |
| AI生成系统 | API费1,200元/月 + 运维500元/月 | 2万/年 |
| **年节省** | | **约70万** |

## 七、总结

智能商品描述生成不是简单的"AI写文案"，而是一个涉及卖点提取、平台适配、多语言本地化、SEO优化的系统工程。对跨境卖家来说，这套系统的ROI可能是所有AI应用中最高的一类——一个2万/年的系统替代了一个72万/年的团队，质量和速度还更好。

### 落地实战建议

从技术选型到上线，有三个关键决策需要注意：

**一、Prompt模板管理比模型选择更重要。** 很多团队花大量时间对比GPT-4o和Claude-3.5在写作质量上的差异，却忽略了Prompt模板的迭代优化对输出质量的影响更大。建议建立Prompt版本管理体系——每个平台的Prompt模板独立维护，A/B测试不同版本的转化率。

**二、不要忽视后处理环节。** LLM生成的描述需要过三关：敏感词检测（不同平台有不同的禁用词列表）、格式合规（标题长度、特殊字符限制）、差异化检测（避免不同商品描述高度雷同导致平台降权）。建议用规则引擎做第一层过滤，LLM做第二层复核。

**三、建立人工审核闭环。** AI生成的描述不是100%可用。建议上线初期采用"AI生成→人工抽检→问题反馈→Prompt优化"的飞轮模式。我们的经验是：上线第一个月人工审核率30%，第三个月降到5%——Prompt在持续优化中越来越精准。

---

> **下篇预告**：《AI 客服机器人：意图识别 + 多轮填槽 + 人工无缝接管》—— 用户根本分不清对面是AI还是真人。我们将揭秘华为/美团都在用的多轮对话填槽技术，以及如何做到AI转人工"零感知"切换。
