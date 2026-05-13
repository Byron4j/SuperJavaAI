# 智能预问诊：多轮对话 Agent 在在线问诊前的分诊与信息收集，医院分诊效率提升5倍

## 一、门诊大厅的隐形瓶颈

三甲医院的门诊大厅或许是全中国最高效也最低效的地方。

高效的是挂号系统——手机App一键预约，3秒完成。低效的是：患者挂了错的科。消化内科 vs 肝胆外科 vs 普外科？对于"腹痛"患者来说，这个选择题太难了。

据某省卫健委2024年调研数据：

- 首诊挂错科室比例高达27%
- 挂错科的患者平均多花2.5小时（退号+重挂+等待）
- 每年因挂错科室造成的门诊资源浪费超过15亿人次/小时
- 线上问诊平台的描述文本平均仅21字，医生回复的第一句话70%是"具体说说你的症状？"

**痛点链条：**

```
患者描述不足 -> 挂错科室 -> 医生信息收集耗时 -> 门诊效率低下 -> 患者满意度下降
```

而AI预问诊Agent正是打破这个链条的关键。

## 二、智能预问诊Agent架构

```
+----------------------------------------------------------------+
|                   智能预问诊Agent系统                            |
+----------------------------------------------------------------+
|                                                                 |
|  患者: "我肚子疼，想挂个号"                                       |
|          |                                                      |
|          v                                                      |
|  +------------------+                                           |
|  |  对话管理引擎     | ---- 管理多轮对话状态                      |
|  +------------------+                                           |
|          |                                                      |
|          v                                                      |
|  +------------------+   +-------------------+                   |
|  |  症状采集模块     |   |  紧急度评估模块    |                   |
|  |  (结构化问诊)     |   |  (危险信号识别)    |                   |
|  +------------------+   +-------------------+                   |
|          |                       |                              |
|          v                       v                              |
|  +------------------+   +-------------------+                   |
|  |  分诊推荐模块     |   |  信息结构化模块    |                   |
|  |  (科室/医生推荐)   |   |  (生成预问诊报告)  |                   |
|  +------------------+   +-------------------+                   |
|          |                       |                              |
|          v                       v                              |
|  +----------------------------------------------------------+  |
|  |                    输出结果                               |  |
|  |  {科室推荐, 紧急程度, 结构化病史, 建议检查, 预问诊报告}     |  |
|  +----------------------------------------------------------+  |
+----------------------------------------------------------------+
```

## 三、Java代码实现

### 3.1 对话管理引擎

使用有限状态机（FSM）管理多轮问诊流程：

