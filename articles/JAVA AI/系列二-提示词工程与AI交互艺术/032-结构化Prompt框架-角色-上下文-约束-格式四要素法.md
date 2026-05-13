# 结构化 Prompt 框架：角色-上下文-约束-格式 四要素法，告别"AI听不懂人话"

> 阅读时间：18分钟 | 适合人群：Java开发者、所有使用AI编程工具的工程师 | 关键词：Prompt工程、四要素法、AI交互、结构化提示词

---

## 一、开篇暴击：同样一个需求，两种 Prompt，两种人生

先来看一组真实对比。

我在某项目里需要一个用户注册接口，分别用两种方式向 AI 提需求：

**Prompt A（大多数人的写法）：**

```
帮我写个注册接口
```

**AI 输出：**

```java
@RestController
public class UserController {
    @PostMapping("/register")
    public String register(@RequestBody User user) {
        userService.save(user);
        return "success";
    }
}
```

没有参数校验，没有密码加密，没有异常处理，没有统一返回体——这种代码你敢往生产环境放吗？

**Prompt B（用上四要素法之后）：**

```
【角色】你是一个资深Java后端架构师，精通Spring Boot 3.x、MyBatis-Plus、JSR303参数校验和Spring Security。

【上下文】项目技术栈：Spring Boot 3.3 + JDK 21 + MySQL 8.0 + Redis。已有统一返回体 Result<T>（code/message/data 三字段），已有全局异常处理器 GlobalExceptionHandler。遵循项目三层架构：Controller → Service → Mapper。

【约束】1. 不要使用 Lombok，手动写 getter/setter/构造器
2. 密码必须使用 BCryptPasswordEncoder 加密存储
3. 用户名和邮箱必须做唯一性校验
4. 不要生成 import 语句
5. 所有异常用项目统一的 BusinessException 抛出

【格式】只输出 Java 代码，分文件输出，不要任何解释
```

**AI 输出：** 一个完整的三层架构实现——Controller 含 `@Valid` 校验、Service 含 BCrypt 加密 + 唯一性检查 + 事务管理、全局异常处理、`Result<T>` 统一返回体。直接可以合并到主分支。

**同一个 AI，同样的问题，差距为什么这么大？**

关键就在于你的 Prompt 有没有"结构化"。别人在使用 AI 提效 10 倍，你在跟 AI 来回拉扯 10 轮——中间的差距，就是本文要讲的**四要素法**。

---

## 二、四要素法总览：RCCF 模型

四要素法的核心框架被称为 **RCCF 模型**：

```
Role（角色） → Context（上下文） → Constraints（约束） → Format（格式）
```

这四要素的关系不是并列的，而是**递进式的上下文收缩**：

- **角色** 定义了 AI 的"能力边界"——它该以什么样的知识体系来回答问题
- **上下文** 定义了"项目边界"——它应该基于哪些已有信息来工作
- **约束** 定义了"行为边界"——它能做什么、不能做什么
- **格式** 定义了"输出边界"——它应该以什么形式交付结果

每增加一个要素，AI 输出的确定性就提升一个量级。下面逐一拆解。

---

## 三、四要素逐一深度拆解

### 3.1 角色（Role）—— 给 AI 戴上"专家面具"

#### 为什么要设定角色？

LLM 在预训练阶段学习了几十种编程语言、数百种框架、无数种代码风格。如果你不给它指定角色，它会从概率最大的"通用开发者"视角来写代码——这就是为什么你拿到的是"基础版"输出。

当我写：

```
你是一个AI助手，帮我写Java代码
```

LLM 激活的是一个宽泛的"Java 知识区域"。但当我写：

```
你是一个10年经验的Java架构师，擅长高并发系统设计，精通Spring Cloud微服务架构
```

LLM 会激活更具体的知识模式：JVM 调优、线程池参数设计、分布式锁、熔断降级——这些高阶知识储备在训练数据中是与"资深架构师"这类角色标签强关联的。

#### Java 场景示例

**场景 1：生成线程池配置**

