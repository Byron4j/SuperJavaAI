# Prompt 持续优化：基于用户反馈的迭代改进方法，好的 Prompt 不是一次写出来的而是"养"出来的

## 开篇：Prompt 是一个活着的存在

第一版 Prompt 生成了能用的代码，第二版加了约束生成了规范的代码，第三版加了示例生成了满意的代码——Prompt 优化是一个迭代过程。

很多开发者都有一个误区：以为写 Prompt 就像写 SQL，一次写对就能一直用。但实际上，Prompt 更像一棵植物——你需要不断浇水、修剪、施肥，它才能持续产出最好的结果。

回想一下你用 ChatGPT 或 Copilot 的经历：第一次的 Prompt 往往只能得到 60 分的输出，然后你不断调整措辞、补充上下文、添加约束，直到满意为止。这个"调整"的过程，本质上就是 Prompt 优化。

但问题在于——大多数人的优化是即兴的、无记录的、不可复制的。今天调好了，下周就忘了。这篇文章的核心目的，就是帮你建立一套**基于用户反馈的迭代优化体系**，让 Prompt 优化从"凭感觉"变成"有方法论"。

## 一、优化循环：Prompty 持续进化的六步引擎

在 Google、微软等大厂的 Prompt Engineering 实践中，反复出现的优化模式可以总结为以下六个步骤：

```
记录反馈 → 分析问题 → 定位根源 → 改进Prompt → 对比评测 → 发布上线
```

### 1.1 记录反馈

反馈来源主要有三类：

| 反馈来源 | 收集方式 | 典型问题 |
|---------|---------|---------|
| 自我审查 | 对 AI 输出进行 Code Review 时记录 | "生成的代码没加 @Transactional 注解" |
| 团队审查 | PR Review 中被同事指出 | "这个异常处理太粗糙了，线上会炸" |
| 用户反馈 | 下游消费者（前端、测试、运维）反馈 | "生成的接口响应字段命名不规范" |

建议为每个 Prompt 维护一个反馈日志：

```markdown
## Prompt: generate-service-layer
### 反馈记录
- 2024-12-01: 生成的 Service 方法没加 @Transactional（来源：张三 CR）
- 2024-12-03: 生成的 Controller 没加参数校验注解（来源：前端反馈）
- 2024-12-05: 生成的异常处理吞掉了原始异常（来源：线上故障）
```

### 1.2 分析问题

拿到反馈后不要立刻改 Prompt，先问自己三个问题：

1. **这个问题是偶发的还是系统性的？**——如果是偶发（比如某次 AI"抽风"），记录即可；如果是系统性的（每次都出），必须改 Prompt。
2. **是 Prompt 没说到，还是 Prompt 说错了？**——前者是"遗漏"，需要添加约束；后者是"误导"，需要修改措辞。
3. **改完会不会引入新问题？**——Prompt 优化也存在"过度拟合"的风险，改了 A 约束可能让 B 输出变差。

### 1.3 定位根源

根据反馈分类（后文会详述），定位到 Prompt 中的具体问题点：

```
AI输出问题 → 对应Prompt缺陷
─────────────────────────────
代码有bug      → 缺少技术约束或示例
风格不符合规范  → 缺少代码风格描述
功能不完整      → 缺少需求描述细节
过度设计        → 缺少简洁性约束
有安全隐患     → 缺少安全约束
```

### 1.4 改进 Prompt

这是核心步骤。改进 Prompt 有五种常用策略：

1. **添加约束**：`不要使用 Lombok` → `请手写 getter/setter 方法`
2. **添加示例**：`生成一个分页查询` → `生成一个分页查询，参考以下代码格式：[example]`
3. **调整角色**：`你是一个程序员` → `你是一个在阿里巴巴工作 10 年的 Java 高级工程师`
4. **调整结构**：从一句话 Prompt 升级为结构化 Prompt（角色+上下文+约束+格式）
5. **添加负面示例**：`不要这样做：[bad example]`

### 1.5 对比评测

改进后必须做 A/B 测试。方法很简单：

