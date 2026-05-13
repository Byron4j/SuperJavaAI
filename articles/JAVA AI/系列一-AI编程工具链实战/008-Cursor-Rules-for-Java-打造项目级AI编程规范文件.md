# Cursor Rules for Java：打造项目级 AI 编程规范文件，让Cursor写出你想要的代码

> 同样是生成一个订单接口，默认 Cursor 写出来的代码像刚毕业的实习生，加了 `.cursorrules` 之后像 5 年经验的架构师。差距有多大？往下看。

---

## 一、你是否有过这种崩溃瞬间？

**场景 A：没有 .cursorrules**

你让 Cursor 生成一个 `OrderService`：

```java
public class OrderService {
    public void createOrder(Order order) {
        // 直接把参数拼进 MyBatis XML
        // 返回类型是 void，前端怎么接？
        // 异常没人 catch，500 一把梭
        // 日志用的 System.out.println
    }
}
```

你一看，血压拉满。异常不兜底、返回不统一、日志用 `sysout`、甚至还把用户输入直接拼 SQL。你花了 20 分钟改代码，30 分钟改测试，原本指望 AI 提效，结果变成了 AI 挖坑你来填。

**场景 B：有了 .cursorrules**

同样一句话，Cursor 生成出来的是：

```java
@Slf4j
@Service
@RequiredArgsConstructor
public class OrderServiceImpl implements OrderService {

    private final OrderMapper orderMapper;
    private final OrderItemMapper orderItemMapper;

    @Override
    @Transactional(rollbackFor = Exception.class)
    public Result<OrderVO> createOrder(CreateOrderCmd cmd) {
        log.info("[创建订单] 请求参数：{}", JsonUtils.toJson(cmd));
        // 参数校验、DDD 领域模型转换
        Order order = OrderAssembler.toDomain(cmd);
        orderMapper.insert(order);
        // 统一返回体、异常全局兜底
        return Result.success(OrderAssembler.toVO(order));
    }
}
```

事务有了、日志有了、统一返回体有了、DDD 分层有了、参数校验在前面、异常交给全局处理器。这才是能直接上生产的代码。

**同一句 Prompt，差距为什么这么大？**

答案只有三个字：**规则文件**。

而且这个规则文件不仅 Cursor 能用，GitHub Copilot、Codeium、Windsurf 等多款 AI 编程工具都能识别。这就是我们今天要聊的主角——`.cursorrules`。

---

## 二、.cursorrules 到底是什么？

`.cursorrules` 是 Cursor 编辑器的**项目级 AI 编程规范文件**，放在项目根目录下，Cursor 在生成代码时会自动读取并遵循其中的规范。

简单理解：它是你写给 AI 的"团队编码规范"，告诉 AI：
- 这个项目用什么技术栈
- 代码该用什么风格
- 异常该怎么处理
- 返回体该长什么样
- 什么能做、什么坚决不能做

类似的文件还有 `.github/copilot-instructions.md`（Copilot 专用），后面会讲两者的共存方案。

---

## 三、开箱即用：Java 项目 .cursorrules 完整模板

下面是一份可以直接复制到你项目根目录的 `.cursorrules` 模板。基于**多模块 Maven + DDD 分层**，覆盖了 99% 的 Java 后端规范场景。

