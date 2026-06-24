# MySQL 源码深度解析

> **InnoDB 内核**：B+Tree 索引、MVCC 多版本并发控制、锁机制、Redo/Undo/Binlog 三大日志、SQL 优化实战。

---

## 目录

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

*全文 13 章，基于 MySQL 8.0 / InnoDB 编写。*
