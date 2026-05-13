# Prompt 模板引擎：使用 Jinja2/Mustache 动态生成 Prompt

> 你有10个不同的需求，但 Prompt 结构是一样的只是参数不同——复制粘贴10次只改变量？不，应该用模板引擎。一行 Java 代码生成不同场景的 Prompt，告别硬编码和复制粘贴。

---

## 一、开篇：你的 Prompt 管理现状

先来看一个典型场景——你所在的团队接入了 LLM 来做代码生成，目前有这些需求：

1. 为 `User` 实体生成 CRUD 接口
2. 为 `Order` 实体生成 CRUD 接口
3. 为 `Product` 实体生成 CRUD 接口
4. 为 `Category` 实体生成 CRUD 接口
5. ……还有6个实体等着你

你的做法是不是这样？

```java
// 需求1
String prompt1 = """
    你是一位Java开发专家。请为User实体生成完整的CRUD代码。
    User实体包含字段：id(Long), username(String), email(String), age(Integer)。
    框架：Spring Boot 3.2 + MyBatis-Plus。
    数据库：MySQL 8.0。
    要求：包含参数校验、分页查询、全局异常处理。
    """;

// 需求2
String prompt2 = """
    你是一位Java开发专家。请为Order实体生成完整的CRUD代码。
    Order实体包含字段：id(Long), orderNo(String), userId(Long), amount(BigDecimal)。
    框架：Spring Boot 3.2 + MyBatis-Plus。
    数据库：MySQL 8.0。
    要求：包含参数校验、分页查询、全局异常处理。
    """;

// 需求3...需求10...继续复制粘贴...
```

**问题很明显：** 大量重复的 Prompt 文本，只有少数几个变量在变化。一旦需要修改"要求"部分（比如加个"日志记录"），就要改10个地方。

更可怕的是，随着项目推进，你的 Prompt 会散落在代码的各个角落，版本管理、效果追踪、团队成员复用都成了大问题。

**解决方案：Prompt 模板化。**

---

## 二、Prompt 模板化的核心价值

在深入代码之前，先理解为什么要做模板化：

### 2.1 复用性

一个"通用CRUD生成"模板，可以服务100个不同的实体。一次编写，到处使用。

### 2.2 版本管理

```bash
prompts/
├── v1/
│   ├── crud-generation.mustache
│   ├── unit-test-generation.mustache
│   └── bug-fix-analysis.mustache
├── v2/
│   ├── crud-generation.mustache  # 优化版：加了安全约束
│   └── unit-test-generation.mustache
└── latest -> v2/
```

Prompts 像代码一样存储在 Git 中，有提交历史、有版本号、可以做 Code Review。

### 2.3 参数驱动

```
输入：{entity: "User", fields: [...], framework: "Spring Boot 3.2"}
输出：完整的 Prompt 文本
```

同一套逻辑，不同参数生成不同 Prompt。测试、AB 实验都变得简单。

### 2.4 防止注入

```java
// 用户输入：entityName = "User; DROP TABLE users; --"
// 不做转义的模板：直接把恶意输入拼进去
// 模板引擎自动转义：entityName = "User; DROP TABLE users; --"（安全）
```

模板引擎天然防止 Prompt 注入攻击——所有变量都经过转义处理。

---

## 三、Java 实现：Mustache 模板引擎

为什么选择 Mustache？

| 模板引擎 | Java 支持 | 语法简洁度 | 学习成本 | 依赖大小 |
|---------|:------:|:------:|:-----:|:-----:|
| Mustache | 官方 Java 实现 | 极简 | 低 | <100KB |
| Jinja2 | 需通过 Python 桥接 | 丰富但复杂 | 中 | N/A |
| StringTemplate | Java 原生 | 中等 | 低 | 约1MB |
| Thymeleaf | Java 原生 | 丰富 | 高 | 约3MB |
| Freemarker | Java 原生 | 丰富 | 高 | 约1.5MB |

**Mustache（逻辑无关模板 logic-less templates）** 专为"只是替换变量"的场景设计，语法极简、运行时零逻辑，非常适合 Prompt 模板的场景。

### 3.1 引入依赖

```xml
<!-- pom.xml -->
<dependency>
    <groupId>com.github.spullara.mustache.java</groupId>
    <artifactId>compiler</artifactId>
    <version>0.9.11</version>
</dependency>
```

