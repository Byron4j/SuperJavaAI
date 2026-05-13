# Copilot Chat 实战：用自然语言重构遗留 Java 代码

> 你接手了一个 Struts 项目，Ctrl+C/V 了三年，终于你决定：不能再这样苟且下去了。

---

## 一、那个下午，我打开了传说中的"祖传代码"

2024 年某个周三下午，leader 在群里 @ 我：

> "老王离职了，他负责的交易对账模块你接一下，下周上线一个新渠道。"

我点开项目目录的那一刻，手指微微颤抖——

- **框架**：Struts 1.x + JDK 1.6
- **持久层**：JDBC 直连，SQL 硬编码在 Java 文件里
- **依赖管理**：没有 Maven，没有 Gradle，lib 文件夹 147 个 jar 包
- **最离谱的**：一个 `UserController.java`，**2100 行**，里面有 137 个方法

那天下午，我抽了半包烟，喝了三杯咖啡，然后在 commit message 里写下：

```
fix: 添加新渠道
```

然后默默关掉了 IDE。

**这，就是中国 80% Java 开发者的日常。**

> 金句：**"祖传代码不可怕，可怕的是你对它产生了感情。"**

---

## 二、Copilot Chat 是什么？为什么它是重构利器？

GitHub Copilot Chat 是 VS Code / JetBrains IDE 中的对话式 AI 助手（基于 GPT-4）。和普通 ChatGPT 不同，它最大的优势是：

| 特性 | Copilot Chat | ChatGPT 网页版 |
|------|-------------|---------------|
| 上下文感知 | 自动读取当前打开的文件 | 需要手动粘贴代码 |
| IDE 集成 | 代码直接插入编辑器 | 需要复制粘贴 |
| 项目理解 | 可引用 `#file`、`#selection` | 无项目上下文 |
| 效率 | 5 秒响应 | 来回切换窗口 |

**简单说：Copilot Chat 就像你身边坐了一个 10 年经验的架构师，你只需要用自然语言告诉他你想干嘛。**

---

## 三、实战：一个 Struts 遗留项目的重生之路

### 3.1 原始代码长什么样？

下面这段代码是我根据真实案例模拟的——它几乎集齐了所有"祖传代码"的要素：

