# 定时任务 Agent：每天自动收集行业新闻并生成日报，你的私人 AI 情报员

## 一、引言：信息焦虑时代的自救方案

每天早上打开电脑，十几篇文章要刷——技术博客、产品动态、竞品报告、行业政策、投资动向。光是把这些文章标题扫一遍就要半小时，更别提炼有价值的摘要了。

管理者更痛苦：要让助理每天整理行业日报，找人，培训，汇报格式统一，一套流程下来成本不低。

如果我告诉你，一个 Java 程序就能自动完成这些——每天 7:00 自动爬取你关注的新闻源，AI 摘要后生成日报，9:00 前准时推送到你的飞书/钉钉——你愿意花 30 分钟把它搭起来吗？

今天我们就来写一个**定时任务 Agent**，一个能自动收集情报、智能整理、定时推送的 AI 助手。

> 全文约 5000 字。完整代码可直接运行，文末有 GitHub 仓库地址。

---

## 二、架构设计：三阶段流水线

定时任务 Agent 的运作是一个典型的 ETL 流程——Extract、Transform、Load，但每一步都有 AI 增强：

```
┌──────────────────────────────────────────────────────────┐
│            Schedule Agent 三阶段流水线                      │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  ┌─────────┐    ┌──────────────┐    ┌──────────────┐     │
│  │ Stage 1 │ →  │   Stage 2    │ →  │   Stage 3    │     │
│  │ 信息采集 │    │  AI智能处理   │    │  日报生成推送 │     │
│  └─────────┘    └──────────────┘    └──────────────┘     │
│       │                │                    │            │
│  · RSS源抓取      · 去重合并          · Markdown编排      │
│  · 网页爬取       · 内容摘要          · 分类排版          │
│  · API数据        · 关键词提取        · 飞书/钉钉推送     │
│  · GitHub Trending · 重要性评分       · 邮件发送          │
│  · 行业公众号     · 情感分析          · 定时调度          │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

时间规划：
- **06:00**：Quartz 触发采集任务
- **06:00 - 06:30**：并行爬取所有信息源
- **06:30 - 07:00**：AI 处理（摘要、分类、评分）
- **07:00 - 07:15**：日报编排生成
- **07:30**：推送到飞书群 / 钉钉群 / 邮件
- **09:00**：团队成员阅读日报

---

## 三、Stage 1：多渠道信息采集

### 3.1 信息源抽象与注册

```java
/**
 * 信息源抽象接口
 */
public interface NewsSource {
    
    /** 信息源名称 */
    String getName();
    
    /** 信息源类型 */
    SourceType getType();
    
    /**
     * 抓取文章列表
     * @param sinceHours 只获取最近 N 小时内的文章
     */
    List<NewsArticle> fetch(int sinceHours);
    
    /** 该信息源是否启用 */
    default boolean isEnabled() { return true; }
    
    /** 权重（重要性系数，0.0-1.0） */
    default double getWeight() { return 1.0; }
}

enum SourceType {
    RSS,           // RSS 订阅源
    WEB_SCRAPE,    // 网页爬虫
    API,           // 第三方 API
    SOCIAL_MEDIA,  // 社交媒体
    GITHUB,        // GitHub Trending / Releases
    CUSTOM         // 自定义
}

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class NewsArticle {
    private String id;              // 唯一标识
    private String title;           // 标题
    private String url;             // 原文链接
    private String summary;         // 原始摘要（来自 RSS）
    private String content;         // 正文内容
    private String author;          // 作者
    private String source;          // 来源名称
    private SourceType sourceType;  // 来源类型
    private LocalDateTime publishTime; // 发布时间
    private LocalDateTime fetchTime;   // 采集时间
    private List<String> tags;      // 原始标签
    private Map<String, Object> metadata; // 扩展元数据
    
    // AI 处理后填充的字段
    private String aiSummary;       // AI 摘要
    private List<String> aiKeywords; // AI 提取的关键词
    private String aiCategory;      // AI 分类（产品/技术/行业/政策/竞品/投资）
    private double importanceScore;  // AI 评分（0-10）
    private String sentiment;       // 情感分析（正面/负面/中性）
}
```

### 3.2 RSS 源采集器

```java
@Component
@Slf4j
public class RSSNewsSource implements NewsSource {
    
    private final ObjectMapper objectMapper;
    private final OkHttpClient httpClient;
    private final List<String> rssUrls;
    
    public RSSNewsSource(
            @Value("${news-agent.sources.rss.urls}") String rssUrlsStr,
            ObjectMapper objectMapper) {
        this.objectMapper = objectMapper;
        this.httpClient = new OkHttpClient();
        this.rssUrls = Arrays.asList(rssUrlsStr.split(","));
    }
    
