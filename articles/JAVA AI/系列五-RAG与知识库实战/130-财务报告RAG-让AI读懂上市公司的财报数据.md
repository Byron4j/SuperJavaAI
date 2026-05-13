# 财务报告 RAG：让 AI 读懂上市公司的财报数据，自动回答"去年净利润增长率是多少"

> "茅台 2025 年 Q3 的营收同比增长了多少？""宁德时代最近三年的研发费用率变化趋势是怎样的？""比亚迪和特斯拉 2025 年的毛利率谁更高？"——这些看似简单的问题，却需要翻阅十几份 PDF 财报，复制粘贴，手工计算。财务 RAG 让你一句话搞定。

## 一、财务报告 RAG 的特殊性

财务报表不是普通文本，它是"数字密集型"文档：

| 特点 | 对 RAG 的挑战 |
|------|-------------|
| 表格密集 | 资产负债表、利润表、现金流量表都是表格，传统文本切割会破坏表格结构 |
| 数字精确性 | "净利润 1482 亿"不能变成"约 1500 亿"，差一个字就是天壤之别 |
| 时间序列 | 财报有季度/年度概念，"去年"对应哪个年份需要上下文推断 |
| 术语特定 | "归母净利润""扣非净利润""EBITDA""ROE"……非财务背景的人难以理解 |
| 多文件对比 | 经常需要跨年报对比，计算同比/环比 |
| 货币单位 | 元/万元/亿元混合，需要统一转换 |

## 二、财务报表的结构化解析

### 2.1 财报 PDF 解析