### 3.2 第一个 Mustache Prompt 模板

**模板文件：** `prompts/crud-generation.mustache`

```mustache
You are a senior Java backend developer specializing in Spring Boot.

Generate complete CRUD code for {{entityName}} entity.

Entity fields:
{{#fields}}
  - {{name}}: {{type}}{{#notNull}} (NOT NULL){{/notNull}}{{#maxLength}}, max length: {{maxLength}}{{/maxLength}}
{{/fields}}

Technology stack:
- Framework: {{framework}}
- ORM: {{orm}}
- Database: {{database}}
- Java version: {{javaVersion}}

Requirements:
{{#requirements}}
  - {{.}}
{{/requirements}}

{{#needsPagination}}
Pagination: Use {{paginationTool}} for list queries with pageNum and pageSize parameters.
{{/needsPagination}}

{{#needsValidation}}
Validation: Use @Valid + JSR303 annotations on all input DTOs.
{{/needsValidation}}

Output the following files:
1. {{entityName}}.java (Entity class)
2. {{entityName}}Mapper.java (MyBatis-Plus mapper)
3. {{entityName}}Service.java (Interface)
4. {{entityName}}ServiceImpl.java (Implementation)
5. {{entityName}}Controller.java (REST controller)
6. {{entityName}}DTO.java (Request DTO)
7. {{entityName}}VO.java (Response VO)

Only output Java code. No explanations.
```

### 3.3 核心 Prompt 模板引擎实现

```java
import com.github.mustachejava.DefaultMustacheFactory;
import com.github.mustachejava.Mustache;
import com.github.mustachejava.MustacheFactory;

import java.io.*;
import java.nio.file.*;
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;

/**
 * Prompt 模板引擎
 * 基于 Mustache 实现，支持模板缓存、参数驱动、模板继承
 */
public class PromptTemplateEngine {

    private final MustacheFactory mustacheFactory;
    private final Map<String, Mustache> templateCache;
    private final Map<String, Map<String, Object>> paramDefaults;
    private final Path templateBasePath;

    public PromptTemplateEngine(String templateDir) {
        this.mustacheFactory = new DefaultMustacheFactory();
        this.templateCache = new ConcurrentHashMap<>();
        this.paramDefaults = new HashMap<>();
        this.templateBasePath = Paths.get(templateDir);

        loadDefaultParams();
    }

    /**
     * 渲染 Prompt 模板
     * @param templateName 模板名称（不含.mustache后缀）
     * @param params 模板参数
     * @return 渲染后的 Prompt 文本
     */
    public String render(String templateName, Map<String, Object> params) {
        Mustache template = getOrCompileTemplate(templateName);

        // 合并默认参数
        Map<String, Object> mergedParams = new HashMap<>();
        Map<String, Object> defaults = paramDefaults.getOrDefault(templateName, Collections.emptyMap());
        mergedParams.putAll(defaults);
        mergedParams.putAll(params);

        StringWriter writer = new StringWriter();
        template.execute(writer, new Object[]{mergedParams});
        return writer.toString();
    }

    /**
     * 从文件渲染 Prompt
     * @param templateName 模板名
     * @param params 参数
     * @return 渲染后的 Prompt
     */
    public String renderFromFile(String templateName, Map<String, Object> params) {
        Path templatePath = templateBasePath.resolve(templateName + ".mustache");
        if (!Files.exists(templatePath)) {
            throw new IllegalArgumentException("Template not found: " + templatePath);
        }

        try (Reader reader = Files.newBufferedReader(templatePath)) {
            Mustache template = mustacheFactory.compile(reader, templateName);
            StringWriter writer = new StringWriter();
            Map<String, Object> mergedParams = mergeWithDefaults(templateName, params);
            template.execute(writer, new Object[]{mergedParams});
            return writer.toString();
        } catch (IOException e) {
            throw new UncheckedIOException("Failed to render template: " + templateName, e);
        }
    }

    /**
     * 批量渲染：同一模板，不同参数
     */
    public Map<String, String> batchRender(String templateName, List<Map<String, Object>> paramsList) {
        Map<String, String> results = new LinkedHashMap<>();

        for (int i = 0; i < paramsList.size(); i++) {
            Map<String, Object> params = paramsList.get(i);
            String entityName = (String) params.getOrDefault("entityName", "Entity" + i);
            String prompt = render(templateName, params);
            results.put(entityName, prompt);
        }

        return results;
    }

    /**
     * 模板继承：将基础模板的渲染结果嵌入到子模板
     */
    public String renderWithInheritance(String baseTemplate, String childTemplate,
                                         Map<String, Object> baseParams, Map<String, Object> childParams) {
        String basePrompt = render(baseTemplate, baseParams);
        childParams.put("basePrompt", basePrompt);
        return render(childTemplate, childParams);
    }

    /**
     * 设置模板的默认参数
     */
    public void setDefaultParams(String templateName, Map<String, Object> defaults) {
        this.paramDefaults.put(templateName, new HashMap<>(defaults));
    }

    /**
     * 清除模板缓存（模板文件更新后调用）
     */
    public void invalidateCache(String templateName) {
        templateCache.remove(templateName);
    }

    public void invalidateAllCache() {
        templateCache.clear();
    }

    // ---- Private methods ----

    private Mustache getOrCompileTemplate(String templateName) {
        return templateCache.computeIfAbsent(templateName, name -> {
            Path templatePath = templateBasePath.resolve(name + ".mustache");
            try (Reader reader = Files.newBufferedReader(templatePath)) {
                return mustacheFactory.compile(reader, name);
            } catch (IOException e) {
                throw new UncheckedIOException("Failed to compile template: " + name, e);
            }
        });
    }

    private Map<String, Object> mergeWithDefaults(String templateName, Map<String, Object> params) {
        Map<String, Object> merged = new HashMap<>();
        Map<String, Object> defaults = paramDefaults.getOrDefault(templateName, Collections.emptyMap());
        merged.putAll(defaults);
        merged.putAll(params);
        return merged;
    }

    /**
     * 加载全局默认参数
     */
    private void loadDefaultParams() {
        Map<String, Object> crudDefaults = new HashMap<>();
        crudDefaults.put("framework", "Spring Boot 3.2");
        crudDefaults.put("orm", "MyBatis-Plus 3.5");
        crudDefaults.put("database", "MySQL 8.0");
        crudDefaults.put("javaVersion", "JDK 17");
        crudDefaults.put("needsPagination", true);
        crudDefaults.put("needsValidation", true);
        crudDefaults.put("paginationTool", "MyBatis-Plus PageHelper");
        crudDefaults.put("requirements", Arrays.asList(
                "Use @Valid + JSR303 validation on all input DTOs",
                "Use @RestControllerAdvice for global exception handling",
                "Use constructor injection (@RequiredArgsConstructor)",
                "Add @Transactional on service methods",
                "Return unified ApiResponse wrapper",
                "Use Swagger/OpenAPI 3 annotations (@Tag, @Operation)"
        ));
        paramDefaults.put("crud-generation", crudDefaults);
    }
}
```