```yaml
# =============================================================
# Cursor Rules for Java 后端项目
# 放置位置：项目根目录 /.cursorrules
# 适用场景：多模块 Maven + Spring Boot + MyBatis-Plus + DDD
# =============================================================

# ---------- 项目架构 ----------
project_architecture:
  type: "Maven 多模块项目"
  parent: "pom.xml（根目录）"
  modules:
    - name: "xxx-common"
      layer: "公共模块"
      content: "通用工具类、统一返回体 Result<T>、全局异常枚举、基础常量"
    - name: "xxx-domain"
      layer: "领域层"
      content: "Domain 实体、Repository 接口（不含实现）、领域服务"
    - name: "xxx-infrastructure"
      layer: "基础设施层"
      content: "Mapper 接口及 XML、RepositoryImpl、外部 API 调用、消息队列"
    - name: "xxx-application"
      layer: "应用层"
      content: "Application Service、DTO/CMD/VO 定义、Assembler 转换器"
    - name: "xxx-api"
      layer: "接口层"
      content: "Controller、全局异常处理器、拦截器、Spring Boot 启动类"
  module_dependencies:
    "xxx-api": ["xxx-application", "xxx-common"]
    "xxx-application": ["xxx-domain", "xxx-infrastructure", "xxx-common"]
    "xxx-infrastructure": ["xxx-domain", "xxx-common"]
    "xxx-domain": ["xxx-common"]

# ---------- 技术栈约束 ----------
tech_stack:
  language: "Java 17"
  framework: "Spring Boot 3.2.x"
  orm: "MyBatis-Plus 3.5.x"
  database: "MySQL 8.0 / PostgreSQL 15"
  cache: "Redis 7.x（Spring Cache + RedisTemplate）"
  mq: "RocketMQ 5.x / RabbitMQ 3.x"
  rpc: "OpenFeign（Spring Cloud）或 gRPC"
  build_tool: "Maven 3.9+"
  test:
    unit: "JUnit 5 + Mockito"
    integration: "SpringBootTest + Testcontainers"
    coverage: "JaCoCo，核心业务覆盖率 ≥ 80%"

# ---------- 代码风格规范 ----------
code_style:
  naming:
    class: "大驼峰，如 OrderService"
    method: "小驼峰，如 createOrder"
    constant: "全大写下划线，如 MAX_RETRY_TIMES"
    package: "全小写，如 com.xxx.order.domain"
    dto: "以 DTO/CMD/VO/Query 结尾，如 CreateOrderCmd"
    service_impl: "接口名 + Impl，如 OrderServiceImpl"
    mapper: "实体名 + Mapper，如 OrderMapper"
  annotation_usage:
    # 必须使用的 Lombok 注解
    must_use:
      - "@Slf4j（日志）"
      - "@Data（DTO/VO 上）"
      - "@Builder（配合 @AllArgsConstructor + @NoArgsConstructor）"
    recommend:
      - "@RequiredArgsConstructor（注入 Bean，替代 @Autowired）"
      - "@RestController（Controller 类上）"
      - "@RequestMapping(\"/api/v1/xxx\")（统一路径前缀）"
    avoid:
      - "@Autowired 字段注入（禁止），改用构造器注入或 @RequiredArgsConstructor"
      - "@RequestMapping 写在方法上（不够清晰）"
  log_format:
    logger: "使用 @Slf4j，禁止 System.out.println"
    level:
      info: "业务流程关键节点"
      error: "异常捕获处，传入 Throwable 参数"
      debug: "调试信息，如 SQL 参数"
    pattern: "前置方括号标识模块，如 log.info(\"[创建订单] 请求参数：{}\", json);"

# ---------- 异常处理规范 ----------
exception_handling:
  global_handler: "使用 @RestControllerAdvice 全局捕获"
  exception_enum: "在 common 模块定义统一异常枚举，如 ErrorCode.ORDER_NOT_FOUND"
  business_exception: "自定义 BusinessException(code, message)，由全局处理器统一处理"
  controller_layer: "不要写 try-catch，全部上抛给全局处理器"
  transaction: "@Transactional(rollbackFor = Exception.class)，放在 Service 层 public 方法上"
  forbidden:
    - "禁止吞异常（catch 后什么都不做）"
    - "禁止把异常栈信息直接返回给前端"
    - "禁止在 Controller 中写 try-catch 包裹所有逻辑"

# ---------- 统一响应格式 ----------
response_format:
  class_name: "Result<T>（位于 xxx-common 模块）"
  structure:
    code: "Integer，业务状态码，200 表示成功"
    message: "String，提示信息"
    data: "T，泛型数据体"
  success_method: "Result.success(data)"
  fail_method: "Result.fail(errorCode)"
  forbidden:
    - "禁止 Controller 直接返回 Map 或 Object"
    - "禁止返回原始异常信息给前端"

# ---------- 数据库操作规范 ----------
database:
  orm_rules:
    - "简单 CRUD 使用 MyBatis-Plus BaseMapper 自带方法"
    - "复杂查询在 Mapper 接口中定义方法，在 XML 中写 SQL"
    - "禁止在 Java 代码中拼接 SQL 字符串"
    - "分页查询使用 MyBatis-Plus 的 IPage + Page 对象"
  entity_mapping:
    entity: "使用 @TableName 指定表名，@TableId 指定主键策略"
    logic_delete: "使用 @TableLogic 标记逻辑删除字段"
    autofill: "使用 @TableField(fill = FieldFill.INSERT) 标记自动填充字段（如 create_time）"
  sql_injection_prevention:
    - "所有动态 SQL 使用 #{} 占位符，禁止使用 ${}（除非 ORDER BY 等必须场景，且需白名单校验）"
    - "禁止拼接用户输入到 SQL 中"

# ---------- 测试规范 ----------
testing:
  unit_test:
    framework: "JUnit 5 + Mockito"
    naming: "被测试方法名 + 场景 + 期望结果，如 createOrder_WithValidCmd_ShouldReturnSuccess"
    structure: "Given-When-Then 三段式"
    coverage: "Service 层核心方法必须覆盖，覆盖率 ≥ 80%"
  integration_test:
    framework: "@SpringBootTest + Testcontainers"
    isolation: "每个测试类独立数据库容器，互不影响"
  forbidden:
    - "禁止测试依赖执行顺序（每个测试必须独立）"
    - "禁止在单元测试中启动 Spring 容器（那是集成测试）"

# ---------- 安全规范 ----------
security:
  forbidden:
    - "代码中禁止硬编码密码、Token、AccessKey 等敏感信息，统一使用配置中心或环境变量"
    - "禁止将 .env / application-prod.yml 等敏感配置文件提交到仓库"
    - "用户输入必须经过参数校验（@Valid + JSR-303）"
    - "SQL 查询参数必须使用 #{} 占位符，ORDER BY 等动态字段必须走白名单校验"
    - "接口必须进行权限校验（@PreAuthorize 或自定义切面）"
  recommend:
    - "敏感数据（手机号、身份证）日志输出时脱敏"
    - "密码使用 BCrypt 加密存储"
```