> ❌ 不写角色：
> ```
> 帮我写一个线程池
> ```
> → AI 输出：`Executors.newFixedThreadPool(10)` —— 阿里规范明确禁止！

> ✅ 写了角色：
> ```
> 你是一个熟悉《阿里巴巴Java开发手册》的Java高级工程师，帮我写一个用于异步发短信的线程池配置
> ```
> → AI 输出：手动 `new ThreadPoolExecutor`，包含 corePoolSize=CPU核数、maximumPoolSize=CPU核数×2、LinkedBlockingQueue(1000)、CallerRunsPolicy 拒绝策略——完全符合阿里规范。

**场景 2：设计 API 接口**

> ❌ 不写角色：
> ```
> 设计订单模块的API接口
> ```
> → AI 输出：随便给你几个 URL，GET/POST 随意搭配，没有版本号，没有 RESTful 规范。

> ✅ 写了角色：
> ```
> 你是一个遵循OpenAPI 3.0规范的API架构师，擅长RESTful API设计，参考GitHub和Stripe的API设计风格
> ```
> → AI 输出：`POST /api/v1/orders`、`GET /api/v1/orders/{orderId}`，请求体/响应体结构完整，包含分页参数和错误码定义。

**场景 3：代码审查**

> ❌ 不写角色：
> ```
> 帮我审查这段代码
> ```
> → AI 输出：泛泛而谈"可以考虑添加注释"、"变量命名可以更清晰"。

> ✅ 写了角色：
> ```
> 你是一个专注Java代码质量的QA专家，严格遵循SonarQube规则，特别关注空指针风险、SQL注入、线程安全、资源泄漏四个维度
> ```
> → AI 输出：逐行指出 `user.getName().length()` 有 NPE 风险，`jdbcTemplate` 未使用 try-with-resources，`StringBuilder` 在循环内拼接应改为批量操作——每条都有严重级别和修复建议。

---

### 3.2 上下文（Context）—— 给 AI 画出"项目地图"

#### 为什么上下文比角色更重要？

角色让 AI 知道"我是谁"，但上下文让 AI 知道"我在哪"。

没有上下文，AI 就像一个顶级架构师被丢进一个陌生的代码仓库——他能力再强，也只能瞎猜。我见过最常见的一个错误：不给 AI 技术栈版本，它默认给你 Spring Boot 2.x + JDK 8，但你的项目是 Spring Boot 3.x + JDK 21 + 虚拟线程。

#### 上下文应该包含哪些信息？

| 类别 | 具体内容 | 示例 |
|------|---------|------|
| 技术栈版本 | 框架及版本号 | Spring Boot 3.3.0、JDK 21、MySQL 8.0 |
| 项目结构 | 模块划分、包名 | 多模块 Maven 项目、`com.example.order` |
| 已有代码 | 关键类的接口和常量 | `Result<T>` 统一返回体、`BaseEntity` 基类 |
| 团队规范 | 命名约定、架构约束 | 三层架构、禁止在 Controller 写业务逻辑 |
| 业务背景 | 领域术语、流程 | "订单状态流转：待支付→已支付→已发货→已完成" |

#### Java 场景示例

**场景 1：缺少版本信息**

> ❌
> ```
> 帮我写一个Redis缓存配置类
> ```
> → AI 默认给你写 Jedis 客户端配置（Spring Boot 2.x 默认）。但你的项目使用的是 Spring Boot 3.x + Lettuce + Spring Cache，完全不同。

> ✅
> ```
> 项目使用 Spring Boot 3.3 + Spring Data Redis（Lettuce客户端），已有 RedisConfig 类在 com.example.config 包下。需要新增一个 CacheConfig，使用 @EnableCaching 注解，序列化方式使用 Jackson2JsonRedisSerializer
> ```
> → AI 产出可直接粘贴使用的配置类，序列化方式、连接工厂注入都匹配项目现状。

**场景 2：缺少已有代码结构**