```java
// UserController.java —— Struts 1.x 时代的"上帝类"
public class UserController extends Action {

    // 硬编码的数据库连接信息
    private static final String DB_URL = "jdbc:mysql://10.0.0.53:3306/legacy_db";
    private static final String DB_USER = "root";
    private static final String DB_PASS = "root123456";

    public ActionForward execute(ActionMapping mapping, ActionForm form,
            HttpServletRequest request, HttpServletResponse response) {
        
        String action = request.getParameter("action");
        
        if ("list".equals(action)) {
            return listUser(mapping, form, request, response);
        } else if ("add".equals(action)) {
            return addUser(mapping, form, request, response);
        } else if ("delete".equals(action)) {
            return deleteUser(mapping, form, request, response);
        } else if ("update".equals(action)) {
            return updateUser(mapping, form, request, response);
        }
        return mapping.findForward("error");
    }

    private ActionForward listUser(ActionMapping mapping, ActionForm form,
            HttpServletRequest request, HttpServletResponse response) {
        Connection conn = null;
        Statement stmt = null;
        ResultSet rs = null;
        List<User> userList = new ArrayList<>();
        
        try {
            Class.forName("com.mysql.jdbc.Driver");
            conn = DriverManager.getConnection(DB_URL, DB_USER, DB_PASS);
            
            String keyword = request.getParameter("keyword");
            String sortBy = request.getParameter("sortBy");
            String sortOrder = request.getParameter("sortOrder");
            
            // 手动拼接SQL —— SQL注入风险！
            StringBuilder sql = new StringBuilder("SELECT * FROM t_user WHERE 1=1");
            if (keyword != null && !keyword.isEmpty()) {
                sql.append(" AND (username LIKE '%" + keyword + "%' OR email LIKE '%" 
                    + keyword + "%')");
            }
            if (sortBy != null && !sortBy.isEmpty()) {
                sql.append(" ORDER BY " + sortBy);
                if ("desc".equalsIgnoreCase(sortOrder)) {
                    sql.append(" DESC");
                } else {
                    sql.append(" ASC");
                }
            }
            
            stmt = conn.createStatement();
            rs = stmt.executeQuery(sql.toString());
            
            while (rs.next()) {
                User user = new User();
                user.setId(rs.getInt("id"));
                user.setUsername(rs.getString("username"));
                user.setEmail(rs.getString("email"));
                user.setPhone(rs.getString("phone"));
                user.setStatus(rs.getInt("status"));
                user.setCreateTime(rs.getDate("create_time"));
                userList.add(user);
            }
            
            // 手动分页
            int page = 1;
            int pageSize = 10;
            try { page = Integer.parseInt(request.getParameter("page")); } catch(Exception e){}
            try { pageSize = Integer.parseInt(request.getParameter("pageSize")); } catch(Exception e){}
            int total = userList.size();
            int fromIndex = (page - 1) * pageSize;
            int toIndex = Math.min(fromIndex + pageSize, total);
            if (fromIndex < total) {
                userList = userList.subList(fromIndex, toIndex);
            }
            
            request.setAttribute("userList", userList);
            request.setAttribute("total", total);
            request.setAttribute("page", page);
            
            return mapping.findForward("list");
            
        } catch (Exception e) {
            e.printStackTrace(); // 吃掉了异常！
            request.setAttribute("error", "系统错误");
            return mapping.findForward("error");
        } finally {
            try { if (rs != null) rs.close(); } catch(Exception e) {}
            try { if (stmt != null) stmt.close(); } catch(Exception e) {}
            try { if (conn != null) conn.close(); } catch(Exception e) {}
        }
    }

    private ActionForward addUser(ActionMapping mapping, ActionForm form,
            HttpServletRequest request, HttpServletResponse response) {
        Connection conn = null;
        PreparedStatement ps = null;
        
        try {
            Class.forName("com.mysql.jdbc.Driver");
            conn = DriverManager.getConnection(DB_URL, DB_USER, DB_PASS);
            
            String username = request.getParameter("username");
            String email = request.getParameter("email");
            String phone = request.getParameter("phone");
            
            // 参数校验靠"信任"
            if (username == null || username.trim().isEmpty()) {
                request.setAttribute("error", "用户名不能为空");
                return mapping.findForward("error");
            }
            if (email == null || !email.contains("@")) {
                request.setAttribute("error", "邮箱格式不正确");
                return mapping.findForward("error");
            }
            
            String sql = "INSERT INTO t_user (username, email, phone, status, create_time) "
                + "VALUES (?, ?, ?, 1, NOW())";
            ps = conn.prepareStatement(sql);
            ps.setString(1, username);
            ps.setString(2, email);
            ps.setString(3, phone);
            ps.executeUpdate();
            
            request.setAttribute("message", "添加成功");
            return mapping.findForward("success");
            
        } catch (Exception e) {
            e.printStackTrace();
            request.setAttribute("error", "添加失败：" + e.getMessage());
            return mapping.findForward("error");
        } finally {
            try { if (ps != null) ps.close(); } catch(Exception e) {}
            try { if (conn != null) conn.close(); } catch(Exception e) {}
        }
    }

    private ActionForward deleteUser(ActionMapping mapping, ActionForm form,
            HttpServletRequest request, HttpServletResponse response) {
        Connection conn = null;
        PreparedStatement ps = null;
        
        try {
            Class.forName("com.mysql.jdbc.Driver");
            conn = DriverManager.getConnection(DB_URL, DB_USER, DB_PASS);
            
            String id = request.getParameter("id");
            String sql = "DELETE FROM t_user WHERE id = ?";
            ps = conn.prepareStatement(sql);
            ps.setInt(1, Integer.parseInt(id));
            int rows = ps.executeUpdate();
            
            if (rows > 0) {
                request.setAttribute("message", "删除成功");
            } else {
                request.setAttribute("error", "用户不存在");
            }
            return mapping.findForward("success");
            
        } catch (Exception e) {
            e.printStackTrace();
            request.setAttribute("error", "删除失败");
            return mapping.findForward("error");
        } finally {
            try { if (ps != null) ps.close(); } catch(Exception e) {}
            try { if (conn != null) conn.close(); } catch(Exception e) {}
        }
    }

    private ActionForward updateUser(ActionMapping mapping, ActionForm form,
            HttpServletRequest request, HttpServletResponse response) {
        Connection conn = null;
        PreparedStatement ps = null;
        
        try {
            Class.forName("com.mysql.jdbc.Driver");
            conn = DriverManager.getConnection(DB_URL, DB_USER, DB_PASS);
            
            String id = request.getParameter("id");
            String username = request.getParameter("username");
            String email = request.getParameter("email");
            String phone = request.getParameter("phone");
            String status = request.getParameter("status");
            
            String sql = "UPDATE t_user SET username=?, email=?, phone=?, status=? WHERE id=?";
            ps = conn.prepareStatement(sql);
            ps.setString(1, username);
            ps.setString(2, email);
            ps.setString(3, phone);
            ps.setInt(4, Integer.parseInt(status));
            ps.setInt(5, Integer.parseInt(id));
            ps.executeUpdate();
            
            request.setAttribute("message", "更新成功");
            return mapping.findForward("success");
            
        } catch (Exception e) {
            e.printStackTrace();
            request.setAttribute("error", "更新失败");
            return mapping.findForward("error");
        } finally {
            try { if (ps != null) ps.close(); } catch(Exception e) {}
            try { if (conn != null) conn.close(); } catch(Exception e) {}
        }
    }
}
```