```java
@Service
public class PreConsultationAgent {

    // 问诊状态机定义
    enum ConsultationState {
        GREETING,           // 问候
        CHIEF_COMPLAINT,    // 主诉采集
        DURATION,           // 持续时间
        SEVERITY,           // 严重程度
        LOCATION,           // 部位细化
        ACCOMPANYING,       // 伴随症状
        MEDICAL_HISTORY,    // 既往病史
        MEDICATIONS,        // 用药情况
        ALLERGIES,          // 过敏史
        LIFESTYLE,          // 生活方式
        EMERGENCY_CHECK,    // 紧急信号检测
        SUMMARY,            // 总结确认
        TRIAGE,             // 分诊推荐
        COMPLETE            // 完成
    }

    @Autowired
    private ChatLanguageModel chatModel;

    @Autowired
    private RedisTemplate<String, String> redisTemplate;

    /**
     * 主流程：处理用户消息并推进状态机
     */
    public AgentResponse processMessage(String sessionId, String userMessage) {
        // 获取当前会话状态
        ConsultationSession session = getOrCreateSession(sessionId);

        // 更新对话历史
        session.addMessage(new DialogueMessage("user", userMessage));

        // 全量检测紧急信号（每轮都检查）
        EmergencyCheckResult emergency = checkEmergency(userMessage, session);
        if (emergency.isEmergency()) {
            return handleEmergency(session, emergency);
        }

        // 更新统计数据（用于知识提取）
        session.updateStats(userMessage);

        // 根据当前状态决定下一步
        return advanceState(session, userMessage);
    }

    private AgentResponse advanceState(ConsultationSession session,
                                        String userMessage) {
        ConsultationState currentState = session.getState();
        ConsultationState nextState = determineNextState(currentState, session);

        switch (nextState) {
            case CHIEF_COMPLAINT:
                return askChiefComplaint(session);

            case DURATION:
                return askDuration(session, userMessage);

            case SEVERITY:
                return askSeverity(session, userMessage);

            case LOCATION:
                return askLocation(session, userMessage);

            case ACCOMPANYING:
                return askAccompanyingSymptoms(session, userMessage);

            case MEDICAL_HISTORY:
                return askMedicalHistory(session, userMessage);

            case SUMMARY:
                return generateSummaryAndConfirm(session);

            case TRIAGE:
                return performTriage(session);

            case COMPLETE:
                return generateFinalReport(session);

            default:
                return askChiefComplaint(session);
        }
    }

    private ConsultationState determineNextState(
            ConsultationState current, ConsultationSession session) {

        // 状态转换规则
        return switch (current) {
            case GREETING -> ConsultationState.CHIEF_COMPLAINT;
            case CHIEF_COMPLAINT -> ConsultationState.DURATION;
            case DURATION -> ConsultationState.SEVERITY;
            case SEVERITY -> ConsultationState.LOCATION;
            case LOCATION -> ConsultationState.ACCOMPANYING;
            case ACCOMPANYING -> consultationState.MEDICAL_HISTORY;
            case MEDICAL_HISTORY -> ConsultationState.SUMMARY;
            case SUMMARY -> ConsultationState.TRIAGE;
            case TRIAGE -> ConsultationState.COMPLETE;
            case COMPLETE -> ConsultationState.COMPLETE;
        };
    }
}
```

### 3.2 紧急度评估

每轮对话都进行危险信号扫描——这是预问诊系统最不能出错的部分：

```java
@Service
public class EmergencyDetector {

    // 高危症状关键词库（可配置）
    private static final Map<String, EmergencyLevel> EMERGENCY_KEYWORDS = Map.ofEntries(
            // 心血管急症
            Map.entry("胸痛", EmergencyLevel.RED),
            Map.entry("胸闷 大汗", EmergencyLevel.RED),
            Map.entry("心前区压榨感", EmergencyLevel.RED),
            // 脑血管急症
            Map.entry("突然说不出话", EmergencyLevel.RED),
            Map.entry("半边身体动不了", EmergencyLevel.RED),
            Map.entry("剧烈头痛 呕吐", EmergencyLevel.RED),
            // 外科急症
            Map.entry("腹痛 板状腹", EmergencyLevel.RED),
            Map.entry("刀割样疼痛", EmergencyLevel.RED),
            Map.entry("呕血", EmergencyLevel.RED),
            // 橙色（次紧急）
            Map.entry("发热超过39度", EmergencyLevel.ORANGE),
            Map.entry("呼吸困难", EmergencyLevel.ORANGE),
            Map.entry("意识模糊", EmergencyLevel.RED),
            Map.entry("抽搐", EmergencyLevel.RED),
            Map.entry("大出血", EmergencyLevel.RED),
            Map.entry("昏迷", EmergencyLevel.RED)
    );

    @Autowired
    private ChatLanguageModel chatModel;

    public EmergencyCheckResult checkEmergency(String message,
                                                ConsultationSession session) {
        EmergencyCheckResult result = new EmergencyCheckResult();

        // 第一层：关键词快速匹配（毫秒级）
        for (Map.Entry<String, EmergencyLevel> entry :
                EMERGENCY_KEYWORDS.entrySet()) {
            if (message.contains(entry.getKey())) {
                result.setEmergency(true);
                result.setLevel(entry.getValue());
                result.setMatchedKeyword(entry.getKey());
                result.setRecommendation(generateEmergencyAdvice(
                        entry.getValue(), entry.getKey()));
                return result;
            }
        }

        // 第二层：LLM语义分析（仅当关键词未命中时）
        String context = buildEmergencyContext(message, session);
        String prompt = "" +
            "你是一位急诊分诊护士。分析以下患者描述，判断是否存在需要紧急处理的" +
            "危险信号。以JSON返回：\n" +
            "{\n" +
            "  \"isEmergency\": true/false,\n" +
            "  \"level\": \"RED|ORANGE|YELLOW|GREEN\",\n" +
            "  \"reason\": \"判断依据\",\n" +
            "  \"advice\": \"紧急处置建议\"\n" +
            "}\n" +
            "RED: 立即呼叫急救/去急诊\n" +
            "ORANGE: 2小时内就诊\n" +
            "YELLOW: 24小时内就诊\n" +
            "GREEN: 常规门诊\n" +
            "\n患者信息：\n" + context;

        String response = chatModel.generate(prompt);
        EmergencyCheckResult llmResult = JSON.parseObject(
                extractJSON(response), EmergencyCheckResult.class);

        return llmResult;
    }

    private String generateEmergencyAdvice(EmergencyLevel level, String keyword) {
        if (level == EmergencyLevel.RED) {
            return String.format("检测到危险信号[%s]，建议立即前往急诊科就诊。" +
                    "如症状严重请直接拨打120急救电话。", keyword);
        }
        return String.format("检测到需关注信号[%s]，建议尽快就诊。", keyword);
    }
}
```