> ❌
> ```
> 帮我写一个全局异常处理器
> ```
> → AI 给出的异常处理返回格式是 `{"error": "...", "msg": "..."}`，但你的项目用的是 `Result<T>` 统一返回体 `{"code": 500, "message": "...", "data": null}`。

> ✅
> ```
> 项目中已有统一返回体 Result<T>，字段为 code(Integer)、message(String)、data(T)。已有业务异常类 BusinessException(code, message)。请基于这些现有类写一个 GlobalExceptionHandler。
> ```
> → AI 直接使用 `Result.fail(code, message)` 格式，与项目现有代码完全一致。

**场景 3：缺少团队规范**

> ❌
> ```
> 帮我写一个查询订单列表的 Service
> ```
> → AI 给你生成 `List<Order> findByCondition(Condition cond)`，没有分页参数，返回实体类直接暴露——但你们团队规定 list 查询必须分页，且 Service 层不暴露 Entity。

> ✅
> ```
> 团队规范：1) 所有列表查询必须分页，入参统一为 PageQuery，出参为 PageResult<VO>；2) Service 层不暴露 Entity，统一返回 VO/DTO；3) 参考项目中已有的 UserService.listUsers 的实现风格
> ```
> → AI 直接输出符合团队规范的分页查询代码，连 `PageHelper.startPage()` 的调用位置都正确。

---

### 3.3 约束（Constraints）—— "不要做什么"比"要做什么"更重要

#### 为什么约束是第一生产力？

这是四要素中被低估最严重的一个。约束的本质是**排除错误答案的空间**。

LLM 在生成代码时，每一步下一个 token 都有多个候选。没有约束时，它可能选到 Lombok 的 `@Data`，也可能选 `private String name;` + 手动 getter。加入"不要使用 Lombok"后，`@Data` 直接出局，概率空间大幅收敛——准确性自然提升。

**黄金法则：约束越具体越好。**

- ❌ "写规范的代码"
- ✅ "不要使用 Lombok"、"不要生成 import 语句"、"方法超过 30 行就拆分为子方法"

#### Java 场景示例

**场景 1：Lombok 问题**

有过团队因为 Lombok 版本冲突导致编译失败的经验吗？

> ❌ 不加约束：
> ```
> 帮我写一个 User 实体类
> ```
> → AI 大概率输出：
> ```java
> @Data
> @AllArgsConstructor
> @NoArgsConstructor
> public class User { ... }
> ```

> ✅ 加约束：
> ```
> 不要使用 Lombok，手动写 getter/setter/构造器
> ```
> → AI 老老实实生成完整的手写代码，虽然代码行数多了，但编译零问题。

**场景 2：多余的类**

> ❌ 不加约束：
> ```
> 帮我写一个发送HTTP请求的工具方法
> ```
> → AI 可能给你建一个新类 `HttpUtil.java`，还附带 `HttpConfig.java`、`HttpClientFactory.java`——三个新文件。

> ✅ 加约束：
> ```
> 项目中已有 HttpUtils 类（com.example.util 包下），不要创建额外的类，只在这个类中新增一个静态方法
> ```
> → AI 直接输出一个新方法，复制粘贴即可。

**场景 3：import 语句**

> ❌ 不加约束：
> AI 生成的代码包含 15 行 import，其中 3 个是错的（不同版本的包路径），IDE 直接报红。

> ✅ 加约束：
> ```
> 不要生成 import 语句，IDE 会自动处理
> ```
> → 生成的代码干干净净，IDE 自动补全 import，不会引入错误依赖。

#### 常用的约束模板（建议收藏）

```
【技术约束】
- 不要使用 Lombok，手动写 getter/setter/构造器
- 不要使用 @Autowired 字段注入，统一使用构造器注入
- 不要在 Controller 层写任何业务逻辑
- 不要生成 import 语句

【代码质量约束】
- 方法不超过 50 行
- 圈复杂度不超过 15
- 每个 public 方法必须有参数校验
- 不要有魔法数字，统一提取为常量

【输出约束】
- 不要生成配置文件
- 不要创建额外的类
- 不要写任何注释（除非我明确要求）
```

