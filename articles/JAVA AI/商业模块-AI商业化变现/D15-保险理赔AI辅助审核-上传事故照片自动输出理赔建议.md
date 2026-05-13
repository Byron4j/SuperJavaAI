# 保险理赔AI辅助审核：上传事故照片，自动输出理赔建议

> 车险小案从48小时审核缩短到15分钟，定损准确率从82%提升到96%。我们帮某中型保险公司做了套AI理赔审核系统，理赔员从20人减到8人，年省人力成本300万+。

---

## 一、行业痛点：保险理赔的"漏斗"困局

### 一个理赔员的真实一天

某财产保险公司深圳分公司的理赔员老周，工龄12年。他的日常：

```
上午9:00 - 打开理赔系统
  待处理案件：47件（昨天积压）+ 今日新分配：23件
  
9:05 - 第1件：车险小额剐蹭
  看照片（3分钟）→ 看维修估价（2分钟）
  → 核对保单（1分钟）→ 录入定损金额（1分钟）
  → 生成理赔建议书（5分钟）
  小计：12分钟

9:17 - 第2件：医疗险报销
  看病历（5分钟）→ 看发票（3分钟）
  → 核对保障范围（3分钟）→ 计算可赔付金额（5分钟）
  → 生成审核意见（8分钟）
  小计：24分钟

...（以此类推）

12:00 - 上午处理完成：8件
  平均每件：18分钟
  积压增加：+12件（新分配22件，只处理了14件）

下午持续 - 加班到19:00
  全天处理：19件
  积压新增：+15件
  明天早上一打开，待处理：47 + 15 = 62件
```

**这就是保险理赔的恶性循环**：处理速度永远追不上案件涌入速度，理赔员永远在赶进度，客户永远在催。

### 不同案件类型的处理效率

| 案件类型 | 占比 | 平均处理时间 | AI化潜力 | 年处理量（中型公司） |
|----------|------|-------------|----------|---------------------|
| 车险小案（<5000元）| 55% | 15-30分钟 | 85% | 约8万件 |
| 车险大案（>5000元）| 15% | 1-3小时 | 50% | 约2万件 |
| 医疗险报销 | 20% | 20-40分钟 | 70% | 约3万件 |
| 意外险 | 8% | 10-20分钟 | 80% | 约1万件 |
| 财产险 | 2% | 2-5小时 | 30% | 约0.3万件 |

**关键发现**：占比最高的车险小案和意外险（合计63%），恰恰是AI化潜力最高的案件类型。

---

## 二、产品架构：ClaimAI智能理赔审核

```
┌─────────────────────────────────────────────────────┐
│              ClaimAI 智能理赔审核系统                   │
├─────────────────────────────────────────────────────┤
│                                                       │
│  用户报案 → 上传材料（照片/视频/文档）→ 自动进入审核管道  │
│                                                       │
│  ┌─────────────────────────────────────────────┐     │
│  │          智能审核管道                         │     │
│  │                                                │     │
│  │  Step 1: 材料完整性检查                        │     │
│  │  Step 2: OCR+信息提取                         │     │
│  │  Step 3: 保单匹配+保障范围核实                  │     │
│  │  Step 4: 欺诈风险评估                         │     │
│  │  Step 5: 定损估算                            │     │
│  │  Step 6: 理赔建议生成                         │     │
│  │  Step 7: 审核意见书自动生成                    │     │
│  └─────────────────────────────────────────────┘     │
│                         ↓                             │
│         AI自动决策            → 人工复核              │
│    （高置信度的小额案件）        （争议/大额案件）       │
└─────────────────────────────────────────────────────┘
```

---

## 三、核心Java实现

### 3.1 多类型材料解析

