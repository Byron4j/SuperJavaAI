# 银行客户经理AI日报：自动收集客户动态+市场行情

> 一个对公客户经理要同时服务80个企业客户，每天光看新闻、查数据、写日报就要2小时。我们用AI做了套智能日报系统，早上一杯咖啡的时间，客户动态+市场行情+待办事项一目了然。

---

## 一、行业痛点：客户经理的信息焦虑

### 一个对公客户经理的周一早晨

某股份制银行深圳分行，对公客户经理小陈打开电脑：

```
8:30 - 到岗，打开邮箱
  - 未读邮件：67封
  - 其中客户相关：约40封
  - 行业新闻简报：5封
  - 内部通知：22封

8:45 - 开始筛查重要信息
  - 客户A某：上周末发了朋友圈说公司要融资 → 机会！
  - 客户B某：行业新闻说他们家被监管部门点名 → 风险！
  - 客户C某：季报出了，营收下滑20% → 需要跟进！
  - 客户D某：大股东减持公告 → 是否影响还款？
  ...（花了40分钟才筛完）

9:30 - 查看市场行情
  - LPR有没有变动？
  - 对公存款利率调整了吗？
  - 行业政策有什么新动态？
  ...（30分钟）

10:00 - 写今日日报/工作计划
  - 今日重点拜访/跟进客户：列清单
  - 待办事项：4-5项
  ...（30分钟）

11:00 - 终于开始真正的工作
  - 但是2.5小时已经过去了
```

**数据统计**：一个对公客户经理平均管理60-100个客户，每天花在信息收集和整理上的时间约2小时——占工作时间的25%。

---

## 二、产品设计：BankDaily AI客户经理日报

```
┌─────────────────────────────────────────────────────┐
│            BankDaily AI 客户经理日报                   │
├─────────────────────────────────────────────────────┤
│                                                       │
│  每天自动执行的信息采集管道（早7点开始）                  │
│                                                       │
│  ┌──────────────────────────────────────────────┐    │
│  │          信息采集层（多数据源并行）              │    │
│  │                                                │    │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────────┐   │    │
│  │  │ 客户动态  │ │ 行业新闻  │ │   市场行情    │   │    │
│  │  │ - 工商变更 │ │ - 行业政策│ │ - LPR/利率   │   │    │
│  │  │ - 司法涉诉 │ │ - 监管动向│ │ - 汇率变动   │   │    │
│  │  │ - 财报公告 │ │ - 竞品动态│ │ - 债券市场   │   │    │
│  │  │ - 舆情监测 │ │ - 技术趋势│ │ - 同业动态   │   │    │
│  │  └──────────┘ └──────────┘ └──────────────┘   │    │
│  │                                                │    │
│  │  ┌──────────┐ ┌──────────┐                     │    │
│  │  │ 内部数据  │ │ 关系图谱  │                     │    │
│  │  │ - 账户异动│ │ - 担保链  │                     │    │
│  │  │ - 到期提醒│ │ - 关联方  │                     │    │
│  │  │ - 额度变化│ │ - 供应链  │                     │    │
│  │  └──────────┘ └──────────┘                     │    │
│  └──────────────────────────────────────────────┘    │
│                         ↓                             │
│  ┌──────────────────────────────────────────────┐    │
│  │           AI智能分析与摘要                      │    │
│  │  • 事件重要性打分 • 风险等级判定 • 机会识别     │    │
│  └──────────────────────────────────────────────┘    │
│                         ↓                             │
│  ┌──────────────────────────────────────────────┐    │
│  │           日报生成 + 推送                       │    │
│  │  • 邮件日报 • 微信推送 • 工作台卡片             │    │
│  └──────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────┘
```

---

## 三、核心Java实现

### 3.1 多源信息采集引擎