### 3.4 使用示例

```java
public class PromptTemplateDemo {

    public static void main(String[] args) {
        PromptTemplateEngine engine = new PromptTemplateEngine("prompts/");

        // 单次渲染：生成 User 的 CRUD Prompt
        Map<String, Object> userParams = new HashMap<>();
        userParams.put("entityName", "User");
        userParams.put("fields", Arrays.asList(
                Map.of("name", "id", "type", "Long", "notNull", true),
                Map.of("name", "username", "type", "String", "notNull", true, "maxLength", 50),
                Map.of("name", "email", "type", "String", "notNull", true, "maxLength", 100),
                Map.of("name", "age", "type", "Integer"),
                Map.of("name", "status", "type", "Integer", "notNull", true)
        ));

        String userPrompt = engine.render("crud-generation", userParams);
        System.out.println("=== User CRUD Prompt ===");
        System.out.println(userPrompt);

        // 批量渲染：一口气为所有实体生成 Prompt
        List<Map<String, Object>> allEntities = Arrays.asList(
                buildEntityParams("User", Arrays.asList(
                        field("id", "Long", true),
                        field("username", "String", true, 50),
                        field("email", "String", true, 100)
                )),
                buildEntityParams("Order", Arrays.asList(
                        field("id", "Long", true),
                        field("orderNo", "String", true, 32),
                        field("userId", "Long", true),
                        field("amount", "BigDecimal", true),
                        field("status", "Integer", true)
                )),
                buildEntityParams("Product", Arrays.asList(
                        field("id", "Long", true),
                        field("name", "String", true, 200),
                        field("price", "BigDecimal", true),
                        field("stock", "Integer", true)
                ))
        );

        Map<String, String> allPrompts = engine.batchRender("crud-generation", allEntities);
        System.out.println("批量生成了 " + allPrompts.size() + " 个 Prompt");

        allPrompts.forEach((entity, prompt) -> {
            System.out.println("\n=== " + entity + " (Token: "
                    + TokenCounter.estimateTokens(prompt) + ") ===");
            System.out.println(prompt.substring(0, 200) + "...");
        });
    }

    private static Map<String, Object> buildEntityParams(String name, List<Map<String, Object>> fields) {
        Map<String, Object> params = new HashMap<>();
        params.put("entityName", name);
        params.put("fields", fields);
        return params;
    }

    private static Map<String, Object> field(String name, String type, boolean notNull) {
        Map<String, Object> f = new HashMap<>();
        f.put("name", name);
        f.put("type", type);
        f.put("notNull", notNull);
        return f;
    }

    private static Map<String, Object> field(String name, String type, boolean notNull, int maxLength) {
        Map<String, Object> f = field(name, type, notNull);
        f.put("maxLength", maxLength);
        return f;
    }
}
```