    @Override
    public String getName() { return "RSS聚合源"; }
    
    @Override
    public SourceType getType() { return SourceType.RSS; }
    
    @Override
    public List<NewsArticle> fetch(int sinceHours) {
        List<NewsArticle> articles = new ArrayList<>();
        LocalDateTime since = LocalDateTime.now().minusHours(sinceHours);
        
        for (String rssUrl : rssUrls) {
            try {
                List<NewsArticle> fetched = fetchSingleRSS(rssUrl, since);
                articles.addAll(fetched);
                log.info("RSS {} fetched {} articles", rssUrl, fetched.size());
            } catch (Exception e) {
                log.error("Failed to fetch RSS: {}", rssUrl, e);
            }
        }
        
        return articles;
    }
    
    private List<NewsArticle> fetchSingleRSS(String rssUrl, LocalDateTime since) 
            throws Exception {
        Request request = new Request.Builder()
                .url(rssUrl.trim())
                .header("User-Agent", "NewsBot/1.0")
                .build();
        
        try (Response response = httpClient.newCall(request).execute()) {
            String xmlBody = response.body().string();
            return parseRSSXML(xmlBody, since);
        }
    }
    
    private List<NewsArticle> parseRSSXML(String xml, LocalDateTime since) {
        List<NewsArticle> articles = new ArrayList<>();
        
        try {
            Document doc = DocumentBuilderFactory.newInstance()
                    .newDocumentBuilder()
                    .parse(new InputSource(new StringReader(xml)));
            
            NodeList items = doc.getElementsByTagName("item");
            
            for (int i = 0; i < items.getLength(); i++) {
                Element item = (Element) items.item(i);
                
                String title = getTextContent(item, "title");
                String link = getTextContent(item, "link");
                String description = getTextContent(item, "description");
                String pubDate = getTextContent(item, "pubDate");
                
                LocalDateTime publishTime = parsePubDate(pubDate);
                
                // 过滤太旧的文章
                if (publishTime != null && publishTime.isBefore(since)) {
                    continue;
                }
                
                NewsArticle article = NewsArticle.builder()
                        .id(DigestUtils.md5Hex(link))
                        .title(cleanHtml(title))
                        .url(link)
                        .summary(cleanHtml(description))
                        .source(getName())
                        .sourceType(SourceType.RSS)
                        .publishTime(publishTime != null ? publishTime : LocalDateTime.now())
                        .fetchTime(LocalDateTime.now())
                        .tags(extractCategories(item))
                        .build();
                
                articles.add(article);
            }
        } catch (Exception e) {
            log.error("Failed to parse RSS XML", e);
        }
        
        return articles;
    }
    
    private String getTextContent(Element parent, String tagName) {
        NodeList nodes = parent.getElementsByTagName(tagName);
        if (nodes.getLength() > 0) {
            return nodes.item(0).getTextContent();
        }
        return "";
    }
    
    private LocalDateTime parsePubDate(String pubDate) {
        if (pubDate == null || pubDate.isEmpty()) return null;
        
        // RSS 有多种日期格式
        DateTimeFormatter[] formatters = {
                DateTimeFormatter.RFC_1123_DATE_TIME,
                DateTimeFormatter.ofPattern("yyyy-MM-dd'T'HH:mm:ssXXX"),
                DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss"),
                DateTimeFormatter.ofPattern("EEE, dd MMM yyyy HH:mm:ss Z")
        };
        
        for (DateTimeFormatter formatter : formatters) {
            try {
                return LocalDateTime.parse(pubDate.trim(), formatter);
            } catch (Exception e) {
                // try next format
            }
        }
        return null;
    }
    
    private String cleanHtml(String html) {
        if (html == null) return "";
        return html.replaceAll("<[^>]*>", "")
                .replaceAll("&nbsp;", " ")
                .replaceAll("&amp;", "&")
                .replaceAll("&lt;", "<")
                .replaceAll("&gt;", ">")
                .replaceAll("&quot;", "\"")
                .replaceAll("\\s+", " ")
                .trim();
    }
    
    private List<String> extractCategories(Element item) {
        NodeList cats = item.getElementsByTagName("category");
        List<String> categories = new ArrayList<>();
        for (int i = 0; i < cats.getLength(); i++) {
            categories.add(cats.item(i).getTextContent());
        }
        return categories;
    }
}
```

### 3.3 GitHub Trending 采集器

```java
@Component
@Slf4j
public class GitHubTrendingSource implements NewsSource {
    
    private final OkHttpClient httpClient;
    
    public GitHubTrendingSource() {
        this.httpClient = new OkHttpClient();
    }
    
    @Override
    public String getName() { return "GitHub Trending"; }
    
    @Override
    public SourceType getType() { return SourceType.GITHUB; }
    