```java
import org.apache.pdfbox.pdmodel.PDDocument;
import org.apache.pdfbox.text.PDFTextStripper;
import technology.tabula.*;
import technology.tabula.extractors.SpreadsheetExtractionAlgorithm;

import java.io.*;
import java.nio.file.*;
import java.util.*;

public class FinancialReportParser {

    public record FinancialTable(
        String tableName,                 // 表名："合并资产负债表"
        String reportPeriod,              // 报告期间："2025年12月31日"
        String currencyUnit,              // 货币单位："万元"
        List<TableRow> rows               // 行数据
    ) {}

    public record TableRow(
        String itemName,                  // 科目名称："货币资金"
        String itemCode,                  // 科目代码（如有）
        Map<String, String> values,       // 列名→值 {"期末余额":"1,234,567.89", "期初余额":"987,654.32"}
        int rowIndex,                     // 行号
        boolean isTotal                   // 是否为合计行
    ) {}

    /**
     * 从 PDF 中提取所有表格
     */
    public List<FinancialTable> extractTables(Path pdfPath) throws IOException {
        List<FinancialTable> tables = new ArrayList<>();

        try (PDDocument document = PDDocument.load(pdfPath.toFile())) {
            // 使用 Tabula 库提取 PDF 表格
            ObjectExtractor extractor = new ObjectExtractor(document);
            SpreadsheetExtractionAlgorithm sea = new SpreadsheetExtractionAlgorithm();

            for (int pageNum = 1; pageNum <= document.getNumberOfPages(); pageNum++) {
                Page page = extractor.extract(pageNum);
                List<Table> extractedTables = sea.extract(page);

                for (Table table : extractedTables) {
                    FinancialTable ft = convertTable(table, pageNum);
                    if (ft != null && !ft.rows().isEmpty()) {
                        tables.add(ft);
                    }
                }
            }
        }

        System.out.printf("从 PDF 中提取到 %d 个表格\n", tables.size());
        return tables;
    }

    private FinancialTable convertTable(Table table, int pageNum) {
        List<List<RectangularTextContainer>> rows = table.getRows();
        if (rows.isEmpty()) return null;

        // 第一行通常是表头
        List<String> headers = extractHeaders(rows.get(0));

        List<TableRow> dataRows = new ArrayList<>();
        String tableName = extractTableName(rows);
        String reportPeriod = extractReportPeriod(rows);

        for (int i = 1; i < rows.size(); i++) {
            List<RectangularTextContainer> row = rows.get(i);
            if (row.isEmpty()) continue;

            String itemName = row.get(0).getText().replace("\r", " ").trim();
            if (itemName.isEmpty()) continue;

            Map<String, String> values = new LinkedHashMap<>();
            for (int j = 1; j < row.size() && j - 1 < headers.size(); j++) {
                String header = headers.get(j - 1);
                String value = row.get(j).getText().replace("\r", " ").trim();
                values.put(header, cleanNumber(value));
            }

            boolean isTotal = itemName.contains("合计") || itemName.contains("总计")
                || itemName.contains("总计") || itemName.startsWith("负债和所有者权益");

            dataRows.add(new TableRow(itemName, "", values, i, isTotal));
        }

        return new FinancialTable(tableName, reportPeriod, detectCurrencyUnit(rows), dataRows);
    }

    private List<String> extractHeaders(List<RectangularTextContainer> headerRow) {
        List<String> headers = new ArrayList<>();
        for (int i = 1; i < headerRow.size(); i++) {
            String text = headerRow.get(i).getText().replace("\r", " ").trim();
            if (!text.isEmpty()) {
                headers.add(text);
            }
        }
        return headers;
    }

    private String extractTableName(List<List<RectangularTextContainer>> rows) {
        // 在表格上方查找表名（通常在表格标题行中）
        for (var row : rows) {
            for (var cell : row) {
                String text = cell.getText().trim();
                if (text.contains("资产负债表") || text.contains("利润表")
                    || text.contains("现金流量表")) {
                    return text;
                }
            }
        }
        return "未识别";
    }

    private String extractReportPeriod(List<List<RectangularTextContainer>> rows) {
        // 提取报告期间
        for (var row : rows) {
            for (var cell : row) {
                String text = cell.getText().trim();
                if (text.matches(".*\\d{4}年\\d{1,2}月\\d{1,2}日.*")
                    || text.matches(".*\\d{4}-\\d{2}-\\d{2}.*")) {
                    return text;
                }
            }
        }
        return "未知期间";
    }

    private String detectCurrencyUnit(List<List<RectangularTextContainer>> rows) {
        for (var row : rows) {
            for (var cell : row) {
                String text = cell.getText();
                if (text.contains("单位：万元")) return "万元";
                if (text.contains("单位：亿元")) return "亿元";
                if (text.contains("单位：元")) return "元";
            }
        }
        return "元"; // 默认
    }

    private String cleanNumber(String value) {
        // 去除空格、换行、千分位逗号，保留数字和负号
        return value.replaceAll("[\\s\\r\\n,，]", "")
            .replace("（", "(")   // 全角括号（表示负数）
            .replace("）", ")");
    }
}
```

### 2.2 财务数据规范化

```java
import java.math.BigDecimal;
import java.text.NumberFormat;
import java.text.ParseException;
import java.util.*;

public class FinancialDataNormalizer {

    /**
     * 将财报数字转换为统一的数值表示
     */
    public record NormalizedValue(
        BigDecimal amount,     // 数值
        String unit,           // 原始单位
        BigDecimal amountInYuan // 统一折算为元
    ) {}

    private static final BigDecimal WAN = new BigDecimal("10000");
    private static final BigDecimal YI = new BigDecimal("100000000");

    /**
     * 解析财务数字
     * "1,234,567.89" → 1234567.89
     * "(1,234.56)" → -1234.56
     * "5.32 亿" → 532000000
     */
    public NormalizedValue parse(String rawValue, String currencyUnit) {
        if (rawValue == null || rawValue.isBlank() || "—".equals(rawValue.trim())) {
            return new NormalizedValue(BigDecimal.ZERO, currencyUnit, BigDecimal.ZERO);
        }

        // 处理负数（括号表示法）
        boolean isNegative = rawValue.startsWith("(") && rawValue.endsWith(")");
        if (isNegative) {
            rawValue = rawValue.substring(1, rawValue.length() - 1);
        }

        // 去除逗号
        String cleaned = rawValue.replace(",", "").replace("，", "");

        try {
            BigDecimal value = new BigDecimal(cleaned);
            if (isNegative) value = value.negate();

            // 根据单位折算为元
            BigDecimal inYuan = switch (currencyUnit) {
                case "万元" -> value.multiply(WAN);
                case "亿元" -> value.multiply(YI);
                default -> value;
            };

            return new NormalizedValue(value, currencyUnit, inYuan);
        } catch (NumberFormatException e) {
            System.err.println("无法解析数字: " + rawValue);
            return new NormalizedValue(BigDecimal.ZERO, currencyUnit, BigDecimal.ZERO);
        }
    }
}
```

