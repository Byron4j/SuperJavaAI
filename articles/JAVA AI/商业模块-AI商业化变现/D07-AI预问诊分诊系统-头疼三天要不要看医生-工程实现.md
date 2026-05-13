# AI预问诊分诊系统：头疼三天要不要看医生？工程实现

> 一个深夜头疼到睡不着的患者，打开手机AI助手描述症状，30秒后获得分诊建议和就医指引。这套系统帮某三甲医院将急诊分诊准确率提升了23%。

---

## 一、行业痛点：分诊台的崩溃

### 急诊室的"排队悖论"

凌晨2点，某三甲医院急诊科。分诊护士小陈面对这样的场景：

- 排队人数：47人
- 重症患者（3级及以上）：6人，需要立即处理
- 轻症患者（4-5级）：41人，其中至少15人其实不需要来急诊

小陈需要在**每位患者30秒内**完成分诊判断——问症状、评估严重程度、判断科室。但她不是全科医生，面对非典型症状时只能凭经验判断。

**分诊错误的数据代价**：

```
某三甲医院2023年急诊数据（年急诊量约12万人次）：
- 分诊准确率：76.3%（目标≥90%）
- 因为分诊错误导致的：
  × 二次转科：3200+ 人次
  × 延误治疗：180+ 例（可能造成严重后果）
  × 医疗纠纷：12 起
- 因为"其实不需要来急诊"造成的：
  × 资源浪费：约24000人次/年
  × 急诊医生无效工作量：约8000小时/年
```

### 更深层的矛盾

除了急诊分诊，还有两个更普遍的痛点：

1. **就医决策困难**："头疼三天了，要不要去医院？挂什么科？"——每个人都会遇到的问题，但大部分人不会判断
2. **基层医疗能力不足**：社区卫生服务中心的全科医生，面对复杂症状缺乏分诊信心

---

## 二、产品设计：AI预问诊分诊系统

我们的产品 **MediTriage** 分三个场景：

```
场景A：患者端（C端）    →  症状自查+就医指引
场景B：急诊分诊台（B端） →  辅助护士快速准确分诊  
场景C：基层医疗（B端）   →  辅助全科医生判断转诊
```

### 核心流程

```
患者描述症状（自然语言）
    ↓
【症状标准化】LLM将"口语描述"转为"医学术语"
    ↓
【紧急度评估】5级分诊标准判定
    ↓
【科室推荐】匹配最合适的就诊科室
    ↓
【问诊追问】如果信息不足，智能追问关键问题
    ↓
【就医建议】输出：紧急程度+推荐科室+注意事项+预计等待时间
```

---

## 三、技术架构与实现

### 3.1 整体架构

```java
// 技术栈
// 后端: Spring Boot + Spring AI + MySQL + Redis
// 医学知识库: Neo4j (症状-疾病-科室关系图)
// 分诊决策: Drools规则引擎 + LLM语义引擎
// 前端: Vue3 (分诊台Web) + 微信小程序 (患者端)
```

```java
@Configuration
public class MediTriageConfig {
    
    /**
     * 分诊决策引擎组合：规则引擎 + AI
     * 
     * 轻量场景（常见症状）→ 规则引擎毫秒级响应
     * 复杂场景（非典型症状）→ LLM深度分析
     * 争议场景 → 规则引擎+LLM双重判断 + 人工复核
     */
    @Bean
    public TriageDecisionEngine decisionEngine(
            RuleEngineService ruleEngine,
            LLMTriageService llmService,
            MedicalKnowledgeGraph kgService) {
        return new TriageDecisionEngine(ruleEngine, llmService, kgService);
    }
}
```

### 3.2 医学知识图谱构建