    @Override
    public double getWeight() { return 0.8; }
    
    @Override
    public List<NewsArticle> fetch(int sinceHours) {
        List<NewsArticle> articles = new ArrayList<>();
        
        // 抓取 GitHub Trending 页面
        String[] languages = {"java", "python", "typescript", "go", "rust", "ai"};
        
        for (String lang : languages) {
            try {
                List<NewsArticle> langArticles = fetchTrendingForLanguage(lang);
                articles.addAll(langArticles);
                // 礼貌性延迟
                Thread.sleep(2000);
            } catch (Exception e) {
                log.error("Failed to fetch GitHub trending for {}", lang, e);
            }
        }
        
        return articles;
    }
    
    private List<NewsArticle> fetchTrendingForLanguage(String language) throws IOException {
        Request request = new Request.Builder()
                .url("https://github.com/trending/" + language + "?since=daily")
                .header("User-Agent", "NewsBot/1.0")
                .header("Accept", "text/html")
                .build();
        
        try (Response response = httpClient.newCall(request).execute()) {
            String html = response.body().string();
            return parseTrendingHTML(html, language);
        }
    }
    
    private List<NewsArticle> parseTrendingHTML(String html, String language) {
        List<NewsArticle> articles = new ArrayList<>();
        
        // 使用 Jsoup 解析（简化版本，生产环境建议用专用解析器）
        Pattern repoPattern = Pattern.compile(
                "/trending/[^\"]*\"[^>]*>\\s*<h2[^>]*>\\s*<a[^>]*href=\"/([^\"]+)\"[^>]*>\\s*(?:<span[^>]*>)?\\s*([^<]+)\\s*(?:</span>)?\\s*/\\s*\\1\\s*</a>",
                Pattern.DOTALL);
        
        Pattern descPattern = Pattern.compile(
                "<p class=\"col-9 color-fg-muted my-1 pr-4\">\\s*([^<]*)\\s*</p>");
        
        Matcher repoMatcher = repoPattern.matcher(html);
        List<String> repos = new ArrayList<>();
        while (repoMatcher.find()) {
            repos.add(repoMatcher.group(1));
        }
        
        Matcher descMatcher = descPattern.matcher(html);
        List<String> descriptions = new ArrayList<>();
        while (descMatcher.find()) {
            descriptions.add(descMatcher.group(1).trim());
        }
        
        for (int i = 0; i < Math.min(repos.size(), 25); i++) {
            String desc = i < descriptions.size() ? descriptions.get(i) : "";
            articles.add(NewsArticle.builder()
                    .id(DigestUtils.md5Hex(repos.get(i)))
                    .title(repos.get(i) + " - " + desc)
                    .url("https://github.com/" + repos.get(i))
                    .summary(desc)
                    .source(getName())
                    .sourceType(SourceType.GITHUB)
                    .publishTime(LocalDateTime.now())
                    .fetchTime(LocalDateTime.now())
                    .tags(List.of(language, "trending"))
                    .build());
        }
        
        return articles;
    }
}
```

---

## 四、Stage 2：AI 智能处理引擎

```java
@Service
@Slf4j
public class AINewsProcessor {
    
    private final OpenAiService openAiService;
    private final String model;
    private final int batchSize;
    
    public AINewsProcessor(OpenAiService openAiService,
                           @Value("${news-agent.llm.model}") String model,
                           @Value("${news-agent.llm.batch-size:10}") int batchSize) {
        this.openAiService = openAiService;
        this.model = model;
        this.batchSize = batchSize;
    }
    
    /**
     * 批量处理文章：AI 摘要 + 分类 + 评分
     */
    public List<NewsArticle> process(List<NewsArticle> articles) {
        log.info("AI Processing {} articles...", articles.size());
        
        // 分批处理
        List<List<NewsArticle>> batches = partition(articles, batchSize);
        List<CompletableFuture<List<NewsArticle>>> futures = new ArrayList<>();
        
        for (List<NewsArticle> batch : batches) {
            futures.add(CompletableFuture.supplyAsync(() -> processBatch(batch)));
        }
        
        // 等待所有批次完成
        List<NewsArticle> processed = futures.stream()
                .map(CompletableFuture::join)
                .flatMap(List::stream)
                .collect(Collectors.toList());
        
        log.info("AI Processing complete");
        return processed;
    }
    