---

## 四、模板继承与组合

实际项目中，一个 CRUD 生成 Prompt 可能需要根据不同场景组合不同的"模块"：基础结构模块、安全约束模块、测试生成模块等。

### 4.1 模板分层设计

```
基础模板 (base-prompt.mustache)
  ├── 角色定义（System Prompt部分）
  ├── 输出格式规范
  └── 通用约束

业务模板 (crud-base.mustache)
  ├── 继承基础模板
  ├── CRUD 特有的需求描述
  └── 字段列表渲染逻辑

场景模板 (crud-with-security.mustache)
  ├── 继承业务模板
  ├── 追加安全约束（XSS防护、SQL注入校验、权限控制）
  └── 场景特有的输出要求
```

### 4.2 模板 Partials（片段）

Mustache 支持 Partials——将一个模板作为另一个模板的"零件"嵌入。

**基础模板：** `prompts/partials/role-java-expert.mustache`

```mustache
You are a senior Java backend developer with {{yearsOfExperience}}+ years of experience.
You specialize in {{#specialties}}{{.}}, {{/specialties}}and writing clean, production-ready code.
```

**安全约束模板：** `prompts/partials/security-constraints.mustache`

```mustache
Security requirements:
- All user input MUST be validated and sanitized
- Use parameterized queries to prevent SQL injection
- Passwords MUST be encrypted with BCrypt
- JWT tokens MUST have expiration time
- {{#enableCORS}}Enable CORS with configured origins{{/enableCORS}}
- {{#enableRateLimit}}Add rate limiting on sensitive endpoints{{/enableRateLimit}}
```

**主模板中使用 Partials：**

```mustache
{{> role-java-expert}}

Task: Generate CRUD code for {{entityName}}.

{{> security-constraints}}

Fields:
{{#fields}}
  - {{name}}: {{type}}
{{/fields}}
```

### 4.3 Java 实现模板 Partial 注册

```java
/**
 * 增强版模板引擎，支持 Partials
 */
public class AdvancedPromptEngine extends PromptTemplateEngine {

    private final Map<String, String> partialTemplates = new HashMap<>();

    public AdvancedPromptEngine(String templateDir, String partialsDir) {
        super(templateDir);
        loadPartials(partialsDir);
    }

    private void loadPartials(String partialsDir) {
        try {
            Files.list(Paths.get(partialsDir))
                    .filter(p -> p.toString().endsWith(".mustache"))
                    .forEach(p -> {
                        try {
                            String name = p.getFileName().toString()
                                    .replace(".mustache", "");
                            String content = Files.readString(p);
                            partialTemplates.put(name, content);
                        } catch (IOException e) {
                            throw new UncheckedIOException(e);
                        }
                    });
        } catch (IOException e) {
            throw new UncheckedIOException("Failed to load partials from: " + partialsDir, e);
        }
        System.out.println("Loaded " + partialTemplates.size() + " partial templates");
    }

    /**
     * 构建一个组合 Prompt：基础模块 + 可选模块叠加
     */
    public String composePrompt(String baseTemplate, String[] modules, Map<String, Object> params) {
        StringBuilder composedTemplate = new StringBuilder();

        // 先加载基础模板
        composedTemplate.append("{{> ").append(baseTemplate).append("}}\n\n");

        // 逐个叠加模块
        for (String module : modules) {
            composedTemplate.append("{{> ").append(module).append("}}\n\n");
        }

        // 用 Mustache 编译并渲染
        String templateStr = composedTemplate.toString();
        MustacheFactory mf = new DefaultMustacheFactory() {
            @Override
            public Reader getReader(String resourceName) {
                String partialName = resourceName.replace(".mustache", "");
                String partialContent = partialTemplates.get(partialName);
                if (partialContent != null) {
                    return new StringReader(partialContent);
                }
                return super.getReader(resourceName);
            }
        };

        Mustache mustache = mf.compile(new StringReader(templateStr), "composed");
        StringWriter writer = new StringWriter();
        mustache.execute(writer, new Object[]{params});
        return writer.toString();
    }
}
```