```
旧Prompt 输出 | 新Prompt 输出 | 评估
─────────────|─────────────|─────
（截图/代码）  | （截图/代码）  | ✅ 提升 / ⚠️ 持平 / ❌ 退化
```

至少运行 3 次来排除随机性。如果新 Prompt 在某个维度退化，需要回退或微调。

### 1.6 发布上线

通过评测后，更新 Prompt 版本号并归档：

```yaml
version: 2.3.0
date: 2024-12-10
author: zhangsan
changes:
  - 添加了 @Transactional 约束
  - 添加了参数校验示例
  - 修正了异常处理描述
evaluation:
  correctness: ↑ 15%
  style_compliance: ↑ 10%
  token_cost: ≈
```

## 二、反馈分类体系：精准定位 Prompt 问题

把所有用户反馈归纳为以下五大类别，让问题定位更加高效：

### 2.1 代码错误（Bug）

**症状**：AI 生成的代码有编译错误、逻辑错误、运行时异常。

**常见表现**：
- 引用了不存在的类或方法
- 空指针未处理
- 数据类型不匹配
- 并发安全问题

**Prompt 改进方向**：
- 明确技术栈版本：`使用 Spring Boot 3.2 + JDK 21`
- 添加类型约束：`所有返回值必须用 Optional 包装，禁止返回 null`
- 添加代码审查提示：`生成后请自行检查：1.所有import是否存在 2.所有方法调用是否匹配签名`

**优化案例**：
```
❌ 旧Prompt：
"写一个用户登录方法"

✅ 新Prompt：
"使用 Spring Boot 3.2 + Spring Security 6 写一个用户登录方法。
约束：
1. 使用 BCryptPasswordEncoder 验证密码
2. 登录成功后返回 JWT Token
3. 不要直接返回 null，使用 Optional<User> 包装
4. 对输入参数做非空校验"
```

### 2.2 风格不符（Style）

**症状**：代码能跑，但不符团队编码规范。

**常见表现**：
- 使用了禁止的库（如 Lombok）
- 命名不符合团队规范（如 Controller 方法命名不 RESTful）
- 缩进、换行等格式不一致
- 注释风格不符合 Javadoc 标准

**Prompt 改进方向**：
- 在 Prompt 开头放置团队代码规范摘要
- 使用 Few-Shot 示例展示期望的风格
- 使用正向约束替代禁止约束

**优化案例**：
```
❌ 旧Prompt：
"生成一个用户管理的 REST API"
（AI 使用了 Lombok 的 @Data 注解，但团队禁止使用 Lombok）

✅ 新Prompt：
"生成一个用户管理的 REST API。
代码规范：
1. 禁止使用 Lombok，所有 getter/setter 手写
2. 所有 public 方法必须有 Javadoc
3. 使用 Google Java Style 格式
4. Controller 命名规范：{Entity}Controller
5. Service 方法命名规范：create{Entity}, get{Entity}ById, update{Entity}, delete{Entity}

参考代码风格示例：
```java
public class UserController {
    private final UserService userService;
    
    /**
     * 根据ID获取用户信息。
     * @param id 用户ID
     * @return 用户详情
     */
    @GetMapping("/{id}")
    public ResponseEntity<UserResponse> getUserById(@PathVariable Long id) {
        // ...
    }
}
```"
```

### 2.3 不够完整（Incomplete）

**症状**：AI 生成的功能缺少必要的组成部分。

**常见表现**：
- 只有核心逻辑，缺少参数校验
- 只有正常流程，缺少异常处理
- 只有 Java 代码，缺少配置文件
- 只有后端代码，缺少单元测试

**Prompt 改进方向**：
- 使用 Checklist 明确需求："请确保包含以下内容：[ ]参数校验 [ ]异常处理 [ ]日志 [ ]单元测试"
- 使用结构化输出格式：要求 AI 按模块输出
- 分步骤引导："先生成 Service 层，确认后再生成 Controller 层"