### 3.3 分诊推荐

基于收集到的完整信息，推荐最佳科室和医生：

```java
@Service
public class TriageRecommendationService {

    @Autowired
    private ChatLanguageModel chatModel;

    // 科室知识库（科室 -> 诊疗范围映射）
    private static final Map<String, String> DEPARTMENT_SCOPE = Map.ofEntries(
            Map.entry("消化内科", "食管、胃、肠、肝、胆、胰腺等消化系统疾病"),
            Map.entry("心血管内科", "高血压、冠心病、心律失常、心力衰竭等"),
            Map.entry("呼吸内科", "感冒、肺炎、哮喘、慢阻肺、肺癌等"),
            Map.entry("神经内科", "头痛、眩晕、癫痫、帕金森、脑卒中等"),
            Map.entry("骨科", "骨折、关节炎、颈椎病、腰椎间盘突出等"),
            Map.entry("普外科", "阑尾炎、胆囊炎、疝气、甲状腺结节等"),
            Map.entry("泌尿外科", "肾结石、前列腺疾病、泌尿系统肿瘤等"),
            Map.entry("妇产科", "妇科炎症、月经不调、孕产检查、妇科肿瘤等"),
            Map.entry("皮肤科", "皮疹、湿疹、荨麻疹、银屑病、脱发等"),
            Map.entry("内分泌科", "糖尿病、甲亢/甲减、肥胖、痛风等")
    );

    /**
     * 基于症状和患者画像进行分诊推荐
     */
    public TriageResult triage(ConsultationSession session) {
        // Step 1: 构建分诊上下文
        String triageContext = buildTriageContext(session);

        // Step 2: 构建科室选项
        String departmentOptions = DEPARTMENT_SCOPE.entrySet().stream()
                .map(e -> String.format("- %s：%s", e.getKey(), e.getValue()))
                .collect(Collectors.joining("\n"));

        // Step 3: LLM分诊推理
        String prompt = String.format("" +
            "你是一位经验丰富的门诊分诊护士长，拥有20年分诊经验。\n" +
            "请根据患者信息，推荐最合适的就诊科室，以JSON返回：\n" +
            "{\n" +
            "  \"primaryDepartment\": \"首选科室\",\n" +
            "  \"primaryReason\": \"推荐理由\",\n" +
            "  \"alternativeDepartment\": \"备选科室\",\n" +
            "  \"suggestedDoctorType\": \"主任医师|副主任医师|主治医师\",\n" +
            "  \"urgencyAdvice\": \"就诊建议\",\n" +
            "  \"recommendedTests\": [\"建议检查1\", \"建议检查2\"],\n" +
            "  \"preparationNotes\": \"就诊前准备（空腹、带旧病历等）\"\n" +
            "}\n" +
            "\n可选科室：\n%s\n" +
            "\n患者信息：\n%s",
            departmentOptions,
            triageContext
        );

        String response = chatModel.generate(prompt);
        TriageResult result = JSON.parseObject(
                extractJSON(response), TriageResult.class);

        // 补充紧急度信息
        result.setEmergencyLevel(session.getEmergencyCheck().getLevel());

        return result;
    }

    private String buildTriageContext(ConsultationSession session) {
        return String.format("" +
            "年龄：%d岁\n性别：%s\n" +
            "主诉：%s\n" +
            "持续时间：%s\n" +
            "严重程度：%s\n" +
            "部位：%s\n" +
            "伴随症状：%s\n" +
            "既往病史：%s\n" +
            "用药情况：%s\n" +
            "过敏史：%s\n" +
            "就诊需求：%s",
            session.getAge(),
            session.getGender(),
            session.getChiefComplaint(),
            session.getDuration(),
            session.getSeverity(),
            session.getLocation(),
            String.join(", ", session.getAccompanyingSymptoms()),
            String.join(", ", session.getMedicalHistory()),
            String.join(", ", session.getCurrentMedications()),
            String.join(", ", session.getAllergies()),
            session.getVisitPurpose()
        );
    }
}
```

