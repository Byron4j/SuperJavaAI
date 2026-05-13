# 月入5万的AI API文档生成工具，技术栈比你想象的简单——Spring Boot + GPT API

> 我做了一个AI驱动的API文档自动生成工具，月收入稳定在5万人民币。技术栈非常简单：Spring Boot + GPT API + PostgreSQL。没有LangChain，没有RAG，没有向量数据库，没有Agent。这篇文章告诉你技术不是壁垒，找准场景才是。

## 一、为什么选API文档这个方向？

2025年中，我在一家中型互联网公司做技术负责人。我们有个项目要交付给甲方，甲方要求"完整的API文档"。

我们的后端有127个REST接口，Swagger自动生成的文档有300多页。但甲方说："你这文档我看不懂，每个接口是干什么的？请求里每个字段是什么意思？什么情况下返回什么错误码？有完整的请求响应示例吗？"

没办法，只能安排3个开发花了2周时间手写API文档。3个人×2周×平均日薪约2000元 = 约12万的人力成本。

做完之后我就在想：**如果AI能自动读懂代码逻辑，生成人类可读的API文档，这个市场有多大？**

调研之后发现，需求巨大：

- 约60%的软件项目需要向甲方交付API文档
- 约40%的公司没有专职文档工程师（靠开发自己写）
- 开发写文档的效率是写代码的1/5（极其低效）
- Swagger等工具生成的文档太"技术化"，非技术人员看不懂

**这就是一个典型的高频刚需场景。**

## 二、产品功能

```
AI API文档生成工具 - DocFlow

核心功能：
├── 📄 从Java源码自动生成API文档
│   ├── Controller层接口自动识别
│   ├── 请求参数自动解析（含校验规则）
│   ├── 响应结构自动推导
│   └── 错误码自动归纳
│
├── 🤖 AI增强文档
│   ├── 接口功能描述（AI从代码逻辑推断）
│   ├── 字段业务含义解释（AI从命名和注释推断）
│   ├── 调用示例生成（含curl和代码示例）
│   └── 注意事项自动标注
│
├── 🌐 多格式导出
│   ├── Markdown（GitHub友好）
│   ├── HTML（静态网站）
│   ├── PDF（甲方交付）
│   └── OpenAPI 3.0（Swagger兼容）
│
├── 📊 文档质量检测
│   ├── 接口覆盖率检测
│   ├── 字段注释完整度
│   └── 示例代码可运行性检测
│
└── 🔄 持续更新
    ├── Git Webhook集成（代码提交自动更新文档）
    └── 变更差异对比
```

## 三、技术架构

```
┌──────────────────────────────────────────────┐
│                 Spring Boot 3.2               │
├──────────┬──────────┬──────────┬─────────────┤
│ 源码分析  │ AI增强    │ 文档生成  │ 导出服务     │
│          │          │          │             │
│ JavaParser│ GPT API  │ Template │ Markdown    │
│ AST解析  │ Prompt   │ Engine   │ HTML        │
│          │          │          │ PDF         │
│ 注解分析  │ 代码语义  │ 合并      │ OpenAPI 3   │
│          │ 分析     │          │             │
│ 调用链   │ 业务推断  │ 格式化    │             │
│ 分析     │          │          │             │
└──────────┴──────────┴──────────┴─────────────┘
          ↓              ↓
    ┌──────────┐  ┌──────────┐
    │ PostgreSQL│  │  Redis   │
    │ 项目/文档  │  │  缓存    │
    └──────────┘  └──────────┘
```

## 四、核心实现

### 4.1 源码分析——自动提取API接口信息

