# MCP 协议实战：让 AI 编程助手直接操作你的数据库，说"帮我查下昨天的订单量"它就真去查

> **想象一下**：你在 Copilot/Cursor 里输入"帮我分析一下 tb_order 表中最近 7 天支付成功率下降的原因"，AI 自动查了数据库、看了代码逻辑、分析了调用链，然后给了你一份报告——**这不是科幻，MCP 协议让它正在变成现实。**

---

## 一、先看效果：一个真实的对话

在你继续往下读之前，先给你看一段真实对话，感受一下 MCP 带来的体验变化：

```
你：帮我查一下昨天（2026-05-04）tb_order 表的订单总量和支付成功的订单量。

AI：好的，我先查询数据库获取数据。

[AI 自动调用 MCP 工具: query_database]
→ SELECT COUNT(*) AS total_orders,
         SUM(CASE WHEN pay_status = 'SUCCESS' THEN 1 ELSE 0 END) AS paid_orders
  FROM tb_order
  WHERE create_time >= '2026-05-04 00:00:00'
    AND create_time < '2026-05-05 00:00:00'
[返回: total_orders=12840, paid_orders=11556]

AI：昨天共有 12840 笔订单，其中支付成功 11556 笔，支付成功率约为 90.0%。

你：成功率比前天下降了，能帮我分析一下可能的原因吗？

AI：好的，我从几个维度来分析：

1. 先看下昨天各时段的支付成功率分布……
2. 再看下昨天支付失败的错误码分布……
3. 最后看下昨天是否有系统异常或超时……

[AI 连续调用多次 MCP 工具，查询多个维度数据]
[最终输出分析报告]
```

**关键来了**：这一切发生在你的 IDE 编辑器里，AI 不再是"用嘴回答"，而是"直接动手干活"。

这就是 MCP 协议带来的改变。

---

## 二、MCP 协议：5 分钟快速入门

### 2.1 MCP 是什么？

**MCP（Model Context Protocol，模型上下文协议）** 是 Anthropic 于 2024 年底开源的一套标准化协议。它的核心目标是：**为 AI 模型与外部工具/数据源之间建立一套统一的通信标准。**

说人话就是：以前 AI 只能"聊天"，有了 MCP 之后 AI 可以"干活"——查数据库、调 API、读文件、发消息。

### 2.2 为什么 MCP 重要？

在 MCP 出现之前，如果你想给 AI 增加外部能力，通常需要：

- **每个 AI 产品单独开发插件**（Copilot 的扩展、Cursor 的扩展、ChatGPT 的插件……互不兼容）
- **厂商锁定**——你给 ChatGPT 写的工具，Claude 用不了，反之亦然
- **碎片化生态**——每个 AI 产品的工具接入方式都不同

MCP 解决的就是这个问题。它定义了一套**通用协议**，就像 USB-C 统一了充电接口，MCP 统一了 AI 与外部工具的通信方式。

### 2.3 MCP 的三大核心概念

MCP 协议定义了三种"AI 可以消费的东西"：

| 概念 | 作用 | 类比 | 示例 |
|------|------|------|------|
| **Resources** | 暴露数据/文件给 AI 读取 | "AI 的文件系统" | 数据库表结构、配置文件、日志文件 |
| **Tools** | 允许 AI 执行操作 | "AI 的双手" | 执行 SQL 查询、调用 API、发送消息 |
| **Prompts** | 预定义的提示词模板 | "AI 的快捷指令" | "分析订单数据"模板、"检查代码质量"模板 |

### 2.4 MCP 的通信架构

```
┌──────────────┐     JSON-RPC 2.0      ┌──────────────┐
│              │ ◄──────────────────► │              │
│  MCP Client  │   (stdio / SSE)      │  MCP Server  │
│  (Cursor/    │                      │  (你写的      │
│   Copilot/   │   Resources          │   Java 服务)  │
│   Claude)    │   Tools              │              │
│              │   Prompts            │              │
└──────────────┘                      └──────┬───────┘
                                              │
                                              ▼
                                       ┌──────────┐
                                       │  数据库   │
                                       │  API     │
                                       │  文件系统 │
                                       └──────────┘
```

- **传输方式**：stdio（标准输入输出，用于本地）或 SSE（Server-Sent Events，用于远程）
- **协议**：JSON-RPC 2.0（轻量、易实现、跨语言）
- **安全性**：连接在本地完成，数据不出本地机器