```java
@Service
public class MedicalKnowledgeGraphService {
    
    private final Neo4jTemplate neo4jTemplate;
    
    /**
     * 医学知识图谱节点类型
     */
    @Node
    @Data
    public static class Symptom {
        @Id private String id;
        private String name;              // "头痛"
        private List<String> aliases;     // ["头疼", "偏头痛", "头胀"]
        private String location;          // 身体部位
        private String description;
        private SymptomCategory category; // LOCAL/GENERAL/PSYCHOLOGICAL
    }
    
    @Node
    @Data
    public static class Disease {
        @Id private String id;
        private String name;
        private String icdCode;           // ICD-10编码
        private String department;        // 就诊科室
        private TriageLevel defaultLevel; // 默认分诊级别
        private Boolean isEmergency;      // 是否属于急症
        private List<String> redFlags;    // 危险信号
    }
    
    @Node
    @Data
    public static class Department {
        @Id private String id;
        private String name;              // "神经内科"
        private String hospital;          // 所属医院
        private Integer currentQueueLength; // 当前排队人数
        private Integer estimatedWaitMinutes; // 预计等待时间
    }
    
    /**
     * 查询：给定一组症状，找到可能的相关疾病
     */
    public List<DiseaseWithProbability> findDiseasesBySymptoms(
            List<String> symptomNames) {
        
        // Cypher查询：找到与越多症状关联的疾病，得分越高
        String cypher = """
            MATCH (s:Symptom)-[:INDICATES]->(d:Disease)
            WHERE s.name IN $symptomNames
            WITH d, COUNT(s) AS matchCount, COLLECT(s.name) AS matchedSymptoms
            MATCH (d)-[:INDICATED_BY]->(allS:Symptom)
            WITH d, matchCount, matchedSymptoms, COUNT(allS) AS totalSymptomCount
            RETURN d, 
                   matchCount,
                   matchedSymptoms,
                   totalSymptomCount,
                   (matchCount * 1.0 / totalSymptomCount) AS coverage
            ORDER BY coverage DESC, matchCount DESC
            LIMIT 20
            """;
        
        return neo4jTemplate.query(cypher, Map.of("symptomNames", symptomNames))
            .fetchAs(DiseaseWithProbability.class)
            .all();
    }
    
    /**
     * 查询危险信号
     * 某些症状组合可能暗示危重症
     */
    public List<RedFlag> checkRedFlags(List<String> symptoms, 
                                         UserDemographics demographics) {
        
        String cypher = """
            MATCH (s:Symptom)-[:TRIGGERS_RED_FLAG]->(rf:RedFlag)
            WHERE s.name IN $symptoms
            RETURN DISTINCT rf
            """;
        
        List<RedFlag> redFlags = neo4jTemplate.query(cypher, 
                Map.of("symptoms", symptoms))
            .fetchAs(RedFlag.class)
            .all();
        
        // 额外规则：年龄>65 + 胸痛 = 高危
        if (demographics.getAge() > 65 && symptoms.contains("胸痛")) {
            redFlags.add(new RedFlag("ACS_RISK", "急性冠脉综合征风险",
                "建议立即急诊，完善心电图和肌钙蛋白", TriageLevel.LEVEL_1));
        }
        
        return redFlags;
    }
}
```

### 3.3 分诊决策引擎

