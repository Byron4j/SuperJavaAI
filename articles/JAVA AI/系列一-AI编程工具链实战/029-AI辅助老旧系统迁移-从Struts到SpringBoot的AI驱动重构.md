# AI 辅助老旧系统迁移：从 Struts 到 Spring Boot 的 AI 驱动重构，祖传代码重获新生

## 一、开篇：那个没人敢动的"老祖宗"

"小张啊，公司那个 2008 年的 Struts 项目，10 万行代码，36 个模块，你带两个人，3 个月内迁移到 Spring Boot，没问题吧？"

你说你内心不是崩溃的，我都不信。

打开项目一看——`web.xml` 配置了 800 多行，`struts-config.xml` 更是直逼 3000 行，Action 类里混杂着 JDBC 裸写 SQL、手写线程池、甚至还有 15 年前的注释 `// TODO: 这个Bug下个月改`……而"下个月"指向的是 2010 年。

更绝的是，原来的开发团队已经换了好几茬，懂这套代码的同事要么离职要么转岗。你面对的是一座没有地图、没有说明书、还埋满地雷的代码遗迹。

**传统做法是什么？** 5 个人，逐行阅读、手工重写、反复调试。三个月？半年能跑通就烧高香了。

但现在是 2026 年，你手上有 AI。

这篇文章，我会完整复盘我们团队用 AI 辅助将一个 10 万行 Struts 项目迁移到 Spring Boot 的全过程——包括每个阶段的具体 Prompt、转换前后的代码对比，以及那些 AI 也搞不定的"硬骨头"。

---

## 二、迁移总策略：渐进式，而非大爆炸

先记住一个铁律：**永远不要尝试"一夜重写"。** 你以为的"全量迁移冲刺"× "停产三个月事故"。

我们的策略是**绞杀者模式（Strangler Fig Pattern）**：

1. **不动现有系统**，新建一个 Spring Boot 外壳
2. 通过反向代理，把新系统中的模块逐步"劫持"过来
3. 每迁移一个模块，新旧系统并行运行至少一周
4. 验证通过后，Struts 端的相关 Action 标记为废弃、下线

整个迁移拆分为 5 个 Phase，AI 在每个 Phase 承担不同的角色：

```
Phase 1：代码分析    → AI 扫描项目结构，生成模块依赖图和迁移优先级
Phase 2：模块迁移    → AI 将 Struts Action 转换为 Spring Controller
Phase 3：配置迁移    → AI 将 web.xml/struts-config.xml 转换为 Spring Boot 配置
Phase 4：数据层迁移  → AI 将 JDBC Template 裸写 SQL 转换为 MyBatis-Plus
Phase 5：测试验证    → AI 生成兼容性测试，保证新旧实现行为一致
```

下面逐 Phase 拆解。

---

## 三、Phase 1：代码分析——让 AI 帮你读懂"祖传代码"

### 目标

搞清楚三件事：
- 项目结构是什么（模块划分、依赖关系）
- 哪些模块最独立、最容易迁移（迁移优先级排序）
- 哪些模块与核心业务耦合最紧（需要特殊对待）

### Prompt 模板

```
你是一个资深 Java 架构师。请分析以下 Struts 1.x 项目的目录结构，
完成以下任务：

1. 识别项目的模块划分（按功能而非目录）
2. 生成模块依赖关系图（用 Mermaid 语法）
3. 标记每个模块的迁移难度（1-5星，5星最难）
4. 给出推荐的迁移顺序（先易后难）
5. 列出所有引用到的外部 jar 包及其对应的 Spring Boot 替代方案

项目结构如下：
[贴入 tree 命令输出或项目目录结构]

关键文件内容：
struts-config.xml: [贴入内容]
web.xml: [贴入内容]
```

### AI 输出示例

AI 会给你类似这样的分析结果：

```
模块依赖关系图（Mermaid）：
graph TB
    A[登录认证模块] --> C[权限校验 Filter]
    B[用户管理模块] --> A
    D[订单管理模块] --> B
    D --> E[库存模块]
    ...

迁移优先级：

| 优先级 | 模块 | 难度 | 理由 |
|--------|------|------|------|
| P0 | 日志拦截器 | ★★ | 独立组件，无业务耦合 |
| P1 | 用户管理   | ★★★ | 依赖登录，但登录可暂时保留 |
| P2 | 文件上传   | ★★★ | 涉及临时目录、大小限制等配置 |
| P3 | 订单管理   | ★★★★ | 核心业务，耦合多个子模块 |
| P5 | 工作流引擎 | ★★★★★ | 自定义状态机，需深度重写 |

Jar 包替代方案：
struts-core-1.3.10.jar  → spring-boot-starter-web
commons-dbutils-1.3.jar  → mybatis-plus-boot-starter+mysql-connector
custom-cache-0.0.1.jar   → spring-boot-starter-cache + Caffeine
```

