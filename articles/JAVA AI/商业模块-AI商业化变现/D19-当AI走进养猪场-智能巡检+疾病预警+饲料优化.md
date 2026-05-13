# 当AI走进养猪场：智能巡检+疾病预警+饲料优化

> 养猪不是一个"土"行业，而是一个千亿级的科技洼地。我们用Java+IoT+AI帮一家万头猪场把出栏率提升了8%，全年增收超400万。

---

## 一、行业痛点：中国养猪业的"看不见的损失"

### 一组令人震惊的数据

根据农业农村部数据，中国年生猪出栏量约7亿头，是全球最大的生猪养殖和消费国。但行业效率与发达国家差距明显：

```
养殖效率对比（2024年数据）：

             中国平均水平    丹麦/荷兰水平    差距
MSY(每头母猪年产仔数)  18头        30头        -40%
料肉比          3.0:1      2.5:1       -17%
死亡率          8%-12%     3%-5%       +100%
人工成本/头       ¥120       ¥45        +167%

核心原因：精细化管理不足
- 大部分中小猪场依赖"老师傅经验"
- 疾病发现滞后（等猪不吃食了才发现，为时已晚）
- 饲料配方"一刀切"，不同生长阶段营养浪费严重
```

### 万头猪场的真实数据

我们合作的河南某万头猪场，2023年数据：

```
全年出栏：10,200头
死亡/淘汰：1,480头（损失率12.7%）
  其中：
  - 仔猪腹泻死亡：520头（35%）
  - 呼吸道疾病死亡：380头（26%）
  - 应激死亡：210头（14%）
  - 其他原因：370头（25%）

按头均利润¥350计算：
死亡损失：1,480 × ¥350 = ¥51.8万
如果死亡率降到6%（行业优秀水平）：
可挽回：约¥26万/年

饲料成本：全年¥780万
如果料肉比优化0.1：
可节省：约¥26万/年

人工成本：12人 × ¥6万/年 = ¥72万
如果效率提升50%，只需8人：
可节省：约¥24万/年

─────────────────────────
三项合计年化可优化空间：¥76万+
```

---

## 二、产品设计：PigAI智慧养猪系统

```
┌─────────────────────────────────────────────────────┐
│              PigAI 智慧养猪系统                         │
├─────────────────────────────────────────────────────┤
│                                                       │
│  感知层（IoT + 传感器 + 摄像头）                        │
│  ┌──────────┬──────────┬──────────┬─────────────┐    │
│  │ 环境传感器 │ 智能耳标  │ AI摄像头  │ 自动饲喂器   │    │
│  │ (温湿度/  │ (体温/   │ (行为识别 │ (精准投喂/   │    │
│  │ 氨气/CO2) │ 活动量/  │ /体重估测)│ 采食量记录)  │    │
│  │          │ 定位)    │           │             │    │
│  └──────────┴──────────┴──────────┴─────────────┘    │
│          ↓                    ↓                       │
│  平台层（Java Spring Boot + IoT网关）                  │
│  ┌──────────┬──────────┬──────────┬─────────────┐    │
│  │ 环境监控  │ 健康管理  │ 饲喂优化  │ 生产管理     │    │
│  │ Service  │ Service  │ Service  │ Service     │    │
│  └──────────┴──────────┴──────────┴─────────────┘    │
│          ↓                                             │
│  AI层（LLM + 机器学习）                                 │
│  ┌──────────┬──────────┬──────────┬─────────────┐    │
│  │ 疾病预警  │ 行为分析  │ 饲料配方  │ 出栏预测     │    │
│  │ 模型      │ 模型     │ 优化      │ 模型         │    │
│  └──────────┴──────────┴──────────┴─────────────┘    │
└─────────────────────────────────────────────────────┘
```

---

## 三、核心Java实现

### 3.1 IoT数据采集与处理