> **来，我们一起数一数这段代码有多少个"坏味道"：**

- 数据库连接四散各处，每个方法自己 `DriverManager.getConnection`
- SQL 字符串拼接，SQL 注入风险拉满
- 异常全被 `e.printStackTrace()` 吃掉，生产环境出了事你根本查不到
- Connection / Statement / ResultSet 手动关闭，`finally` 块套了三层
- 分页用 `subList` 在内存里做，100 万条数据直接 OOM
- 密码明文写在代码里，一份代码就是一个安全漏洞
- 没有任何分层：Controller 里干完了 DAO 的活、Service 的活、Validator 的活

**如果你接手过类似项目，请在评论区扣"1"。**

---

### 3.2 重构第一步：让 AI 理解现状

打开 Copilot Chat（快捷键：`Ctrl+Shift+I` 或 `Cmd+Shift+I`），第一步，我直接选中文全文，然后输入：

> **Prompt 1：**
> ```
> #selection 帮我分析这段 Java 代码存在哪些问题，按严重程度排序，
> 包括安全风险、性能问题、可维护性问题、架构问题。
> ```

**Copilot Chat 返回：**

```
这段代码存在以下问题，按严重程度排序：

【严重】安全问题：
1. 数据库密码硬编码在代码中（第4行）
2. SQL 注入风险：keyword 参数直接拼接到 SQL（第37-39行）
3. sortBy/sortOrder 直接拼接，可被注入恶意 SQL
4. 数据库连接使用 root 账户，权限过大

【严重】架构问题：
1. 所有逻辑集中在一个类，违反单一职责原则
2. Controller 直接操作 JDBC API，没有分层
3. 硬编码 JDBC 驱动和连接字符串

【高】性能问题：
1. 每次请求都新建数据库连接，没有连接池
2. 分页使用 subList 在内存中做，数据量大时会 OOM
3. SELECT * 查询所有字段，无法利用索引覆盖

【中】可维护性问题：
1. 重复代码：每个方法都有相同的连接创建/关闭逻辑
2. 异常被 e.printStackTrace() 吞噬，没有日志框架
3. 没有参数校验框架
4. 魔法数字：status=1 没有常量定义
5. 空 try-catch 块忽略异常（兜底 try）

建议：迁移到 Spring Boot + JPA/MyBatis，引入连接池、日志框架、校验框架。
```

