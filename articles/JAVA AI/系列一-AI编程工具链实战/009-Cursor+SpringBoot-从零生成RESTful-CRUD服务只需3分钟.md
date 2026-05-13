# Cursor + Spring Boot：从零生成一个 RESTful CRUD 服务只需 3 分钟，连 Postman 测试用例都帮你写好

> 你还在手写 Controller、Service、Repository？那你一定还没体验过 Cursor Composer 的“一句话建项目”。

---

## 一、开篇：这不是科幻，这是 2025 年的日常

想象一下这个画面——你坐在工位上，打开 Cursor，创建了一个空目录。按下 `Cmd+I`，呼出 Composer 模式。然后，你像跟同事说话一样，敲下一行中文需求。

回车之后，屏幕左侧的对话开始快速滚动，右侧的文件树里一个个文件自动冒出来——就像有一双无形的手在帮你写代码。

你去接了杯水，和旁边同事聊了两句周末的安排。

回来的时候，水还没喝到一半，你发现屏幕上已经安静下来了。低头一看——**一个完整的 Spring Boot 项目安安静静地躺在目录里，16 个文件，一个不少。**

从打开空目录到拥有一个可以立即跑起来的 RESTful CRUD 服务，全程不超过三分钟。

让我们看看这三分钟里究竟发生了什么。

```
book-manager/
├── pom.xml
├── src/main/java/com/example/bookmanager/
│   ├── BookManagerApplication.java
│   ├── controller/
│   │   └── BookController.java
│   ├── service/
│   │   ├── BookService.java
│   │   └── impl/
│   │       └── BookServiceImpl.java
│   ├── repository/
│   │   └── BookRepository.java
│   ├── entity/
│   │   └── Book.java
│   ├── dto/
│   │   ├── BookDTO.java
│   │   ├── BookRequest.java
│   │   └── ApiResponse.java
│   ├── exception/
│   │   ├── ResourceNotFoundException.java
│   │   ├── GlobalExceptionHandler.java
│   │   └── ErrorDetail.java
│   └── config/
│       └── SwaggerConfig.java
├── src/main/resources/
│   └── application.yml
├── src/test/java/com/example/bookmanager/
│   ├── controller/
│   │   └── BookControllerTest.java
│   └── service/
│       └── BookServiceTest.java
├── postman/
│   └── BookManager.postman_collection.json
└── BOOK_API.md
```

对，你没看错——**Controller、Service、Repository、Entity、DTO、异常处理、单元测试、Swagger 文档、Postman Collection，一次性全部生成。**

这不是科幻，这是 Cursor Composer + Claude 的日常操作。

今天这篇文章，我就带你**全程录屏式复盘**，每一步的 Prompt 怎么写、Cursor 吐出什么、最终怎么跑起来，全部公开。

读完这篇文章，你将彻底告别“从零搭项目”的痛苦。

---

## 二、Step 1：一句话生成项目骨架

### 2.1 打开 Cursor，选中一个空目录

首先确保你已经安装了 Cursor IDE。打开之后，菜单栏选 File → Open Folder，随便找一个空目录，比如 `~/projects/book-manager`。

你看到的应该是一个完全空白的 IDE 界面——左侧文件树空空荡荡，右侧编辑器一片空白。这就是我们的起点。

然后按下 `Cmd+I`（Windows 上按 `Ctrl+I`），打开 **Composer** 模式。注意，这里不是那个小而美的 `Ctrl+K` 行内编辑，而是具备全项目上下文感知能力的 Composer 多文件生成模式。Composer 的输入框会在左侧面板弹出来，等待你输入指令。

### 2.2 Prompt

在 Composer 输入框里，像跟产品经理确认需求一样，清晰地写下你的要求：

```
创建一个 Spring Boot 3.2 + Java 17 的 Maven 项目骨架，包含以下内容：

1. pom.xml：
   - groupId: com.example，artifactId: book-manager
   - Spring Boot 3.2.5
   - 依赖：spring-boot-starter-web, spring-boot-starter-data-jpa, spring-boot-starter-validation
   - MySQL 驱动：mysql-connector-j（runtime）
   - Lombok, springdoc-openapi-starter-webmvc-ui 2.3.0
   - 测试依赖：spring-boot-starter-test (scope test), H2 内存数据库 (scope test)
   - 插件：spring-boot-maven-plugin

2. 启动类：com.example.bookmanager.BookManagerApplication

3. application.yml：
   - MySQL 数据源配置（localhost:3306/book_manager）
   - JPA 配置：ddl-auto=update, show-sql=true, 方言 MySQL8Dialect
   - HikariCP 连接池配置
   - server.port=8080
```

### 2.3 Cursor 输出结果

Cursor Composer 会分析你的 Prompt，然后**一次性生成 3 个文件**。点击 "Accept All" 全部接受。

生成的 `pom.xml` 核心依赖部分：

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.2.5</version>
    <relativePath/>
</parent>

<groupId>com.example</groupId>
<artifactId>book-manager</artifactId>
<version>0.0.1-SNAPSHOT</version>
<name>book-manager</name>
<description>Book Manager RESTful API</description>

<properties>
    <java.version>17</java.version>
</properties>

<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-validation</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>

    <dependency>
        <groupId>com.mysql</groupId>
        <artifactId>mysql-connector-j</artifactId>
        <scope>runtime</scope>
    </dependency>
    <dependency>
        <groupId>com.h2database</groupId>
        <artifactId>h2</artifactId>
        <scope>test</scope>
    </dependency>

    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <optional>true</optional>
    </dependency>

    <dependency>
        <groupId>org.springdoc</groupId>
        <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
        <version>2.3.0</version>
    </dependency>
</dependencies>
```

`BookManagerApplication.java`：

```java
package com.example.bookmanager;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class BookManagerApplication {
    public static void main(String[] args) {
        SpringApplication.run(BookManagerApplication.class, args);
    }
}
```

`application.yml`：

```yaml
server:
  port: 8080

spring:
  datasource:
    url: jdbc:mysql://localhost:3306/book_manager?useSSL=false&serverTimezone=UTC&allowPublicKeyRetrieval=true
    username: root
    password: root
    driver-class-name: com.mysql.cj.jdbc.Driver
    hikari:
      maximum-pool-size: 10
      minimum-idle: 5
      idle-timeout: 300000
      connection-timeout: 20000
      max-lifetime: 1200000

  jpa:
    hibernate:
      ddl-auto: update
    show-sql: true
    properties:
      hibernate:
        dialect: org.hibernate.dialect.MySQL8Dialect
        format_sql: true