```java
/**
 * 从Java Controller源码中提取API接口信息
 */
@Component
public class ApiExtractor {
    
    /**
     * 扫描整个项目，提取所有REST API接口
     */
    public List<ApiEndpoint> extractAllEndpoints(String projectPath) {
        List<ApiEndpoint> endpoints = new ArrayList<>();
        
        // 1. 递归扫描所有Java文件
        List<Path> javaFiles = Files.walk(Paths.get(projectPath))
            .filter(Files::isRegularFile)
            .filter(p -> p.toString().endsWith(".java"))
            .collect(Collectors.toList());
        
        for (Path javaFile : javaFiles) {
            String sourceCode = Files.readString(javaFile);
            List<ApiEndpoint> fileEndpoints = extractEndpoints(sourceCode);
            
            // 只处理Controller类
            if (!fileEndpoints.isEmpty()) {
                endpoints.addAll(fileEndpoints);
            }
        }
        
        return endpoints;
    }
    
    /**
     * 从单个Java文件中提取API接口
     */
    private List<ApiEndpoint> extractEndpoints(String sourceCode) {
        List<ApiEndpoint> endpoints = new ArrayList<>();
        
        CompilationUnit cu = StaticJavaParser.parse(sourceCode);
        
        // 检查是否是Controller类
        Optional<ClassOrInterfaceDeclaration> classOpt = 
            cu.findFirst(ClassOrInterfaceDeclaration.class);
        
        if (classOpt.isEmpty()) return endpoints;
        
        ClassOrInterfaceDeclaration clazz = classOpt.get();
        
        // 获取类级别的@RequestMapping
        String classPath = extractRequestMappingPath(clazz);
        
        // 遍历所有方法
        clazz.getMethods().forEach(method -> {
            
            // 查找HTTP方法注解
            Optional<AnnotationExpr> requestAnnotation = 
                findRequestAnnotation(method);
            
            if (requestAnnotation.isEmpty()) return;
            
            // 构建API端点信息
            ApiEndpoint endpoint = ApiEndpoint.builder()
                .className(clazz.getNameAsString())
                .methodName(method.getNameAsString())
                .httpMethod(extractHttpMethod(requestAnnotation.get()))
                .path(classPath + extractPath(requestAnnotation.get()))
                .summary(extractSummary(method))
                .description(extractDescription(method))
                .parameters(extractParameters(method))
                .requestBody(extractRequestBody(method))
                .responseBody(extractResponseBody(method, cu))
                .security(extractSecurity(method, clazz))
                .deprecated(method.isAnnotationPresent("Deprecated"))
                .build();
            
            endpoints.add(endpoint);
        });
        
        return endpoints;
    }
    
    /**
     * 提取请求参数（含校验注解信息）
     */
    private List<ApiParameter> extractParameters(MethodDeclaration method) {
        List<ApiParameter> parameters = new ArrayList<>();
        
        method.getParameters().forEach(param -> {
            ApiParameter apiParam = ApiParameter.builder()
                .name(param.getNameAsString())
                .type(param.getTypeAsString())
                .build();
            
            // 提取参数注解
            param.getAnnotations().forEach(ann -> {
                String annName = ann.getNameAsString();
                
                switch (annName) {
                    case "RequestParam":
                        apiParam.setSource("QUERY");
                        apiParam.setRequired(
                            extractAnnotationProperty(ann, "required", "true")
                        );
                        apiParam.setDefaultValue(
                            extractAnnotationProperty(ann, "defaultValue")
                        );
                        break;
                        
                    case "PathVariable":
                        apiParam.setSource("PATH");
                        apiParam.setRequired(true);
                        break;
                        
                    case "RequestHeader":
                        apiParam.setSource("HEADER");
                        apiParam.setRequired(
                            extractAnnotationProperty(ann, "required", "true")
                        );
                        break;
                        
                    case "Valid":
                        apiParam.setValidationRequired(true);
                        break;
                }
                
                // 提取校验注解
                if (annName.equals("NotNull")) {
                    apiParam.setConstraints(
                        apiParam.getConstraints() + " | @NotNull");
                }
                if (annName.equals("NotBlank")) {
                    apiParam.setConstraints(
                        apiParam.getConstraints() + " | @NotBlank");
                }
                if (annName.equals("NotEmpty")) {
                    apiParam.setConstraints(
                        apiParam.getConstraints() + " | @NotEmpty");
                }
                if (annName.equals("Size")) {
                    apiParam.setConstraints(apiParam.getConstraints() 
                        + " | @Size(" + extractAnnotationProperty(ann, "min", "0")
                        + "," + extractAnnotationProperty(ann, "max", "∞") + ")");
                }
                if (annName.equals("Email")) {
                    apiParam.setConstraints(
                        apiParam.getConstraints() + " | @Email");
                }
            });
            
            parameters.add(apiParam);
        });
        
        return parameters;
    }
    
    /**
     * 提取响应体类型（从方法返回值和ResponseEntity泛型中）
     */
    private ApiResponse extractResponseBody(MethodDeclaration method, 
                                             CompilationUnit cu) {
        
        Type returnType = method.getType();
        String typeString = returnType.asString();
        
        ApiResponse response = new ApiResponse();
        
        // 处理 ResponseEntity<T>
        if (typeString.startsWith("ResponseEntity")) {
            response.setWrapperClass("ResponseEntity");
            
            // 提取泛型参数T（通过字符串解析简化处理）
            if (returnType.isClassOrInterfaceType()) {
                ClassOrInterfaceType cit = (ClassOrInterfaceType) returnType;
                cit.getTypeArguments().ifPresent(args -> {
                    if (!args.isEmpty()) {
                        response.setDataType(args.get(0).asString());
                    }
                });
            }
        } else if (typeString.startsWith("Result<") || typeString.startsWith("R<")) {
            response.setWrapperClass("Result");
            // 类似的泛型提取
        } else {
            response.setDataType(typeString);
        }
        
        // 提取可能的错误码
        response.setErrorCodes(extractErrorCodes(method));
        
        return response;
    }
    
    /**
     * 从方法体中提取业务错误码
     */
    private List<ErrorCode> extractErrorCodes(MethodDeclaration method) {
        List<ErrorCode> errorCodes = new ArrayList<>();
        
        method.getBody().ifPresent(body -> {
            // 搜索常见的错误码赋值模式
            String bodyStr = body.toString();
            
            // 模式：Result.fail("错误码", "错误消息")
            Pattern pattern = Pattern.compile(
                "Result\\.(?:fail|error)\\s*\\(\\s*\"([^\"]+)\"\\s*,\\s*\"([^\"]+)\""
            );
            Matcher matcher = pattern.matcher(bodyStr);
            
            while (matcher.find()) {
                errorCodes.add(ErrorCode.builder()
                    .code(matcher.group(1))
                    .message(matcher.group(2))
                    .build());
            }
            
            // 模式：throw new BusinessException("错误码", "消息")
            Pattern exPattern = Pattern.compile(
                "throw\\s+new\\s+\\w+Exception\\s*\\(\\s*\"([^\"]+)\"\\s*,\\s*\"([^\"]+)\""
            );
            Matcher exMatcher = exPattern.matcher(bodyStr);
            
            while (exMatcher.find()) {
                errorCodes.add(ErrorCode.builder()
                    .code(exMatcher.group(1))
                    .message(exMatcher.group(2))
                    .build());
            }
        });
        
        return errorCodes;
    }
}
```

