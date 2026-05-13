# MySQL事务隔离级别深度解析：从ACID到MVCC的完整旅程

**文章标签：** #mysql #事务 #隔离级别 #mvcc #readview #undolog #锁机制 #面试必备

## 目录

- [引言：事务为何是数据库的基石](#引言事务为何是数据库的基石)
- [理论基础：ACID的实现原理](#理论基础acid的实现原理)
- [源码深度分析：事务启动与提交流程](#源码深度分析事务启动与提交流程)
- [并发问题：脏读、不可重复读、幻读](#并发问题脏读不可重复读幻读)
- [四种隔离级别详解](#四种隔离级别详解)
- [MVCC多版本并发控制机制](#mvcc多版本并发控制机制)
- [Read View可见性判断深度剖析](#read-view可见性判断深度剖析)
- [Undo Log版本链与Purge机制](#undo-log版本链与purge机制)
- [快照读与当前读的区别](#快照读与当前读的区别)
- [锁与隔离级别的关系](#锁与隔离级别的关系)
- [实战案例：幻读问题排查与解决](#实战案例幻读问题排查与解决)
- [对比分析：RC vs RR的业务影响](#对比分析rc-vs-rr的业务影响)
- [性能分析：长事务的危害与监控](#性能分析长事务的危害与监控)
- [常见陷阱与最佳实践](#常见陷阱与最佳实践)
- [面试题与参考答案](#面试题与参考答案)

---

## 引言：事务为何是数据库的基石

事务（Transaction）是数据库区别于文件系统的核心特性之一。它将一组操作封装成一个原子单元，要么全部成功，要么全部失败，确保了数据的完整性和一致性。

**核心认知**：

```
事务的本质：将多个操作打包成一个"原子操作"

没有事务：
- 转账操作：扣款成功，加款失败 → 钱消失了
- 订单系统：库存扣减成功，订单创建失败 → 超卖

有事务：
- 扣款和加款要么都成功，要么都失败
- 保证数据始终处于一致状态
```

**关键洞察**：理解事务隔离级别和MVCC机制，是排查并发问题、优化数据库性能的关键。

---

## 理论基础：ACID的实现原理

### 1. ACID概述

```
ACID四大特性：
┌─────────────────────────────────────────────────┐
│ A - Atomicity（原子性）                          │
│     事务是不可分割的最小执行单元                   │
│     实现：Undo Log                                │
│                                                  │
│ C - Consistency（一致性）                        │
│     事务执行前后，数据处于一致状态                 │
│     实现：约束 + 触发器 + 存储过程                 │
│                                                  │
│ I - Isolation（隔离性）                          │
│     并发事务之间相互隔离                           │
│     实现：MVCC + 锁                               │
│                                                  │
│ D - Durability（持久性）                         │
│     事务提交后，数据永久保存                       │
│     实现：Redo Log + Doublewrite                  │
└─────────────────────────────────────────────────┘
```

### 2. ACID实现机制详解

| 特性 | 实现机制 | 源码位置 | 作用 |
|------|---------|---------|------|
| Atomicity | Undo Log | `storage/innobase/roll/roll0undo.cc` | 记录修改前的数据，用于回滚 |
| Consistency | 约束检查 | Server层 | 外键、CHECK、NOT NULL等 |
| Isolation | MVCC + 锁 | `storage/innobase/lock/lock0lock.cc` | 多版本并发控制 + 锁机制 |
| Durability | Redo Log + Doublewrite | `storage/innobase/log/log0log.cc` | 崩溃恢复 |

### 3. 事务的基本操作

```sql
-- 开启事务
BEGIN;
-- 或 START TRANSACTION;
-- 或 SET autocommit = 0;

-- 执行SQL
INSERT INTO account (user_id, balance) VALUES (1, 1000);
UPDATE account SET balance = balance - 100 WHERE user_id = 1;
UPDATE account SET balance = balance + 100 WHERE user_id = 2;

-- 提交事务
COMMIT;

-- 回滚事务（出错时）
ROLLBACK;

-- 保存点
SAVEPOINT sp1;
-- ... 执行一些操作
ROLLBACK TO sp1;  -- 回滚到保存点
COMMIT;
```

---

## 源码深度分析：事务启动与提交流程

### 1. 事务启动源码

```cpp
// storage/innobase/trx/trx0trx.cc

/* 事务启动 */
static void trx_start_low(trx_t *trx, bool read_write) {
    // 1. 分配回滚段
    trx_assign_rseg(trx);
    
    // 2. 分配事务ID
    trx->id = trx_get_new_trx_id();
    
    // 3. 设置事务状态为ACTIVE
    trx->state = TRX_STATE_ACTIVE;
    
    // 4. 初始化Read View（RR隔离级别）
    if (trx->isolation_level == TRX_ISO_REPEATABLE_READ) {
        trx->read_view = read_view_open_now(trx->id, ...);
    }
    
    // 5. 记录开始时间
    trx->start_time = ut_time();
}

/* 事务ID分配 */
trx_id_t trx_get_new_trx_id(void) {
    // 全局事务ID计数器，原子递增
    return trx_sys->max_trx_id++;
}
```

### 2. 事务提交源码

```cpp
// storage/innobase/trx/trx0trx.cc

/* 事务提交 */
static void trx_commit(trx_t *trx) {
    // 1. 获取commit序号（LSN）
    trx_write_serialisation(trx);
    
    // 2. 写Redo Log（保证持久性）
    trx_commit_in_memory(trx);
    
    // 3. 释放锁
    lock_release(trx);
    
    // 4. 清理Read View
    if (trx->read_view != NULL) {
        read_view_close(trx->read_view);
        trx->read_view = NULL;
    }
    
    // 5. 设置事务状态为COMMITTED
    trx->state = TRX_STATE_COMMITTED;
    
    // 6. 标记Undo Log为可清理（Purge线程后续清理）
    trx_purge_add_update_undo_to_purge(trx);
}
```

### 3. 查看事务状态

```sql
-- 查看当前活跃事务
SELECT 
    trx_id,
    trx_state,
    trx_started,
    TIMESTAMPDIFF(SECOND, trx_started, NOW()) AS trx_seconds,
    trx_mysql_thread_id,
    LEFT(trx_query, 100) AS query
FROM information_schema.innodb_trx
ORDER BY trx_started;

-- 查看事务锁等待
SELECT 
    r.trx_id AS waiting_trx,
    r.trx_mysql_thread_id AS waiting_thread,
    b.trx_id AS blocking_trx,
    b.trx_mysql_thread_id AS blocking_thread,
    w.lock_id AS waiting_lock,
    b_lock.lock_id AS blocking_lock
FROM information_schema.innodb_lock_waits w
JOIN information_schema.innodb_trx b ON b.trx_id = w.blocking_trx_id
JOIN information_schema.innodb_trx r ON r.trx_id = w.requesting_trx_id
JOIN information_schema.innodb_locks b_lock ON b_lock.lock_id = w.blocking_lock_id
JOIN information_schema.innodb_locks r_lock ON r_lock.lock_id = w.requesting_lock_id;

-- MySQL 8.0+推荐使用performance_schema
SELECT 
    OBJECT_SCHEMA,
    OBJECT_NAME,
    THREAD_ID,
    OBJECT_TYPE,
    LOCK_TYPE,
    LOCK_STATUS,
    PROCESSLIST_ID,
    PROCESSLIST_INFO
FROM performance_schema.data_locks dl
JOIN performance_schema.threads t ON dl.THREAD_ID = t.THREAD_ID
WHERE LOCK_STATUS = 'WAITING';
```

---

## 并发问题：脏读、不可重复读、幻读

### 1. 脏读（Dirty Read）

**定义**：读到其他事务未提交的数据。

```sql
-- 事务A
BEGIN;
UPDATE account SET balance = 2000 WHERE id = 1;  -- 未提交

-- 事务B（并发）
SELECT balance FROM account WHERE id = 1;  -- 读取到2000（脏读！）

-- 事务A
ROLLBACK;  -- balance恢复1000，但B已经读到了2000

-- 结果：B读到了不存在的数据（脏数据）
```

**危害**：最严重，因为未提交的数据可能回滚，导致业务逻辑错误。

### 2. 不可重复读（Non-Repeatable Read）

**定义**：同一事务内两次读取同一行，数据被其他事务修改并提交。

```sql
-- 事务A
BEGIN;
SELECT balance FROM account WHERE id = 1;  -- 1000

-- 事务B（并发）
UPDATE account SET balance = 2000 WHERE id = 1;
COMMIT;

-- 事务A
SELECT balance FROM account WHERE id = 1;  -- 2000（不可重复读！）
-- 同一事务内，两次读取结果不同

COMMIT;
```

**危害**：影响业务逻辑的一致性，如基于第一次读取做判断，第二次读取时条件已变。

### 3. 幻读（Phantom Read）

**定义**：同一事务内两次查询同一范围，结果集行数不同（新增或删除）。

```sql
-- 事务A
BEGIN;
SELECT * FROM account WHERE balance > 1000;  -- 10条

-- 事务B（并发）
INSERT INTO account VALUES (11, 2000);
COMMIT;

-- 事务A
SELECT * FROM account WHERE balance > 1000;  -- 11条（幻读！）
-- 同一事务内，多了"幻影"行

COMMIT;
```

**危害**：影响范围查询的一致性，如统计、分页等场景。

### 4. 三种并发问题对比

```
┌─────────────────┬─────────────────┬─────────────────┬─────────────────┐
│     问题        │     脏读        │   不可重复读     │     幻读        │
├─────────────────┼─────────────────┼─────────────────┼─────────────────┤
│ 读取的数据      │ 未提交的        │ 已提交的        │ 新插入的        │
│ 影响范围        │ 单行            │ 单行            │ 范围查询        │
│ 严重程度        │ 最严重          │ 中等            │ 较轻            │
│ RC解决？        │      是         │      否         │      否         │
│ RR解决？        │      是         │      是         │  部分解决       │
└─────────────────┴─────────────────┴─────────────────┴─────────────────┘
```

---

## 四种隔离级别详解

### 1. 隔离级别概述

```sql
-- 查看当前隔离级别
SELECT @@transaction_isolation;
-- 或
SHOW VARIABLES LIKE 'transaction_isolation';

-- 设置隔离级别
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
SET GLOBAL TRANSACTION ISOLATION LEVEL REPEATABLE READ;
```

| 隔离级别 | 脏读 | 不可重复读 | 幻读 | 实现方式 | 适用场景 |
|---------|------|-----------|------|---------|---------|
| READ UNCOMMITTED | ✓ | ✓ | ✓ | 直接读最新数据 | 极少使用 |
| READ COMMITTED | ✗ | ✓ | ✓ | 每次SELECT生成新Read View | Oracle默认，适合需要最新数据的场景 |
| REPEATABLE READ | ✗ | ✗ | ✓ | 事务开始时生成Read View | MySQL默认，适合需要事务内一致性的场景 |
| SERIALIZABLE | ✗ | ✗ | ✗ | 所有SELECT加共享锁 | 严格串行化，性能差 |

### 2. 各隔离级别的实现机制

```
READ UNCOMMITTED：
├── 实现：不加锁，直接读最新数据（包括未提交的）
├── 性能：最高（无锁开销）
└── 问题：可能脏读、不可重复读、幻读

READ COMMITTED：
├── 快照读：每次SELECT生成新Read View
│   └── 能读到其他事务已提交的最新数据
├── 当前读：只锁符合条件的行（Record Lock），无Gap Lock
└── 问题：不可重复读、幻读

REPEATABLE READ：
├── 快照读：事务第一次SELECT时生成Read View，后续复用
│   └── 保证事务内看到的数据一致
├── 当前读：Next-Key Lock（Record Lock + Gap Lock）
│   └── 防止幻读（但极端情况下仍可能发生）
└── 问题：幻读（当前读场景）

SERIALIZABLE：
├── 实现：所有SELECT加共享锁（S Lock）
├── 效果：完全串行化，没有并发
└── 性能：最差
```

### 3. 隔离级别的锁行为

```sql
-- READ UNCOMMITTED：不加锁，直接读最新数据（可能脏读）

-- READ COMMITTED：
--   - 快照读：每次SELECT生成新Read View
--   - 当前读：只锁符合条件的行（Record Lock），无Gap Lock
BEGIN;
SELECT * FROM users WHERE id = 5 FOR UPDATE;
-- 只锁id=5的记录

-- REPEATABLE READ：
--   - 快照读：事务开始时生成Read View，一直复用
--   - 当前读：Next-Key Lock（Record Lock + Gap Lock）
BEGIN;
SELECT * FROM users WHERE id = 5 FOR UPDATE;
-- 如果id=5存在：只锁id=5的记录（退化）
-- 如果id=5不存在：锁(前一条, 后一条)间隙

-- SERIALIZABLE：
--   - 所有SELECT加共享锁
BEGIN;
SELECT * FROM users WHERE id = 5;
-- 自动加S锁，其他事务不能修改id=5
```

---

## MVCC多版本并发控制机制

### 1. MVCC核心思想

**读不加锁，读写不冲突**。通过保存数据的历史版本，让读操作可以读取到一致性的数据快照。

```
MVCC的核心价值：
┌─────────────────────────────────────────┐
│ 无锁读（快照读）                         │
│ - SELECT不加锁                           │
│ - 不阻塞其他事务的写操作                  │
│ - 提高并发性能                           │
│                                          │
│ 一致性视图                               │
│ - 每个事务看到的数据版本可能不同           │
│ - 基于Read View判断数据可见性             │
│                                          │
│ 版本链                                   │
│ - 通过Undo Log保存历史版本                 │
│ - 需要时回溯到合适版本                     │
└─────────────────────────────────────────┘
```

### 2. 隐藏字段

InnoDB每行记录有3个隐藏字段：

```cpp
// storage/innobase/include/dict0mem.h
/* 隐藏字段定义 */
#define DATA_ROW_ID     0   /* 6字节：隐藏主键 */
#define DATA_TRX_ID     1   /* 6字节：最后修改该行的事务ID */
#define DATA_ROLL_PTR   2   /* 7字节：回滚指针，指向Undo Log记录 */
```

| 字段 | 长度 | 说明 |
|------|------|------|
| DB_ROW_ID | 6字节 | 隐藏主键（无显式主键时） |
| DB_TRX_ID | 6字节 | 最后修改该行的事务ID |
| DB_ROLL_PTR | 7字节 | 回滚指针，指向Undo Log记录 |

```
行记录结构：
┌─────────────┬─────────────┬─────────────┬──────────┐
│ DB_ROW_ID   │ DB_TRX_ID   │ DB_ROLL_PTR │ 列数据   │
│ 6字节       │ 6字节       │ 7字节       │          │
└─────────────┴─────────────┴─────────────┴──────────┘
```

### 3. Undo Log版本链

```
记录当前值: DB_TRX_ID=100, name='Alice_v3', DB_ROLL_PTR -> Undo Log

Undo Log记录: DB_TRX_ID=50, name='Alice_v2', DB_ROLL_PTR -> Undo Log
Undo Log记录: DB_TRX_ID=20, name='Alice_v1', DB_ROLL_PTR = NULL

形成版本链：当前值 -> v2 -> v1

事务查找时：
1. 查看当前记录的DB_TRX_ID
2. 用Read View判断该版本是否可见
3. 如果不可见，沿DB_ROLL_PTR找到Undo Log中的上一个版本
4. 重复步骤2-3，直到找到可见版本
```

```cpp
// storage/innobase/roll/roll0undo.cc
/* Undo Log记录结构 */
struct undo_rec_t {
    ulint   type;           // INSERT/UPDATE
    ulint   table_id;       // 表ID
    trx_id_t trx_id;        // 事务ID
    roll_ptr_t roll_ptr;    // 上一个Undo Log指针
    byte    old_cols[];     // 修改前的列值
};
```

### 4. Undo Log类型

| 类型 | 作用 | 清理时机 | 特点 |
|------|------|---------|------|
| INSERT Undo Log | 插入操作的回滚 | 事务提交后立即删除 | 不需要维护版本链 |
| UPDATE Undo Log | 更新操作的回滚 | 无事务引用该版本后，Purge线程清理 | 需要维护版本链供MVCC使用 |

```sql
-- 查看Undo Log大小
SELECT 
    tablespace_name,
    file_name,
    ROUND(total_extents * extent_size / 1024 / 1024, 2) AS size_mb
FROM information_schema.files
WHERE file_name LIKE '%undo%';

-- MySQL 8.0+独立Undo表空间
SHOW VARIABLES LIKE 'innodb_undo_tablespaces';  -- 默认2个
SHOW VARIABLES LIKE 'innodb_max_undo_log_size'; -- 最大Undo日志大小
```

---

## Read View可见性判断深度剖析

### 1. Read View的字段

```cpp
// storage/innobase/read/read0read.cc
/* Read View结构 */
struct ReadView {
    trx_id_t m_low_limit_id;      // max_trx_id：生成ReadView时分配的下一个事务ID
    trx_id_t m_up_limit_id;       // min_trx_id：m_ids中最小的事务ID
    trx_id_t m_creator_trx_id;    // 创建该Read View的事务ID
    ids_t m_ids;                  // 生成ReadView时活跃的事务ID列表
};
```

| 字段 | 说明 |
|------|------|
| m_low_limit_id | 生成ReadView时系统分配的下一个事务ID（max_trx_id） |
| m_up_limit_id | 生成ReadView时活跃事务列表中最小的事务ID（min_trx_id） |
| m_creator_trx_id | 创建该ReadView的事务ID |
| m_ids | 生成ReadView时所有活跃（未提交）事务的ID列表 |

### 2. 可见性判断规则

对于某行数据的DB_TRX_ID：

```cpp
// storage/innobase/read/read0read.cc - changes_visible()
bool changes_visible(trx_id_t id, const table_name_t& name) const {
    if (id < m_up_limit_id || id == m_creator_trx_id) {
        // DB_TRX_ID < min_trx_id：在ReadView生成前已提交，可见
        // DB_TRX_ID == creator_trx_id：当前事务修改的，可见
        return true;
    }
    if (id >= m_low_limit_id) {
        // DB_TRX_ID >= max_trx_id：在ReadView生成后开始的，不可见
        return false;
    }
    // min_trx_id <= DB_TRX_ID < max_trx_id
    // 检查是否在m_ids列表中
    return !m_ids.find(id);  // 在列表中：未提交，不可见；不在：已提交，可见
}
```

```
可见性判断流程图：

              DB_TRX_ID == creator_trx_id?
                   /        \
                 是         否
                  |          |
               可见    DB_TRX_ID < min_trx_id?
                          /      \
                        是        否
                         |         |
                      可见   DB_TRX_ID >= max_trx_id?
                                 /      \
                               是        否
                                |         |
                             不可见  DB_TRX_ID在m_ids中?
                                      /      \
                                    是        否
                                     |         |
                                  不可见     可见

总结：
- 当前事务修改的：可见
- ReadView生成前已提交的：可见
- ReadView生成后开始的：不可见
- ReadView生成时活跃的：不可见（未提交）
```

### 3. RC vs RR的Read View差异

```sql
-- READ COMMITTED：每次SELECT生成新的Read View
-- 事务A
BEGIN;
-- T1: SELECT * FROM users WHERE id = 1;  -- 生成Read View RV1

-- 事务B更新并提交
UPDATE users SET name = 'Bob' WHERE id = 1;
COMMIT;

-- T2: SELECT * FROM users WHERE id = 1;  -- 生成Read View RV2
-- 读到B提交后的最新数据（name='Bob'）

-- 特点：
-- - 每次SELECT都能看到其他事务已提交的最新数据
-- - 不可重复读：同一事务内两次读取可能不同
```

```sql
-- REPEATABLE READ：事务第一次SELECT时生成Read View，后续复用
-- 事务A
BEGIN;
-- T1: SELECT * FROM users WHERE id = 1;  -- 生成Read View RV1

-- 事务B更新并提交
UPDATE users SET name = 'Bob' WHERE id = 1;
COMMIT;

-- T2: SELECT * FROM users WHERE id = 1;  -- 复用RV1
-- 读到T1时的快照（name还是原来的值）

-- 特点：
-- - 事务内所有SELECT使用同一个Read View
-- - 保证可重复读：同一事务内两次读取结果相同
-- - 但当前读（FOR UPDATE）可能读到最新数据
```

### 4. 可见性判断示例

```sql
-- 场景：
-- 事务10：已提交，将name改为'Alice_v1'
-- 事务20：已提交，将name改为'Alice_v2'
-- 事务50：活跃中（未提交），将name改为'Alice_v3'
-- 事务100：生成Read View

-- Read View：
-- creator_trx_id = 100
-- m_ids = [50]        -- 活跃事务
-- min_trx_id = 50     -- m_ids中最小
-- max_trx_id = 101    -- 下一个分配的事务ID

-- 当前记录：DB_TRX_ID=50, name='Alice_v3'
-- 判断：50 == 100? 否
--      50 < 50? 否
--      50 >= 101? 否
//      50在m_ids中? 是 -> 不可见

-- 沿Undo Log找到DB_TRX_ID=20的记录：
-- 判断：20 < 50? 是 -> 可见
-- 结果：读到name='Alice_v2'

-- 如果事务50也提交了：
-- m_ids = []（空）
-- 重新判断：50在m_ids中? 否 -> 可见
-- 结果：读到name='Alice_v3'
```

---

## Undo Log版本链与Purge机制

### 1. Undo Log存储结构

```
Undo Tablespace（MySQL 5.7+独立）
├── undo_001
└── undo_002
    ├── Rollback Segment（回滚段，默认128个）
    │   └── Undo Slot（1024个slot）
    │       └── Undo Segment（段）
    │           └── Extent（区）
    │               └── Page（页）
    │                   └── Undo Log记录
    │                       ├── type: INSERT/UPDATE
    │                       ├── trx_id: 事务ID
    │                       ├── roll_ptr: 上一个Undo Log指针
    │                       └── old_cols: 修改前的列值
```

源码位置：`storage/innobase/roll/roll0undo.cc`

### 2. Purge机制

```sql
-- Purge线程清理不再需要的Undo Log
-- 何时可以清理？
-- 当没有事务需要访问某个历史版本时

-- 查看Purge状态
SHOW ENGINE INNODB STATUS\G
-- TRANSACTIONS部分：
-- Purge done for trx's n:o < 5000 undo n:o < 1000
-- 表示事务ID<5000的Undo Log已清理

-- 查看Undo Log历史列表长度
SHOW STATUS LIKE 'Innodb_history_list_length';
-- 值越大，说明Undo Log积累越多，需要关注的Purge延迟

-- 查看Purge线程配置
SHOW VARIABLES LIKE 'innodb_purge_threads';       -- 默认4个
SHOW VARIABLES LIKE 'innodb_purge_batch_size';    -- 每批清理大小
```

```cpp
// storage/innobase/trx/trx0purge.cc
/* Purge线程 */
static void trx_purge(void) {
    // 1. 获取当前最老的Read View
    ReadView* oldest_view = trx_sys->oldest_view;
    
    // 2. 计算可以清理的Undo Log范围
    trx_id_t purge_trx_id = oldest_view->m_up_limit_id;
    
    // 3. 批量清理Undo Log
    for (undo_rec_t *undo = undo_list; undo != NULL; ) {
        if (undo->trx_id < purge_trx_id) {
            // 该Undo Log不再被任何事务需要，可以清理
            undo_rec_t *next = undo->next;
            undo_page_free(undo);
            undo = next;
        } else {
            // 还有事务可能需要该版本，保留
            undo = undo->next;
        }
    }
}
```

### 3. 长事务导致Undo Log膨胀

```sql
-- 长事务示例
BEGIN;
SELECT * FROM users WHERE id = 1;  -- T1时刻生成Read View

-- 其他事务大量更新...
-- UPDATE users SET ... WHERE ...;（执行100万次）

-- 1小时后
SELECT * FROM users WHERE id = 1;  -- 仍用T1的Read View
COMMIT;

-- 问题：
-- 1. Undo Log无法清理（长事务可能用到历史版本）
-- 2. Undo Tablespace膨胀
// 3. 查询性能下降（需要遍历很长的版本链）
// 4. 其他事务的更新被阻塞（如果Undo空间满）
```

```sql
-- 监控长事务
SELECT 
    trx_id,
    trx_state,
    trx_started,
    TIMESTAMPDIFF(SECOND, trx_started, NOW()) AS trx_seconds,
    LEFT(trx_query, 100) AS query
FROM information_schema.innodb_trx
WHERE TIMESTAMPDIFF(SECOND, trx_started, NOW()) > 60
ORDER BY trx_started;

-- 设置长事务报警阈值
SET GLOBAL innodb_max_undo_log_size = 1073741824;  -- 1GB

-- 查看Undo Tablespace使用情况
SELECT 
    tablespace_name,
    file_name,
    ROUND(total_extents * extent_size / 1024 / 1024, 2) AS size_mb
FROM information_schema.files
WHERE file_name LIKE '%undo%';
```

---

## 快照读与当前读的区别

### 1. 快照读（Snapshot Read）

```sql
-- 快照读：基于MVCC，不加锁
SELECT * FROM users WHERE id = 1;
SELECT * FROM users WHERE age > 20;

-- 原理：
-- 1. 生成Read View（RC每次生成，RR事务开始时生成）
// 2. 根据Read View判断数据可见性
-- 3. 读取合适的版本

-- 特点：
-- - 不加锁，不阻塞其他事务
-- - 可能读到历史版本（非最新）
-- - 并发性能高
```

### 2. 当前读（Current Read）

```sql
-- 当前读：读取最新数据，加锁
SELECT * FROM users WHERE id = 1 FOR UPDATE;     -- X锁（排他锁）
SELECT * FROM users WHERE id = 1 LOCK IN SHARE MODE;  -- S锁（共享锁）

INSERT INTO users ...;   -- 自动加X锁
UPDATE users SET ...;    -- 自动加X锁
DELETE FROM users ...;   -- 自动加X锁

-- 原理：
-- 1. 不加Read View，直接读最新数据
// 2. 对读取的记录加锁（Record Lock / Gap Lock / Next-Key Lock）

-- 特点：
-- - 读取最新数据
-- - 加锁，可能阻塞其他事务
-- - 保证数据一致性
```

### 3. 快照读与当前读的对比

```
┌─────────────────┬─────────────────┬─────────────────┐
│     特性        │     快照读       │     当前读      │
├─────────────────┼─────────────────┼─────────────────┤
│ SQL示例         │ SELECT          │ SELECT ... FOR UPDATE |
│ 读取数据        │ 历史版本         │ 最新版本        │
│ 是否加锁        │ 不加锁           │ 加锁            │
│ 实现机制        │ MVCC + Read View │ 锁机制          │
│ 并发性能        │ 高               │ 低（锁竞争）     │
│ 一致性          │ 事务内一致       │ 全局一致        │
│ 适用场景        │ 读多写少         │ 需要最新数据     │
└─────────────────┴─────────────────┴─────────────────┘
```

### 4. 幻读的两种理解

```sql
-- 快照读下的幻读（RR已解决）
-- 事务A
BEGIN;
SELECT * FROM users WHERE age > 20;  -- 10条（快照读）

-- 事务B插入age=25的记录并提交

-- 事务A
SELECT * FROM users WHERE age > 20;  -- 仍是10条（RR复用Read View）
-- RR通过MVCC解决了快照读的幻读

-- 当前读下的幻读（RR未完全解决）
-- 事务A
BEGIN;
SELECT * FROM users WHERE age > 20 FOR UPDATE;  -- 10条（当前读，加Next-Key Lock）

-- 事务B插入age=25的记录
-- 如果B的插入位置不在A的锁范围内，可能成功

-- 事务A
SELECT * FROM users WHERE age > 20 FOR UPDATE;  -- 可能11条！
-- 但InnoDB的Next-Key Lock通常能防止（锁住间隙）
```

---

## 锁与隔离级别的关系

### 1. 锁的类型

```
InnoDB锁类型：
┌─────────────────────────────────────────┐
│ 按粒度分类：                              │
│ - 行级锁（Record Lock）                   │
│ - 间隙锁（Gap Lock）                      │
│ - 临键锁（Next-Key Lock）                │
│ - 插入意向锁（Insert Intention Lock）     │
│ - 自增锁（Auto-inc Lock）                 │
│                                          │
│ 按模式分类：                              │
│ - 共享锁（S Lock / 读锁）                 │
│ - 排他锁（X Lock / 写锁）                 │
│                                          │
│ 按使用方式：                              │
│ - 显式锁：SELECT ... FOR UPDATE          │
│ - 隐式锁：UPDATE/DELETE/INSERT自动加锁   │
└─────────────────────────────────────────┘
```

### 2. 不同隔离级别的锁行为

```sql
-- READ COMMITTED
-- 当前读只加Record Lock（行锁），无Gap Lock
BEGIN;
SELECT * FROM users WHERE id = 5 FOR UPDATE;
-- 只锁id=5的记录
-- 如果id=5不存在，不锁任何记录

-- REPEATABLE READ
-- 当前读加Next-Key Lock（Record Lock + Gap Lock）
BEGIN;
SELECT * FROM users WHERE id = 5 FOR UPDATE;
-- 如果id=5存在：只锁id=5的记录（退化）
-- 如果id=5不存在：锁(前一条, 后一条)间隙

-- SERIALIZABLE
-- 所有SELECT加共享锁
BEGIN;
SELECT * FROM users WHERE id = 5;
-- 自动加S锁，其他事务不能修改id=5
```

### 3. Gap Lock（间隙锁）

```sql
-- 数据：id = [1, 5, 10, 15]

BEGIN;
SELECT * FROM users WHERE id > 5 AND id < 15 FOR UPDATE;
-- Next-Key Lock锁住：
-- (5, 10]：id=10的记录 + (5,10)间隙
-- (10, 15]：id=10已锁，(10,15)间隙

-- 其他事务：
INSERT INTO users VALUES (7, ...);  -- 阻塞！Gap Lock锁住(5,10)间隙
INSERT INTO users VALUES (12, ...); -- 阻塞！Gap Lock锁住(10,15)间隙
INSERT INTO users VALUES (16, ...); -- 不阻塞，不在锁范围内
INSERT INTO users VALUES (3, ...);  -- 不阻塞，不在锁范围内
```

### 4. 锁可视化

```sql
-- 查看当前锁（MySQL 8.0+）
SELECT 
    OBJECT_SCHEMA,
    OBJECT_NAME,
    THREAD_ID,
    OBJECT_TYPE,
    LOCK_TYPE,
    LOCK_STATUS,
    PROCESSLIST_ID,
    PROCESSLIST_INFO
FROM performance_schema.data_locks dl
JOIN performance_schema.threads t ON dl.THREAD_ID = t.THREAD_ID
WHERE OBJECT_TYPE = 'TABLE' OR LOCK_STATUS = 'WAITING';

-- 查看锁等待
SELECT 
    REQUESTING_ENGINE_TRANSACTION_ID AS requesting_trx,
    BLOCKING_ENGINE_TRANSACTION_ID AS blocking_trx,
    OBJECT_SCHEMA,
    OBJECT_NAME,
    LOCK_TYPE AS waiting_lock,
    LOCK_TYPE AS blocking_lock
FROM performance_schema.data_lock_waits
JOIN performance_schema.data_locks dl1 ON dl1.ENGINE_LOCK_ID = REQUESTING_ENGINE_LOCK_ID
JOIN performance_schema.data_locks dl2 ON dl2.ENGINE_LOCK_ID = BLOCKING_ENGINE_LOCK_ID;
```

---

## 实战案例：幻读问题排查与解决

### 案例1：财务系统重复扣款

**背景**：某财务系统，RR隔离级别下出现"重复扣款"问题。

```sql
-- 业务逻辑：检查余额，扣款
BEGIN;
SELECT balance FROM account WHERE user_id = 100 FOR UPDATE;
-- 查到balance = 1000

-- 检查是否足够...
UPDATE account SET balance = balance - 500 WHERE user_id = 100;
COMMIT;
```

**问题现象**：
并发执行时，同一user_id被扣款两次，余额变成0（应剩余500）。

**问题分析**：

```sql
-- 事务A
BEGIN;
SELECT balance FROM account WHERE user_id = 100 FOR UPDATE;
-- 假设user_id=100不存在（新用户）
-- 由于user_id有唯一索引，记录不存在时：
-- Gap Lock锁住(前一条user_id, 后一条user_id)间隙

-- 事务B
BEGIN;
SELECT balance FROM account WHERE user_id = 100 FOR UPDATE;
-- 同样锁住相同间隙（Gap Lock兼容，多个事务可同时持有）

-- 事务A
INSERT INTO account (user_id, balance) VALUES (100, 1000);
-- 成功

-- 事务B
INSERT INTO account (user_id, balance) VALUES (100, 1000);
-- 死锁！或如果A已提交，B的INSERT被阻塞，但SELECT时没读到A的插入
```

**根因**：
RR隔离级别下，`SELECT ... FOR UPDATE`对不存在的记录加Gap Lock，但Gap Lock之间兼容（多个事务可同时持有同一间隙的Gap Lock），导致都能INSERT成功。

**解决方案**：

```sql
-- 方案1：使用唯一索引 + INSERT ... ON DUPLICATE KEY UPDATE
INSERT INTO account (user_id, balance) VALUES (100, 1000)
ON DUPLICATE KEY UPDATE balance = balance - 500;
-- 利用唯一索引的排他性，防止并发插入

-- 方案2：使用SERIALIZABLE（性能差）
SET SESSION TRANSACTION ISOLATION LEVEL SERIALIZABLE;

-- 方案3：应用层分布式锁（推荐）
-- 用Redis SETNX加锁
SETNX lock:user:100 1 EX 10
-- 执行业务逻辑...
DEL lock:user:100

-- 方案4：先INSERT再UPDATE
BEGIN;
INSERT IGNORE INTO account (user_id, balance) VALUES (100, 1000);
-- 如果已存在，忽略；如果不存在，插入
UPDATE account SET balance = balance - 500 WHERE user_id = 100;
COMMIT;
```

### 案例2：库存超卖问题

```sql
-- 场景：秒杀活动，库存扣减

-- 错误写法（RR下可能超卖）
BEGIN;
SELECT stock FROM products WHERE id = 100 FOR UPDATE;
-- stock = 10
-- 检查stock > 0...
UPDATE products SET stock = stock - 1 WHERE id = 100;
COMMIT;

-- 问题：
-- 如果多个事务同时执行，可能都读到stock=10
-- 然后都UPDATE成功，导致超卖

-- 正确写法1：UPDATE时判断库存
UPDATE products SET stock = stock - 1 
WHERE id = 100 AND stock > 0;
-- 利用行锁和原子性，只有一个事务能成功

-- 正确写法2：使用乐观锁
-- 增加version字段
UPDATE products SET stock = stock - 1, version = version + 1
WHERE id = 100 AND version = 5;
-- 如果version已被修改，UPDATE影响行数为0

-- 正确写法3：使用数据库排队（消息队列）
-- 将秒杀请求放入队列，单线程处理
```

---

## 对比分析：RC vs RR的业务影响

### 1. RC（Read Committed）特点

```sql
-- RC隔离级别
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;

BEGIN;
SELECT balance FROM account WHERE id = 1;  -- 1000

-- 事务B更新为2000并提交

SELECT balance FROM account WHERE id = 1;  -- 2000（读到最新）

COMMIT;
```

**适用场景**：
- 需要看到最新提交数据的场景
- 报表查询（需要实时数据）
- Oracle默认隔离级别

**优点**：
- 减少锁竞争（无Gap Lock）
- 减少死锁
- 更好的并发性能

**缺点**：
- 不可重复读
- 幻读

### 2. RR（Repeatable Read）特点

```sql
-- RR隔离级别（MySQL默认）
SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ;

BEGIN;
SELECT balance FROM account WHERE id = 1;  -- 1000

-- 事务B更新为2000并提交

SELECT balance FROM account WHERE id = 1;  -- 1000（快照读，不变）

COMMIT;
```

**适用场景**：
- 需要事务内数据一致性的场景
- 银行转账（事务内余额不变）
- 统计计算（事务内数据稳定）

**优点**：
- 可重复读
- 幻读部分解决（快照读）

**缺点**：
- 更多锁竞争（Gap Lock）
- 更多死锁
- Undo Log膨胀风险

### 3. 业务选择建议

```
选择RC的场景：
- 读多写少
- 需要实时数据（如报表）
- 并发要求高
- 能接受不可重复读

选择RR的场景：
- 写多读少
- 需要事务内一致性（如金融）
- 复杂事务逻辑
- 不能接受不可重复读

MySQL 8.0+建议：
- 大多数业务可用RC（配合Row格式binlog）
- 金融等强一致性场景用RR
```

---

## 性能分析：长事务的危害与监控

### 1. 长事务的危害

```
长事务的危害：
┌─────────────────────────────────────────┐
│ 1. Undo Log膨胀                          │
│    - 无法清理历史版本                     │
│    - Undo Tablespace持续增长              │
│    - 可能导致磁盘满                       │
│                                          │
│ 2. 锁长时间持有                          │
│    - 阻塞其他事务                         │
│    - 导致锁等待和死锁                     │
│    - 系统吞吐量下降                       │
│                                          │
│ 3. 查询性能下降                          │
│    - 需要遍历很长的版本链                  │
│    - 每次查询都要判断可见性                │
│    - CPU使用率上升                        │
│                                          │
│ 4. 主从延迟                              │
│    - 长事务的binlog提交延迟               │
│    - 从库数据滞后                         │
│                                          │
│ 5. 连接池耗尽                            │
│    - 长事务占用连接不释放                  │
│    - 新请求无法获取连接                   │
└─────────────────────────────────────────┘
```

### 2. 监控长事务

```sql
-- 查看活跃事务（按持续时间排序）
SELECT 
    trx_id,
    trx_state,
    trx_started,
    TIMESTAMPDIFF(SECOND, trx_started, NOW()) AS trx_seconds,
    TIMESTAMPDIFF(MINUTE, trx_started, NOW()) AS trx_minutes,
    trx_mysql_thread_id,
    LEFT(trx_query, 200) AS query
FROM information_schema.innodb_trx
WHERE trx_state = 'RUNNING'
ORDER BY trx_started;

-- 查看长事务报警（超过60秒）
SELECT 
    trx_id,
    trx_started,
    TIMESTAMPDIFF(SECOND, trx_started, NOW()) AS trx_seconds,
    LEFT(trx_query, 100) AS query
FROM information_schema.innodb_trx
WHERE TIMESTAMPDIFF(SECOND, trx_started, NOW()) > 60
ORDER BY trx_seconds DESC;

-- 查看Undo Log大小
SHOW STATUS LIKE 'Innodb_history_list_length';
-- 值 > 10000 说明Undo Log积累严重

-- 查看Undo Tablespace大小
SELECT 
    file_name,
    ROUND(total_extents * extent_size / 1024 / 1024, 2) AS size_mb
FROM information_schema.files
WHERE file_name LIKE '%undo%'
ORDER BY size_mb DESC;
```

### 3. 长事务处理

```sql
-- 设置事务超时
SET GLOBAL innodb_lock_wait_timeout = 50;  -- 锁等待超时50秒
SET GLOBAL lock_wait_timeout = 60;         -- 元数据锁等待超时60秒

-- 设置事务最大执行时间（MySQL 8.0）
SET GLOBAL max_execution_time = 30000;     -- 30秒

-- 杀死长事务
SELECT CONCAT('KILL ', trx_mysql_thread_id, ';') AS kill_cmd
FROM information_schema.innodb_trx
WHERE TIMESTAMPDIFF(SECOND, trx_started, NOW()) > 300;
-- 执行生成的KILL命令
```

---

## 常见陷阱与最佳实践

### 陷阱1：认为REPEATABLE READ完全解决幻读

**问题**：RR下快照读确实解决幻读，但当前读（FOR UPDATE）可能遇到幻读。

```sql
-- RR下
BEGIN;
SELECT * FROM users WHERE age > 20;  -- 10条，快照读

-- B插入age=25并提交

SELECT * FROM users WHERE age > 20;  -- 仍是10条（OK）

SELECT * FROM users WHERE age > 20 FOR UPDATE;  -- 可能11条！
```

**最佳实践**：
- 需要完全避免幻读时，使用SERIALIZABLE或业务层幂等
- 理解快照读和当前读的区别
- INSERT前先检查是否存在，用唯一约束防重

### 陷阱2：长事务导致Undo Log膨胀

**问题**：事务中执行大量操作或长时间不提交，Undo Log无法清理。

```sql
-- 监控Undo Log大小
SHOW STATUS LIKE 'Innodb_history_list_length';
-- 值 > 10000 说明有问题

-- 查看Undo Tablespace大小
SELECT 
    file_name,
    ROUND(total_extents * extent_size / 1024 / 1024, 2) AS size_mb
FROM information_schema.files
WHERE file_name LIKE '%undo%';
```

**最佳实践**：
- 事务尽快提交，不要混用业务逻辑和RPC调用
- 设置超时：`innodb_lock_wait_timeout = 50`
- 监控长事务并报警
- 大事务拆分为多个小事务

### 陷阱3：滥用SERIALIZABLE隔离级别

**问题**：为了"绝对安全"使用SERIALIZABLE，导致所有查询加锁，并发度极低。

```sql
-- SERIALIZABLE下
SELECT * FROM users WHERE id = 1;
-- 自动加S锁，其他事务不能修改id=1

-- 性能对比（100并发）：
-- RC/RR：QPS 5000
// SERIALIZABLE：QPS 200（锁竞争严重）
```

**最佳实践**：
- 绝大多数业务RR或RC足够
- 需要严格串行化时，优先考虑乐观锁或分布式锁
- 不要为解决幻读而直接升级到SERIALIZABLE

### 陷阱4：不注意RC和RR的业务语义差异

**问题**：从Oracle（RC）迁移到MySQL（RR）后，不加注意地认为行为一致。

```sql
-- RC下
BEGIN;
SELECT balance FROM account WHERE id = 1;  -- 1000

-- B更新为2000并提交

SELECT balance FROM account WHERE id = 1;  -- 2000（读到最新）

-- RR下
BEGIN;
SELECT balance FROM account WHERE id = 1;  -- 1000

-- B更新为2000并提交

SELECT balance FROM account WHERE id = 1;  -- 1000（快照读，不变）
```

**最佳实践**：
- RC适合需要看到最新提交数据的场景
- RR适合需要事务内数据一致性的场景
- 根据业务需求选择，不是越严格越好

### 陷阱5：快照读和当前读混用导致不一致

**问题**：同一事务中先快照读后当前读，读到不同数据。

```sql
BEGIN;
SELECT balance FROM account WHERE id = 1;  -- 快照读：1000

-- B更新为2000并提交

SELECT balance FROM account WHERE id = 1 FOR UPDATE;  -- 当前读：2000

-- 同一事务中，balance先是1000，后是2000，逻辑混乱
```

**最佳实践**：
- 同一事务中统一使用快照读或当前读
- 如果必须用当前读，从一开始就使用FOR UPDATE

### 陷阱6：忽视Gap Lock导致的死锁

**问题**：RR下Gap Lock可能导致意外的死锁。

```sql
-- 数据：id = [1, 5, 10]

-- 事务A
BEGIN;
SELECT * FROM users WHERE id = 3 FOR UPDATE;
-- Gap Lock锁住(1, 5)间隙

-- 事务B
BEGIN;
SELECT * FROM users WHERE id = 4 FOR UPDATE;
-- Gap Lock也锁住(1, 5)间隙（兼容）

-- 事务A
INSERT INTO users VALUES (3, ...);
-- 等待B释放Gap Lock

-- 事务B
INSERT INTO users VALUES (4, ...);
-- 等待A释放Gap Lock

-- 死锁！
```

**最佳实践**：
- 理解Gap Lock的行为
- 尽量使用RC减少Gap Lock
- 应用层做好重试机制

---

## 面试题与参考答案

**Q1：脏读、不可重复读、幻读的区别？**

**A**：

- **脏读**：读到其他事务未提交的数据。最严重，因为数据可能回滚。
- **不可重复读**：同一事务内两次读取同一行，数据被其他事务修改并提交。
- **幻读**：同一事务内两次查询同一范围，结果集行数不同（新增或删除）。

严重程度：脏读 > 不可重复读 > 幻读

MySQL解决方式：
- 脏读：RC及以上级别（Read View过滤未提交数据）
- 不可重复读：RR级别（复用Read View）
- 幻读：RR快照读解决，当前读用Next-Key Lock解决

**Q2：MySQL默认隔离级别是什么？为什么？**

**A**：

MySQL默认**REPEATABLE READ**，Oracle默认READ COMMITTED。

MySQL选择RR的历史原因：
1. **早期复制需求**：MySQL 5.1前Statement格式binlog，需要RR保证主从一致性
2. **InnoDB设计**：MVCC天然支持RR，性能好
3. **兼容性**：很多应用依赖RR的语义

现代MySQL（5.7+，Row格式binlog）可以用RC，但有些遗留代码可能依赖RR。

**Q3：MVCC如何实现读不加锁？**

**A**：

**核心机制**：
1. **隐藏字段**：每行记录有DB_TRX_ID（事务ID）和DB_ROLL_PTR（回滚指针）
2. **Undo Log版本链**：修改数据时生成Undo Log，形成历史版本链
3. **Read View**：快照读时生成读视图，判断数据可见性

**流程**：

```
1. SELECT时生成Read View
2. 读取记录，查看DB_TRX_ID
3. 用Read View判断该版本是否可见
4. 如果不可见，沿DB_ROLL_PTR找到Undo Log中的上一个版本
5. 重复步骤3-4，直到找到可见版本
```

**Q4：Read View的可见性判断规则？**

**A**：

对于数据的DB_TRX_ID：
1. **DB_TRX_ID == creator_trx_id**：当前事务修改，可见
2. **DB_TRX_ID < min_trx_id**：Read View生成前已提交，可见
3. **DB_TRX_ID >= max_trx_id**：Read View生成后开始，不可见
4. **min_trx_id <= DB_TRX_ID < max_trx_id**：
   - 在m_ids列表中：未提交，不可见
   - 不在m_ids列表中：已提交，可见

**Q5：RC和RR在Read View上的区别？**

**A**：

- **RC**：每次SELECT都生成新的Read View，能读到其他事务已提交的最新数据
- **RR**：事务第一次SELECT时生成Read View，后续复用，保证可重复读

**示例**：

```sql
-- 事务A在T1时刻SELECT，生成Read View
-- 事务B在T2时刻提交更新

-- RC：T3时刻SELECT，生成新Read View，看到B的更新
// RR：T3时刻SELECT，复用T1的Read View，看不到B的更新
```

**Q6：Undo Log的作用是什么？为什么UPDATE Undo Log不能立即删除？**

**A**：

**作用**：
1. **事务回滚**：撤销未提交的修改
2. **MVCC**：提供历史版本数据供快照读

**清理时机**：
- **INSERT Undo Log**：事务提交后立即删除（没有事务需要插入前的状态）
- **UPDATE Undo Log**：需要提供历史版本，不能立即删除。只有当没有事务需要访问该历史版本时（即所有活跃的Read View都>该版本的事务ID），Purge线程才能清理。

**Q7：长事务有什么危害？**

**A**：

1. **Undo Log膨胀**：无法清理历史版本，Undo Tablespace持续增长
2. **锁长时间持有**：阻塞其他事务
3. **查询性能下降**：需要遍历很长的版本链
4. **主从延迟**：长事务的binlog提交延迟

**监控**：

```sql
SELECT * FROM information_schema.innodb_trx 
WHERE TIMESTAMPDIFF(SECOND, trx_started, NOW()) > 60;
```

**Q8：快照读和当前读有什么区别？**

**A**：

| 特性 | 快照读 | 当前读 |
|------|--------|--------|
| SQL | 普通SELECT | SELECT ... FOR UPDATE/LOCK IN SHARE MODE |
| 读取数据 | 历史版本（基于Read View） | 最新版本 |
| 是否加锁 | 不加锁 | 加锁 |
| 实现 | MVCC | 锁机制 |
| 用途 | 读多写少场景 | 需要数据最新且一致的场景 |

**Q9：InnoDB如何解决幻读？**

**A**：

**快照读**：通过MVCC + RR隔离级别解决。
- RR下事务复用Read View，其他事务插入的数据对新版本可见，但对当前事务的旧Read View不可见。

**当前读**：通过Next-Key Lock解决。
- Next-Key Lock = Record Lock + Gap Lock
- 锁住记录及其前面的间隙，阻止其他事务插入

**例外**：如果当前读的范围不包含新插入记录的位置，或锁退化，仍可能幻读。

**Q10：事务的ACID特性分别如何实现？**

**A**：

- **A（原子性）**：Undo Log。事务回滚时，根据Undo Log撤销所有修改。
- **C（一致性）**：数据库约束（外键、CHECK、NOT NULL等）+ 应用层逻辑。
- **I（隔离性）**：MVCC（快照读）+ 锁（当前读）。
- **D（持久性）**：Redo Log + Doublewrite Buffer。事务提交时写Redo Log，崩溃后通过Redo Log恢复。

**Q11：Gap Lock是什么？什么情况下会触发？**

**A**：

**Gap Lock**：间隙锁，锁住索引记录之间的"间隙"，防止其他事务插入。

**触发条件**：
- RR隔离级别
- 当前读（SELECT ... FOR UPDATE / LOCK IN SHARE MODE）
- 查询条件为范围查询或等值查询但记录不存在

**示例**：

```sql
-- 数据：id = [1, 5, 10]
SELECT * FROM users WHERE id = 3 FOR UPDATE;
-- id=3不存在，Gap Lock锁住(1, 5)间隙
-- 其他事务INSERT INTO users VALUES (2, ...)会阻塞
```

**Q12：如何监控和优化死锁？**

**A**：

**监控**：

```sql
-- 查看死锁日志
SHOW ENGINE INNODB STATUS\G
-- LATEST DETECTED DEADLOCK部分

-- 开启死锁日志记录
SET GLOBAL innodb_print_all_deadlocks = ON;
-- 死锁信息会记录到error log

-- 查看死锁次数
SHOW STATUS LIKE 'Innodb_deadlocks';
```

**优化**：
1. 尽量使用RC减少Gap Lock
2. 统一加锁顺序（如先锁id小的，再锁id大的）
3. 减少事务持有锁的时间
4. 使用乐观锁替代悲观锁
5. 应用层做好重试机制

---

*此文原创，转载请注明出处。*