### 这个阶段的关键价值

没有 AI 之前，你至少要花 **2-3 周** 去通读代码、画依赖图、评估工作量。现在**1-2 天**即可完成。而且 AI 不会遗漏边角模块——人类很容易在翻了几百个文件后产生"阅读疲劳"。

> **实战心得**：如果项目太大（超过 200 个类），不要一次性喂给 AI。先按目录拆成 4-5 批，每批生成一份分析报告，再用"汇总" Prompt 让 AI 合并。

---

## 四、Phase 2：模块迁移——Struts Action → Spring Controller

这是整个迁移工程的主体，也是最考验 AI 能力的地方。

### 迁移前 vs 迁移后

**迁移前（典型的 Struts 1.x Action）**：

```java
// LoginAction.java —— Struts 1.x 风格
public class LoginAction extends Action {

    @Override
    public ActionForward execute(ActionMapping mapping, ActionForm form,
            HttpServletRequest request, HttpServletResponse response) {

        LoginForm loginForm = (LoginForm) form;
        String username = loginForm.getUsername();
        String password = loginForm.getPassword();
        String verifyCode = loginForm.getVerifyCode();

        // 验证码校验
        String sessionCode = (String) request.getSession().getAttribute("RAND_CODE");
        if (sessionCode == null || !sessionCode.equalsIgnoreCase(verifyCode)) {
            request.setAttribute("error", "验证码错误");
            return mapping.findForward("loginFail");
        }

        // 手工获取数据源（你没看错，Action里直接获取DataSource）
        DataSource ds = getDataSource(request, "jdbc/mysql");
        try (Connection conn = ds.getConnection()) {
            String sql = "SELECT * FROM t_user WHERE username=? AND password=?";
            PreparedStatement ps = conn.prepareStatement(sql);
            ps.setString(1, username);
            ps.setString(2, password); // MD5都没加，直接明文
            ResultSet rs = ps.executeQuery();

            if (rs.next()) {
                HttpSession session = request.getSession();
                session.setAttribute("USER_INFO", rs);
                return mapping.findForward("loginSuccess");
            } else {
                request.setAttribute("error", "用户名或密码错误");
                return mapping.findForward("loginFail");
            }
        } catch (SQLException e) {
            e.printStackTrace();
            return mapping.findForward("error");
        }
    }
}
```

### Prompt 模板（逐个模块迁移）

```
请将以下 Struts 1.x 的 LoginAction 迁移为 Spring Boot 风格的代码。
要求：

1. 将 ActionForm 转换为 @RequestBody DTO 或 @RequestParam
2. 将 ActionMapping.findForward 转换为 @RestController 的方法返回
3. 将数据源获取改为依赖注入（@Autowired）
4. JDBC 裸写 SQL 暂时保留，仅改变获取 Connection 的方式
5. 验证码校验逻辑保持不变
6. 生成对应的 Controller、Service、DTO 类
7. 密码暂不做加密处理，保留原逻辑并在代码注释中标明"TODO: 接入加密"

原始代码：
[贴入上面的 LoginAction.java]
```

### AI 输出（迁移后代码）