### 4.2 AI增强——让文档具有人类可读性

```java
/**
 * AI增强：让技术文档变成人类可读的业务文档
 */
@Service
public class AIEnhancementService {
    
    private final RestTemplate restTemplate;
    
    @Value("${openai.api.key}")
    private String apiKey;
    
    @Value("${openai.api.url}")
    private String apiUrl;
    
    /**
     * 为API接口生成业务描述
     * 这里不是简单翻译，而是从代码逻辑推断业务含义
     */
    public String generateBusinessDescription(ApiEndpoint endpoint, 
                                               String methodBody) {
        
        String prompt = """
            你是一个专业的API文档撰写专家。请根据以下Java REST API接口的代码，
            生成面向业务人员的接口描述。
            
            # 接口信息
            HTTP方法：%s
            路径：%s
            方法名：%s
            
            # 方法体（简化版）
            ```java
            %s
            ```
            
            # 文档要求
            请用中文撰写，包含以下内容：
            
            ## 接口概述
            （1-2句话说明这个接口是干什么的，面向什么业务场景）
            
            ## 前置条件
            （调用这个接口之前需要满足什么条件？比如需要先创建订单、需要用户登录等）
            
            ## 业务流程
            （描述这个接口处理的业务逻辑流程，用普通人能理解的语言）
            
            ## 注意事项
            （调用时需要注意什么？比如幂等性要求、并发限制、超时时间等）
            """.formatted(
                endpoint.getHttpMethod(),
                endpoint.getPath(),
                endpoint.getMethodName(),
                truncateIfNeeded(methodBody, 3000)
            );
        
        return callGPT(prompt, 0.3, 1000);
    }
    
    /**
     * 为请求参数字段生成业务含义说明
     */
    public Map<String, String> generateFieldDescriptions(
            List<ApiParameter> parameters, String dtoSourceCode) {
        
        Map<String, String> fieldDescriptions = new HashMap<>();
        
        for (ApiParameter param : parameters) {
            String prompt = """
                请为以下API参数字段生成1-2句话的业务含义说明。
                
                参数名：%s
                参数类型：%s
                校验约束：%s
                是否必填：%s
                
                相关的DTO类代码：
                ```java
                %s
                ```
                
                要求：
                - 用中文说明
                - 不要重复参数名
                - 说明取值范围、格式要求（如果有）
                - 举个具体的例子
                
                示例输出风格：
                "用户手机号，必须是11位中国大陆手机号格式。示例：13800138000"
                """.formatted(
                    param.getName(),
                    param.getType(),
                    param.getConstraints(),
                    param.isRequired() ? "是" : "否",
                    dtoSourceCode != null ? dtoSourceCode : "无"
                );
            
            String desc = callGPT(prompt, 0.3, 200);
            fieldDescriptions.put(param.getName(), desc.trim());
        }
        
        return fieldDescriptions;
    }
    
    /**
     * 生成请求示例（curl命令和Java代码示例）
     */
    public CodeExamples generateCodeExamples(ApiEndpoint endpoint) {
        
        String prompt = """
            为以下API接口生成请求示例代码：
            
            HTTP方法：%s
            路径：%s
            参数：%s
            请求体类型：%s
            
            请生成两种格式：
            1. curl命令（Linux/Mac上可以直接复制执行的完整命令）
            2. Java代码示例（使用RestTemplate，包含完整的请求和响应处理）
            
            格式要求：
            ```curl
            ...
            ```
            
            ```java
            ...
            ```
            
            注意：
            - curl示例中的URL使用 https://api.example.com 作为base URL
            - 请求体中的值要真实可信（不要用 "string" "123" 这种）
            - 如果有Authorization头，使用 Bearer YOUR_TOKEN
            """.formatted(
                endpoint.getHttpMethod(),
                endpoint.getPath(),
                formatParametersForPrompt(endpoint.getParameters()),
                endpoint.getRequestBody() != null 
                    ? endpoint.getRequestBody().getTypeName() : "无"
            );
        
        String response = callGPT(prompt, 0.1, 1500);
        return parseCodeExamples(response);
    }
    
    /**
     * 统一的GPT API调用
     */
    private String callGPT(String prompt, double temperature, int maxTokens) {
        
        Map<String, Object> requestBody = new HashMap<>();
        requestBody.put("model", "gpt-4o");
        requestBody.put("messages", List.of(
            Map.of("role", "user", "content", prompt)
        ));
        requestBody.put("temperature", temperature);
        requestBody.put("max_tokens", maxTokens);
        
        HttpHeaders headers = new HttpHeaders();
        headers.setBearerAuth(apiKey);
        headers.setContentType(MediaType.APPLICATION_JSON);
        
        ResponseEntity<Map> response = restTemplate.postForEntity(
            apiUrl,
            new HttpEntity<>(requestBody, headers),
            Map.class
        );
        
        Map<String, Object> body = response.getBody();
        List<Map<String, Object>> choices = 
            (List<Map<String, Object>>) body.get("choices");
        Map<String, Object> message = 
            (Map<String, Object>) choices.get(0).get("message");
        
        return (String) message.get("content");
    }
}
```

