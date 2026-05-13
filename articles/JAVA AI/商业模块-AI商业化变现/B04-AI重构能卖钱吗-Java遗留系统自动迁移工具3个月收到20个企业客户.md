# "AI重构"能卖钱吗？我做了个Java遗留系统自动迁移工具，3个月收到20个企业客户

> 2025年，中国仍有超过60%的大型企业核心系统运行在Java 8甚至更老的版本上。Struts2、Hibernate 3、JSP、WebLogic这些"古董级"技术栈，养活了无数外包公司。但当我用AI做了一个能自动把这些老系统迁移到Spring Boot 3 + JDK 21的工具时，3个月收到了20个企业客户的正式询价。AI重构，可能是B端最被低估的变现方向。

## 一、为什么要做这个产品？

2025年初，我参与了一个银行的遗留系统升级项目。技术栈是：

```
迁移前（2008年开发，运行了17年）：
├── JDK 6（2013年已停止支持）
├── Struts 2.0（2017年爆出致命漏洞后不再更新）
├── Hibernate 2.1（现在最新版本是6.x，差了4个大版本）
├── JSP + JSTL（十年前就过时的视图技术）
├── WebLogic 10（一台服务器每年的License费20万+）
└── Ant构建（连Maven命令都不支持）

迁移目标：
├── JDK 21（最新LTS版本）
├── Spring Boot 3.2
├── MyBatis-Plus 3.5
├── Thymeleaf
├── Docker + Kubernetes
└── Maven多模块
```

项目规模：约120万行Java代码，300+数据库表，500+个JSP页面。

团队配置：10个开发（含2个架构师），计划工期8个月，预算300万。

但实际执行时发现，大部分开发时间不是花在"重新设计"上，而是花在"机械式搬运"上——把Struts2的Action改成Spring Boot的Controller，把Hibernate的HQL改成MyBatis的Mapper XML，把JSP模板语法改成Thymeleaf语法...

**这活儿AI不能干吗？当然能。**

于是我用了一个月时间，做了一个AI驱动的遗留系统迁移引擎。核心思路是：

```
Struts2 Action源码 → JavaParser AST分析 → 提取业务逻辑 → AI语义重构 →
Spring Boot Controller → 人工审核 → 最终代码
```

## 二、核心架构

```
┌────────────────────────────────────────────────────────┐
│            AI遗留系统迁移引擎 (Migration Engine)           │
├────────────┬───────────────┬───────────────┬───────────┤
│  源码分析层  │  AI语义迁移层   │  代码生成层    │  验证层    │
│            │               │               │           │
│ AST解析    │ 框架映射       │ 代码模板       │ 编译验证   │
│ ├ Struts   │ Struts→Spring │ Controller    │ mvn compile│
│ ├ Hibernate│ HQL→MyBatis   │ Service/DAO   │           │
│ ├ JSP      │ JSP→Thymeleaf │ XML Mapper    │ 功能验证   │
│ └ 注解     │               │               │ 原测试用例  │
│            │ Prompt工程     │ 配置生成       │           │
│ 依赖分析   │ 业务逻辑重构    │ POM.xml       │ 质量报告   │
│ ├ 调用链   │ 事务管理       │ application   │ ├ 覆盖率    │
│ └ 类型依赖 │ 异常处理       │   .yml        │ └ 问题清单  │
│            │               │               │           │
└────────────┴───────────────┴───────────────┴───────────┘
```

## 三、核心代码

### 3.1 Struts2 → Spring Boot迁移引擎