    private List<NewsArticle> processBatch(List<NewsArticle> batch) {
        String prompt = buildBatchPrompt(batch);
        
        try {
            List<ChatMessage> messages = List.of(
                    new ChatMessage(ChatMessageRole.SYSTEM.value(), getSystemPrompt()),
                    new ChatMessage(ChatMessageRole.USER.value(), prompt)
            );
            
            ChatCompletionRequest request = ChatCompletionRequest.builder()
                    .model(model)
                    .messages(messages)
                    .temperature(0.3)
                    .maxTokens(3000)
                    .responseFormat(Map.of("type", "json_object"))
                    .build();
            
            ChatCompletionResult response = openAiService.createChatCompletion(request);
            String content = response.getChoices().get(0).getMessage().getContent();
            
            return parseAIResult(content, batch);
        } catch (Exception e) {
            log.error("AI processing batch failed", e);
            // 失败时返回原始文章
            return batch;
        }
    }
    
    private String getSystemPrompt() {
        return """
            你是一个专业的信息分析师，擅长快速理解和整理技术/商业新闻。
            
            对每篇文章，请提供：
            1. 中文摘要（50字以内，核心信息密度最高）
            2. 分类（技术动态 / 产品发布 / 行业政策 / 竞品分析 / 投融资 / 其他）
            3. 重要性评分（1-10，10 为必须关注）
            4. 3-5 个关键词
            5. 情感倾向（正面 / 负面 / 中性）
            
            JSON 格式返回：
            {
              "articles": [
                {
                  "id": "文章ID",
                  "summary": "AI摘要",
                  "category": "分类",
                  "importance": 评分数字,
                  "keywords": ["关键词1", "关键词2"],
                  "sentiment": "正面/负面/中性"
                }
              ]
            }
            """;
    }
    
    private String buildBatchPrompt(List<NewsArticle> batch) {
        StringBuilder sb = new StringBuilder();
        sb.append("请分析以下 ").append(batch.size()).append(" 篇文章：\n\n");
        
        for (int i = 0; i < batch.size(); i++) {
            NewsArticle article = batch.get(i);
            sb.append("--- Article ").append(i + 1).append(" ---\n");
            sb.append("ID: ").append(article.getId()).append("\n");
            sb.append("标题: ").append(article.getTitle()).append("\n");
            sb.append("来源: ").append(article.getSource()).append("\n");
            if (article.getSummary() != null && !article.getSummary().isEmpty()) {
                sb.append("原始摘要: ").append(article.getSummary()).append("\n");
            }
            if (article.getContent() != null && !article.getContent().isEmpty()) {
                // 限制内容长度
                String content = article.getContent().length() > 500 
                        ? article.getContent().substring(0, 500) + "..." 
                        : article.getContent();
                sb.append("正文预览: ").append(content).append("\n");
            }
            sb.append("\n");
        }
        
        return sb.toString();
    }
    
    private List<NewsArticle> parseAIResult(String jsonContent, List<NewsArticle> originalBatch) {
        try {
            ObjectMapper mapper = new ObjectMapper();
            JsonNode root = mapper.readTree(jsonContent);
            JsonNode articlesNode = root.get("articles");
            
            // 建立 ID → 原始文章映射
            Map<String, NewsArticle> articleMap = originalBatch.stream()
                    .collect(Collectors.toMap(NewsArticle::getId, Function.identity()));
            
            List<NewsArticle> processed = new ArrayList<>();
            
            for (JsonNode node : articlesNode) {
                String id = node.get("id").asText();
                NewsArticle article = articleMap.get(id);
                if (article == null) continue;
                
                article.setAiSummary(node.get("summary").asText());
                article.setAiCategory(node.get("category").asText());
                article.setImportanceScore(node.get("importance").asDouble());
                article.setSentiment(node.get("sentiment").asText());
                
                List<String> keywords = new ArrayList<>();
                node.get("keywords").forEach(k -> keywords.add(k.asText()));
                article.setAiKeywords(keywords);
                
                processed.add(article);
            }
            
            return processed;
        } catch (Exception e) {
            log.error("Failed to parse AI result", e);
            return originalBatch;
        }
    }
    
    private <T> List<List<T>> partition(List<T> list, int size) {
        List<List<T>> partitions = new ArrayList<>();
        for (int i = 0; i < list.size(); i += size) {
            partitions.add(list.subList(i, Math.min(i + size, list.size())));
        }
        return partitions;
    }
}
```

---

## 五、Stage 3：日报生成与多渠道推送

### 5.1 日报编排器

```java
@Service
public class DailyReportGenerator {
    