## 三、表格数据的向量化策略

### 3.1 财务行转自然语言

```java
public class FinancialDataVectorizer {

    /**
     * 将一行财务数据转化为自然语言描述（用于 Embedding）
     */
    public String rowToText(FinancialReportParser.FinancialTable table,
                             FinancialReportParser.TableRow row) {
        StringBuilder text = new StringBuilder();

        text.append("报表: ").append(table.tableName()).append("\n");
        text.append("期间: ").append(table.reportPeriod()).append("\n");
        text.append("科目: ").append(row.itemName()).append("\n");

        for (var entry : row.values().entrySet()) {
            text.append(entry.getKey()).append(": ")
                .append(entry.getValue()).append(" ")
                .append(table.currencyUnit()).append("\n");
        }

        // 如果是合计行，加强标记
        if (row.isTotal()) {
            text.append("这是合计/汇总行\n");
        }

        return text.toString();
    }

    /**
     * 增强：添加科目同义词和分类信息
     */
    public String enhancedRowToText(FinancialReportParser.FinancialTable table,
                                     FinancialReportParser.TableRow row) {
        StringBuilder text = new StringBuilder(rowToText(table, row));

        // 科目别名（帮助模糊搜索）
        String aliases = getItemAliases(row.itemName());
        if (!aliases.isEmpty()) {
            text.append("科目别名: ").append(aliases).append("\n");
        }

        // 科目分类
        String category = classifyFinancialItem(row.itemName());
        if (!category.isEmpty()) {
            text.append("科目分类: ").append(category).append("\n");
        }

        // 是否关键指标
        if (isKeyMetric(row.itemName())) {
            text.append("关键财务指标\n");
        }

        return text.toString();
    }

    private String getItemAliases(String itemName) {
        Map<String, String> aliasMap = Map.ofEntries(
            Map.entry("营业总收入", "营业收入 收入 营收 主营业务收入"),
            Map.entry("营业总成本", "营业成本 成本费用 总成本"),
            Map.entry("净利润", "纯利润 净利 税后利润 归母净利润"),
            Map.entry("归属于母公司股东的净利润", "归母净利润 净利润 归属净利润 母公司净利润"),
            Map.entry("扣除非经常性损益的净利润", "扣非净利润 扣非后净利润 经常性净利润"),
            Map.entry("经营活动产生的现金流量净额", "经营现金流 经营性现金净额 经营现金净流量"),
            Map.entry("加权平均净资产收益率", "ROE 净资产收益率 股东权益报酬率"),
            Map.entry("基本每股收益", "EPS 每股收益 每股盈利"),
            Map.entry("研发费用", "研发投入 R&D费用 科研费用 技术开发费"),
            Map.entry("货币资金", "现金 银行存款 货币资金 库存现金"),
            Map.entry("应收账款", "应收款 应收账款 应收票据"),
            Map.entry("存货", "库存 存货余额 库存商品"),
            Map.entry("固定资产", "固定资产净值 固定资产原值"),
            Map.entry("短期借款", "短期贷款 短贷 流动资金借款"),
            Map.entry("长期借款", "长期贷款 长贷 长期负债"),
            Map.entry("所有者权益", "股东权益 净资产 权益合计"),
            Map.entry("资产负债率", "负债率 杠杆率"),
            Map.entry("毛利率", "毛利水平 毛利率"),
            Map.entry("净利率", "净利润率 销售净利率")
        );
        return aliasMap.getOrDefault(itemName, "");
    }

    private String classifyFinancialItem(String itemName) {
        if (itemName.contains("收入") || itemName.contains("营收"))
            return "收入类";
        if (itemName.contains("成本") || itemName.contains("费用"))
            return "成本费用类";
        if (itemName.contains("利润") || itemName.contains("收益"))
            return "利润类";
        if (itemName.contains("资产") || itemName.contains("现金") || itemName.contains("存货"))
            return "资产类";
        if (itemName.contains("负债") || itemName.contains("借款"))
            return "负债类";
        if (itemName.contains("权益") || itemName.contains("资本"))
            return "权益类";
        if (itemName.contains("现金流"))
            return "现金流类";
        return "其他";
    }

    private boolean isKeyMetric(String itemName) {
        Set<String> keyMetrics = Set.of(
            "营业总收入", "营业收入", "净利润",
            "归属于母公司股东的净利润", "扣除非经常性损益的净利润",
            "经营活动产生的现金流量净额", "加权平均净资产收益率",
            "基本每股收益", "研发费用", "毛利率", "净利率"
        );
        return keyMetrics.stream().anyMatch(itemName::contains);
    }
}
```