### 4.3 文档生成与导出

```java
/**
 * 文档生成器：将API信息+AI增强合并为完整文档
 */
@Service
public class DocumentGenerator {
    
    @Autowired
    private ApiExtractor apiExtractor;
    
    @Autowired
    private AIEnhancementService aiEnhancer;
    
    /**
     * 生成完整的API文档
     */
    public ApiDocumentation generateDocumentation(Project project) {
        
        ApiDocumentation doc = new ApiDocumentation();
        doc.setProjectName(project.getName());
        doc.setVersion(project.getVersion());
        doc.setGeneratedAt(LocalDateTime.now());
        
        // 1. 提取所有API接口
        List<ApiEndpoint> endpoints = apiExtractor.extractAllEndpoints(
            project.getSourcePath()
        );
        
        // 2. 对每个接口进行AI增强
        List<EnhancedEndpoint> enhancedEndpoints = new ArrayList<>();
        
        for (ApiEndpoint endpoint : endpoints) {
            EnhancedEndpoint enhanced = EnhancedEndpoint.builder()
                .endpoint(endpoint)
                .businessDescription(
                    aiEnhancer.generateBusinessDescription(
                        endpoint, getMethodBody(project, endpoint)
                    )
                )
                .fieldDescriptions(
                    aiEnhancer.generateFieldDescriptions(
                        endpoint.getParameters(), 
                        findDtoSource(project, endpoint.getRequestBody())
                    )
                )
                .codeExamples(
                    aiEnhancer.generateCodeExamples(endpoint)
                )
                .build();
            
            enhancedEndpoints.add(enhanced);
        }
        
        // 3. 按模块分组
        Map<String, List<EnhancedEndpoint>> groupedEndpoints = 
            enhancedEndpoints.stream()
                .collect(Collectors.groupingBy(
                    e -> extractModuleName(e.getEndpoint().getClassName())
                ));
        
        doc.setModuleGroups(groupedEndpoints);
        
        // 4. 生成接口统计
        doc.setStatistics(generateStatistics(enhancedEndpoints));
        
        return doc;
    }
    
    /**
     * 导出为Markdown格式
     */
    public String exportToMarkdown(ApiDocumentation doc) {
        StringBuilder md = new StringBuilder();
        
        // 文档头部
        md.append("# ").append(doc.getProjectName())
          .append(" API 文档\n\n");
        md.append("> 版本: ").append(doc.getVersion())
          .append(" | 生成时间: ").append(doc.getGeneratedAt())
          .append(" | 接口总数: ").append(doc.getStatistics().getTotalEndpoints())
          .append("\n\n");
        
        // 目录
        md.append("## 目录\n\n");
        int moduleIndex = 1;
        for (String module : doc.getModuleGroups().keySet()) {
            md.append(moduleIndex++).append(". [").append(module)
              .append("](#").append(slugify(module)).append(")\n");
            
            for (EnhancedEndpoint endpoint : doc.getModuleGroups().get(module)) {
                md.append("    - [").append(endpoint.getEndpoint().getSummary())
                  .append("](#").append(slugify(endpoint.getEndpoint().getMethodName()))
                  .append(")\n");
            }
        }
        md.append("\n---\n\n");
        
        // 每个模块的详细内容
        for (Map.Entry<String, List<EnhancedEndpoint>> entry : 
             doc.getModuleGroups().entrySet()) {
            
            md.append("## ").append(entry.getKey()).append("\n\n");
            
            for (EnhancedEndpoint enhanced : entry.getValue()) {
                ApiEndpoint endpoint = enhanced.getEndpoint();
                
                // 接口标题
                md.append("### ").append(endpoint.getSummary()).append("\n\n");
                md.append("- **HTTP方法**: `").append(endpoint.getHttpMethod())
                  .append("`\n");
                md.append("- **路径**: `").append(endpoint.getPath())
                  .append("`\n");
                md.append("- **认证**: ").append(endpoint.getSecurity() != null 
                    ? endpoint.getSecurity() : "无").append("\n\n");
                
                // 业务描述
                md.append("#### 接口说明\n\n");
                md.append(enhanced.getBusinessDescription()).append("\n\n");
                
                // 请求参数
                if (!endpoint.getParameters().isEmpty()) {
                    md.append("#### 请求参数\n\n");
                    md.append("| 参数名 | 类型 | 必填 | 位置 | 约束 | 说明 |\n");
                    md.append("|--------|------|------|------|------|------|\n");
                    
                    for (ApiParameter param : endpoint.getParameters()) {
                        md.append("| ").append(param.getName())
                          .append(" | ").append(param.getType())
                          .append(" | ").append(param.isRequired() ? "是" : "否")
                          .append(" | ").append(param.getSource())
                          .append(" | ").append(
                              param.getConstraints() != null 
                                  ? param.getConstraints() : "-")
                          .append(" | ").append(
                              enhanced.getFieldDescriptions()
                                  .getOrDefault(param.getName(), ""))
                          .append(" |\n");
                    }
                    md.append("\n");
                }
                
                // 错误码
                if (endpoint.getResponseBody() != null 
                    && !endpoint.getResponseBody().getErrorCodes().isEmpty()) {
                    md.append("#### 错误码\n\n");
                    md.append("| 错误码 | 说明 |\n");
                    md.append("|--------|------|\n");
                    for (ErrorCode ec : endpoint.getResponseBody().getErrorCodes()) {
                        md.append("| ").append(ec.getCode())
                          .append(" | ").append(ec.getMessage())
                          .append(" |\n");
                    }
                    md.append("\n");
                }
                
                // 代码示例
                if (enhanced.getCodeExamples() != null) {
                    md.append("#### 请求示例\n\n");
                    md.append(enhanced.getCodeExamples().getCurlExample())
                      .append("\n\n");
                    md.append(enhanced.getCodeExamples().getJavaExample())
                      .append("\n\n");
                }
                
                md.append("---\n\n");
            }
        }
        
        return md.toString();
    }
}
```

### 4.4 PDF导出——利用AI生成自然语言描述

```java
/**
 * PDF导出服务
 * 使用Thymeleaf模板 + Flying Saucer渲染
 */