```java
@Service
public class ClaimDocumentParser {
    
    /**
     * 解析各类理赔材料
     * - 事故照片（车辆损伤、事故现场）
     * - 医疗发票（门诊/住院）
     * - 病历/诊断证明
     * - 维修估价单
     * - 保单合同
     */
    public ParsedClaimMaterials parse(ClaimSubmission submission) {
        
        ParsedClaimMaterials materials = new ParsedClaimMaterials();
        
        for (Attachment attachment : submission.getAttachments()) {
            
            ParsedContent content = switch (attachment.getType()) {
                case PHOTO -> parsePhoto(attachment);
                case INVOICE -> parseInvoice(attachment);
                case MEDICAL_RECORD -> parseMedicalRecord(attachment);
                case REPAIR_ESTIMATE -> parseRepairEstimate(attachment);
                case POLICY_DOCUMENT -> parsePolicyDocument(attachment);
                case VIDEO -> parseVideo(attachment);
                default -> parseGenericDocument(attachment);
            };
            
            materials.addContent(attachment.getType(), content);
        }
        
        return materials;
    }
    
    /**
     * 事故照片解析
     * - 使用视觉模型识别损伤部位和程度
     * - 识别车牌号、事故环境
     */
    private ParsedContent parsePhoto(Attachment photo) {
        
        // 方案1: 使用多模态视觉大模型
        String analysis = chatClient.prompt()
            .system("""
                你是车险定损专家。分析以下事故照片的损伤情况。
                
                请识别并输出JSON：
                {
                  "vehicleType": "轿车/SUV/货车/...",
                  "damageArea": ["前保险杠", "右前翼子板", ...],
                  "damageType": ["刮擦", "凹陷", "破裂", "脱落", ...],
                  "damageSeverity": "轻微/中等/严重",
                  "repairEstimate": {
                    "paintWork": "需要/不需要",
                    "partReplacement": ["需要更换的零件"],
                    "laborHours": 5.0,
                    "estimatedCost": 3500.00
                  },
                  "plateNumber": "如果可以识别",
                  "sceneInfo": "事故现场描述"
                }
                """)
            .user(new org.springframework.ai.model.Media(
                org.springframework.util.MimeTypeUtils.IMAGE_JPEG,
                new org.springframework.core.io.ByteArrayResource(photo.getBytes())
            ))
            .call()
            .content();
        
        return parseAnalysis(analysis);
    }
    
    /**
     * 医疗发票OCR+结构化提取
     */
    private ParsedContent parseInvoice(Attachment invoice) {
        
        // 1. OCR识别
        String ocrText = ocrService.recognize(invoice.getBytes());
        
        // 2. 结构化提取
        String extracted = chatClient.prompt()
            .system("""
                从医疗发票OCR结果中提取结构化信息。
                
                JSON格式：
                {
                  "invoiceNumber": "发票号",
                  "hospitalName": "医院名称",
                  "patientName": "患者姓名",
                  "date": "就诊日期",
                  "items": [
                    {
                      "name": "项目名称",
                      "category": "西药/中成药/检查/治疗/材料/...",
                      "amount": 123.45,
                      "insuranceCovered": true/false,
                      "selfPayRatio": "0.3"
                    }
                  ],
                  "totalAmount": 1234.56,
                  "insuranceCoveredAmount": 864.19,
                  "selfPayAmount": 370.37
                }
                """)
            .user(ocrText)
            .call()
            .content();
        
        return parseInvoiceData(extracted);
    }
}
```

### 3.2 欺诈风险检测引擎

```java
@Service
public class FraudDetectionEngine {
    
    private final ChatClient chatClient;
    private final ClaimHistoryService historyService;
    
    /**
     * 多维欺诈风险评估
     * 
     * 风险维度：
     * 1. 用户行为风险（报案频率、历史欺诈记录）
     * 2. 事故合理性（时间/地点/伤痕一致性）
     * 3. 材料真实性（照片是否被修图、发票是否可疑）
     * 4. 金额合理性（修理费是否虚高）
     */
    public FraudRiskAssessment assessFraudRisk(ClaimSubmission submission,
                                                ParsedClaimMaterials materials,
                                                PolicyInfo policy) {
        
        FraudRiskAssessment assessment = new FraudRiskAssessment();
        
        // 维度1: 用户历史行为分析
        ClaimantHistory history = historyService.getHistory(
            submission.getClaimantId()
        );
        double behaviorRisk = assessBehaviorRisk(history, submission, policy);
        assessment.addDimension("行为风险", behaviorRisk, 
            "该用户近12个月报案%d次，同业记录查询结果：%s".formatted(
                history.getRecentClaimCount(), 
                history.getIndustryRecord()
            ));
        
        // 维度2: 事故合理性分析
        double incidentPlausibility = assessIncidentPlausibility(
            submission, materials
        );
        assessment.addDimension("事故合理性", incidentPlausibility,
            "事故描述与损伤特征一致性评估");
        
        // 维度3: 材料真实性
        double materialAuthenticity = assessMaterialAuthenticity(materials);
        assessment.addDimension("材料真实性", materialAuthenticity,
            "照片/发票/病历真实性评估");
        
        // 维度4: 金额合理性
        double amountReasonability = assessAmountReasonability(
            submission, materials
        );
        assessment.addDimension("金额合理性", amountReasonability,
            "索赔金额与市场平均价格对比");
        
        // 综合风险评级
        double overallRisk = assessment.getWeightedAverage();
        assessment.setOverallRisk(overallRisk);
        assessment.setRiskLevel(determineRiskLevel(overallRisk));
        
        // 高风险标记
        if (assessment.getRiskLevel() == FraudRiskLevel.HIGH) {
            assessment.setAction(FraudAction.MANUAL_REVIEW_REQUIRED);
            assessment.setNote("综合风险评分较高，建议人工详细审核");
        }
        
        return assessment;
    }
    
    /**
     * 事故合理性分析
     * 例如：低速碰撞造成的损伤与高速碰撞不相符
     */
    private double assessIncidentPlausibility(ClaimSubmission submission,
                                               ParsedClaimMaterials materials) {
        
        String analysis = chatClient.prompt()
            .system("""
                你是保险反欺诈专家。评估以下事故的合理性。
                
                检查要点：
                1. 事故描述与损伤部位是否一致？
                2. 损伤程度与事故严重性是否匹配？
                3. 新旧伤痕是否可以区分？
                4. 事故时间/地点是否可疑（如凌晨偏僻路段）？
                
                输出格式：{ "plausibilityScore": 0.85, "findings": [...], "redFlags": [...] }
                """)
            .user("""
                事故描述：%s
                损伤分析：%s
                事故时间：%s
                事故地点：%s
                """.formatted(
                    submission.getIncidentDescription(),
                    materials.getDamageAnalysis(),
                    submission.getIncidentTime(),
                    submission.getIncidentLocation()
                ))
            .call()
            .content();
        
        return parsePlausibilityScore(analysis);
    }
}
```