---

## 四、5 组震撼对比：不加规范 vs 加了规范

口说无凭，我们上实战对比。

### 对比 1：依赖注入方式

**不加规范，Cursor 默认生成：**

```java
@RestController
public class OrderController {

    @Autowired
    private OrderService orderService;  // 字段注入，单元测试无法 mock
}
```

**加了规范（禁止 @Autowired 字段注入）：**

```java
@RestController
@RequiredArgsConstructor
public class OrderController {

    private final OrderService orderService;  // 构造器注入，天然支持测试和不可变性
}
```

**差距点**：字段注入在单元测试时无法方便 Mock，且状态可变。构造器注入配合 `@RequiredArgsConstructor` 是 Spring 官方推荐的最佳实践。

---

### 对比 2：异常处理方式

**不加规范，Cursor 默认生成：**

```java
@PostMapping("/create")
public Result create(@RequestBody CreateOrderCmd cmd) {
    try {
        Order order = orderService.create(cmd);
        return Result.success(order);
    } catch (Exception e) {
        log.error("创建订单失败", e);
        return Result.fail("创建失败：" + e.getMessage());  // 异常栈泄露给前端
    }
}
```

**加了规范（禁止 Controller 写 try-catch、统一全局处理）：**

```java
@PostMapping("/create")
public Result<OrderVO> create(@Valid @RequestBody CreateOrderCmd cmd) {
    return orderApplicationService.createOrder(cmd);
}
// 异常统一由 @RestControllerAdvice 处理，自动包装为 Result.fail(errorCode)
```

**差距点**：没有规范时，每个方法都要写 try-catch，代码又臭又长，还容易泄露异常栈。有了规范后，Controller 干干净净，异常交给全局处理器，前端拿到的永远是规范的 `Result`。

---

### 对比 3：日志输出方式

**不加规范，Cursor 默认生成：**

```java
public void cancelOrder(Long orderId) {
    System.out.println("取消订单：" + orderId);  // sysout，没有时间戳、日志级别、模块标识
    // 业务逻辑...
    // 异常时 System.out.println("失败：" + e);  // 连堆栈都丢不出去
}
```

