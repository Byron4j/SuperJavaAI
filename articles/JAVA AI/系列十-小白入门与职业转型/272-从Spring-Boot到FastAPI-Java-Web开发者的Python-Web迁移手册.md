# 从 Spring Boot 到 FastAPI：Java Web 开发者的 Python Web 迁移手册

## 开篇：你其实不需要学新框架

作为一个写过上百个 Spring Boot Controller 的 Java 程序员，第一次看到 FastAPI 时我愣住了。

```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/hello")
def hello():
    return {"message": "Hello World"}
```

**4 行代码，一个 Web 服务就跑起来了。**

要知道在 Spring Boot 里，同样的事情你需要：建项目、配置 Maven、写 Application 类、写 Controller 类、加 @RestController 注解、加 @GetMapping 注解...怎么也得十几行。

今天这篇文章，我把 Spring Boot 和 FastAPI 放在一起，逐项对比它们对应功能的写法。读完你会发现：**FastAPI 就是 Python 世界的 Spring Boot**，而且更轻、更快、更好用。

## 一、项目初始化对比

### Spring Boot 方式

```java
// 1. 用 Spring Initializr 生成项目
// 2. pom.xml 里加一堆依赖
// 3. Application.java
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

### FastAPI 方式

```bash
# 1. 安装
pip install fastapi uvicorn

# 2. main.py
```

```python
from fastapi import FastAPI

app = FastAPI(title="我的第一个 FastAPI 项目", version="1.0.0")

@app.get("/")
def root():
    return {"message": "服务已启动"}

# 3. 一行命令启动！
# uvicorn main:app --reload
```

```bash
# 启动命令
uvicorn main:app --reload --port 8080

# 访问 http://localhost:8080
# 自动生成的 API 文档：http://localhost:8080/docs
# 自动生成的 ReDoc 文档：http://localhost:8080/redoc
```

**Spring Boot 你需要额外集成 Swagger 才能看到 API 文档。FastAPI 自带，而且是自动生成的。**

## 二、Controller → 路由定义

### Spring Boot 的 Controller

```java
@RestController
@RequestMapping("/api/users")
public class UserController {
    
    @Autowired
    private UserService userService;
    
    @GetMapping
    public List<User> listUsers() {
        return userService.findAll();
    }
    
    @GetMapping("/{id}")
    public User getUser(@PathVariable Long id) {
        return userService.findById(id);
    }
    
    @PostMapping
    public User createUser(@RequestBody @Valid UserCreateRequest request) {
        return userService.create(request);
    }
    
    @PutMapping("/{id}")
    public User updateUser(@PathVariable Long id, @RequestBody UserUpdateRequest request) {
        return userService.update(id, request);
    }
    
    @DeleteMapping("/{id}")
    public void deleteUser(@PathVariable Long id) {
        userService.delete(id);
    }
}
```

### FastAPI 的路由

```python
from fastapi import FastAPI, HTTPException
from typing import List

app = FastAPI()

# 假数据库
users_db = []
user_counter = 0

@app.get("/api/users", response_model=List[dict])
def list_users():
    return users_db

@app.get("/api/users/{user_id}")
def get_user(user_id: int):
    for user in users_db:
        if user["id"] == user_id:
            return user
    raise HTTPException(status_code=404, detail="用户不存在")

@app.post("/api/users", status_code=201)
def create_user(name: str, email: str, age: int = None):
    global user_counter
    user_counter += 1
    user = {
        "id": user_counter,
        "name": name,
        "email": email,
        "age": age
    }
    users_db.append(user)
    return user

@app.put("/api/users/{user_id}")
def update_user(user_id: int, name: str = None, email: str = None):
    for user in users_db:
        if user["id"] == user_id:
            if name: user["name"] = name
            if email: user["email"] = email
            return user
    raise HTTPException(status_code=404, detail="用户不存在")

@app.delete("/api/users/{user_id}")
def delete_user(user_id: int):
    for i, user in enumerate(users_db):
        if user["id"] == user_id:
            users_db.pop(i)
            return {"message": "删除成功"}
    raise HTTPException(status_code=404, detail="用户不存在")