---

## 三、实战：构建一个 Java MCP Server

下面我们来写一个真实的 MCP Server——**让 AI 可以查询你的数据库**。

### 3.1 环境准备

- JDK 17+
- Spring Boot 3.3+
- MCP Java SDK：Spring AI MCP 或官方 [mcp-java](https://github.com/modelcontextprotocol/java-sdk)
- MySQL 8.0（可替换为其他数据库）

### 3.2 项目结构

```
mcp-database-server/
├── pom.xml
└── src/main/java/com/example/mcp/
    ├── McpDatabaseApplication.java
    ├── config/
    │   └── McpServerConfig.java          # MCP Server 核心配置
    ├── tool/
    │   └── DatabaseQueryTool.java        # 数据库查询工具
    ├── resource/
    │   └── TableSchemaResource.java      # 表结构资源
    └── prompt/
        └── OrderAnalysisPrompt.java      # 订单分析提示词模板
```

### 3.3 核心依赖（pom.xml）

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter</artifactId>
</dependency>

<!-- MCP Java SDK -->
<dependency>
    <groupId>io.modelcontextprotocol</groupId>
    <artifactId>mcp-core</artifactId>
    <version>0.9.0</version>
</dependency>

<!-- 数据库驱动 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jdbc</artifactId>
</dependency>
<dependency>
    <groupId>com.mysql</groupId>
    <artifactId>mysql-connector-j</artifactId>
</dependency>
```

### 3.4 MCP Server 核心配置

```java
@Configuration
public class McpServerConfig {

    @Bean
    public McpServer mcpServer(
            DatabaseQueryTool queryTool,
            TableSchemaResource schemaResource,
            OrderAnalysisPrompt analysisPrompt) {

        return McpServer.builder()
                .serverName("database-mcp-server")
                .serverVersion("1.0.0")
                // 注册 Tools：AI 可以调用的操作
                .tool(queryTool)
                // 注册 Resources：AI 可以读取的数据
                .resource(schemaResource)
                // 注册 Prompts：预定义的提示词模板
                .prompt(analysisPrompt)
                // 使用 stdio 传输（Cursor/Copilot 通过 stdio 与本进程通信）
                .transport(new StdioServerTransport())
                .build();
    }
}
```

### 3.5 数据库查询工具（Tools）

这是最核心的部分——把 SQL 查询能力安全地暴露给 AI：

```java
@Component
public class DatabaseQueryTool implements McpTool {

    @Autowired
    private JdbcTemplate jdbcTemplate;

    @Autowired
    private SqlSecurityGuard securityGuard;

    @Override
    public String getName() {
        return "query_database";
    }

    @Override
    public String getDescription() {
        return """
                Execute a READ-ONLY SQL query against the MySQL database.
                Only SELECT statements are allowed.
                Use this tool to query order data, user data, payment records, etc.
                The database contains the following tables:
                - tb_order: order information
                - tb_order_item: order line items
                - tb_payment: payment records
                - tb_user: user information
                - tb_product: product catalog
                """;
    }

    @Override
    public Map<String, Object> getInputSchema() {
        return Map.of(
            "type", "object",
            "properties", Map.of(
                "sql", Map.of(
                    "type", "string",
                    "description", "A READ-ONLY SELECT SQL query to execute"
                ),
                "limit", Map.of(
                    "type", "integer",
                    "description", "Maximum rows to return (default: 100, max: 1000)",
                    "default", 100
                )
            ),
            "required", List.of("sql")
        );
    }

    @Override
    public McpToolResult execute(Map<String, Object> arguments) {
        String sql = (String) arguments.get("sql");
        int limit = arguments.containsKey("limit")
                ? ((Number) arguments.get("limit")).intValue()
                : 100;

        // ===== 安全检查：三道防线 =====

        // 防线1：只允许 SELECT 语句
        if (!securityGuard.isSelectOnly(sql)) {
            return McpToolResult.error(
                "SECURITY: Only SELECT queries are allowed. " +
                "INSERT/UPDATE/DELETE/DDL statements are blocked. " +
                "Attempted SQL: " + sql
            );
        }

        // 防线2：限制返回行数，防止全表扫描
        if (limit > 1000) {
            limit = 1000;
        }

        // 防线3：禁止危险关键词（INTO OUTFILE、LOAD_FILE 等）
        if (securityGuard.containsDangerousKeywords(sql)) {
            return McpToolResult.error(
                "SECURITY: Dangerous SQL keywords detected."
            );
        }

        try {
            // 添加 LIMIT 子句（智能处理已有 LIMIT 的情况）
            String safeSql = securityGuard.addLimitIfNeeded(sql, limit);

            List<Map<String, Object>> results = jdbcTemplate.queryForList(safeSql);

            return McpToolResult.success(
                Map.of(
                    "row_count", results.size(),
                    "sql_executed", safeSql,
                    "data", results
                )
            );
        } catch (Exception e) {
            return McpToolResult.error("Query failed: " + e.getMessage());
        }
    }
}
```

### 3.6 SQL 安全守卫（安全防线实现）

```java
@Component
public class SqlSecurityGuard {

    private static final Set<String> DANGEROUS_KEYWORDS = Set.of(
        "INTO OUTFILE", "INTO DUMPFILE", "LOAD_FILE",
        "LOAD DATA", "EXEC", "EXECUTE", "SLEEP",
        "BENCHMARK", "WAITFOR"
    );

    /**
     * 检查 SQL 是否仅为 SELECT 查询
     */
    public boolean isSelectOnly(String sql) {
        String trimmed = sql.trim().toUpperCase();

        // 必须以 SELECT 开头
        if (!trimmed.startsWith("SELECT")) {
            return false;
        }

        // 禁止 DML/DDL 关键词
        String[] forbidden = {"INSERT", "UPDATE", "DELETE", "DROP",
                "CREATE", "ALTER", "TRUNCATE", "RENAME",
                "REPLACE", "MERGE", "GRANT", "REVOKE"};

        for (String keyword : forbidden) {
            // 使用词边界匹配，避免误判（如 column 名包含这些词）
            if (trimmed.matches(".*\\b" + keyword + "\\b.*")) {
                return false;
            }
        }
        return true;
    }

    /**
     * 检查是否包含危险关键词
     */
    public boolean containsDangerousKeywords(String sql) {
        String upper = sql.toUpperCase();
        return DANGEROUS_KEYWORDS.stream().anyMatch(upper::contains);
    }

    /**
     * 智能添加 LIMIT 子句
     */
    public String addLimitIfNeeded(String sql, int limit) {
        String upper = sql.toUpperCase().trim();
        // 如果已有 LIMIT，取较小值
        if (upper.matches(".*\\bLIMIT\\s+\\d+\\s*$")) {
            return sql.replaceAll("(?i)\\bLIMIT\\s+(\\d+)\\s*$",
                    "LIMIT " + Math.min(
                        Integer.parseInt(sql.replaceAll(".*LIMIT\\s+(\\d+).*", "$1")),
                        limit
                    ));
        }
        // 去掉末尾分号后添加 LIMIT
        String cleaned = sql.replaceAll(";\\s*$", "");
        return cleaned + " LIMIT " + limit;
    }
}
```

> **重要提示**：生产环境强烈建议使用**独立的只读数据库用户**，即便 SQL 审查被绕过，数据库层面也没有写权限。这是纵深防御的关键。

### 3.7 表结构资源（Resources）

把数据库的表结构暴露给 AI，让它知道有哪些表、哪些字段：

```java
@Component
public class TableSchemaResource implements McpResource {

    @Autowired
    private JdbcTemplate jdbcTemplate;

    @Override
    public String getUri() {
        return "database://schema";
    }

    @Override
    public String getName() {
        return "Database Schema";
    }

    @Override
    public String getDescription() {
        return "Complete schema of all tables in the database, " +
               "including column names, types, and comments";
    }

    @Override
    public String getMimeType() {
        return "application/json";
    }

    @Override
    public McpResourceResult read() {
        // 从 information_schema 拉取表结构
        String sql = """
            SELECT TABLE_NAME, COLUMN_NAME, DATA_TYPE,
                   COLUMN_COMMENT, IS_NULLABLE, COLUMN_KEY
            FROM information_schema.COLUMNS
            WHERE TABLE_SCHEMA = 'your_database'
            ORDER BY TABLE_NAME, ORDINAL_POSITION
            """;

        List<Map<String, Object>> columns = jdbcTemplate.queryForList(sql);

        // 按表名分组
        Map<String, List<Map<String, Object>>> schema = columns.stream()
            .collect(Collectors.groupingBy(
                row -> (String) row.get("TABLE_NAME")
            ));

        return McpResourceResult.success(schema);
    }
}
```

### 3.8 提示词模板（Prompts）

预置一些常用的分析模板，AI 可以直接调用：

```java
@Component
public class OrderAnalysisPrompt implements McpPrompt {

    @Override
    public String getName() {
        return "analyze_orders";
    }

    @Override
    public String getDescription() {
        return "Analyze order data for a given time period, " +
               "including total orders, payment success rate, " +
               "hourly distribution, and anomaly detection";
    }

    @Override
    public Map<String, Object> getArguments() {
        return Map.of(
            "start_date", Map.of(
                "type", "string",
                "description", "Start date in YYYY-MM-DD format (default: 7 days ago)"
            ),
            "end_date", Map.of(
                "type", "string",
                "description", "End date in YYYY-MM-DD format (default: yesterday)"
            )
        );
    }

    @Override
    public String getTemplate(Map<String, Object> args) {
        String start = (String) args.getOrDefault("start_date", "7 days ago");
        String end = (String) args.getOrDefault("end_date", "yesterday");

        return """
            You are a data analyst. Please analyze the order data from %s to %s.

            Follow these steps:
            1. Use the query_database tool to get:
               - Total order count and payment success rate for each day
               - Hourly order distribution
               - Top 10 products by sales volume
               - Payment failure reasons and their counts

            2. After getting the data, analyze:
               - Is there any downward trend in payment success rate?
               - Are there specific hours with abnormally high failure rates?
               - Which products show unusual order volume changes?

            3. Provide your analysis in a structured format with:
               - Key metrics summary
               - Trend analysis
               - Anomaly detection results
               - Actionable recommendations

            Important: ALL queries must use the query_database tool.
            Do NOT guess or fabricate data.
            """.formatted(start, end);
    }
}
```

### 3.9 启动类

```java
@SpringBootApplication
public class McpDatabaseApplication {

    public static void main(String[] args) {
        SpringApplication.run(McpDatabaseApplication.class, args);
    }
}
```

### 3.10 打包与运行

```bash
# 编译打包
mvn clean package -DskipTests

# 启动 MCP Server（stdio 模式，由 Cursor/Copilot 通过子进程启动）
java -jar target/mcp-database-server-1.0.0.jar
```

启动后，MCP Server 通过标准输入/输出（stdio）等待 JSON-RPC 消息。你的 IDE 中的 MCP Client 会作为父进程启动它并建立通信。

---

## 四、在 Cursor / Copilot 中配置 MCP Client

### 4.1 Cursor 配置

在 Cursor 中，编辑 MCP 配置文件（macOS 路径）：

```bash
# 配置文件路径
~/.cursor/mcp.json
```

```json
{
  "mcpServers": {
    "database-local": {
      "command": "java",
      "args": ["-jar", "/path/to/mcp-database-server-1.0.0.jar"],
      "env": {
        "DB_URL": "jdbc:mysql://localhost:3306/your_db?useSSL=false",
        "DB_USER": "mcp_readonly",
        "DB_PASSWORD": "your_readonly_password"
      }
    }
  }
}
```

### 4.2 VS Code Copilot 配置

VS Code 中的 GitHub Copilot 也支持 MCP，配置文件：

```json
// .vscode/mcp.json（项目级）或 ~/.vscode/mcp.json（全局级）
{
  "servers": {
    "database": {
      "type": "stdio",
      "command": "java",
      "args": ["-jar", "/path/to/mcp-database-server-1.0.0.jar"],
      "env": {}
    }
  }
}
```

### 4.3 Claude Desktop 配置

你也可以在 Claude Desktop 中使用 MCP：

```json
// ~/Library/Application Support/Claude/claude_desktop_config.json (macOS)
{
  "mcpServers": {
    "database-local": {
      "command": "java",
      "args": ["-jar", "/path/to/mcp-database-server-1.0.0.jar"]
    }
  }
}
```

> 配置完成后重启 IDE，你会在聊天面板中看到新注册的工具列表。

---

## 五、完整对话流程：从自然语言到数据库查询

下面我们来拆解一次完整的 MCP 调用链路：

### Step 1：用户输入自然语言

```
用户：tb_order 表里昨天有多少笔订单？支付成功率是多少？
```

### Step 2：AI 解析意图并生成工具调用

AI 首先会去读取 `database://schema` 资源，了解 tb_order 表的结构：

```json
// AI 通过 MCP Client 发送：
{
  "jsonrpc": "2.0",
  "method": "resources/read",
  "params": { "uri": "database://schema" },
  "id": 1
}

// MCP Server 返回：
{
  "jsonrpc": "2.0",
  "result": {
    "contents": [
      {
        "uri": "database://schema",
        "text": "{\"tb_order\": [{\"column\": \"id\", \"type\": \"bigint\"}, ...]}"
      }
    ]
  },
  "id": 1
}
```

### Step 3：AI 生成 SQL 并调用工具

了解了表结构后，AI 生成 SQL 并通过 `query_database` 工具执行：

```json
// AI 通过 MCP Client 发送：
{
  "jsonrpc": "2.0",
  "method": "tools/call",
  "params": {
    "name": "query_database",
    "arguments": {
      "sql": "SELECT COUNT(*) AS total, SUM(CASE WHEN pay_status='SUCCESS' THEN 1 ELSE 0 END) AS paid FROM tb_order WHERE create_time >= '2026-05-04' AND create_time < '2026-05-05'",
      "limit": 100
    }
  },
  "id": 2
}

// MCP Server 经过安全审查后执行 SQL，返回：
{
  "jsonrpc": "2.0",
  "result": {
    "content": [
      {
        "type": "text",
        "text": "{\"row_count\":1,\"data\":[{\"total\":12840,\"paid\":11556}]}"
      }
    ]
  },
  "id": 2
}
```

### Step 4：AI 将结果转换为自然语言

```
AI：昨天（2026-05-04）tb_order 表共有 12,840 笔订单，
    其中支付成功 11,556 笔，支付成功率为 90.0%。
```

### 完整的时序图

```
用户 ──► Cursor/Copilot ──► MCP Server ──► MySQL
 │              │                │            │
 │  输入自然语言  │                │            │
 │─────────────►│                │            │
 │              │  resources/read │            │
 │              │───────────────►│            │
 │              │                │  查表结构   │
 │              │                │───────────►│
 │              │                │◄───────────│
 │              │  返回 schema   │            │
 │              │◄───────────────│            │
 │              │                │            │
 │              │  tools/call    │            │
 │              │  (query_database)          │
 │              │───────────────►│            │
 │              │                │ SQL审查通过 │
 │              │                │───SELECT──►│
 │              │                │◄──结果─────│
 │              │  返回查询结果   │            │
 │              │◄───────────────│            │
 │              │                │            │
 │  返回分析结果  │                │            │
 │◄─────────────│                │            │
```

---

## 六、MCP 生态：当前可用的 MCP Server

MCP 协议虽然发布时间不长，但生态已经相当丰富。以下是一些官方和社区维护的 MCP Server：

### 6.1 官方 & 知名 MCP Server

| MCP Server | 功能 | 安装方式 |
|------------|------|----------|
| **@anthropic/mcp-server-postgres** | 查询 PostgreSQL 数据库 | npm 包，开箱即用 |
| **@anthropic/mcp-server-sqlite** | 查询 SQLite 数据库 | npm 包 |
| **@modelcontextprotocol/server-github** | 操作 GitHub（Issues/PR/Labels） | 官方维护 |
| **@modelcontextprotocol/server-slack** | 发送/查询 Slack 消息 | 官方维护 |
| **@modelcontextprotocol/server-filesystem** | 安全读写本地文件 | 官方维护 |
| **@modelcontextprotocol/server-google-maps** | 查询 Google Maps 地理信息 | 官方维护 |
| **mcp-server-jira** | 操作 Jira（创建/查询/更新 Issue） | 社区维护 |
| **mcp-server-brave-search** | 通过 Brave Search 搜索网页 | 官方维护 |

### 6.2 快速体验（无需写代码）

如果你只是想快速体验，可以直接在 Claude Desktop 或 Cursor 中配置现成的 MCP Server。例如，配置 GitHub MCP Server：

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "ghp_xxxxxxxxxxxx"
      }
    }
  }
}
```

配置完成后，你就可以在 Cursor 里说：

> "帮我查看最近 3 天合并到 main 分支的 PR，总结一下都改了些什么"

AI 会自动调用 GitHub MCP Server，拉取 PR 列表，读取 Diff，然后给你一份摘要。

---

## 七、企业级应用场景

### 7.1 场景一：AI 直接查询生产数据库（只读）做数据分析

**痛点**：数据分析需求频繁，每次都要写 SQL、跑查询、导出 Excel、做图表。数据开发和业务方之间反复沟通。

**MCP 方案**：
- 部署只读 MCP 数据库 Server（连接生产库从库/只读副本）
- 业务人员在 IDE 中用自然语言提问
- AI 自动生成 SQL → 执行 → 分析 → 出报告

**示例对话**：

```
PM：上周新上线的优惠券功能，使用率怎么样？