---

### 3.4 格式（Format）—— 定义 AI 的"交付标准"

#### 为什么要约束格式？

AI 默认的输出风格是"解答型"——它会自然而然地给你解释、示例、注意事项打包输出。但有时候你要的是纯粹的代码，有时候你要的是结构化的技术文档。不约定格式，你就得从 AI 的输出里"手动提取"你想要的部分。

**一句话记住：你想要什么形式的结果，就说清楚。**

#### Java 场景示例

**场景 1：只要代码，不要解释**

> ❌
> ```
> 帮我写一个雪花算法ID生成器
> ```
> → AI 输出一大段解释："雪花算法是 Twitter 开源的分布式 ID 生成算法...它的结构是...首先我们创建一个类...这里用了 synchronized..."——800 字说明 + 50 行代码。

> ✅
> ```
> 只输出 Java 代码，不要任何解释
> ```
> → AI 直接输出 50 行纯代码。

**场景 2：结构化技术文档**

> ❌
> ```
> 帮我写订单模块的技术方案
> ```
> → AI 输出一段长文，层次不清。

> ✅
> ```
> 用以下 Markdown 格式输出：
> ## 1. 模块概述
> ## 2. 数据库设计（用表格列出字段）
> ## 3. 接口设计（方法 + URL + 请求体/响应体示例）
> ## 4. 关键流程（用文字描述步骤）
> ## 5. 注意事项
> ```
> → AI 严格按照指定结构输出，可以直接作为技术文档提交。

**场景 3：分文件输出**

> ❌
> ```
> 帮我实现用户管理的增删改查
> ```
> → AI 把所有代码混在一起输出，你需要手动拆分。

> ✅
> ```
> 按以下顺序分文件输出，每个文件用文件名作为分隔标识：
> -- UserController.java --
> （代码）
> -- UserService.java --
> （代码）
> -- UserMapper.java --
> （代码）
> ```
> → 每个文件的代码完全独立，复制到对应文件路径即可。

---

## 四、反面教材：缺失每个要素时的翻车现场

为了让你直观感受每个要素的必要性，我做了一组对照实验——同一个需求，逐一去掉四要素中的一项，看看 AI 的产出如何"沦陷"。

**需求：为已有项目新增一个"根据手机号发送短信验证码"的接口。**

### 4.1 缺少角色 —— 技能树点歪

```
只给了上下文+约束+格式，没有角色。
```

**AI 输出分析：** 代码能跑，但 `HttpURLConnection` 原生调用短信接口，没有连接池，没有超时重试，没有异常降级。这是一个"刚毕业的初级程序员"的写法——因为他没有被告知自己是一个"精通第三方API集成的高级工程师"。

### 4.2 缺少上下文 —— 闭门造车

```
只给了角色+约束+格式，没有告诉AI项目已有Redis配置和验证码工具类。
```

**AI 输出分析：** AI 自己实现了一套过期时间管理（内存 Map + ScheduledExecutorService 清理）——而项目中明明已经集成了 Redis，有现成的 `RedisUtil.setEx(key, value, ttl)` 可以用。这就是典型的"重复造轮子"，根因是 AI 不知道你的项目已经有了什么。

### 4.3 缺少约束 —— 代码"过于自由"

```
只给了角色+上下文+格式，没有约束。
```

**AI 输出分析：** 给你生成的代码用了 `@Autowired` 字段注入（而不是构造器注入），用了 `System.out.println` 打日志（而不是 SLF4J），验证码生成了 4 位数字（而不是团队规范的 6 位）。没有约束时，AI 会按"最常见"而非"最规范"的方式写代码。

### 4.4 缺少格式 —— 输出失控

```
只给了角色+上下文+约束，没有指定格式。
```

**AI 输出分析：** 输出了一篇 2000 字的技术文章——先讲验证码的原理，再讲短信平台的选型，然后是一个不完整的代码片段。你需要从中"抠"出实际可用的代码，效率极低。

---