```java
@Service
public class TriageDecisionEngine {
    
    private final RuleEngineService ruleEngine;
    private final LLMTriageService llmService;
    private final MedicalKnowledgeGraph kgService;
    
    /**
     * 分诊主流程
     */
    public TriageResult triage(TriageRequest request) {
        
        // Step 1: 症状标准化（口语→医学术语）
        StandardizedSymptoms standardized = standardizeSymptoms(
            request.getSymptomDescription()
        );
        
        // Step 2: 危险信号检查（必须最先执行，生命优先）
        List<RedFlag> redFlags = kgService.checkRedFlags(
            standardized.getSymptomNames(),
            request.getDemographics()
        );
        
        if (!redFlags.isEmpty()) {
            // 触发危险信号 → 最高优先级处理
            TriageLevel highestLevel = redFlags.stream()
                .map(RedFlag::getLevel)
                .min(Comparator.naturalOrder())
                .orElse(TriageLevel.LEVEL_1);
            
            return TriageResult.builder()
                .triageLevel(highestLevel)
                .isEmergency(true)
                .redFlags(redFlags)
                .recommendation("⚠️ 检测到危险信号，建议立即就医")
                .estimatedUrgency("立即")
                .build();
        }
        
        // Step 3: 规则引擎初筛（常见症状快速分诊）
        Optional<TriageResult> ruleResult = ruleEngine.triage(
            standardized, request.getDemographics()
        );
        
        // Step 4: 知识图谱搜索可能疾病
        List<DiseaseWithProbability> possibleDiseases = kgService
            .findDiseasesBySymptoms(standardized.getSymptomNames());
        
        // Step 5: 如果规则引擎结果置信度够高，直接返回
        if (ruleResult.isPresent() && ruleResult.get().getConfidence() >= 0.85) {
            return enrichWithDiseases(ruleResult.get(), possibleDiseases);
        }
        
        // Step 6: 否则调用LLM做深度分析
        TriageResult llmResult = llmService.deepTriage(
            request.getSymptomDescription(),
            standardized,
            possibleDiseases,
            request.getDemographics()
        );
        
        // Step 7: 低置信度场景标记为需要人工复核
        if (llmResult.getConfidence() < 0.7) {
            llmResult.setNeedsHumanReview(true);
        }
        
        return llmResult;
    }
    
    /**
     * 症状标准化：通俗描述→标准医学术语
     * 
     * 输入："最近三天一直头疼，太阳穴这里一跳一跳的，有时候还想吐"
     * 输出：["头痛", "搏动性头痛", "颞部疼痛", "恶心"]
     */
    private StandardizedSymptoms standardizeSymptoms(String rawDescription) {
        
        String result = chatClient.prompt()
            .system("""
                你是一名医学信息学专家。将患者的口语化症状描述
                转化为标准的医学术语列表。
                
                转换原则：
                1. 使用标准医学术语
                2. 包含症状的部位、性质、持续时间、诱因、伴随症状
                3. 不添加患者未提到的症状
                4. 输出格式：JSON数组 ["术语1", "术语2", ...]
                
                示例：
                输入："嗓子疼了两天，吞口水都疼，还有点发烧"
                输出：["咽痛", "吞咽痛", "发热", "病程2天"]
                """)
            .user(rawDescription)
            .call()
            .content();
        
        List<String> standardTerms = parseStringList(result);
        
        // 在标准术语和用户口语之间建立映射（用于后续展示）
        Map<String, String> termMapping = new HashMap<>();
        for (String term : standardTerms) {
            termMapping.put(term, rawDescription);
        }
        
        return new StandardizedSymptoms(standardTerms, termMapping);
    }
}
```

### 3.4 分诊规则引擎（Drools）

```java
@Service
public class RuleEngineService {
    
    private KieSession kieSession;
    
    @PostConstruct
    public void init() {
        KieServices ks = KieServices.Factory.get();
        KieContainer kContainer = ks.getKieClasspathContainer();
        this.kieSession = kContainer.newKieSession("triage-rules");
    }
    
    /**
     * Drools规则引擎快速分诊
     */
    public Optional<TriageResult> triage(StandardizedSymptoms symptoms,
                                          UserDemographics demographics) {
        
        TriageFacts facts = TriageFacts.builder()
            .symptoms(symptoms.getSymptomNames())
            .age(demographics.getAge())
            .gender(demographics.getGender())
            .hasChronicDisease(demographics.getChronicDiseases() != null 
                               && !demographics.getChronicDiseases().isEmpty())
            .build();
        
        kieSession.insert(facts);
        kieSession.fireAllRules();
        
        if (facts.getTriageLevel() != null) {
            return Optional.of(TriageResult.builder()
                .triageLevel(facts.getTriageLevel())
                .confidence(facts.getConfidence())
                .department(facts.getDepartment())
                .ruleMatched(facts.getRuleMatched())
                .build());
        }
        
        return Optional.empty();
    }
}

// Drools规则文件: triage-rules.drl
/*
package com.meditriage.rules;

import com.meditriage.model.*;

rule "胸痛合并大汗淋漓"
    when
        $f: TriageFacts(
            symptoms contains "胸痛", 
            symptoms contains "大汗"
        )
    then
        $f.setTriageLevel(TriageLevel.LEVEL_1);
        $f.setConfidence(0.95);
        $f.setDepartment("心血管内科/急诊科");
        $f.setRuleMatched("胸痛+大汗=急性冠脉综合征可能");
end

rule "头痛合并喷射性呕吐"
    when
        $f: TriageFacts(
            symptoms contains "头痛",
            symptoms contains "喷射性呕吐"
        )
    then
        $f.setTriageLevel(TriageLevel.LEVEL_1);
        $f.setConfidence(0.92);
        $f.setDepartment("神经外科/急诊科");
        $f.setRuleMatched("头痛+喷射性呕吐=颅内压增高可能");
end

rule "发热超过3天儿童"
    when
        $f: TriageFacts(
            age < 12,
            symptoms contains "发热",
            symptoms contains "病程超过3天"
        )
    then
        $f.setTriageLevel(TriageLevel.LEVEL_2);
        $f.setConfidence(0.85);
        $f.setDepartment("儿科");
        $f.setRuleMatched("儿童发热超3天需警惕");
end

rule "轻微外伤轻症"
    when
        $f: TriageFacts(
            symptoms contains "擦伤" || symptoms contains "浅表切割伤",
            symptoms not contains "活动性出血"
        )
    then
        $f.setTriageLevel(TriageLevel.LEVEL_5);
        $f.setConfidence(0.90);
        $f.setDepartment("急诊科/普外科");
        $f.setRuleMatched("轻微外伤，非紧急");
end
*/
```