```java
/**
 * Struts2到Spring Boot的自动迁移引擎
 * 核心思路：分析Struts2 Action的每一个方法和路径映射，自动生成Spring Boot Controller
 */
@Service
public class Struts2ToSpringBootMigrator {
    
    @Autowired
    private ChatLanguageModel aiModel;
    
    @Autowired
    private JavaCodeAnalyzer codeAnalyzer;
    
    /**
     * 迁移一个Struts2 Action类到Spring Boot Controller
     */
    public MigrationResult migrateActionToController(String strutsActionSource) {
        
        MigrationResult result = new MigrationResult();
        
        // Step 1: AST分析Struts2 Action
        ClassMetaInfo strutsAction = codeAnalyzer.analyze(strutsActionSource);
        
        // Step 2: 提取Struts2特有信息
        Struts2Info strutsInfo = extractStrutsInfo(strutsActionSource);
        
        // Step 3: AI语义迁移
        String springController = aiMigration(
            strutsActionSource, 
            strutsInfo, 
            MigrationType.STRUTS2_TO_SPRING
        );
        
        // Step 4: 验证编译
        CompileResult compileResult = verifyCompilation(springController);
        
        // Step 5: 生成对比报告
        result.setOriginalSource(strutsActionSource);
        result.setMigratedSource(springController);
        result.setMigrationNotes(generateNotes(strutsInfo));
        result.setCompileSuccess(compileResult.isSuccess());
        result.setCompileErrors(compileResult.getErrors());
        
        return result;
    }
    
    /**
     * 提取Struts2特有的配置信息
     */
    private Struts2Info extractStrutsInfo(String sourceCode) {
        CompilationUnit cu = StaticJavaParser.parse(sourceCode);
        
        Struts2Info info = new Struts2Info();
        
        // 查找Action类
        cu.findAll(ClassOrInterfaceDeclaration.class).forEach(clazz -> {
            
            // 检查是否继承ActionSupport
            clazz.getExtendedTypes().forEach(t -> {
                if (t.getNameAsString().contains("ActionSupport")) {
                    info.setActionType("ActionSupport");
                }
                if (t.getNameAsString().contains("BaseAction")) {
                    info.setBaseAction(t.getNameAsString());
                }
            });
            
            // 提取所有Action方法
            clazz.getMethods().forEach(method -> {
                StrutsActionMethod actionMethod = new StrutsActionMethod();
                actionMethod.setMethodName(method.getNameAsString());
                
                // 提取Struts特有的字段和方法
                method.getBody().ifPresent(body -> {
                    // 检查是否使用了Servlet API
                    body.findAll(MethodCallExpr.class).forEach(call -> {
                        String calledMethod = call.getNameAsString();
                        
                        // Struts2的ActionContext
                        if (calledMethod.equals("getRequest") 
                            || calledMethod.equals("getResponse")
                            || calledMethod.equals("getSession")) {
                            actionMethod.getStrutsDependencies().add("Servlet_API");
                        }
                        
                        // Struts2的getText（国际化）
                        if (calledMethod.equals("getText")) {
                            actionMethod.getStrutsDependencies().add("i18n");
                        }
                    });
                    
                    // 检查全局结果
                    Pattern returnPattern = Pattern.compile("return\\s+\"(\\w+)\"");
                    Matcher matcher = returnPattern.matcher(body.toString());
                    while (matcher.find()) {
                        actionMethod.getResultNames().add(matcher.group(1));
                    }
                });
                
                info.getActionMethods().add(actionMethod);
            });
        });
        
        return info;
    }
    
    /**
     * 使用AI执行语义迁移
     */
    private String aiMigration(String sourceCode, Struts2Info strutsInfo,
                                MigrationType migrationType) {
        
        String prompt = buildMigrationPrompt(sourceCode, strutsInfo);
        return aiModel.generate(prompt);
    }
    
    /**
     * 构建迁移Prompt
     */
    private String buildMigrationPrompt(String sourceCode, Struts2Info strutsInfo) {
        
        return """
            # 任务
            将以下Struts2 Action类迁移到Spring Boot 3.2 Controller。
            
            # 原始代码 (Struts2 Action)
            ```java
            %s
            ```
            
            # Struts2分析信息
            - Action类型：%s
            - 结果映射：%s
            
            # 迁移规则（严格遵守）
            
            1. **类注释转换**
               - @Namespace → @RequestMapping("/xxx")
               - @Action(value="/path") → @GetMapping/@PostMapping("/path")
               - @Result(name="success", location="/page.jsp") → return "page"
            
            2. **方法签名转换**
               - public String execute() → public String getXxx()
               - 移除ActionForm参数 → 改为@RequestParam或@RequestBody
            
            3. **Struts2 API替换**
               - ActionContext.getContext().getSession() → HttpSession参数注入
               - ServletActionContext.getRequest() → HttpServletRequest参数注入
               - getText("key") → messageSource.getMessage("key")
               - addActionError/Message → BindingResult或抛出异常
            
            4. **返回类型转换**
               - return "success"/"error"/"input" → 返回视图名或ResponseEntity
               - 保持原有的模型数据（Action的属性）→ 使用Model或@ModelAttribute
            
            5. **依赖注入修正**
               - 如果有DAO/Service字段但无@Autowired → 添加@Autowired或构造函数注入
               
            6. **异常处理**
               - catch块中的Struts2异常处理 → Spring的@ExceptionHandler
            
            7. **代码风格**
               - 使用Lombok的@Slf4j替代手动Logger
               - 方法命名遵循Spring规范
               - 添加必要的@Valid校验
            
            # 输出要求
            只输出完整的Spring Boot Controller的Java代码。
            不要解释任何内容。
            """.formatted(
                sourceCode,
                strutsInfo.getActionType() != null 
                    ? strutsInfo.getActionType() : "普通Action",
                strutsInfo.getActionMethods().stream()
                    .flatMap(m -> m.getResultNames().stream())
                    .distinct()
                    .collect(Collectors.joining(", "))
            );
    }
}
```