## 五、完整综合示例：四要素法让 AI 生成"带缓存和限流的 API 调用工具类"

读完理论，来看一个完整的实战案例。这是我在实际项目中用四要素法让 AI 生成的一个生产级工具类。

### 5.1 Prompt 设计

```
【角色】
你是一个精通Java高并发和系统稳定性的资深架构师，有10年以上大型互联网项目经验，
特别擅长：HTTP客户端封装、缓存策略设计、限流算法实现、容错降级。

【上下文】
- 项目：Spring Boot 3.3 + JDK 21 电商后台系统
- 已有类：com.example.util.RedisUtil（基于 StringRedisTemplate），
  提供 setEx(key, value, seconds)、get(key)、exists(key) 方法
- 已有配置：application.yml 中定义了 app.external-api.base-url = https://api.partner.com
- 需要封装一个第三方合作伙伴 API 的调用工具类，接口返回 JSON，响应慢（平均 800ms+），
  调用有频率限制（每秒最多 10 次），数据更新频率低（每小时一次）
- 团队规范：所有工具类放在 com.example.util 包下，方法必须有完整的参数校验

【约束】
1. 不要使用 Lombok
2. 不要生成 import 语句
3. 不要创建额外的类，所有逻辑放在一个类中
4. 不要使用 @Autowired 字段注入，使用构造器注入
5. 缓存过期时间用常量定义，不要有魔法数字
6. 限流不要用第三方库（如 Guava RateLimiter），手写一个简单的滑动窗口实现
7. 所有异常统一用项目已有的 BusinessException（构造方法：BusinessException(String message)）
8. 日志统一使用 SLF4J（Logger + LoggerFactory）
9. 方法不超过 80 行
10. 不要使用 Java 21 虚拟线程（兼容 JDK 17）

【格式】
只输出 Java 代码，分以下两部分，用 ====== 分隔：
====== ApiCallUtil.java ======
（完整代码）
====== 使用示例 ======
（一个 main 方法或测试方法的示例调用代码）
```

### 5.2 AI 输出结果

```java
====== ApiCallUtil.java ======
public class ApiCallUtil {

    private static final Logger log = LoggerFactory.getLogger(ApiCallUtil.class);

    private static final long CACHE_TTL_SECONDS = 3600L;
    private static final int RATE_LIMIT_PER_SECOND = 10;
    private static final long WINDOW_SIZE_MS = 1000L;
    private static final int HTTP_TIMEOUT_MS = 5000;

    private final RedisUtil redisUtil;
    private final String baseUrl;
    private final HttpClient httpClient;
    private final ConcurrentLinkedQueue<Long> slidingWindow;

    public ApiCallUtil(RedisUtil redisUtil,
                       @Value("${app.external-api.base-url}") String baseUrl) {
        if (redisUtil == null) {
            throw new IllegalArgumentException("redisUtil must not be null");
        }
        if (baseUrl == null || baseUrl.isBlank()) {
            throw new IllegalArgumentException("baseUrl must not be blank");
        }
        this.redisUtil = redisUtil;
        this.baseUrl = baseUrl;
        this.httpClient = HttpClient.newBuilder()
                .connectTimeout(Duration.ofMillis(HTTP_TIMEOUT_MS))
                .build();
        this.slidingWindow = new ConcurrentLinkedQueue<>();
    }

    public String callApi(String apiPath) {
        if (apiPath == null || apiPath.isBlank()) {
            throw new IllegalArgumentException("apiPath must not be blank");
        }

        String cacheKey = "api:cache:" + apiPath;
        String cached = redisUtil.get(cacheKey);
        if (cached != null) {
            log.debug("Cache hit for key: {}", cacheKey);
            return cached;
        }

        checkRateLimit();

        String url = baseUrl + apiPath;
        HttpRequest request = HttpRequest.newBuilder()
                .uri(URI.create(url))
                .timeout(Duration.ofMillis(HTTP_TIMEOUT_MS))
                .GET()
                .build();

        try {
            HttpResponse<String> response = httpClient.send(
                    request, HttpResponse.BodyHandlers.ofString());
            if (response.statusCode() != 200) {
                throw new BusinessException(
                        "API call failed with status: " + response.statusCode());
            }
            String body = response.body();
            redisUtil.setEx(cacheKey, body, CACHE_TTL_SECONDS);
            log.debug("Cached response for key: {}", cacheKey);
            return body;
        } catch (IOException e) {
            throw new BusinessException("API call IO error: " + e.getMessage());
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new BusinessException("API call interrupted");
        }
    }

    private void checkRateLimit() {
        long now = System.currentTimeMillis();
        long windowStart = now - WINDOW_SIZE_MS;

        slidingWindow.add(now);

        while (true) {
            Long oldest = slidingWindow.peek();
            if (oldest == null || oldest >= windowStart) {
                break;
            }
            slidingWindow.poll();
        }

        if (slidingWindow.size() > RATE_LIMIT_PER_SECOND) {
            throw new BusinessException(
                    "Rate limit exceeded, max " + RATE_LIMIT_PER_SECOND + " requests per second");
        }
    }
}

====== 使用示例 ======
public class Demo {
    public static void main(String[] args) {
        RedisUtil redisUtil = new RedisUtil(new StringRedisTemplate());
        ApiCallUtil apiUtil = new ApiCallUtil(redisUtil,
                "https://api.partner.com");

        String result = apiUtil.callApi("/v1/product/price?sku=ABC123");
        System.out.println(result);

        String result2 = apiUtil.callApi("/v1/product/price?sku=ABC123");
        System.out.println("Second call (should be cached): " + result2);
    }
}
```