---

## 五、实战：通用 CRUD 生成 Prompt 模板

一个生产级别的 CRUD 生成模板，需要覆盖各种可变参数：

```mustache
{{! prompts/crud-advanced.mustache }}
{{> role-java-expert}}

Task: Generate a complete CRUD module for **{{entityName}}** entity.

## Entity Specification

Table: `{{tableName}}`
Entity class: `{{packageName}}.entity.{{entityName}}`

Fields:
| Field | Type | DB Column | PK | NotNull | Default | Validation |
|-------|------|-----------|----|---------|---------|------------|
{{#fields}}
| {{name}} | {{type}} | {{columnName}} | {{#pk}}Yes{{/pk}}{{^pk}}No{{/pk}} | {{#notNull}}Yes{{/notNull}}{{^notNull}}No{{/notNull}} | {{defaultValue}} | {{validation}} |
{{/fields}}

## Technology Stack
- Framework: {{framework}}
- ORM: {{orm}}
- Database: {{databaseType}}
- Cache: {{#useRedis}}Redis (Spring Cache){{/useRedis}}{{^useRedis}}None{{/useRedis}}
- API Doc: {{#useSwagger}}SpringDoc OpenAPI 3{{/useSwagger}}{{^useSwagger}}None{{/useSwagger}}

## Files to Generate
1. **`{{entityName}}.java`** — JPA Entity / MyBatis-Plus Entity
2. **`{{entityName}}Mapper.java`** — Data access layer
   {{#ormMybatisPlus}}Extend BaseMapper<{{entityName}}>{{/ormMybatisPlus}}
3. **`{{entityName}}DTO.java`** — Request DTO with validation
4. **`{{entityName}}VO.java`** — Response VO
5. **`{{entityName}}Query.java`** — Query parameters ({{#needsPagination}}extends PageQuery{{/needsPagination}})
6. **`{{entityName}}Service.java`** — Business interface
7. **`{{entityName}}ServiceImpl.java`** — Business implementation
8. **`{{entityName}}Controller.java`** — REST controller

## API Endpoints
| Method | URL | Description | Request | Response |
|--------|-----|-------------|---------|----------|
| GET | /api/{{apiPath}} | Paginated list | {{entityName}}Query | Page<{{entityName}}VO> |
| GET | /api/{{apiPath}}/{id} | Get by ID | Long id | {{entityName}}VO |
| POST | /api/{{apiPath}} | Create | {{entityName}}DTO | {{entityName}}VO |
| PUT | /api/{{apiPath}}/{id} | Update | {{entityName}}DTO | {{entityName}}VO |
| DELETE | /api/{{apiPath}}/{id} | Delete | Long id | void |
{{#needsBatchDelete}}
| DELETE | /api/{{apiPath}}/batch | Batch delete | List<Long> ids | void |
{{/needsBatchDelete}}
{{#needsExport}}
| GET | /api/{{apiPath}}/export | Export to Excel | {{entityName}}Query | file |
{{/needsExport}}

## Code Requirements
{{#requirements}}
{{.}}
{{/requirements}}

## {{#needsTests}}Unit Tests{{/needsTests}}
{{#needsTests}}
Generate JUnit 5 unit tests for:
- {{entityName}}ServiceImpl (mock all dependencies)
- {{entityName}}Controller (MockMvc test)
Coverage target: {{testCoverage}}%
{{/needsTests}}

Output ONLY Java code. Each file separated by `// === FILENAME: ... ===`.
```

**使用示例：**

```java
public class CrudPromptGenerator {