> 这一步的价值：**让 AI 帮你做代码审查，比你一个个肉眼去看要快 10 倍。**

---

### 3.3 重构第二步：提取数据库访问层

我选中了每个方法里重复出现的 JDBC 连接代码，包括 `finally` 块里的资源释放，然后输入：

> **Prompt 2：**
> ```
> #selection 将这些重复的 JDBC 连接和资源释放代码，抽取成一个
> 单独的 DBUtil 工具类。要求：
> 1. 使用 try-with-resources 简化资源管理
> 2. 使用 HikariCP 连接池替代 DriverManager
> 3. 数据库配置从 application.properties 读取
> 4. 支持返回 List<Map<String, Object>> 的查询方法
> 5. 支持带 PreparedStatement 参数的增删改方法
> 6. 使用 Java 8 语法
> ```

**Copilot Chat 生成了：**

```java
// DBUtil.java
import com.zaxxer.hikari.HikariConfig;
import com.zaxxer.hikari.HikariDataSource;

import javax.sql.DataSource;
import java.io.InputStream;
import java.sql.*;
import java.util.*;

public class DBUtil {
    
    private static final DataSource dataSource;
    
    static {
        Properties props = new Properties();
        try (InputStream is = DBUtil.class.getClassLoader()
                .getResourceAsStream("application.properties")) {
            props.load(is);
        } catch (Exception e) {
            throw new RuntimeException("Failed to load DB config", e);
        }
        
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl(props.getProperty("db.url"));
        config.setUsername(props.getProperty("db.username"));
        config.setPassword(props.getProperty("db.password"));
        config.setMaximumPoolSize(Integer.parseInt(
            props.getProperty("db.pool.max-size", "20")));
        config.setMinimumIdle(Integer.parseInt(
            props.getProperty("db.pool.min-idle", "5")));
        dataSource = new HikariDataSource(config);
    }
    
    public static List<Map<String, Object>> query(String sql, Object... params) {
        List<Map<String, Object>> result = new ArrayList<>();
        try (Connection conn = dataSource.getConnection();
             PreparedStatement ps = conn.prepareStatement(sql)) {
            for (int i = 0; i < params.length; i++) {
                ps.setObject(i + 1, params[i]);
            }
            try (ResultSet rs = ps.executeQuery()) {
                ResultSetMetaData meta = rs.getMetaData();
                while (rs.next()) {
                    Map<String, Object> row = new LinkedHashMap<>();
                    for (int i = 1; i <= meta.getColumnCount(); i++) {
                        row.put(meta.getColumnLabel(i), rs.getObject(i));
                    }
                    result.add(row);
                }
            }
        } catch (SQLException e) {
            throw new RuntimeException("Query failed: " + sql, e);
        }
        return result;
    }
    
    public static int update(String sql, Object... params) {
        try (Connection conn = dataSource.getConnection();
             PreparedStatement ps = conn.prepareStatement(sql)) {
            for (int i = 0; i < params.length; i++) {
                ps.setObject(i + 1, params[i]);
            }
            return ps.executeUpdate();
        } catch (SQLException e) {
            throw new RuntimeException("Update failed: " + sql, e);
        }
    }
}
```

> 你可能注意到了，我让它生成的是一个**低侵入性**的工具类，而不是直接上 MyBatis。为什么？**渐进式重构——先挪开一块砖，再拆一堵墙。** 如果你上来就说"用 MyBatis-Plus 重写整个项目"，大概率会死在半路上。

---

### 3.4 重构第三步：用 DAO 模式替换硬编码 SQL

我选中 `listUser` 方法里的整段 SQL 拼接和结果集映射逻辑，输入：

