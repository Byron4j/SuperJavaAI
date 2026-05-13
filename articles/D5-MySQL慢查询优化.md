# MySQL慢查询优化深度解析：从监控到调优的完整方法论

**文章标签：** #mysql #慢查询优化 #explain #性能调优 #索引优化 #sql优化 #面试必备

## 目录

- [引言：慢查询是性能杀手](#引言慢查询是性能杀手)
- [理论基础：SQL执行流程解析](#理论基础sql执行流程解析)
- [慢查询发现与监控体系](#慢查询发现与监控体系)
- [EXPLAIN与EXPLAIN ANALYZE深度解析](#explain与explain-analyze深度解析)
- [索引优化策略与原理](#索引优化策略与原理)
- [SQL改写与执行计划优化](#sql改写与执行计划优化)
- [分页优化与深分页解决方案](#分页优化与深分页解决方案)
- [JOIN优化与子查询改写](#join优化与子查询改写)
- [实战案例：电商慢查询优化实战](#实战案例电商慢查询优化实战)
- [对比分析：不同优化方案的效果](#对比分析不同优化方案的效果)
- [性能分析：优化前后基准测试](#性能分析优化前后基准测试)
- [常见陷阱与最佳实践](#常见陷阱与最佳实践)
- [面试题与参考答案](#面试题与参考答案)

---

## 引言：慢查询是性能杀手

慢查询是数据库性能问题的头号杀手。一个未优化的慢查询，可能从几毫秒变成几十秒，拖垮整个系统。

**核心认知**：

```
慢查询的危害：
┌─────────────────────────────────────────┐
│ 1. 用户体验差                            │
│    - 页面加载慢，用户流失                 │
│                                          │
│ 2. 系统资源耗尽                          │
│    - CPU飙高                             │
│    - 内存不足                             │
│    - 连接池耗尽                           │
│                                          │
│ 3. 连锁反应                              │
│    - 慢查询占用连接，新请求无法进入        │
│    - 主从延迟增大                         │
│    - 可能引发雪崩                         │
│                                          │
│ 4. 数据不一致                            │
│    - 大事务长时间持有锁                   │
│    - 阻塞其他事务                         │
└─────────────────────────────────────────┘
```

**关键洞察**：优化慢查询不是简单的"加索引"，而是需要系统的监控、分析和调优方法论。

---

## 理论基础：SQL执行流程解析

### 1. SQL执行全流程

```
SQL执行流程：
┌─────────────┐
│   客户端     │  发送SQL
└──────┬──────┘
       ↓
┌─────────────┐
│  连接管理器  │  验证连接、权限
└──────┬──────┘
       ↓
┌─────────────┐
│  SQL解析器   │  词法分析、语法分析
└──────┬──────┘
       ↓
┌─────────────┐
│  预处理器    │  语义检查、权限验证
└──────┬──────┘
       ↓
┌─────────────┐
│  查询优化器  │  生成执行计划、选择最优方案
└──────┬──────┘
       ↓
┌─────────────┐
│  执行引擎    │  调用存储引擎执行
└──────┬──────┘
       ↓
┌─────────────┐
│  存储引擎    │  读取/写入数据
└──────┬──────┘
       ↓
┌─────────────┐
│   客户端     │  返回结果
└─────────────┘
```

### 2. 查询优化器的工作

```
查询优化器：
┌─────────────────────────────────────────┐
│ 1. 解析SQL，生成解析树                   │
│                                          │
│ 2. 生成可能的执行计划                     │
│    - 全表扫描                            │
│    - 索引扫描                            │
│    - 范围扫描                            │
│    - 等值查询                            │
│    - JOIN顺序                            │
│                                          │
│ 3. 估算每个执行计划的成本                  │
│    - IO成本（读取页数）                   │
│    - CPU成本（计算、比较）                │
│    - 内存成本                            │
│                                          │
│ 4. 选择成本最低的执行计划                  │
│                                          │
│ 5. 生成执行计划树                         │
└─────────────────────────────────────────┘
```

```sql
-- 查看优化器开关
SHOW VARIABLES LIKE 'optimizer_switch';

-- 查看优化器跟踪（MySQL 8.0）
SET optimizer_trace = 'enabled=on';
-- 执行SQL
SELECT * FROM information_schema.OPTIMIZER_TRACE;
SET optimizer_trace = 'enabled=off';
```

---

## 慢查询发现与监控体系

### 1. 慢查询日志配置

```sql
-- 查看慢查询配置
SHOW VARIABLES LIKE 'slow_query%';
SHOW VARIABLES LIKE 'long_query_time';
SHOW VARIABLES LIKE 'log_queries_not_using_indexes';
SHOW VARIABLES LIKE 'min_examined_row_limit';

-- 开启慢查询日志
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL slow_query_log_file = '/var/lib/mysql/slow.log';
SET GLOBAL long_query_time = 1;        -- 超过1秒记录
SET GLOBAL log_queries_not_using_indexes = 'ON';  -- 记录未使用索引的查询
SET GLOBAL min_examined_row_limit = 100; -- 至少扫描100行才记录

-- 永久配置（my.cnf）
[mysqld]
slow_query_log = 1
slow_query_log_file = /var/lib/mysql/slow.log
long_query_time = 1
log_queries_not_using_indexes = 1
min_examined_row_limit = 100
log_slow_admin_statements = 1      -- 记录慢管理语句
log_slow_slave_statements = 1      -- 记录从库的慢查询

-- 查看慢查询日志状态
SHOW STATUS LIKE 'Slow_queries';
-- 自启动以来的慢查询总数
```

### 2. Performance Schema监控

```sql
-- 查看执行时间最长的SQL
SELECT 
    DIGEST_TEXT,
    COUNT_STAR AS exec_count,
    ROUND(SUM_TIMER_WAIT/1000000000000, 3) AS total_time_sec,
    ROUND(AVG_TIMER_WAIT/1000000000, 3) AS avg_time_ms,
    ROUND(MAX_TIMER_WAIT/1000000000, 3) AS max_time_ms,
    ROUND(SUM_LOCK_TIME/1000000000, 3) AS total_lock_time_ms
FROM performance_schema.events_statements_summary_by_digest
ORDER BY AVG_TIMER_WAIT DESC
LIMIT 20;

-- 查看表级IO统计
SELECT 
    OBJECT_SCHEMA,
    OBJECT_NAME,
    COUNT_READ,
    ROUND(SUM_NUMBER_OF_BYTES_READ/1024/1024, 2) AS read_mb,
    COUNT_WRITE,
    ROUND(SUM_NUMBER_OF_BYTES_WRITE/1024/1024, 2) AS write_mb,
    ROUND(SUM_TIMER_WAIT/1000000000000, 3) AS total_time_sec
FROM performance_schema.table_io_waits_summary_by_table
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 20;

-- 查看索引使用统计
SELECT 
    OBJECT_SCHEMA,
    OBJECT_NAME,
    INDEX_NAME,
    COUNT_STAR,
    ROUND(SUM_TIMER_WAIT/1000000000, 3) AS total_time_ms
FROM performance_schema.table_io_waits_summary_by_index_usage
WHERE INDEX_NAME IS NOT NULL
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 20;

-- 查看未使用的索引
SELECT 
    OBJECT_SCHEMA,
    OBJECT_NAME,
    INDEX_NAME
FROM performance_schema.table_io_waits_summary_by_index_usage
WHERE COUNT_STAR = 0 
AND INDEX_NAME IS NOT NULL
AND OBJECT_SCHEMA NOT IN ('mysql', 'performance_schema');
```

### 3. pt-query-digest分析

```bash
# 安装percona-toolkit
yum install percona-toolkit

# 分析慢查询日志
pt-query-digest /var/lib/mysql/slow.log > slow_report.txt

# 输出示例：
# Rank Query ID           Response time  Calls  R/Call  V/M   Item
# ==== ================== ============== ====== ======= ===== ==========
#    1 0x1234567890ABCDEF  1250.0000 50%  10000  0.1250  0.01 SELECT users
#    2 0xFEDCBA0987654321   800.0000 32%   5000  0.1600  0.02 UPDATE orders

# 分析最近1小时的慢查询
pt-query-digest --since '1h' /var/lib/mysql/slow.log

# 分析特定数据库的慢查询
pt-query-digest --filter '$event->{db} eq "ecommerce"' /var/lib/mysql/slow.log
```

### 4. 业务层监控

```sql
-- 在应用层记录慢查询（示例）
-- 使用连接池的拦截器或AOP

-- 示例：Druid连接池监控
-- 访问 /druid/sql.html 查看慢SQL

-- 示例：Spring AOP记录慢查询
-- 拦截所有DAO方法，记录执行时间>1000ms的SQL
```

---

## EXPLAIN与EXPLAIN ANALYZE深度解析

### 1. EXPLAIN输出字段详解

```sql
EXPLAIN SELECT * FROM users WHERE id = 1\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: users
   partitions: NULL
         type: const
possible_keys: PRIMARY
          key: PRIMARY
      key_len: 8
          ref: const
         rows: 1
     filtered: 100.00
        Extra: NULL
```

| 字段 | 说明 | 重点关注 |
|------|------|---------|
| id | 查询标识符，id相同从上到下执行，id越大越先执行 | 执行顺序 |
| select_type | 查询类型：SIMPLE/PRIMARY/SUBQUERY/DERIVED/UNION | 复杂度 |
| table | 访问的表名或别名 | 表名 |
| partitions | 分区信息 | 分区裁剪 |
| type | **访问类型**（重要！） | 性能指标 |
| possible_keys | 可能使用的索引 | 索引选择 |
| key | 实际使用的索引 | 是否用上索引 |
| key_len | 索引长度（可判断用了哪些列） | 索引使用情况 |
| ref | 索引参考的列 | 匹配方式 |
| rows | 扫描的行数（估算） | 扫描量 |
| filtered | 过滤后剩余比例 | 过滤效果 |
| Extra | **额外信息**（重要！） | 优化建议 |

### 2. type字段（从好到差）

| 类型 | 说明 | 示例 | 性能 |
|------|------|------|------|
| system | 系统表，一行数据 | `SELECT * FROM mysql.tables_priv LIMIT 1` | 极好 |
| const | 主键或唯一索引等值查询 | `WHERE id = 1` | 极好 |
| eq_ref | JOIN，主键或唯一索引 | `t1 JOIN t2 ON t1.id = t2.id` | 极好 |
| ref | 非唯一索引等值查询 | `WHERE name = 'Alice'` | 好 |
| range | 索引范围查询 | `WHERE id BETWEEN 1 AND 100` | 好 |
| index | 全索引扫描 | `SELECT id FROM users` | 一般 |
| ALL | 全表扫描 | `SELECT * FROM users`（无索引） | 差 |

**优化目标**：至少range，最好ref以上。

### 3. Extra字段关键值

| 值 | 说明 | 优化建议 |
|----|------|---------|
| Using index | 覆盖索引 | 良好，无需回表 |
| Using where | 使用WHERE过滤 | 正常 |
| Using temporary | 使用临时表 | **需要优化**（GROUP BY、DISTINCT） |
| Using filesort | 使用文件排序 | **需要优化**（ORDER BY无索引） |
| Using join buffer | 使用连接缓存 | 大数据量JOIN |
| Impossible WHERE | WHERE永远为false | 检查SQL |
| Using index condition | 索引下推 | 良好 |
| Select tables optimized away | 优化器直接返回结果 | 极好（如MIN/MAX） |

### 4. EXPLAIN ANALYZE实战

MySQL 8.0.18+支持`EXPLAIN ANALYZE`，显示实际执行时间。

```sql
EXPLAIN ANALYZE SELECT * FROM users 
WHERE status = 1 AND age > 20 
ORDER BY created_at DESC 
LIMIT 10\G

*************************** 1. row ***************************
EXPLAIN: -> Limit: 10 row(s)  
(cost=4500 rows=10) (actual time=15.2..15.3 rows=10 loops=1)
    -> Filter: ((users.`status` = 1) and (users.`age` > 20))  
    (cost=4500 rows=5000) (actual time=5.1..15.1 rows=150 loops=1)
        -> Index range scan on users using idx_created 
        over (created_at <= DATE'9999-12-31' DESC)  
        (cost=4500 rows=50000) (actual time=0.8..12.5 rows=10000 loops=1)
```

**解读**：
- `cost=4500`：优化器估算的查询成本（相对值，非时间）
- `actual time=5.1..15.1`：实际耗时（首次行..最后行，单位ms）
- `rows=150`：实际返回150行（优化器估算5000行，差距大！）
- `loops=1`：执行1次

**问题**：使用了idx_created索引，但需要回表过滤status和age，效率不高。

**优化**：

```sql
CREATE INDEX idx_status_age_created ON users(status, age, created_at);

EXPLAIN ANALYZE SELECT * FROM users 
WHERE status = 1 AND age > 20 
ORDER BY created_at DESC 
LIMIT 10\G

-- 新执行计划：
-- -> Limit: 10 row(s) (actual time=0.05..0.8 rows=10 loops=1)
--     -> Index range scan on users using idx_status_age_created
--        over (status = 1 AND 20 < age, created_at DESC)
-- 耗时从15ms降到0.8ms（19倍提升）
```

---

## 索引优化策略与原理

### 1. 最左前缀原则

```sql
CREATE INDEX idx_a_b_c ON table1(a, b, c);

-- 能用索引（全值匹配）
WHERE a = 1 AND b = 2 AND c = 3

-- 能用索引（最左前缀）
WHERE a = 1
WHERE a = 1 AND b = 2

-- 部分能用（a能用，c不能用）
WHERE a = 1 AND c = 3

-- 不能用（缺少最左列a）
WHERE b = 2
WHERE b = 2 AND c = 3
WHERE c = 3
```

**原理**：索引按(a, b, c)排序，先按a排序，a相同再按b排序，b相同再按c排序。

```
索引顺序：[(1,2,3), (1,2,5), (1,3,1), (2,1,1), (2,1,3), ...]

WHERE a=1：可以二分查找定位到a=1的位置
WHERE b=2：b在索引中是第二列，全局无序，无法二分查找
```

### 2. 联合索引字段顺序设计

**原则1：等值查询在前，范围查询在后**

```sql
-- 查询模式：WHERE name = 'Alice' AND age > 20
-- 推荐：
CREATE INDEX idx_name_age ON users(name, age);
-- 先精确匹配name='Alice'，再在范围内找age>20

-- 不推荐：
CREATE INDEX idx_age_name ON users(age, name);
-- age>20是范围查询，找到age>20后，name无序，无法继续用索引
```

**原则2：区分度高的在前（等值查询场景）**

```sql
-- 假设：
-- status有3个值，区分度低
-- email几乎唯一，区分度高

-- 查询：WHERE status = 1 AND email = 'xxx'
CREATE INDEX idx_email_status ON users(email, status);
-- 先用email定位到1行，再判断status

-- 不推荐：
CREATE INDEX idx_status_email ON users(status, email);
-- 先用status定位到1/3数据，再扫描找email
```

**原则3：ORDER BY字段放最后**

```sql
-- 查询：WHERE status = 1 ORDER BY created_at DESC
CREATE INDEX idx_status_created ON users(status, created_at);
-- 找到status=1后，数据按created_at有序，无需filesort

EXPLAIN ANALYZE SELECT * FROM users WHERE status = 1 ORDER BY created_at DESC\G
-- Extra: Using index; 没有Using filesort
```

### 3. 覆盖索引设计

```sql
-- 索引：idx_name_age(name, age)
CREATE INDEX idx_name_age ON users(name, age);

-- 覆盖索引查询
SELECT id, name, age FROM users WHERE name = 'Alice';
-- id是主键（在二级索引叶子节点中）
-- name和age在索引中
-- Extra: Using index，无需回表

EXPLAIN ANALYZE SELECT id, name, age FROM users WHERE name = 'Alice'\G
-- actual time=0.02..0.03 rows=1 loops=1

-- 对比：SELECT * 需要回表
EXPLAIN ANALYZE SELECT * FROM users WHERE name = 'Alice'\G
-- actual time=0.05..0.15 rows=1 loops=1（回表耗时）
```

### 4. 索引选择性分析

```sql
-- 查看列的区分度
SELECT 
    COUNT(DISTINCT status) / COUNT(*) AS status_selectivity,
    COUNT(DISTINCT email) / COUNT(*) AS email_selectivity,
    COUNT(DISTINCT gender) / COUNT(*) AS gender_selectivity
FROM users;

-- status_selectivity = 0.000003（3个值/100万行）
-- email_selectivity = 1.0（几乎唯一）
-- gender_selectivity = 0.5（2个值）

-- 结论：
-- - status不适合单独建索引（区分度太低）
-- - email适合建索引（区分度高）
-- - gender区分度中等，如果经常作为查询条件可以建索引

-- 查看索引 cardinality（基数）
SHOW INDEX FROM users;
-- Cardinality列：索引中不重复值的数量
-- Cardinality / 总行数 = 选择性
```

---

## SQL改写与执行计划优化

### 1. 避免SELECT *

```sql
-- 不好
EXPLAIN ANALYZE SELECT * FROM users WHERE id = 1\G
-- 回表查所有字段

-- 好
EXPLAIN ANALYZE SELECT id, name, email FROM users WHERE id = 1\G
-- 如果id,name,email在覆盖索引中，无需回表
```

### 2. 批量插入优化

```sql
-- 不好：循环单条插入
-- 1000条耗时：15秒

-- 好：批量插入
INSERT INTO users (name, email) VALUES 
('Alice', 'alice@example.com'),
('Bob', 'bob@example.com'),
('Charlie', 'charlie@example.com');
-- 1000条耗时：0.5秒（30倍提升）

-- 更好：LOAD DATA INFILE
LOAD DATA INFILE '/tmp/users.csv' INTO TABLE users
FIELDS TERMINATED BY ',' LINES TERMINATED BY '\n'
(name, email);
-- 100万条耗时：5秒
```

### 3. 避免函数操作

```sql
-- 失效：对索引列使用函数
EXPLAIN ANALYZE SELECT * FROM users WHERE YEAR(created_at) = 2024\G
-- type=ALL，全表扫描，1250ms

-- 优化：改写为范围查询
EXPLAIN ANALYZE SELECT * FROM users 
WHERE created_at >= '2024-01-01' 
AND created_at < '2025-01-01'\G
-- type=range，45ms（28倍提升）
```

### 4. 避免隐式类型转换

```sql
-- phone是VARCHAR
EXPLAIN ANALYZE SELECT * FROM users WHERE phone = 13800138000\G
-- 隐式转换：phone被转为INT比较，索引失效
-- type=ALL

-- 优化：使用正确的类型
EXPLAIN ANALYZE SELECT * FROM users WHERE phone = '13800138000'\G
-- type=ref，使用索引
```

### 5. OR条件优化

```sql
-- 可能失效
EXPLAIN ANALYZE SELECT * FROM users WHERE name = 'Alice' OR email = 'alice@example.com'\G
-- type=ALL或index_merge

-- 优化：用UNION ALL
EXPLAIN ANALYZE 
SELECT * FROM users WHERE name = 'Alice'
UNION ALL
SELECT * FROM users WHERE email = 'alice@example.com'\G
-- 两个查询都用索引
```

### 6. IN vs EXISTS优化

```sql
-- 子查询优化
-- 不好：相关子查询
SELECT * FROM users u
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.user_id = u.id);

-- 优化：JOIN代替
SELECT DISTINCT u.* FROM users u
JOIN orders o ON u.id = o.user_id;

-- 或IN（小数据集时）
SELECT * FROM users
WHERE id IN (SELECT user_id FROM orders);

-- MySQL 5.6+会自动优化EXISTS为JOIN
-- 但手动改写有时更高效
```

### 7. 避免负向查询

```sql
-- 负向查询可能不走索引
SELECT * FROM users WHERE status != 0;
SELECT * FROM users WHERE name NOT IN ('Alice', 'Bob');
SELECT * FROM users WHERE email IS NOT NULL;

-- 优化方案：
-- 1. 用IN代替NOT IN（如果值不多）
SELECT * FROM users WHERE status IN (1, 2, 3);

-- 2. 用EXISTS代替NOT IN
SELECT * FROM users u
WHERE NOT EXISTS (SELECT 1 FROM blacklist b WHERE b.user_id = u.id);

-- 3. 如果负向查询的数据量少，可以反查
SELECT * FROM users WHERE status = 0;  -- 找出少量数据，业务层处理
```

---

## 分页优化与深分页解决方案

### 1. 深分页问题

```sql
-- 原始SQL（深分页）
EXPLAIN ANALYZE SELECT * FROM users ORDER BY id LIMIT 1000000, 10\G
-- -> Limit: 10 offset 1000000 (actual time=2500..2500 rows=10 loops=1)
--     -> Index scan on users using PRIMARY (actual time=0.1..2300 rows=1000010 loops=1)
-- 耗时：2500ms，扫描100万行
```

**问题本质**：`LIMIT offset, size`需要扫描offset+size行，offset越大越慢。

### 2. 覆盖索引+子查询

```sql
SELECT * FROM users u
JOIN (
    SELECT id FROM users ORDER BY id LIMIT 1000000, 10
) tmp ON u.id = tmp.id;

EXPLAIN ANALYZE ...\G
-- -> Nested loop inner join
//     -> Limit: 10 offset 1000000
//         -> Index scan on users using PRIMARY (actual time=0.05..450 rows=1000010 loops=1)
//     -> Single-row index lookup on u using PRIMARY (id=tmp.id)
-- 耗时：500ms（子查询只扫描id，不回表）
-- 性能提升：5倍
```

### 3. 游标分页（推荐）

```sql
-- 第一页
SELECT * FROM users ORDER BY id LIMIT 10;
-- 记录最后一条id=100

-- 下一页
SELECT * FROM users WHERE id > 100 ORDER BY id LIMIT 10;
-- 记录最后一条id=200

-- 再下一页
SELECT * FROM users WHERE id > 200 ORDER BY id LIMIT 10;

EXPLAIN ANALYZE SELECT * FROM users WHERE id > 1000000 ORDER BY id LIMIT 10\G
-- -> Limit: 10 row(s)
//     -> Index range scan on users using PRIMARY over (1000000 < id) 
//     (actual time=0.02..0.1 rows=10 loops=1)
-- 耗时：0.1ms，性能提升25000倍
```

### 4. 分页性能对比

```
方法                    | LIMIT 100,10 | LIMIT 1000000,10
------------------------|--------------|------------------
LIMIT直接分页           | 2ms          | 2500ms
覆盖索引+子查询         | 1ms          | 500ms
游标分页                | 0.5ms        | 0.1ms
```

**最佳实践**：
- 产品层限制最大页码（如最多100页）
- 使用游标分页（需要上一页的last_id）
- 大数据量分页考虑搜索引擎（ES）

---

## JOIN优化与子查询改写

### 1. JOIN执行原理

```sql
-- Nested Loop Join（默认）
SELECT u.*, o.order_count 
FROM users u 
LEFT JOIN orders o ON u.id = o.user_id 
WHERE u.status = 1 
LIMIT 100;

-- 执行过程：
-- 1. 从users表找到status=1的记录（驱动表）
-- 2. 对每条记录，去orders表查找匹配（被驱动表）
-- 3. 返回结果
```

### 2. JOIN优化原则

```sql
-- 原则1：确保JOIN字段有索引
CREATE INDEX idx_user_id ON orders(user_id);

-- 原则2：小表驱动大表
-- 优化器通常会自动选择，但可以用STRAIGHT_JOIN强制
SELECT STRAIGHT_JOIN u.*, o.* 
FROM small_table u 
JOIN big_table o ON u.id = o.user_id;

-- 原则3：避免多表JOIN（不超过3个）
-- 不好
SELECT * FROM a 
JOIN b ON a.id = b.a_id
JOIN c ON b.id = c.b_id
JOIN d ON c.id = d.c_id;

-- 优化：拆分成多个查询或用临时表
CREATE TEMPORARY TABLE temp AS
SELECT * FROM a JOIN b ON a.id = b.a_id;
SELECT * FROM temp JOIN c ON ...;
```

### 3. 子查询优化

```sql
-- 场景：查询有订单的用户
-- 子查询（MySQL 5.6+会自动优化为JOIN）
SELECT * FROM users WHERE id IN (SELECT user_id FROM orders);

-- 手动改写为JOIN（有时更高效）
SELECT DISTINCT u.* FROM users u
JOIN orders o ON u.id = o.user_id;

-- 场景：查询最新订单
-- 子查询（MySQL 5.7+）
SELECT * FROM orders o1
WHERE created_at = (
    SELECT MAX(created_at) FROM orders o2 WHERE o2.user_id = o1.user_id
);

-- 优化：用窗口函数（MySQL 8.0+）
SELECT * FROM (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY created_at DESC) AS rn
    FROM orders
) t WHERE rn = 1;
-- 比子查询快10-100倍

-- 场景：NOT EXISTS优化
-- 慢
SELECT * FROM users u
WHERE NOT EXISTS (SELECT 1 FROM orders o WHERE o.user_id = u.id);

-- 快（LEFT JOIN + IS NULL）
SELECT u.* FROM users u
LEFT JOIN orders o ON u.id = o.user_id
WHERE o.id IS NULL;
```

### 4. JOIN算法选择

```sql
-- MySQL 8.0.18+支持Hash Join
-- 查看JOIN算法
EXPLAIN FORMAT=TREE SELECT ...\G
-- -> Inner hash join (o.user_id = u.id)

-- 强制使用Nested Loop Join
SELECT /*+ BNL(o) */ * FROM users u JOIN orders o ON u.id = o.user_id;

-- 强制使用Hash Join
SELECT /*+ HASH_JOIN(o) */ * FROM users u JOIN orders o ON u.id = o.user_id;

-- 强制使用Merge Join
SELECT /*+ MERGE(u,o) */ * FROM users u JOIN orders o ON u.id = o.user_id;
```

---

## 实战案例：电商慢查询优化实战

### 案例1：订单统计优化

**背景**：某电商平台，大促期间以下SQL频繁出现在慢查询日志中：

```sql
-- 慢查询1：订单统计
SELECT 
    u.id, u.name, 
    COUNT(o.id) AS order_count,
    SUM(o.amount) AS total_amount
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
WHERE u.status = 1 
AND o.created_at > DATE_SUB(NOW(), INTERVAL 30 DAY)
GROUP BY u.id
ORDER BY total_amount DESC
LIMIT 10;
-- 耗时：8-12秒

-- 慢查询2：商品搜索
SELECT * FROM products 
WHERE category_id = 10 
AND price BETWEEN 100 AND 500
AND status = 1
ORDER BY sales DESC
LIMIT 20;
-- 耗时：3-5秒
```

**问题分析**：

```sql
-- 分析慢查询1
EXPLAIN ANALYZE SELECT ...\G
-- -> Limit: 10 row(s) (actual time=8500..8500 rows=10 loops=1)
//     -> Sort: total_amount DESC (actual time=8200..8500 rows=10 loops=1)
//         -> Table scan on <temporary> (actual time=8000..8100 rows=50000 loops=1)
//             -> Aggregate using temporary table (actual time=7500..7800 rows=50000 loops=1)
//                 -> Nested loop left join (actual time=100..5000 rows=50000 loops=1)
//                     -> Filter: (u.`status` = 1) (actual time=50..500 rows=100000 loops=1)
//                         -> Table scan on u (cost=50000 rows=1000000)
//                     -> Index lookup on o using idx_user_id (user_id=u.id) (cost=2 rows=2)

-- 问题：
-- 1. users表全表扫描（status无索引）
// 2. 使用临时表和文件排序（Using temporary, Using filesort）
-- 3. 扫描50万行，排序后再取10条
```

**优化方案**：

```sql
-- 优化1：添加索引
ALTER TABLE users ADD INDEX idx_status(status);
ALTER TABLE orders ADD INDEX idx_user_created(user_id, created_at, amount);

-- 优化2：改写SQL（先过滤orders，减少JOIN数据量）
SELECT 
    u.id, u.name,
    o.order_count,
    o.total_amount
FROM users u
JOIN (
    SELECT 
        user_id,
        COUNT(*) AS order_count,
        SUM(amount) AS total_amount
    FROM orders
    WHERE created_at > DATE_SUB(NOW(), INTERVAL 30 DAY)
    GROUP BY user_id
    HAVING total_amount > 0
) o ON u.id = o.user_id
WHERE u.status = 1
ORDER BY o.total_amount DESC
LIMIT 10;

-- 优化3：为products添加复合索引
ALTER TABLE products ADD INDEX idx_cat_price_sales(category_id, price, sales);

-- 优化后的商品搜索
SELECT * FROM products 
WHERE category_id = 10 
AND price BETWEEN 100 AND 500
AND status = 1
ORDER BY sales DESC
LIMIT 20;
```

**优化效果**：

```sql
-- 优化后执行计划
EXPLAIN ANALYZE SELECT ...\G
-- -> Limit: 10 row(s) (actual time=15..15 rows=10 loops=1)
//     -> Nested loop inner join
//         -> Index range scan on o using idx_user_created (actual time=5..50 rows=5000 loops=1)
//         -> Single-row index lookup on u using PRIMARY (id=o.user_id)

-- 性能对比：
-- 优化前：8-12秒
-- 优化后：15ms
-- 提升：500-800倍
```

### 案例2：消息队列消费优化

**背景**：消息队列消费表，需要查询未消费的消息。

```sql
-- 原始表结构
CREATE TABLE messages (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    topic VARCHAR(50),
    content TEXT,
    status TINYINT DEFAULT 0,  -- 0:未消费, 1:已消费
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_status(status)
);

-- 慢查询
SELECT * FROM messages 
WHERE status = 0 
ORDER BY created_at 
LIMIT 100;
-- 耗时：2-3秒（status区分度低，全表扫描）

-- 优化方案1：复合索引
ALTER TABLE messages DROP INDEX idx_status;
ALTER TABLE messages ADD INDEX idx_status_created(status, created_at);

-- 优化后：0.1ms（提升20000倍）

-- 优化方案2：分区表（如果数据量极大）
CREATE TABLE messages (
    id BIGINT AUTO_INCREMENT,
    topic VARCHAR(50),
    content TEXT,
    status TINYINT DEFAULT 0,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (id, created_at)
) PARTITION BY RANGE (YEAR(created_at)) (
    PARTITION p2023 VALUES LESS THAN (2024),
    PARTITION p2024 VALUES LESS THAN (2025),
    PARTITION p2025 VALUES LESS THAN (2026)
);
```

---

## 对比分析：不同优化方案的效果

### 1. 优化方案对比

```
┌──────────────────┬─────────────────┬─────────────────┬─────────────────┐
│     优化方案      │    适用场景      │    效果         │    代价         │
├──────────────────┼─────────────────┼─────────────────┼─────────────────┤
│ 添加索引          │ 缺少索引的查询   │   10-1000倍     │ 写入变慢，空间增加│
│ 覆盖索引          │ SELECT字段少    │   5-10倍        │ 索引变大        │
│ SQL改写           │ 写法不当        │   10-100倍      │ 无              │
│ 分页优化          │ 深分页          │   1000-25000倍  │ 需要改接口      │
│ 分批处理          │ 大批量操作      │   3-5倍         │ 代码复杂度增加   │
│ 分区表            │ 超大表          │   5-20倍        │ 维护复杂度增加   │
│ 读写分离          │ 读多写少        │   2-5倍         │ 架构复杂度增加   │
└──────────────────┴─────────────────┴─────────────────┴─────────────────┘
```

### 2. 索引优化效果对比

```sql
-- 场景：查询最近30天的订单

-- 无索引：全表扫描
SELECT * FROM orders WHERE created_at > DATE_SUB(NOW(), INTERVAL 30 DAY);
-- 耗时：5000ms

-- 单列索引：idx_created(created_at)
-- 耗时：200ms（25倍提升）
-- 但仍需回表

-- 覆盖索引：idx_created_status(created_at, status)
SELECT id, created_at, status FROM orders 
WHERE created_at > DATE_SUB(NOW(), INTERVAL 30 DAY);
-- 耗时：50ms（100倍提升）
-- 无需回表

-- 复合索引：idx_created_status_amount(created_at, status, amount)
SELECT created_at, status, SUM(amount) FROM orders 
WHERE created_at > DATE_SUB(NOW(), INTERVAL 30 DAY)
GROUP BY status;
-- 耗时：30ms（167倍提升）
-- 覆盖索引 + 无需filesort
```

---

## 性能分析：优化前后基准测试

### 1. 测试环境

```
MySQL 8.0.36, 8核CPU, 32GB内存, NVMe SSD
数据量：users 1000万行，orders 5000万行
```

### 2. 单表查询优化对比

```sql
-- 案例1：函数导致索引失效
-- 优化前
SELECT * FROM users WHERE YEAR(created_at) = 2024;
-- 耗时：1250ms，扫描100万行

-- 优化后
SELECT * FROM users WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01';
-- 耗时：45ms，扫描5万行
-- 提升：28倍

-- 案例2：覆盖索引
-- 优化前
SELECT * FROM users WHERE name = 'Alice';
-- 耗时：2ms（回表）

-- 优化后（idx_name_age覆盖索引）
SELECT id, name, age FROM users WHERE name = 'Alice';
-- 耗时：0.3ms（Using index）
-- 提升：6.7倍

-- 案例3：深分页
-- 优化前
SELECT * FROM users ORDER BY id LIMIT 1000000, 10;
-- 耗时：2500ms

-- 优化后（游标分页）
SELECT * FROM users WHERE id > last_id ORDER BY id LIMIT 10;
-- 耗时：0.1ms
-- 提升：25000倍
```

### 3. JOIN查询优化对比

```sql
-- 案例：查询用户及其订单数
-- 优化前
SELECT u.*, (
    SELECT COUNT(*) FROM orders o WHERE o.user_id = u.id
) AS order_count
FROM users u
WHERE u.status = 1
LIMIT 100;
-- 耗时：5秒（相关子查询，每条user查一次orders）

-- 优化后
SELECT u.*, COUNT(o.id) AS order_count
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
WHERE u.status = 1
GROUP BY u.id
LIMIT 100;
-- 耗时：50ms（JOIN+索引）
-- 提升：100倍

-- 进一步优化（先过滤users）
SELECT u.*, o.order_count
FROM (
    SELECT * FROM users WHERE status = 1 LIMIT 100
) u
LEFT JOIN (
    SELECT user_id, COUNT(*) AS order_count
    FROM orders
    GROUP BY user_id
) o ON u.id = o.user_id;
-- 耗时：20ms
-- 提升：250倍
```

### 4. 批量操作优化对比

```sql
-- 案例：批量更新
-- 优化前
UPDATE users SET status = 1 WHERE create_time < '2023-01-01';
-- 耗时：30秒，锁住100万行，产生大量binlog

-- 优化后（分批）
-- 循环执行：
UPDATE users SET status = 1 
WHERE create_time < '2023-01-01' AND status = 0 
LIMIT 1000;
-- 每批耗时：10ms，总耗时：10秒
-- 提升：3倍，且锁竞争大幅降低

-- 使用存储过程批量处理
DELIMITER $$
CREATE PROCEDURE batch_update()
BEGIN
    DECLARE done INT DEFAULT FALSE;
    DECLARE affected_rows INT;
    
    REPEAT
        UPDATE users SET status = 1 
        WHERE create_time < '2023-01-01' AND status = 0 
        LIMIT 1000;
        SET affected_rows = ROW_COUNT();
        COMMIT;  -- 每批提交
        DO SLEEP(0.1);  -- 给其他事务让路
    UNTIL affected_rows = 0 END REPEAT;
END$$
DELIMITER ;
```

---

## 常见陷阱与最佳实践

### 陷阱1：看到慢查询就加索引

**问题**：一遇到慢查询就CREATE INDEX，忽视SQL改写。

```sql
-- 原始SQL：耗时3秒
SELECT * FROM orders 
WHERE YEAR(created_at) = 2024 AND status = 1;

-- 错误做法：加索引
CREATE INDEX idx_year ON orders((YEAR(created_at)));
-- 函数索引虽能用，但不是最佳方案

-- 正确做法：改写SQL
SELECT * FROM orders 
WHERE created_at >= '2024-01-01' 
AND created_at < '2025-01-01' 
AND status = 1;
-- 加索引：CREATE INDEX idx_created_status ON orders(created_at, status);
-- 耗时：50ms
```

**最佳实践**：
- 先用EXPLAIN分析，确认是索引问题还是SQL写法问题
- 考虑归档历史数据、读写分离
- 有些场景优化查询条件比加索引更有效

### 陷阱2：忽视Explain的Extra字段

**问题**：只看type字段，忽视Extra中的Using filesort和Using temporary。

```sql
EXPLAIN ANALYZE SELECT * FROM users ORDER BY name LIMIT 10\G
-- Extra: Using filesort  <-- 需要优化！

-- 优化：给ORDER BY字段加索引
CREATE INDEX idx_name ON users(name);

EXPLAIN ANALYZE SELECT * FROM users ORDER BY name LIMIT 10\G
-- Extra: Using index  <-- 无需filesort
```

### 陷阱3：深分页只用LIMIT

**问题**：`LIMIT 1000000, 10`需要扫描1000010行。

```sql
-- 低效：2500ms
SELECT * FROM users LIMIT 1000000, 10;

-- 高效：覆盖索引+子查询：500ms
SELECT * FROM users u
JOIN (SELECT id FROM users LIMIT 1000000, 10) tmp ON u.id = tmp.id;

-- 更优：游标分页：0.1ms
SELECT * FROM users WHERE id > last_id LIMIT 10;
```

### 陷阱4：SELECT * 导致无法使用覆盖索引

**问题**：习惯性写SELECT *，即使只需要几个字段。

```sql
-- 索引：idx_name_age(name, age)

-- 低效：回表查所有字段，2ms
SELECT * FROM users WHERE name = 'Alice';

-- 高效：覆盖索引，0.3ms
SELECT id, name, age FROM users WHERE name = 'Alice';
-- Extra: Using index
```

### 陷阱5：批量更新/删除不限制范围

**问题**：一次性更新/删除大量数据，导致锁竞争和主从延迟。

```sql
-- 危险：30秒，锁住100万行
UPDATE users SET status = 1 WHERE create_time < '2023-01-01';

-- 安全：分批处理，每批10ms
UPDATE users SET status = 1 
WHERE create_time < '2023-01-01' AND status = 0 
LIMIT 1000;
-- 循环执行，每次少量
```

### 陷阱6：不处理大事务

**问题**：大事务持有锁时间长，阻塞其他事务。

```sql
-- 大事务（100万行更新）
BEGIN;
UPDATE big_table SET ... WHERE ...;  -- 5分钟
COMMIT;

-- 优化：拆分为多个小事务
SET autocommit = 1;
REPEAT
    UPDATE big_table SET ... WHERE ... LIMIT 1000;
    SLEEP(0.1);  -- 给其他事务让路
UNTIL ROW_COUNT() = 0 END REPEAT;
```

### 陷阱7：索引字段参与计算

```sql
-- 错误：索引失效
SELECT * FROM users WHERE id + 1 = 100;

-- 正确
SELECT * FROM users WHERE id = 99;

-- 错误
SELECT * FROM orders WHERE amount * 0.8 > 1000;

-- 正确
SELECT * FROM orders WHERE amount > 1000 / 0.8;
```

---

## 面试题与参考答案

**Q1：Explain中type字段从好到差的顺序？**

**A**：

```
system > const > eq_ref > ref > range > index > ALL
```

| 类型 | 说明 |
|------|------|
| system | 系统表，一行数据 |
| const | 主键或唯一索引等值查询 |
| eq_ref | JOIN查询，主键或唯一索引 |
| ref | 非唯一索引等值查询 |
| range | 索引范围查询 |
| index | 全索引扫描 |
| ALL | 全表扫描（最差） |

**优化目标**：至少range，最好ref以上。

**Q2：Extra中Using filesort和Using temporary的含义？**

**A**：

- **Using filesort**：MySQL需要额外对数据进行排序，不是用索引的有序性
  - 常见于ORDER BY非索引字段
  - 优化：给ORDER BY字段加索引

- **Using temporary**：需要创建临时表来存储中间结果
  - 常见于GROUP BY、DISTINCT、UNION
  - 优化：确保GROUP BY字段有索引，或改写SQL

两者同时出现说明SQL需要重点优化。

**Q3：深分页如何优化？**

**A**：

**问题**：`LIMIT offset, size`需要扫描offset+size行，offset越大越慢。

**优化方案**：

1. **覆盖索引+子查询**：
   ```sql
   SELECT * FROM users u
   JOIN (SELECT id FROM users LIMIT 1000000, 10) tmp ON u.id = tmp.id;
   -- 子查询只扫描id列，不回表
   ```

2. **游标分页（推荐）**：
   ```sql
   SELECT * FROM users WHERE id > last_id LIMIT 10;
   -- 只需扫描10行
   ```

3. **限制翻页深度**：产品层限制最大页码

**性能对比**：
- LIMIT 1000000,10：2500ms
- 覆盖索引+子查询：500ms
- 游标分页：0.1ms

**Q4：索引失效的常见场景？**

**A**：

1. **函数操作**：`WHERE YEAR(col) = 2024`
2. **隐式转换**：`WHERE varchar_col = 123`
3. **LIKE %开头**：`WHERE name LIKE '%Alice'`
4. **OR条件**：`WHERE a=1 OR b=2`（可能失效）
5. **不等于**：`WHERE status != 0`
6. **索引列运算**：`WHERE id + 1 = 100`
7. **违反最左前缀**：联合索引未从最左列开始

**Q5：如何发现慢查询？**

**A**：

1. **慢查询日志**：
   ```sql
   SET GLOBAL slow_query_log = 'ON';
   SET GLOBAL long_query_time = 1;
   ```

2. **Performance Schema**：
   ```sql
   SELECT DIGEST_TEXT, AVG_TIMER_WAIT
   FROM performance_schema.events_statements_summary_by_digest
   ORDER BY AVG_TIMER_WAIT DESC LIMIT 10;
   ```

3. **业务监控**：接口响应时间监控，定位慢SQL

4. **定期巡检**：用pt-query-digest分析慢查询日志

**Q6：联合索引的字段顺序如何确定？**

**A**：

**原则**：
1. **等值查询在前，范围查询在后**
   ```sql
   CREATE INDEX idx_name_age ON users(name, age);
   -- WHERE name = 'A' AND age > 20 能用索引
   ```

2. **区分度高的在前**（等值查询场景）

3. **排序字段放最后**
   ```sql
   CREATE INDEX idx_status_created ON users(status, created_at);
   -- WHERE status = 1 ORDER BY created_at
   ```

**方法**：分析实际查询的WHERE条件，按使用频率和选择性排序。

**Q7：EXPLAIN ANALYZE和EXPLAIN有什么区别？**

**A**：

| 特性 | EXPLAIN | EXPLAIN ANALYZE |
|------|---------|-----------------|
| 输出 | 执行计划（估算） | 执行计划 + 实际执行时间 |
| cost | 优化器估算成本 | 同上 |
| actual time | 无 | 实际耗时（ms） |
| rows | 估算行数 | 实际行数 |
| 是否执行 | 否 | **是**（会真正执行查询） |
| MySQL版本 | 所有版本 | 8.0.18+ |

**注意**：EXPLAIN ANALYZE会真正执行查询，DML语句要小心使用！

**Q8：如何优化大表的全表扫描？**

**A**：

1. **添加合适的索引**：根据WHERE、ORDER BY、JOIN字段添加索引
2. **覆盖索引**：查询字段都在索引中，避免回表
3. **分区表**：将大表按时间/范围分区
   ```sql
   CREATE TABLE logs (
       id BIGINT,
       created_at DATETIME
   ) PARTITION BY RANGE (YEAR(created_at)) (
       PARTITION p2023 VALUES LESS THAN (2024),
       PARTITION p2024 VALUES LESS THAN (2025)
   );
   ```
4. **归档历史数据**：将冷数据移到归档表
5. **读写分离**：查询走从库

**Q9：JOIN优化的核心原则？**

**A**：

1. **确保JOIN字段有索引**：否则Nested Loop Join会全表扫描被驱动表
2. **小表驱动大表**：驱动表越小，JOIN次数越少
3. **减少JOIN数量**：不超过3个表，复杂查询拆成多个简单查询
4. **避免SELECT ***：只取需要的字段，减少IO
5. **考虑连接算法**：
   - Nested Loop Join：适合小表JOIN大表
   - Hash Join（MySQL 8.0.18+）：适合大表JOIN大表
   - Merge Join：适合有序数据

```sql
-- 查看JOIN算法
EXPLAIN FORMAT=TREE SELECT ...\G
-- -> Inner hash join (o.user_id = u.id)
```

**Q10：批量操作如何优化？**

**A**：

**INSERT优化**：
```sql
-- 批量插入（比单条快30倍）
INSERT INTO users VALUES (...), (...), (...);

-- 使用LOAD DATA INFILE（比INSERT快20倍）
LOAD DATA INFILE '/tmp/data.csv' INTO TABLE users;
```

**UPDATE/DELETE优化**：
```sql
-- 分批处理，避免大事务
UPDATE users SET status = 1 WHERE ... LIMIT 1000;
-- 循环执行直到影响行数为0
```

**参数优化**：
```sql
SET GLOBAL innodb_buffer_pool_size = 20G;     -- 增大缓冲池
SET GLOBAL innodb_log_file_size = 2G;         -- 增大日志文件
SET GLOBAL max_allowed_packet = 256M;          -- 增大数据包大小（批量插入）
```

**Q11：如何监控MySQL性能？**

**A**：

```sql
-- 1. 查看QPS/TPS
SHOW STATUS LIKE 'Queries';
SHOW STATUS LIKE 'Com_insert';
SHOW STATUS LIKE 'Com_update';
SHOW STATUS LIKE 'Com_delete';

-- 2. 查看连接数
SHOW STATUS LIKE 'Threads_connected';
SHOW STATUS LIKE 'Threads_running';
SHOW STATUS LIKE 'Max_used_connections';

-- 3. 查看Buffer Pool命中率
SHOW STATUS LIKE 'Innodb_buffer_pool_read%';

-- 4. 查看锁等待
SHOW STATUS LIKE 'Innodb_row_lock_waits';
SHOW STATUS LIKE 'Innodb_row_lock_time_avg';

-- 5. 使用Performance Schema
SELECT * FROM performance_schema.events_waits_summary_global_by_event_name
WHERE COUNT_STAR > 0
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 10;
```

**Q12：SQL执行计划中rows和filtered如何解读？**

**A**：

- **rows**：优化器估算的扫描行数
- **filtered**：过滤后剩余的比例（%）

```sql
-- 示例：
EXPLAIN SELECT * FROM users WHERE status = 1 AND age > 20\G
-- rows: 1000000（users表总行数）
// filtered: 15.00（status=1过滤后剩15%，约15万行）
-- 然后age>20再过滤

-- 如果filtered很低（如1%），说明WHERE条件过滤效果好
-- 如果filtered很高（如90%），说明索引选择性差，可能需要优化
```

---

*此文原创，转载请注明出处。*