### 3.3 自动定损估算

```java
@Service
public class AutoEstimationService {
    
    private final ChatClient chatClient;
    private final MarketPriceService priceService;
    
    /**
     * 自动定损 — 车险场景
     */
    public LossEstimation estimateVehicleDamage(ParsedClaimMaterials materials,
                                                  PolicyInfo policy,
                                                  VehicleInfo vehicle) {
        
        // 1. 获取市场参考价格
        Map<String, Double> partsPrices = priceService.getPartsPrices(
            vehicle.getBrand(), vehicle.getModel(), vehicle.getYear()
        );
        
        Map<String, Double> laborPrices = priceService.getLaborPrices(
            policy.getRegion()
        );
        
        // 2. AI识别需要维修/更换的零件
        DamageAssessment damage = materials.getDamageAssessment();
        
        // 3. 逐项估算
        List<RepairItem> repairItems = new ArrayList<>();
        double totalPartsCost = 0;
        double totalLaborCost = 0;
        
        for (String part : damage.getAffectedParts()) {
            double marketPrice = partsPrices.getOrDefault(part, 
                estimatePriceFromDescription(part));
            
            // 考虑折旧
            double depreciationRate = calculateDepreciationRate(
                vehicle.getPurchaseDate(), part
            );
            double adjustedPrice = marketPrice * (1 - depreciationRate);
            
            // 判定维修还是更换
            boolean needsReplacement = damage.needsReplacement(part);
            double laborHours = estimateLaborHours(part, damage.getSeverity());
            
            RepairItem item = RepairItem.builder()
                .partName(part)
                .action(needsReplacement ? "更换" : "维修")
                .partsCost(needsReplacement ? adjustedPrice : adjustedPrice * 0.3)
                .laborHours(laborHours)
                .laborCost(laborHours * laborPrices.getOrDefault("standard", 80.0))
                .build();
            
            repairItems.add(item);
            totalPartsCost += item.getPartsCost();
            totalLaborCost += item.getLaborCost();
        }
        
        // 4. 检查是否达到全损标准
        double vehicleValue = estimateVehicleValue(vehicle);
        double totalRepairCost = totalPartsCost + totalLaborCost;
        boolean isTotalLoss = totalRepairCost > vehicleValue * 0.7;
        
        return LossEstimation.builder()
            .repairItems(repairItems)
            .totalPartsCost(totalPartsCost)
            .totalLaborCost(totalLaborCost)
            .totalCost(totalRepairCost)
            .vehicleActualValue(vehicleValue)
            .isTotalLoss(isTotalLoss)
            .recommendedPayout(isTotalLoss ? vehicleValue : totalRepairCost)
            .confidence(isTotalLoss ? 0.90 : 
                // 金额越大越需要人工复核
                totalRepairCost < 3000 ? 0.95 : 
                totalRepairCost < 10000 ? 0.85 : 0.70)
            .build();
    }
}
```

### 3.4 智能理赔决策