**优化案例**：
```
❌ 旧Prompt：
"生成一个文件上传接口"

✅ 新Prompt：
"生成一个文件上传接口，必须包含以下内容：
1. Controller 层：接收 MultipartFile，返回文件URL
2. Service 层：文件校验（类型、大小）、存储（本地/OSS）、返回结果
3. 异常处理：文件太大、格式不支持、存储失败
4. 配置项：上传路径、允许的类型、最大大小（从 application.yml 读取）
5. 单元测试：正常上传、文件过大、类型不支持 3 个场景
6. 全局异常处理器中注册对应的异常处理

请分别输出每一部分的代码，并用注释标注文件路径。
如：<!-- Controller: src/main/java/com/example/controller/FileController.java -->"
```

### 2.4 过度设计（Over-engineering）

**症状**：简单需求被 AI 设计成复杂的架构。

**常见表现**：
- 一个简单的 CRUD 生成了工厂模式+策略模式+观察者模式
- 简单工具类被设计成 Spring Bean + 接口 + 抽象类 + 实现类
- 引入了不必要的第三方库
- 对不确定的"未来可能"需求做了过多预留

**Prompt 改进方向**：
- 明确 YAGNI 原则："用最简单的方式实现，不做任何过度设计"
- 限制类数量："整个功能不超过 3 个类"
- 禁止设计模式："不需要使用任何设计模式，如果强行使用反而是错误的"

**优化案例**：
```
❌ 旧Prompt：
"写一个功能：根据用户类型（VIP/普通）计算折扣"
（AI 生成了：DiscountStrategy 接口 + VipDiscountStrategy + NormalDiscountStrategy + DiscountCalculator + DiscountFactory + DiscountType 枚举）

✅ 新Prompt：
"写一个功能：根据用户类型（VIP: 8折, 普通: 不打折）计算折扣。
要求：用最简单的方式实现，不需要任何设计模式。
只用一个类、一个方法即可完成。
禁止使用：策略模式、工厂模式、接口抽象等任何模式化设计。
如果超过20行代码，请重新思考是否过度设计。"
```

### 2.5 安全隐患（Security）

**症状**：AI 生成的代码存在安全漏洞。

**常见表现**：
- SQL 注入（字符串拼接 SQL）
- XSS 漏洞（未对用户输入做转义）
- 敏感信息硬编码（密码写死在代码里）
- 权限校验缺失

**Prompt 改进方向**：
- 添加安全约束："使用参数化查询，禁止字符串拼接 SQL"
- 指定安全框架："所有接口必须经过 Spring Security 权限校验"
- 添加安全检查："生成后自查 OWASP Top 10 相关问题"

**优化案例**：
```
❌ 旧Prompt：
"生成一个根据用户名搜索用户的功能"

✅ 新Prompt：
"生成一个根据用户名搜索用户的功能。
安全要求：
1. SQL 查询使用 MyBatis 的 #{} 参数化查询，禁止使用 ${}
2. 搜索关键词必须转义 SQL 通配符（%, _）
3. 返回结果必须脱敏：手机号显示为 138****1234
4. 接口必须经过认证和授权检查
5. 记录搜索行为日志（不记录用户敏感信息）
6. 对搜索关键词做长度限制（不超过50字符）
7. 使用 RateLimiter 防止接口被暴力调用"
```

## 三、完整优化案例：从 V1 到 V3 的进化之路

### 案例一：让 AI 生成的代码从"能编译"到"符合团队规范"的 10 轮优化

**场景**：需要 AI 帮忙生成一个 Java 后端的分页查询接口。

#### V1：最初的 Prompt

```
"帮我写一个分页查询用户列表的接口"
```

**AI 输出**：
```java
// AI 生成的 V1 版本 - 能跑但很粗糙
@RestController
public class UserController {
    @Autowired
    private JdbcTemplate jdbcTemplate;
    
    @GetMapping("/users")
    public List<Map<String, Object>> getUsers(
        @RequestParam(defaultValue = "1") int page,
        @RequestParam(defaultValue = "10") int size) {
        String sql = "SELECT * FROM users LIMIT " + (page-1)*size + ", " + size;
        return jdbcTemplate.queryForList(sql);
    }
}
```

**问题分析**：
- ❌ 代码错误：SQL 注入风险（直接拼接参数）
- ❌ 风格不符：使用 `@Autowired` 字段注入、没有分层、变量名不规范
- ❌ 不够完整：没有总数统计、没有参数校验、没有异常处理
- ❌ 安全隐患：SQL 注入 + 未做权限控制