```java
@Service
public class MultiSourceCollector {
    
    private final List<DataSourceCollector> collectors;
    
    public MultiSourceCollector() {
        this.collectors = List.of(
            new EnterpriseInfoCollector(),      // 天眼查/企查查API
            new JudicialInfoCollector(),        // 裁判文书/执行信息
            new FinancialReportCollector(),     // 上市公司公告
            new NewsMediaCollector(),           // 新闻/舆情
            new MarketDataCollector(),          // LPR/利率/汇率
            new InternalSystemCollector(),      // 银行内部CRM系统
            new SocialMediaCollector()          // 企业公众号/官网
        );
    }
    
    /**
     * 定时任务：每天早上7:00执行
     */
    @Scheduled(cron = "0 0 7 * * ?")
    public void dailyCollection() {
        log.info("开始每日客户信息采集...");
        
        // 获取所有活跃客户经理及其客户列表
        List<AccountManager> managers = accountManagerService.findAllActive();
        
        for (AccountManager manager : managers) {
            CompletableFuture.runAsync(() -> 
                collectForManager(manager)
            );
        }
    }
    
    private void collectForManager(AccountManager manager) {
        
        List<String> clientIds = manager.getClientIds();
        
        // 并行从多个数据源采集
        Map<DataSourceType, CompletableFuture<List<CollectedItem>>> futures = 
            new HashMap<>();
        
        for (DataSourceCollector collector : collectors) {
            futures.put(collector.getType(), 
                CompletableFuture.supplyAsync(() -> 
                    collector.collect(clientIds, manager.getPreferences())
                )
            );
        }
        
        // 汇聚所有数据源的结果
        List<CollectedItem> allItems = futures.values().stream()
            .flatMap(f -> f.join().stream())
            .collect(Collectors.toList());
        
        // 保存到缓存，供日报生成使用
        cacheService.saveDailyData(manager.getUserId(), allItems);
        
        log.info("客户经理{}客户信息采集完成：{}条原始数据",
            manager.getUserId(), allItems.size());
    }
}

/**
 * 企业信息采集器
 */
@Component
public class EnterpriseInfoCollector implements DataSourceCollector {
    
    @Override
    public DataSourceType getType() { return DataSourceType.ENTERPRISE_INFO; }
    
    @Override
    public List<CollectedItem> collect(List<String> clientIds, 
                                        UserPreferences prefs) {
        
        List<CollectedItem> items = new ArrayList<>();
        
        for (String clientId : clientIds) {
            ClientInfo client = clientService.getById(clientId);
            
            // 调用天眼查/企查查等API获取企业变更信息
            EnterpriseChangeInfo changes = thirdPartyApiService
                .getEnterpriseChanges(client.getCompanyName(), 
                    LocalDate.now().minusDays(1));
            
            if (changes.hasNewChanges()) {
                for (EnterpriseChange change : changes.getAll()) {
                    items.add(CollectedItem.builder()
                        .source(DataSourceType.ENTERPRISE_INFO)
                        .clientId(clientId)
                        .clientName(client.getCompanyName())
                        .title(change.getType() + "变更")
                        .content(change.getDescription())
                        .category(determineCategory(change))
                        .importance(calculateEnterpriseChangeImportance(change))
                        .timestamp(change.getDate())
                        .build());
                }
            }
        }
        
        return items;
    }
    
    private Importance calculateEnterpriseChangeImportance(EnterpriseChange change) {
        return switch (change.getType()) {
            case "法人变更", "股权变更", "注册资本变更" -> Importance.HIGH;
            case "经营范围变更", "地址变更" -> Importance.MEDIUM;
            default -> Importance.LOW;
        };
    }
}

/**
 * 舆情/新闻采集器
 */
@Component
public class NewsMediaCollector implements DataSourceCollector {
    
    @Override
    public List<CollectedItem> collect(List<String> clientIds, 
                                        UserPreferences prefs) {
        
        List<CollectedItem> items = new ArrayList<>();
        
        for (String clientId : clientIds) {
            ClientInfo client = clientService.getById(clientId);
            
            // 从多个新闻源搜索企业相关新闻
            List<NewsArticle> articles = newsApiService.search(
                client.getCompanyName(),
                LocalDate.now().minusDays(2),
                LocalDate.now()
            );
            
            for (NewsArticle article : articles) {
                // AI判断新闻的情感倾向和对客户的影响
                SentimentAnalysis sentiment = analyzeSentiment(article);
                ImpactAssessment impact = assessBusinessImpact(article, client);
                
                items.add(CollectedItem.builder()
                    .source(DataSourceType.NEWS_MEDIA)
                    .clientId(clientId)
                    .clientName(client.getCompanyName())
                    .title(article.getTitle())
                    .content(article.getSummary())
                    .category(Category.NEWS)
                    .sentiment(sentiment.getLabel())
                    .impact(impact)
                    .importance(impact.getImportance())
                    .url(article.getUrl())
                    .timestamp(article.getPublishDate())
                    .build());
            }
        }
        
        return items;
    }
    
    private SentimentAnalysis analyzeSentiment(NewsArticle article) {
        
        String result = chatClient.prompt()
            .system("""
                你是一个金融舆情分析专家。分析新闻对企业的情感倾向。
                
                标签：
                - POSITIVE: 正面（融资成功、业绩增长、获得荣誉）
                - NEGATIVE: 负面（亏损、处罚、纠纷、高管负面）
                - NEUTRAL: 中性（日常经营、行业分析）
                - ALERT: 预警（需要立即关注的负面信息）
                
                JSON: {"label": "NEGATIVE", "score": 0.85, "keyPoints": ["要点1"]}
                """)
            .user("新闻标题：%s\n摘要：%s".formatted(
                article.getTitle(), article.getSummary()))
            .call()
            .content();
        
        return parseSentiment(result);
    }
    
    private ImpactAssessment assessBusinessImpact(NewsArticle article, 
                                                   ClientInfo client) {
        
        double loanAmount = client.getTotalLoanAmount();
        
        String result = chatClient.prompt()
            .system("""
                你是一名银行风险分析师。评估新闻事件对银行信贷业务的影响。
                
                评估维度：
                1. 对企业还款能力的影响程度（1-5分）
                2. 对银行资产质量的影响（1-5分）
                3. 是否需要立即采取行动
                
                输出JSON: {"impactScore": 3, "affectsRepayment": true, "recommendedAction": "..."}
                """)
            .user("""
                企业：%s（在我行贷款：¥%.0f万）
                新闻：%s
                """.formatted(client.getCompanyName(), loanAmount / 10000, 
                             article.getSummary()))
            .call()
            .content();
        
        return parseImpact(result);
    }
}
```