### 3.5 LLM深度分诊

```java
@Service
public class LLMTriageService {
    
    private final ChatClient chatClient;
    
    /**
     * 对规则引擎无法确定的情况，使用LLM深度分析
     */
    public TriageResult deepTriage(String rawDescription,
                                    StandardizedSymptoms symptoms,
                                    List<DiseaseWithProbability> diseases,
                                    UserDemographics demographics) {
        
        // 构建详细的医学上下文
        String medicalContext = buildMedicalContext(diseases);
        
        String result = chatClient.prompt()
            .system("""
                你是一名急诊医学专家，拥有20年临床经验。
                你的任务是根据患者症状进行预分诊。
                
                分诊级别（5级制，卫生部标准）：
                - 1级（濒危）：需立即抢救，如心跳呼吸骤停、大出血
                - 2级（危重）：需15分钟内处理，如急性心梗、脑卒中
                - 3级（紧急）：需30分钟内处理，如高热惊厥、严重外伤
                - 4级（亚急）：需60分钟内处理，如普通发热、轻度外伤
                - 5级（非急）：可等待，如慢性病复诊、轻微不适
                
                注意事项：
                1. 宁可高估不可低估（敏感性优先于特异性）
                2. 特别关注"红旗征"（危险信号）
                3. 考虑患者年龄、基础疾病等危险因素
                4. 如果不确定，建议升级分诊级别
                
                请以JSON格式返回：
                {
                  "triageLevel": 3,
                  "confidence": 0.85,
                  "primaryDepartment": "神经内科",
                  "alternativeDepartment": "急诊科",
                  "reasoning": "分析依据",
                  "redFlagsFound": ["危险信号"],
                  "suggestedExaminations": ["建议检查"],
                  "waitTimeTolerance": "可等待30分钟",
                  "selfCareAdvice": "就医前可采取的措施",
                  "whenToEmergency": "什么情况下需立即就医",
                  "followUpQuestions": ["需要追问的问题"]
                }
                """)
            .user("""
                患者信息：
                - 年龄：%d岁
                - 性别：%s
                - 基础疾病：%s
                
                症状描述（患者原话）：
                "%s"
                
                标准化症状：%s
                
                可能性疾病（仅供参考）：
                %s
                
                请进行分诊评估。
                """.formatted(
                    demographics.getAge(),
                    demographics.getGender(),
                    demographics.getChronicDiseases() != null ? 
                        String.join("、", demographics.getChronicDiseases()) : "无",
                    rawDescription,
                    String.join("、", symptoms.getSymptomNames()),
                    medicalContext
                ))
            .call()
            .content();
        
        return parseTriageResult(result);
    }
    
    private String buildMedicalContext(List<DiseaseWithProbability> diseases) {
        if (diseases.isEmpty()) return "暂无匹配疾病信息";
        
        return diseases.stream()
            .limit(10)
            .map(d -> String.format(
                "- %s（ICD-10: %s, 匹配度: %.0f%%, 就诊科室: %s）",
                d.getDisease().getName(),
                d.getDisease().getIcdCode(),
                d.getProbability() * 100,
                d.getDisease().getDepartment()
            ))
            .collect(Collectors.joining("\n"));
    }
}
```