springdoc:
  swagger-ui:
    path: /swagger-ui.html
  api-docs:
    path: /v3/api-docs
```

到这里，项目骨架已经就绪。**耗时约 40 秒。**

---

## 三、Step 2：生成 Entity 实体类

项目骨架有了，但它还只是一个空壳。我们需要定义业务核心——**实体类**。在传统开发中，一个 JPA 实体类要写上 `@Entity`、`@Table`、`@Id`、`@GeneratedValue`、`@Column`、各种校验注解，再加上 Lombok 注解和审计字段的处理方法，少说也要二三十行注解，一不留神还容易写错。

但在 Cursor 里，你只需要把你的字段需求说清楚就行。

### 3.1 Prompt

在 Composer 中继续输入：
- 表名：books
- 字段：
  - id: Long, 主键, 自增策略 IDENTITY
  - title: String(200), 非空, 书籍名称
  - author: String(100), 非空
  - isbn: String(20), 唯一, 非空
  - price: BigDecimal, 精度(10,2)
  - publishDate: LocalDate
  - description: String(1000), @Column columnDefinition="TEXT"
  - status: 枚举(BookStatus: AVAILABLE, BORROWED, ARCHIVED), 默认AVAILABLE
  - createdAt: LocalDateTime, 自动填充创建时间
  - updatedAt: LocalDateTime, 自动填充更新时间

- 使用 Lombok: @Data, @NoArgsConstructor, @AllArgsConstructor, @Builder
- 使用 JPA 注解: @Entity, @Table
- 使用 @PrePersist / @PreUpdate 自动设置时间
- 加上必要的校验注解
```

### 3.2 Cursor 输出

```java
package com.example.bookmanager.entity;

import jakarta.persistence.*;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.Size;
import lombok.*;
import java.math.BigDecimal;
import java.time.LocalDate;
import java.time.LocalDateTime;

@Entity
@Table(name = "books")
@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class Book {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @NotBlank(message = "书名不能为空")
    @Size(max = 200, message = "书名不能超过200个字符")
    @Column(nullable = false, length = 200)
    private String title;

    @NotBlank(message = "作者不能为空")
    @Size(max = 100, message = "作者名不能超过100个字符")
    @Column(nullable = false, length = 100)
    private String author;

    @NotBlank(message = "ISBN不能为空")
    @Size(max = 20, message = "ISBN不能超过20个字符")
    @Column(nullable = false, unique = true, length = 20)
    private String isbn;

    @Column(precision = 10, scale = 2)
    private BigDecimal price;

    private LocalDate publishDate;

    @Size(max = 1000, message = "描述不能超过1000个字符")
    @Column(columnDefinition = "TEXT")
    private String description;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 20)
    @Builder.Default
    private BookStatus status = BookStatus.AVAILABLE;

    @Column(updatable = false)
    private LocalDateTime createdAt;

    private LocalDateTime updatedAt;

    @PrePersist
    protected void onCreate() {
        this.createdAt = LocalDateTime.now();
        this.updatedAt = LocalDateTime.now();
    }

    @PreUpdate
    protected void onUpdate() {
        this.updatedAt = LocalDateTime.now();
    }

    public enum BookStatus {
        AVAILABLE, BORROWED, ARCHIVED
    }
}
```

一气呵成——Lombok、JPA 注解、校验注解、枚举类型、`@PrePersist/@PreUpdate` 全部到位。**耗时 20 秒。**

---

## 四、Step 3：生成 Repository 层

### 4.1 Prompt

```
在 com.example.bookmanager.repository 包下创建 BookRepository 接口：

- 继承 JpaRepository<Book, Long>
- 使用 JpaSpecificationExecutor<Book> 支持动态查询
- 自定义查询方法：
  1. 根据书名模糊查询 + 分页：findByTitleContaining(String title, Pageable pageable)
  2. 根据作者模糊查询：findByAuthorContaining(String author, Pageable pageable)
  3. 根据状态查询：findByStatus(BookStatus status, Pageable pageable)
  4. 根据 ISBN 查询：Optional<Book> findByIsbn(String isbn)
  5. 检查 ISBN 是否存在：boolean existsByIsbn(String isbn)
  6. 根据状态统计数量：long countByStatus(BookStatus status)
```

### 4.2 Cursor 输出

```java
package com.example.bookmanager.repository;

import com.example.bookmanager.entity.Book;
import com.example.bookmanager.entity.Book.BookStatus;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.JpaSpecificationExecutor;
import org.springframework.stereotype.Repository;
import java.util.Optional;

@Repository
public interface BookRepository extends JpaRepository<Book, Long>,
        JpaSpecificationExecutor<Book> {

    Page<Book> findByTitleContaining(String title, Pageable pageable);

    Page<Book> findByAuthorContaining(String author, Pageable pageable);

    Page<Book> findByStatus(BookStatus status, Pageable pageable);

    Optional<Book> findByIsbn(String isbn);

    boolean existsByIsbn(String isbn);

    long countByStatus(BookStatus status);
}
```

扩展了 `JpaSpecificationExecutor` 为后续动态查询留好了余地（比如前端传多个过滤条件做组合查询），自定义方法命名完全符合 Spring Data JPA 的方法命名规范，你直接就能用，不需要任何修改。**耗时 15 秒。**

---

## 五、Step 4：生成 Service 层（CRUD + 分页 + 异常处理）

Service 层是业务逻辑的核心。一个成熟的 CRUD Service 需要处理的问题远不止“增删改查”四个字那么简单——分页排序的灵活性怎么保证？更新操作如何做到部分字段更新？删除前要不要做状态校验？ISBN 重复怎么办？资源找不到怎么办？

这些细节，正是拉开“demo 代码”和“生产级代码”差距的地方。而 Cursor 的优势就在于——**你只要把你的业务规则说清楚，它就能把这些规则变成健壮的代码。**