AI：[查询 tb_coupon 和 tb_order 表]
    上周共发放优惠券 58,320 张，使用 12,487 张，使用率 21.4%。
    使用优惠券的订单中，客单价从 ¥128 提升到 ¥156（+21.9%）。
    建议：可以考虑提高满减门槛，进一步提升客单价。
```

### 7.2 场景二：AI 通过 MCP 操作 Jira 自动创建 Bug Ticket

**痛点**：线上出问题后，排查 → 定位 → 建 Jira Ticket → 填写复现步骤 → 指派负责人，流程繁琐。

**MCP 方案**：
- 配置 Jira MCP Server
- 配合日志分析 MCP Server
- AI 自动分析异常 → 创建 Jira Ticket → 填入上下文

**示例对话**：

```
Dev：刚才 gateway 报了一波 502，帮我分析一下并建个 Bug Ticket。

AI：[查询日志、监控数据]
    已分析：14:30-14:35 期间 gateway 出现 347 次 502 错误，
    原因是下游 user-service 发生 Full GC 导致 STW 约 4.2 秒。
    
    已自动创建 Jira Bug Ticket：
    - 标题：user-service Full GC 导致 gateway 502 雪崩
    - 优先级：P1
    - 指派：user-service 团队
    - 复现信息：gc.log 片段已附上
    - 链接：https://jira.company.com/browse/BUG-2026