```

### 关键区别

| 操作 | Spring Boot | FastAPI |
|------|------------|---------|
| 路由定义 | `@GetMapping("/path")` | `@app.get("/path")` |
| 路径参数 | `@PathVariable Long id` | `user_id: int`（参数名匹配） |
| 查询参数 | `@RequestParam String name` | `name: str`（非路径参数自动视为查询参数） |
| 请求体 | `@RequestBody UserDto dto` | 使用 Pydantic 模型 |
| 参数校验 | `@Valid` + Jakarta Validation | Pydantic 自动校验 |
| 返回状态码 | `@ResponseStatus(HttpStatus.CREATED)` | `status_code=201` |

## 三、DTO/Bean → Pydantic 模型

这是 Spring Boot 程序员转 FastAPI 最需要适应的变化。

### Spring Boot 方式

```java
// UserCreateRequest.java
@Getter @Setter
public class UserCreateRequest {
    @NotBlank(message = "用户名不能为空")
    @Size(min = 2, max = 50)
    private String name;
    
    @Email(message = "邮箱格式不正确")
    @NotBlank
    private String email;
    
    @Min(0) @Max(150)
    private Integer age;
}

// UserResponse.java
@Getter @Setter
@AllArgsConstructor
public class UserResponse {
    private Long id;
    private String name;
    private String email;
    private Integer age;
    private LocalDateTime createdAt;
}
```

### FastAPI 方式（Pydantic）

```python
from pydantic import BaseModel, EmailStr, Field
from datetime import datetime
from typing import Optional

class UserCreateRequest(BaseModel):
    name: str = Field(..., min_length=2, max_length=50, description="用户名")
    email: EmailStr = Field(..., description="邮箱")
    age: Optional[int] = Field(None, ge=0, le=150, description="年龄")

class UserResponse(BaseModel):
    id: int
    name: str
    email: str
    age: Optional[int] = None
    created_at: datetime

# 使用 Pydantic 后，上面的 POST 接口可以这样写：
@app.post("/api/users", response_model=UserResponse, status_code=201)
def create_user(request: UserCreateRequest):
    """
    request 已经被 Pydantic 自动校验过了
    如果 name 为空或 age > 150，FastAPI 自动返回 422 错误
    完全不需要手写校验代码！
    """
    user = UserResponse(
        id=generate_id(),
        name=request.name,
        email=request.email,
        age=request.age,
        created_at=datetime.now()
    )
    return user
```

### Pydantic vs Lombok + Validation

```java
// Spring Boot + Lombok + Validation
@Getter @Setter
public class Request {
    @NotBlank @Size(min=2, max=50)
    private String name;
    
    @Email @NotBlank
    private String email;
    
    @Min(0) @Max(150)
    private Integer age;
}
// 要三个库（Lombok, Validation, Jackson），还要编译预处理
```

```python
# Pydantic：一个库搞定所有
class Request(BaseModel):
    name: str = Field(..., min_length=2, max_length=50)
    email: EmailStr
    age: int | None = Field(None, ge=0, le=150)
# 类型声明、校验、序列化、文档生成——全自动
```

**一个 Pydantic 模型 = Lombok + Validation + Jackson + Swagger 注解，而且不需要编译，不需要注解处理器。**

## 四、Service 层和依赖注入

### Spring Boot 的依赖注入

```java
@Service
public class UserService {
    
    @Autowired
    private UserRepository userRepository;
    
    @Autowired
    private EmailService emailService;
    
    public User createUser(UserCreateRequest request) {
        User user = userRepository.save(new User(request));
        emailService.sendWelcomeEmail(user.getEmail());
        return user;
    }
}
```

### FastAPI 的依赖注入——用函数！

```python
from fastapi import Depends

# FastAPI 没有 @Service 和 @Autowired
# 但它有更好的东西：Depends（依赖注入函数）

# 定义一个"依赖"——数据库连接
def get_db():
    db = connect_to_database()
    try:
        yield db  # 返回依赖
    finally:
        db.close()  # 请求结束后自动清理

# 定义一个"依赖"——获取当前用户
def get_current_user(token: str = Header(...)):
    # 从 token 中解析用户信息
    user = verify_token(token)
    if not user:
        raise HTTPException(status_code=401, detail="未授权")
    return user