    public String generateMarkdownReport(List<NewsArticle> articles, LocalDate date) {
        StringBuilder report = new StringBuilder();
        
        // ===== 日报头部 =====
        report.append("# 📰 行业日报\n");
        report.append("**日期**: ").append(date.format(DateTimeFormatter.ofPattern("yyyy年MM月dd日")))
                .append(" | **生成时间**: ")
                .append(LocalDateTime.now().format(DateTimeFormatter.ofPattern("HH:mm")))
                .append("\n\n");
        
        // ===== 要点速览 =====
        report.append("## 📌 今日要点\n\n");
        List<NewsArticle> topNews = articles.stream()
                .filter(a -> a.getImportanceScore() >= 7)
                .sorted(Comparator.comparingDouble(NewsArticle::getImportanceScore).reversed())
                .limit(5)
                .collect(Collectors.toList());
        
        for (int i = 0; i < topNews.size(); i++) {
            NewsArticle article = topNews.get(i);
            report.append(i + 1).append(". **").append(article.getTitle()).append("**");
            if (article.getAiSummary() != null) {
                report.append(" —— ").append(article.getAiSummary());
            }
            report.append("\n");
        }
        report.append("\n---\n\n");
        
        // ===== 按分类展示 =====
        Map<String, List<NewsArticle>> byCategory = articles.stream()
                .collect(Collectors.groupingBy(
                        a -> a.getAiCategory() != null ? a.getAiCategory() : "未分类",
                        LinkedHashMap::new,
                        Collectors.toList()));
        
        // 排序：重要性高的分类排在前面
        List<Map.Entry<String, List<NewsArticle>>> sortedCategories = byCategory.entrySet()
                .stream()
                .sorted((a, b) -> {
                    double avgA = a.getValue().stream()
                            .mapToDouble(NewsArticle::getImportanceScore).average().orElse(0);
                    double avgB = b.getValue().stream()
                            .mapToDouble(NewsArticle::getImportanceScore).average().orElse(0);
                    return Double.compare(avgB, avgA);
                })
                .collect(Collectors.toList());
        
        for (Map.Entry<String, List<NewsArticle>> entry : sortedCategories) {
            String category = entry.getKey();
            List<NewsArticle> categoryArticles = entry.getValue();
            
            report.append("## ").append(getCategoryEmoji(category)).append(" ")
                    .append(category).append(" (").append(categoryArticles.size()).append("条)\n\n");
            
            // 按重要性排序
            categoryArticles.sort(Comparator.comparingDouble(NewsArticle::getImportanceScore).reversed());
            
            for (NewsArticle article : categoryArticles) {
                report.append("### 🔹 ");
                if (article.getUrl() != null) {
                    report.append("[").append(article.getTitle()).append("](")
                            .append(article.getUrl()).append(")");
                } else {
                    report.append(article.getTitle());
                }
                report.append("\n");
                
                if (article.getAiSummary() != null) {
                    report.append("> ").append(article.getAiSummary()).append("\n");
                }
                report.append("- 来源: ").append(article.getSource());
                report.append(" | 重要性: ");
                report.append("⭐".repeat(Math.min(10, (int) article.getImportanceScore())));
                report.append(" | 情感: ").append(article.getSentiment());
                if (article.getAiKeywords() != null && !article.getAiKeywords().isEmpty()) {
                    report.append("\n- 关键词: ");
                    report.append(article.getAiKeywords().stream()
                            .map(k -> "`" + k + "`")
                            .collect(Collectors.joining(" ")));
                }
                report.append("\n\n");
            }
        }
        
        // ===== 统计信息 =====
        report.append("---\n\n");
        report.append("## 📊 今日统计\n\n");
        report.append("| 指标 | 数值 |\n");
        report.append("| --- | --- |\n");
        report.append("| 总抓取文章 | ").append(articles.size()).append(" |\n");
        report.append("| 高重要性文章(≥7分) | ")
                .append(articles.stream().filter(a -> a.getImportanceScore() >= 7).count())
                .append(" |\n");
        report.append("| 正面/负面/中性 | ")
                .append(countBySentiment(articles, "正面")).append(" / ")
                .append(countBySentiment(articles, "负面")).append(" / ")
                .append(countBySentiment(articles, "中性")).append(" |\n");
        report.append("| 信息源数量 | ")
                .append(articles.stream().map(NewsArticle::getSource).distinct().count())
                .append(" |\n");
        
        report.append("\n> 🤖 本日报由 AI News Agent 自动生成，内容仅供参考。\n");
        report.append("> 📧 如需调整推送内容，请联系管理员。\n");
        
        return report.toString();
    }
    
    private String getCategoryEmoji(String category) {
        return switch (category) {
            case "技术动态" -> "💻";
            case "产品发布" -> "🚀";
            case "行业政策" -> "📋";
            case "竞品分析" -> "🔍";
            case "投融资" -> "💰";
            default -> "📄";
        };
    }
    
    private long countBySentiment(List<NewsArticle> articles, String sentiment) {
        return articles.stream()
                .filter(a -> sentiment.equals(a.getSentiment()))
                .count();
    }
}
```

### 5.2 多渠道推送器

```java
@Component
@Slf4j
public class ReportPublisher {
    