**加了规范（@Slf4j + 方括号模块标识 + 异常参数）：**

```java
@Slf4j
@Service
public class OrderServiceImpl {

    @Transactional(rollbackFor = Exception.class)
    public void cancelOrder(Long orderId) {
        log.info("[取消订单] 订单ID：{}", orderId);
        // 业务逻辑...
    }

    // 异常捕获处
    // } catch (Exception e) {
    //     log.error("[取消订单] 订单取消失败，订单ID：{}", orderId, e);
    //     throw new BusinessException(ErrorCode.ORDER_CANCEL_FAILED);
    // }
}
```

**差距点**：`log.info` 比 `sysout` 多了时间戳、日志级别、线程名、类名，还支持参数化（不是字符串拼接）。方括号便于 ELK 中快速定位日志所属业务模块。异常时传入 `e` 参数，堆栈完整保留。

---

### 对比 4：SQL 动态参数处理

**不加规范，Cursor 默认生成：**

```java
// Mapper XML 中
SELECT * FROM t_order WHERE 1=1
<if test="orderStatus != null and orderStatus != ''">
    AND order_status = ${orderStatus}  ← SQL 注入风险！
</if>
```

**加了规范（禁止 ${}，除非 ORDER BY 且白名单校验）：**

```java
// Mapper XML 中
SELECT * FROM t_order WHERE 1=1
<if test="orderStatus != null and orderStatus != ''">
    AND order_status = #{orderStatus}  ← 参数化查询，安全
</if>
```

**差距点**：`${}` 是字符串替换，直接拼接到 SQL 中，给了 SQL 注入可乘之机。`#{}` 是预编译占位符，MyBatis 底层用 `PreparedStatement` 处理，从根本上杜绝 SQL 注入。

---

### 对比 5：统一返回体

**不加规范，Cursor 默认生成：**

```java
// 不同 Controller 返回的类型五花八门
@GetMapping("/{id}")
public OrderVO getOrder(@PathVariable Long id) { ... }

@PostMapping("/create")
public Map<String, Object> create(@RequestBody OrderDTO dto) { ... }

@GetMapping("/list")
public List<OrderVO> list() { ... }
// 前端要分别处理：OrderVO、Map、List、String……崩溃
```

**加了规范（统一 Result<T> 返回）：**

```java
@GetMapping("/{id}")
public Result<OrderVO> getOrder(@PathVariable Long id) {
    return Result.success(orderApplicationService.getOrder(id));
}

@PostMapping("/create")
public Result<OrderVO> create(@Valid @RequestBody CreateOrderCmd cmd) {
    return Result.success(orderApplicationService.createOrder(cmd));
}

@GetMapping("/list")
public Result<PageResult<OrderVO>> list(@Valid OrderQuery query) {
    return Result.success(orderApplicationService.listOrders(query));
}
```

**差距点**：前端只需要统一解析 `Result<T>`，`code=200` 取 `data`，`code!=200` 展示 `message`。前后端对接成本骤降。

---

### 对比 1-5 小结表

| 对比维度 | 无规范（默认生成） | 有规范（.cursorrules） |
|---------|-------------------|----------------------|
| 依赖注入 | `@Autowired` 字段注入 | 构造器注入 + `@RequiredArgsConstructor` |
| 异常处理 | Controller 写 try-catch，泄露异常栈 | 全局处理器统一拦截，干净安全 |
| 日志输出 | `System.out.println` | `@Slf4j` + 模块标识 + 参数化 |
| SQL 参数 | `${}` 字符串拼接，有注入风险 | `#{}` 预编译占位符，安全 |
| 返回格式 | Object / Map / List 五花八门 | 统一 `Result<T>` 封装 |

---

## 五、落地实战：让规则真正生效的三个关键步骤

很多朋友拿到模板后直接丢到项目里，结果发现 AI 还是不听话。问题出在哪里？下面三步帮你排查。

### 5.1 确保 Cursor 正确加载规则

规则文件必须满足以下条件才能被 Cursor 识别：