### 5.1 Prompt
BookService 接口方法：
- Page<Book> getAllBooks(int page, int size, String sortBy, String sortDir)
- Page<Book> searchBooks(String keyword, String searchType, int page, int size)
- Book getBookById(Long id)
- Book createBook(Book book)
- Book updateBook(Long id, Book bookDetails)
- void deleteBook(Long id)
- Book patchBookStatus(Long id, BookStatus status)
- Map<String, Object> getBookStats()

实现类要求：
- 使用 @Service 注解
- 分页查询支持按字段动态排序（sortBy=suffix对应实体字段）
- 按标题/作者搜索（根据searchType选择）
- 创建时校验 ISBN 唯一性，重复抛出 DuplicateIsbnException
- 更新/删除/查询时，ID 不存在抛出 ResourceNotFoundException
- updateBook 使用 BeanUtils.copyProperties 忽略 null 属性和 id/createdAt
- 删除前检查状态是否为 BORROWED，已借出的书不能删除，抛出 BusinessException
- 统计信息返回：总数、可用数、已借出数、已归档数
- 加上 @Slf4j 日志
```

### 5.2 Cursor 输出

Service 接口：

```java
package com.example.bookmanager.service;

import com.example.bookmanager.entity.Book;
import com.example.bookmanager.entity.Book.BookStatus;
import org.springframework.data.domain.Page;
import java.util.Map;

public interface BookService {
    Page<Book> getAllBooks(int page, int size, String sortBy, String sortDir);
    Page<Book> searchBooks(String keyword, String searchType, int page, int size);
    Book getBookById(Long id);
    Book createBook(Book book);
    Book updateBook(Long id, Book bookDetails);
    void deleteBook(Long id);
    Book patchBookStatus(Long id, BookStatus status);
    Map<String, Object> getBookStats();
}
```

实现类核心片段：

```java
package com.example.bookmanager.service.impl;

import com.example.bookmanager.entity.Book;
import com.example.bookmanager.entity.Book.BookStatus;
import com.example.bookmanager.exception.BusinessException;
import com.example.bookmanager.exception.DuplicateIsbnException;
import com.example.bookmanager.exception.ResourceNotFoundException;
import com.example.bookmanager.repository.BookRepository;
import com.example.bookmanager.service.BookService;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.BeanUtils;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.PageRequest;
import org.springframework.data.domain.Pageable;
import org.springframework.data.domain.Sort;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import java.beans.PropertyDescriptor;
import java.util.*;
import java.util.stream.Collectors;

@Slf4j
@Service
@RequiredArgsConstructor
@Transactional
public class BookServiceImpl implements BookService {

    private final BookRepository bookRepository;

    private static final Set<String> VALID_SORT_FIELDS = Set.of("id", "title", "author", "isbn", "price", "publishDate", "createdAt", "updatedAt");

    /**
     * 校验并生成排序字段
     */
    private String resolveSortField(String sortBy) {
        if (sortBy == null || sortBy.isBlank()) {
            return "id";
        }
        return VALID_SORT_FIELDS.contains(sortBy) ? sortBy : "id";
    }

    @Override
    @Transactional(readOnly = true)
    public Page<Book> getAllBooks(int page, int size, String sortBy, String sortDir) {
        Sort sort = "desc".equalsIgnoreCase(sortDir)
                ? Sort.by(resolveSortField(sortBy)).descending()
                : Sort.by(resolveSortField(sortBy)).ascending();
        Pageable pageable = PageRequest.of(page, size, sort);
        return bookRepository.findAll(pageable);
    }

    @Override
    @Transactional(readOnly = true)
    public Page<Book> searchBooks(String keyword, String searchType, int page, int size) {
        Pageable pageable = PageRequest.of(page, size, Sort.by("id").descending());
        if ("author".equalsIgnoreCase(searchType)) {
            return bookRepository.findByAuthorContaining(keyword, pageable);
        }
        return bookRepository.findByTitleContaining(keyword, pageable);
    }

    @Override
    public Book createBook(Book book) {
        if (bookRepository.existsByIsbn(book.getIsbn())) {
            throw new DuplicateIsbnException("ISBN 已存在: " + book.getIsbn());
        }
        book.setStatus(BookStatus.AVAILABLE);
        Book saved = bookRepository.save(book);
        log.info("创建图书成功, id={}, title={}", saved.getId(), saved.getTitle());
        return saved;
    }

    @Override
    public Book updateBook(Long id, Book bookDetails) {
        Book existing = bookRepository.findById(id)
                .orElseThrow(() -> new ResourceNotFoundException("图书不存在, id=" + id));

        String[] ignoreProperties = getNullPropertyNames(bookDetails);

        // 排除 id、createdAt、isbn 字段
        Set<String> alwaysIgnore = new HashSet<>(Set.of("id", "createdAt", "isbn"));
        alwaysIgnore.addAll(Arrays.asList(ignoreProperties));

        BeanUtils.copyProperties(bookDetails, existing, alwaysIgnore.toArray(new String[0]));
        Book updated = bookRepository.save(existing);
        log.info("更新图书成功, id={}", updated.getId());
        return updated;
    }

    @Override
    public void deleteBook(Long id) {
        Book book = bookRepository.findById(id)
                .orElseThrow(() -> new ResourceNotFoundException("图书不存在, id=" + id));
        if (book.getStatus() == BookStatus.BORROWED) {
            throw new BusinessException("已借出的图书不能删除, id=" + id);
        }
        bookRepository.delete(book);
        log.info("删除图书成功, id={}, title={}", id, book.getTitle());
    }