# 在路由中使用依赖
@app.post("/api/users")
def create_user(
    request: UserCreateRequest,
    db = Depends(get_db),              # 自动注入数据库连接
    current_user = Depends(get_current_user)  # 自动注入当前用户
):
    """
    每个请求过来，FastAPI 自动：
    1. 调用 get_db() → 获取数据库连接 → 注入到 db
    2. 调用 get_current_user() → 获取当前用户 → 注入到 current_user
    3. 请求结束后自动关闭数据库连接
    
    是不是比 @Autowired 更灵活、更直观？
    """
    return user_service.create(db, request, current_user)
```

### 对比总结

| 功能 | Spring Boot | FastAPI |
|------|------------|---------|
| 声明依赖 | `@Autowired` / `@Bean` | `Depends(func)` |
| 注入方式 | 字段注入/构造器注入 | 函数参数注入 |
| 生命周期 | `@Scope("singleton")` 等 | 通过 yield 和 finally 控制 |
| 测试便利性 | 需要 `@MockBean` | 直接传参覆盖依赖 |

## 五、中间件 → Middleware

### Spring Boot 方式

```java
@Component
public class LoggingInterceptor implements HandlerInterceptor {
    @Override
    public boolean preHandle(HttpServletRequest request, 
                            HttpServletResponse response, 
                            Object handler) {
        log.info("Request: {} {}", request.getMethod(), request.getRequestURI());
        return true;
    }
    
    @Override
    public void afterCompletion(...) {
        log.info("Response sent");
    }
}

// 还要在 WebMvcConfigurer 中注册...
```

### FastAPI 方式

```python
import time
from fastapi import Request

@app.middleware("http")
async def logging_middleware(request: Request, call_next):
    """记录每个请求的耗时"""
    start = time.time()
    
    # 请求前
    print(f"--> {request.method} {request.url.path}")
    
    # 执行请求
    response = await call_next(request)
    
    # 请求后
    elapsed = time.time() - start
    print(f"<-- {request.method} {request.url.path} [{elapsed:.3f}s]")
    
    # 可以修改 response
    response.headers["X-Process-Time"] = str(elapsed)
    return response
```

## 六、全局异常处理

### Spring Boot

```java
@RestControllerAdvice
public class GlobalExceptionHandler {
    
    @ExceptionHandler(UserNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleUserNotFound(UserNotFoundException e) {
        return ResponseEntity.status(404)
            .body(new ErrorResponse("USER_NOT_FOUND", e.getMessage()));
    }
    
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleGeneric(Exception e) {
        return ResponseEntity.status(500)
            .body(new ErrorResponse("INTERNAL_ERROR", e.getMessage()));
    }
}
```

### FastAPI

```python
from fastapi import Request
from fastapi.responses import JSONResponse

class UserNotFoundException(Exception):
    pass

@app.exception_handler(UserNotFoundException)
async def user_not_found_handler(request: Request, exc: UserNotFoundException):
    return JSONResponse(
        status_code=404,
        content={"error": "USER_NOT_FOUND", "message": str(exc)}
    )

@app.exception_handler(Exception)
async def generic_handler(request: Request, exc: Exception):
    return JSONResponse(
        status_code=500,
        content={"error": "INTERNAL_ERROR", "message": str(exc)}
    )
```

## 七、配置管理

### Spring Boot：application.yml

```yaml
# application.yml
server:
  port: 8080

app:
  openai:
    api-key: ${OPENAI_API_KEY}
    model: gpt-4o-mini
  database:
    url: jdbc:mysql://localhost:3306/mydb
    username: root
    password: ${DB_PASSWORD}
```

### FastAPI：Pydantic Settings

```python
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    """应用配置——自动从环境变量和 .env 文件读取"""
    
    # OpenAI 配置
    openai_api_key: str = ""
    openai_model: str = "gpt-4o-mini"
    
    # 数据库配置
    database_url: str = "mysql://localhost:3306/mydb"
    database_username: str = "root"
    database_password: str = ""
    