### 3.2 AI日报生成引擎

```java
@Service
public class DailyReportGenerator {
    
    private final ChatClient chatClient;
    
    /**
     * 为指定客户经理生成AI日报
     */
    public DailyReport generateReport(String userId, LocalDate date) {
        
        AccountManager manager = accountManagerService.findById(userId);
        
        // 1. 从缓存获取今日采集的所有数据
        List<CollectedItem> allItems = cacheService.getDailyData(userId, date);
        
        // 2. 按客户聚合
        Map<String, List<CollectedItem>> clientItems = allItems.stream()
            .collect(Collectors.groupingBy(CollectedItem::getClientId));
        
        // 3. 筛选高重要性事件
        List<CollectedItem> highPriorityItems = allItems.stream()
            .filter(i -> i.getImportance() == Importance.HIGH)
            .sorted(Comparator.comparing(CollectedItem::getImportance))
            .collect(Collectors.toList());
        
        // 4. 获取市场行情摘要
        MarketSummary marketSummary = marketDataService.getDailySummary(date);
        
        // 5. 生成待办事项建议
        List<TodoItem> suggestedTodos = generateTodoSuggestions(
            highPriorityItems, manager
        );
        
        // 6. LLM生成日报正文
        String reportBody = generateReportBody(
            manager, clientItems, highPriorityItems, 
            marketSummary, suggestedTodos
        );
        
        // 7. 生成风险预警摘要
        List<RiskAlert> riskAlerts = generateRiskAlerts(allItems);
        
        return DailyReport.builder()
            .userId(userId)
            .managerName(manager.getName())
            .date(date)
            .clientCount(manager.getClientIds().size())
            .totalEvents(allItems.size())
            .highPriorityEvents(highPriorityItems.size())
            .marketSummary(marketSummary)
            .reportBody(reportBody)
            .suggestedTodos(suggestedTodos)
            .riskAlerts(riskAlerts)
            .topStories(selectTopStories(allItems, 5))
            .build();
    }
    
    private String generateReportBody(AccountManager manager,
                                       Map<String, List<CollectedItem>> clientItems,
                                       List<CollectedItem> highPriorityItems,
                                       MarketSummary market,
                                       List<TodoItem> todos) {
        
        // 构建给LLM的上下文
        String clientUpdates = clientItems.entrySet().stream()
            .map(entry -> {
                String clientName = entry.getValue().get(0).getClientName();
                String updates = entry.getValue().stream()
                    .map(i -> String.format("[%s] %s",
                        i.getCategory(), i.getTitle()))
                    .collect(Collectors.joining("；"));
                return String.format("%s：%s", clientName, updates);
            })
            .collect(Collectors.joining("\n"));
        
        return chatClient.prompt()
            .system("""
                你是一名资深银行客户经理的AI助理。为你的老板撰写每日工作日报。
                
                日报结构：
                1. 📊 概览（一句话总结今日要点 + 关键数字）
                2. ⚠️ 重点关注（高风险/高优先级事项，不超过3条）
                3. 💼 客户动态（按重要性排列的关键客户变化）
                4. 📈 市场速览（今日重要的市场行情变化）
                5. ✅ 今日待办（最重要的3-5件事）
                6. 💡 机会提示（可能的业务机会）
                
                语言风格：简洁、专业、有洞察力，像资深助理在汇报。
                每条信息不超过40字。
                """)
            .user("""
                客户经理：%s（管理%d个企业客户）
                
                今日客户动态汇总：
                %s
                
                高优先级事件（%d条）：
                %s
                
                市场行情摘要：
                %s
                
                建议待办事项：
                %s
                
                请撰写日报（200-400字）。
                """.formatted(
                    manager.getName(),
                    manager.getClientIds().size(),
                    clientUpdates,
                    highPriorityItems.size(),
                    highPriorityItems.stream()
                        .map(i -> String.format("[%s优先级] %s：%s",
                            i.getImportance(), i.getClientName(), i.getTitle()))
                        .collect(Collectors.joining("\n")),
                    market.toSummary(),
                    todos.stream()
                        .map(t -> String.format("%s - %s (优先级:%s)",
                            t.getTitle(), t.getDescription(), t.getPriority()))
                        .collect(Collectors.joining("\n"))
                ))
            .call()
            .content();
    }
    
    /**
     * 智能生成待办事项
     */
    private List<TodoItem> generateTodoSuggestions(
            List<CollectedItem> highPriorityItems,
            AccountManager manager) {
        
        if (highPriorityItems.isEmpty()) {
            return List.of(
                TodoItem.of("跟进潜在客户开发", 
                    "今日无紧急事项，建议做客户拓展工作", Priority.LOW)
            );
        }
        
        String todosJson = chatClient.prompt()
            .system("""
                你是银行客户经理的工作助理。根据今日客户动态，
                生成3-5条待办事项建议。
                
                待办类型：
                - CALL: 电话联系客户
                - VISIT: 拜访客户
                - REVIEW: 审核/评估
                - REPORT: 向上汇报
                - FOLLOW_UP: 跟进
                
                JSON: [{"type": "CALL", "title": "...", "description": "...", "priority": "HIGH", "clientName": "..."}]
                """)
            .user("""
                客户经理：%s
                今日重点事件：
                %s
                """.formatted(
                    manager.getName(),
                    highPriorityItems.stream()
                        .map(i -> "%s - %s".formatted(i.getClientName(), i.getTitle()))
                        .collect(Collectors.joining("\n"))
                ))
            .call()
            .content();
        
        return parseTodos(todosJson);
    }
    
    /**
     * 风险预警生成
     */
    private List<RiskAlert> generateRiskAlerts(List<CollectedItem> allItems) {
        
        return allItems.stream()
            .filter(i -> i.getSentiment() == Sentiment.ALERT
                      || i.getImpact().getImpactScore() >= 4)
            .map(i -> RiskAlert.builder()
                .clientName(i.getClientName())
                .alertType(determineAlertType(i))
                .description(i.getContent())
                .severity(i.getImpact().getImpactScore() >= 5 
                    ? Severity.CRITICAL : Severity.WARNING)
                .recommendedAction(i.getImpact().getRecommendedAction())
                .build())
            .collect(Collectors.toList());
    }
}
```