    /**
     * 获取对象中值为 null 的属性名数组，更新时忽略这些字段
     */
    private String[] getNullPropertyNames(Object source) {
        return Arrays.stream(BeanUtils.getPropertyDescriptors(source.getClass()))
                .map(PropertyDescriptor::getName)
                .filter(name -> {
                    try {
                        return BeanUtils.getPropertyDescriptor(source.getClass(), name)
                                .getReadMethod().invoke(source) == null;
                    } catch (Exception e) {
                        return false;
                    }
                })
                .toArray(String[]::new);
    }
}
```

> **设计亮点**：`getNullPropertyNames` 方法自动过滤 null 属性，实现部分更新（PATCH 语义），这在实际项目中非常实用。

**耗时约 50 秒**，Service 层 + 接口 + 完整实现一次性搞定。

---

## 六、Step 5：生成 Controller 层 + 统一返回体

Controller 层是面向前端和调用方的“门面”。一个好的 API 设计需要做到：统一返回格式、规范 HTTP 状态码、友好的 Swagger 注解、清晰的参数说明。如果手写，光是每个接口上贴 `@Operation`、`@ApiResponses`、`@Parameter` 这些注解就要耗费大量精力。

Cursor 的做法是：你定义好 DTO 结构和接口列表，它来帮你把注解、校验、响应包装一次性搞定。

### 6.1 Prompt

先创建 DTO 层：1. ApiResponse<T>：统一返回体
   - code: int, message: String, data: T, timestamp: LocalDateTime
   - 静态工厂方法：success(T data), error(int code, String message)
   - 加上 @Builder 注解

2. BookRequest DTO：
   - 与 Book 字段基本一致，但不包含 id/createdAt/updatedAt
   - 使用 @Valid 校验注解

在 com.example.bookmanager.controller 包下创建 BookController：
- @RestController + @RequestMapping("/api/v1/books")
- 所有接口返回 ApiResponse<T> 包装
- 接口列表：
  1. GET /api/v1/books?page=0&size=10&sortBy=id&sortDir=desc
     → 分页查询所有图书
  2. GET /api/v1/books/search?keyword=xxx&searchType=title&page=0&size=10
     → 按条件搜索
  3. GET /api/v1/books/{id} → 查询单本图书
  4. POST /api/v1/books → 创建图书(@Valid @RequestBody BookRequest)
  5. PUT /api/v1/books/{id} → 更新图书(@Valid @RequestBody BookRequest)
  6. PATCH /api/v1/books/{id}/status → 修改图书状态(@RequestBody Map<String,String>)
  7. DELETE /api/v1/books/{id} → 删除图书
  8. GET /api/v1/books/stats → 获取统计信息
- 每个方法上加 Swagger 注解：@Operation(summary=...), @ApiResponses, @Parameter
- 所有 default value 都明确指定
```

### 6.2 Cursor 输出

`ApiResponse.java`：

```java
package com.example.bookmanager.dto;

import com.fasterxml.jackson.annotation.JsonInclude;
import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;
import java.time.LocalDateTime;

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
@JsonInclude(JsonInclude.Include.NON_NULL)
public class ApiResponse<T> {

    private int code;
    private String message;
    private T data;
    private LocalDateTime timestamp;

    public static <T> ApiResponse<T> success(T data) {
        return ApiResponse.<T>builder()
                .code(200)
                .message("success")
                .data(data)
                .timestamp(LocalDateTime.now())
                .build();
    }

    public static <T> ApiResponse<T> error(int code, String message) {
        return ApiResponse.<T>builder()
                .code(code)
                .message(message)
                .timestamp(LocalDateTime.now())
                .build();
    }
}
```

`BookController.java` 核心片段：

```java
package com.example.bookmanager.controller;

import com.example.bookmanager.dto.ApiResponse;
import com.example.bookmanager.dto.BookRequest;
import com.example.bookmanager.entity.Book;
import com.example.bookmanager.service.BookService;
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.Parameter;
import io.swagger.v3.oas.annotations.responses.ApiResponses;
import io.swagger.v3.oas.annotations.tags.Tag;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import org.springframework.data.domain.Page;
import org.springframework.http.HttpStatus;
import org.springframework.web.bind.annotation.*;
import java.util.Map;

@RestController
@RequestMapping("/api/v1/books")
@RequiredArgsConstructor
@Tag(name = "图书管理", description = "图书增删改查接口")
public class BookController {

    private final BookService bookService;

    @GetMapping
    @Operation(summary = "分页查询图书列表")
    public ApiResponse<Page<Book>> getAllBooks(
            @Parameter(description = "页码") @RequestParam(defaultValue = "0") int page,
            @Parameter(description = "每页大小") @RequestParam(defaultValue = "10") int size,
            @Parameter(description = "排序字段") @RequestParam(defaultValue = "id") String sortBy,
            @Parameter(description = "排序方向") @RequestParam(defaultValue = "desc") String sortDir) {
        return ApiResponse.success(bookService.getAllBooks(page, size, sortBy, sortDir));
    }

    @GetMapping("/search")
    @Operation(summary = "搜索图书")
    public ApiResponse<Page<Book>> searchBooks(
            @Parameter(description = "搜索关键词") @RequestParam String keyword,
            @Parameter(description = "搜索类型(title/author)") @RequestParam(defaultValue = "title") String searchType,
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "10") int size) {
        return ApiResponse.success(bookService.searchBooks(keyword, searchType, page, size));
    }

    @GetMapping("/{id}")
    @Operation(summary = "查询单本图书")
    public ApiResponse<Book> getBookById(@PathVariable Long id) {
        return ApiResponse.success(bookService.getBookById(id));
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    @Operation(summary = "创建图书")
    public ApiResponse<Book> createBook(@Valid @RequestBody BookRequest request) {
        return ApiResponse.success(bookService.createBook(request.toEntity()));
    }

    @PutMapping("/{id}")
    @Operation(summary = "更新图书")
    public ApiResponse<Book> updateBook(@PathVariable Long id, @Valid @RequestBody BookRequest request) {
        return ApiResponse.success(bookService.updateBook(id, request.toEntity()));
    }

    @DeleteMapping("/{id}")
    @Operation(summary = "删除图书")
    public ApiResponse<Void> deleteBook(@PathVariable Long id) {
        bookService.deleteBook(id);
        return ApiResponse.success(null);
    }

    @GetMapping("/stats")
    @Operation(summary = "获取图书统计信息")
    public ApiResponse<Map<String, Object>> getStats() {
        return ApiResponse.success(bookService.getBookStats());
    }
}
```

**耗时约 40 秒**，Controller 层 + DTO + 统一返回体一气呵成。

---

## 七、Step 6：全局异常处理器 + 完整异常体系

### 7.1 Prompt

```
在 com.example.bookmanager.exception 包下创建：

1. ResourceNotFoundException extends RuntimeException
2. DuplicateIsbnException extends RuntimeException
3. BusinessException extends RuntimeException
4. ErrorDetail 类：timestamp, status, error, message, path
5. GlobalExceptionHandler：
   - @RestControllerAdvice
   - 处理 MethodArgumentNotValidException → 400（收集所有字段校验错误）
   - 处理 ResourceNotFoundException → 404
   - 处理 DuplicateIsbnException → 409
   - 处理 BusinessException → 400
   - 处理 HttpMessageNotReadableException → 400
   - 处理 Exception → 500（通用兜底）
   - 所有响应体使用 ApiResponse<Void> 格式
   - 打印 error 级别日志
```