1. **文件名必须是 `.cursorrules`**（注意开头有个点），放在项目根目录
2. **文件编码必须是 UTF-8**，否则中文规则会被当成乱码忽略
3. **必须在 Cursor 中打开项目根目录**（不是子目录），否则 Cursor 找不到规则文件

**验证方法**：打开 Cursor，按 `Cmd+Shift+P`（Windows 按 `Ctrl+Shift+P`），输入 `Cursor: Open Settings`，确认 Agent 和 Rules 相关配置已启用。

### 5.2 规则表述的黄金法则：说"要什么"而非"不要什么"

AI 对否定句的理解精度远低于肯定句。同样一条规则，两种写法效果天差地别：

❌ **差的表述（否定句）**：
```yaml
code_style:
  forbidden:
    - "不要使用 System.out.println"
    - "不要用 @Autowired"
```

✅ **好的表述（肯定句 + 示例）**：
```yaml
code_style:
  log: "使用 @Slf4j 注解获取 Logger，通过 log.info() 输出日志"
  injection: "使用 @RequiredArgsConstructor + private final 实现构造器注入"
  example: |
    @Slf4j
    @Service
    @RequiredArgsConstructor
    public class OrderService {
        private final OrderMapper orderMapper;
        public void create() {
            log.info("创建订单");
        }
    }
```

**原理**：当你写"不要做 A"，AI 需要先理解 A，再排除 A，最后在剩下的选项里选择——这个过程容易出错。而直接写"做 B，像这样"，AI 直接模仿即可，准确率大幅提升。

### 5.3 阶段性规则：新人项目 vs 成熟项目

规则不是越多越好。项目早期规则太苛刻，反而拖慢进度。建议分阶段渐进式增强：

**第一阶段（0→1 原型期）**——规则文件只放 3 条核心规范：
- 技术栈限定（Java 版本、Spring Boot 版本、ORM 框架）
- 统一返回体 Result<T>
- 包名和模块划分

这个阶段追求的是**快速试错**，过度规范会限制 AI 的创造力。

**第二阶段（1→10 基建期）**——加入代码质量规范：
- 异常处理（全局处理器）
- 日志规范（@Slf4j）
- 数据库操作规范（防 SQL 注入）

团队逐渐稳定，该阶段通过规则保证代码**底线质量**。

**第三阶段（10→N 规模化期）**——全量规范上线：
- 单元测试规范（覆盖率要求）
- 安全规范（禁止硬编码、敏感信息脱敏）
- 环境差异化规则（dev/test/prod）

团队规模扩大、系统进入生产环境，规则侧重**安全与稳定性**。

---

## 六、.cursorrules 高级配置技巧

### 6.1 @文件引用：拆分规则文件，告别千行配置

当项目变复杂时，`.cursorrules` 很容易膨胀到上千行。Cursor 支持通过 `@` 语法引用其他文件，实现模块化管理。

**拆分方案：**

```
项目根目录/
├── .cursorrules              ← 主入口，只做引用
└── .cursor/
    ├── 01-architecture.rules
    ├── 02-code-style.rules
    ├── 03-exception.rules
    ├── 04-database.rules
    ├── 05-security.rules
    └── 06-testing.rules
```

**主入口 `.cursorrules` 写法：**

```yaml
# 主规则文件：只负责引用子规则
@.cursor/01-architecture.rules
@.cursor/02-code-style.rules
@.cursor/03-exception.rules
@.cursor/04-database.rules
@.cursor/05-security.rules
@.cursor/06-testing.rules
```

**优点**：
- 每个文件职责单一，便于团队协作维护
- 不同项目可以按需引用，不用大段删改
- 新增规范不影响已有文件，降低冲突风险

---

### 6.2 多环境差异化规则

开发环境、测试环境、生产环境对代码的要求不一样，怎么办？

**方案一：分支级区分（推荐）**

在 `dev` / `test` / `main` 分支上维护不同的 `.cursorrules`，合并时不冲突：

- `dev` 分支：允许 `log.debug()`、可以不写事务注解
- `test` 分支：数据库连接改为测试库、Mocks 设置更宽松
- `main` 分支：开启安全检查、强制事务、禁止调试代码

**方案二：文件内条件标记**