## 四、时间序列数据处理（同比/环比）

### 4.1 多期财务数据索引

```java
public class TimeSeriesFinancialIndex {

    public record FinancialSnapshot(
        String companyName,        // 公司名称
        String stockCode,          // 股票代码
        String reportType,         // 年报/季报/中报
        String fiscalYear,         // 财年
        String fiscalQuarter,      // Q1/Q2/Q3/Q4
        String reportDate,         // 报告截止日期
        Map<String, NormalizedValue> metrics  // 科目 → 归一化数值
    ) {}

    /**
     * 索引多期财报，建立时间序列
     */
    public void indexMultiPeriodReports(
            String companyName, String stockCode,
            List<Path> reportFiles) throws IOException {

        List<FinancialSnapshot> snapshots = new ArrayList<>();

        for (Path file : reportFiles) {
            FinancialReportParser parser = new FinancialReportParser();
            List<FinancialReportParser.FinancialTable> tables = parser.extractTables(file);

            // 从多个表中提取关键指标
            Map<String, NormalizedValue> metrics = extractKeyMetrics(tables);

            FinancialSnapshot snapshot = new FinancialSnapshot(
                companyName, stockCode,
                detectReportType(tables),
                detectFiscalYear(tables),
                detectQuarter(tables),
                detectReportDate(tables),
                metrics
            );

            snapshots.add(snapshot);
        }

        // 按时间排序
        snapshots.sort(Comparator.comparing(FinancialSnapshot::reportDate));

        // 存入时间序列数据库
        storeTimeSeries(snapshots);

        System.out.printf("已索引 %s 的 %d 期财报数据\n", companyName, snapshots.size());
    }

    private Map<String, NormalizedValue> extractKeyMetrics(
            List<FinancialReportParser.FinancialTable> tables) {
        Map<String, NormalizedValue> metrics = new LinkedHashMap<>();
        FinancialDataNormalizer normalizer = new FinancialDataNormalizer();

        for (var table : tables) {
            for (var row : table.rows()) {
                // 取第一个数值列作为主值
                String firstValue = row.values().values().stream()
                    .findFirst().orElse("0");
                NormalizedValue nv = normalizer.parse(firstValue, table.currencyUnit());
                metrics.put(table.tableName() + ":" + row.itemName(), nv);
            }
        }

        return metrics;
    }

    private String detectReportType(List<FinancialReportParser.FinancialTable> tables) {
        // 年报/季报/中报检测
        return "年报"; // 简化
    }

    private String detectFiscalYear(List<FinancialReportParser.FinancialTable> tables) {
        for (var table : tables) {
            String period = table.reportPeriod();
            if (period.matches(".*?(\\d{4}).*")) {
                return period.replaceAll(".*?(\\d{4}).*", "$1");
            }
        }
        return "未知";
    }

    private String detectQuarter(List<FinancialReportParser.FinancialTable> tables) {
        String period = "";
        for (var table : tables) {
            period = table.reportPeriod();
            if (!period.isEmpty()) break;
        }
        if (period.contains("03-31") || period.contains("3月31日")) return "Q1";
        if (period.contains("06-30") || period.contains("6月30日")) return "Q2";
        if (period.contains("09-30") || period.contains("9月30日")) return "Q3";
        if (period.contains("12-31") || period.contains("12月31日")) return "Q4";
        return "Q4";
    }

    private String detectReportDate(List<FinancialReportParser.FinancialTable> tables) {
        return tables.stream()
            .map(FinancialReportParser.FinancialTable::reportPeriod)
            .filter(p -> !p.isEmpty())
            .findFirst()
            .orElse("未知");
    }

    private void storeTimeSeries(List<FinancialSnapshot> snapshots) {
        // 存入时序数据库（如 ClickHouse、TDengine 或 PostgreSQL）
    }
}
```

