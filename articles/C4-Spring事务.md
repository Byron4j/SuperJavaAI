# Spring事务深度解析：传播机制、隔离级别与失效场景全链路源码分析

**文章标签：** #java #spring #事务 #传播机制 #隔离级别 #源码分析 #设计模式 #面试 #深度解析

## 目录

- [引言：为什么需要事务管理](#引言为什么需要事务管理)
- [理论基础：ACID与事务隔离](#理论基础acid与事务隔离)
- [来龙去脉：Spring事务的发展史](#来龙去脉spring事务的发展史)
- [Spring事务架构与源码深度分析](#spring事务架构与源码深度分析)
- [事务的传播行为详解](#事务的传播行为详解)
- [事务的隔离级别详解](#事务的隔离级别详解)
- [@Transactional工作原理与源码分析](#transactional工作原理与源码分析)
- [设计模式解析](#设计模式解析)
- [实战案例：工业级事务管理](#实战案例工业级事务管理)
- [对比分析：Spring事务 vs 替代方案](#对比分析spring事务-vs-替代方案)
- [性能分析与优化](#性能分析与优化)
- [常见陷阱与最佳实践](#常见陷阱与最佳实践)
- [面试题与参考答案](#面试题与参考答案)

---

## 引言：为什么需要事务管理

事务管理不是简单的"begin-commit-rollback"，而是**保证数据一致性**的核心机制。在分布式系统和并发环境下，事务管理变得尤为复杂和重要。

**核心认知：**

```
没有事务管理的问题：
┌─────────────────────────────────────────┐
│  转账操作                                │
│  1. 扣减A账户余额                         │
│  2. 增加B账户余额                         │
│                                         │
│  如果步骤1成功，步骤2失败：               │
│  - A的钱扣了，B的钱没增加                │
│  - 数据不一致！                          │
│  - 资金丢失！                            │
└─────────────────────────────────────────┘

事务管理保证：
┌─────────────────────────────────────────┐
│  转账操作（事务保护）                      │
│  BEGIN TRANSACTION                        │
│  1. 扣减A账户余额                         │
│  2. 增加B账户余额                         │
│  COMMIT                                   │
│                                         │
│  如果步骤2失败：                          │
│  - ROLLBACK                               │
│  - A和B的余额都恢复到操作前              │
│  - 数据一致性得到保证                     │
└─────────────────────────────────────────┘
```

**关键洞察**：Spring事务的核心价值在于**声明式事务管理**，通过AOP将事务逻辑从业务代码中分离，使得开发者可以专注于业务逻辑，而无需关心事务的开启、提交和回滚。

---

## 理论基础：ACID与事务隔离

### 1. ACID特性

```
ACID四大特性：

A - Atomicity（原子性）
  事务是不可分割的最小工作单位，要么全部成功，要么全部失败。
  实现：undo log（回滚日志）

C - Consistency（一致性）
  事务执行前后，数据库从一个一致状态变为另一个一致状态。
  实现：约束检查、触发器、级联操作

I - Isolation（隔离性）
  多个事务并发执行时，一个事务的执行不应影响其他事务。
  实现：锁机制、MVCC

D - Durability（持久性）
  事务一旦提交，对数据库的修改就是永久性的。
  实现：redo log（重做日志）、磁盘写入
```

### 2. 并发事务问题

```
并发事务的三种问题：

1. 脏读（Dirty Read）
   事务A读取了事务B未提交的数据
   ┌─────────────────────────────────────────┐
   │  事务A          │  事务B                │
   │  读取余额=100   │                       │
   │                 │  扣减余额=90（未提交） │
   │  读取余额=90    │  ← 脏读！             │
   │                 │  ROLLBACK             │
   │  余额实际=100   │                       │
   └─────────────────────────────────────────┘

2. 不可重复读（Non-repeatable Read）
   同一事务内，两次读取同一数据，结果不同
   ┌─────────────────────────────────────────┐
   │  事务A          │  事务B                │
   │  读取余额=100   │                       │
   │                 │  扣减余额=90（提交）   │
   │  读取余额=90    │  ← 不可重复读！       │
   └─────────────────────────────────────────┘

3. 幻读（Phantom Read）
   同一事务内，两次查询，结果集行数不同
   ┌─────────────────────────────────────────┐
   │  事务A          │  事务B                │
   │  查询：10条记录 │                       │
   │                 │  插入1条记录（提交）   │
   │  查询：11条记录 │  ← 幻读！             │
   └─────────────────────────────────────────┘
```

### 3. 事务隔离级别

| 隔离级别 | 脏读 | 不可重复读 | 幻读 |
|---------|------|-----------|------|
| READ_UNCOMMITTED | 可能 | 可能 | 可能 |
| READ_COMMITTED | 否 | 可能 | 可能 |
| REPEATABLE_READ | 否 | 否 | 可能 |
| SERIALIZABLE | 否 | 否 | 否 |

MySQL默认`REPEATABLE_READ`，Oracle默认`READ_COMMITTED`。

---

## 来龙去脉：Spring事务的发展史

### 第一阶段：编程式事务（Spring 1.x）

```java
// Spring 1.x的编程式事务
TransactionTemplate template = new TransactionTemplate(transactionManager);
template.execute(new TransactionCallback() {
    public Object doInTransaction(TransactionStatus status) {
        jdbcTemplate.update("INSERT INTO user ...");
        return null;
    }
});
```

### 第二阶段：声明式事务（Spring 2.0+）

引入`@Transactional`注解：

```java
@Transactional
public void createOrder() {
    orderDao.insert(order);
    stockDao.decrease(stock);
}
```

### 第三阶段：注解驱动（Spring 2.5+）

`@EnableTransactionManagement`：

```java
@Configuration
@EnableTransactionManagement
public class AppConfig {
    @Bean
    public PlatformTransactionManager transactionManager() {
        return new DataSourceTransactionManager(dataSource());
    }
}
```

### 第四阶段：Spring Boot自动配置（2014至今）

Spring Boot自动配置事务管理器：

```java
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
// 无需手动配置，Spring Boot自动检测DataSource并配置事务管理器
```

---

## Spring事务架构与源码深度分析

### Spring事务架构图

```
┌─────────────────────────────────────────────────────────────┐
│                    Spring 事务架构                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  业务层                                                      │
│  ┌─────────────────────────────────────────────────────────┐│
│  │ @Transactional 业务方法                                  ││
│  └─────────────────────────────────────────────────────────┘│
│                          │                                  │
│  AOP代理层                                                  │
│  ┌─────────────────────────────────────────────────────────┐│
│  │ TransactionInterceptor                                  ││
│  │  ┌───────────────────────────────────────────────────┐  ││
│  │  │ TransactionAspectSupport                           │  ││
│  │  │  - invokeWithinTransaction()                        │  ││
│  │  │  - createTransactionIfNecessary()                   │  ││
│  │  │  - commitTransactionAfterReturning()                │  ││
│  │  │  - completeTransactionAfterThrowing()               │  ││
│  │  └───────────────────────────────────────────────────┘  ││
│  └─────────────────────────────────────────────────────────┘│
│                          │                                  │
│  事务管理层                                                 │
│  ┌─────────────────────────────────────────────────────────┐│
│  │ PlatformTransactionManager                              ││
│  │  ┌─────────────┐ ┌─────────────┐ ┌───────────────────┐  ││
│  │  │DataSource   │ │Jpa          │ │Hibernate          │  ││
│  │  │Transaction  │ │Transaction  │ │Transaction        │  ││
│  │  │Manager      │ │Manager      │ │Manager            │  ││
│  │  └─────────────┘ └─────────────┘ └───────────────────┘  ││
│  └─────────────────────────────────────────────────────────┘│
│                          │                                  │
│  资源层                                                     │
│  ┌─────────────────────────────────────────────────────────┐│
│  │ DataSource / SessionFactory / EntityManagerFactory      ││
│  │  ┌───────────────────────────────────────────────────┐  ││
│  │  │ Connection / Session (绑定到线程)                  │  ││
│  │  │ TransactionSynchronizationManager                  │  ││
│  │  └───────────────────────────────────────────────────┘  ││
│  └─────────────────────────────────────────────────────────┘│
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### PlatformTransactionManager源码分析

Spring通过AOP提供声明式事务管理，核心抽象是`PlatformTransactionManager`：

```java
public interface PlatformTransactionManager {
    // 获取事务状态
    TransactionStatus getTransaction(@Nullable TransactionDefinition definition) 
            throws TransactionException;
    
    // 提交事务
    void commit(TransactionStatus status) throws TransactionException;
    
    // 回滚事务
    void rollback(TransactionStatus status) throws TransactionException;
}
```

### DataSourceTransactionManager源码分析

```java
// DataSourceTransactionManager - JDBC/MyBatis事务管理器
public class DataSourceTransactionManager extends AbstractPlatformTransactionManager
        implements ResourceTransactionManager, InitializingBean {
    
    @Nullable
    private DataSource dataSource;
    
    @Override
    protected Object doGetTransaction() {
        DataSourceTransactionObject txObject = new DataSourceTransactionObject();
        txObject.setSavepointAllowed(isNestedTransactionAllowed());
        
        // 从线程本地变量获取连接持有者
        ConnectionHolder conHolder = (ConnectionHolder) 
            TransactionSynchronizationManager.getResource(obtainDataSource());
        txObject.setConnectionHolder(conHolder, false);
        return txObject;
    }
    
    @Override
    protected void doBegin(Object transaction, TransactionDefinition definition) {
        DataSourceTransactionObject txObject = (DataSourceTransactionObject) transaction;
        Connection con = null;
        
        // 获取数据库连接
        if (txObject.getConnectionHolder() == null ||
                txObject.getConnectionHolder().isSynchronizedWithTransaction()) {
            Connection newCon = obtainDataSource().getConnection();
            txObject.setConnectionHolder(new ConnectionHolder(newCon), true);
        }
        
        con = txObject.getConnectionHolder().getConnection();
        
        // 设置隔离级别
        Integer previousIsolationLevel = DataSourceUtils.prepareConnectionForTransaction(con, definition);
        txObject.setPreviousIsolationLevel(previousIsolationLevel);
        txObject.setReadOnly(definition.isReadOnly());
        
        // 切换为手动提交
        if (con.getAutoCommit()) {
            txObject.setMustRestoreAutoCommit(true);
            con.setAutoCommit(false);
        }
        
        // 设置超时
        int timeout = determineTimeout(definition);
        if (timeout != TransactionDefinition.TIMEOUT_DEFAULT) {
            txObject.getConnectionHolder().setTimeoutInSeconds(timeout);
        }
        
        // 绑定连接到线程
        if (txObject.isNewConnectionHolder()) {
            TransactionSynchronizationManager.bindResource(
                obtainDataSource(), txObject.getConnectionHolder());
        }
    }
    
    @Override
    protected void doCommit(DefaultTransactionStatus status) {
        DataSourceTransactionObject txObject = (DataSourceTransactionObject) status.getTransaction();
        Connection con = txObject.getConnectionHolder().getConnection();
        try {
            con.commit();
        } catch (SQLException ex) {
            throw new TransactionSystemException("Could not commit JDBC transaction", ex);
        }
    }
    
    @Override
    protected void doRollback(DefaultTransactionStatus status) {
        DataSourceTransactionObject txObject = (DataSourceTransactionObject) status.getTransaction();
        Connection con = txObject.getConnectionHolder().getConnection();
        try {
            con.rollback();
        } catch (SQLException ex) {
            throw new TransactionSystemException("Could not roll back JDBC transaction", ex);
        }
    }
}
```

### 事务定义与状态

```java
public interface TransactionDefinition {
    int PROPAGATION_REQUIRED = 0;
    int PROPAGATION_SUPPORTS = 1;
    int PROPAGATION_MANDATORY = 2;
    int PROPAGATION_REQUIRES_NEW = 3;
    int PROPAGATION_NOT_SUPPORTED = 4;
    int PROPAGATION_NEVER = 5;
    int PROPAGATION_NESTED = 6;
    
    int ISOLATION_DEFAULT = -1;
    int ISOLATION_READ_UNCOMMITTED = 1;
    int ISOLATION_READ_COMMITTED = 2;
    int ISOLATION_REPEATABLE_READ = 4;
    int ISOLATION_SERIALIZABLE = 8;
    
    int TIMEOUT_DEFAULT = -1;
    
    int getPropagationBehavior();
    int getIsolationLevel();
    int getTimeout();
    boolean isReadOnly();
    @Nullable String getName();
}

public interface TransactionStatus extends SavepointManager, Flushable {
    boolean isNewTransaction();
    boolean hasSavepoint();
    void setRollbackOnly();
    boolean isRollbackOnly();
    void flush();
    boolean isCompleted();
}
```

---

## 事务的传播行为详解

传播行为定义了**当前方法如何参与已存在的事务**。

```java
public enum Propagation {
    REQUIRED(0),      // 默认，有则加入，无则新建
    SUPPORTS(1),      // 有则加入，无则以非事务执行
    MANDATORY(2),     // 有则加入，无则抛异常
    REQUIRES_NEW(3),  // 新建事务，挂起当前事务
    NOT_SUPPORTED(4), // 以非事务执行，挂起当前事务
    NEVER(5),         // 以非事务执行，有事务则抛异常
    NESTED(6);        // 在当前事务中创建嵌套事务（savepoint）
}
```

### REQUIRED（默认）

```java
@Service
public class OrderService {
    @Transactional
    public void createOrder() {
        // 有事务，直接加入
        orderDao.insert(order);
        
        // 调用另一个事务方法
        stockService.decreaseStock(); // 加入当前事务
    }
}

@Service
public class StockService {
    @Transactional(propagation = Propagation.REQUIRED)
    public void decreaseStock() {
        stockDao.update(stock);
    }
}
```

createOrder和decreaseStock在同一个事务中，任何一个失败都会回滚。

### REQUIRES_NEW

```java
@Service
public class LogService {
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void saveLog(Log log) {
        logDao.insert(log);
    }
}

@Service
public class OrderService {
    @Transactional
    public void createOrder() {
        try {
            orderDao.insert(order);
            stockService.decreaseStock();
        } catch (Exception e) {
            // 订单失败，但日志要保存
            logService.saveLog(new Log("订单失败: " + e.getMessage()));
            throw e;
        }
    }
}
```

saveLog在独立的事务中，不受外部事务影响。

### NESTED

```java
@Transactional(propagation = Propagation.NESTED)
public void nestedMethod() {
    // 创建savepoint
    try {
        // 业务逻辑
    } catch (Exception e) {
        // 回滚到savepoint，而不是整个事务
        throw e;
    }
}
```

NESTED与REQUIRES_NEW的区别：
- NESTED：在当前事务中创建savepoint，失败只回滚到savepoint
- REQUIRES_NEW：创建全新的事务，完全独立

---

## 事务的隔离级别详解

```java
public enum Isolation {
    DEFAULT(-1),      // 使用数据库默认
    READ_UNCOMMITTED(1),  // 读未提交
    READ_COMMITTED(2),    // 读已提交
    REPEATABLE_READ(4),   // 可重复读
    SERIALIZABLE(8);      // 串行化
}
```

| 隔离级别 | 脏读 | 不可重复读 | 幻读 |
|---------|------|-----------|------|
| READ_UNCOMMITTED | 可能 | 可能 | 可能 |
| READ_COMMITTED | 否 | 可能 | 可能 |
| REPEATABLE_READ | 否 | 否 | 可能 |
| SERIALIZABLE | 否 | 否 | 否 |

MySQL默认`REPEATABLE_READ`，Oracle默认`READ_COMMITTED`。

---

## @Transactional工作原理与源码分析

### TransactionInterceptor源码分析

Spring通过AOP在`@Transactional`方法前后管理事务：

```java
public class TransactionInterceptor extends TransactionAspectSupport implements MethodInterceptor {
    
    @Override
    @Nullable
    public Object invoke(MethodInvocation invocation) throws Throwable {
        Class<?> targetClass = (invocation.getThis() != null ? 
            AopUtils.getTargetClass(invocation.getThis()) : null);
        
        return invokeWithinTransaction(invocation.getMethod(), targetClass, 
            invocation::proceed);
    }
}
```

### TransactionAspectSupport核心逻辑

```java
public abstract class TransactionAspectSupport implements BeanFactoryAware, InitializingBean {
    
    @Nullable
    protected Object invokeWithinTransaction(Method method, @Nullable Class<?> targetClass,
            final InvocationCallback invocation) throws Throwable {
        
        // 1. 获取事务属性（@Transactional配置）
        TransactionAttributeSource tas = getTransactionAttributeSource();
        final TransactionAttribute txAttr = (tas != null ? 
            tas.getTransactionAttribute(method, targetClass) : null);
        
        // 2. 获取事务管理器
        final PlatformTransactionManager tm = determineTransactionManager(txAttr);
        
        // 3. 获取连接点标识（方法全限定名）
        final String joinpointIdentification = methodIdentification(method, targetClass, txAttr);
        
        // 4. 声明式事务处理
        if (txAttr == null || !(tm instanceof CallbackPreferringPlatformTransactionManager)) {
            // 创建事务（根据传播行为决定加入或新建）
            TransactionInfo txInfo = createTransactionIfNecessary(tm, txAttr, joinpointIdentification);
            
            Object retVal;
            try {
                // 执行业务方法
                retVal = invocation.proceedWithInvocation();
            }
            catch (Throwable ex) {
                // 异常时回滚
                completeTransactionAfterThrowing(txInfo, ex);
                throw ex;
            }
            finally {
                cleanupTransactionInfo(txInfo);
            }
            
            // 提交事务
            commitTransactionAfterReturning(txInfo);
            return retVal;
        }
        else {
            // 编程式事务处理
            // ...
        }
    }
    
    protected void completeTransactionAfterThrowing(@Nullable TransactionInfo txInfo, Throwable ex) {
        if (txInfo != null && txInfo.getTransactionStatus() != null) {
            // 判断异常是否需要回滚
            if (txInfo.transactionAttribute != null && 
                    txInfo.transactionAttribute.rollbackOn(ex)) {
                // 回滚
                txInfo.getTransactionManager().rollback(txInfo.getTransactionStatus());
            }
            else {
                // 提交（异常不需要回滚）
                txInfo.getTransactionManager().commit(txInfo.getTransactionStatus());
            }
        }
    }
}
```

### 事务执行流程

```
调用方
  │
  ▼
Proxy (AOP代理)
  │
  ├──► TransactionInterceptor.invoke()
  │       │
  │       ├──► 获取@Transactional属性
  │       │
  │       ├──► 确定TransactionManager
  │       │
  │       ├──► createTransactionIfNecessary()
  │       │       │
  │       │       ├──► tm.getTransaction(txAttr) // 根据传播行为获取/创建事务
  │       │       │       │
  │       │       │       ├──► 无事务 → 新建事务（doBegin）
  │       │       │       │       ├──► 获取Connection
  │       │       │       │       ├──► 设置隔离级别
  │       │       │       │       ├──► setAutoCommit(false)
  │       │       │       │       └──► 绑定到线程
  │       │       │       │
  │       │       │       └──► 有事务 → 根据传播行为处理
  │       │       │               ├──► REQUIRED → 加入当前事务
  │       │       │               ├──► REQUIRES_NEW → 挂起当前，新建事务
  │       │       │               └──► NESTED → 创建savepoint
  │       │       │
  │       │       └──► 绑定TransactionInfo到线程
  │       │
  │       ├──► invocation.proceed() // 执行业务方法
  │       │
  │       ├──► 业务方法正常返回
  │       │       └──► commitTransactionAfterReturning()
  │       │               └──► tm.commit() // 提交
  │       │
  │       └──► 业务方法抛异常
  │               └──► completeTransactionAfterThrowing()
  │                       └──► 判断异常类型
  │                               ├──► 需要回滚 → tm.rollback()
  │                               └──► 不需要回滚 → tm.commit()
  │
  └──► 返回结果
```

---

## 设计模式解析

### 1. 代理模式（Proxy Pattern）

**体现：** Spring通过AOP代理在方法前后添加事务控制逻辑。

**为什么：** 在不修改业务代码的情况下，统一添加事务管理。业务代码只关注业务逻辑，事务控制由代理透明处理。

### 2. 模板方法模式（Template Method）

**体现：** `AbstractPlatformTransactionManager`定义了事务管理的标准流程（getTransaction → doBegin → doCommit/doRollback），具体实现由子类完成。

**为什么：** 统一事务管理流程，同时支持不同数据源（JDBC、JPA、Hibernate）。

```java
public abstract class AbstractPlatformTransactionManager {
    @Override
    public final TransactionStatus getTransaction(@Nullable TransactionDefinition definition) {
        // 模板方法：定义标准流程
        Object transaction = doGetTransaction();
        // ...
        if (isExistingTransaction(transaction)) {
            return handleExistingTransaction(definition, transaction, debugEnabled);
        }
        // ...
        return startTransaction(definition, transaction, debugEnabled, suspendedResources);
    }
    
    // 抽象方法，子类实现
    protected abstract Object doGetTransaction() throws TransactionException;
    protected abstract void doBegin(Object transaction, TransactionDefinition definition);
    protected abstract void doCommit(DefaultTransactionStatus status);
    protected abstract void doRollback(DefaultTransactionStatus status);
}
```

### 3. 策略模式（Strategy Pattern）

**体现：** `TransactionDefinition`定义事务配置接口，不同的传播行为和隔离级别是具体的策略。

**为什么：** 允许灵活配置事务行为，而不改变事务管理器的核心逻辑。

---

## 实战案例：工业级事务管理

### 案例1：完整声明式事务示例

```java
@Service
public class OrderService {
    
    @Autowired
    private OrderDao orderDao;
    
    @Autowired
    private StockService stockService;
    
    @Autowired
    private LogService logService;
    
    // 主业务方法：创建订单
    @Transactional(rollbackFor = Exception.class, timeout = 30)
    public Order createOrder(CreateOrderRequest request) {
        // 1. 创建订单
        Order order = new Order();
        order.setUserId(request.getUserId());
        order.setAmount(request.getAmount());
        orderDao.insert(order);
        
        // 2. 扣减库存（加入当前事务）
        stockService.decreaseStock(request.getProductId(), request.getQuantity());
        
        // 3. 记录日志（独立事务）
        logService.saveLog("创建订单: " + order.getId());
        
        return order;
    }
    
    // 查询方法：不需要事务
    @Transactional(readOnly = true, propagation = Propagation.SUPPORTS)
    public Order getOrder(Long orderId) {
        return orderDao.selectById(orderId);
    }
}
```

```java
@Service
public class StockService {
    
    @Autowired
    private StockDao stockDao;
    
    @Transactional(propagation = Propagation.REQUIRED)
    public void decreaseStock(Long productId, Integer quantity) {
        Stock stock = stockDao.selectByProductId(productId);
        if (stock.getQuantity() < quantity) {
            throw new InsufficientStockException("库存不足");
        }
        stockDao.decrease(productId, quantity);
    }
}
```

```java
@Service
public class LogService {
    
    @Autowired
    private LogDao logDao;
    
    // 独立事务：即使主事务回滚，日志也要保存
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void saveLog(String message) {
        Log log = new Log();
        log.setMessage(message);
        log.setCreateTime(new Date());
        logDao.insert(log);
    }
}
```

### 案例2：编程式事务示例

```java
@Service
public class ProgrammaticTransactionService {
    
    // 方式1：使用TransactionTemplate
    @Autowired
    private TransactionTemplate transactionTemplate;
    
    public void createOrderWithTemplate(Order order) {
        Boolean result = transactionTemplate.execute(status -> {
            try {
                orderDao.insert(order);
                stockDao.decrease(order.getProductId(), order.getQuantity());
                return true;
            } catch (InsufficientStockException e) {
                status.setRollbackOnly();
                return false;
            }
        });
        
        if (!Boolean.TRUE.equals(result)) {
            throw new BusinessException("创建订单失败");
        }
    }
    
    // 方式2：使用TransactionManager
    @Autowired
    private PlatformTransactionManager transactionManager;
    
    public void createOrderWithManager(Order order) {
        TransactionDefinition def = new DefaultTransactionDefinition();
        TransactionStatus status = transactionManager.getTransaction(def);
        
        try {
            orderDao.insert(order);
            stockDao.decrease(order.getProductId(), order.getQuantity());
            transactionManager.commit(status);
        } catch (Exception e) {
            transactionManager.rollback(status);
            throw new BusinessException("创建订单失败", e);
        }
    }
}
```

### 案例3：嵌套事务示例

```java
@Service
public class BatchService {
    
    @Autowired
    private ItemService itemService;
    
    // 主事务
    @Transactional(rollbackFor = Exception.class)
    public void batchProcess(List<OrderItem> items) {
        for (OrderItem item : items) {
            try {
                // 每个item使用嵌套事务
                itemService.processItem(item);
            } catch (Exception e) {
                // 某个item失败，记录错误，继续处理其他
                log.error("处理item失败: " + item.getId(), e);
                // 由于NESTED创建的是savepoint，这里可以回滚到savepoint
                // 继续处理下一个item
            }
        }
    }
}

@Service
public class ItemService {
    
    @Autowired
    private ItemDao itemDao;
    
    // 嵌套事务：失败只回滚当前item
    @Transactional(propagation = Propagation.NESTED)
    public void processItem(OrderItem item) {
        itemDao.updateStatus(item.getId(), "PROCESSING");
        
        // 处理逻辑
        if (item.getAmount().compareTo(BigDecimal.ZERO) <= 0) {
            throw new IllegalArgumentException("金额必须大于0");
        }
        
        itemDao.updateStatus(item.getId(), "COMPLETED");
    }
}
```

---

## 对比分析：Spring事务 vs 替代方案

### Spring声明式事务 vs 编程式事务

| 特性 | @Transactional | TransactionTemplate |
|------|---------------|---------------------|
| 代码侵入性 | 低（注解） | 中（代码块） |
| 灵活性 | 低（固定边界） | 高（精确控制） |
| 可读性 | 高 | 中 |
| 适用场景 | 简单事务 | 复杂事务逻辑 |

### Spring事务 vs JTA全局事务

| 特性 | Spring本地事务 | JTA全局事务 |
|------|---------------|-------------|
| 资源数 | 单个数据源 | 多个数据源 |
| 复杂度 | 低 | 高 |
| 性能 | 高 | 低（两阶段提交） |
| 适用场景 | 单数据库 | 分布式事务 |

### Spring事务 vs EJB容器事务

| 特性 | Spring | EJB |
|------|--------|-----|
| 容器依赖 | 无（独立框架） | 需要EJB容器 |
| 配置方式 | 注解/XML | 注解 |
| 灵活性 | 高 | 中 |
| 学习曲线 | 低 | 高 |

---

## 性能分析与优化

### 1. 连接池配置

```properties
# HikariCP配置
spring.datasource.hikari.maximum-pool-size=20
spring.datasource.hikari.minimum-idle=5
spring.datasource.hikari.connection-timeout=30000
spring.datasource.hikari.idle-timeout=600000
spring.datasource.hikari.max-lifetime=1800000
```

### 2. 事务超时设置

```java
@Transactional(timeout = 10) // 10秒超时
public void quickOperation() {
    // 简短的事务操作
}

@Transactional(timeout = 300) // 5分钟超时
public void longRunningOperation() {
    // 长时间操作
}
```

### 3. 只读事务优化

```java
@Transactional(readOnly = true)
public List<User> getUsers() {
    // 只读事务可以优化：
    // 1. 不刷新持久化上下文
    // 2. 不检测脏数据
    // 3. 某些数据库可以优化查询计划
    return userDao.findAll();
}
```

### 4. 避免大事务

```java
// 错误：大事务
@Transactional
public void processAll() {
    for (int i = 0; i < 100000; i++) {
        dao.insert(data[i]); // 长时间占用连接
    }
}

// 正确：分批处理
public void processAll() {
    for (int i = 0; i < 100000; i += 1000) {
        processBatch(data, i, i + 1000);
    }
}

@Transactional
public void processBatch(Object[] data, int start, int end) {
    for (int i = start; i < end; i++) {
        dao.insert(data[i]);
    }
}
```

### 5. 事务监控

```java
@Component
public class TransactionMetrics {
    
    @Around("@annotation(transactional)")
    public Object monitorTransaction(ProceedingJoinPoint pjp, Transactional transactional) throws Throwable {
        long start = System.currentTimeMillis();
        try {
            return pjp.proceed();
        } finally {
            long cost = System.currentTimeMillis() - start;
            if (cost > 5000) {
                System.out.println("慢事务: " + pjp.getSignature().getName() + " 耗时: " + cost + "ms");
            }
        }
    }
}
```

---

## 常见陷阱与最佳实践

### 陷阱1：默认事务只回滚RuntimeException

```java
@Transactional
public void transfer() throws Exception {
    // 业务逻辑
    throw new Exception("checked exception"); // 不会回滚！
}
```

**最佳实践：** 明确指定`rollbackFor = Exception.class`，或统一使用运行时异常。

```java
@Transactional(rollbackFor = Exception.class)
public void transfer() throws Exception {
    // ...
}
```

### 陷阱2：事务方法中调用外部HTTP/RPC服务

```java
@Transactional
public void createOrder() {
    orderDao.insert(order);
    // 长时间阻塞，占用数据库连接
    paymentService.charge(order.getAmount()); // RPC调用
}
```

**最佳实践：** 将外部调用放在事务外，或缩短事务边界，避免长时间占用数据库连接。

```java
public void createOrder() {
    // 前置校验（无事务）
    validate(order);
    
    // 缩短事务范围
    transactionTemplate.execute(status -> {
        orderDao.insert(order);
        return order;
    });
    
    // 外部调用（事务外）
    paymentService.charge(order.getAmount());
}
```

### 陷阱3：REQUIRES_NEW滥用导致死锁

```java
@Transactional
public void parent() {
    updateTableA();
    childService.child(); // REQUIRES_NEW，可能死锁
}

@Transactional(propagation = Propagation.REQUIRES_NEW)
public void child() {
    updateTableA(); // 同一行数据，死锁！
}
```

**最佳实践：** 避免在REQUIRES_NEW中操作父事务已持有的资源，或调整业务逻辑避免资源竞争。

### 陷阱4：忽略事务超时设置

```java
@Transactional
public void batchProcess() {
    // 批量处理大量数据，可能超时
    for (int i = 0; i < 100000; i++) {
        dao.insert(data);
    }
}
```

**最佳实践：** 大事务拆分为小事务，或设置合理的超时时间。

```java
@Transactional(timeout = 30) // 30秒超时
public void batchProcess() {
    // ...
}
```

### 陷阱5：同类方法调用导致事务失效

```java
@Service
public class OrderService {
    @Transactional
    public void createOrder() {
        updateOrder(); // 同类调用，事务失效！
    }
    
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void updateOrder() {
    }
}
```

**最佳实践：** 通过注入自身代理或从容器获取Bean调用。

```java
@Service
public class OrderService {
    @Autowired
    private OrderService self;
    
    @Transactional
    public void createOrder() {
        self.updateOrder(); // 通过代理调用
    }
}
```

### 陷阱6：异常被捕获导致事务不回滚

```java
@Transactional
public void createOrder() {
    try {
        orderDao.insert(order);
        stockService.decreaseStock();
    } catch (Exception e) {
        log.error("创建订单失败", e);
        // 异常被捕获，事务不会回滚！
    }
}
```

**最佳实践：** 重新抛出异常或手动回滚。

```java
@Transactional
public void createOrder() {
    try {
        orderDao.insert(order);
        stockService.decreaseStock();
    } catch (Exception e) {
        log.error("创建订单失败", e);
        TransactionAspectSupport.currentTransactionStatus().setRollbackOnly();
        throw new BusinessException("创建订单失败", e);
    }
}
```

---

## 面试题与参考答案

### Q1：Spring事务的7种传播行为分别是什么？各自的适用场景？

**答：** 
- **REQUIRED（默认）**：有则加入，无则新建。适用于大多数场景。
- **SUPPORTS**：有则加入，无则以非事务执行。适用于查询方法，不强制要求事务。
- **MANDATORY**：有则加入，无则抛异常。适用于必须存在事务的场景，如子方法。
- **REQUIRES_NEW**：新建事务，挂起当前事务。适用于独立操作，如日志记录。
- **NOT_SUPPORTED**：以非事务执行，挂起当前事务。适用于不需要事务的方法。
- **NEVER**：以非事务执行，有事务则抛异常。适用于绝对不能在事务中执行的操作。
- **NESTED**：在当前事务中创建嵌套事务（savepoint）。适用于需要部分回滚的场景。

### Q2：@Transactional注解为什么不建议加在Controller层？

**答：** 
1. **职责分离**：Controller负责请求处理，Service负责业务逻辑和事务管理
2. **粒度控制**：Service层可以更细粒度地控制事务边界
3. **复用性**：Service方法可能被多个Controller调用，事务统一管理
4. **测试性**：Service层独立测试更方便
5. **异常处理**：Controller层需要处理请求参数校验、异常转换等，与事务管理关注点不同

**最佳实践：** 事务注解加在Service层，Controller层负责参数校验和结果封装。

### Q3：Spring事务失效的常见场景有哪些？如何排查？

**答：** 
1. **非public方法**：Spring AOP基于代理，无法拦截非public方法
2. **同类方法调用**：`this.method()`不走代理
3. **异常被捕获**：捕获异常后未重新抛出或手动回滚
4. **异常类型不匹配**：默认只回滚RuntimeException
5. **数据库不支持事务**：如MyISAM引擎
6. **未配置事务管理器**：缺少`@EnableTransactionManagement`
7. **final方法/类**：无法被CGLIB代理
8. **异步方法**：`@Async`在新线程执行，事务可能不生效

**排查方法：** 开启事务日志`logging.level.org.springframework.transaction=DEBUG`，检查代理对象类型。

### Q4：REQUIRED和REQUIRES_NEW有什么区别？NESTED和REQUIRES_NEW呢？

**答：** 

**REQUIRED vs REQUIRES_NEW：**
- **REQUIRED**：加入当前事务，与父方法共用一个事务，失败一起回滚
- **REQUIRES_NEW**：创建全新事务，挂起当前事务，独立提交/回滚

**NESTED vs REQUIRES_NEW：**
- **NESTED**：在当前事务中创建savepoint，失败只回滚到savepoint，外层事务继续
- **REQUIRES_NEW**：完全独立的新事务，有独立的连接和隔离级别

使用场景：
- 需要独立提交（如日志）→ REQUIRES_NEW
- 需要部分回滚（如批量处理中某条失败）→ NESTED

### Q5：脏读、不可重复读、幻读有什么区别？MySQL默认隔离级别是什么？

**答：** 

| 问题 | 描述 | 隔离级别解决 |
|------|------|-------------|
| **脏读** | 读取到其他事务未提交的数据 | READ_COMMITTED |
| **不可重复读** | 同一事务内两次读取同一数据结果不同 | REPEATABLE_READ |
| **幻读** | 同一事务内两次查询，结果集行数不同 | SERIALIZABLE（MySQL的RR通过MVCC基本解决） |

**MySQL默认隔离级别：REPEATABLE_READ（可重复读）**

MySQL通过MVCC（多版本并发控制）在REPEATABLE_READ级别下基本解决了幻读问题，但不完全（如当前读仍可能出现幻读）。

### Q6：编程式事务和声明式事务如何选择？

**答：** 
- **声明式事务**（`@Transactional`）：简单、解耦，适用于大多数场景。通过AOP实现，无法控制细粒度的事务边界。
- **编程式事务**（`TransactionTemplate`）：灵活，可以精确控制事务范围、处理复杂逻辑（如部分回滚）。代码侵入性较强。

**选择建议：** 优先使用声明式事务，遇到复杂事务逻辑（如循环中部分提交/回滚）时使用编程式事务。

### Q7：Spring事务和数据库事务是什么关系？

**答：** Spring事务是对数据库事务的抽象和封装。Spring通过PlatformTransactionManager管理底层数据库连接的事务状态（begin/commit/rollback），但最终还是依赖于数据库的事务支持。Spring事务的主要价值在于：
1. 统一的事务管理API，屏蔽不同数据访问技术的差异
2. 声明式事务，通过AOP简化事务管理
3. 事务传播行为，处理复杂的事务边界

### Q8：什么是事务挂起（Suspension）？

**答：** 事务挂起是指当使用REQUIRES_NEW或NOT_SUPPORTED传播行为时，Spring会将当前事务的上下文（连接、状态等）保存起来，创建新的事务（或非事务环境），待新事务完成后，再恢复之前的事务上下文。

```java
// 挂起和恢复的核心代码在AbstractPlatformTransactionManager中
protected final SuspendedResourcesHolder suspend(@Nullable Object transaction) {
    if (TransactionSynchronizationManager.isSynchronizationActive()) {
        List<TransactionSynchronization> suspendedSynchronizations = 
            TransactionSynchronizationManager.getSynchronizations();
        // 解绑资源、清除同步
        // ...
        return new SuspendedResourcesHolder(
            suspendedResources, suspendedSynchronizations, name, readOnly, isolationLevel, wasActive);
    }
    // ...
}
```

### Q9：Spring事务的隔离级别是如何实现的？

**答：** Spring事务的隔离级别最终是通过设置数据库连接的隔离级别来实现的：

```java
// DataSourceUtils.prepareConnectionForTransaction()
public static Integer prepareConnectionForTransaction(Connection con, @Nullable TransactionDefinition definition)
        throws SQLException {
    if (definition != null && definition.getIsolationLevel() != TransactionDefinition.ISOLATION_DEFAULT) {
        int currentIsolation = con.getTransactionIsolation();
        if (currentIsolation != definition.getIsolationLevel()) {
            con.setTransactionIsolation(definition.getIsolationLevel());
            return currentIsolation;
        }
    }
    return null;
}
```

### Q10：如何在Spring中实现分布式事务？

**答：** Spring支持多种分布式事务方案：
1. **JTA**：使用Atomikos或Bitronix等JTA实现
2. **XA协议**：支持两阶段提交
3. **Seata**：阿里巴巴开源的分布式事务解决方案
4. **Saga模式**：长事务拆分，通过补偿实现一致性

**示例（JTA）：**
```java
@Configuration
public class XAConfig {
    @Bean
    public JtaTransactionManager jtaTransactionManager() {
        UserTransactionManager userTransactionManager = new UserTransactionManager();
        return new JtaTransactionManager(userTransactionManager);
    }
}
```

### Q11：Spring事务的隔离级别有哪些？

**答：** 
- **DEFAULT**：使用数据库默认隔离级别
- **READ_UNCOMMITTED**：读未提交，可能产生脏读、不可重复读、幻读
- **READ_COMMITTED**：读已提交，可能产生不可重复读、幻读
- **REPEATABLE_READ**：可重复读，可能产生幻读
- **SERIALIZABLE**：串行化，最高隔离级别，但性能最差

### Q12：@Transactional注解的常用属性有哪些？

**答：** 
- **propagation**：事务传播行为（默认REQUIRED）
- **isolation**：事务隔离级别（默认DEFAULT）
- **timeout**：事务超时时间（秒）
- **readOnly**：是否只读事务
- **rollbackFor**：需要回滚的异常类型
- **noRollbackFor**：不需要回滚的异常类型

### Q13：Spring事务管理器有哪些实现？

**答：** 
- **DataSourceTransactionManager**：用于JDBC和MyBatis
- **JpaTransactionManager**：用于JPA
- **HibernateTransactionManager**：用于Hibernate
- **JtaTransactionManager**：用于JTA分布式事务
- **WebSphereUowTransactionManager**：用于WebSphere

### Q14：@Transactional注解可以标注在类上吗？

**答：** 可以。标注在类上时，该类中的所有public方法都会继承事务配置。方法上的@Transactional注解会覆盖类上的配置。

```java
@Service
@Transactional(rollbackFor = Exception.class)
public class OrderService {
    // 所有public方法都有事务
    
    @Transactional(readOnly = true) // 覆盖类配置
    public Order getOrder(Long id) {
        return orderDao.selectById(id);
    }
}
```

### Q15：Spring事务的readOnly=true有什么作用？

**答：** `readOnly=true`表示只读事务，有以下优化：
1. **Hibernate/JPA**：不检测脏数据，不刷新持久化上下文
2. **MySQL**：可以优化查询计划，某些场景下性能更好
3. **Oracle**：只读事务可以保证一致性读

**注意：** `readOnly=true`只是优化提示，某些数据库和驱动可能不支持。

### Q16：如何在Spring事务中设置保存点（Savepoint）？

**答：** 使用TransactionStatus接口的createSavepoint和rollbackToSavepoint方法：

```java
@Transactional
public void processWithSavepoint() {
    TransactionStatus status = TransactionAspectSupport.currentTransactionStatus();
    
    Object savepoint = status.createSavepoint();
    try {
        // 可能失败的操作
        riskyOperation();
    } catch (Exception e) {
        // 回滚到保存点，而不是整个事务
        status.rollbackToSavepoint(savepoint);
    } finally {
        status.releaseSavepoint(savepoint);
    }
}
```

### Q17：Spring事务和@Async一起使用会有什么问题？

**答：** `@Async`会在新线程中执行方法，而Spring事务是基于ThreadLocal的，新线程无法获取原线程的事务上下文。解决方案：

1. **在异步方法上也加@Transactional**
```java
@Async
@Transactional
public void asyncProcess() {
    // 新线程中开启新事务
}
```

2. **使用TransactionTemplate传递事务上下文**
```java
public void mainMethod() {
    transactionTemplate.execute(status -> {
        // 主线程事务
        asyncService.asyncProcess(data);
        return null;
    });
}
```

### Q18：Spring事务的timeout是如何实现的？

**答：** Spring事务的timeout是通过设置数据库连接的超时时间来实现的：

```java
// DataSourceTransactionManager.doBegin()
int timeout = determineTimeout(definition);
if (timeout != TransactionDefinition.TIMEOUT_DEFAULT) {
    txObject.getConnectionHolder().setTimeoutInSeconds(timeout);
}
```

**注意：** timeout是从事务开始计算，不是从方法开始计算。

### Q19：如何在同一个类中调用另一个@Transactional方法？

**答：** 同类方法调用会导致AOP失效，事务不生效。解决方案：

1. **注入自身代理**
```java
@Service
public class OrderService {
    @Autowired
    private OrderService self;
    
    @Transactional
    public void createOrder() {
        self.updateOrder(); // 通过代理调用
    }
    
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void updateOrder() {
    }
}
```

2. **使用AopContext**
```java
@Transactional
public void createOrder() {
    ((OrderService) AopContext.currentProxy()).updateOrder();
}
```

### Q20：Spring事务在测试环境中如何回滚？

**答：** 使用`@Transactional`注解在测试类上，测试完成后自动回滚：

```java
@SpringBootTest
@Transactional
class OrderServiceTest {
    @Autowired
    private OrderService orderService;
    
    @Test
    void testCreateOrder() {
        orderService.createOrder(order);
        // 测试完成后自动回滚，数据库不会残留数据
    }
}
```

如果需要提交，使用`@Rollback(false)`或`@Commit`。

### Q21：Spring事务的isolation和数据库的隔离级别是什么关系？

**答：** Spring事务的isolation属性最终映射到数据库连接的隔离级别设置。Spring通过`Connection.setTransactionIsolation()`方法设置数据库连接的隔离级别。如果数据库不支持某个隔离级别，可能会使用最接近的级别或抛出异常。

### Q22：Spring事务管理器如何选择？

**答：** 
- **单数据源JDBC/MyBatis**：使用`DataSourceTransactionManager`
- **JPA**：使用`JpaTransactionManager`
- **Hibernate**：使用`HibernateTransactionManager`
- **多数据源/分布式事务**：使用`JtaTransactionManager`
- **Spring Boot自动配置**：根据classpath自动选择合适的管理器

### Q23：Spring事务的name属性有什么用？

**答：** `name`属性用于给事务指定一个名称，便于监控和调试。事务名称会传递给底层的事务管理器，在数据库的监控工具中可以查看。

```java
@Transactional(name = "createOrderTransaction")
public void createOrder() {
    // 事务名称为 "createOrderTransaction"
}
```

### Q24：Spring事务的propagation.REQUIRED和默认传播行为有什么区别？

**答：** `Propagation.REQUIRED`是Spring事务的默认传播行为，两者没有区别。如果不指定传播行为，Spring默认使用`REQUIRED`。在有事务时加入当前事务，无事务时新建事务。

### Q25：如何在Spring Boot中配置多数据源事务？

**答：** 配置多个`DataSource`和对应的`PlatformTransactionManager`：

```java
@Configuration
public class MultiDataSourceConfig {
    
    @Bean
    @Primary
    @ConfigurationProperties("spring.datasource.primary")
    public DataSource primaryDataSource() {
        return DataSourceBuilder.create().build();
    }
    
    @Bean
    @ConfigurationProperties("spring.datasource.secondary")
    public DataSource secondaryDataSource() {
        return DataSourceBuilder.create().build();
    }
    
    @Bean
    @Primary
    public PlatformTransactionManager primaryTransactionManager() {
        return new DataSourceTransactionManager(primaryDataSource());
    }
    
    @Bean
    public PlatformTransactionManager secondaryTransactionManager() {
        return new DataSourceTransactionManager(secondaryDataSource());
    }
}
```

使用`@Transactional`时指定事务管理器：
```java
@Service
public class OrderService {
    @Transactional("primaryTransactionManager")
    public void createOrder() {
        // 使用主库事务
    }
    
    @Transactional("secondaryTransactionManager")
    public void syncOrder() {
        // 使用从库事务
    }
}
```

---

*此文原创，转载请注明出处。*