### 3.2 Hibernate → MyBatis-Plus迁移引擎

```java
/**
 * Hibernate到MyBatis-Plus的自动迁移
 * 最困难的部分：HQL → SQL/Mapper XML
 */
@Service
public class HibernateToMyBatisMigrator {
    
    /**
     * 迁移Hibernate DAO到MyBatis-Plus Mapper
     */
    public MigrationResult migrateDao(String hibernateDaoSource, 
                                       String entitySource) {
        
        MigrationResult result = new MigrationResult();
        
        // Step 1: 分析Hibernate DAO
        HibernateDaoInfo daoInfo = analyzeHibernateDao(hibernateDaoSource);
        
        // Step 2: 迁移实体类（Hibernate注解 → MyBatis-Plus注解）
        String mybatisEntity = migrateEntity(entitySource);
        
        // Step 3: 迁移DAO方法
        List<MapperMethod> mapperMethods = new ArrayList<>();
        
        for (DaoMethod daoMethod : daoInfo.getMethods()) {
            MapperMethod mapperMethod = migrateDaoMethod(daoMethod);
            mapperMethods.add(mapperMethod);
        }
        
        // Step 4: 生成Mapper接口和XML
        String mapperInterface = generateMapperInterface(
            daoInfo, mapperMethods
        );
        String mapperXml = generateMapperXml(
            daoInfo, mapperMethods, entitySource
        );
        
        result.setMapperInterface(mapperInterface);
        result.setMapperXml(mapperXml);
        result.setEntity(mybatisEntity);
        
        return result;
    }
    
    /**
     * HQL/JPQL到MyBatis SQL的转换
     * 这是迁移中最复杂的部分
     */
    private String convertHQLToSQL(String hql, String entitySource) {
        
        String prompt = """
            将以下HQL/JPQL查询语句转换为MyBatis兼容的SQL。
            
            # HQL语句
            ```sql
            %s
            ```
            
            # 对应的实体类
            ```java
            %s
            ```
            
            # 转换规则：
            1. 类名 → 表名（@Table注解或下划线转换）
            2. 属性名 → 列名（@Column注解或驼峰转下划线）
            3. JOIN FETCH → LEFT JOIN + SELECT中列出需要的列
            4. new com.xxx.DTO(...) → 构造ResultMap
            5. :paramName → #{paramName}
            6. in (:list) → IN <foreach>
            7. left join fetch → LEFT OUTER JOIN
            8. 移除Hibernate特有的函数调用
            
            # 输出要求
            只输出转换后的SQL语句。
            """.formatted(hql, entitySource);
        
        return aiModel.generate(prompt);
    }
    
    /**
     * 生成MyBatis Mapper XML
     */
    private String generateMapperXml(HibernateDaoInfo daoInfo, 
                                      List<MapperMethod> methods,
                                      String entitySource) {
        
        StringBuilder xml = new StringBuilder();
        xml.append("<?xml version=\"1.0\" encoding=\"UTF-8\"?>\n");
        xml.append("<!DOCTYPE mapper PUBLIC \"-//mybatis.org//DTD Mapper 3.0//EN\"\n");
        xml.append("  \"http://mybatis.org/dtd/mybatis-3-mapper.dtd\">\n");
        xml.append("<mapper namespace=\"").append(daoInfo.getPackageName())
            .append(".").append(daoInfo.getMapperName()).append("\">\n\n");
        
        // 基础ResultMap
        xml.append(generateResultMap(entitySource));
        
        // 每个方法的SQL
        for (MapperMethod method : methods) {
            xml.append("    <select id=\"").append(method.getMethodName())
                .append("\" resultMap=\"BaseResultMap\">\n");
            xml.append("        ").append(method.getSql()).append("\n");
            xml.append("    </select>\n\n");
        }
        
        xml.append("</mapper>");
        return xml.toString();
    }
}
```

### 3.3 JSP → Thymeleaf迁移