    private final OkHttpClient httpClient;
    private final ObjectMapper objectMapper;
    
    public ReportPublisher(ObjectMapper objectMapper) {
        this.httpClient = new OkHttpClient();
        this.objectMapper = objectMapper;
    }
    
    @Value("${news-agent.push.feishu.webhook:}")
    private String feishuWebhook;
    
    @Value("${news-agent.push.dingtalk.webhook:}")
    private String dingtalkWebhook;
    
    @Value("${news-agent.push.email.smtp-host:}")
    private String smtpHost;
    
    /**
     * 推送到所有已配置的渠道
     */
    public void publishAll(String reportMarkdown, List<String> recipients) {
        List<CompletableFuture<PushResult>> futures = new ArrayList<>();
        
        if (isConfigured(feishuWebhook)) {
            futures.add(CompletableFuture.supplyAsync(() -> 
                    pushToFeishu(reportMarkdown)));
        }
        
        if (isConfigured(dingtalkWebhook)) {
            futures.add(CompletableFuture.supplyAsync(() -> 
                    pushToDingTalk(reportMarkdown)));
        }
        
        if (isConfigured(smtpHost) && recipients != null && !recipients.isEmpty()) {
            futures.add(CompletableFuture.supplyAsync(() -> 
                    pushToEmail(reportMarkdown, recipients)));
        }
        
        // 等待所有推送完成
        List<PushResult> results = futures.stream()
                .map(CompletableFuture::join)
                .collect(Collectors.toList());
        
        log.info("Report published to {} channels", results.size());
        results.forEach(r -> log.info("  {}: {}", r.getChannel(), r.isSuccess() ? "OK" : "FAIL"));
    }
    
    /**
     * 飞书机器人推送（支持 Markdown）
     */
    private PushResult pushToFeishu(String markdown) {
        try {
            // 飞书消息卡片格式
            Map<String, Object> card = new LinkedHashMap<>();
            card.put("msg_type", "interactive");
            
            Map<String, Object> cardContent = new LinkedHashMap<>();
            cardContent.put("config", Map.of("wide_screen_mode", true));
            cardContent.put("header", Map.of(
                    "template", "blue",
                    "title", Map.of("content", "📰 " + 
                            LocalDate.now().format(DateTimeFormatter.ofPattern("MM月dd日")) 
                            + " 行业日报", "tag", "plain_text")));
            
            // Markdown 内容太大时分段发送
            List<String> chunks = splitMarkdown(markdown, 8000);
            
            for (int i = 0; i < chunks.size(); i++) {
                String chunk = chunks.get(i);
                String title = i == 0 ? "📰 今日要闻" : "📄 续 (" + (i + 1) + "/" + chunks.size() + ")";
                
                cardContent.put("elements", List.of(Map.of(
                        "tag", "markdown",
                        "content", title + "\n\n" + chunk)));
                
                card.put("card", cardContent);
                
                Request request = new Request.Builder()
                        .url(feishuWebhook)
                        .post(RequestBody.create(
                                objectMapper.writeValueAsString(card),
                                MediaType.parse("application/json")))
                        .build();
                
                try (Response response = httpClient.newCall(request).execute()) {
                    if (!response.isSuccessful()) {
                        log.warn("Feishu push failed: {}", response.body().string());
                        return new PushResult("飞书", false);
                    }
                }
            }
            
            return new PushResult("飞书", true);
        } catch (Exception e) {
            log.error("Failed to push to Feishu", e);
            return new PushResult("飞书", false);
        }
    }
    
    /**
     * 钉钉机器人推送
     */
    private PushResult pushToDingTalk(String markdown) {
        try {
            Map<String, Object> message = new LinkedHashMap<>();
            message.put("msgtype", "markdown");
            
            Map<String, String> markdownContent = new LinkedHashMap<>();
            markdownContent.put("title", "行业日报 - " + 
                    LocalDate.now().format(DateTimeFormatter.ofPattern("MM月dd日")));
            markdownContent.put("text", markdown);
            message.put("markdown", markdownContent);
            
            Request request = new Request.Builder()
                    .url(dingtalkWebhook)
                    .post(RequestBody.create(
                            objectMapper.writeValueAsString(message),
                            MediaType.parse("application/json")))
                    .build();
            
            try (Response response = httpClient.newCall(request).execute()) {
                boolean success = response.isSuccessful();
                if (!success) {
                    log.warn("DingTalk push failed: {}", response.body().string());
                }
                return new PushResult("钉钉", success);
            }
        } catch (Exception e) {
            log.error("Failed to push to DingTalk", e);
            return new PushResult("钉钉", false);
        }
    }
    