    private final AdvancedPromptEngine engine;

    public CrudPromptGenerator(String templatesDir) {
        this.engine = new AdvancedPromptEngine(templatesDir, templatesDir + "/partials");
    }

    public String generateCrudPrompt(CrudGenerationConfig config) {
        Map<String, Object> params = new HashMap<>();

        // 基础信息
        params.put("entityName", config.entityName);
        params.put("tableName", config.tableName);
        params.put("packageName", config.packageName);
        params.put("apiPath", config.apiPath);

        // 字段列表
        List<Map<String, Object>> fields = new ArrayList<>();
        for (FieldConfig field : config.fields) {
            Map<String, Object> f = new HashMap<>();
            f.put("name", field.name);
            f.put("type", field.type);
            f.put("columnName", field.columnName);
            f.put("pk", field.isPk);
            f.put("notNull", field.notNull);
            f.put("defaultValue", field.defaultValue != null ? field.defaultValue : "—");
            f.put("validation", field.validation != null ? field.validation : "—");
            fields.add(f);
        }
        params.put("fields", fields);

        // 技术栈
        params.put("framework", config.framework);
        params.put("orm", config.orm);
        params.put("ormMybatisPlus", "MyBatis-Plus".equals(config.orm));
        params.put("databaseType", config.databaseType);
        params.put("useRedis", config.useRedis);
        params.put("useSwagger", config.useSwagger);

        // 功能开关
        params.put("needsPagination", config.needsPagination);
        params.put("needsBatchDelete", config.needsBatchDelete);
        params.put("needsExport", config.needsExport);
        params.put("needsTests", config.needsTests);
        params.put("testCoverage", config.testCoveragePercent);

        // 需求列表
        params.put("requirements", config.customRequirements);

        return engine.render("crud-advanced", params);
    }

    // ---- 配置类 ----

    public static class CrudGenerationConfig {
        public String entityName;
        public String tableName;
        public String packageName = "com.example";
        public String apiPath;
        public String framework = "Spring Boot 3.2";
        public String orm = "MyBatis-Plus";
        public String databaseType = "MySQL 8.0";
        public boolean useRedis = false;
        public boolean useSwagger = true;
        public boolean needsPagination = true;
        public boolean needsBatchDelete = false;
        public boolean needsExport = false;
        public boolean needsTests = false;
        public int testCoveragePercent = 80;
        public List<FieldConfig> fields = new ArrayList<>();
        public List<String> customRequirements = new ArrayList<>();
    }

    public static class FieldConfig {
        public String name;
        public String type;
        public String columnName;
        public boolean isPk;
        public boolean notNull;
        public String defaultValue;
        public String validation;

        public FieldConfig(String name, String type, String columnName) {
            this.name = name;
            this.type = type;
            this.columnName = columnName;
        }
    }
}
```

---

## 六、团队 Prompt 模板库管理

### 6.1 目录结构

```
prompts-repo/
├── README.md                     # 模板使用说明
├── CHANGELOG.md                  # 模板变更记录
├── templates/
│   ├── java/
│   │   ├── crud-generation.mustache
│   │   ├── unit-test.mustache
│   │   ├── bug-fix.mustache
│   │   ├── refactoring.mustache
│   │   └── performance-tuning.mustache
│   ├── database/
│   │   ├── sql-generation.mustache
│   │   └── schema-design.mustache
│   └── documentation/
│       ├── api-doc.mustache
│       └── javadoc.mustache
├── partials/
│   ├── role-java-expert.mustache
│   ├── security-constraints.mustache
│   ├── coding-standards.mustache
│   └── output-format.mustache
├── test-data/
│   ├── user-entity.json
│   ├── order-entity.json
│   └── product-entity.json
└── evaluation/
    └── benchmark-results.md
```

### 6.2 Git 工作流管理 Prompt 模板

```java
/**
 * 基于 Git 的 Prompt 模板管理器
 * 支持版本切换、AB测试、效果追踪
 */
public class GitPromptTemplateManager {

    private final Path repoPath;
    private final String gitRemote;

    public GitPromptTemplateManager(String repoPath, String gitRemote) {
        this.repoPath = Paths.get(repoPath);
        this.gitRemote = gitRemote;
    }