> **Prompt 3：**
> ```
> #selection 把这段查询逻辑抽取到 UserDao 类中。
> 要求：
> 1. SQL 语句使用 PreparedStatement 参数化，防止 SQL 注入
> 2. sortBy 和 sortOrder 使用白名单校验（只允许 id, username, create_time）
> 3. 返回 List<User> 而非 List<Map>
> 4. 使用 DBUtil（刚才生成的工具类）
> 5. 添加分页参数 int page, int pageSize，在 SQL 层面用 LIMIT 分页
> 6. 返回 PageResult 对象，包含 list, total, page, pageSize
> ```

**Copilot Chat 生成了：**

```java
// PageResult.java
public class PageResult<T> {
    private List<T> list;
    private long total;
    private int page;
    private int pageSize;
    
    public PageResult(List<T> list, long total, int page, int pageSize) {
        this.list = list;
        this.total = total;
        this.page = page;
        this.pageSize = pageSize;
    }
    // getters...
}

// UserDao.java
import java.util.*;

public class UserDao {
    
    private static final Set<String> ALLOWED_SORT_COLUMNS = 
        new HashSet<>(Arrays.asList("id", "username", "create_time"));
    
    public PageResult<User> listUsers(String keyword, String sortBy, 
            String sortOrder, int page, int pageSize) {
        
        // 构建 WHERE 条件
        StringBuilder whereClause = new StringBuilder();
        List<Object> params = new ArrayList<>();
        
        if (keyword != null && !keyword.trim().isEmpty()) {
            whereClause.append(" AND (username LIKE ? OR email LIKE ?)");
            params.add("%" + keyword + "%");
            params.add("%" + keyword + "%");
        }
        
        // 排序字段白名单校验
        String orderBy = "id";
        if (sortBy != null && ALLOWED_SORT_COLUMNS.contains(sortBy)) {
            orderBy = sortBy;
        }
        String direction = "desc".equalsIgnoreCase(sortOrder) ? "DESC" : "ASC";
        
        // 查询总数
        String countSql = "SELECT COUNT(*) FROM t_user WHERE 1=1" + whereClause;
        long total = ((Number) DBUtil.query(countSql, params.toArray())
                .get(0).values().iterator().next()).longValue();
        
        // 分页查询
        int offset = (page - 1) * pageSize;
        String listSql = "SELECT * FROM t_user WHERE 1=1" + whereClause 
            + " ORDER BY " + orderBy + " " + direction
            + " LIMIT " + offset + ", " + pageSize;
        
        List<Map<String, Object>> rows = DBUtil.query(listSql, params.toArray());
        List<User> users = new ArrayList<>();
        for (Map<String, Object> row : rows) {
            users.add(mapToUser(row));
        }
        
        return new PageResult<>(users, total, page, pageSize);
    }
    
    public int addUser(String username, String email, String phone) {
        String sql = "INSERT INTO t_user (username, email, phone, status, create_time) "
            + "VALUES (?, ?, ?, 1, NOW())";
        return DBUtil.update(sql, username, email, phone);
    }
    
    public int deleteUser(int id) {
        return DBUtil.update("DELETE FROM t_user WHERE id = ?", id);
    }
    
    public int updateUser(int id, String username, String email, 
            String phone, int status) {
        String sql = "UPDATE t_user SET username=?, email=?, phone=?, status=? WHERE id=?";
        return DBUtil.update(sql, username, email, phone, status, id);
    }
    
    private User mapToUser(Map<String, Object> row) {
        User user = new User();
        user.setId(((Number) row.get("id")).intValue());
        user.setUsername((String) row.get("username"));
        user.setEmail((String) row.get("email"));
        user.setPhone((String) row.get("phone"));
        user.setStatus(((Number) row.get("status")).intValue());
        user.setCreateTime((java.util.Date) row.get("create_time"));
        return user;
    }
}
```

> 关键点解读：**我给 sortBy 加了白名单校验**——这在 Prompt 里明确写了，AI 老老实实执行了。这就是"用自然语言编程"的正确姿势：**你必须告诉它你的安全诉求，AI 不是默认安全的。**