```java
// ============ LoginController.java ============
@RestController
@RequestMapping("/login")
public class LoginController {

    @Autowired
    private LoginService loginService;

    @PostMapping
    public ResponseEntity<LoginResponse> login(@RequestBody LoginRequest request,
            HttpSession session) {

        if (request.getVerifyCode() == null
                || !request.getVerifyCode().equalsIgnoreCase(
                    (String) session.getAttribute("RAND_CODE"))) {
            return ResponseEntity.ok(
                LoginResponse.fail("验证码错误"));
        }

        LoginResult result = loginService.login(request.getUsername(),
            request.getPassword());

        if (result.isSuccess()) {
            session.setAttribute("USER_INFO", result.getUser());
            return ResponseEntity.ok(LoginResponse.success());
        } else {
            return ResponseEntity.ok(
                LoginResponse.fail("用户名或密码错误"));
        }
    }
}

// ============ LoginRequest.java ============
@Data
public class LoginRequest {
    @NotBlank(message = "用户名不能为空")
    private String username;
    @NotBlank(message = "密码不能为空")
    private String password;
    private String verifyCode;
}

// ============ LoginResponse.java ============
@Data
public class LoginResponse {
    private int code;
    private String message;

    public static LoginResponse success() {
        LoginResponse r = new LoginResponse();
        r.setCode(200);
        r.setMessage("success");
        return r;
    }

    public static LoginResponse fail(String msg) {
        LoginResponse r = new LoginResponse();
        r.setCode(400);
        r.setMessage(msg);
        return r;
    }
}

// ============ LoginService.java ============
@Service
public class LoginService {

    @Autowired
    private JdbcTemplate jdbcTemplate; // 先保留 JDBC，Phase 4 再迁移

    public LoginResult login(String username, String password) {
        // TODO: Phase 4 迁移到 MyBatis-Plus
        String sql = "SELECT * FROM t_user WHERE username=? AND password=?";
        // TODO: 接入密码加密，当前为明文比较
        List<Map<String, Object>> rows = jdbcTemplate.queryForList(
            sql, username, password);
        if (rows.isEmpty()) {
            return LoginResult.fail();
        }
        return LoginResult.success(rows.get(0));
    }
}
```

### 关键小结

对比写法的变化：

| 维度 | Struts 1.x | Spring Boot |
|------|-----------|-------------|
| 参数接收 | ActionForm 手动转型 | @RequestBody + DTO |
| 页面跳转 | ActionMapping.findForward | ResponseEntity JSON 返回 |
| 数据源 | Action 内 getDataSource | @Autowired 注入 |
| 异常处理 | e.printStackTrace() | @ControllerAdvice 全局处理 |
| 事务管理 | 手动 conn.commit() | @Transactional |

每个模块的迁移基本沿用这个套路，**36 个模块中，AI 帮我们完成了约 70% 的代码转换**，剩下 30% 是需要人工介入的复杂业务逻辑。

---

## 五、Phase 3：配置迁移——web.xml → Spring Boot 配置

### 场景

一个典型的 Struts 1.x 项目的 `web.xml` 动辄 500-1000 行，内容极其庞杂：

```xml
<!-- web.xml（节选） -->
<servlet>
    <servlet-name>action</servlet-name>
    <servlet-class>org.apache.struts.action.ActionServlet</servlet-class>
    <init-param>
        <param-name>config</param-name>
        <param-value>/WEB-INF/struts-config.xml</param-value>
    </init-param>
    <load-on-startup>1</load-on-startup>
</servlet>
<servlet-mapping>
    <servlet-name>action</servlet-name>
    <url-pattern>*.do</url-pattern>
</servlet-mapping>

<filter>
    <filter-name>encodingFilter</filter-name>
    <filter-class>org.springframework.web.filter.CharacterEncodingFilter</filter-class>
    <init-param>
        <param-name>encoding</param-name>
        <param-value>UTF-8</param-value>
    </init-param>
</filter>
<filter>
    <filter-name>loginCheckFilter</filter-name>
    <filter-class>com.oldcompany.filter.LoginCheckFilter</filter-class>
</filter>
<filter-mapping>
    <filter-name>loginCheckFilter</filter-name>
    <url-pattern>*.do</url-pattern>
</filter-mapping>

<resource-ref>
    <res-ref-name>jdbc/mysql</res-ref-name>
    <res-type>javax.sql.DataSource</res-type>
    <res-auth>Container</res-auth>
</resource-ref>

<session-config>
    <session-timeout>30</session-timeout>
</session-config>
<welcome-file-list>
    <welcome-file>index.jsp</welcome-file>
</welcome-file-list>
<error-page>
    <error-code>404</error-code>
    <location>/error/404.jsp</location>
</error-page>
```

### Prompt