### 5.3 产出分析：四要素各自贡献了什么

| 要素 | 在这段代码中的体现 | 如果没有它 |
|------|-------------------|-----------|
| **角色** | 使用了 `HttpClient`（JDK 11+）而非老旧的 `HttpURLConnection`；手写滑动窗口限流而非无脑引 Guava | 可能用 `RestTemplate`（Spring 已废弃）或胡写线程 sleep 做限流 |
| **上下文** | 正确使用了 `RedisUtil` 的 `get/setEx` API；注入的是 `@Value` 配置的 `baseUrl` | 自己 new 一个 Jedis 连接，或把 URL 硬编码 |
| **约束** | 构造器注入、无 Lombok、无魔法数字、`BusinessException` 统一异常、80 行内 | `@Autowired` 字段注入、`@Data` 注解、`RuntimeException`、200 行长方法 |
| **格式** | 代码 + 示例清晰分隔，直接可用 | 2000 字说明 + 不完整代码片段 |

---

## 六、四要素进阶：动态上下文注入

掌握了基础四要素之后，还有一个进阶技巧：**动态上下文注入**。

### 6.1 核心思路

前面的"上下文"是静态写死在 Prompt 中的。但实际项目中，上下文会变化——pom.xml 升级了依赖、application.yml 改了配置、同事重构了某个类。每次手动同步到 Prompt 里不现实。

**解决思路：把项目文件内容动态注入到 Prompt 上下文。**

### 6.2 实战：注入 pom.xml + application.yml + 关键源代码

假设你的项目目录结构如下：

```
project/
├── pom.xml
├── src/main/resources/application.yml
├── src/main/java/com/example/
│   ├── common/Result.java
│   ├── exception/BusinessException.java
│   └── util/RedisUtil.java
```

在你给 AI 的 Prompt 后面，直接拼接以下格式的内容块：

```markdown
## 项目上下文（自动注入）

### pom.xml 关键依赖
```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.3.0</version>
</parent>
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-redis</artifactId>
    </dependency>
    <dependency>
        <groupId>com.baomidou</groupId>
        <artifactId>mybatis-plus-spring-boot3-starter</artifactId>
        <version>3.5.6</version>
    </dependency>
</dependencies>
```

### application.yml 关键配置
```yaml
spring:
  redis:
    host: localhost
    port: 6379
app:
  external-api:
    base-url: https://api.partner.com
    timeout: 5000
    rate-limit: 10