#### V2：第一次优化

```
"帮我写一个分页查询用户列表的接口。
要求：
1. 使用 Spring Boot + MyBatis-Plus 3.5.x
2. 分层架构：Controller → Service → Mapper
3. 使用构造器注入而非字段注入
4. 返回分页结果（包含总数、当前页、数据列表）
5. 使用 MyBatis-Plus 的 Page 对象进行分页
6. 禁止使用字符串拼接 SQL"
```

**AI 输出**：
```java
// V2 - 架构改善但仍有问题
@RestController
@RequestMapping("/api/users")
@RequiredArgsConstructor // 使用了 Lombok
public class UserController {
    
    private final UserService userService;
    
    @GetMapping
    public PageResult<UserVo> getUsers(
            @RequestParam(defaultValue = "1") Integer pageNum,
            @RequestParam(defaultValue = "10") Integer pageSize,
            @RequestParam(required = false) String keyword) {
        return userService.getUserPage(pageNum, pageSize, keyword);
    }
}

// 但 Service 层没有事务管理，没有日志
@Service
@RequiredArgsConstructor
public class UserService {
    // ...
}
```

**问题分析**：
- ✅ 分层架构 OK
- ✅ 分页返回 OK
- ❌ 风格不符：使用了 Lombok（团队禁止）
- ❌ 不够完整：缺少事务、缺少日志、缺少参数校验
- ⚠️ V1 的很多东西改善了但仍有遗留

#### V3：引入团队规范

```
"你是一名资深 Java 后端工程师，团队使用以下技术栈和规范：

技术栈：Spring Boot 3.2、MyBatis-Plus 3.5.5、JDK 21、MySQL 8.0
代码规范：
1. 不使用 Lombok，所有代码手写
2. 使用构造器注入
3. Controller 方法命名：{action}{Entity}，如 listUsers、getUserById
4. Service 层所有写操作必须加 @Transactional
5. 所有方法必须有完整的 Javadoc
6. 分页查询统一返回 PageResult<T>（含 data、total、pageNum、pageSize）
7. 参数校验使用 @Valid + JSR-303 注解
8. 异常统一使用 GlobalExceptionHandler 处理

请按照以下模板生成分页查询用户列表的完整代码：
- Controller: UserController.listUsers()
- Service: UserService.listUsers()
- Service 方法必须包含：参数校验、日志记录、分页查询、结果脱敏
- 用户表字段：id, username, email, phone, status, create_time

输出格式：仅输出 Java 代码，不要输出解释。每个文件用注释标记路径。"
```