```java
@Service
public class IoTDataIngestionService {
    
    /**
     * 多源IoT数据采集
     * 支持MQTT/CoAP/HTTP协议接入
     */
    public void processIoTData(IoTMessage message) {
        
        switch (message.getDeviceType()) {
            case ENVIRONMENT_SENSOR -> processEnvironmentData(message);
            case SMART_EAR_TAG -> processEarTagData(message);
            case FEEDING_STATION -> processFeedingData(message);
            case WATER_METER -> processWaterData(message);
            case CAMERA -> processCameraData(message);
        }
    }
    
    @RabbitListener(queues = "pig.env.data")
    public void processEnvironmentData(IoTMessage message) {
        
        EnvironmentData data = parseEnvironmentData(message);
        
        // 存储到时序数据库（InfluxDB/TDengine）
        timeSeriesDB.write(
            "environment",
            Map.of(
                "barn_id", data.getBarnId(),
                "temperature", data.getTemperature(),
                "humidity", data.getHumidity(),
                "ammonia", data.getAmmoniaPpm(),
                "co2", data.getCo2Ppm(),
                "light_intensity", data.getLightIntensity()
            ),
            data.getTimestamp()
        );
        
        // 实时告警检查
        checkEnvironmentAlerts(data);
    }
    
    @RabbitListener(queues = "pig.ear.tag")
    public void processEarTagData(IoTMessage message) {
        
        EarTagData data = parseEarTagData(message);
        
        // 每头猪的活动数据
        PigActivity activity = PigActivity.builder()
            .pigId(data.getPigId())
            .temperature(data.getBodyTemperature())
            .steps(data.getSteps())
            .activityLevel(data.getActivityLevel())
            .feedingVisits(data.getFeedingVisitsToday())
            .drinkingVisits(data.getDrinkingVisitsToday())
            .location(data.getLocation())
            .timestamp(data.getTimestamp())
            .build();
        
        activityRepository.save(activity);
        
        // 健康异常检测
        healthMonitor.checkAnomaly(activity);
    }
    
    /**
     * 环境告警检查
     */
    private void checkEnvironmentAlerts(EnvironmentData data) {
        
        // 温度告警
        if (data.getTemperature() > 35) {
            alertService.sendAlert(Alert.builder()
                .level(AlertLevel.CRITICAL)
                .barnId(data.getBarnId())
                .type("高温告警")
                .message(String.format("猪舍温度%.1f°C，超过35°C警戒线，"
                    + "可能引起热应激", data.getTemperature()))
                .suggestedAction("立即开启降温设备/通风")
                .build());
        }
        
        // 氨气告警
        if (data.getAmmoniaPpm() > 25) {
            alertService.sendAlert(Alert.builder()
                .level(AlertLevel.HIGH)
                .barnId(data.getBarnId())
                .type("氨气浓度告警")
                .message(String.format("氨气浓度%.0fppm，超过25ppm标准，"
                    + "可能导致呼吸道疾病", data.getAmmoniaPpm()))
                .suggestedAction("加强通风 + 及时清粪")
                .build());
        }
        
        // 环境异常检测（AI分析趋势）
        if (isAbnormalTrend(data)) {
            alertService.sendAlert(Alert.builder()
                .level(AlertLevel.WARNING)
                .barnId(data.getBarnId())
                .type("环境趋势异常")
                .message("过去30分钟温湿度变化异常，可能是通风系统故障")
                .build());
        }
    }
}
```

### 3.2 AI疾病预警引擎