```

### 7.3 场景三：AI 查询 GitHub PR 状态并汇总

**痛点**：技术 Leader 每天要 review 多个仓库的 PR，手工打开 GitHub 逐个查看效率低。

**MCP 方案**：

```
TL：帮我汇总一下组里 3 个仓库待 review 的 PR，按优先级排个序。

AI：[查询 repo-a, repo-b, repo-c]
    找到 7 个待 review 的 PR：

    🔴 高优：
    1. repo-a#342: 修复支付回调幂等性 - 已等待 2 天 - 作者：张三
    2. repo-b#89: 修复 SQL 注入漏洞 - 已等待 1 天 - 作者：李四

    🟡 中优：
    3. repo-c#211: 新增导出 CSV 功能 - 作者：王五
    ...

    建议优先 review repo-b#89，涉及安全漏洞修复。
```

---

## 八、MCP 安全最佳实践

MCP 让 AI 获得了"动手能力"，安全就成了必须严肃对待的问题。以下是你在生产环境中使用 MCP 时必须遵守的安全准则：

### 8.1 原则一：最小权限（Principle of Least Privilege）

```json
// ❌ 错误：给 MCP 使用 root 数据库用户
{
  "DB_USER": "root",
  "DB_PASSWORD": "root123"
}

// ✅ 正确：创建专用于 MCP 的只读用户
// CREATE USER 'mcp_readonly'@'localhost' IDENTIFIED BY 'xxx';
// GRANT SELECT ON your_db.* TO 'mcp_readonly'@'localhost';
{
  "DB_USER": "mcp_readonly",
  "DB_PASSWORD": "secure_random_password"
}
```

### 8.2 原则二：输入验证（Input Validation）

```java
// 永远在 MCP Server 端做输入校验，不要信任 AI 生成的任何输入