**AI 输出**：
```java
// src/main/java/com/example/controller/UserController.java
@RestController
@RequestMapping("/api/users")
public class UserController {

    private final UserService userService;

    public UserController(UserService userService) {
        this.userService = userService;
    }

    /**
     * 分页查询用户列表。
     *
     * @param pageNum  页码，默认为1
     * @param pageSize 每页条数，默认为10，最大100
     * @param keyword  搜索关键词（支持用户名和邮箱模糊匹配）
     * @return 分页结果
     */
    @GetMapping
    public ResponseEntity<PageResult<UserResponse>> listUsers(
            @RequestParam(defaultValue = "1") @Min(1) Integer pageNum,
            @RequestParam(defaultValue = "10") @Min(1) @Max(100) Integer pageSize,
            @RequestParam(required = false) @Size(max = 50) String keyword) {
        PageResult<UserResponse> result = userService.listUsers(pageNum, pageSize, keyword);
        return ResponseEntity.ok(result);
    }
}

// src/main/java/com/example/service/UserService.java
@Service
public class UserService {

    private static final Logger log = LoggerFactory.getLogger(UserService.class);

    private final UserMapper userMapper;

    public UserService(UserMapper userMapper) {
        this.userMapper = userMapper;
    }

    /**
     * 分页查询用户列表。
     *
     * @param pageNum  页码
     * @param pageSize 每页条数
     * @param keyword  搜索关键词
     * @return 分页结果
     */
    @Transactional(readOnly = true)
    public PageResult<UserResponse> listUsers(Integer pageNum, Integer pageSize, String keyword) {
        log.info("开始分页查询用户: pageNum={}, pageSize={}, keyword={}", 
                 pageNum, pageSize, keyword);

        Page<User> page = new Page<>(pageNum, pageSize);
        LambdaQueryWrapper<User> wrapper = new LambdaQueryWrapper<>();
        
        if (StringUtils.hasText(keyword)) {
            String escapedKeyword = escapeSqlWildcard(keyword);
            wrapper.and(w -> w.like(User::getUsername, escapedKeyword)
                             .or()
                             .like(User::getEmail, escapedKeyword));
        }
        
        wrapper.orderByDesc(User::getCreateTime);
        Page<User> userPage = userMapper.selectPage(page, wrapper);

        List<UserResponse> dataList = userPage.getRecords().stream()
                .map(this::toUserResponse)
                .collect(Collectors.toList());

        log.info("分页查询用户完成: total={}, currentPage={}", 
                 userPage.getTotal(), pageNum);
        
        return new PageResult<>(dataList, userPage.getTotal(), pageNum, pageSize);
    }

    /**
     * 转义SQL通配符，防止LIKE注入。
     */
    private String escapeSqlWildcard(String keyword) {
        return keyword.replace("\\", "\\\\")
                      .replace("%", "\\%")
                      .replace("_", "\\_");
    }

    /**
     * 将User实体转换为响应对象，手机号脱敏。
     */
    private UserResponse toUserResponse(User user) {
        UserResponse response = new UserResponse();
        response.setId(user.getId());
        response.setUsername(user.getUsername());
        response.setEmail(user.getEmail());
        response.setPhone(desensitizePhone(user.getPhone()));
        response.setStatus(user.getStatus());
        response.setCreateTime(user.getCreateTime());
        return response;
    }

    /**
     * 手机号脱敏：138****1234。
     */
    private String desensitizePhone(String phone) {
        if (phone == null || phone.length() < 11) {
            return phone;
        }
        return phone.substring(0, 3) + "****" + phone.substring(7);
    }
}
```

**V3 评估**：
- ✅ 代码错误：无编译错误，逻辑正确
- ✅ 风格符合：不使用 Lombok、构造器注入、完整的 Javadoc
- ✅ 功能完整：分页、搜索、脱敏、校验、日志
- ✅ 安全隐患：Like 注入已防范、手机号脱敏、参数长度校验

**优化总结**：

| 维度 | V1 | V2 | V3 |
|------|-----|-----|-----|
| 编译通过 | ✅ | ✅ | ✅ |
| 代码规范 | ❌ | ⚠️ | ✅ |
| 功能完整度 | 30% | 60% | 95% |
| 安全性 | ❌ | ⚠️ | ✅ |
| Token 消耗 | ~50 | ~120 | ~280 |
| 可用程度 | 不可用 | 需要大幅修改 | 可直接使用 |

### 案例二：让 AI 生成的 SQL 从"有注入风险"到"安全+高效"

**场景**：需要一个动态条件查询的 SQL。

#### V1

```
"写一个 SQL 查询用户的订单，支持按状态、时间范围、金额范围筛选"
```

**AI 输出**：
```sql
SELECT * FROM orders WHERE 1=1
<if test="status != null">
AND status = '${status}'
</if>
<if test="startTime != null">
AND create_time >= '${startTime}'
</if>
<if test="endTime != null">
AND create_time <= '${endTime}'
</if>
<if test="minAmount != null">
AND amount >= ${minAmount}
</if>
```

**安全红线**：全部使用 `${}` 导致 SQL 注入风险！

#### V2

```
"写一个 MyBatis 的动态 SQL，查询订单，支持按状态、时间范围、金额范围筛选。
要求使用 #{} 参数化查询，禁止使用 ${}。"
```