### 3.6 智能追问引擎

当信息不足以做判断时，系统需要智能追问：

```java
@Service
public class SmartFollowUpService {
    
    private final ChatClient chatClient;
    
    /**
     * 评估是否需要追问以及追问什么
     */
    public FollowUpDecision decideFollowUp(TriageResult currentResult,
                                            StandardizedSymptoms symptoms) {
        
        // 如果置信度足够高，不需要追问
        if (currentResult.getConfidence() >= 0.85) {
            return FollowUpDecision.noFollowUp();
        }
        
        // 否则生成追问问题
        String followUpJson = chatClient.prompt()
            .system("""
                你是经验丰富的分诊护士。当前信息不足以做出准确分诊判断。
                
                追问原则：
                1. 每次最多问3个问题
                2. 问题简短、口语化，患者容易理解
                3. 优先问最能改变分诊决策的信息
                4. 考虑患者的描述能力，不要问太专业的问题
                
                请以JSON格式返回：
                {
                  "isNeeded": true,
                  "questions": [
                    {"question": "...", "purpose": "这个问题想了解什么", "impactOnTriage": "答案如何影响分诊"},
                    ...
                  ]
                }
                """)
            .user("""
                当前已知症状：%s
                当前分诊结果：%s级别（置信度%.0f%%）
                缺少的关键信息类型：%s
                
                请生成追问问题。
                """.formatted(
                    String.join("、", symptoms.getSymptomNames()),
                    currentResult.getTriageLevel(),
                    currentResult.getConfidence() * 100,
                    currentResult.getMissingInfo()
                ))
            .call()
            .content();
        
        return parseFollowUpDecision(followUpJson);
    }
}
```

---

## 四、安全与合规设计

```java
@Component
public class MedicalSafetyGuard {
    
    /**
     * 免责声明和风险提示
     * 医疗AI产品必须明确的边界
     */
    public String getDisclaimer(TriageResult result) {
        StringBuilder sb = new StringBuilder();
        sb.append("⚠️ 重要提示：\n");
        sb.append("本系统提供的是辅助参考建议，不能替代专业医生的诊断。\n");
        
        if (result.getTriageLevel().getLevel() <= 2) {
            sb.append("🚨 您的症状可能提示紧急情况，建议立即就医！\n");
        }
        
        sb.append("\n以下情况请立即拨打120：\n");
        sb.append("- 意识丧失或严重意识障碍\n");
        sb.append("- 呼吸困难或窒息感\n");
        sb.append("- 大出血无法止住\n");
        sb.append("- 剧烈胸痛持续超过15分钟\n");
        sb.append("- 严重外伤或骨折\n");
        
        return sb.toString();
    }
    
    /**
     * 药物建议的安全控制
     * 绝不允许AI直接推荐药品
     */
    @PostConstruct
    public void configureSafetyFilters() {
        // 输出过滤：检测并拦截药品推荐
        safetyFilters.add(output -> {
            if (containsDrugRecommendation(output)) {
                return "【系统提示】具体用药请咨询医生或药师，"
                     + "本系统不提供用药建议。";
            }
            return output;
        });
    }
}
```

---

## 五、商业模式

### 定价

| 场景 | 模式 | 价格 |
|------|------|------|
| 患者端C端 | 免费（挂号平台导流分成） | 免费 |
| 医院急诊分诊台 | SaaS订阅 | ¥2999/月/院区 |
| 基层医疗机构 | 政府购买服务 | ¥5000-20000/年/点 |

### 盈利模式

- **B端SaaS订阅**（核心收入）
- **挂号平台分润**（C端导流）
- **政府公卫项目**（基层医疗AI辅助）

---

> **下一篇预告**：《药企AI合规审查——药品说明书自动过30道法律审核的技术方案》，一款药品说明书上线要过N道合规审查，我们用AI帮药企把审核从3周缩短到3小时。