```
请将以下 web.xml 和 struts-config.xml 转换为 Spring Boot 的配置方式。
要求：

1. Servlet 配置 → 无需显式配置，Spring Boot 自动配置 DispatcherServlet
2. Filter 配置 → 转换为 @Component + @WebFilter 或 FilterRegistrationBean
3. Filter 映射规则 → 转换为 Spring Security 或自定义 Filter 的 URL 匹配
4. 资源引用 → 转换为 application.yml 中的 datasource 配置
5. session-config → 转换为 application.yml 中的 server.servlet.session.timeout
6. error-page → 转换为 @ControllerAdvice 全局异常处理
7. ActionMapping → 转换为 Controller 的 @RequestMapping 注解

原始 web.xml：
[贴入内容]
```

### AI 输出示例

```java
// ============ WebConfig.java ============
@Configuration
public class WebConfig {

    /**
     * 字符编码 Filter（原 encodingFilter）
     * Spring Boot 已经有 CharacterEncodingFilter Bean，
     * 只需在 application.yml 配置即可
     */
    @Bean
    public FilterRegistrationBean<LoginCheckFilter> loginCheckFilter() {
        FilterRegistrationBean<LoginCheckFilter> reg = new FilterRegistrationBean<>();
        reg.setFilter(new LoginCheckFilter());
        reg.addUrlPatterns("/api/*");
        reg.setOrder(1);
        return reg;
    }
}

// ============ LoginCheckFilter.java ============
@Component
@WebFilter(urlPatterns = "/api/*")
public class LoginCheckFilter implements Filter {

    @Override
    public void doFilter(ServletRequest request, ServletResponse response,
            FilterChain chain) throws IOException, ServletException {
        HttpServletRequest req = (HttpServletRequest) request;
        String uri = req.getRequestURI();

        // 白名单
        if (uri.contains("/login") || uri.contains("/register")) {
            chain.doFilter(request, response);
            return;
        }

        HttpSession session = req.getSession(false);
        if (session == null || session.getAttribute("USER_INFO") == null) {
            HttpServletResponse resp = (HttpServletResponse) response;
            resp.setContentType("application/json;charset=UTF-8");
            resp.getWriter().write("{\"code\":401,\"message\":\"未登录\"}");
            return;
        }
        chain.doFilter(request, response);
    }
}

// ============ GlobalExceptionHandler.java ============
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(AuthenticationException.class)
    @ResponseStatus(HttpStatus.UNAUTHORIZED)
    public ApiResponse<Void> handleAuth(AuthenticationException e) {
        return ApiResponse.fail(401, e.getMessage());
    }

    // 原 error-page 404 → 统一 JSON 返回
    @RequestMapping("/error")
    public ApiResponse<Void> handleError(HttpServletRequest request) {
        int status = (Integer) request.getAttribute(
            "javax.servlet.error.status_code");
        return ApiResponse.fail(status, "请求出错");
    }
}
```

```yaml
# ============ application.yml ============
server:
  servlet:
    session:
      timeout: 30m  # 替代 web.xml 中的 <session-timeout>30</session-timeout>
  error:
    whitelabel:
      enabled: false # 关闭默认白标签错误页

spring:
  datasource:
    url: jdbc:mysql://localhost:3306/old_db?useSSL=false&serverTimezone=Asia/Shanghai
    username: root
    password: ${DB_PASSWORD}
    driver-class-name: com.mysql.cj.jdbc.Driver
    hikari:
      minimum-idle: 5
      maximum-pool-size: 20
```

### 这个阶段的核心坑

AI 对 XML 到 JavaConfig 的语法转换做得非常好。但以下两点必须人工介入：

1. **Filter 的执行顺序**：`web.xml` 中是按声明顺序执行，Spring Boot 中用 `@Order` 或 `FilterRegistrationBean.setOrder()` 控制。AI 会保留顺序映射，但你需要确认原系统中 Filter 之间是否有隐式依赖（比如 A Filter 在 request 里塞了某个属性，B Filter 读取这个属性；`web.xml` 的顺序保证了 A 在 B 前面）。
2. **JNDI 资源引用**：原项目中使用 `java:comp/env/jdbc/mysql` 这种 JNDI 资源引用，迁移到 Spring Boot 后一般改为 `application.yml` 直接配置。但如果原系统部署在 WebLogic/WebSphere 等应用服务器上，JNDI 绑定是由运维配置的——你需要确认是否继续使用 JNDI。

---

## 六、Phase 4：数据层迁移——JDBC Template → MyBatis-Plus

旧项目中数据访问层的情况通常惨不忍睹：
- `JdbcTemplate` / `commons-dbutils` / 裸 `Connection`
- SQL 散落在 Action、Service、JSP 页面的 `<% %>` 脚本里
- 有的地方用拼接字符串，有的用 `PreparedStatement`，风格七零八落