**AI 输出**：
```xml
<select id="queryOrders" resultType="Order">
    SELECT * FROM orders WHERE 1=1
    <if test="status != null and status != ''">
        AND status = #{status}
    </if>
    <if test="startTime != null">
        AND create_time &gt;= #{startTime}
    </if>
    <if test="endTime != null">
        AND create_time &lt;= #{endTime}
    </if>
    <if test="minAmount != null">
        AND amount &gt;= #{minAmount}
    </if>
    <if test="maxAmount != null">
        AND amount &lt;= #{maxAmount}
    </if>
    ORDER BY create_time DESC
</select>
```

提升明显但缺少索引提示和分页。

#### V3

```
"写一个 MyBatis 的订单动态查询 SQL，要求：

1. 安全：所有参数使用 #{}，禁止 ${}（包括排序字段也用白名单控制）
2. 性能：
   - 使用索引友好的查询条件顺序
   - 必须有 LIMIT 分页（防止全表扫描）
   - 使用覆盖索引（只查询需要的列，禁止 SELECT *）
   - 分页查询前先 count 总数
3. 完整性：
   - 参数非空校验
   - 时间范围校验（开始时间不能大于结束时间）
   - SQL 使用 <script> 标签包裹
4. 查询列：id, order_no, user_id, status, amount, create_time（不使用 SELECT *）
5. 排序默认按 create_time DESC，前端传参时用白名单校验

订单表索引：idx_status_create_time(status, create_time)，idx_user_id(user_id)"
```

**AI 输出**：
```xml
<select id="countOrders" resultType="long">
    SELECT COUNT(1) FROM orders
    <where>
        <if test="status != null and status != ''">
            AND status = #{status}
        </if>
        <if test="startTime != null">
            AND create_time &gt;= #{startTime}
        </if>
        <if test="endTime != null">
            AND create_time &lt;= #{endTime}
        </if>
        <if test="minAmount != null">
            AND amount &gt;= #{minAmount}
        </if>
        <if test="maxAmount != null">
            AND amount &lt;= #{maxAmount}
        </if>
    </where>
</select>

<select id="queryOrders" resultType="com.example.domain.Order">
    SELECT id, order_no, user_id, status, amount, create_time
    FROM orders
    <where>
        <if test="status != null and status != ''">
            AND status = #{status}
        </if>
        <if test="startTime != null">
            AND create_time &gt;= #{startTime}
        </if>
        <if test="endTime != null">
            AND create_time &lt;= #{endTime}
        </if>
        <if test="minAmount != null">
            AND amount &gt;= #{minAmount}
        </if>
        <if test="maxAmount != null">
            AND amount &lt;= #{maxAmount}
        </if>
    </where>
    ORDER BY
    <choose>
        <when test="sortField == 'amount'">
            amount
        </when>
        <when test="sortField == 'create_time'">
            create_time
        </when>
        <otherwise>
            create_time
        </otherwise>
    </choose>
    <choose>
        <when test="sortOrder == 'asc'">ASC</when>
        <otherwise>DESC</otherwise>
    </choose>
    LIMIT #{offset}, #{pageSize}
</select>
```

**V3 对比 V1 的进步**：
- 🔒 安全：从 `${}` 注入风险到白名单排序 + `#{}` 参数化
- ⚡ 性能：从 `SELECT *` 到覆盖索引 + 分页 + 索引友好的条件顺序
- 📋 完整：从单条 SQL 到 count + query 组合

### 案例三：让 AI 回复从"啰嗦 500 字"到"精准 3 句话"

**场景**：用 AI 做 Code Review 输出，期望简洁直接。

#### V1

```
"帮我 review 这段代码"
```

**AI 输出**：
> 这段代码整体来看还是比较不错的。我仔细阅读了一下，发现有几个地方可以考虑优化。首先，关于变量命名方面，我们通常建议使用有意义的命名方式，这样可以让代码的可读性更好……（省略 500 字啰嗦内容）

#### V2

```
"帮我 review 这段代码。输出格式：
- 每个问题一行
- 格式：文件:行号:级别:问题描述
- 级别分为：严重/建议/风格"
```

**AI 输出**在格式上有所改观，但仍有冗余解释。

#### V3