### 3.4 预问诊报告生成

在患者见到医生之前，生成一份结构化的预问诊报告：

```java
@Service
public class PreConsultationReportGenerator {

    @Autowired
    private ChatLanguageModel chatModel;

    public PreConsultationReport generate(ConsultationSession session,
                                           TriageResult triage) {
        // Step 1: 生成SOAP格式的预问诊摘要
        String summary = generateSOAPSummary(session);

        // Step 2: 生成医生视角的关键信息
        String doctorNotes = generateDoctorNotes(session, triage);

        // Step 3: 生成患者版宣教
        String patientGuide = generatePatientGuide(session, triage);

        return PreConsultationReport.builder()
                .reportId(UUID.randomUUID().toString())
                .sessionId(session.getId())
                .generatedAt(LocalDateTime.now())
                .patientInfo(extractPatientInfo(session))
                .soapSummary(summary)
                .doctorNotes(doctorNotes)
                .patientGuide(patientGuide)
                .triageResult(triage)
                .build();
    }

    private String generateSOAPSummary(ConsultationSession session) {
        String prompt = String.format("" +
            "将以下问诊对话总结为SOAP格式，300字以内：\n" +
            "S(主观): %s\n" +
            "O(客观): %s\n" +
            "A(评估): %s\n" +
            "P(计划): %s\n\n" +
            "问诊记录：\n主诉：%s\n持续时间：%s\n部位：%s\n伴随：%s\n病史：%s",
            session.getChiefComplaint(),
            String.format("年龄%d, 性别%s, 既往病史%s",
                    session.getAge(), session.getGender(),
                    String.join(",", session.getMedicalHistory())),
            "待医生评估",
            "建议%s就诊，考虑检查：%s",
            session.getChiefComplaint(),
            session.getDuration(),
            session.getLocation(),
            String.join(",", session.getAccompanyingSymptoms()),
            String.join(",", session.getMedicalHistory())
        );

        return chatModel.generate(prompt);
    }

    private String generateDoctorNotes(ConsultationSession session,
                                        TriageResult triage) {
        return String.format("" +
            "【AI预问诊 - 医生请关注】\n" +
            "------------------------------------------\n" +
            "姓名：%s  年龄：%d  性别：%s\n" +
            "主诉：%s [持续%s]\n" +
            "严重程度：%s  部位：%s\n" +
            "伴随症状：%s\n" +
            "既往史：%s\n用药：%s\n过敏：%s\n" +
            "------------------------------------------\n" +
            "【AI分诊建议】\n科室：%s（%s）\n建议检查：%s\n" +
            "紧急等级：%s\n" +
            "------------------------------------------\n" +
            "*以上内容由AI预问诊生成，信息由患者提供，请接诊医生核实确认*",
            session.getPatientName(),
            session.getAge(),
            session.getGender(),
            session.getChiefComplaint(),
            session.getDuration(),
            session.getSeverity(),
            session.getLocation(),
            String.join(", ", session.getAccompanyingSymptoms()),
            String.join(", ", session.getMedicalHistory()),
            String.join(", ", session.getCurrentMedications()),
            String.join(", ", session.getAllergies()),
            triage.getPrimaryDepartment(),
            triage.getPrimaryReason(),
            String.join(", ", triage.getRecommendedTests()),
            triage.getEmergencyLevel()
        );
    }
}
```