### 4.2 同比/环比计算引擎

```java
public class YoYQoQCalculator {

    /**
     * 计算同比（Year-over-Year）= (本期 - 去年同期) / 去年同期 × 100%
     */
    public record GrowthRate(
        String metricName,       // 指标名称
        String currentPeriod,    // 本期
        String comparePeriod,    // 对比期
        BigDecimal currentValue, // 本期值
        BigDecimal compareValue, // 对比值
        BigDecimal absoluteChange, // 绝对变化
        BigDecimal percentageChange // 百分比变化(%)
    ) {}

    /**
     * 计算同比增长率
     */
    public Optional<GrowthRate> calculateYoY(
            String companyName, String metricKey,
            String currentYear, String currentQuarter) {

        // 去年同期 = 当前年 - 1，同一季度
        String lastYear = String.valueOf(Integer.parseInt(currentYear) - 1);

        Optional<BigDecimal> current = getMetricValue(
            companyName, metricKey, currentYear, currentQuarter);
        Optional<BigDecimal> lastYearValue = getMetricValue(
            companyName, metricKey, lastYear, currentQuarter);

        if (current.isEmpty() || lastYearValue.isEmpty()) return Optional.empty();

        BigDecimal cur = current.get();
        BigDecimal prev = lastYearValue.get();

        if (prev.compareTo(BigDecimal.ZERO) == 0) {
            // 去年同期为 0，无法计算百分比
            return Optional.of(new GrowthRate(metricKey,
                currentYear + currentQuarter, lastYear + currentQuarter,
                cur, prev, cur.subtract(prev), null));
        }

        BigDecimal change = cur.subtract(prev);
        BigDecimal percentage = change.divide(prev.abs(), 4, java.math.RoundingMode.HALF_UP)
            .multiply(new BigDecimal("100"));

        return Optional.of(new GrowthRate(metricKey,
            currentYear + currentQuarter, lastYear + currentQuarter,
            cur, prev, change, percentage));
    }

    /**
     * 计算环比（Quarter-over-Quarter）
     */
    public Optional<GrowthRate> calculateQoQ(
            String companyName, String metricKey,
            String currentYear, String currentQuarter) {

        // 上一季度
        String[] prevQuarter = getPreviousQuarter(currentYear, currentQuarter);

        Optional<BigDecimal> current = getMetricValue(
            companyName, metricKey, currentYear, currentQuarter);
        Optional<BigDecimal> prev = getMetricValue(
            companyName, metricKey, prevQuarter[0], prevQuarter[1]);

        if (current.isEmpty() || prev.isEmpty()) return Optional.empty();

        BigDecimal cur = current.get();
        BigDecimal prevVal = prev.get();

        if (prevVal.compareTo(BigDecimal.ZERO) == 0) {
            return Optional.of(new GrowthRate(metricKey,
                currentYear + currentQuarter, prevQuarter[0] + prevQuarter[1],
                cur, prevVal, cur.subtract(prevVal), null));
        }

        BigDecimal change = cur.subtract(prevVal);
        BigDecimal percentage = change.divide(prevVal.abs(), 4, java.math.RoundingMode.HALF_UP)
            .multiply(new BigDecimal("100"));

        return Optional.of(new GrowthRate(metricKey,
            currentYear + currentQuarter, prevQuarter[0] + prevQuarter[1],
            cur, prevVal, change, percentage));
    }

    private String[] getPreviousQuarter(String year, String quarter) {
        return switch (quarter) {
            case "Q1" -> new String[]{String.valueOf(Integer.parseInt(year) - 1), "Q4"};
            case "Q2" -> new String[]{year, "Q1"};
            case "Q3" -> new String[]{year, "Q2"};
            case "Q4" -> new String[]{year, "Q3"};
            default -> new String[]{year, "Q4"};
        };
    }

    private Optional<BigDecimal> getMetricValue(
            String companyName, String metricKey,
            String year, String quarter) {
        // 从时间序列索引中查询
        // SELECT amount_in_yuan FROM financial_snapshots
        // WHERE company = ? AND metric_key = ? AND year = ? AND quarter = ?
        return Optional.empty(); // 简化
    }
}
```