### Prompt

```
请将以下使用 JdbcTemplate 的数据访问代码迁移为 MyBatis-Plus 风格。
要求：

1. 生成对应的实体类（Entity），使用 @TableName 和 @TableId 等注解
2. 生成 Mapper 接口，继承 BaseMapper<T>
3. 复杂 SQL 使用 @Select/@Update 注解或 XML Mapper
4. 保留原有业务逻辑，不改变 SQL 语义
5. 如果原代码中有分页查询，使用 MyBatis-Plus 的 Page 对象
6. 事务管理改为 @Transactional 注解

原始代码：
[贴入 JdbcTemplate 代码片段]
```

### 迁移前 vs 迁移后

**迁移前**：

```java
@Service
public class OrderService {

    @Autowired
    private JdbcTemplate jdbcTemplate;

    public List<Map<String, Object>> queryOrders(String status, int page, int size) {
        // 拼接 SQL（注意这里有 SQL 注入风险）
        String sql = "SELECT * FROM t_order WHERE 1=1 ";
        if (status != null && !status.isEmpty()) {
            sql += " AND status='" + status + "'"; // 经典 SQL 注入
        }
        sql += " LIMIT " + (page - 1) * size + ", " + size;
        return jdbcTemplate.queryForList(sql);
    }

    public int createOrder(Map<String, Object> orderMap) {
        String sql = "INSERT INTO t_order(order_no, user_id, amount, status, create_time) "
            + "VALUES(?,?,?,?,?)";
        return jdbcTemplate.update(sql,
            orderMap.get("orderNo"),
            orderMap.get("userId"),
            orderMap.get("amount"),
            orderMap.get("status"),
            new Date());
    }
}
```

**迁移后**：

```java
// ============ Order.java ============
@Data
@TableName("t_order")
public class Order {
    @TableId(type = IdType.AUTO)
    private Long id;
    private String orderNo;
    private Long userId;
    private BigDecimal amount;
    private String status;
    private Date createTime;
    private Date updateTime;
}

// ============ OrderMapper.java ============
@Mapper
public interface OrderMapper extends BaseMapper<Order> {

    @Select("SELECT * FROM t_order WHERE status = #{status}")
    List<Order> selectByStatus(@Param("status") String status);

    // 带条件的动态查询
    @Select("<script>" +
            "SELECT * FROM t_order WHERE 1=1" +
            "<if test='status != null and status != \"\"'>" +
            " AND status = #{status}" +
            "</if>" +
            " ORDER BY create_time DESC" +
            "</script>")
    IPage<Order> selectPage(Page<Order> page, @Param("status") String status);
}

// ============ OrderService.java ============
@Service
public class OrderService {

    @Autowired
    private OrderMapper orderMapper;

    public IPage<Order> queryOrders(String status, int pageNum, int pageSize) {
        Page<Order> page = new Page<>(pageNum, pageSize);
        return orderMapper.selectPage(page, status);
    }

    @Transactional(rollbackFor = Exception.class)
    public boolean createOrder(Order order) {
        order.setCreateTime(new Date());
        return orderMapper.insert(order) > 0;
    }
}
```

### 这个 Phase 的核心价值

- **消除 SQL 注入风险**：拼接 SQL → 参数化查询
- **统一数据访问范式**：零散的 JDBC → 统一的 Mapper + MyBatis-Plus CRUD
- **自动分页**：手动 count + limit → `Page` 对象一行搞定

> **小贴士**：旧项目中有大量 `SELECT *`。用 AI 迁移完 SQL 后，再用一条 Prompt 收尾：_"请将以下 Mapper XML 中的 SELECT * 替换为明确的字段列表，字段名需与 Entity 中的 @TableField 对齐"_。这一步对后续代码维护和性能分析非常有帮助。

---

## 七、Phase 5：测试验证——让 AI 生成兼容性测试

迁移最怕的事情是什么？**功能跑通了，但你不知道它是不是"对"的。** 某条 SQL 的条件写错了、某个 if 分支漏掉了……这些"软 Bug"线上发现就是事故。

我们的做法是：**给新旧系统发同样的请求，对比返回结果。**

### Prompt