    /**
     * 邮件推送
     */
    private PushResult pushToEmail(String markdown, List<String> recipients) {
        try {
            Properties props = new Properties();
            props.put("mail.smtp.host", smtpHost);
            props.put("mail.smtp.auth", "true");
            
            Session session = Session.getInstance(props);
            
            MimeMessage message = new MimeMessage(session);
            message.setFrom(new InternetAddress("news-agent@your-company.com"));
            message.setSubject("📰 行业日报 - " + 
                    LocalDate.now().format(DateTimeFormatter.ofPattern("yyyy年MM月dd日")));
            
            // HTML 格式邮件（将 Markdown 渲染为 HTML）
            String htmlBody = convertMarkdownToHtml(markdown);
            message.setContent(htmlBody, "text/html; charset=utf-8");
            
            // 添加收件人
            for (String recipient : recipients) {
                message.addRecipient(Message.RecipientType.TO, 
                        new InternetAddress(recipient));
            }
            
            Transport.send(message);
            return new PushResult("邮件", true);
        } catch (Exception e) {
            log.error("Failed to push to Email", e);
            return new PushResult("邮件", false);
        }
    }
    
    private boolean isConfigured(String value) {
        return value != null && !value.isEmpty() && !value.startsWith("${");
    }
    
    private List<String> splitMarkdown(String markdown, int maxChars) {
        List<String> chunks = new ArrayList<>();
        if (markdown.length() <= maxChars) {
            chunks.add(markdown);
            return chunks;
        }
        
        String[] sections = markdown.split("\n## ");
        StringBuilder current = new StringBuilder();
        
        for (String section : sections) {
            if (current.length() + section.length() > maxChars) {
                chunks.add(current.toString());
                current = new StringBuilder();
            }
            current.append(current.isEmpty() ? "" : "\n## ").append(section);
        }
        
        if (!current.isEmpty()) {
            chunks.add(current.toString());
        }
        
        return chunks;
    }
    
    private String convertMarkdownToHtml(String markdown) {
        // 简化实现：使用 Flexmark 或 Wrapl 等库更好
        return markdown
                .replaceAll("^# (.+)$", "<h1>$1</h1>")
                .replaceAll("(?m)^## (.+)$", "<h2>$1</h2>")
                .replaceAll("(?m)^### (.+)$", "<h3>$1</h3>")
                .replaceAll("\\*\\*(.+?)\\*\\*", "<strong>$1</strong>")
                .replaceAll("`(.+?)`", "<code>$1</code>")
                .replaceAll("\\[(.+?)\\]\\((.+?)\\)", "<a href='$2'>$1</a>")
                .replaceAll("(?m)^> (.+)$", "<blockquote>$1</blockquote>")
                .replaceAll("(?m)^- (.+)$", "<li>$1</li>")
                .replaceAll("\n---\n", "<hr>");
    }
    
    @AllArgsConstructor
    @Data
    private static class PushResult {
        private String channel;
        private boolean success;
    }
}
```

---

## 六、Quartz 调度与主流程编排

```java
@Configuration
@EnableScheduling
public class ScheduleConfig {
    
    /**
     * 每天早上 6:00 执行日报生成任务
     */
    @Bean
    public JobDetail dailyReportJob() {
        return JobBuilder.newJob(DailyReportJob.class)
                .withIdentity("dailyReportJob")
                .storeDurably()
                .build();
    }
    
    @Bean
    public Trigger dailyReportTrigger() {
        return TriggerBuilder.newTrigger()
                .forJob(dailyReportJob())
                .withIdentity("dailyReportTrigger")
                .withSchedule(CronScheduleBuilder.dailyAtHourAndMinute(6, 0))
                .build();
    }
}

@Component
@Slf4j
public class DailyReportJob implements Job {
    
    private final NewsService newsService;
    
    public DailyReportJob(NewsService newsService) {
        this.newsService = newsService;
    }
    
    @Override
    public void execute(JobExecutionContext context) {
        log.info("=== Daily Report Job Started at {} ===", LocalDateTime.now());
        
        try {
            newsService.generateAndPublishDailyReport();
            log.info("=== Daily Report Job Completed ===");
        } catch (Exception e) {
            log.error("Daily Report Job failed", e);
            // 通过告警通道通知管理员
        }
    }
}

@Service
@Slf4j
public class NewsService {
    
    private final List<NewsSource> newsSources;
    private final AINewsProcessor aiProcessor;
    private final DailyReportGenerator reportGenerator;
    private final ReportPublisher publisher;
    private final ArticleDeduplicator deduplicator;
    
    public NewsService(List<NewsSource> newsSources,
                       AINewsProcessor aiProcessor,
                       DailyReportGenerator reportGenerator,
                       ReportPublisher publisher,
                       ArticleDeduplicator deduplicator) {
        this.newsSources = newsSources;
        this.aiProcessor = aiProcessor;
        this.reportGenerator = reportGenerator;
        this.publisher = publisher;
        this.deduplicator = deduplicator;
    }
    