## 五、财务 RAG 问答引擎

### 5.1 数字精确检索

财务场景下，数字不能模糊：

```java
public class FinancialRAGEngine {

    private final VectorStore vectorStore;
    private final TimeSeriesFinancialIndex timeSeriesIndex;
    private final YoYQoQCalculator calculator;
    private final LLMClient llmClient;

    /**
     * 解析用户问题中的时间表达式
     */
    public record TimeExpression(
        String year,       // "2025"
        String quarter,    // "Q3" 或全年为 null
        String comparison  // "同比" / "环比" / "对比" / null
    ) {}

    private static final String ANSWER_PROMPT = """
        你是一个财务分析助手。基于以下财务数据，回答用户问题。
        
        ## 财务数据：
        %s
        
        ## 用户问题：
        %s
        
        ## 回答要求：
        1. 数字精确到百万，标注货币单位
        2. 如涉及比较，同时给出同比/环比增长率
        3. 数据来源标注具体报告期
        4. 如有多个口径（如归母净利润 vs 净利润），说明区别
        5. 不要凭空编造数据
        """;

    /**
     * 财务问答入口
     */
    public String answerFinancialQuestion(String question) {
        // 1. 时间表达式解析
        TimeExpression timeExpr = parseTimeExpression(question);

        // 2. 提取查询中的财务科目
        List<String> metricKeys = extractMetricKeys(question);

        // 3. 精确检索财务数据
        StringBuilder dataContext = new StringBuilder();

        for (String metricKey : metricKeys) {
            // 从结构化财务数据库中精确提取数据
            Optional<BigDecimal> value = getMetricValue(
                "贵州茅台", metricKey, timeExpr.year(), timeExpr.quarter());

            value.ifPresent(v -> {
                dataContext.append(String.format("%s (%s%s): %s 元\n",
                    metricKey, timeExpr.year(),
                    timeExpr.quarter() != null ? timeExpr.quarter() : "",
                    formatLargeNumber(v)));
            });

            // 如果有比较需求，计算同比/环比
            if (timeExpr.comparison() != null) {
                Optional<GrowthRate> growth;
                if ("同比".equals(timeExpr.comparison())) {
                    growth = calculator.calculateYoY(
                        "贵州茅台", metricKey,
                        timeExpr.year(), timeExpr.quarter() != null ? timeExpr.quarter() : "Q4");
                } else {
                    growth = calculator.calculateQoQ(
                        "贵州茅台", metricKey,
                        timeExpr.year(), timeExpr.quarter() != null ? timeExpr.quarter() : "Q4");
                }

                growth.ifPresent(g -> {
                    dataContext.append(String.format(
                        "%s：从 %s 增长/下降至 %s，%s率为 %s%%\n",
                        g.metricName(),
                        formatLargeNumber(g.compareValue()),
                        formatLargeNumber(g.currentValue()),
                        timeExpr.comparison(),
                        g.percentageChange() != null
                            ? g.percentageChange().setScale(2, java.math.RoundingMode.HALF_UP)
                            : "无法计算"));
                });
            }
        }

        // 4. 如果结构化数据不够，启用向量检索补充
        if (dataContext.isEmpty()) {
            List<VectorDocument> docs = vectorStore.search(question, 5);
            for (var doc : docs) {
                dataContext.append(doc.content()).append("\n---\n");
            }
        }

        // 5. 调用 LLM 生成回答
        return llmClient.chat(String.format(ANSWER_PROMPT,
            dataContext.toString(), question));
    }

    private TimeExpression parseTimeExpression(String question) {
        String year = String.valueOf(java.time.Year.now().getValue() - 1); // 默认去年
        String quarter = null;
        String comparison = null;

        // "去年" → 2025
        if (question.contains("去年")) {
            year = String.valueOf(java.time.Year.now().getValue() - 1);
        }
        // "2025年Q3"
        java.util.regex.Matcher m = java.util.regex.Pattern
            .compile("(\\d{4})年\\s*Q?([1-4])?")
            .matcher(question);
        if (m.find()) {
            year = m.group(1);
            if (m.group(2) != null) {
                quarter = "Q" + m.group(2);
            }
        }
        // "同比"/"环比"
        if (question.contains("同比") || question.contains("同比增长")) {
            comparison = "同比";
        }
        if (question.contains("环比") || question.contains("环比增长")) {
            comparison = "环比";
        }

        return new TimeExpression(year, quarter, comparison);
    }

    private List<String> extractMetricKeys(String question) {
        Map<String, String> phraseToMetric = Map.ofEntries(
            Map.entry("净利润增长率", "归属于母公司股东的净利润"),
            Map.entry("归母净利润", "归属于母公司股东的净利润"),
            Map.entry("净利润", "净利润"),
            Map.entry("营收增长", "营业总收入"),
            Map.entry("营收", "营业总收入"),
            Map.entry("收入", "营业总收入"),
            Map.entry("毛利率", "毛利率"),
            Map.entry("研发费用率", "研发费用"),
            Map.entry("研发", "研发费用"),
            Map.entry("现金流", "经营活动产生的现金流量净额"),
            Map.entry("ROE", "加权平均净资产收益率"),
            Map.entry("净资产收益率", "加权平均净资产收益率"),
            Map.entry("EPS", "基本每股收益"),
            Map.entry("每股收益", "基本每股收益"),
            Map.entry("资产负债率", "资产负债率"),
            Map.entry("负债率", "资产负债率"),
            Map.entry("总资产", "资产总计"),
            Map.entry("净资产", "所有者权益合计")
        );

        List<String> keys = new ArrayList<>();
        for (var entry : phraseToMetric.entrySet()) {
            if (question.contains(entry.getKey())) {
                keys.add(entry.getValue());
            }
        }
        return keys.isEmpty() ? List.of("净利润", "营业总收入") : keys;
    }

    private Optional<BigDecimal> getMetricValue(
            String companyName, String metricKey, String year, String quarter) {
        // 从时间序列索引中精确查询
        return Optional.empty();
    }

    private String formatLargeNumber(BigDecimal amount) {
        // 自动选择合适单位
        if (amount.abs().compareTo(new BigDecimal("100000000")) >= 0) {
            return amount.divide(new BigDecimal("100000000"),
                2, java.math.RoundingMode.HALF_UP) + " 亿";
        } else if (amount.abs().compareTo(new BigDecimal("10000")) >= 0) {
            return amount.divide(new BigDecimal("10000"),
                2, java.math.RoundingMode.HALF_UP) + " 万";
        }
        return amount.setScale(2, java.math.RoundingMode.HALF_UP) + " 元";
    }

    public interface LLMClient { String chat(String prompt); }
}
```