```java
/**
 * JSP到Thymeleaf的模板自动迁移
 */
@Service
public class JSPToThymeleafMigrator {
    
    /**
     * 将JSP文件内容转换为Thymeleaf HTML
     */
    public String migrateJSP(String jspContent) {
        
        // 先做确定性替换（JSP标签 → Thymeleaf属性）
        String processed = performDeterministicMapping(jspContent);
        
        // 再做AI语义迁移（处理复杂嵌套和业务逻辑）
        String finalResult = aiSemanticMigration(processed);
        
        return finalResult;
    }
    
    /**
     * 确定性映射规则（不需要AI的部分）
     */
    private String performDeterministicMapping(String jsp) {
        return jsp
            // JSTL核心标签
            .replaceAll("<c:if test=\"\\$\\{(.+?)}\">", 
                "<div th:if=\"${$1}\">")
            .replaceAll("</c:if>", "</div>")
            .replaceAll("<c:forEach items=\"\\$\\{(.+?)}\" var=\"(.+?)\">", 
                "<div th:each=\"$2 : ${$1}\">")
            .replaceAll("</c:forEach>", "</div>")
            .replaceAll("<c:choose>", "<div>")
            .replaceAll("<c:when test=\"\\$\\{(.+?)}\">", 
                "<div th:if=\"${$1}\">")
            .replaceAll("</c:when>", "</div>")
            .replaceAll("<c:otherwise>", "<div th:unless=\"...\">")
            .replaceAll("</c:otherwise>", "</div>")
            .replaceAll("</c:choose>", "</div>")
            
            // EL表达式
            .replaceAll("\\$\\{(.+?)}", "\\[\\[$1\\]\\]")
            // 注意：上面先用特有标记占位，防止下一步误替换
            
            // Struts2标签（常用）
            .replaceAll("<s:property value=\"(.+?)\"/>", 
                "<span th:text=\"${$1}\"></span>")
            .replaceAll("<s:text name=\"(.+?)\"/>", 
                "<span th:text=\"#{messages.$1}\"></span>")
            .replaceAll("<s:url action=\"(.+?)\"", 
                "<a th:href=\"@{/$1}\"")
            .replaceAll("<s:a href=\"(.+?)\">", 
                "<a th:href=\"@{/$1}\">")
            .replaceAll("</s:a>", "</a>")
            
            // 公共包含
            .replaceAll("<%@ include file=\"(.+?)\" %>",
                "<div th:replace=\"~{$1 :: content}\"></div>")
            .replaceAll("<jsp:include page=\"(.+?)\"",
                "<div th:replace=\"~{$1}\"")
            
            // 格式化日期
            .replaceAll("<fmt:formatDate value=\"\\$\\{(.+?)}\" pattern=\"(.+?)\"/>",
                "<span th:text=\"${#temporals.format($1, '$2')}\"></span>");
    }
    
    /**
     * AI语义迁移（处理复杂场景）
     */
    private String aiSemanticMigration(String jsp) {
        String prompt = """
            将以下JSP页面迁移到Thymeleaf 3.1模板。请逐行转换，保证页面功能和展示效果完全一致。
            
            # JSP源代码
            ```html
            %s
            ```
            
            # 迁移规则
            1. 所有EL表达式 ${...} → th:text="${...}", th:value="${...}"等
            2. Struts2 <s:xxx> 标签 → 标准HTML + Thymeleaf属性
            3. JSTL fmt 标签 → Thymeleaf #temporals, #numbers, #strings等
            4. 保持原有的CSS class和id不变
            5. 保留JavaScript代码不变（JSP中的JS不需要改）
            6. 国际化：<s:text name="xxx"/> → #{xxx}
            7. URL：所有链接使用 @{/path} 语法
            """.formatted(jsp);
        
        return aiModel.generate(prompt);
    }
}
```

### 3.4 迁移质量验证