### 7.2 Cursor 输出

`GlobalExceptionHandler.java`：

```java
package com.example.bookmanager.exception;

import com.example.bookmanager.dto.ApiResponse;
import jakarta.servlet.http.HttpServletRequest;
import lombok.extern.slf4j.Slf4j;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.http.converter.HttpMessageNotReadableException;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;
import java.util.stream.Collectors;

@Slf4j
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ApiResponse<Void>> handleResourceNotFound(
            ResourceNotFoundException ex, HttpServletRequest request) {
        log.warn("资源未找到: {}", ex.getMessage());
        return buildResponse(HttpStatus.NOT_FOUND, ex.getMessage(), request.getRequestURI());
    }

    @ExceptionHandler(DuplicateIsbnException.class)
    public ResponseEntity<ApiResponse<Void>> handleDuplicateIsbn(
            DuplicateIsbnException ex, HttpServletRequest request) {
        log.warn("ISBN重复: {}", ex.getMessage());
        return buildResponse(HttpStatus.CONFLICT, ex.getMessage(), request.getRequestURI());
    }

    @ExceptionHandler(BusinessException.class)
    public ResponseEntity<ApiResponse<Void>> handleBusiness(
            BusinessException ex, HttpServletRequest request) {
        log.warn("业务异常: {}", ex.getMessage());
        return buildResponse(HttpStatus.BAD_REQUEST, ex.getMessage(), request.getRequestURI());
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ApiResponse<Void>> handleValidation(
            MethodArgumentNotValidException ex, HttpServletRequest request) {
        String message = ex.getBindingResult().getFieldErrors().stream()
                .map(e -> e.getField() + ": " + e.getDefaultMessage())
                .collect(Collectors.joining("; "));
        log.warn("参数校验失败: {}", message);
        return buildResponse(HttpStatus.BAD_REQUEST, message, request.getRequestURI());
    }

    @ExceptionHandler(HttpMessageNotReadableException.class)
    public ResponseEntity<ApiResponse<Void>> handleMessageNotReadable(
            HttpMessageNotReadableException ex, HttpServletRequest request) {
        log.warn("请求体不可读: {}", ex.getMessage());
        return buildResponse(HttpStatus.BAD_REQUEST, "请求格式错误", request.getRequestURI());
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ApiResponse<Void>> handleGeneral(
            Exception ex, HttpServletRequest request) {
        log.error("系统异常", ex);
        return buildResponse(HttpStatus.INTERNAL_SERVER_ERROR,
                "服务器内部错误", request.getRequestURI());
    }

    private ResponseEntity<ApiResponse<Void>> buildResponse(HttpStatus status, String message, String path) {
        ApiResponse<Void> response = ApiResponse.error(status.value(), message);
        return new ResponseEntity<>(response, status);
    }
}
```

**耗时约 25 秒**，生产级的异常处理体系——404（资源未找到）、400（参数校验失败/业务异常）、409（ISBN 冲突）、500（兜底异常）完整覆盖。每一个异常都带着清晰的日志级别（warn 给业务异常，error 给系统异常），排查问题时不至于抓瞎。

---

## 八、Step 7：生成单元测试

AI 编程最让我感动的一点，就是**它终于帮我们解决了“写测试”这个老大难问题**。多数开发者在时间压力下会跳过单元测试，不是不想写，而是——正向用例、反向用例、边界条件、Mock 设置、AssertJ 断言链……一个 Service 的完整测试写下来，代码量往往是业务代码的 2-3 倍。

而 Cursor 帮你写测试，不只是把方法名罗列出来。它会自动识别你 Service 中哪些地方可能抛异常、哪些地方有 if-else 分支需要分别覆盖，然后用 JUnit 5 + Mockito 的标准写法给你全部端上来。

### 8.1 Prompt，使用 JUnit 5 + Mockito + @WebMvcTest：

1. BookServiceTest：
   - 使用 @ExtendWith(MockitoExtension.class)
   - Mock BookRepository
   - 测试用例覆盖：
     - getAllBooks 正常分页查询
     - getBookById 存在/不存在（抛异常）
     - createBook ISBN唯一/重复（抛DuplicateIsbnException）
     - updateBook 正常更新/ID不存在
     - deleteBook 正常删除/已借出状态不能删
     - searchBooks 按标题搜索
     - getBookStats 统计信息

2. BookControllerTest：
   - 使用 @WebMvcTest(BookController.class)
   - Mock BookService
   - 使用 MockMvc
   - 测试用例：GET列表、GET单本、GET不存在返回404、POST创建成功201、POST参数校验失败400、DELETE、GET stats
```

### 8.2 Cursor 输出

`BookServiceTest.java` 核心片段：

```java
package com.example.bookmanager.service;

import com.example.bookmanager.entity.Book;
import com.example.bookmanager.entity.Book.BookStatus;
import com.example.bookmanager.exception.BusinessException;
import com.example.bookmanager.exception.DuplicateIsbnException;
import com.example.bookmanager.exception.ResourceNotFoundException;
import com.example.bookmanager.repository.BookRepository;
import com.example.bookmanager.service.impl.BookServiceImpl;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Nested;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;
import org.springframework.data.domain.*;
import java.math.BigDecimal;
import java.time.LocalDate;
import java.util.List;
import java.util.Optional;
import static org.assertj.core.api.Assertions.*;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
@DisplayName("BookService 单元测试")
class BookServiceTest {

    @Mock
    private BookRepository bookRepository;

    @InjectMocks
    private BookServiceImpl bookService;

    private Book sampleBook;

    @BeforeEach
    void setUp() {
        sampleBook = Book.builder()
                .id(1L)
                .title("Java编程思想")
                .author("Bruce Eckel")
                .isbn("978-7111243823")
                .price(new BigDecimal("108.00"))
                .publishDate(LocalDate.of(2007, 6, 1))
                .status(BookStatus.AVAILABLE)
                .build();
    }

    @Nested
    @DisplayName("查询测试")
    class QueryTests {