    /**
     * 完整的日报生成流程
     */
    public String generateAndPublishDailyReport() {
        log.info("Starting daily report generation...");
        long startTime = System.currentTimeMillis();
        
        // Step 1: 并行采集所有信息源
        log.info("Step 1: Fetching news from {} sources...", newsSources.size());
        List<NewsArticle> allArticles = fetchFromAllSources();
        log.info("  Fetched {} raw articles", allArticles.size());
        
        // Step 2: 去重合并
        log.info("Step 2: Deduplicating...");
        List<NewsArticle> uniqueArticles = deduplicator.deduplicate(allArticles);
        log.info("  {} articles after dedup", uniqueArticles.size());
        
        // Step 3: AI 处理
        log.info("Step 3: AI processing...");
        List<NewsArticle> processedArticles = aiProcessor.process(uniqueArticles);
        log.info("  Processing complete");
        
        // Step 4: 生成 Markdown 日报
        log.info("Step 4: Generating report...");
        String report = reportGenerator.generateMarkdownReport(
                processedArticles, LocalDate.now());
        
        // Step 5: 推送
        log.info("Step 5: Publishing...");
        publisher.publishAll(report, getDefaultRecipients());
        
        long totalTime = System.currentTimeMillis() - startTime;
        log.info("Daily report generation completed in {}ms", totalTime);
        
        return report;
    }
    
    private List<NewsArticle> fetchFromAllSources() {
        List<CompletableFuture<List<NewsArticle>>> futures = newsSources.stream()
                .filter(NewsSource::isEnabled)
                .map(source -> CompletableFuture.supplyAsync(() -> {
                    log.debug("Fetching from: {}", source.getName());
                    return source.fetch(24); // 最近24小时
                }))
                .collect(Collectors.toList());
        
        return futures.stream()
                .map(CompletableFuture::join)
                .flatMap(List::stream)
                .collect(Collectors.toList());
    }
    
    private List<String> getDefaultRecipients() {
        // 从配置文件或数据库读取默认接收人
        return List.of("team@your-company.com");
    }
}

@Component
public class ArticleDeduplicator {
    
    private final Set<String> seenIds = ConcurrentHashMap.newKeySet();
    
    public List<NewsArticle> deduplicate(List<NewsArticle> articles) {
        return articles.stream()
                .filter(a -> {
                    // 基于 URL 或标题的相似度去重
                    if (a.getId() != null && seenIds.add(a.getId())) {
                        return true;
                    }
                    // 基于标题前 50 个字符的模糊去重
                    if (a.getTitle() != null && a.getTitle().length() > 20) {
                        String titleHash = DigestUtils.md5Hex(
                                a.getTitle().substring(0, Math.min(50, a.getTitle().length())));
                        return seenIds.add(titleHash);
                    }
                    return false;
                })
                .collect(Collectors.toList());
    }
}
```

---

## 七、配置文件

```yaml
news-agent:
  sources:
    rss:
      urls: >
        https://rsshub.app/36kr/motif/1,
        https://www.infoq.cn/feed,
        https://juejin.cn/rss
  llm:
    model: gpt-4-turbo
    batch-size: 10
  push:
    feishu:
      webhook: https://open.feishu.cn/open-apis/bot/v2/hook/xxx
    dingtalk:
      webhook: https://oapi.dingtalk.com/robot/send?access_token=xxx
    email:
      smtp-host: smtp.your-company.com
      recipients: team@your-company.com,manager@your-company.com
  firewall:
    max-rows: 1000
    max-execution-time-ms: 30000
```

---

## 八、总结与优化方向

**这个定时任务 Agent 的核心能力**：

1. **多渠道信息采集**：RSS + 网页爬虫 + GitHub Trending + 可扩展自定义源
2. **AI 智能处理**：自动摘要、分类、重要性评分、情感分析
3. **全链路自动化**：定时触发 → 采集 → 处理 → 日报生成 → 多渠道推送

**进阶优化方向**：

- **记忆能力**：记住用户阅读偏好，个性化推荐
- **溯源能力**：支持回看历史日报，生成周报/月报
- **交互能力**：用户可以在飞书群 @机器人 提问（"昨天关于 AI 的文章有哪些？"）
- **多语言支持**：自动翻译外文新闻源

---

**下篇预告**：《LangGraph 实战：构建有状态的、可持久化的 Agent 工作流》——LangGraph 是什么？如何在 Java 中实现 StateGraph + Checkpoint + 条件分支？即使你不懂 Python，也能用 Java 构建有记忆的 Agent 工作流！