```java
/**
 * 迁移质量验证器
 * 确保迁移后的代码能编译、能运行
 */
@Service
public class MigrationValidator {
    
    /**
     * 编译验证
     */
    public ValidationResult validateCompilation(String sourceCode, String className) {
        
        ValidationResult result = new ValidationResult();
        
        try {
            // 使用Java Compiler API编译验证
            JavaCompiler compiler = ToolProvider.getSystemJavaCompiler();
            DiagnosticCollector<JavaFileObject> diagnostics = new DiagnosticCollector<>();
            
            JavaFileObject file = new JavaSourceFromString(className, sourceCode);
            
            Iterable<? extends JavaFileObject> compilationUnits = List.of(file);
            JavaCompiler.CompilationTask task = compiler.getTask(
                null, null, diagnostics, 
                List.of("-source", "21", "-target", "21"),
                null, compilationUnits
            );
            
            boolean success = task.call();
            result.setCompileSuccess(success);
            
            if (!success) {
                result.setCompileErrors(
                    diagnostics.getDiagnostics().stream()
                        .map(d -> String.format("Line %d: %s", 
                            d.getLineNumber(), d.getMessage(null)))
                        .collect(Collectors.toList())
                );
            }
            
        } catch (Exception e) {
            result.setCompileSuccess(false);
            result.setCompileErrors(List.of("编译验证异常: " + e.getMessage()));
        }
        
        return result;
    }
    
    /**
     * 运行原项目的单元测试，验证迁移没有破坏功能
     */
    public ValidationResult validateTests(String projectPath) {
        
        ValidationResult result = new ValidationResult();
        
        try {
            // 使用Maven运行测试
            Process process = new ProcessBuilder(
                "mvn", "test", "-f", projectPath,
                "-Dmaven.test.failure.ignore=true"
            ).start();
            
            String output = new String(
                process.getInputStream().readAllBytes()
            );
            
            // 解析测试结果
            Pattern testsRun = Pattern.compile("Tests run: (\\d+), Failures: (\\d+), Errors: (\\d+), Skipped: (\\d+)");
            Pattern pattern = Pattern.compile("Tests run: (\\d+), Failures: (\\d+), Errors: (\\d+), Skipped: (\\d+)");
            Matcher matcher = pattern.matcher(output);
            
            int totalRun = 0, totalFailures = 0, totalErrors = 0, totalSkipped = 0;
            while (matcher.find()) {
                totalRun += Integer.parseInt(matcher.group(1));
                totalFailures += Integer.parseInt(matcher.group(2));
                totalErrors += Integer.parseInt(matcher.group(3));
                totalSkipped += Integer.parseInt(matcher.group(4));
            }
            
            result.setTotalTests(totalRun);
            result.setFailedTests(totalFailures);
            result.setErrorTests(totalErrors);
            result.setSkippedTests(totalSkipped);
            result.setPassedTests(totalRun - totalFailures - totalErrors - totalSkipped);
            result.setTestSuccess(totalFailures == 0 && totalErrors == 0);
            
        } catch (Exception e) {
            result.setTestSuccess(false);
            result.setCompileErrors(List.of("测试执行异常: " + e.getMessage()));
        }
        
        return result;
    }
}
```

## 四、商业模式

### 4.1 定价策略

```
方案         价格           适用场景
───────────────────────────────────────────
按项目报价    3-15万/项目    一次性迁移项目（主力方案）
按代码行报价  1.5-3元/行     大型项目（>100万行代码）
企业年度订阅  25万/年        持续有迁移需求的企业（含更新升级）
SaaS自助版    ¥1999/月       小团队自助使用（功能受限）
```

### 4.2 20个客户从哪来？

```
渠道来源：
  1. 技术社区（CSDN/掘金）- 发了3篇"遗留系统迁移"系列文章 → 6个客户
  2. 外包公司推荐 - 合作过的3家外包公司 → 7个客户  
  3. 行业会议 - 在一个银行科技峰会上分享 → 4个客户
  4. 熟人推荐 - 前同事在新公司负责技术 → 3个客户
```

### 4.3 典型客户案例

**案例1：某城商行核心信贷系统迁移**
- 代码量：45万行
- 报价：12万
- 实际迁移时间：5天（含人工审核）
- 对比手动迁移预估：10人×3个月 = 约60万人力成本
- 客户节省：约48万

**案例2：某政务系统升级**
- 代码量：28万行
- 报价：7.5万
- 实际迁移时间：2天
- 迁移成功率：93%（剩余7%需要手动调整，主要是业务规则硬编码的部分）

## 五、写在最后

遗留系统迁移是一个被严重低估的市场。根据Gartner的数据，全球企业在遗留系统维护上的支出每年超过1万亿美元。

而AI重构工具的市场逻辑很简单：**如果AI能帮你省10万的人力成本，你收3万，客户会很乐意付钱。**

这比做普通SaaS容易得多——SaaS你每个月收$9，10万个用户才年收入$10M。但做企业级AI重构，一个项目收10万，一年做20个项目就是200万。而你的成本几乎只有GPT API的调用费。

**B端AI工具的本质：不是卖技术，是卖"省下来的钱"。**

---

*下期预告：**B05-程序员接外包新姿势：用AI把开发效率提升10倍，利润翻5倍，甲方还夸你专业**——AI不仅改变了打工人的工作方式，也在重塑外包行业。我将展示一个真实案例：一个人用AI把原本需要5人团队的项目3周搞定，利润率从15%提升到75%。*

---

> **作者简介**：某大厂Java架构师转AI技术负责人，专注Java+AI工程化落地。AI代码审查SaaS作者，遗留系统迁移专家。关注我，每周一篇Java+AI硬核实战。