        @Test
        @DisplayName("分页查询所有图书")
        void shouldGetAllBooksWithPagination() {
            Pageable pageable = PageRequest.of(0, 10, Sort.by("id").descending());
            Page<Book> page = new PageImpl<>(List.of(sampleBook), pageable, 1);
            when(bookRepository.findAll(any(Pageable.class))).thenReturn(page);

            Page<Book> result = bookService.getAllBooks(0, 10, "id", "desc");
            assertThat(result.getTotalElements()).isEqualTo(1);
            assertThat(result.getContent().get(0).getTitle()).isEqualTo("Java编程思想");
        }

        @Test
        @DisplayName("根据ID查询图书-存在")
        void shouldGetBookByIdWhenExists() {
            when(bookRepository.findById(1L)).thenReturn(Optional.of(sampleBook));
            Book result = bookService.getBookById(1L);
            assertThat(result.getTitle()).isEqualTo("Java编程思想");
        }

        @Test
        @DisplayName("根据ID查询图书-不存在时抛异常")
        void shouldThrowWhenBookNotFound() {
            when(bookRepository.findById(99L)).thenReturn(Optional.empty());
            assertThatThrownBy(() -> bookService.getBookById(99L))
                    .isInstanceOf(ResourceNotFoundException.class)
                    .hasMessageContaining("99");
        }
    }

    @Nested
    @DisplayName("创建测试")
    class CreateTests {

        @Test
        @DisplayName("创建图书成功")
        void shouldCreateBookSuccessfully() {
            when(bookRepository.existsByIsbn(anyString())).thenReturn(false);
            when(bookRepository.save(any(Book.class))).thenReturn(sampleBook);

            Book result = bookService.createBook(sampleBook);
            assertThat(result).isNotNull();
            assertThat(result.getStatus()).isEqualTo(BookStatus.AVAILABLE);
        }

        @Test
        @DisplayName("创建时ISBN重复抛异常")
        void shouldThrowWhenIsbnDuplicated() {
            when(bookRepository.existsByIsbn("978-7111243823")).thenReturn(true);
            assertThatThrownBy(() -> bookService.createBook(sampleBook))
                    .isInstanceOf(DuplicateIsbnException.class);
        }
    }

    @Nested
    @DisplayName("删除测试")
    class DeleteTests {