```
"你是一个 Code Review 机器人。请 review 以下代码。

输出格式（严格遵守，不要输出任何其他内容）：
[严重] 文件:行号 - 问题描述（一句话）
[建议] 文件:行号 - 问题描述（一句话）
[风格] 文件:行号 - 问题描述（一句话）

不要输出代码示例。
不要输出"建议怎么改"。
不要输出评分或总结。
不要输出任何问候语或结束语。
只输出问题列表。如果没有发现问题，输出 'No issues found.'。

代码：
{code}"
```

**AI 输出**：
```
[严重] UserController.java:15 - SQL使用字符串拼接存在注入风险
[建议] UserService.java:42 - 缺少@Transactional注解
[风格] UserController.java:8 - 使用了@Autowired字段注入建议改为构造器注入
```

从 500 字到 3 行，精准直达。

## 四、建立持续优化 SOP

### 4.1 反馈收集渠道

```yaml
反馈渠道:
  自我审查:
    - 每次使用Prompt后花1分钟记录问题
    - 使用模板: "Prompt:{name} | 问题:{description} | 分类:{bug/style/incomplete/overengineer/security}"
  
  代码评审:
    - 在PR模板中加入: "AI辅助生成的代码部分: [是/否]"
    - 如果AI代码被修改, 记录改动原因到Prompt反馈日志
  
  自动化检测:
    - 静态代码分析（SonarQube）结果
    - 单元测试通过率
    - 安全扫描（SpotBugs/Checkmarx）结果
```

### 4.2 优化决策矩阵

| 反馈类型 | 频率 | 影响面 | 优先级 | 响应策略 |
|---------|------|--------|--------|---------|
| 安全隐患 | 低频 | 所有代码 | P0 | 立即修复，全量回归测试 |
| 代码错误 | 中频 | 特定场景 | P1 | 24小时内修复 |
| 风格不符 | 高频 | 代码质量 | P2 | 纳入下一迭代 |
| 不够完整 | 中频 | 开发效率 | P2 | 补充Checklist |
| 过度设计 | 低频 | 维护成本 | P3 | 添加简洁性约束 |

### 4.3 Prompt 版本管理目录结构

```
prompts/
├── java/
│   ├── generate-service/
│   │   ├── v1.md
│   │   ├── v2.md
│   │   ├── v3.md          # 当前版本
│   │   ├── CHANGELOG.md   # 变更记录
│   │   └── test-cases.md  # 测试用例与评测结果
│   ├── generate-controller/
│   └── generate-mapper/
├── sql/
│   ├── dynamic-query/
│   └── batch-insert/
└── review/
    ├── code-review-java/
    └── security-review/
```

每个 Prompt 目录下的 `CHANGELOG.md`：
```markdown
# Changelog - generate-service

## v3.0.0 (2024-12-10)
### 新增
- 添加 @Transactional 事务约束
- 添加参数校验要求

### 修复
- 修正 Exception 处理，不再吞掉原始异常

### 移除
- 移除 Over-engineered 的设计模式要求，添加简洁性约束

### 评测
- 正确率: ↑ 15% (88% → 100%)
- 风格符合度: ↑ 10% (90% → 100%)
- Token成本: ≈

## v2.0.0 (2024-11-15)
...
```

## 五、写在最后

Prompt 优化不是一次性工程，而是一个持续的、渐进的过程。每次 AI 给你不满意输出的瞬间，都是一次优化 Prompt 的机会——问题是你有没有抓住它、记录它、系统化它。

关键心得：
1. **善用反向思考**：有时说"不要做什么"比说"要做什么"更有效（但注意正向约束通常更稳定）
2. **以始为终**：从你期望的最终输出倒推 Prompt 应该有什么内容
3. **小步迭代**：每次只改 1-2 个点，避免一次改太多导致无法判断哪个改动有效
4. **可复现性 > 一次惊艳**：一个稳定的 80 分 Prompt 比一个偶尔能 95 分但经常 60 分的 Prompt 更有价值

---

**下一篇预告**：团队 Prompt 知识库建设——如何把你个人积累的 Prompt 精华变成整个团队的资产。别让每个新人都从零开始摸索，我们已经踩过的坑不该让他们再踩一遍。下一篇我们将详细拆解知识库的架构设计、Git 仓库落地方案、以及如何让团队真正愿意使用和维护它。敬请期待！