### 3.3 智能推送策略

```java
@Component
public class SmartPushService {
    
    /**
     * 根据紧急度和用户偏好选择推送渠道
     */
    public void pushDailyReport(DailyReport report, AccountManager manager) {
        
        // 渠道1: 邮件（默认，包含完整报告）
        sendEmailReport(report, manager.getEmail());
        
        // 渠道2: 微信/飞书（短摘要）
        String wechatSummary = generateWechatSummary(report);
        sendWechatMessage(manager.getWechatId(), wechatSummary);
        
        // 渠道3: 如果有紧急预警，立即电话/短信通知
        if (report.hasCriticalAlerts()) {
            sendUrgentNotification(manager, report.getRiskAlerts());
        }
    }
    
    private void sendUrgentNotification(AccountManager manager,
                                         List<RiskAlert> criticalAlerts) {
        
        String smsContent = String.format(
            "【紧急预警】%d条客户风险预警，请立即查看日报。涉及客户：%s",
            criticalAlerts.size(),
            criticalAlerts.stream()
                .map(RiskAlert::getClientName)
                .collect(Collectors.joining("、"))
        );
        
        smsService.send(manager.getPhone(), smsContent);
    }
}
```

---

## 四、商业模式

| 版本 | 价格 | 管理客户数 |
|------|------|-----------|
| 个人版 | ¥99/月 | 20个客户 |
| 专业版 | ¥299/月 | 100个客户 |
| 团队版 | ¥999/月 | 500个客户+团队协作 |
| 企业版 | 定制 | 无限+银行系统集成 |

**目标市场**：全国银行对公客户经理约50万人，证券/保险/信托客户经理约30万人。

---

> **下一篇预告**：《餐饮菜单AI优化——定价+排列+文案一体化工具》，餐厅菜单不是随便写的，定价心理学+视觉热区+文案设计，AI一键优化让客单价提升15%。