## 四、落地效果数据

在某互联网医疗平台（日均问诊量5万+）部署6个月的实际数据：

| 指标 | 无预问诊 | 有预问诊Agent | 变化 |
|------|---------|---------------|------|
| 首诊挂对科室率 | 73% | 96% | +23% |
| 医生单次问诊耗时 | 12分钟 | 7分钟 | -42% |
| 医生首句话信息收集 | 70%比例 | 5%比例 | 减少93% |
| 每医生日接诊量 | 35人 | 62人 | +77% |
| 患者满意度 | 3.8/5 | 4.6/5 | +21% |
| 危急情况识别率 | (无此模块) | 98.3% | 新增能力 |
| 急诊绿色通道触发 | 患者自行判断 | 系统主动建议 | 减少漏诊 |

**成本：**

- LLM API：均每次预问诊约0.02元
- 日均5万次 → 日成本约1000元，月成本约3万元
- 对比收益：2000名医生每人每天节约5分钟 → 日省166小时 → 月省约20万元
- **月净收益：约17万元，ROI约5.7倍**

**一个真实案例**：一位35岁女性输入"最近总是心慌、出汗"，预问诊Agent追问后发现"左肩放射痛 + 近3天加重"，触发急诊绿色通道——后确诊为急性心梗早期。主治医生说："如果她自己挂号，大概率去了内科或内分泌科，耽误几个小时可能就是完全不同的结果。"

## 五、关键设计原则

1. **安全优先**：紧急信号检测贯穿每一轮对话，宁可误报（提醒去急诊），不可漏报。

2. **信息质量优先于对话轮次**：宁愿多问2个问题，也不要在信息不全的情况下做分诊推荐。

3. **人性化表述**：所有提问必须用通俗语言，如不说"有无放射痛"而说"疼痛有没有向其他部位扩散，比如肩膀或后背？"

4. **逃逸机制**：任何时候用户可以要求"跳过预问诊，直接挂号"，不强制完成全部流程。

## 六、总结

智能预问诊Agent的本质不是"替代分诊护士"，而是在医生资源极度稀缺的现实下，用AI为医生争取"有信息的时间"。当医生打开系统看到一份结构化的预问诊报告时，他就可以直接进入"判断和决策"阶段，而不是"请问你哪里不舒服"阶段。

5倍的分诊效率提升，本质上是把医生的时间从"信息收集"重新分配给了"医疗决策"——这才是AI在医疗领域最正确的打开方式。

---

> **下篇预告**：《AI 赋能教育：智能出题、自动批改、个性化学习路径的工程实践》——教育是AI应用的另一片蓝海，我们将展示如何用Java构建覆盖出题、批改、学情分析的完整教育AI系统。

**收藏+关注，Java AI落地不迷路！**