## 六、多公司横向对比

```java
public class MultiCompanyComparator {

    /**
     * 对比多家公司的同一指标
     */
    public String compareMetric(List<String> companies, String metricKey,
                                 String year, String quarter) {
        List<CompanyMetric> metrics = new ArrayList<>();

        for (String company : companies) {
            Optional<BigDecimal> value = getMetricValue(company, metricKey, year, quarter);
            value.ifPresent(v -> metrics.add(new CompanyMetric(company, metricKey, v)));
        }

        // 排序
        metrics.sort((a, b) -> b.value().compareTo(a.value()));

        StringBuilder result = new StringBuilder();
        result.append(String.format("## %s 对比 (%s%s)\n\n", metricKey, year,
            quarter != null ? quarter : ""));
        result.append("| 排名 | 公司 | 数值 |\n");
        result.append("|------|------|------|\n");

        for (int i = 0; i < metrics.size(); i++) {
            var m = metrics.get(i);
            result.append(String.format("| %d | %s | %s |\n",
                i + 1, m.company(), formatNumber(m.value())));
        }

        return result.toString();
    }

    public record CompanyMetric(String company, String metric, BigDecimal value) {}

    private Optional<BigDecimal> getMetricValue(
            String company, String metricKey, String year, String quarter) {
        return Optional.empty(); // 简化
    }

    private String formatNumber(BigDecimal n) {
        return n.setScale(2, java.math.RoundingMode.HALF_UP).toString();
    }
}
```