    # FastAPI 配置
    app_name: str = "My AI App"
    debug: bool = False
    
    class Config:
        env_file = ".env"  # 自动读取 .env 文件

# 使用
settings = Settings()
app = FastAPI(title=settings.app_name, debug=settings.debug)

@app.get("/")
def root():
    return {"app": settings.app_name}
```

**.env 文件**：
```bash
OPENAI_API_KEY=sk-your-key-here
DATABASE_PASSWORD=secret123
```

## 八、数据库操作对比

### Spring Boot + JPA

```java
@Entity
@Table(name = "users")
public class User {
    @Id @GeneratedValue
    private Long id;
    
    @Column(nullable = false)
    private String name;
    
    private String email;
}

@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    List<User> findByNameContaining(String name);
}

@Service
public class UserService {
    public User create(String name, String email) {
        User user = new User();
        user.setName(name);
        user.setEmail(email);
        return userRepository.save(user);
    }
}
```

### FastAPI + SQLAlchemy

```python
from sqlalchemy import create_engine, Column, Integer, String
from sqlalchemy.orm import Session, DeclarativeBase

# 数据库连接
engine = create_engine("sqlite:///./app.db")

class Base(DeclarativeBase):
    pass

# 模型定义
class User(Base):
    __tablename__ = "users"
    
    id = Column(Integer, primary_key=True, autoincrement=True)
    name = Column(String(50), nullable=False)
    email = Column(String(100))

# 创建表
Base.metadata.create_all(bind=engine)

# 使用
def create_user(name: str, email: str):
    with Session(engine) as session:
        user = User(name=name, email=email)
        session.add(user)
        session.commit()
        session.refresh(user)
        return user

def find_users_by_name(name: str):
    with Session(engine) as session:
        return session.query(User).filter(User.name.contains(name)).all()
```

## 九、实战：把 Spring Boot 项目迁移到 FastAPI

假设你有一个简单的任务管理系统，现在用 FastAPI 重写。

### 原始 Spring Boot 核心代码

```java
@RestController
@RequestMapping("/api/tasks")
public class TaskController {
    @Autowired private TaskService taskService;
    
    @GetMapping
    public List<Task> list(@RequestParam(defaultValue = "all") String status) {
        return taskService.findByStatus(status);
    }
    
    @PostMapping
    public Task create(@Valid @RequestBody TaskCreateRequest req) {
        return taskService.create(req);
    }
    
    @PutMapping("/{id}")
    public Task update(@PathVariable Long id, @Valid @RequestBody TaskUpdateRequest req) {
        return taskService.update(id, req);
    }
    
    @DeleteMapping("/{id}")
    public void delete(@PathVariable Long id) {
        taskService.delete(id);
    }
}
```

### FastAPI 重写版本

```python
from fastapi import FastAPI, HTTPException, Depends, Query
from pydantic import BaseModel, Field
from datetime import datetime
from typing import Optional, List
from enum import Enum

app = FastAPI(title="任务管理系统")

# ----- 模型 -----
class TaskStatus(str, Enum):
    TODO = "todo"
    IN_PROGRESS = "in_progress"
    DONE = "done"

class TaskCreateRequest(BaseModel):
    title: str = Field(..., min_length=1, max_length=100)
    description: Optional[str] = None
    priority: int = Field(default=1, ge=1, le=5)

class TaskUpdateRequest(BaseModel):
    title: Optional[str] = None
    description: Optional[str] = None
    status: Optional[TaskStatus] = None
    priority: Optional[int] = Field(None, ge=1, le=5)

class TaskResponse(BaseModel):
    id: int
    title: str
    description: Optional[str]
    status: TaskStatus
    priority: int
    created_at: datetime
    updated_at: datetime

# ----- 模拟数据库 -----
tasks_db = []
counter = 0

# ----- 依赖（相当于 Service） -----
def get_task_service():
    """返回任务服务——这里用闭包模拟"""
    return TaskService()

