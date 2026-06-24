# 数据库源码深度解析：MySQL + PostgreSQL

> **MySQL InnoDB + PostgreSQL 双核**：B+Tree 索引、MVCC 双实现、锁机制、日志系统、VACUUM、SQL 优化、扩展生态。

---

## 目录

### 第一部分：MySQL

- [1. MySQL 架构全景](#1-mysql-架构全景)
- [2. InnoDB 存储引擎架构](#2-innodb-存储引擎架构)
- [3. B+Tree 索引原理](#3-btree-索引原理)
- [4. 事务隔离级别与 MVCC](#4-事务隔离级别与-mvcc)
- [5. 锁机制详解](#5-锁机制详解)
- [6. 三大日志：Redo / Undo / Binlog](#6-三大日志redo--undo--binlog)
- [7. SQL 执行全链路](#7-sql-执行全链路)
- [8. SQL 优化实战](#8-sql-优化实战)
- [9. 慢查询与 Explain 详解](#9-慢查询与-explain-详解)
- [10. 数据库设计原则](#10-数据库设计原则)
- [11. 主从复制与读写分离](#11-主从复制与读写分离)
- [12. 分库分表](#12-分库分表)
- [13. 面试真题与陷阱](#13-面试真题与陷阱)
- [14. InnoDB 进阶机制](#14-innodb-进阶机制)
- [15. MySQL 8.0 关键新特性](#15-mysql-80-关键新特性)
- [16. 设计哲学与权衡](#16-设计哲学与权衡)
- [17. 实战案例](#17-实战案例)

---

## 1. MySQL 架构全景

```mermaid
flowchart TB
    Client["客户端<br/>mysql/JDBC/ORM"]
    
    Client --> Conn["连接器<br/>Connection Pool<br/>认证/权限/连接管理"]
    
    Conn --> Cache["查询缓存 Query Cache<br/>MySQL 8.0 已移除"]
    
    Conn --> Parser["分析器 Parser<br/>词法分析 + 语法分析<br/>生成 AST 语法树"]
    
    Parser --> Optimizer["优化器 Optimizer<br/>索引选择/Join 顺序<br/>★ 决定 SQL 怎么执行"]
    
    Optimizer --> Executor["执行器 Executor<br/>调用存储引擎接口<br/>逐行读取/写入"]
    
    Executor --> InnoDB["InnoDB 引擎<br/>默认, 支持事务"]
    Executor --> MyISAM["MyISAM<br/>不支持事务, 已废弃"]
    Executor --> Memory["Memory<br/>内存表"]
    
    InnoDB --> Disk["磁盘<br/>.ibd 文件<br/>Redo Log / Undo Log"]
```

**Server 层 vs 存储引擎层**：

| 层 | 职责 | 是否可替换 |
|----|------|-----------|
| **Server 层** | 连接、解析、优化、执行、内置函数 | ❌ 固定 |
| **存储引擎层** | 数据存储、索引实现、事务、锁 | ✅ 可插拔（InnoDB/MyISAM/Memory） |

---

## 2. InnoDB 存储引擎架构

### 2.1 内存 + 磁盘双层结构

```mermaid
flowchart TB
    subgraph MEM["内存 Buffer Pool — 缓存热点数据"]
        BP["Buffer Pool<br/>默认 128M, 建议设为物理内存 50-80%"]
        BP --> DATA["数据页 Data Page<br/>16KB"]
        BP --> INDEX["索引页 Index Page<br/>16KB"]
        BP --> UNDO["Undo Page<br/>旧版本数据"]
        
        CHANGE["Change Buffer<br/>缓存非唯一二级索引变更"]
        AHASH["Adaptive Hash Index<br/>自动创建哈希索引"]
        LOG_BUF["Redo Log Buffer<br/>WAL 先写日志"]
    end
    
    subgraph DISK["磁盘"]
        TBS["共享表空间 ibdata1<br/>Undo Log / Double Write"]
        IDB["独立表空间 .ibd<br/>每个表一个文件"]
        REDO["Redo Log<br/>ib_logfile0, ib_logfile1"]
        BINLOG["Binlog<br/>Server 层日志"]
    end
    
    LOG_BUF -->|"事务提交时刷盘"| REDO
    BP -->|"Checkpoint 刷脏页"| IDB
```

### 2.2 InnoDB 页结构

```
InnoDB Page (16KB, 默认):
┌─────────────────────────────┐
│ File Header       (38B)     │  页类型/页号/校验和
│ Page Header       (56B)     │  槽数量/记录数/层级
│ Infimum + Supremum (26B)   │  最小/最大虚拟记录
│ User Records      (变长)    │  ★ 实际数据行
│ Free Space        (变长)    │  空闲空间
│ Page Directory    (变长)    │  槽位数组(二分查找)
│ File Trailer      (8B)      │  校验和
└─────────────────────────────┘
```

**Compact 行格式**：

```
字段1长度 | 字段2长度 | NULL标志位 | 记录头(5B) | 列1数据 | 列2数据 | ...

记录头信息(5B):
  - delete_flag: 1bit (标记删除 → 由 Purge 线程清理)
  - min_rec_flag: 1bit (B+Tree 非叶子页最小记录)
  - n_owned: 4bit (当前记录拥有的记录数, 用于 Page Directory)
  - heap_no: 13bit (记录在页中的位置编号)
  - record_type: 3bit (0=普通, 1=B+非叶子, 2=Infimum, 3=Supremum)
  - next_record: 16bit (★ 下一条记录的相对偏移, 形成单向链表)
```

---

## 3. B+Tree 索引原理

### 3.1 为什么是 B+Tree 而不是其他？

| 数据结构 | 读 O(log n) | 范围查询 | 磁盘友好 | MySQL 选择 |
|----------|------------|----------|----------|-----------|
| **哈希表** | O(1) | ❌ | ❌ | 自适应哈希（辅助） |
| **二叉搜索树** | 可能 O(n) | ✅ | ❌（深度大） | ❌ |
| **AVL/红黑树** | O(log n) | ✅ | ❌（深度大，随机IO） | ❌ |
| **B-Tree** | O(log n) | ⚠️ 需回退 | ✅ | ❌ |
| **B+Tree** | O(log n) | **✅ 叶子链表** | **✅ 高度低** | ★ |

```mermaid
flowchart TB
    subgraph BTREE["B-Tree: 数据存在所有节点"]
        B1["根: 10(数据)"] --> B2["左: 5,8(数据)"]
        B1 --> B3["右: 15,20(数据)"]
    end
    
    subgraph BPLUS["B+Tree: 数据只存叶子节点 ★"]
        P1["根: 10(索引)"] --> P2["左: 5,8(索引)"]
        P1 --> P3["右: 15,20(索引)"]
        P2 --> L1["叶子: 1→3→5→7"]
        P3 --> L2["叶子: 10→12→15→18→20"]
        L1 --> L2
        Note["★ 叶子之间有双向链表<br/>范围查询只需遍历叶子"]
    end
```

**B+Tree 的核心优势**：

1. **高度低**：3 层 B+Tree 可存约 2000 万行数据（假设每行 1KB）
2. **叶子层链表**：`SELECT * FROM t WHERE id BETWEEN 100 AND 200` 只需定位 100 然后沿着链表遍历
3. **非叶子节点只存索引 key**：一个 16KB 页可以存更多索引项，树更矮

### 3.2 聚簇索引 vs 二级索引

```mermaid
flowchart TB
    subgraph CLUSTER["聚簇索引 Primary Key"]
        C1["根页"] --> C2["非叶页"]
        C2 --> C3["叶子页: ★ 存完整行数据"]
    end
    
    subgraph SECONDARY["二级索引 idx_name"]
        S1["根页"] --> S2["非叶页"]
        S2 --> S3["叶子页: ★ 只存主键值"]
    end
    
    S3 -.->|"★ 回表!<br/>用主键值再查<br/>聚簇索引"| C3
    
    COVER["★ 覆盖索引:<br/>SELECT id FROM t WHERE name='Tom'<br/>不需要回表!"]
```

```sql
-- 回表示例:
SELECT * FROM users WHERE name = 'Tom';
-- ① 走 idx_name 二级索引 → 找到 Tom 对应的主键 id=100
-- ② ★ 回表: 用 id=100 再到聚簇索引查完整行 → 额外磁盘 IO!

-- 避免回表:
SELECT id, name FROM users WHERE name = 'Tom';
-- ★ 覆盖索引: id 和 name 都在 idx_name 中, 不需要回表!
```

### 3.3 索引优化黄金法则

| 法则 | 说明 | 反例 |
|------|------|------|
| **最左前缀** | 联合索引 `(a,b,c)` → a / a,b / a,b,c 都能用 | 跳过 a 直接用 b → 失效 |
| **覆盖索引** | SELECT 的列都包含在索引中 | `SELECT *` → 无法覆盖 |
| **避免函数** | `WHERE YEAR(date) = 2024` → 索引失效 | 改为 `WHERE date >= '2024-01-01' AND date < '2025-01-01'` |
| **避免隐式转换** | `WHERE phone = 13800138000` 如果 phone 是 `VARCHAR` → 失效 | `WHERE phone = '13800138000'` |

---

## 4. 事务隔离级别与 MVCC

### 4.1 四种隔离级别

```mermaid
flowchart LR
    RU["READ UNCOMMITTED<br/>读未提交<br/>★ 脏读"]
    RC["READ COMMITTED<br/>读已提交<br/>★ 不可重复读"]
    RR["REPEATABLE READ<br/>可重复读 — InnoDB 默认<br/>★ 幻读(部分解决)"]
    SERIALIZABLE["SERIALIZABLE<br/>串行化<br/>★ 性能最差"]

    RU --> RC --> RR --> SERIALIZABLE
```

| 隔离级别 | 脏读 | 不可重复读 | 幻读 | InnoDB 实现 |
|----------|------|-----------|------|------------|
| READ UNCOMMITTED | ✅ | ✅ | ✅ | 无锁读 |
| READ COMMITTED | ❌ | ✅ | ✅ | 每次语句创建新 ReadView |
| **REPEATABLE READ** | ❌ | ❌ | ⚠️ 部分 | ★ 事务开始时创建 ReadView |
| SERIALIZABLE | ❌ | ❌ | ❌ | 锁读 |

### 4.2 MVCC 多版本并发控制

```mermaid
flowchart TB
    subgraph ROW["一行数据的三个隐藏列"]
        DB_TRX_ID["DB_TRX_ID (6B)<br/>最近修改的事务 ID"]
        DB_ROLL_PTR["DB_ROLL_PTR (7B)<br/>★ 指向 Undo Log 旧版本"]
        DB_ROW_ID["DB_ROW_ID (6B)<br/>行 ID (无主键时用)"]
    end
    
    subgraph UNDO["Undo Log 版本链"]
        V1["版本1: trx_id=100, name='Tom'<br/>roll_ptr=null"]
        V2["版本2: trx_id=200, name='Jerry'<br/>roll_ptr→V1"]
        V3["版本3: trx_id=300, name='Bob'<br/>roll_ptr→V2"]
    end
    
    DB_ROLL_PTR --> V3
    V3 --> V2
    V2 --> V1
```

**MVCC 的核心原理 — ReadView**：

```sql
-- ★ REPEATABLE READ 下:
-- 事务开始时就创建 ReadView, 记录:
--   1. m_ids: 当前活跃事务 ID 列表
--   2. min_trx_id: 活跃事务最小 ID
--   3. max_trx_id: 下一个要分配的事务 ID

-- 可见性判断:
SELECT * FROM users WHERE id = 1;

-- 对于每一行:
-- if trx_id < min_trx_id:        ✅ 可见 (事务在 ReadView 创建前已提交)
-- if trx_id >= max_trx_id:       ❌ 不可见 (事务在 ReadView 创建后才开始)
-- if trx_id in m_ids:            ❌ 不可见 (事务在 ReadView 创建时仍活跃)
--    → 沿 roll_ptr 找 Undo Log 中的可见版本
-- else:                          ✅ 可见 (事务在 ReadView 创建时已提交)
```

**RR vs RC 的 ReadView 区别**：

```sql
-- RR: ReadView 在事务开始时创建一次
--     整个事务期间用同一个 ReadView → 可重复读

-- RC: 每次语句执行时创建新的 ReadView
--     能读到其他事务已提交的修改 → 不可重复读
```

---

## 5. 锁机制详解

### 5.1 锁的类型全景

```mermaid
flowchart TB
    LOCK["InnoDB 锁"]
    
    LOCK --> GRAN["按粒度"]
    GRAN --> TABLE["表锁<br/>LOCK TABLES / MDL"]
    GRAN --> ROW["行锁"]
    GRAN --> PAGE["页锁(BDB)"]
    
    LOCK --> MODE["按模式"]
    MODE --> S["共享锁 S<br/>SELECT ... LOCK IN SHARE MODE"]
    MODE --> X["排他锁 X<br/>SELECT ... FOR UPDATE<br/>UPDATE / DELETE / INSERT"]
    
    LOCK --> ALGO["按算法"]
    ALGO --> RECORD["Record Lock<br/>锁定索引记录"]
    ALGO --> GAP["Gap Lock<br/>锁定索引间隙"]
    ALGO --> NEXT_KEY["Next-Key Lock<br/>★ Record + Gap<br/>InnoDB 默认行锁算法"]
    
    NEXT_KEY --> EXAMPLE["示例: 索引值 5, 10, 15, 20<br/>SELECT * FROM t WHERE id=15 FOR UPDATE<br/>★ 锁住 (10,15] + (15,20)<br/>防止其他事务插入 11-19 之间的值"]
```

### 5.2 Next-Key Lock 解决幻读

```sql
-- 表: id(主键) = 1, 5, 10, 15, 20

-- RR 隔离级别下:
SELECT * FROM t WHERE id = 10 FOR UPDATE;
-- 加锁: Record Lock on id=10
--       Gap Lock on (5,10)
--       Gap Lock on (10,15)
-- ★ 其他事务不能插入 id=6,7,8,9,11,12,13,14!

SELECT * FROM t WHERE id > 10 AND id < 15 FOR UPDATE;
-- 加锁: Gap Lock on (10,15)
-- 其他事务不能插入 id=11,12,13,14
```

### 5.3 死锁排查

```sql
-- ★ 死锁场景:
-- Thread A: UPDATE t SET name='a' WHERE id=1; -- 锁 id=1
--           UPDATE t SET name='b' WHERE id=2; -- 等待 id=2
-- Thread B: UPDATE t SET name='c' WHERE id=2; -- 锁 id=2
--           UPDATE t SET name='d' WHERE id=1; -- 等待 id=1 → 死锁!

-- 查看当前锁:
SELECT * FROM performance_schema.data_locks;

-- 查看死锁日志:
SHOW ENGINE INNODB STATUS\G
-- LATEST DETECTED DEADLOCK 部分

-- 预防:
-- 1. 所有事务按相同顺序加锁
-- 2. 尽量使用索引, 避免锁升级为表锁
-- 3. 缩短事务时间
```

---

## 6. 三大日志：Redo / Undo / Binlog

### 6.1 日志全景

```mermaid
flowchart TB
    subgraph INNODB["InnoDB 引擎层"]
        REDO["Redo Log<br/>★ 物理日志: 记录"页做了什么修改"<br/>循环写, 空间固定<br/>作用: Crash Recovery"]
        UNDO["Undo Log<br/>★ 逻辑日志: 记录"修改前的数据"<br/>作用: 事务回滚 + MVCC"]
    end
    
    subgraph SERVER["Server 层"]
        BINLOG["Binlog<br/>★ 逻辑日志: 记录 SQL 语句<br/>追加写, 无限增长<br/>作用: 主从复制 + 数据恢复"]
    end
    
    TX["事务执行"] --> REDO
    REDO -->|"两阶段提交"| BINLOG
    TX --> UNDO
```

### 6.2 Redo Log — 崩溃恢复

```mermaid
sequenceDiagram
    participant TX as 事务
    participant BP as Buffer Pool<br/>内存
    participant RLB as Redo Log Buffer
    participant RL as Redo Log<br/>磁盘
    participant Data as 数据文件<br/>磁盘

    TX->>BP: UPDATE name='Jerry'
    TX->>RLB: ★ 先写 Redo Log Buffer
    Note over RLB: "将 page 100 offset 50 改为 Jerry"
    
    TX->>TX: COMMIT
    
    Note over TX: 事务提交
    TX->>RL: ★ fsync 刷 Redo Log 到磁盘
    
    Note over BP: ★ 数据页不立即刷盘!
    BP-->>Data: 后台 Checkpoint 时机再刷脏页
    
    Note over Data: ★ 如果此时崩溃:
    Note over Data: 重启后读 Redo Log → 重做未刷盘的操作
```

**WAL (Write-Ahead Logging) 核心思想**：修改数据前先写日志。顺序写 Redo Log 比随机写数据页快 100 倍。

### 6.3 两阶段提交

```mermaid
sequenceDiagram
    participant TX as 事务
    participant Redo as Redo Log
    participant Binlog as Binlog

    TX->>Redo: 1. Prepare 阶段<br/>写入 Redo Log, 标记 prepare
    TX->>Binlog: 2. ★ 写入 Binlog
    TX->>Redo: 3. Commit 阶段<br/>Redo Log 标记 commit

    Note over Redo,Binlog: ★ 两阶段提交保证 Redo 和 Binlog 一致
    Note over Redo,Binlog: 任意阶段崩溃后可以判断是否需要提交
```

**为什么需要两阶段提交？**

```
场景1: Redo 写了但 Binlog 没写 → 主从数据不一致!
场景2: Binlog 写了但 Redo 没 commit → 同上!

两阶段提交解决:
  崩溃时检查: Redo 中有 prepare 标记 + Binlog 完整 → 提交
              Redo 中有 prepare 标记 + Binlog 不完整 → 回滚
```

### 6.4 日志对比

| 维度 | Redo Log | Undo Log | Binlog |
|------|----------|----------|--------|
| 所属层 | InnoDB | InnoDB | **Server 层** |
| 记录内容 | 物理: 页的修改 | 逻辑: 行修改前的值 | 逻辑: SQL 语句 |
| 存储方式 | 循环写, 固定大小 | 随机写, Undo 表空间 | 追加写, 无限增长 |
| 作用 | **Crash Recovery** | **回滚 + MVCC** | **主从复制 + 数据恢复** |
| 刷盘时机 | 事务提交时 | 事务开始时 | 事务提交时 |

---

## 7. SQL 执行全链路

```mermaid
sequenceDiagram
    participant Client as 客户端
    participant Conn as 连接器
    participant Cache as 查询缓存(8.0移除)
    participant Parser as 分析器
    participant Optimizer as 优化器
    participant Executor as 执行器
    participant InnoDB as InnoDB

    Client->>Conn: SELECT * FROM users WHERE id=1
    Conn->>Conn: 认证 + 权限检查
    
    Parser->>Parser: 词法分析: SELECT/FROM/WHERE...
    Parser->>Parser: 语法分析: 生成 AST 语法树
    
    Optimizer->>Optimizer: ★ 选择索引: PRIMARY or idx_name?
    Optimizer->>Optimizer: ★ 确定 Join 顺序
    Optimizer->>Optimizer: 生成执行计划
    
    Executor->>InnoDB: 调用 handler API
    InnoDB->>InnoDB: 检查 Buffer Pool<br/>有 → 直接返回<br/>无 → 读磁盘
    InnoDB->>InnoDB: ★ MVCC 可见性判断
    InnoDB-->>Executor: 返回符合条件的行
    Executor-->>Client: 返回结果集
```

---

## 8. SQL 优化实战

### 8.1 索引优化 Checklist

```sql
-- 1. ★ 最左前缀原则检查
-- 索引: (a, b, c)
SELECT * FROM t WHERE a = 1;              -- ✅ 用索引
SELECT * FROM t WHERE a = 1 AND b = 2;    -- ✅ 用索引
SELECT * FROM t WHERE b = 2 AND c = 3;    -- ❌ 跳过 a, 索引失效!
SELECT * FROM t WHERE a = 1 AND c = 3;    -- ⚠️ 只用 a 列, c 不生效

-- 2. ★ 不等操作符的位置决定范围查询生效范围
-- 索引: (a, b, c)
SELECT * FROM t WHERE a = 1 AND b > 2 AND c = 3;
-- ✅ a=1 精确, b>2 范围, c=3 索引失效(范围后的列不再使用)

-- 3. LIKE 以通配符开头 → 索引失效
SELECT * FROM t WHERE name LIKE '%Tom';    -- ❌
SELECT * FROM t WHERE name LIKE 'Tom%';    -- ✅

-- 4. OR → 两边都要有索引
SELECT * FROM t WHERE a = 1 OR b = 2;
-- ⚠️ a 有索引 b 无索引 → 全表扫描!
```

### 8.2 Join 优化

```sql
-- ★ Join Buffer (Block Nested-Loop) 原理
-- 驱动表 t1 (小表), 被驱动表 t2 (大表)
SELECT * FROM t1 JOIN t2 ON t1.id = t2.t1_id;

-- 优化策略:
-- 1. ★ 小表驱动大表: t1 尽量小
-- 2. ★ 被驱动表关联字段必须有索引: CREATE INDEX idx_t2_t1_id ON t2(t1_id)
-- 3. 避免 SELECT *: 需要的数据列越少 → Join Buffer 能装更多行

-- ★ MySQL 8.0 的 Hash Join (等值连接)
-- 不再用 BNL, 而是对驱动表建 Hash Table
```

### 8.3 分页优化

```sql
-- ❌ 慢: 大偏移量全表扫描
SELECT * FROM users ORDER BY id LIMIT 1000000, 20;
-- 扫描 1000020 行, 丢弃 1000000 行!

-- ✅ 方案1: 基于主键的延迟关联
SELECT * FROM users
WHERE id >= (SELECT id FROM users ORDER BY id LIMIT 1000000, 1)
ORDER BY id LIMIT 20;

-- ✅ 方案2: 游标分页 (适合连续翻页)
SELECT * FROM users WHERE id > 1000000 ORDER BY id LIMIT 20;
-- 前端记录 last_id, 下一页传 last_id
```

### 8.4 count() 优化

```sql
-- ★ count(*)  ≠  count(1)  ≠  count(column)
-- InnoDB 下 count(*) 和 count(1) 性能几乎相同 (MySQL 8.0 优化后)
-- count(column) 不统计 NULL

-- ★ 大表 count 优化:
-- 方案1: 用 EXPLAIN 估算
EXPLAIN SELECT COUNT(*) FROM users;  -- rows 就是近似值

-- 方案2: 单独维护计数表
-- 每次 INSERT → count+1, DELETE → count-1
```

---

## 9. 慢查询与 Explain 详解

### 9.1 开启慢查询

```sql
-- 查看慢查询配置
SHOW VARIABLES LIKE 'slow_query%';
SHOW VARIABLES LIKE 'long_query_time';

-- 开启
SET GLOBAL slow_query_log = ON;
SET GLOBAL long_query_time = 1;       -- > 1 秒的查询记录
SET GLOBAL log_queries_not_using_indexes = ON;  -- 记录未使用索引的查询

-- ★ 生产推荐: 
SET GLOBAL long_query_time = 0.1;     -- > 100ms 就记录
```

### 9.2 Explain 字段全解

```sql
EXPLAIN SELECT * FROM users u
JOIN orders o ON u.id = o.user_id
WHERE u.status = 'ACTIVE' AND o.amount > 100;
```

| 字段 | 含义 | 关注点 |
|------|------|--------|
| **id** | 执行顺序 | id 越大越先执行 |
| **select_type** | 查询类型 | SIMPLE/PRIMARY/SUBQUERY/DERIVED |
| **type** | ★ 访问类型 | **最重点!** |
| **key** | ★ 实际使用的索引 | NULL = 全表扫描 |
| **rows** | ★ 预估扫描行数 | 越小越好 |
| **Extra** | 额外信息 | Using filesort/Using temporary = ⚠️ |

**type 从好到差**：

```
NULL > system > const > eq_ref > ref > range > index > ALL
                                     ★               ★
                                  目标级别         必须避免
```

```sql
-- const: 主键等值查询 (最快)
EXPLAIN SELECT * FROM users WHERE id = 1;           -- type=const ★

-- eq_ref: Join 时用主键/唯一索引关联
EXPLAIN SELECT * FROM u JOIN o ON o.id = u.id;      -- type=eq_ref ★

-- ref: 非唯一索引等值查询
EXPLAIN SELECT * FROM users WHERE name = 'Tom';     -- type=ref ★

-- range: 索引范围扫描
EXPLAIN SELECT * FROM users WHERE id > 100;          -- type=range

-- index: 全索引扫描 (比 ALL 好, 但比 ref 差)
EXPLAIN SELECT id FROM users;                        -- type=index

-- ALL: 全表扫描 (★ 必须优化!)
EXPLAIN SELECT * FROM users WHERE status = 'NEW';    -- type=ALL ❌
```

**Extra 关键值**：

| 值 | 含义 | 对策 |
|----|------|------|
| `Using index` | ★ 覆盖索引，最优 | 不需要改 |
| `Using where` | 正常 | 正常过滤 |
| `Using filesort` | ⚠️ 额外排序，需要优化 | 加索引覆盖 ORDER BY |
| `Using temporary` | ⚠️ 临时表，需要优化 | 加索引规避临时表 |
| `Using index condition` | 索引下推 ICP | 正常优化 |

---

## 10. 数据库设计原则

### 10.1 范式与反范式

| 范式 | 要求 | 优点 | 缺点 |
|------|------|------|------|
| **1NF** | 列不可再分 | 数据原子性 | 冗余 |
| **2NF** | 消除部分函数依赖 | 减少冗余 | 需要 Join |
| **3NF** | 消除传递函数依赖 | 更新无异常 | Join 更多 |
| **反范式** | 适当冗余加速查询 | 减少 Join | 更新需维护 |

```sql
-- ★ 实际项目中: 3NF 保证更新正确, 反范式(冗余)保证查询速度
-- 订单表通常保留当时的价格(冗余), 而不是 Join 去查历史价格
CREATE TABLE orders (
    id BIGINT PRIMARY KEY,
    product_name VARCHAR(200),   -- ★ 冗余! 但查询时不用 Join
    product_price DECIMAL(10,2), -- ★ 冗余当时价格
    created_at DATETIME
);
```

### 10.2 字段类型选择

| 场景 | 推荐 | 避免 |
|------|------|------|
| 主键/Join列 | `BIGINT` | `VARCHAR` (慢) |
| 状态字段 | `TINYINT` + 注释 | `VARCHAR` |
| 金额 | `DECIMAL(18,2)` | `FLOAT` (精度丢失) |
| 时间 | `DATETIME(3)` | `TIMESTAMP` (2038 问题) |
| 大文本 | `TEXT` + 独立存储 | 与其他列一起 |
| IP 地址 | `INT UNSIGNED` + `INET_ATON()` | `VARCHAR(15)` |
| 布尔值 | `TINYINT(1)` | `ENUM` (扩展难) |

### 10.3 索引设计原则

```
1. ★ 必然要建:
   - WHERE 条件列
   - JOIN 关联列
   - ORDER BY / GROUP BY 列

2. ★ 不要建:
   - 区分度低的列 (性别: 只有 M/F → 索引无意义)
   - 频繁更新的列 (索引维护开销)
   - 长字符串 (可用前缀索引)

3. ★ 联合索引 vs 单列索引:
   - 如果有 a=1 AND b=2 和 只有 a=1 的查询:
     建 (a,b) 联合索引, 不要分别建 idx_a 和 idx_b
     (a,b) 能同时满足 a 单列和 a+b 组合
```

---

## 11. 主从复制与读写分离

### 11.1 复制原理

```mermaid
sequenceDiagram
    participant Master as 主库
    participant Binlog as Binlog
    participant IO as IO 线程<br/>从库
    participant Relay as Relay Log<br/>从库
    participant SQL_T as SQL 线程<br/>从库
    participant Slave as 从库数据

    Master->>Binlog: 1. 事务提交, 写入 Binlog
    IO->>Master: 2. 请求 Binlog
    Master-->>IO: 3. 推送 Binlog Event
    IO->>Relay: 4. 写入 Relay Log
    SQL_T->>Relay: 5. 读取 Relay Log
    SQL_T->>Slave: 6. 重放 SQL → 数据同步
```

### 11.2 复制延迟原因与应对

```sql
-- 查看延迟:
SHOW SLAVE STATUS\G
-- Seconds_Behind_Master: 延迟秒数

-- 延迟原因:
-- 1. 从库 SQL 线程单线程 (MySQL 5.6-) → 升级到 5.7+ MTS 并行复制
-- 2. 主库大事务 → 拆分小事务
-- 3. 从库硬件差 → 升级从库

-- 应对延迟:
-- ★ 写后立刻读 → 强制走主库
-- ★ 业务可容忍延迟 → 走从库
```

---

## 12. 分库分表

### 12.1 分片策略

```mermaid
flowchart TB
    Q["数据量 > 2000 万行或<br/>磁盘 > 2TB"] --> VERTICAL{"垂直拆分?"}
    
    VERTICAL -->|"是"| V_SPLIT["按业务: user_db / order_db / product_db<br/>★ 每个库结构不同"]
    VERTICAL -->|"否"| HORIZONTAL{"水平拆分?"}
    
    HORIZONTAL -->|"是"| H_SPLIT["按行分片: user_0 / user_1 / user_2<br/>★ 每个表结构相同"]
    
    H_SPLIT --> SHARD_KEY["分片键选择:<br/>1. user_id → 用户维度查询全在一张表<br/>2. order_id → 订单均匀分布<br/>3. 时间 → 按周期归档"]
```

### 12.2 分片后的问题

| 问题 | 解决方案 |
|------|----------|
| 全局唯一 ID | **Snowflake** / 号段模式 / Redis 自增 |
| 跨分片 Join | 应用层聚合 / 冗余数据 / 全局表 |
| 跨分片事务 | **Seata AT/TCC** / 最终一致性 |
| 扩容 | 一致性 Hash / 双写迁移 |
| 分片键选择 | 选最常用的查询维度 |

---

## 13. 面试真题与陷阱

### 13.1 高频真题

**Q1: InnoDB 为什么用 B+Tree 而不用 B-Tree？**

B+Tree 的数据只存叶子层，非叶子节点存更多索引 → 树高度更低 → 磁盘 IO 更少。且叶子层有双向链表 → 范围查询不需要回到非叶子节点。

**Q2: 什么是回表？怎么避免？**

二级索引叶子存的是主键值，找到主键后还需到聚簇索引查完整行 = 回表。避免：用覆盖索引使 SELECT 的列都在索引中。

**Q3: MVCC 如何实现可重复读？**

RR 级别在事务开始创建 ReadView，后续读用同一个 ReadView 判断可见性，看不到其他事务提交的修改。

**Q4: Next-Key Lock 如何解决幻读？**

`Record Lock + Gap Lock` 既锁记录又锁间隙，阻止其他事务插入满足条件的行。

**Q5: count(*) 怎么优化？**

InnoDB 下不存在 count 缓存（MVCC 每个事务看到的数据不同）。优化：用 EXPLAIN 估算或单独维护计数表。

### 13.2 常见陷阱

```sql
-- 陷阱1: 索引失效 — 隐式类型转换
CREATE INDEX idx_phone ON users(phone);  -- phone 是 VARCHAR

SELECT * FROM users WHERE phone = 13800138000;   -- ❌ 全表扫描!
SELECT * FROM users WHERE phone = '13800138000'; -- ✅ 走索引

-- 陷阱2: 事务中混用存储引擎
START TRANSACTION;
INSERT INTO innodb_table VALUES (1);   -- 事务生效
INSERT INTO myisam_table VALUES (1);   -- ★ 自动提交! MyISAM 不支持事务
ROLLBACK;  -- innodb 回滚, myisam 的数据已提交!

-- 陷阱3: 长事务不提交
START TRANSACTION;
UPDATE users SET status = 'DONE' WHERE id = 1;
-- ... 长时间不提交 ... 
-- ★ Undo Log 无法回收 → 历史版本链暴涨 → 查询变慢
```

---

*全文 17 章，基于 MySQL 8.0 / InnoDB 编写。*


---

# 第二部分：PostgreSQL

> **进程模型多版本并发**：与 MySQL 截然不同的设计哲学——MVCC 无 Undo Log、VACUUM 垃圾回收、四种索引类型、最严谨的 SQL 标准支持。

## 18. PostgreSQL 架构全景

```mermaid
flowchart TB
    Client["客户端<br/>psql / JDBC / ORM"]

    Client --> PM["Postmaster 主进程<br/>★ 监听连接, fork 子进程"]

    PM --> Backend1["Backend 进程 1<br/>★ 一连接一进程<br/>独立地址空间"]
    PM --> Backend2["Backend 进程 2<br/>★ 进程崩溃不影响他人"]
    PM --> BackendN["Backend 进程 N"]

    Backend1 --> SharedMem["共享内存 Shared Memory<br/>★ Shared Buffers<br/>★ WAL Buffers<br/>★ Lock Tables"]
    Backend2 --> SharedMem

    subgraph BG["后台进程"]
        BGWR["Background Writer<br/>刷脏页"]
        WAL["WAL Writer<br/>写 WAL 日志"]
        CHECK["Checkpointer<br/>Checkpoint"]
        AUTO["Autovacuum Launcher<br/>自动垃圾回收"]
        STATS["Stats Collector<br/>统计信息"]
        LOG["Logger<br/>日志"]
    end

    SharedMem --> DISK["磁盘<br/>数据文件 / WAL 文件"]
    BG --> DISK
```

---

## 19. 进程模型 vs MySQL 线程模型

```mermaid
flowchart LR
    subgraph PG["PostgreSQL — 进程模型"]
        PG1["连接1 → Backend PID 1001<br/>独立虚拟内存"]
        PG2["连接2 → Backend PID 1002<br/>独立虚拟内存"]
        PG3["连接崩溃 → 只影响本连接"]
    end

    subgraph MySQL["MySQL — 线程模型"]
        M1["连接1 → Thread 1<br/>共享进程内存"]
        M2["连接2 → Thread 2<br/>共享进程内存"]
        M3["线程崩溃 → CPU 100%风险"]
    end

    PG1 --> SharedMem["通过共享内存通信"]
    PG2 --> SharedMem

    M1 --> Mutex["通过 Mutex/RWLock 同步"]
    M2 --> Mutex
```

| 维度 | PostgreSQL 进程模型 | MySQL 线程模型 |
|------|-------------------|---------------|
| 内存隔离 | ✅ 强（独立地址空间） | ❌ 弱（共享地址空间） |
| 稳定性 | ✅ 单个连接崩不影响全局 | ⚠️ 线程 bug 可能影响全局 |
| 上下文切换 | ⚠️ 重（进程切换） | ✅ 轻（线程切换） |
| 连接池必要性 | **必须**（PgBouncer） | 建议 |
| 共享内存 | 通过 `mmap` + `shmget` | 进程内直接访问 |
| 适合 CPU | 多核 | 少核 |

```sql
-- PostgreSQL 必须用连接池!
-- 每个连接 fork 一个进程(约 5-10MB), 1000 连接 = 5-10GB 内存

-- 方案1: PgBouncer (轻量级, 推荐)
-- pgbouncer.ini:
-- pool_mode = transaction
-- max_client_conn = 1000
-- default_pool_size = 25

-- 方案2: HikariCP (应用层)
# Spring Boot:
spring.datasource.hikari.maximumPoolSize=25
```

---

## 20. 存储引擎与页结构

### 3.1 PostgreSQL 不支持存储引擎切换

与 MySQL 不同，PostgreSQL **只有一个存储引擎**。它不像 InnoDB/MyISAM 那样可切换——所有特性（事务、MVCC、索引）都是内核原生支持，一个引擎提供所有。

### 3.2 页结构

```
PostgreSQL Page (8KB, 默认):
┌─────────────────────────────┐
│ PageHeaderData     (24B)    │  LSN / 校验和 / 空闲空间指针
│ ItemIdData[]       (变长)   │  ★ 行指针数组 (linp)
│ Free Space         (变长)   │  空闲空间
│ Tuple Data         (变长)   │  ★ 实际数据行 (从页尾向前增长)
│ Special Space      (变长)   │  索引页专用
└─────────────────────────────┘

★ 与 MySQL 的关键区别:
  - ItemIdData 指向 Tuple 的偏移量 (类似数组索引)
  - 空闲空间在中间, 数据和指针从两端向中间增长
  - Tuple 可以跨页 (TOAST: 大对象自动分片)
```

### 3.3 Tuple 结构（数据行）

```
Tuple (元组, 即一行数据):
┌──────────────────────────────────────────────────┐
│ xmin (4B)        │ ★ 创建此版本的事务 ID          │
│ xmax (4B)        │ ★ 删除此版本的事务 ID          │
│ cmin (4B)        │ 命令 ID (事务内部)              │
│ cmax (4B)        │ 命令 ID                         │
│ ctid (6B)        │ ★ (页号, 偏移) — 物理位置       │
│ infomask (2B)    │ 标志位 (是否已提交/是否 HOT 更新)│
│ NULL bitmap      │ NULL 列位图                     │
│ user data        │ 实际列数据                       │
└──────────────────────────────────────────────────┘

★ 关键洞察: 每个 Tuple 自带 xmin 和 xmax!
  不需要像 InnoDB 那样去 Undo Log 找版本 —
  版本信息就在 Tuple 里!
```

---

## 21. MVCC 多版本并发控制

### 4.1 PostgreSQL MVCC vs MySQL InnoDB MVCC

```mermaid
flowchart LR
    subgraph MySQL["MySQL InnoDB MVCC"]
        M1["当前行: Jerry"]
        M2["Undo Log: Tom → Bob → Alice"]
        M1 -.->|"roll_ptr"| M2
        M3["★ 版本链在 Undo Log 中<br/>当前行只有最新版本"]
    end

    subgraph PG["PostgreSQL MVCC"]
        P1["Tuple 1: Bob (xmin=300, xmax=400)"]
        P2["Tuple 2: Jerry (xmin=400, xmax=0)<br/>★ 当前版本"]
        P1 --> P2
        P3["★ 所有版本都在数据页中<br/>老版本由 VACUUM 清理"]
    end
```

**核心区别**：

| 维度 | MySQL InnoDB | PostgreSQL |
|------|-------------|-----------|
| 版本存储 | 只有最新版本在行中, 旧版本在 **Undo Log** | **所有版本都在数据页中** |
| 版本查找 | 沿 roll_ptr 遍历 Undo Log 链表 | xmin/xmax 直接判断可见性 |
| 旧版本清理 | Purge 线程 | **VACUUM (手动/自动)** |
| 回滚 | Undo Log 回滚 | **直接标记 xmax, 不回滚页面** |
| 优缺点 | 更新快(只改当前行), 旧版本查找慢 | **更新慢(写新 Tuple), 查询快(不需要回链)** |

### 4.2 可见性判断

```sql
-- PostgreSQL 快照 (Snapshot):
-- 每个事务开始时获取当前快照, 包含:
--   xmin: 最早仍活跃的事务 ID
--   xmax: 下一个要分配的事务 ID
--   xip[]: 当前活跃事务 ID 列表

-- 对 Tuple (xmin=100, xmax=200) 的可见性判断:
-- 
-- 1. xmin 对应的事务已提交且 xmin < snapshot.xmin
--    → ✅ 可见 (创建者在快照前已提交)
--
-- 2. xmin 对应的事务在 xip 列表中(仍活跃)
--    → ❌ 不可见 (创建者还没提交)
--
-- 3. xmax 不为 0:
--    a) xmax 在 xip 中 → ✅ 可见 (删除者还没提交, 行未死)
--    b) xmax 不在 xip 中 → ❌ 不可见 (删除者已提交, 行已死)
--
-- 4. xmin 在 xip 中且 xmin = 当前事务
--    → ✅ 可见 (自己创建的)
```

### 4.3 事务隔离级别

| 级别 | PostgreSQL | MySQL |
|------|-----------|-------|
| READ UNCOMMITTED | ❌ **不支持** (等同于 RC) | ✅ |
| READ COMMITTED | ✅ 默认 | ❌ (默认 RR) |
| REPEATABLE READ | ✅ 无幻读 | ⚠️ 部分幻读 |
| SERIALIZABLE | ✅ **SSI 实现, 真正可串行** | ✅ 锁实现 |

```sql
-- PostgreSQL 的 SERIALIZABLE 最特殊:
-- 不是锁, 而是 Serializable Snapshot Isolation (SSI)
-- 检测"读写依赖环" → 如果冲突, 回滚其中一个事务
-- 代价: 可能回滚(应用需处理 retry)

-- 查看当前隔离级别:
SHOW transaction_isolation;  -- 默认 read committed
```

---

## 22. VACUUM 垃圾回收

VACUUM 是 PostgreSQL 特有的**核心维护操作**。因为 PG 的 MVCC 是直接在数据页中保留所有版本，所以需要定期清理"对任何事务都不可见"的旧版本。

### 5.1 VACUUM vs VACUUM FULL

```mermaid
flowchart TB
    subgraph VACUUM["VACUUM — 标记空闲空间"]
        V1["扫描数据页"] --> V2["标记死 Tuple 为可复用"]
        V2 --> V3["更新 Free Space Map (FSM)"]
        V3 --> V4["★ 不缩表! 空闲空间留待复用"]
        V5["不锁表, 在线操作"]
    end

    subgraph VACFULL["VACUUM FULL — 物理清理"]
        VF1["扫描表"] --> VF2["将活 Tuple 移到新页面"]
        VF2 --> VF3["删除旧页面"]
        VF3 --> VF4["★ 缩表! 回收磁盘空间"]
        VF5["★ 锁表! 不可在线"]
    end
```

```sql
-- 手动 VACUUM
VACUUM users;                         -- 标记空间, 不锁表
VACUUM FULL users;                    -- ★ 物理整理, 锁表! 谨慎使用!
VACUUM ANALYZE users;                 -- 回收 + 更新统计信息

-- ★ 死 Tuple 比例查询:
SELECT schemaname, relname,
       n_live_tup AS live, n_dead_tup AS dead,
       round(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 2) AS dead_ratio
FROM pg_stat_user_tables
WHERE n_dead_tup > 10000
ORDER BY n_dead_tup DESC;

-- dead_ratio > 20% → 需要 VACUUM
```

### 5.2 Autovacuum — 自动垃圾回收

```sql
-- ★ 默认开启! 不需要手动 VACUUM
SHOW autovacuum;  -- on

-- 触发条件 (两个都满足才触发):
-- 1. 死 Tuple 数 > autovacuum_vacuum_threshold + autovacuum_vacuum_scale_factor * 活行数
--    (默认: 50 + 0.2 * live)
-- 2. 距离上次 VACUUM > autovacuum_naptime (默认 1min)

-- ★ 高频更新表调优:
ALTER TABLE orders SET (
    autovacuum_vacuum_scale_factor = 0.05,   -- 5% 死行就触发 (默认 20%)
    autovacuum_vacuum_threshold = 1000
);

-- Autovacuum 监控:
SELECT relname, last_autovacuum, autovacuum_count, n_dead_tup
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;
```

### 5.3 VACUUM FAIL 的后果：事务 ID 回卷

```sql
-- ★ PostgreSQL 的事务 ID 是 32 位 (约 40 亿)
-- 用完会发生 "事务 ID 回卷" → 数据库进入只读模式 → 需要停机维护!

-- 预防: 自动触发 aggressive VACUUM (更频繁, 跳过索引清理豁免)
-- 监控:
SELECT datname, age(datfrozenxid) AS xid_age
FROM pg_database
ORDER BY xid_age DESC;
-- ★ age > 2 亿 → 需要关注!
-- ★ age > 10 亿 → 危险!
```

---

## 23. 四种索引类型

```mermaid
flowchart TB
    INDEX["PostgreSQL 索引"]

    INDEX --> BTREE["B-Tree<br/>默认, 等值+范围<br/>★ 与 InnoDB B+Tree 不同:<br/>不是聚簇索引!<br/>Index 和 Heap 分开存储"]
    INDEX --> HASH["Hash<br/>★ 只有等值查询<br/>PG 10+ 支持 WAL 日志"]
    INDEX --> GIN["GIN 倒排索引<br/>★ 数组、JSONB、全文搜索<br/>一个 key 对应多个行"]
    INDEX --> GIST["GiST 通用搜索树<br/>★ 几何/GIS/全文搜索<br/>平衡树结构"]
    INDEX --> BRIN["BRIN 块范围索引<br/>★ 超大表(时序数据)<br/>索引极小但精度低"]
```

### 6.1 索引速查

| 类型 | 适用 | 大小 | 查询类型 |
|------|------|------|----------|
| **B-Tree** | 通用 (默认) | 中 | `=, >, <, BETWEEN, LIKE 'abc%'` |
| **Hash** | 等值查询 | 小 | `=` |
| **GIN** | `@>`, `?`, `@@` | 大 | 数组包含, JSON键存在, 全文搜索 |
| **GiST** | 几何距离 | 大 | `<<->>`, `@>`, `&&` |
| **BRIN** | 10 亿行时序表 | 极小 | 范围扫描 (精度低) |

```sql
-- GIN: JSONB 查询
CREATE INDEX idx_gin_tags ON products USING GIN (tags);
SELECT * FROM products WHERE tags @> '{"color":"red"}';  -- 包含查询

-- GIN: 全文搜索
CREATE INDEX idx_fts ON articles USING GIN (to_tsvector('english', content));
SELECT * FROM articles WHERE to_tsvector('english', content) @@ to_tsquery('hello & world');

-- BRIN: 10 亿行日志表
CREATE INDEX idx_brin_time ON logs USING BRIN (created_at)
WITH (pages_per_range = 32);  -- 每 32 个页一个摘要
-- 索引只有几十 MB, B-Tree 可能要几百 GB!
```

### 6.2 非聚簇索引 — Secondary Index 直接指向 Heap

```sql
-- ★ 与 MySQL 的关键区别:
-- MySQL: 二级索引 → 聚簇索引 → 数据行 (两次 B+Tree 查找)
-- PostgreSQL: 二级索引 → 直接指向 Heap Tuple (一次 B-Tree 查找)

-- Index Only Scan: 如果所需列都在索引中, 不需要查 Heap
-- 但需要 VACUUM 更新 Visibility Map (标记页中所有 Tuple 都可见)
CREATE INDEX idx_name ON users (name) INCLUDE (email);  -- PG 11+ 覆盖索引

-- 查看索引使用情况:
SELECT schemaname, tablename, indexname,
       idx_scan, idx_tup_read, idx_tup_fetch
FROM pg_stat_user_indexes
ORDER BY idx_scan DESC;
```

---

## 24. WAL 预写日志

### 7.1 WAL 核心流程

```mermaid
sequenceDiagram
    participant TX as 事务
    participant WALBuf as WAL Buffer<br/>内存
    participant WALFile as WAL 文件<br/>磁盘
    participant DataFile as 数据文件

    TX->>WALBuf: 1. 写 WAL Buffer
    WALBuf->>WALFile: 2. WAL Writer 异步刷盘
    TX->>TX: 3. COMMIT
    TX->>WALFile: ★ 4. fsync WAL 到磁盘
    TX->>TX: 5. 返回"事务已提交"
    
    Note over DataFile: ★ 数据页不立即写!
    TX-->>DataFile: Checkpoint 时刷脏页
```

### 7.2 WAL 相关参数

```sql
-- WAL 级别 (控制写入量):
-- minimal: 只保证崩溃恢复 (不能做 PITR 和复制)
-- replica: 默认, 支持复制
-- logical: 支持逻辑复制 + 逻辑解码
ALTER SYSTEM SET wal_level = 'replica';

-- WAL 大小
-- max_wal_size = 1GB (默认)
-- min_wal_size = 80MB
-- ★ 大的 max_wal_size → 减少 Checkpoint 频率 → 提升写入性能
-- ★ 但崩溃恢复时间增加 (需要重做更多 WAL)

-- 同步提交:
-- on: 默认, 事务提交 fsync WAL → 最安全, 稍慢
-- off: 事务提交不 fsync → 最快, 可能丢数据
-- remote_write: 主从复制用
ALTER SYSTEM SET synchronous_commit = 'on';
```

---

## 25. 查询优化器

### 8.1 基于成本的优化器 (CBO)

```sql
-- ★ PostgreSQL 的优化器比 MySQL 更强大:
-- 1. 支持遗传算法 (GEQO) 解决多表 Join 顺序组合爆炸
-- 2. 支持自定义统计 (多列依赖统计)
-- 3. 支持并行查询计划 (多个 Worker 同时扫描)

EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM orders WHERE user_id = 100;
```

### 8.2 EXPLAIN 扫描类型

| 类型 | 含义 | 性能 |
|------|------|------|
| **Seq Scan** | 全表扫描 | ❌ |
| **Index Scan** | 索引扫描 + 回 Heap 取数据 | ✅ |
| **Index Only Scan** | 只扫索引, 不回 Heap | ★ 最优 |
| **Bitmap Scan** | 先扫索引建 Bitmap, 再批量回表 | ✅ 大量行时 |
| **Parallel Seq Scan** | 多 Worker 并行扫描 | ✅ |

```sql
-- Index Only Scan 条件:
-- 1. SELECT 的列都在索引中
-- 2. Visibility Map 标记该页所有 Tuple 对当前事务可见
--    (需要 VACUUM 定期更新 Visibility Map!)

-- 强制使用:
SET enable_seqscan = OFF;  -- 临时禁止全表扫描(开发调试用, 生产勿用)
```

---

## 26. 扩展生态

PostgreSQL 最强大的特性之一是**扩展(Extension)**——通过插件系统添加新功能，无需修改内核。

```sql
-- 查看已安装扩展
SELECT * FROM pg_extension;

-- 安装扩展
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";   -- UUID 生成
CREATE EXTENSION IF NOT EXISTS "pg_stat_statements";  -- SQL 统计
```

### 9.1 核心扩展

| 扩展 | 功能 | 为什么重要 |
|------|------|-----------|
| **PostGIS** | 地理空间数据 | ★ PG 最大杀手级应用, 替代 Oracle Spatial |
| **TimescaleDB** | 时序数据 | 自动分区 + 压缩 + 数据保留策略 |
| **pg_stat_statements** | SQL 统计分析 | 找出最耗时的查询 |
| **pg_partman** | 自动分区管理 | 按时间自动创建/删除分区 |
| **pg_cron** | 定时任务 | 数据库内置 Cron |
| **pgAudit** | 审计日志 | 合规审计 |
| **pgvector** | 向量搜索 | AI Embedding 相似度搜索 |
| **Citus** | 分布式数据库 | 水平分片, 并行查询 |

### 9.2 PostGIS 地理查询

```sql
-- 查找 5 公里内的所有餐厅
SELECT name, ST_Distance(location, 'POINT(121.5 31.2)'::geography) AS dist
FROM restaurants
WHERE ST_DWithin(location, 'POINT(121.5 31.2)'::geography, 5000)
ORDER BY dist;
```

---

## 27. 主从复制

```mermaid
flowchart TB
    Master["主库"] -->|"WAL Sender 进程"| Replica["从库"]

    subgraph Physical["物理复制 (默认)"]
        PHY["WAL 文件传输 → 从库重放<br/>★ 数据库级别, 所有表一起复制"]
    end

    subgraph Logical["逻辑复制 (PG 10+)"]
        LOG["解析 WAL → 发布-订阅<br/>★ 表级别粒度<br/>★ 可跨大版本复制<br/>★ 可过滤表/行"]
    end

    Master --> Physical
    Master --> Logical
```

```sql
-- 查看复制状态
SELECT * FROM pg_stat_replication;

-- 逻辑复制设置:
-- 主库:
CREATE PUBLICATION my_pub FOR TABLE users, orders;
-- 从库:
CREATE SUBSCRIPTION my_sub CONNECTION 'host=master dbname=mydb' PUBLICATION my_pub;
```

---

## 28. PostgreSQL vs MySQL 全面对比

| 维度 | PostgreSQL | MySQL 8.0 |
|------|-----------|-----------|
| **架构** | 进程模型, 一连接一进程 | 线程模型, 共享进程 |
| **MVCC** | Tuple 多版本 + VACUUM 清理 | Undo Log 版本链 + Purge 线程 |
| **索引** | B-Tree/Hash/GiST/GIN/BRIN | B+Tree/Hash/FullText/Spatial |
| **SQL 标准** | ★ 最接近标准, 窗口函数/CTE/LATERAL | 宽松, 8.0 追赶 |
| **JSON** | JSONB (二进制存储, GIN 索引) | JSON (文本存储, 虚拟列索引) |
| **全文搜索** | 内置, `tsvector` + GIN | 内置, ngram parser |
| **扩展性** | ★ 极强 (PostGIS/TimescaleDB/pgvector) | 弱 (靠 MySQL Shell/Plugin) |
| **连接池** | **必须** PgBouncer | 可选 HikariCP |
| **复制** | 物理 + 逻辑 (发布-订阅) | Binlog 复制 + GTID |
| **数据类型** | ★ 丰富 (Array/Hstore/Enum/Range/UUID) | 较少 |
| **GIS** | PostGIS ★ | 内置简单 GIS |
| **开源协议** | PostgreSQL License (类 MIT) | GPL |
| **维护成本** | 需要关注 VACUUM | 不需要 |

### 选型指南

| 场景 | 推荐 | 原因 |
|------|------|------|
| 地理空间数据 | **PostgreSQL + PostGIS** | PostGIS 是行业标准 |
| JSON 文档存储 | **PostgreSQL JSONB** | GIN 索引 + 二进制存储 |
| 复杂分析查询 | **PostgreSQL** | 优化器更强大, CTE/LATERAL/窗口函数 |
| 高并发简单读写 | **MySQL** | 线程模型更轻量 |
| 时序数据 | **PostgreSQL + TimescaleDB** | 专用时序扩展 |
| 全文搜索 | **PostgreSQL** | 内置功能更强 |
| 标准 SQL 合规 | **PostgreSQL** | 最接近 SQL 标准 |
| 嵌入式/IoT | **MySQL / SQLite** | PG 进程模型太重 |

---

## 29. 面试真题与实战

### Q1: PostgreSQL MVCC 与 MySQL MVCC 的核心区别？

PG 在数据页中保留所有 Tuple 版本，通过 xmin/xmax 判断可见性，用 VACUUM 清理死 Tuple。MySQL 只在行中保留最新版本，旧版本存在 Undo Log 中，通过 roll_ptr 链表查找。

**结果**：PG 查询更快（不需要回链），但写入更慢（要写新 Tuple）且需要维护 VACUUM。

### Q2: VACUUM FULL 和 VACUUM 的区别？

`VACUUM` 只标记死 Tuple 空间为可复用，不缩表，不锁表。`VACUUM FULL` 物理整理并释放磁盘空间，但会**排他锁表**，阻塞所有读写。

### Q3: 为什么 PostgreSQL 需要连接池？

每个连接 fork 一个进程，约 5-10MB 内存。1000 个直连 = 5-10GB 内存。必须用 PgBouncer 等连接池复用后端进程。

### Q4: PostgreSQL 没有聚簇索引，查询性能如何保证？

二级索引直接指向 Heap Tuple 物理位置（ctid），不需要像 MySQL 那样回表到聚簇索引。加上 Index Only Scan + Visibility Map，可以实现比 MySQL 覆盖索引更高的性能。

### Q5: PostgreSQL 适合什么场景不适合什么场景？

**适合**：复杂查询、GIS、JSONB、分析型、时序数据、SQL 合规要求高。
**不适合**：极简 CRUD（进程模型偏重）、需要多存储引擎切换、对 MySQL 生态依赖深的场景。

---



*全文 29 章（MySQL 17 + PostgreSQL 12），基于 MySQL 8.0 / PostgreSQL 16 编写。*

---

## 14. InnoDB 进阶机制

### 14.1 Double Write Buffer — 部分写失效的最后防线

InnoDB 页是 16KB，但磁盘扇区通常是 512B 或 4KB。如果一个 16KB 页的写入只完成了一半就断电——这个页就被"写花"了。**Redo Log 恢复不了这种情况**（Redo 记录的是"page 100 改成 Jerry"的物理操作，但如果 page 100 本身已损坏，重做也没用）。

```mermaid
flowchart TB
    BUFFER["Buffer Pool 脏页"] --> DW["1. 先顺序写 Double Write Buffer<br/>128 页 × 16KB = 2MB<br/>★ 顺序写, 很快!"]
    DW --> DISK["2. 再随机写入实际 .ibd 位置<br/>★ 这就是普通写"]
    
    NOTE["★ 崩溃恢复时:<br/>检查 Double Write Buffer 和实际页<br/>如果不一致 → 用 Double Write Buffer 的副本覆盖"]
```

```sql
-- 查看 Double Write 状态
SHOW GLOBAL STATUS LIKE '%dblwr%';
-- Innodb_dblwr_pages_written: double write 写入的页数
-- Innodb_dblwr_writes: double write 操作次数
```

### 14.2 Change Buffer — 写缓存的加速器

当修改二级索引页时，如果该页**不在 Buffer Pool 中**，InnoDB 不会立刻读磁盘获取该页，而是把修改操作缓存在 Change Buffer 中。该页被读取时，再把 Change Buffer 中的修改合并（merge）到页中。

```mermaid
flowchart LR
    UPDATE["UPDATE SET name='Jerry'<br/>WHERE id=1"] --> CHECK{"idx_name 的页<br/>在 Buffer Pool 吗?"}
    CHECK -->|"在"| DIRECT["直接修改页"]
    CHECK -->|"不在"| CB["★ 写入 Change Buffer<br/>不需要读磁盘!"]
    
    READ["后续 SELECT 需要此页"] --> MERGE["★ merge: 读磁盘 + 应用 Change Buffer 中的修改"]
    CB --> MERGE
```

```sql
-- Change Buffer 生效条件:
-- 1. ★ 只对非唯一二级索引生效
--    (唯一索引需要检查唯一性 → 必须读磁盘)
-- 2. 页不在 Buffer Pool 中
-- 3. 写多读少的场景收益最大

-- 查看 Change Buffer 状态
SHOW ENGINE INNODB STATUS\G
-- INSERT BUFFER AND ADAPTIVE HASH INDEX 部分
```

### 14.3 Buffer Pool 三大链表

```mermaid
flowchart TB
    subgraph BP["Buffer Pool"]
        FREE["Free List<br/>★ 空闲页链表<br/>新读入的页从这取"]
        LRU["LRU List<br/>★ 已使用页链表<br/>按最近使用排序<br/>5/8 区分为 young/old"]
        FLUSH["Flush List<br/>★ 脏页链表<br/>按 oldest_modification 排序<br/>Checkpoint 时刷盘"]
    end
    
    DISK_READ["磁盘读页"] --> FREE
    FREE -->|"被使用"| LRU
    LRU -->|"被修改"| FLUSH
    FLUSH -->|"Checkpoint 刷盘"| CLEAN["变干净, 留在 LRU"]
```

```sql
-- Buffer Pool 调优
-- innodb_buffer_pool_size: 总大小 (推荐物理内存 50-80%)
-- innodb_buffer_pool_instances: 拆成多个实例减少竞争 (≥4)
-- innodb_old_blocks_pct: LRU 中 old 区比例 (默认 37%)
-- innodb_old_blocks_time: 进入 old 区后 1 秒内被访问不算"热", 不移入 young

-- innodb_buffer_pool_size = 8G, innodb_buffer_pool_instances = 8
-- → 每个实例 1G, 减少 Buffer Pool mutex 竞争
```

### 14.4 Adaptive Hash Index — B+Tree 上的自动哈希

InnoDB 会监控 B+Tree 的查询模式。如果某个索引被**等值查询频繁访问**，自动为该索引页构建哈希表——把 B+Tree 的 O(log n) 降为 O(1)。

```sql
-- 自动生效, 无需配置
-- 条件: 等值查询频繁, 且查询模式稳定

-- 查看 AHI 状态
SHOW ENGINE INNODB STATUS\G
-- 找到 "INSERT BUFFER AND ADAPTIVE HASH INDEX"
-- Hash table size: 34679, node heap: 2330 buffer(s)

-- ★ 关闭(极少需要, 除非遇到 AHI 锁竞争):
-- SET GLOBAL innodb_adaptive_hash_index = OFF;
```

### 14.5 Purge 线程 — MVCC 的"保洁员"

```mermaid
sequenceDiagram
    participant TX as 事务
    participant UNDO as Undo Log
    participant PURGE as Purge 线程
    participant IDX as 二级索引

    TX->>TX: DELETE FROM t WHERE id=1
    Note over TX: 标记 delete_flag=1<br/>不物理删除!
    TX->>TX: COMMIT
    
    Note over TX: 事务提交后
    Note over UNDO: Undo Log 中的旧版本<br/>不再被任何事务需要
    
    Note over PURGE: ★ 后台 Purge 线程
    PURGE->>UNDO: 清理不再需要的 Undo Log
    PURGE->>IDX: 物理删除标记的记录
    Note over PURGE: 不需要抢锁, 不影响前台查询!
```

```sql
-- ★ 长事务的危害:
-- 事务 A 一直不提交 → ReadView 一直持有 → 
-- Purge 线程无法清理此 ReadView 之后的 Undo Log →
-- Undo 表空间无限膨胀 → 查询遍历长版本链 → 变慢!

-- 监控:
SELECT trx_id, trx_started, TIMESTAMPDIFF(SECOND, trx_started, NOW()) AS sec
FROM information_schema.innodb_trx
WHERE TIMESTAMPDIFF(SECOND, trx_started, NOW()) > 60;
-- ★ 超过 60 秒未提交的事务 → 告警!
```

---

## 15. MySQL 8.0 关键新特性

| 特性 | 8.0 之前 | 8.0 | 收益 |
|------|---------|-----|------|
| **Hash Join** | 仅 BNL | ★ `t1 JOIN t2 ON t1.a=t2.b` 用 Hash Join | 等值 Join 快 10x |
| **降序索引** | 忽略 DESC, 排序需 filesort | ★ `INDEX idx(a ASC, b DESC)` 真正生效 | 避免 filesort |
| **不可见索引** | 只能删除索引测试 | ★ `ALTER INDEX idx INVISIBLE` | 安全测试索引效果 |
| **直方图统计** | 仅基数统计 | ★ `ANALYZE TABLE t UPDATE HISTOGRAM` | 优化器选择更精准 |
| **原子 DDL** | `TRUNCATE` 半路崩溃 → 表损坏 | ★ 原子化, 要么全成功要么全回滚 | 崩溃安全 |
| **NOWAIT / SKIP LOCKED** | 锁等待只能阻塞 | ★ `SELECT ... FOR UPDATE NOWAIT` | 秒杀/抢购场景 |
| **窗口函数** | 靠子查询自 Join | ★ `ROW_NUMBER() OVER(PARTITION BY dept ORDER BY salary DESC)` | 复杂分析一行 SQL |
| **JSON_TABLE** | JSON 只能简单查询 | ★ 将 JSON 数组转为关系表 | 与 SQL 无缝互操作 |
| **角色管理** | 无 | ★ `CREATE ROLE / GRANT ROLE` | 权限管理更灵活 |

```sql
-- 降序索引示例
CREATE INDEX idx_created_desc ON orders(created_at DESC);

-- 不可见索引: 测试删除索引的影响
ALTER INDEX idx_name INVISIBLE;
-- 观察性能... 如果 OK:
ALTER INDEX idx_name VISIBLE;  -- 恢复, 或 DROP INDEX

-- 窗口函数: 每个部门工资排名
SELECT dept, name, salary,
       ROW_NUMBER() OVER(PARTITION BY dept ORDER BY salary DESC) AS rnk
FROM employees;

-- NOWAIT: 秒杀不下单
SELECT stock FROM products WHERE id = 100 FOR UPDATE NOWAIT;
-- 如果被锁 → 立即报错, 不用等!
```

---

## 16. 设计哲学与权衡

### 16.1 为什么选择 B+Tree 作为默认索引？

```
磁盘的特性决定了数据库索引的选择:
  - 顺序读写: 100MB/s
  - 随机读写: 1MB/s  ← ★ 100 倍差距!

B+Tree 的优势:
  1. 高度极低 (3层 ≈ 2000万行) → 磁盘IO次数少
  2. 叶子层双向链表 → 范围查询只需要一次定位 + 顺序遍历
  3. 非叶子节点只存 key → 一个页能存更多索引项 → 树更矮

对比:
  - 哈希: O(1) 但范围查询要全表 → 数据库查询 90% 是范围
  - 红黑树: O(log n) 但高度大 → 100万行需要20层 → 20次随机IO ❌
```

### 16.2 为什么选择 MVCC 而不是简单的读写锁？

```
方案 A: 读写锁
  - 读锁: 查询时加 S 锁 → 其他事务不能写 → 并发差
  - 写锁: 修改时加 X 锁 → 其他事务不能读写 → 吞吐量低

方案 B: MVCC (InnoDB的选择)
  - 读: 基于快照版本, ★ 不需要加锁 → 写不阻塞读
  - 写: 创建新版本, 旧版本保留给正在读的事务
  - 代价: 需要 Undo Log 存储旧版本 → 空间换并发
  - 收益: 读并发提升 100x+
```

### 16.3 为什么需要 WAL（Write-Ahead Logging）？

```
问题: 修改一个页需要:
  1. 读磁盘 (随机IO, 慢)
  2. 修改内存
  3. 写回磁盘 (随机IO, 慢)

WAL 的思路:
  1. 修改内存 (快)
  2. 写 Redo Log (顺序IO, ★ 100x 快!)
  3. 返回"事务已提交"
  4. 后台慢慢将脏页刷回磁盘 (不阻塞事务)

关键洞察: 顺序写 100 倍于随机写
  Redo Log 是顺序写 → 事务提交很快
  数据文件是随机写 → 后台慢慢做
```

### 16.4 为什么 MySQL 8.0 移除查询缓存？

```
问题: 查询缓存失效太频繁!
  - 表上任何修改 → 整表缓存全部失效
  - 高并发写入的表 → 缓存命中率几乎为 0
  - 缓存锁竞争 → MySQL 5.7 默认禁用

替代方案:
  - 应用层缓存 (Redis)
  - 更精细的控制粒度
  - 对只读表/配置表仍有价值 → 用 Redis 替代
```

### 16.5 为什么两阶段提交？

```mermaid
flowchart LR
    A["先写 Redo 后写 Binlog"] --> A1["崩溃在 Binlog 前<br/>主库 Redo 恢复了 → 数据在<br/>从库 Binlog 没有 → 数据丢失 ❌"]
    B["先写 Binlog 后写 Redo"] --> B1["崩溃在 Redo 前<br/>从库 Binlog 有了 → 数据在<br/>主库 Redo 没恢复 → 数据丢失 ❌"]
    
    C["两阶段提交"] --> C1["★ Binlog 写入成功 + Redo Commit → 主从一致 ✅"]
    C --> C2["Binlog 未写入 → Redo 标记 prepare 但没 commit → 回滚 ✅"]
```

### 16.6 设计决策速查

| 决策 | 原因 | 代价 |
|------|------|------|
| B+Tree | 减少磁盘 IO + 范围查询快 | 插入可能页分裂 |
| MVCC | 读写不互斥, 高并发 | Undo Log 空间开销 |
| WAL | 顺序写快 100x | Redo Log 大小需合理配置 |
| 两阶段提交 | 主从一致性 | 多一次 fsync |
| 移除查询缓存 | 高并发写入时缓存失效严重 | 丢失简单场景的加速 |
| Double Write | 防止部分写失效 | 每次写多写一次（但顺序写, 开销小） |

---

## 17. 实战案例

### 案例 1: 慢查询优化 — 从 5 秒到 3 毫秒

```sql
-- 原 SQL (5秒)
SELECT * FROM orders
WHERE YEAR(created_at) = 2024 AND status = 'PAID'
ORDER BY amount DESC LIMIT 20;

-- Explain 输出:
-- type: ALL, rows: 5000000, Extra: Using where; Using filesort

-- ★ 问题: YEAR() 导致索引失效 + SELECT * + filesort
-- 优化:
-- 1. 建联合索引
CREATE INDEX idx_created_status_amount ON orders(created_at, status, amount);

-- 2. 改 SQL
SELECT id, order_no, amount, created_at FROM orders
WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01'
  AND status = 'PAID'
ORDER BY amount DESC LIMIT 20;

-- 重新 Explain:
-- type: range, rows: 25000, Extra: Using index condition
-- 执行时间: 3ms ← ★ 快 1600x!
```

### 案例 2: 长事务导致的性能雪崩

```sql
-- 现象: 应用响应突然从 10ms 飙升到 2s, QPS 骤降

-- 排查:
-- 1. 查看当前事务
SELECT * FROM information_schema.innodb_trx\G
-- 发现一个事务运行了 3600 秒! (1小时前开始的)

-- 2. 查看锁等待
SELECT * FROM performance_schema.data_lock_waits;

-- ★ 根因: 定时任务开了事务忘记提交
-- START TRANSACTION; ...大批量操作... -- 忘了 COMMIT!

-- 3. Kill 长事务
SELECT CONCAT('KILL ', trx_mysql_thread_id, ';')
FROM information_schema.innodb_trx
WHERE TIMESTAMPDIFF(SECOND, trx_started, NOW()) > 300;

-- ★ 预防:
-- innodb_trx: 监控长事务 > 60 秒 → 告警
-- 代码规范: @Transactional 方法不要做 IO/网络调用
```

### 案例 3: Join 从 30 秒到 50 毫秒

```sql
-- 原 SQL (30秒)
SELECT u.name, COUNT(o.id)
FROM users u LEFT JOIN orders o ON u.id = o.user_id
GROUP BY u.id;

-- Explain:
-- users: 10万行, orders: 500万行
-- Using join buffer (Block Nested Loop)  ← ★ 没走索引!

-- 排查: orders.user_id 没索引!
CREATE INDEX idx_orders_user_id ON orders(user_id);

-- 重新 Explain:
-- users: type=index (全索引扫描, 可接受)
-- orders: type=ref, key=idx_orders_user_id  ← ★ 走索引了!
-- 执行时间: 50ms ← ★ 快 600x!
```

### 案例 4: 分页优化

```sql
-- 场景: 后台管理系统, 按创建时间倒序翻页, 共 500 万行

-- ❌ 第 1000 页 (5秒)
SELECT * FROM orders ORDER BY created_at DESC LIMIT 10000, 20;

-- ✅ 方案 1: 游标分页 (10ms)
-- 第 1 页: SELECT * FROM orders ORDER BY created_at DESC, id DESC LIMIT 20;
-- 前端记录最后一条: created_at='2024-01-15 10:00:00', id=1000
-- 第 2 页: 
SELECT * FROM orders 
WHERE (created_at, id) < ('2024-01-15 10:00:00', 1000)
ORDER BY created_at DESC, id DESC LIMIT 20;
-- ★ 需要联合索引: INDEX idx_created_id(created_at, id)
```