@Service
public class PDFExportService {
    
    @Autowired
    private TemplateEngine templateEngine;
    
    /**
     * 导出为PDF
     */
    public byte[] exportToPDF(ApiDocumentation doc) {
        
        // 1. 用Thymeleaf渲染HTML
        Context context = new Context();
        context.setVariable("doc", doc);
        context.setVariable("generatedAt", 
            doc.getGeneratedAt().format(DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss")));
        
        String html = templateEngine.process("api-doc-template", context);
        
        // 2. 用Flying Saucer将HTML转为PDF
        try (ByteArrayOutputStream baos = new ByteArrayOutputStream()) {
            ITextRenderer renderer = new ITextRenderer();
            renderer.setDocumentFromString(html);
            renderer.layout();
            renderer.createPDF(baos);
            return baos.toByteArray();
        } catch (Exception e) {
            throw new PDFGenerationException("PDF生成失败", e);
        }
    }
}
```

## 五、商业化数据

### 5.1 收入结构

```
总MRR: ¥52,000/月（约$7,200/月）

按套餐分布：
  Basic ($9/月):  180用户 × $9   = $1,620 (22%)
  Pro ($29/月):   120用户 × $29  = $3,480 (48%)
  Team ($99/月):   18用户 × $99  = $1,782 (25%)
  Enterprise:       3客户 × ≥$100 = $320  (5%)
  
按客户类型：
  中小软件公司: 62%
  外包团队:     25%
  企业内部团队: 10%
  个人开发者:    3%
```

### 5.2 成本结构

```
每月成本：约¥8,000

GPT API调用:  ¥2,500 (31%)  ← 最大成本
服务器(VPS):  ¥1,200 (15%)
域名/SSL等:     ¥300  (4%)
各类SaaS工具:  ¥500  (6%)
人力（我自己）: ¥3,500 (44%) ← 机会成本

月净利润: ¥44,000
```

### 5.3 增长速度

```
第1个月: 50注册, 8付费,  MRR $232
第2个月: 200注册, 25付费, MRR $725  (内容营销启动)
第3个月: 500注册, 80付费, MRR $2,320 (SEO开始见效)
第4个月: 1200注册, 180付费, MRR $5,220 (接近目标)
```

## 六、关键成功因素

1. **场景选择精准**：API文档是刚性需求，且传统工具真的做不好
2. **技术栈极简**：就是Spring Boot + GPT API，没有引入复杂框架
3. **定价合理**：$9/$29/$99三档，覆盖了从个人到大团队
4. **获客渠道单一但有效**：主要靠SEO（"API文档生成"等关键词）+ 知乎/CSDN内容营销
5. **部署简单**：支持SaaS云版 + Docker私有化部署（Enterprise客户最爱）

## 七、写在最后

很多程序员问我：你月入5万，是不是技术特别牛？

我的回答是：**恰恰相反，技术非常普通。关键是选对了场景。**

API文档生成这个场景有三个特征让它特别适合AI创业：
1. 输入输出明确（代码→文档），AI特别擅长
2. 需求真实且高频（每个项目都要文档）
3. 传统方案做不好（Swagger太技术化，手写太耗时）

**如果你也想做AI工具创业，先别急着想技术方案。先回答三个问题：**
1. 这个场景用户是不是真的痛？（需要文档的时候，是愿意付费的）
2. AI是不是比传统方案好10倍？（我的工具生成的文档，甲方的反馈是"比手写的还好"）
3. 用户愿意付多少钱？（$9/月不高，但对开发者来说是零花钱级别的）

三个问题都回答清楚之后，技术反而是最简单的部分。

---

*下期预告：**B04-"AI重构"能卖钱吗？我做了个Java遗留系统自动迁移工具，3个月收到20个企业客户**——遗留系统迁移是一个每年百亿规模的市场。我用AI做了一个Java版本迁移工具，能自动把Struts2 → Spring Boot、Hibernate → MyBatis-Plus、JSP → Thymeleaf。3个月收到20个企业客户的询价。*

---

> **作者简介**：某大厂Java架构师转AI技术负责人，专注Java+AI工程化落地。AI代码审查SaaS作者。关注我，每周一篇Java+AI硬核实战。