```yaml
environment_specific:
  dev:
    log_level: "允许 log.debug()"
    sql_print: "允许打印 SQL 到控制台"
  test:
    log_level: "允许 log.debug()"
    database: "使用 Testcontainers，不连外部库"
  prod:
    log_level: "禁止 log.debug()，敏感数据禁止打印"
    sql_print: "禁止打印 SQL"
    force_rule: "所有接口必须有 @PreAuthorize 权限注解"
    critical: "禁止 System.exit()、禁止 shutdown hook 执行危险操作"
```

Cursor 会根据当前文件所在分支或上下文，自动匹配对应环境规则。

---

### 6.3 规则优先级与覆盖机制

当多个规则文件出现冲突时，优先级如下（从高到低）：

```
1. 用户级配置    （~/.cursor/rules 或 ~/.cursorrules）
2. 项目级主文件  （.cursorrules）
3. 项目级子文件  （.cursor/xxx.rules）
4. 目录级规则    （子目录下的 .cursorrules）
5. Cursor 内置默认规则
```

**覆盖策略**：高优先级规则**覆盖**低优先级的同名字段，不会合并。如果用户级和项目级都定义了 `code_style.naming.class`，以用户级为准。

**实战建议**：
- 用户级放个人偏好（比如用 `var` 还是显式类型）
- 项目级放团队共识（技术栈、架构分层、统一返回体）
- 子目录级放模块特定规则（如 `xxx-infrastructure` 模块要求 MyBatis XML 格式）

---

## 七、.cursorrules vs .github/copilot-instructions.md

很多团队两个文件都见过，但是搞不清楚该用哪个、能不能共存。这里讲清楚。

| 对比维度 | .cursorrules | .github/copilot-instructions.md |
|---------|-------------|---------------------------------|
| 所属工具 | Cursor 编辑器 | GitHub Copilot |
| 文件位置 | 项目根目录 | 项目 `.github/` 目录下 |
| 格式要求 | YAML 或 Markdown 皆可 | Markdown 格式 |
| 生效范围 | Cursor Chat / Tab / Composer | Copilot Chat / Code Completion |
| 是否可拆分 | 支持 @文件引用 | 暂不支持模块化引用 |

### 共存方案（少写一份规则）

两个文件的规则内容 90% 是重复的，维护两套太蠢了。推荐的做法是：

**方案：.cursorrules 作为唯一真相源，copilot-instructions.md 指向它**

```markdown
<!-- .github/copilot-instructions.md -->

本项目 AI 编程规范请参考根目录 `.cursorrules` 文件，核心规范包括：

1. **架构**：多模块 Maven + DDD 分层（api → application → domain → infrastructure）
2. **技术栈**：Java 17 / Spring Boot 3.2.x / MyBatis-Plus 3.5.x
3. **返回体**：统一使用 `Result<T>`，位于 xxx-common 模块
4. **异常处理**：统一使用 `@RestControllerAdvice`，Controller 层禁止写 try-catch
5. **注入方式**：禁止 `@Autowired` 字段注入，统一构造器注入 + `@RequiredArgsConstructor`
6. **日志**：`@Slf4j`，禁止 `System.out.println`
7. **SQL 安全**：必须使用 `#{}` 占位符，禁止 `${}` 拼接用户输入

完整规范详见：https://github.com/xxx/xxx/blob/main/.cursorrules
```

这样 `.cursorrules` 是唯一需要维护的规则源，`.github/copilot-instructions.md` 只做一份精简摘要引用，两边的 AI 工具都能覆盖到。

---

## 八、常见踩坑与 FAQ

### 8.1 规则写了但 Cursor 不遵守怎么办？

这是最高频的问题。按以下顺序排查：

**第一层：文件位置对吗？**
规则文件必须精准命名为 `.cursorrules`，放在项目根目录。注意：`cursorrules`（不加点）或者 `cursor.rules`（多了个点）都不行。在 macOS 中，Finder 默认隐藏以点开头的文件，需要通过终端 `ls -la` 确认文件存在。

**第二层：规则写得太抽象了吗？**
如果你写的规则是"写出高质量的代码"，那 Cursor 根本不知道什么是"高质量"。规则必须具体、可执行。对比一下：

```yaml
# 太抽象，AI 不懂
code_quality: "写出高质量的代码"