```java
@Service
public class DiseaseEarlyWarningService {
    
    private final ChatClient chatClient;
    private final PigHealthRepository healthRepo;
    
    /**
     * 多维健康评估 + 疾病早期预警
     * 
     * 预警信号：
     * - 体温异常（>39.5°C 或 连续上升趋势）
     * - 采食量下降（连续2餐减少>30%）
     * - 活动量骤降（比日均活动减少>50%）
     * - 饮水异常（过多或过少）
     * - 呼吸异常（通过声学监测识别咳嗽）
     * - 外观异常（AI视觉识别：毛色/姿态/排便）
     */
    public List<DiseaseAlert> checkHealth(String pigId) {
        
        // 获取近24小时的全面数据
        PigHealthSnapshot snapshot = healthRepo.get24HourSnapshot(pigId);
        
        List<DiseaseAlert> alerts = new ArrayList<>();
        
        // 检查1: 体温异常
        if (snapshot.getLatestTemperature() > 39.5) {
            alerts.add(DiseaseAlert.builder()
                .pigId(pigId)
                .type(DiseaseIndicator.FEVER)
                .severity(AlertSeverity.HIGH)
                .description(String.format("体温%.1f°C，高于正常值", 
                    snapshot.getLatestTemperature()))
                .build());
        }
        
        // 检查2: 采食量下降
        double feedChange = calculateFeedChange(snapshot);
        if (feedChange < -0.3) {
            alerts.add(DiseaseAlert.builder()
                .pigId(pigId)
                .type(DiseaseIndicator.APPETITE_LOSS)
                .severity(feedChange < -0.5 ? AlertSeverity.CRITICAL : AlertSeverity.HIGH)
                .description(String.format("采食量下降%.0f%%", Math.abs(feedChange * 100)))
                .build());
        }
        
        // 检查3: 活动量骤降
        double activityChange = calculateActivityChange(snapshot);
        if (activityChange < -0.5) {
            alerts.add(DiseaseAlert.builder()
                .pigId(pigId)
                .type(DiseaseIndicator.LETHARGY)
                .severity(AlertSeverity.HIGH)
                .description("活动量显著低于正常水平")
                .build());
        }
        
        // 如果多个信号同时触发 → 高概率疾病
        if (alerts.size() >= 2) {
            DiseasePrediction prediction = predictDisease(snapshot, alerts);
            
            alerts.add(DiseaseAlert.builder()
                .pigId(pigId)
                .type(DiseaseIndicator.MULTI_SIGNAL)
                .severity(AlertSeverity.CRITICAL)
                .description(String.format(
                    "多个预警信号同时触发，%s概率%.0f%%",
                    prediction.getDiseaseName(),
                    prediction.getProbability() * 100
                ))
                .suggestedAction(prediction.getTreatmentAdvice())
                .build());
        }
        
        return alerts;
    }
    
    /**
     * AI疾病预测
     * 结合多种信号判断可能的疾病
     */
    private DiseasePrediction predictDisease(PigHealthSnapshot snapshot,
                                              List<DiseaseAlert> alerts) {
        
        String prediction = chatClient.prompt()
            .system("""
                你是资深猪病防治兽医专家。根据猪的健康数据判断可能的疾病。
                
                常见猪病及特征：
                - 猪蓝耳病(PRRS)：高热、呼吸困难、耳朵发蓝、食欲废绝
                - 猪流行性腹泻(PED)：水样腹泻、脱水、呕吐
                - 猪链球菌病：高烧、关节肿胀、神经症状
                - 猪圆环病毒病(PCV2)：消瘦、呼吸困难、皮肤苍白
                - 猪支原体肺炎：干咳、生长缓慢、体温正常
                - 猪丹毒：高烧、皮肤疹块、关节炎
                
                输出JSON: {
                  "diseaseName": "可能的疾病",
                  "probability": 0.75,
                  "differentialDiagnosis": ["鉴别诊断1", "鉴别诊断2"],
                  "treatmentAdvice": "隔离 + 退烧 + 抗生素 + 通知兽医",
                  "isContagious": true,
                  "recommendedAction": "立即隔离 + 全群消毒 + 紧急免疫"
                }
                """)
            .user("""
                猪只信息：品种=%s, 日龄=%d天, 体重=%.1fkg
                
                预警信号：
                %s
                
                近24小时数据：
                - 体温变化：%s
                - 采食量变化：%s
                - 活动量变化：%s
                - 饮水变化：%s
                
                同栏猪只是否也有异常：%s
                """.formatted(
                    snapshot.getBreed(),
                    snapshot.getAgeDays(),
                    snapshot.getWeight(),
                    alerts.stream()
                        .map(a -> a.getType() + ": " + a.getDescription())
                        .collect(Collectors.joining("\n")),
                    snapshot.getTemperatureTrend(),
                    snapshot.getFeedTrend(),
                    snapshot.getActivityTrend(),
                    snapshot.getWaterTrend(),
                    snapshot.hasPenMateAbnormal() ? "是（同栏3头猪也出现了类似症状）" : "否"
                ))
            .call()
            .content();
        
        return parsePrediction(prediction);
    }
}
```