```

### 已有核心类

#### Result.java
```java
public class Result<T> {
    private Integer code;
    private String message;
    private T data;
    
    public static <T> Result<T> success(T data) { ... }
    public static <T> Result<T> fail(Integer code, String message) { ... }
}
```

#### BusinessException.java
```java
public class BusinessException extends RuntimeException {
    private Integer code;
    public BusinessException(String message) { ... }
    public BusinessException(Integer code, String message) { ... }
}
```
```

这样做的好处是：**你在 IDE 中打开文件 → 复制粘贴 → 发给 AI → 100% 匹配当前项目状态**。

### 6.3 自动化方案

如果你用的是 Cursor 或支持 `.cursorrules` / `.aide-rules` 的 AI 编程工具，可以直接把关键文件路径写入规则文件：

```
# .cursorrules
Always read and reference the following files before generating code:
- pom.xml
- src/main/resources/application.yml
- src/main/java/com/example/common/Result.java
- src/main/java/com/example/exception/BusinessException.java
```

AI 会自动读取这些文件作为上下文，不需要你手动复制粘贴。

如果你用的是普通的 ChatGPT/Claude 对话，推荐手动维护一个 `CONTEXT.md` 文件，每次要做大任务时复制粘贴。

---

## 七、总结与行动指南

### 四要素速查表

| 要素 | 解决的问题 | 关键句式 |
|------|-----------|---------|
| **角色 Role** | "AI 水平不稳定" | "你是一个 __经验的 __工程师，擅长__" |
| **上下文 Context** | "AI 写的代码不匹配项目" | "项目使用 __，已有 __类和 __配置" |
| **约束 Constraints** | "AI 写的代码能用但不够好" | "不要使用 __，必须使用 __，不要 __" |
| **格式 Format** | "AI 输出一堆废话" | "只输出 __，用 __ 格式，按 __ 分隔" |

### 你今天就可以开始做的三件事

1. **从现在开始，永远不要在 Prompt 里只说"帮我写一个XX"**。加一个角色至少让你的代码质量提升 30%。
2. **给你的每个项目维护一个上下文模板**——pom.xml 的依赖、application.yml 的核心配置、3 个最常用的内部类。每次对话前先丢给 AI。
3. **养成写约束的习惯**。先想一想"上次 AI 给我生成代码最大的坑是什么"，然后把它变成一个约束条件。

### 延伸思考

有的读者可能会问："我用的不是 ChatGPT，是 Cursor / Copilot / 通义灵码，四要素法还有用吗？"

**答案是：更有用。**

IDE 内嵌的 AI 编程工具会自动读取当前打开的文件作为上下文（相当于帮你做了"上下文"这一步），但**角色、约束、格式这三要素它没办法自动推断**。你在 Chat 面板里写的 Prompt，决定了 AI 是给你一个"基本能跑"的方案，还是一个"直接合入主干"的方案。

---

## 下篇预告

掌握了四要素法，你的 Prompt 已经不再"水"，AI 的输出准确率至少提升一倍。

但如何让 AI 解决更复杂的问题？比如："这个线上 Bug 的根因是什么？"、"这段老代码应该怎么重构？"

这就是下一篇的主题——**《Chain of Thought 思维链：让 AI 像资深工程师一样"推演"而非"猜测"》**。

你将学到：
- Zero-shot CoT：一句"让我们一步步来"的神奇魔咒
- Few-shot CoT：给 AI 两个示例，它就能按你的套路解决问题
- 自洽性（Self-Consistency）：让 AI 生成 3 种方案，投票选出最优解
- 思维树（Tree of Thoughts）：让 AI 自己探索多条路径并回溯

关注本系列，下篇见。

---

> **关于作者**：Java 技术博主，专注于 AI + Java 工程化实践。本系列文章持续更新中，欢迎评论区交流你的 Prompt 工程心得。
>
> **声明**：本文所有代码示例均基于真实项目测试，AI 工具在不同版本下的表现可能存在差异。