# 具体可执行，AI 直接抄
transaction: "所有 Service 层写操作必须加 @Transactional(rollbackFor = Exception.class)"
log: "使用 @Slf4j，关键节点打印 log.info，异常打印 log.error"
```

**第三层：规则之间有冲突吗？**
检查是否有多个规则文件同时存在（比如既有 `.cursorrules` 又有 `.cursor/xxx.rules`），且字段冲突。优先级规则见上文 6.3 节。

**第四层：Prompt 本身提示词是否足够清晰？**
规则是约束条件，但 AI 生成代码的质量还取决于你的 Prompt 质量。如果你只写一句"生成订单模块"，那 Cursor 的发挥空间太大；如果你写"在 xxx-api 模块生成订单 Controller，遵循 DDD 分层调用 xxx-application 的 OrderApplicationService"，效果会好很多。

### 8.2 团队多人协作时规则冲突怎么办？

**推荐流程**：
1. 将 `.cursorrules` 纳入 Git 版本管理（`.cursor/` 目录也一起提交）
2. 规则变更走 PR 流程，和代码变更一样需要 Review
3. 在团队 CI 流水线中加入规则格式校验（写一个简单的 YAML 格式校验脚本）
4. 个人偏好用 `~/.cursorrules`（用户级），不提交到仓库

### 8.3 规则能不能按模块/包路径来定制？

可以。Cursor 支持**目录级规则文件**——在子目录下放一个 `.cursorrules`，该目录及其子目录中的代码生成会使用该规则，覆盖父级规则中的同名字段。

但实际使用中，**不推荐**过度使用这个特性。理由有三：
1. 规则碎片化，维护成本指数级上升
2. 新人不知道哪个目录对应哪套规则
3. 跨模块调用时容易生成风格不一致的代码

**最佳实践**：项目根目录一套主规则 + 个别特殊模块补充 1-2 条即可。比如 `xxx-infrastructure` 模块额外要求：

```yaml
# xxx-infrastructure/.cursorrules
mybatis_xml_style:
  result_map: "所有查询必须定义 resultMap，禁止使用 resultType"
  sql_format: "SQL 关键字大写，字段名小写"
```

### 8.4 Cursor 和 GitHub Copilot 同时用，会不会打架？

不会。两者读取的是不同文件（`.cursorrules` vs `.github/copilot-instructions.md`），互不干扰。但如果你在 Cursor 中安装了 Copilot 插件，Copilot 的 Code Completion 会受 `.github/copilot-instructions.md` 影响，而 Cursor 的 Chat 和 Composer 则受 `.cursorrules` 影响。

**最佳体验**：两个文件都配置，内容指向同一套规范（参见上文共存方案）。这样不管用哪个工具、哪个功能，生成的代码风格都一致。

---

## 九、总结

一套好的 `.cursorrules`，本质上是把你的团队编码规范和项目架构知识，提前"注入"到 AI 的大脑中。结果就是：

- **生成代码的风格统一**：不再是看 AI 心情输出
- **安全漏洞大幅减少**：SQL 注入、硬编码密码、异常泄露从源头杜绝
- **CR 时间缩短 50%+**：不用再反复提醒"这里要用 Result 包裹""那里不要用 @Autowired"
- **新人上手更快**：AI 生成的代码本身就是团队规范的最佳示范

**行动建议**：今天就花 20 分钟，把上面的模板放进你项目的 `.cursorrules`，然后试着让 Cursor 生成一个接口——你会回来给我点赞的。

---

## 下一篇预告

**《Cursor + Spring Boot：3 步生成完整 CRUD，从此告别手写增删改查》**

- 如何用一句 Prompt 生成 Controller + Service + Mapper + XML 全套代码
- 配合 MyBatis-Plus 自动生成分页查询
- 参数校验 + 统一异常处理一键配齐
- 生成后如何一键跑通测试

关注我，不迷路。下一篇，让你感受什么叫"真·10 倍提效"。

---

*本文首发于 CSDN，转载请注明出处。*

*作者：深耕 Java 后端 10 年的技术博主，专注 AI 赋能研发效能。*