// SQL 注入防护：用正则 + 词法分析双重验证
if (!securityGuard.isSelectOnly(sql)) {
    return error("Only SELECT allowed");
}

// 参数边界检查
if (limit < 1 || limit > 1000) {
    return error("Limit must be between 1 and 1000");
}

// 查询超时保护
jdbcTemplate.setQueryTimeout(10);  // 10 秒超时
```

### 8.3 原则三：审计日志（Audit Logging）

```java
// 记录每一次 MCP 工具调用
log.info("MCP_TOOL_CALL tool={} user={} args={} timestamp={}",
    toolName, currentUser, maskSensitive(args), Instant.now());

// 记录被拦截的非法请求
log.warn("MCP_SECURITY_BLOCK reason={} sql={} user={}",
    "DROP_TABLE_ATTEMPT", blockedSql, currentUser);
```

### 8.4 原则四：网络隔离

```
┌─────────────────────────────────────────────┐
│  开发环境（本地机器）                          │
│  ┌─────────┐  stdio   ┌───────────┐         │
│  │ Cursor  │◄────────►│ MCP Server │         │
│  └─────────┘          └─────┬─────┘         │
│                              │               │
└──────────────────────────────┼───────────────┘
                                │  只读连接
                                ▼
┌───────────────────────────────┼───────────────┐
│  生产环境                       │               │
│                        ┌───────▼──────┐        │
│                        │  从库/只读副本 │        │
│                        └──────────────┘        │
│  主库（读写）不暴露给 MCP                       │
└─────────────────────────────────────────────┘
```

### 8.5 安全检查清单

| 检查项 | 说明 |
|--------|------|
| ☐ 使用独立的只读数据库用户 | DBA 层面硬限制，最可靠 |
| ☐ SQL 关键词白名单审查 | 仅允许 SELECT，拦截 DML/DDL |
| ☐ 返回行数上限（≤1000） | 防止全表扫描拖垮数据库 |
| ☐ 查询超时（≤10s） | 防止慢查询 |
| ☐ 所有调用记录审计日志 | 事后可追溯 |
| ☐ MCP Server 仅连接从库/只读副本 | 隔离主库风险 |
| ☐ 敏感字段脱敏（手机号、身份证等） | 数据安全合规 |
| ☐ MCP 配置文件权限设为 600 | 防止其他用户读取数据库密码 |

---

## 九、MCP 的局限与展望

### 当前局限

1. **工具描述靠自然语言**：AI 通过文字描述理解工具功能，如果描述不清晰，AI 可能用错
2. **难以处理大数据量**：目前 MCP 适合查询型场景，传几百 MB 数据会超时
3. **复杂事务不支持**：MCP 当前是单次调用模式，不支持跨工具的事务
4. **调试体验待优化**：MCP Server 在 stdio 模式下，日志和调试不够直观

### 未来方向

- **MCP 2.0**：支持流式响应、多轮工具链编排
- **MCP Registry**：类似 Docker Hub，统一的 MCP Server 发现和分发平台
- **MCP + RAG**：MCP 获取结构化数据 + RAG 检索文档，两者互补

---

## 写在最后

MCP 协议的核心理念可以用一句话概括：

> **给 AI 装上手，但不给钥匙。**

它让 AI 从"聊天机器人"进化为"能干活的工作伙伴"，同时通过协议层面的约束确保安全性。对于 Java 团队来说，用 Spring Boot 实现一个 MCP Server 的门槛并不高，但带来的效能提升却是立竿见影的。

如果你的团队还在让 AI "只说不做"，是时候试试 MCP 了。

---

## 下一篇预告

**系列一 - 第 29 篇：《AI 辅助老旧系统迁移：从 Struts2 到 Spring Boot，AI 能帮你搞定多少？》**

老旧系统的技术栈迁移，往往是团队最头疼的项目。下一篇我将带你实战：
- 用 AI 分析 Struts2 项目的代码结构，自动生成迁移规划
- Controller 层：Action → @RestController 的批量转换
- 配置层：struts.xml → application.yml 的映射
- 依赖层：Maven/Ant → Gradle 的依赖迁移
- AI 能搞定 60% 的重复工作，剩下 40% 才是真正的考验

**关注本系列，不错过任何一篇实战干货。**

---

> **作者简介**：Java 技术博主，专注于 AI 编程工具链实战与团队研发效能提升。本系列文章已在 20+ 个真实项目中验证，累计为团队节省 3000+ 小时开发时间。

> *本文为「AI 编程工具链实战」系列第 28 篇，文章首发于 CSDN。转载请联系作者。*