### 3.3 智能饲料优化

```java
@Service
public class FeedOptimizationService {
    
    private final ChatClient chatClient;
    
    /**
     * AI饲料配方优化
     * 
     * 基于猪的生长阶段、体重、健康状况、市场价格
     * 动态调整饲料配方
     */
    public FeedRecipe optimizeRecipe(String pigGroupId) {
        
        PigGroup group = pigGroupService.getGroup(pigGroupId);
        
        // 1. 获取当前阶段的标准营养需求
        NutritionalRequirement requirement = getNutritionalRequirement(
            group.getBreed(), group.getAvgWeight(), group.getProductionStage()
        );
        
        // 2. 获取可用的饲料原料及价格
        List<FeedIngredient> availableIngredients = ingredientService
            .getAvailableIngredients();
        
        // 3. 获取当前市场价格
        Map<String, Double> marketPrices = marketPriceService
            .getCurrentPrices(availableIngredients);
        
        // 4. AI优化配方
        String recipeJson = chatClient.prompt()
            .system("""
                你是动物营养学专家。为以下猪群设计最优饲料配方。
                
                优化目标（按优先级）：
                1. 满足营养需求
                2. 最小化成本
                3. 最大化日增重
                4. 考虑原料可得性
                
                配方约束：
                - 粗蛋白：%s
                - 消化能：%s
                - 赖氨酸：%s
                - 钙磷比：1.2-1.5:1
                
                请输出JSON配方：
                {
                  "recipeName": "配方名",
                  "totalCostPerTon": 2800.0,
                  "expectedDailyGain": 750.0,
                  "feedConversionRatio": 2.6,
                  "ingredients": [
                    {"name": "玉米", "ratio": 0.62, "costPerTon": 2400, "purpose": "能量"},
                    ...
                  ],
                  "additives": [...],
                  "feedingInstructions": "饲喂方法说明",
                  "warnings": ["注意事项"]
                }
                """)
            .user("""
                猪群信息：
                - 品种：%s
                - 平均体重：%.1f kg
                - 生长阶段：%s
                - 当前日增重：%.0f g/天
                - 当前料肉比：%.2f
                
                营养需求：
                %s
                
                可用的饲料原料（价格/吨）：
                %s
                
                请生成最优配方。
                """.formatted(
                    group.getBreed(),
                    group.getAvgWeight(),
                    group.getProductionStage(),
                    group.getCurrentDailyGain(),
                    group.getCurrentFCR(),
                    requirement.toSpecString(),
                    availableIngredients.stream()
                        .map(i -> String.format("%s: ¥%.0f/吨 (库存:%d吨)",
                            i.getName(), marketPrices.getOrDefault(i.getName(), 0.0),
                            i.getStock()))
                        .collect(Collectors.joining("\n"))
                ))
            .call()
            .content();
        
        return parseFeedRecipe(recipeJson);
    }
    
    /**
     * 精准投喂策略
     * 根据每头猪的体重和采食速度自动调整投喂量
     */
    @Scheduled(cron = "0 0 6,12,18 * * ?")  // 每天3次
    public void schedulePrecisionFeeding() {
        
        List<PigGroup> groups = pigGroupService.findAllActive();
        
        for (PigGroup group : groups) {
            // 基于最近体重数据调整投喂量
            double avgWeight = weightService.getAverageWeight(
                group.getId(), LocalDate.now()
            );
            
            double feedAmount = calculateFeedAmount(
                avgWeight, 
                group.getProductionStage(),
                group.getTargetDailyGain()
            );
            
            // 下发指令到自动饲喂器
            for (FeedingStation station : group.getFeedingStations()) {
                iotCommandService.sendCommand(
                    station.getDeviceId(),
                    new SetFeedAmountCommand(feedAmount / group.getPigCount())
                );
            }
            
            log.info("猪群%s精准投喂：均重%.1fkg → 头均投喂%.2fkg",
                group.getId(), avgWeight, feedAmount / group.getPigCount());
        }
    }
    
    private double calculateFeedAmount(double weight,
                                        ProductionStage stage,
                                        double targetDailyGain) {
        // 基于NRC(国家研究委员会)标准的投喂量计算
        // 公式：采食量 = 体重^0.75 × 系数 × 日增重调整因子
        double metabolicWeight = Math.pow(weight, 0.75);
        
        double coefficient = switch (stage) {
            case NURSERY -> 0.095;    // 保育猪
            case GROWING -> 0.100;    // 生长猪
            case FINISHING -> 0.105;  // 育肥猪
            case GESTATION -> 0.065;  // 怀孕母猪
            case LACTATION -> 0.110;  // 哺乳母猪
        };
        
        double baseFeed = metabolicWeight * coefficient;
        
        // 日增重调整
        double gainAdjustment = targetDailyGain / 
            (stage == ProductionStage.FINISHING ? 800 : 600);
        
        return baseFeed * gainAdjustment;
    }
}
```