---

### 3.5 重构第四步：加上 Service 层和参数校验

我选中原来的 `addUser` 逻辑，输入：

> **Prompt 4：**
> ```
> #selection 创建一个 UserService 类，添加以下方法：
> 1. 使用 @Valid 和 JSR-303 注解做参数校验
> 2. 使用 @Transactional 管理事务（Spring 声明式事务）
> 3. 统一异常处理，定义 BusinessException
> 4. 用户名的唯一性校验（调用 UserDao 检查是否已存在）
> 5. 日志使用 Slf4j，关键操作记录 INFO 日志
> ```

**Copilot Chat 返回：**

```java
// BusinessException.java
public class BusinessException extends RuntimeException {
    private int code;
    
    public BusinessException(int code, String message) {
        super(message);
        this.code = code;
    }
    public int getCode() { return code; }
}

// UserDTO.java
import javax.validation.constraints.*;

public class UserDTO {
    private Integer id;
    
    @NotBlank(message = "用户名不能为空")
    @Size(min = 2, max = 20, message = "用户名长度2-20位")
    private String username;
    
    @NotBlank(message = "邮箱不能为空")
    @Email(message = "邮箱格式不正确")
    private String email;
    
    @Pattern(regexp = "^1[3-9]\\d{9}$", message = "手机号格式不正确")
    private String phone;
    
    @NotNull(message = "状态不能为空")
    @Min(value = 0, message = "状态值无效")
    @Max(value = 1, message = "状态值无效")
    private Integer status;
    // getters, setters...
}

// UserService.java
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import javax.validation.Valid;

@Slf4j
@Service
public class UserService {
    
    private final UserDao userDao;
    
    public UserService(UserDao userDao) {
        this.userDao = userDao;
    }
    
    public PageResult<User> listUsers(String keyword, String sortBy,
            String sortOrder, int page, int pageSize) {
        log.info("查询用户列表: keyword={}, page={}, pageSize={}", keyword, page, pageSize);
        return userDao.listUsers(keyword, sortBy, sortOrder, page, pageSize);
    }
    
    @Transactional
    public void addUser(@Valid UserDTO dto) {
        log.info("新增用户: username={}", dto.getUsername());
        
        // 用户名唯一性校验
        PageResult<User> existing = userDao.listUsers(dto.getUsername(), null, null, 1, 1);
        if (existing.getTotal() > 0) {
            throw new BusinessException(1001, "用户名已存在");
        }
        
        int rows = userDao.addUser(dto.getUsername(), dto.getEmail(), dto.getPhone());
        if (rows == 0) {
            throw new BusinessException(1002, "添加用户失败");
        }
        log.info("用户添加成功: username={}", dto.getUsername());
    }
    
    @Transactional
    public void deleteUser(int id) {
        log.info("删除用户: id={}", id);
        int rows = userDao.deleteUser(id);
        if (rows == 0) {
            throw new BusinessException(1003, "用户不存在或已被删除");
        }
    }
    
    @Transactional
    public void updateUser(@Valid UserDTO dto) {
        log.info("更新用户: id={}", dto.getId());
        userDao.updateUser(dto.getId(), dto.getUsername(), 
            dto.getEmail(), dto.getPhone(), dto.getStatus());
    }
}
```

---

### 3.6 重构第五步：生成全新的 Controller

这是最关键的一步。我选中原 Struts Action 的**全部代码**，输入：

> **Prompt 5：**
> ```
> #selection 将上述 Struts Action 代码重写为 Spring Boot REST Controller。
> 要求：
> 1. 使用 @RestController + @RequestMapping
> 2. RESTful 风格：GET /api/users, POST /api/users, PUT /api/users/{id}, DELETE /api/users/{id}
> 3. 使用 UserService 处理业务逻辑
> 4. 统一返回 Result<T> 包装类（code, message, data）
> 5. 使用 @Valid 校验请求参数
> 6. 全局异常处理器 GlobalExceptionHandler
> 7. 分页参数 page/pageSize 通过 @RequestParam 接收，默认值 page=1, pageSize=10
> 8. 添加 @ApiOperation（Swagger）注解方便接口文档
> ```