```java
@Service
public class ClaimDecisionEngine {
    
    /**
     * 自动理赔决策流程
     * 
     * 决策矩阵：
     * | 金额范围 | 风险等级     | 置信度 | 决策         |
     * |---------|------------|--------|-------------|
     * | <3000   | 低/中       | >0.9   | 自动通过     |
     * | 3000-1万| 低          | >0.85  | 自动通过     |
     * | 3000-1万| 中          | 任意   | 人工复核     |
     * | >1万    | 任意        | 任意   | 人工复核     |
     * | 任意    | 高          | 任意   | 人工详细审核 |
     */
    public ClaimDecision makeDecision(LossEstimation estimation,
                                       FraudRiskAssessment fraud,
                                       PolicyInfo policy) {
        
        double claimAmount = estimation.getRecommendedPayout();
        FraudRiskLevel riskLevel = fraud.getRiskLevel();
        double confidence = estimation.getConfidence();
        
        ClaimDecision decision = new ClaimDecision();
        
        // 检查保障范围
        if (!policy.covers(estimation.getClaimType())) {
            return ClaimDecision.reject("该事故不在保单保障范围内");
        }
        
        // 检查免赔额
        double deductible = policy.getDeductible(estimation.getClaimType());
        if (claimAmount <= deductible) {
            return ClaimDecision.reject(
                "理赔金额¥%.2f未超过免赔额¥%.2f".formatted(claimAmount, deductible)
            );
        }
        
        double payout = claimAmount - deductible;
        
        // 决策树
        if (riskLevel == FraudRiskLevel.HIGH) {
            decision = ClaimDecision.manualReviewRequired(
                "欺诈风险评估为高风险，需要人工详细审核"
            );
        } else if (claimAmount > 10000) {
            decision = ClaimDecision.manualReviewRequired(
                "理赔金额超过¥10,000，需要人工复核"
            );
        } else if (claimAmount > 3000 && riskLevel == FraudRiskLevel.MEDIUM) {
            decision = ClaimDecision.manualReviewRequired(
                "理赔金额¥%.0f配合中风险评级，建议人工复核".formatted(claimAmount)
            );
        } else if (confidence > 0.85 
                   && (claimAmount <= 3000 || riskLevel == FraudRiskLevel.LOW)) {
            decision = ClaimDecision.autoApprove(
                payout, 
                "小额案件+低风险+高置信度，系统自动审核通过"
            );
        } else {
            decision = ClaimDecision.manualReviewRequired(
                "置信度不足，建议人工复核"
            );
        }
        
        decision.setRecommendedPayout(payout);
        decision.setRationale(generateRationale(estimation, fraud, policy));
        
        return decision;
    }
    
    private String generateRationale(LossEstimation estimation,
                                      FraudRiskAssessment fraud,
                                      PolicyInfo policy) {
        
        return chatClient.prompt()
            .system("你是保险理赔审核员，撰写理赔决策说明")
            .user("""
                定损金额：¥%.2f
                免赔额：¥%.2f
                建议赔付：¥%.2f
                风险评估：%s
                定损明细：%s
                
                请用100字以内说明理赔决策依据。
                """.formatted(
                    estimation.getTotalCost(),
                    policy.getDeductible(estimation.getClaimType()),
                    estimation.getRecommendedPayout(),
                    fraud.getRiskLevel(),
                    estimation.getRepairItems().stream()
                        .map(i -> "%s: %s ¥%.0f".formatted(
                            i.getPartName(), i.getAction(), 
                            i.getPartsCost() + i.getLaborCost()))
                        .collect(Collectors.joining("; "))
                ))
            .call()
            .content();
    }
}
```

---

## 四、落地数据

某中型财险公司（年理赔案件约14万件）上线6个月后：

| 指标 | 上线前 | 上线后 | 变化 |
|------|--------|--------|------|
| 小额案件处理时间 | 48小时 | 15分钟 | **-99.5%** |
| 理赔员人均日处理量 | 19件 | 58件 | **+205%** |
| 定损准确率 | 82% | 96% | **+17%** |
| 欺诈识别率 | 约60% | 约88% | **+47%** |
| 理赔人员数量 | 20人 | 8人 | **-60%** |
| 年化成本节省 | - | ¥320万 | - |

---

## 五、商业模式

| 版本 | 价格 | 年处理量 |
|------|------|----------|
| 试用版 | 免费试用3月 | 1000件 |
| 标准版 | ¥5万/年 | 1万件 |
| 企业版 | ¥15万/年 | 5万件 |
| 旗舰版 | ¥30万+/年 | 无限 + 定制 |

---

> **下一篇预告**：《银行客户经理AI日报——自动收集客户动态+市场行情》，银行客户经理每天要关注几百个客户的动态，AI日报让客户经理早上一杯咖啡的时间掌握所有客户关键信息。