        @Test
        @DisplayName("删除已借出图书应抛异常")
        void shouldNotDeleteBorrowedBook() {
            sampleBook.setStatus(BookStatus.BORROWED);
            when(bookRepository.findById(1L)).thenReturn(Optional.of(sampleBook));
            assertThatThrownBy(() -> bookService.deleteBook(1L))
                    .isInstanceOf(BusinessException.class)
                    .hasMessageContaining("已借出");
        }
    }
}
```

`BookControllerTest.java` 核心片段：

```java
@WebMvcTest(BookController.class)
@DisplayName("BookController 接口测试")
class BookControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockitoBean
    private BookService bookService;

    @Test
    @DisplayName("GET /api/v1/books - 分页查询")
    void shouldGetBooksPage() throws Exception {
        Page<Book> emptyPage = new PageImpl<>(List.of());
        when(bookService.getAllBooks(0, 10, "id", "desc")).thenReturn(emptyPage);

        mockMvc.perform(get("/api/v1/books")
                        .param("page", "0")
                        .param("size", "10")
                        .param("sortBy", "id")
                        .param("sortDir", "desc"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.code").value(200))
                .andExpect(jsonPath("$.message").value("success"));
    }

    @Test
    @DisplayName("POST /api/v1/books - 创建图书成功返回201")
    void shouldCreateBook() throws Exception {
        String json = """
                {
                    "title": "Java编程思想",
                    "author": "Bruce Eckel",
                    "isbn": "978-7111243823",
                    "price": 108.00
                }
                """;

        when(bookService.createBook(any())).thenReturn(Book.builder().id(1L).build());

        mockMvc.perform(post("/api/v1/books")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(json))
                .andExpect(status().isCreated())
                .andExpect(jsonPath("$.code").value(200));
    }

    @Test
    @DisplayName("POST /api/v1/books - title为空返回400")
    void shouldReturn400WhenTitleBlank() throws Exception {
        String json = """
                {
                    "title": "",
                    "author": "Bruce Eckel",
                    "isbn": "978-7111243823"
                }
                """;

        mockMvc.perform(post("/api/v1/books")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(json))
                .andExpect(status().isBadRequest())
                .andExpect(jsonPath("$.code").value(400))
                .andExpect(jsonPath("$.message").value(containsString("书名")));
    }

    @Test
    @DisplayName("GET /api/v1/books/999 - 图书不存在返回404")
    void shouldReturn404WhenBookNotFound() throws Exception {
        when(bookService.getBookById(999L))
                .thenThrow(new ResourceNotFoundException("图书不存在, id=999"));

        mockMvc.perform(get("/api/v1/books/999"))
                .andExpect(status().isNotFound())
                .andExpect(jsonPath("$.code").value(404));
    }
}
```

**耗时约 60 秒**，Service 测试 + Controller 测试全部覆盖——正向场景（正常 CRUD）、反向场景（参数校验失败、ID 不存在）、边界场景（ISBN 重复、已借出不能删除）。Mockito 的 `when/thenReturn` 和 `assertThatThrownBy` 写得有板有眼，AssertJ 的流式断言也比传统的 `assertEquals` 可读性高得多。

有一点特别值得提：Controller 测试用的是 `@WebMvcTest` 而不是 `@SpringBootTest`，这意味着只加载 Controller 层，启动飞快——这种最佳实践，AI 自动就给你安排上了。

---

## 九、Step 8：生成 Swagger 文档 + Postman Collection

接口写完了，但怎么让别人知道你的 API 怎么用？传统做法是写接口文档（Word/语雀/飞书），然后测试同学再手动在 Postman 里一个一个建请求——一个 8 个接口的系统，光建 Postman 请求就要十来分钟，还容易遗漏参数。

Cursor 的最后一招：自动生成 Swagger 配置类 **和** Postman Collection JSON 文件。前者让你在浏览器里可视化测试 API，后者让你一键导入 Postman 就能跑完所有用例。真正做到“前后端联调的文档零成本”。

### 9.1 Prompt   - @Configuration
   - 配置 OpenAPI Bean，标题"图书管理系统 API"，版本 v1.0

2. 在项目根目录 postman/ 目录下生成 BookManager.postman_collection.json：
   - 包含所有7个接口的 Postman 请求定义
   - 变量 baseUrl = http://localhost:8080
   - 每个请求包含示例请求体和预期响应结构
```

### 9.2 Cursor 输出

`SwaggerConfig.java`：

```java
package com.example.bookmanager.config;

import io.swagger.v3.oas.models.OpenAPI;
import io.swagger.v3.oas.models.info.Contact;
import io.swagger.v3.oas.models.info.Info;
import io.swagger.v3.oas.models.info.License;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class SwaggerConfig {

    @Bean
    public OpenAPI bookManagerOpenAPI() {
        return new OpenAPI()
                .info(new Info()
                        .title("图书管理系统 API")
                        .description("基于 Spring Boot 3.2 的 RESTful 图书管理服务")
                        .version("v1.0.0")
                        .contact(new Contact()
                                .name("Developer")
                                .email("dev@example.com"))
                        .license(new License()
                                .name("Apache 2.0")
                                .url("https://www.apache.org/licenses/LICENSE-2.0")));
    }
}
```

Postman Collection JSON 结构（`postman/BookManager.postman_collection.json`）：

```json
{
  "info": {
    "name": "Book Manager API",
    "schema": "https://schema.getpostman.com/json/collection/v2.1.0/collection.json"
  },
  "variable": [
    {
      "key": "baseUrl",
      "value": "http://localhost:8080"
    }
  ],
  "item": [
    {
      "name": "获取图书统计",
      "request": {
        "method": "GET",
        "url": {
          "raw": "{{baseUrl}}/api/v1/books/stats",
          "host": ["{{baseUrl}}"],
          "path": ["api", "v1", "books", "stats"]
        }
      }
    },
    {
      "name": "分页查询",
      "request": {
        "method": "GET",
        "url": {
          "raw": "{{baseUrl}}/api/v1/books?page=0&size=10&sortBy=id&sortDir=desc",
          "host": ["{{baseUrl}}"],
          "path": ["api", "v1", "books"],
          "query": [
            { "key": "page", "value": "0" },
            { "key": "size", "value": "10" },
            { "key": "sortBy", "value": "id" },
            { "key": "sortDir", "value": "desc" }
          ]
        }
      }
    },
    {
      "name": "搜索图书",
      "request": {
        "method": "GET",
        "url": {
          "raw": "{{baseUrl}}/api/v1/books/search?keyword=Java&searchType=title&page=0&size=10",
          "host": ["{{baseUrl}}"],
          "path": ["api", "v1", "books", "search"],
          "query": [
            { "key": "keyword", "value": "Java" },
            { "key": "searchType", "value": "title" },
            { "key": "page", "value": "0" },
            { "key": "size", "value": "10" }
          ]
        }
      }
    },
    {
      "name": "查询单本图书",
      "request": {
        "method": "GET",
        "url": {
          "raw": "{{baseUrl}}/api/v1/books/1",
          "host": ["{{baseUrl}}"],
          "path": ["api", "v1", "books", "1"]
        }
      }
    },
    {
      "name": "创建图书",
      "request": {
        "method": "POST",
        "header": [
          { "key": "Content-Type", "value": "application/json" }
        ],
        "url": {
          "raw": "{{baseUrl}}/api/v1/books",
          "host": ["{{baseUrl}}"],
          "path": ["api", "v1", "books"]
        },
        "body": {
          "mode": "raw",
          "raw": "{\n  \"title\": \"Java编程思想\",\n  \"author\": \"Bruce Eckel\",\n  \"isbn\": \"978-7111243823\",\n  \"price\": 108.00,\n  \"publishDate\": \"2007-06-01\",\n  \"description\": \"Java经典入门书籍\"\n}"
        }
      }
    },
    {
      "name": "更新图书",
      "request": {
        "method": "PUT",
        "header": [
          { "key": "Content-Type", "value": "application/json" }
        ],
        "url": {
          "raw": "{{baseUrl}}/api/v1/books/1",
          "host": ["{{baseUrl}}"],
          "path": ["api", "v1", "books", "1"]
        },
        "body": {
          "mode": "raw",
          "raw": "{\n  \"title\": \"Java编程思想(第4版)\",\n  \"author\": \"Bruce Eckel\",\n  \"isbn\": \"978-7111243823\",\n  \"price\": 128.00\n}"
        }
      }
    },
    {
      "name": "删除图书",
      "request": {
        "method": "DELETE",
        "url": {
          "raw": "{{baseUrl}}/api/v1/books/1",
          "host": ["{{baseUrl}}"],
          "path": ["api", "v1", "books", "1"]
        }
      }
    }
  ]
}
```

**耗时约 20 秒**，Swagger 配置 + Postman Collection 完成。导入 Postman 即可一键测试全部接口。

---

## 十、最终项目结构总览

全部 8 个 Step 完成后，你的项目结构是这样的：

```
book-manager/
├── pom.xml                                          # Maven 配置
├── postman/
│   └── BookManager.postman_collection.json          # Postman 测试集
├── src/
│   ├── main/
│   │   ├── java/com/example/bookmanager/
│   │   │   ├── BookManagerApplication.java          # 启动类
│   │   │   ├── config/
│   │   │   │   └── SwaggerConfig.java               # Swagger 配置
│   │   │   ├── controller/
│   │   │   │   └── BookController.java              # RESTful 控制器
│   │   │   ├── dto/
│   │   │   │   ├── ApiResponse.java                 # 统一返回体
│   │   │   │   └── BookRequest.java                 # 请求 DTO
│   │   │   ├── entity/
│   │   │   │   └── Book.java                        # JPA 实体
│   │   │   ├── exception/
│   │   │   │   ├── BusinessException.java           # 业务异常
│   │   │   │   ├── DuplicateIsbnException.java      # ISBN 重复异常
│   │   │   │   ├── ErrorDetail.java                 # 错误详情
│   │   │   │   ├── GlobalExceptionHandler.java      # 全局异常处理
│   │   │   │   └── ResourceNotFoundException.java   # 资源未找到异常
│   │   │   ├── repository/
│   │   │   │   └── BookRepository.java              # JPA Repository
│   │   │   └── service/
│   │   │       ├── BookService.java                 # 服务接口
│   │   │       └── impl/
│   │   │           └── BookServiceImpl.java         # 服务实现
│   │   └── resources/
│   │       └── application.yml                      # 应用配置
│   └── test/java/com/example/bookmanager/
│       ├── controller/
│       │   └── BookControllerTest.java              # Controller 测试
│       └── service/
│           └── BookServiceTest.java                 # Service 测试
```

**16 个源文件，3 分钟内全部生成。**

---

## 十一、跑起来只需要 3 条命令

代码生成完了，接下来就是见证奇迹的时刻——把它跑起来。

```bash
# 1. 启动 MySQL（Docker 一键搞定，不需要手动装 MySQL）
docker run -d --name mysql-book \
  -e MYSQL_ROOT_PASSWORD=root \
  -e MYSQL_DATABASE=book_manager \
  -p 3306:3306 mysql:8

# 2. 编译并启动 Spring Boot
mvn spring-boot:run

# 3. 打开 Swagger UI，可视化测试所有接口
open http://localhost:8080/swagger-ui.html
```

启动成功后，你会在控制台看到熟悉的 Spring Boot Banner，随后 JPA 会根据 Entity 定义自动建表：

```
Hibernate: create table books (id bigint not null auto_increment, ...)
```

然后访问 `http://localhost:8080/swagger-ui.html`，你会在浏览器里看到一个交互式的 API 文档页面。每一个接口都可以直接在页面上点 "Try it out" → 填参数 → "Execute"，真实地调用你的后台服务。不需要 Postman，不需要 cURL，Swagger UI 就是你的第一个客户端。

当然，如果你想做更系统的测试，把前面生成的 `postman/BookManager.postman_collection.json` 文件导入 Postman，所有 7 个接口已经预设好了请求参数和示例请求体，点 "Run Collection" 就能一键跑通。

测试流程建议：

```
POST 创建一本 → GET 列表确认 → GET 单本确认
→ PUT 修改书名 → GET 确认修改生效
→ DELETE 删除 → GET 确认已删除（应返回 404）
→ POST 创建三本 → GET /stats 查看统计
```

这套流程在 Postman 里全程只需点击鼠标，不需要写任何代码。**从 git clone 到完整验证 API，总耗时不超过 10 分钟。**

---

## 十二、总结：AI 编程的质变时刻

回看整个过程，核心价值不在于“快”，而在于**把你的工作重心从“搬砖”变成了“设计”**。

让我们用一张表格来直观感受一下：

| 维度 | 传统方式 | Cursor Composer |
|------|---------|----------------|
| 项目骨架搭建 | 手敲 pom.xml + 启动类 + 配置，20 分钟 | 一句话，40 秒 |
| Entity + Repository | 手写注解 + 方法命名，30 分钟 | 两句话，35 秒 |
| Service 层 | CRUD + 异常处理 + 分页排序，90 分钟 | 一句话，50 秒 |
| Controller + DTO | 接口定义 + 统一返回 + 校验，60 分钟 | 一句话，40 秒 |
| 全局异常处理 | 逐个写 Handler，40 分钟 | 一句话，25 秒 |
| 单元测试 | Mock + 用例编写，120 分钟 | 一句话，60 秒 |
| Swagger + Postman | 手动配置 + 导出，30 分钟 | 一句话，20 秒 |
| **总计** | **约 390 分钟（6.5 小时）** | **约 270 秒（4.5 分钟）** |

**效率提升约 85 倍。**

而且这个对比还是保守估计——因为 6.5 小时是“理想状态下不打断地写”，而现实中你会被各种事情打断、要去 StackOverflow 查 API、要调试各种拼写错误。而 Cursor 生成后，你只需要做三件事：

1. **检查 `application.yml`**——MySQL 连接信息是否与你的环境一致
2. **确认 DTO 转换**——`BookRequest.toEntity()` 方法如果需要，手动补一行代码
3. **微调边界条件**——某些特殊业务逻辑可能需要稍作调整

核心区别在于：**你从“从零创造”变成了“从零审查”**——认知负荷从需要时刻在脑中构建完整代码结构，切换到了拿着现成代码做 Code Review。前者拼的是记忆力和打字速度，后者拼的是架构感和判断力。这，才是 AI 编程真正的质变。

### 写好 Prompt 的 3 个诀窍

复盘全程，你会发现 Prompt 的质量直接决定了代码的质量。三个经验分享给你：

1. **给上下文，不要只给需求**——告诉它“Spring Boot 3.2 + Java 17 + JPA + MySQL”，比只说“写个 CRUD”效果好 10 倍。
2. **给约束，不要只给目标**——说清楚“异常怎么处理、DTO 怎么转换、测试要覆盖哪些场景”，这些约束是区分 demo 代码和工程代码的关键。
3. **分步来，不要一口吃成胖子**——8 步、每一步一个职责，比一股脑全部说完成功率更高。

记住：**你的角色从“写代码的人”变成了“定义需求的人”**。需求写得越精确，AI 就越像一个有 10 年经验的资深工程师在你身边敲键盘。

---

## 十三、下篇预告

**下一篇文章：《Cursor Composer 多文件联动修改——需求变更时，如何一句话让 12 个文件同步更新》**

一个真实的日常场景：

你刚刚把图书管理系统写完，测试也跑通了。这时候产品经理在群里发了条消息：“PRD 更新了，图书的 `price` 字段要改成 `Money` 值对象，包含 `amount` 和 `currency` 两个字段。API 返回里也要加上币种信息。”

你心里一紧——这意味着：Entity 要改、DTO 要改、Service 的更新逻辑要改、Controller 的参数类型要改、单元测试的断言要改、Swagger 文档里的字段描述要改、甚至 Postman Collection 里的示例 JSON 也要改。保守估计涉及 12 个文件。

放在以前，这活儿没有两个小时下不来，中间还得反复检查漏没漏改哪个文件。但在 Cursor Composer 里，你只需要输入一句 Prompt：

> "把 price 字段重构为 Money 值对象（amount + currency），同步更新所有关联文件。"

X 个文件瞬间同步修改完毕，连单元测试的断言都自动适配了新结构。

**这就是 AI 编程的“牵一发而动全身”能力。关注本系列，下期为你完整演示。**

---

*本文使用的工具：Cursor 0.46 + Claude 4（内置）*
*示例项目完整源码可通过 GitHub 搜索 `cursor-springboot-crud-demo` 获取*