**Copilot Chat 生成：**

```java
// Result.java
public class Result<T> {
    private int code;
    private String message;
    private T data;
    
    public static <T> Result<T> success(T data) {
        Result<T> r = new Result<>();
        r.code = 200;
        r.message = "success";
        r.data = data;
        return r;
    }
    
    public static <T> Result<T> error(int code, String message) {
        Result<T> r = new Result<>();
        r.code = code;
        r.message = message;
        return r;
    }
    // getters...
}

// GlobalExceptionHandler.java
import lombok.extern.slf4j.Slf4j;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

@Slf4j
@RestControllerAdvice
public class GlobalExceptionHandler {
    
    @ExceptionHandler(BusinessException.class)
    public Result<Void> handleBusinessException(BusinessException e) {
        log.warn("业务异常: code={}, message={}", e.getCode(), e.getMessage());
        return Result.error(e.getCode(), e.getMessage());
    }
    
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public Result<Void> handleValidation(MethodArgumentNotValidException e) {
        String msg = e.getBindingResult().getFieldErrors().stream()
                .map(f -> f.getField() + ": " + f.getDefaultMessage())
                .reduce((a, b) -> a + "; " + b).orElse("参数校验失败");
        return Result.error(400, msg);
    }
    
    @ExceptionHandler(Exception.class)
    public Result<Void> handleException(Exception e) {
        log.error("系统异常", e);
        return Result.error(500, "系统繁忙，请稍后再试");
    }
}

// UserController.java
import io.swagger.annotations.Api;
import io.swagger.annotations.ApiOperation;
import org.springframework.web.bind.annotation.*;

import javax.validation.Valid;

@Api(tags = "用户管理")
@RestController
@RequestMapping("/api/users")
public class UserController {
    
    private final UserService userService;
    
    public UserController(UserService userService) {
        this.userService = userService;
    }
    
    @ApiOperation("分页查询用户列表")
    @GetMapping
    public Result<PageResult<User>> list(
            @RequestParam(required = false) String keyword,
            @RequestParam(required = false) String sortBy,
            @RequestParam(defaultValue = "asc") String sortOrder,
            @RequestParam(defaultValue = "1") int page,
            @RequestParam(defaultValue = "10") int pageSize) {
        return Result.success(userService.listUsers(keyword, sortBy, sortOrder, page, pageSize));
    }
    
    @ApiOperation("新增用户")
    @PostMapping
    public Result<Void> add(@Valid @RequestBody UserDTO dto) {
        userService.addUser(dto);
        return Result.success(null);
    }
    
    @ApiOperation("更新用户")
    @PutMapping("/{id}")
    public Result<Void> update(@PathVariable int id, @Valid @RequestBody UserDTO dto) {
        dto.setId(id);
        userService.updateUser(dto);
        return Result.success(null);
    }
    
    @ApiOperation("删除用户")
    @DeleteMapping("/{id}")
    public Result<Void> delete(@PathVariable int id) {
        userService.deleteUser(id);
        return Result.success(null);
    }
}
```

---

## 四、重构前后对比

| 维度 | 重构前（Struts Action） | 重构后（Spring Boot） |
|------|----------------------|---------------------|
| 代码行数 | ~2100 行（整个类） | Controller 50 行 + Service 60 行 + DAO 80 行 |
| 单方法行数 | 最长 120 行 | 最长 20 行 |
| 数据库连接 | 每次请求新建连接 | HikariCP 连接池 |
| SQL 安全 | 字符串拼接，SQL 注入 | PreparedStatement 参数化 |
| 异常处理 | `e.printStackTrace()` | 全局异常处理器 + Slf4j |
| 参数校验 | 手写 if-else | `@Valid` + JSR-303 |
| 事务管理 | 无 | `@Transactional` 声明式事务 |
| 可测试性 | 强依赖 Servlet API | 依赖注入，可 Mock |
| 配置管理 | 硬编码 | `application.properties` |