class TaskService:
    def list(self, status: str = "all"):
        if status == "all":
            return tasks_db
        return [t for t in tasks_db if t["status"] == status]
    
    def create(self, request: TaskCreateRequest):
        global counter
        counter += 1
        now = datetime.now()
        task = {
            "id": counter,
            "title": request.title,
            "description": request.description,
            "status": TaskStatus.TODO.value,
            "priority": request.priority,
            "created_at": now,
            "updated_at": now
        }
        tasks_db.append(task)
        return task
    
    def update(self, id: int, request: TaskUpdateRequest):
        for task in tasks_db:
            if task["id"] == id:
                if request.title is not None:
                    task["title"] = request.title
                if request.description is not None:
                    task["description"] = request.description
                if request.status is not None:
                    task["status"] = request.status.value
                if request.priority is not None:
                    task["priority"] = request.priority
                task["updated_at"] = datetime.now()
                return task
        raise HTTPException(404, "任务不存在")
    
    def delete(self, id: int):
        for i, task in enumerate(tasks_db):
            if task["id"] == id:
                tasks_db.pop(i)
                return
        raise HTTPException(404, "任务不存在")

# ----- 路由 -----
@app.get("/api/tasks", response_model=List[TaskResponse])
def list_tasks(
    status: str = Query("all", description="按状态筛选"),
    service: TaskService = Depends(get_task_service)
):
    return service.list(status)

@app.post("/api/tasks", response_model=TaskResponse, status_code=201)
def create_task(
    request: TaskCreateRequest,
    service: TaskService = Depends(get_task_service)
):
    return service.create(request)

@app.put("/api/tasks/{id}", response_model=TaskResponse)
def update_task(
    id: int,
    request: TaskUpdateRequest,
    service: TaskService = Depends(get_task_service)
):
    return service.update(id, request)

@app.delete("/api/tasks/{id}")
def delete_task(
    id: int,
    service: TaskService = Depends(get_task_service)
):
    service.delete(id)
    return {"message": "删除成功"}
```

```bash
# 启动
uvicorn main:app --reload

# 测试
curl http://localhost:8000/api/tasks
curl -X POST http://localhost:8000/api/tasks \
  -H "Content-Type: application/json" \
  -d '{"title": "学习FastAPI", "priority": 5}'

# 查看自动生成的 API 文档
# 打开浏览器访问 http://localhost:8000/docs
```

## 十、迁移检查清单

当你把一个 Spring Boot 项目迁移到 FastAPI 时，按这个清单走：

- [ ] **Controller → 路由函数**：`@GetMapping` → `@app.get()`
- [ ] **DTO → Pydantic 模型**：Lombok + Validation → Pydantic BaseModel
- [ ] **Service → Depends 注入**：`@Service` + `@Autowired` → 函数 + `Depends()`
- [ ] **Interceptor/Filter → Middleware**：`HandlerInterceptor` → `@app.middleware("http")`
- [ ] **@ControllerAdvice → exception_handler**：全局异常处理 → `@app.exception_handler()`
- [ ] **application.yml → Pydantic Settings**：配置文件 → `BaseSettings` + `.env`
- [ ] **JPA → SQLAlchemy**：ORM 框架替换
- [ ] **Spring Security → python-jose + passlib**：认证框架替换
- [ ] **@Scheduled → BackgroundTasks / Celery**：定时任务替换
- [ ] **Spring Cloud → 使用 k8s / 云原生方案**：微服务框架

## 十一、性能对比（冷知识）

```bash
# Spring Boot 启动时间（一个空项目）
# 约 2-5 秒（取决于依赖数量）

# FastAPI 启动时间
# 约 0.5 秒

# Spring Boot 内存占用（一个空项目）
# 约 200-400 MB

# FastAPI 内存占用（一个空项目）
# 约 30-50 MB

# 处理 1000 个并发请求
# Spring Boot: 约 500-2000 req/s（取决于配置）
# FastAPI: 约 3000-10000 req/s（异步原生支持）

# 注意：FastAPI 的优势在于 I/O 密集型任务（数据库查询、API调用）
# 如果是 CPU 密集型任务，两者差别不大
```

---

**下篇预告**：Python 环境管理是新手噩梦——pip、conda、poetry 到底用哪个？.venv 文件夹又是什么？下一篇我把坑全帮你踩平，让你不再被"环境配不好"劝退。