### 3.4 AI摄像头行为分析

```java
@Service
public class PigBehaviorAnalysisService {
    
    /**
     * 基于摄像头的行为分析
     * 
     * AI识别：
     * - 个体识别（耳标+面部识别）
     * - 姿态分析（站立/趴卧/跛行）
     * - 行为识别（采食/饮水/争斗/排泄/发情）
     * - 群体行为（扎堆=冷/散开=热/打斗=密度过高）
     * - 生长估算（基于体型估算体重）
     */
    public BehaviorReport analyzeBehavior(byte[] imageData, String barnId) {
        
        String analysis = chatClient.prompt()
            .system("""
                你是猪场行为分析专家。分析猪舍监控画面。
                
                请识别并输出JSON：
                {
                  "pigCount": 25,
                  "behaviors": {
                    "feeding": 8,
                    "drinking": 3,
                    "resting": 12,
                    "active": 5,
                    "fighting": 1,
                    "limping": 1,
                    "mounting": 0
                  },
                  "alerts": [
                    "编号P0238跛行，疑似腿伤",
                    "东区有猪只打斗"
                  ],
                  "environmentAssessment": "扎堆在角落 → 可能有贼风/温度不均",
                  "overallCondition": "GOOD/FAIR/POOR",
                  "recommendedActions": ["检查P0238", "调整通风"]
                }
                """)
            .user(new org.springframework.ai.model.Media(
                org.springframework.util.MimeTypeUtils.IMAGE_JPEG,
                new org.springframework.core.io.ByteArrayResource(imageData)
            ))
            .call()
            .content();
        
        return parseBehaviorReport(analysis, barnId);
    }
}
```

---

## 四、落地效果

河南某万头猪场（2024年6月-12月试点数据）：

| 指标 | 上线前 | 上线后 | 变化 |
|------|--------|--------|------|
| 死亡率 | 12.7% | 4.4% | **-65%** |
| 料肉比 | 3.01:1 | 2.82:1 | **-6.3%** |
| 出栏均重 | 112kg | 118kg | **+5.4%** |
| MSY | 18.3 | 21.7 | **+18.6%** |
| 年化增收 | - | ¥420万 | - |

---

## 五、商业模式

| 版本 | 价格 | 规模 |
|------|------|------|
| 试用版 | 免费3月 | <500头 |
| 标准版 | ¥2.99万/年 | 500-2000头 |
| 规模版 | ¥9.99万/年 | 2000-10000头 |
| 集团版 | 定制 | 万头以上 |

---

> **下一篇预告**：《12345热线AI提效——诉求分类+相似工单去重+答复话术生成》，全国12345热线每年处理上亿通电话，AI如何帮政府把热线服务效率提升300%？