---

## 五、遗留代码 AI 重构 Checklist

在与 Copilot Chat 配合重构了十几个遗留项目之后，我总结出了一套标准流程。**建议保存这张图：**

```
┌────────────────── AI 重构五步法 ──────────────────┐
│                                                    │
│  1. 【摸底】让 AI 分析现有代码的问题               │
│     Prompt: "分析这段代码的问题，按严重程度排序"    │
│     ─────────────────────────────────────           │
│  2. 【解耦】先抽离基础设施（连接池、工具类）        │
│     Prompt: "把重复的XX逻辑抽取为独立工具类"         │
│     ─────────────────────────────────────           │
│  3. 【分层】创建 DAO/Repository 层                  │
│     Prompt: "把 SQL 逻辑抽取到 XXDao 中"            │
│     ─────────────────────────────────────           │
│  4. 【封装】创建 Service 层 + 参数校验              │
│     Prompt: "创建 XXService，添加参数校验和事务"     │
│     ─────────────────────────────────────           │
│  5. 【替换】生成新的 Controller 或 API              │
│     Prompt: "将 Struts Action 重写为 @RestController"│
│                                                    │
└────────────────────────────────────────────────────┘
```

### 关键原则

- **小步快跑**：一次只改一层，改完立刻跑测试
- **保持可编译**：每一步完成时代码必须能通过编译
- **先加后减**：新增模块先写好，能跑通后再删旧代码
- **Prompt 要具体**：别说"帮我重构"，要说"把 SQL 拼接改成 PreparedStatement 参数化"
- **上下文要给足**：用 `#file`、`#selection` 让 AI 看到关联代码

### 反复验证的 6 个 Prompt 模板

| 场景 | Prompt 模板 |
|------|------------|
| 代码审查 | `#selection 分析这段代码的安全风险和性能问题` |
| 抽取工具类 | `#selection 把这段重复逻辑抽取为独立的工具类，使用连接池` |
| 生成 DAO | `#selection 生成对应的 DAO 层，使用 JdbcTemplate` |
| 生成 Service | `#selection 生成 Service 层，添加事务和校验` |
| 生成 Controller | `#selection 生成 REST Controller，RESTful 风格` |
| 生成测试 | `#selection 为这个 Service 生成单元测试，覆盖率 > 80%` |

---

## 六、写在最后

重构遗留代码，从来不是一个技术问题，而是一个心理问题。

你害怕改坏，害怕背锅，害怕"能跑就行"的东西在你手里崩了。所以一拖再拖，代码债越欠越多。

但 AI 编程工具的出现，正在改变这个局面——**它帮你把"重构"这件高心理成本的事，拆解成了一连串低成本的对话。**

你只需要对着 Copilot Chat 说一句话，它就帮你写好一整个 DAO 层。你只需要选中一段烂代码，它就能告诉你隐患在哪里。

> 金句：**"AI 不会替你写代码，但它能替你壮胆。"**

---

### 💬 聊天区留给你

你在工作中接手过最"精彩"的遗留代码是什么？是连注释都没有的 3000 行 if-else？还是用中文拼音命名的变量？还是那个"千万别动这行注释掉但不知道为啥注释"的代码？

**评论区聊聊，点赞最高的三位，我送一本《重构：改善既有代码的设计（第2版）》电子版。**

---

*下一篇预告：**《Copilot 自动生成单元测试：从 0% 到 85% 覆盖率，我做了这三件事》**——看 AI 如何为几个月没写测试的老代码补上安全网。关注我，别错过。*

---

> 本文使用 GitHub Copilot Chat（GPT-4）辅助撰写。所有 Prompt 均为真实可用，欢迎直接复制到你的 IDE 中尝试。