```
请为新系统的以下 Controller 生成兼容性测试。

测试策略：
1. 使用 MockMvc 或 WebTestClient 发起 HTTP 请求
2. 准备测试数据（使用 @Sql 注解加载测试数据脚本）
3. 验证 HTTP 状态码
4. 验证响应 JSON 的关键字段
5. 验证边界条件（空参数、异常参数、超长字符串等）
6. 生成对应的测试数据 SQL 脚本（insert_test_data.sql）

目标 Controller：
[贴入 Controller 代码]
```

### AI 生成的测试代码

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@AutoConfigureMockMvc
@TestMethodOrder(MethodOrderer.OrderAnnotation.class)
public class LoginControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @Test
    @Order(1)
    @Sql(scripts = "/sql/insert_test_login_data.sql")
    void testLoginSuccess() throws Exception {
        LoginRequest request = new LoginRequest();
        request.setUsername("admin");
        request.setPassword("123456");
        request.setVerifyCode("ABCD");

        // Mock 验证码
        mockMvc.perform(post("/login")
                .contentType(MediaType.APPLICATION_JSON)
                .content(JsonUtil.toJson(request))
                .sessionAttr("RAND_CODE", "ABCD"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.code").value(200))
                .andExpect(jsonPath("$.message").value("success"))
                .andExpect(request().sessionAttribute("USER_INFO",
                    notNullValue()));
    }

    @Test
    @Order(2)
    void testLoginWithWrongVerifyCode() throws Exception {
        LoginRequest request = new LoginRequest();
        request.setUsername("admin");
        request.setPassword("123456");
        request.setVerifyCode("WRONG");

        mockMvc.perform(post("/login")
                .contentType(MediaType.APPLICATION_JSON)
                .content(JsonUtil.toJson(request))
                .sessionAttr("RAND_CODE", "ABCD"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.code").value(400))
                .andExpect(jsonPath("$.message").value("验证码错误"));
    }

    @Test
    @Order(3)
    void testLoginWithEmptyUsername() throws Exception {
        LoginRequest request = new LoginRequest();
        request.setUsername("");
        request.setPassword("123456");

        mockMvc.perform(post("/login")
                .contentType(MediaType.APPLICATION_JSON)
                .content(JsonUtil.toJson(request)))
                .andExpect(status().isBadRequest());
    }
}
```

```sql
-- insert_test_login_data.sql
INSERT INTO t_user(username, password, real_name, status, create_time)
VALUES ('admin', '123456', '管理员', 'ACTIVE', NOW());
```

### 兼容性对比测试的终极手段

我们还写了一个"对比测试 runner"——给旧系统和新系统各发一条请求，用 AI 对比返回的 JSON：

```
请对比以下两个 JSON 响应（旧系统 vs 新系统），判断它们是否在语义上等价。
忽略以下差异：
- 时间戳字段
- JSON 字段顺序
- 旧系统返回的额外字段（如 strutsAction）
- null vs 空字符串（视为等价）

旧系统响应：[贴入]
新系统响应：[贴入]

如果等价，输出 "PASS"；如果不等价，指出具体差异字段及其值。
```

这个对比 Prompt 救了我们不止一次——有一次新系统里 `amount` 字段的精度变成了科学计数法，肉眼检查 JSON 时完全没发现，是 AI 的差异检测揪出来的。

---

## 八、真实痛点清单：迁移中 AI 搞不定的 5 件事

说完 AI 能干的事，必须诚实地聊聊 AI 搞不定的那些坑。如果你以为把代码丢给 AI 就能全自动迁移，那这五个坑会让你怀疑人生。

### 1. Session 管理的"非标准用法"

AI 理解的 Session 是 HTTP Session。但在老旧系统中，我们见过：
- 将 Session 对象存到文件系统做"持久化"
- 跨 JVM 通过 RMI 共享 Session
- 在 `HttpSessionListener` 里维护一个全局 Map 做"在线用户统计"

这些"骚操作"在 AI 的认知范围之外——它会按标准模式生成代码，完全忽略你的非标准依赖。

**解法**：Phase 1 代码分析时，专门搜索 `HttpSession`、`session.setAttribute`、`session.invalidate` 等关键字，人工评估 Session 的使用模式，再决定用 Spring Session（Redis）还是特殊处理。

### 2. 自定义 Filter 链

前面 Phase 3 提到了 Filter 转换。但如果原系统有一个长达 8 层的 Filter 链，且 Filter 之间存在隐式的 request attribute 传递，AI 无法推断这种运行时的"暗通信"。

**实际案例**：我们的旧项目中，`AuthFilter` 在验证通过后向 request 塞了一个 `com.oldcompany.USER_ROLE` 属性，`AuditFilter` 读取这个属性记录操作日志，`PermissionFilter` 又依赖 AuditFilter 先执行完毕。这个依赖关系在 `web.xml` 中只是声明顺序，代码里没有显式依赖——AI 看不出来。

**解法**：画 Filter 调用时序图，标注每个 Filter 对 request 的读写操作。这个活 AI 帮不了，得靠你拿着 `web.xml` 和 Filter 源码一步步梳理。

### 3. XML SPI 和自定义扩展点

很多老系统用 XML 做 SPI 扩展——搞一个 `plugins.xml`，里面注册了一堆实现了某个接口的类，然后用 `Class.forName()` 反射加载。

```xml
<!-- plugins.xml -->
<plugins>
    <plugin name="smsSender" class="com.oldcompany.plugin.SMSSender"/>
    <plugin name="emailSender" class="com.oldcompany.plugin.EmailSender"/>
</plugins>
```

这种自定义的 SPI 机制，AI 完全不知道它是什么、怎么工作、也不了解各个 plugin 之间的调用顺序和依赖关系。

**解法**：人工重写为 Spring 的 `@Component` + `@ConditionalOnProperty` 配置化加载。这块代码量不大但逻辑复杂，必须手动处理。

### 4. 遗留的业务 Bug

迁移过程中最纠结的问题：**原系统的一个 if-else 分支，你发现它明显有逻辑错误，但这个"错误"已经线上跑了 10 年，所有下游系统都依赖这个"错误的行为"。**

AI 在迁移代码时，会忠实地复制业务逻辑——包括 Bug。如果你让 AI "优化"这部分逻辑，它可能把 Bug 一起"优化"掉，导致新系统和下游对接时行为不一致。

**解法**：
- Phase 1 就把所有已知 Bug 列成清单
- 迁移策略定为"先完全复制行为，再做 Bug 修复"
- 每一个 Bug 的修复单独 PR、单独上线，便于回滚

### 5. 缺失的上下文和环境

AI 能看到的只是你喂给它的代码。项目用的是什么版本的 JDK？跑在什么操作系统上？数据库是什么版本？用的哪家云厂商的什么中间件？

这些"环境上下文"你心里清楚，但 AI 不知道。比如：
- 原系统依赖了某个非 Maven 仓库的私有 JAR 包，AI 会默认它是 Maven 中央仓库的包
- 原系统的定时任务依赖 cron 表达式 + Linux crontab，AI 会生成 `@Scheduled` 注解，但你实际用的是 XXL-Job

**解法**：迁移前，用 Markdown 文档形式把"项目环境上下文"整理好，每次 Prompt 都贴在前面。

---

## 九、总结：AI 不是银弹，但它是三头六臂

回顾整个迁移过程：

| 传统方式 | AI 辅助 |
|----------|---------|
| 5 人 × 4 个月 | 3 人 × 2.5 个月 |
| 手工逐行阅读代码 | AI 批量分析、生成依赖图 |
| 手工重写每一行 | AI 完成 70% 代码转换 |
| 手工写测试覆盖 | AI 自动生成测试骨架 |
| 反复"第 N 次理解错误" | AI 的统一风格转换消除低级错误 |

**用时对比**：传统估算 20 人月 → 实际 7.5 人月，效率提升约 2.6 倍。

**AI 迁移三原则**：

1. **大处着眼，小处着手**：先让 AI 做全局分析，再逐个模块迁移
2. **先复制行为，再说优化**：迁移过程中不做"顺便优化"，保证行为一致是第一优先级
3. **人不离铃铛**：AI 生成的每一段代码都要 Code Review。相信我，它会在你意想不到的地方犯低级错误

这篇文章提到的 Prompt 和代码模板，你可以直接复制到你的 AI 编程工具中使用。如果你也在做老旧系统的迁移，欢迎在评论区交流你踩过的坑——毕竟，谁的祖传代码背后没有几段让人哭笑不得的故事呢？

---

*下一篇预告：《打造你自己的 AI 编程工作流：从需求到上线的全链路自动化》——我会分享一个完整的、可复制的每日 AI 编程 SOP，让你的开发效率再上一个台阶。*