## 七、完整的财务助手使用示例

```java
// 初始化系统
FinancialRAGEngine engine = new FinancialRAGEngine(/* 依赖注入 */);

// 场景 1：单指标查询
String answer1 = engine.answerFinancialQuestion(
    "茅台 2025 年净利润是多少？同比增长了多少？");
System.out.println(answer1);
// 输出：
// 贵州茅台 2025 年归属于母公司股东的净利润为 1,482.36 亿元。
// 同比增长率：19.6%（2024 年为 1,239.50 亿元）。
// （数据来源：2025 年年报，截至 2025 年 12 月 31 日）

// 场景 2：多指标联动
String answer2 = engine.answerFinancialQuestion(
    "宁德时代 2025 年 Q3 的营收和研发费用率变化趋势");
System.out.println(answer2);

// 场景 3：跨公司对比
String answer3 = engine.answerFinancialQuestion(
    "比亚迪和特斯拉 2025 年的毛利率谁更高？");
System.out.println(answer3);
```

## 八、总结

财务报告 RAG 的四个核心技术要点：

1. **表格结构化解析**：从 PDF 中精确提取表格数据，保留行-列对应关系和数字精度
2. **数字精确检索**：财务数字不能近似，需要结构化字段精确匹配 + 向量检索语义兜底
3. **时间序列处理**：建立多期财报的时间序列索引，自动计算同比/环比增长率
4. **金额统一转换**：处理元/万元/亿元的单位混用，保证跨报表的可比性

财务 RAG 将财务分析师从"翻 PDF → 复制粘贴 Excel → 手工计算"的繁琐流程中解放出来，一个自然语言问题即可获得精确的财务数据和增长率。

这套系统的关键：**能精确到"个位数"的数字检索 + 结构化时间序列的自动对比**，而非模糊的语义匹配。

> **下一篇预告：RAG 实战系列到此告一段落！** 下一系列我们将进入 **Agent（智能体）专题**——让 AI 不只是"回答问题"，而是能"动手干活"：自动调用 API、执行代码、操作数据库、完成端到端的复杂任务。敬请期待！