    /**
     * 切换到指定版本的模板
     */
    public PromptTemplateEngine loadTemplateVersion(String versionTag) {
        executeGit("git", "checkout", versionTag);
        return new PromptTemplateEngine(repoPath.resolve("templates").toString());
    }

    /**
     * AB 测试：同时加载两个版本的模板
     */
    public ABTemplatePair loadABTest(String versionA, String versionB) {
        PromptTemplateEngine engineA = loadTemplateVersion(versionA);
        PromptTemplateEngine engineB = loadTemplateVersion(versionB);
        return new ABTemplatePair(engineA, engineB);
    }

    /**
     * 发布新版本模板
     */
    public void releaseNewVersion(String version, String message) {
        executeGit("git", "add", ".");
        executeGit("git", "commit", "-m", "release: " + version + " - " + message);
        executeGit("git", "tag", version);
        executeGit("git", "push", "origin", "main", "--tags");
    }

    /**
     * 回滚到上一个版本
     */
    public void rollback() {
        executeGit("git", "revert", "HEAD", "--no-edit");
    }

    private void executeGit(String... command) {
        try {
            ProcessBuilder pb = new ProcessBuilder(command);
            pb.directory(repoPath.toFile());
            Process process = pb.start();
            int exitCode = process.waitFor();
            if (exitCode != 0) {
                String error = new String(process.getErrorStream().readAllBytes());
                throw new RuntimeException("Git command failed: " + String.join(" ", command) + "\n" + error);
            }
        } catch (IOException | InterruptedException e) {
            throw new RuntimeException("Git operation failed", e);
        }
    }

    public record ABTemplatePair(PromptTemplateEngine a, PromptTemplateEngine b) {}
}
```

---

## 七、模板引擎的性能优化

```java
/**
 * 高性能 Prompt 模板引擎
 * 针对高并发场景优化
 */
public class HighPerformancePromptEngine {

    private final LoadingCache<String, CompiledTemplate> templateCache;
    private final ScheduledExecutorService cacheCleaner;

    public HighPerformancePromptEngine(String templateDir, long cacheMaxSize, long ttlMinutes) {
        this.templateCache = Caffeine.newBuilder()
                .maximumSize(cacheMaxSize)
                .expireAfterAccess(ttlMinutes, TimeUnit.MINUTES)
                .recordStats()
                .build(this::compileTemplate);

        this.cacheCleaner = Executors.newSingleThreadScheduledExecutor();
        cacheCleaner.scheduleAtFixedRate(
                () -> templateCache.cleanUp(),
                5, 5, TimeUnit.MINUTES
        );

        // 预热常用模板
        preloadTemplates(templateDir, "crud-generation", "unit-test", "bug-fix");
    }

    public String renderFast(String templateName, Map<String, Object> params) {
        CompiledTemplate compiled = templateCache.get(templateName);
        return compiled.render(params);
    }

    private CompiledTemplate compileTemplate(String name) {
        // 编译并缓存
        return new CompiledTemplate(name);
    }

    private void preloadTemplates(String templateDir, String... names) {
        for (String name : names) {
            templateCache.get(name);
        }
    }

    public CacheStats getCacheStats() {
        return templateCache.stats();
    }

    private static class CompiledTemplate {
        final String name;
        CompiledTemplate(String name) { this.name = name; }
        String render(Map<String, Object> params) { return ""; } // 简化
    }
}
```

---

## 八、总结

Prompt 模板引擎带来的收益：

1. **效率提升**：一行代码生成 Prompt，告别复制粘贴
2. **质量保证**：模板经过团队 Review，每个参数都有默认值
3. **版本可追溯**：Git 管理，每次改动都有记录
4. **AB 测试友好**：同一参数、不同模板版本、对比效果
5. **注入防护**：模板引擎天然转义，防止恶意输入

**核心代码量：** 不到 200 行 Java 就搭建了一个完整的 Prompt 模板引擎，包括模板渲染、缓存、Partials 支持、批量生成、版本管理。对于大多数团队来说，Mustache + Java 的组合已经足够。

---

**下一篇预告：** 《Prompt 效果评估框架：建立自动化评测流水线》—— "你优化了一版 Prompt，感觉效果变好了——是真的变好了还是心理作用？" 我们将构建一套完整的 Prompt 评测体系，包含代码正确率、质量评分、安全检测和持续优化循环。敬请期待！
