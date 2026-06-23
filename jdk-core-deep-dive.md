# JDK 核心源码深度解析

> JUC 并发、IO/NIO、Thread 底层、反射、泛型、SPI、时间 API、MethodHandle — 八大模块一网打尽。

---

## 目录

### 第一部分：JUC 并发编程
- [1. AQS 抽象队列同步器](#1-aqs-抽象队列同步器)
- [2. ThreadPoolExecutor 线程池](#2-threadpoolexecutor-线程池)
- [3. JUC 工具类](#3-juc-工具类)
- [4. CompletableFuture 异步编程](#4-completablefuture-异步编程)
- [5. ForkJoinPool 分治框架](#5-forkjoinpool-分治框架)

### 第二部分：IO/NIO/AIO
- [6. Java IO 装饰器模式](#6-java-io-装饰器模式)
- [7. NIO 三大组件](#7-nio-三大组件)
- [8. AIO 异步 IO](#8-aio-异步-io)

### 第三部分：Java 底层机制
- [9. Thread 状态机与 wait/notify](#9-thread-状态机与-waitnotify)
- [10. 反射与 JDK 动态代理](#10-反射与-jdk-动态代理)
- [11. Java 泛型深度](#11-java-泛型深度)
- [12. SPI 服务发现机制](#12-spi-服务发现机制)
- [13. java.time 时间 API](#13-javatime-时间-api)
- [14. MethodHandle 与 invokedynamic](#14-methodhandle-与-invokedynamic)

---

# 第一部分：JUC 并发编程

## 1. AQS 抽象队列同步器

AQS（AbstractQueuedSynchronizer）是 JUC 的**基石**。`ReentrantLock`、`CountDownLatch`、`Semaphore`、`ReentrantReadWriteLock` 全部基于它。

### 1.1 AQS 核心结构

```mermaid
flowchart TB
    subgraph AQS["AbstractQueuedSynchronizer"]
        State["state: volatile int<br/>★ 同步状态<br/>0=未锁定, 1=锁定"]
        Head["head: Node<br/>CLH 队列头"]
        Tail["tail: Node<br/>CLH 队列尾"]
    end

    subgraph CLH["CLH 变种队列"]
        H["Head<br/>空节点"] --> N1["Node 1<br/>Thread-1<br/>waitStatus=SIGNAL"]
        N1 --> N2["Node 2<br/>Thread-2<br/>waitStatus=SIGNAL"]
        N2 --> N3["Node 3<br/>Thread-3<br/>waitStatus=0"]
    end

    State -->|"CAS 修改"| Lock["加锁/解锁"]
    Head --> H
    Tail --> N3
```

**核心字段**：

```java
public abstract class AbstractQueuedSynchronizer {
    // ★ 同步状态 (volatile 保证可见性)
    private volatile int state;
    
    // ★ CLH 变种队列 (FIFO 双向链表)
    private transient volatile Node head;
    private transient volatile Node tail;
    
    // 内部节点类
    static final class Node {
        volatile Node prev;
        volatile Node next;
        volatile Thread thread;    // 等待的线程
        volatile int waitStatus;   // CANCELLED/SIGNAL/CONDITION/PROPAGATE
        Node nextWaiter;           // Condition 队列的下一个
    }
}
```

### 1.2 ReentrantLock 加锁流程

```mermaid
sequenceDiagram
    participant T1 as Thread-1
    participant Sync as ReentrantLock.Sync
    participant AQS as AQS
    participant Queue as CLH 队列
    participant T2 as Thread-2

    T1->>Sync: lock()
    Sync->>AQS: CAS state 0→1
    AQS-->>Sync: ✅ 成功
    Sync-->>T1: 获取锁

    T2->>Sync: lock()
    Sync->>AQS: CAS state 0→1
    AQS-->>Sync: ❌ 失败 (state=1)
    Sync->>Sync: tryAcquire() 再次尝试
    AQS-->>Sync: ❌ 仍失败
    Sync->>AQS: ★ acquireQueued(addWaiter())
    
    Note over AQS: 创建 Node, CAS 入队尾
    Queue->>Queue: head → Node-1(Thread-1) → Node-2(Thread-2)
    
    AQS->>T2: LockSupport.park() 阻塞 Thread-2

    Note over T1: ... 执行完毕 ...
    T1->>Sync: unlock()
    Sync->>AQS: state 1→0
    AQS->>Queue: 唤醒 head.next (Thread-2)
    Queue->>T2: LockSupport.unpark() 唤醒
    T2->>Sync: 获取锁
```

**核心源码（简化版）**：

```java
// NonfairSync.lock()
final void lock() {
    // ★ 1. 直接 CAS 抢锁 (非公平)
    if (compareAndSetState(0, 1))
        setExclusiveOwnerThread(Thread.currentThread());
    else
        // 2. CAS 失败 → 走 AQS 标准流程
        acquire(1);
}

// AQS.acquire()
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&           // 再次尝试
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))  // 入队 + 阻塞
        selfInterrupt();
}

// ★ addWaiter: 创建节点, CAS 入队
private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode);
    Node pred = tail;
    if (pred != null) {
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {  // ★ CAS 设置 tail
            pred.next = node;
            return node;
        }
    }
    enq(node);  // CAS 失败或队列为空 → 自旋入队
    return node;
}

// ★ acquireQueued: 入队后自旋抢锁或阻塞
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {  // ★ 前驱是 head 才能抢
                setHead(node);  // 设置自己为新 head
                p.next = null;  // 帮助 GC
                failed = false;
                return interrupted;
            }
            // ★ shouldParkAfterFailedAcquire: 前驱 SIGNAL → park
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())      // ★ LockSupport.park(this)
                interrupted = true;
        }
    } finally {
        if (failed) cancelAcquire(node);
    }
}
```

### 1.3 公平锁 vs 非公平锁

```java
// 公平锁 FairSync.tryAcquire()
protected final boolean tryAcquire(int acquires) {
    if (getState() == 0) {
        // ★ 先检查队列中是否有等待者
        if (!hasQueuedPredecessors() &&
            compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(Thread.currentThread());
            return true;
        }
    }
    return false;
}

// 非公平锁 NonfairSync.tryAcquire()
protected final boolean tryAcquire(int acquires) {
    if (getState() == 0) {
        // ★ 不检查队列, 直接抢!
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(Thread.currentThread());
            return true;
        }
    }
    return false;
}
```

| 维度 | 公平锁 | 非公平锁 |
|------|--------|----------|
| 获取顺序 | FIFO | 随机（可插队） |
| 性能 | 低（需切换线程） | **高**（减少上下文切换） |
| 饥饿 | ❌ 不会 | ⚠️ 可能 |
| 默认 | — | **ReentrantLock 默认** |

### 1.4 AQS 子类速查

| 实现类 | state 含义 | 独占/共享 |
|--------|-----------|----------|
| `ReentrantLock` | 0=未锁, 1+=重入次数 | 独占 |
| `CountDownLatch` | count=剩余计数 | 共享 |
| `Semaphore` | permits=剩余许可 | 共享 |
| `ReentrantReadWriteLock` | 高16位=读, 低16位=写 | 写独占, 读共享 |

---

## 2. ThreadPoolExecutor 线程池

### 2.1 核心参数与执行流程

```mermaid
flowchart TD
    Task["submit(task)"] --> Core{"corePoolSize 满了?"}
    Core -->|"否"| NewThread["创建新线程执行"]
    Core -->|"是"| Queue{"workQueue 满了?"}
    Queue -->|"否"| Enqueue["加入队列等待"]
    Queue -->|"是"| Max{"maxPoolSize 满了?"}
    Max -->|"否"| NewThread2["创建新线程执行"]
    Max -->|"是"| Reject["★ 拒绝策略 RejectedExecutionHandler"]
```

```java
// 7 个核心参数
new ThreadPoolExecutor(
    2,                                     // corePoolSize: 常驻线程数
    4,                                     // maxPoolSize: 最大线程数
    60, TimeUnit.SECONDS,                  // keepAliveTime: 空闲线程存活时间
    new LinkedBlockingQueue<>(100),        // workQueue: 阻塞队列
    Executors.defaultThreadFactory(),      // threadFactory: 线程工厂
    new ThreadPoolExecutor.CallerRunsPolicy() // handler: 拒绝策略
);
```

### 2.2 四种拒绝策略

| 策略 | 行为 | 适用场景 |
|------|------|----------|
| `AbortPolicy`（默认） | 抛 `RejectedExecutionException` | 必须感知拒绝 |
| `CallerRunsPolicy` | 由调用线程执行 | **★ 推荐**: 降级, 减缓提交速度 |
| `DiscardPolicy` | 静默丢弃 | 允许丢失（不推荐） |
| `DiscardOldestPolicy` | 丢弃队头任务, 重新提交 | 允许丢弃旧任务 |

### 2.3 线程池大小公式

```
CPU 密集型: 线程数 = CPU 核数 + 1
I/O 密集型: 线程数 = CPU 核数 × (1 + 平均等待时间 / 平均计算时间)
实际公式:   线程数 = CPU 核数 / (1 - 阻塞系数)     (阻塞系数 ≈ 0.8~0.9)
```

```java
int cpuCores = Runtime.getRuntime().availableProcessors();
// CPU 密集型
int poolSize = cpuCores + 1;
// I/O 密集型
int ioPoolSize = cpuCores * 2;  // 经验值
```

### 2.4 禁止用 Executors 创建线程池

```java
// ❌ 禁止! 队列无界 → OOM
Executors.newFixedThreadPool(10);      // LinkedBlockingQueue(Integer.MAX_VALUE)
Executors.newSingleThreadExecutor();   // 同上
Executors.newCachedThreadPool();       // 线程数 Integer.MAX_VALUE

// ✅ 必须用 ThreadPoolExecutor 显式指定参数
```

### 2.5 线程池状态机

```mermaid
flowchart LR
    RUNNING["RUNNING<br/>接受新任务+处理队列"]
    SHUTDOWN["SHUTDOWN<br/>不接受新任务, 处理队列"]
    STOP["STOP<br/>不接受新任务, 不处理队列, 中断执行中线程"]
    TIDYING["TIDYING<br/>所有任务终止, worker=0"]
    TERMINATED["TERMINATED<br/>terminated() 执行完毕"]

    RUNNING -->|"shutdown()"| SHUTDOWN
    RUNNING -->|"shutdownNow()"| STOP
    SHUTDOWN -->|"队列为空+线程为空"| TIDYING
    STOP -->|"线程为空"| TIDYING
    TIDYING -->|"terminated()"| TERMINATED
```

---

## 3. JUC 工具类

### 3.1 CountDownLatch — 一等多

```java
// ★ 主线程等待所有子线程完成
CountDownLatch latch = new CountDownLatch(3);

for (int i = 0; i < 3; i++) {
    new Thread(() -> {
        doWork();
        latch.countDown();  // ★ 计数 -1
    }).start();
}

latch.await();  // ★ 阻塞直到 count = 0
System.out.println("所有线程完成");

// 源码: 基于 AQS 共享模式
// state = count, countDown() = releaseShared(1) → state--
// await() = acquireSharedInterruptibly → 自旋 + park
```

### 3.2 CyclicBarrier — 互相等待

```java
// ★ 所有线程都到齐后一起继续
CyclicBarrier barrier = new CyclicBarrier(3, () -> {
    System.out.println("都到齐了, 执行回调");  // ★ 最后一个到达的线程执行
});

for (int i = 0; i < 3; i++) {
    new Thread(() -> {
        doPhase1();
        barrier.await();  // ★ 阻塞, 等待其他人
        doPhase2();       // 所有人到齐后一起开始
    }).start();
}

// ★ 可复用: reset() 后可以再次使用 (CountDownLatch 不可复用)
```

### 3.3 Semaphore — 限流

```java
// ★ 同时最多 5 个线程访问资源
Semaphore semaphore = new Semaphore(5);

for (int i = 0; i < 20; i++) {
    new Thread(() -> {
        try {
            semaphore.acquire();  // ★ 获取许可 (阻塞直到有许可)
            doLimitedWork();
        } finally {
            semaphore.release();  // ★ 释放许可
        }
    }).start();
}

// 源码: 基于 AQS 共享模式
// state = permits, acquire() → state--, release() → state++
```

### 3.4 CountDownLatch vs CyclicBarrier vs Semaphore

| 维度 | CountDownLatch | CyclicBarrier | Semaphore |
|------|---------------|---------------|-----------|
| 含义 | 一等多 | 互相等 | 限流 |
| 可复用 | ❌ | ✅ (`reset()`) | ✅ |
| 计数器 | countDown 减 | await 增 | acquire 减 |
| 典型场景 | 主等子完成 | 多线程协作 | 连接池限流 |

---

## 4. CompletableFuture 异步编程

### 4.1 核心方法链

```java
// ★ 异步执行 → 转换 → 组合 → 回调
CompletableFuture.supplyAsync(() -> getUser(1L))       // 1. 异步获取用户
    .thenApply(user -> user.getName())                  // 2. 提取姓名 (同步转换)
    .thenApplyAsync(name -> name.toUpperCase())         // 3. 大写转换 (异步)
    .thenAccept(name -> System.out.println(name))       // 4. 消费结果
    .exceptionally(e -> {                               // 5. 异常处理
        System.err.println("出错了: " + e);
        return null;
    });

// ★ 两个异步结果合并
CompletableFuture<String> cf1 = CompletableFuture.supplyAsync(() -> "Hello");
CompletableFuture<String> cf2 = CompletableFuture.supplyAsync(() -> "World");
cf1.thenCombine(cf2, (s1, s2) -> s1 + " " + s2)        // "Hello World"
   .thenAccept(System.out::println);

// ★ 任意一个完成就继续
cf1.applyToEither(cf2, s -> s.toUpperCase())
   .thenAccept(System.out::println);

// ★ 等待所有完成
CompletableFuture.allOf(cf1, cf2, cf3).join();

// ★ 等待任意一个完成
CompletableFuture.anyOf(cf1, cf2, cf3).join();
```

### 4.2 thenApply vs thenCompose

```java
// thenApply: 同步映射 T → U
CompletableFuture<Integer> cf = 
    CompletableFuture.supplyAsync(() -> "42")
        .thenApply(Integer::parseInt);  // String → Integer

// thenCompose: 异步平铺 (避免 CompletableFuture<CompletableFuture<U>>)
CompletableFuture<User> cf2 =
    CompletableFuture.supplyAsync(() -> 1L)
        .thenCompose(id -> getUserAsync(id));  // ★ 返回 CompletableFuture<User>, 不是嵌套
```

### 4.3 allOf + stream 获取所有结果

```java
List<CompletableFuture<User>> futures = userIds.stream()
    .map(id -> CompletableFuture.supplyAsync(() -> getUser(id)))
    .toList();

// ★ allOf + join 收集结果
List<User> users = futures.stream()
    .map(CompletableFuture::join)   // 阻塞等待每个结果
    .toList();
```

---

## 5. ForkJoinPool 分治框架

### 5.1 核心思想

```mermaid
flowchart TB
    Task["大任务: sum[0..999]"] --> F1["ForkJoinTask 1<br/>sum[0..499]"]
    Task --> F2["ForkJoinTask 2<br/>sum[500..999]"]

    F1 --> F11["sum[0..249]"]
    F1 --> F12["sum[250..499]"]
    F2 --> F21["sum[500..749]"]
    F2 --> F22["sum[750..999]"]

    F11 & F12 --> Join1["join: 合并 F1"]
    F21 & F22 --> Join2["join: 合并 F2"]
    Join1 & Join2 --> Result["最终结果"]

    Note1["★ 工作窃取 Work Stealing<br/>空闲线程偷其他线程队列中的任务"]
```

```java
// ForkJoin 求和
class SumTask extends RecursiveTask<Long> {
    private final long[] arr;
    private final int start, end;
    
    @Override
    protected Long compute() {
        if (end - start <= 1000) {
            // ★ 足够小 → 直接计算
            long sum = 0;
            for (int i = start; i < end; i++) sum += arr[i];
            return sum;
        }
        // ★ 否则 → 分治
        int mid = (start + end) / 2;
        SumTask left = new SumTask(arr, start, mid);
        SumTask right = new SumTask(arr, mid, end);
        left.fork();             // 异步执行左半
        return right.compute() + left.join();  // 右同步 + 等待左
    }
}

// 使用
ForkJoinPool pool = new ForkJoinPool();
SumTask task = new SumTask(arr, 0, arr.length);
long result = pool.invoke(task);
```

**核心优化**：每个工作线程有自己的双端队列。自己的任务从队头取（LIFO），其他线程来偷时从队尾取（FIFO），减少竞争。

---

# 第二部分：IO/NIO/AIO

## 6. Java IO 装饰器模式

### 6.1 BIO 流体系

```mermaid
flowchart TB
    subgraph BYTE["字节流"]
        IS["InputStream"]
        OS["OutputStream"]
        IS --> FIS["FileInputStream<br/>文件输入"]
        IS --> BIS["BufferedInputStream<br/>★ 装饰: 缓冲"]
        OS --> FOS["FileOutputStream"]
        OS --> BOS["BufferedOutputStream<br/>★ 装饰: 缓冲"]
    end

    subgraph CHAR["字符流"]
        R["Reader"]
        W["Writer"]
        R --> FR["FileReader"]
        R --> BR["BufferedReader<br/>★ 装饰: 缓冲+readLine"]
        W --> FW["FileWriter"]
        W --> BW["BufferedWriter"]
    end

    subgraph BRIDGE["桥接"]
        ISR["InputStreamReader<br/>★ 字节→字符"]
        OSW["OutputStreamWriter"]
    end

    BYTE --> BRIDGE
    BRIDGE --> CHAR
```

```java
// 装饰器模式的经典: 层层包装
BufferedReader reader = new BufferedReader(          // 装饰: 缓冲
    new InputStreamReader(                            // 桥接: 字节→字符
        new FileInputStream("file.txt"), "UTF-8"));  // 原始: 字节流

// 等价于 (Files.newBufferedReader 内部做了相同包装)
BufferedReader reader = Files.newBufferedReader(Path.of("file.txt"));
```

### 6.2 BIO 的问题

```java
// BIO 的问题: 一个连接一个线程
ServerSocket server = new ServerSocket(8080);
while (true) {
    Socket client = server.accept();  // ★ 阻塞等待连接
    new Thread(() -> {
        InputStream in = client.getInputStream();
        byte[] buf = new byte[1024];
        in.read(buf);  // ★ 阻塞等待数据
        // ... 处理 ...
    }).start();
}
// 问题: 10000 个连接 → 10000 个线程 → OOM / 上下文切换暴涨
```

---

## 7. NIO 三大组件

### 7.1 Channel + Buffer + Selector

```mermaid
flowchart TB
    Selector["Selector<br/>★ 单线程管理所有 Channel"]
    
    Selector --> Ch1["ServerSocketChannel<br/>注册 ACCEPT"]
    Selector --> Ch2["SocketChannel 1<br/>注册 READ"]
    Selector --> Ch3["SocketChannel 2<br/>注册 READ+WRITE"]
    
    Ch1 --> B1["Buffer<br/>ByteBuffer"]
    Ch2 --> B2["Buffer"]
    Ch3 --> B3["Buffer"]
    
    Note1["★ 核心: 一个线程通过 Selector<br/>可以管理成千上万个连接<br/>不需要为每个连接创建线程"]
```

```java
// NIO 非阻塞服务器
Selector selector = Selector.open();
ServerSocketChannel server = ServerSocketChannel.open();
server.bind(new InetSocketAddress(8080));
server.configureBlocking(false);                    // ★ 非阻塞
server.register(selector, SelectionKey.OP_ACCEPT);  // ★ 注册 ACCEPT

while (true) {
    selector.select();  // ★ 阻塞直到有就绪的 Channel
    
    for (SelectionKey key : selector.selectedKeys()) {
        if (key.isAcceptable()) {
            SocketChannel client = server.accept();
            client.configureBlocking(false);
            client.register(selector, SelectionKey.OP_READ);
        } else if (key.isReadable()) {
            SocketChannel client = (SocketChannel) key.channel();
            ByteBuffer buf = ByteBuffer.allocate(1024);
            client.read(buf);
            // 处理数据...
        }
        iterator.remove();  // ★ 必须手动移除!
    }
}
```

### 7.2 ByteBuffer 三指针

```java
// ★ ByteBuffer 的三个关键指针
ByteBuffer buf = ByteBuffer.allocate(10);  // position=0, limit=10, capacity=10

buf.put((byte) 'H');   // position=1
buf.put((byte) 'i');   // position=2

buf.flip();            // ★ 写模式→读模式: limit=position, position=0

char c1 = (char) buf.get();  // 读 'H', position=1
char c2 = (char) buf.get();  // 读 'i', position=2

buf.compact();         // ★ 压缩: 将未读数据移到开头, position=未读长度

buf.clear();           // 清空(数据不清理, 仅重置指针)
```

```mermaid
flowchart LR
    subgraph WRITE["写模式"]
        W1["H"] --> W2["i"]
        W3["_"] --> W4["_"]
        WP["position=2"]
        WL["limit=10=capacity"]
    end

    subgraph READ["flip() 后 - 读模式"]
        R1["H"] --> R2["i"]
        RP["position=0"]
        RL["limit=2"]
    end

    subgraph COMPACT["compact() 后"]
        C1["H"] --> C2["i"]
        CP["position=2"]
    end

    WRITE -->|"flip()"| READ
    READ -->|"compact()"| COMPACT
```

### 7.3 BIO vs NIO vs AIO

| 维度 | BIO | NIO | AIO (NIO 2) |
|------|-----|-----|------------|
| I/O 模型 | 同步阻塞 | 同步非阻塞 | 异步非阻塞 |
| 线程模型 | 1 连接 : 1 线程 | **1 线程 : N 连接** | 回调 |
| 适用场景 | 低并发、短连接 | **高并发、长连接** | 高并发 I/O |
| Java 实现 | `ServerSocket` | `Selector` + `Channel` | `AsynchronousServerSocketChannel` |
| 数据读取 | `read()` 阻塞 | `select()` 监听就绪 | `read()` + `CompletionHandler` |

---

## 8. AIO 异步 IO

```java
// AIO: 发起 IO 操作后立即返回, 完成时自动回调
AsynchronousServerSocketChannel server = 
    AsynchronousServerSocketChannel.open();
server.bind(new InetSocketAddress(8080));

// ★ 异步 accept: 有连接时自动回调
server.accept(null, new CompletionHandler<AsynchronousSocketChannel, Void>() {
    @Override
    public void completed(AsynchronousSocketChannel client, Void attachment) {
        // 成功: 处理当前连接
        ByteBuffer buf = ByteBuffer.allocate(1024);
        client.read(buf, null, new CompletionHandler<Integer, Void>() {
            @Override
            public void completed(Integer bytesRead, Void att) {
                // ★ 数据读取完成, 自动回调
                buf.flip();
                // 处理 buf...
            }
            @Override
            public void failed(Throwable exc, Void att) { /* ... */ }
        });
        // ★ 继续接受下一个连接
        server.accept(null, this);
    }
    @Override
    public void failed(Throwable exc, Void attachment) { /* ... */ }
});
```

---

# 第三部分：Java 底层机制

## 9. Thread 状态机与 wait/notify

### 9.1 六种线程状态

```mermaid
flowchart TB
    NEW["NEW<br/>线程创建但未 start()"]
    RUNNABLE["RUNNABLE<br/>就绪 + 运行中"]
    BLOCKED["BLOCKED<br/>等待 synchronized 锁"]
    WAITING["WAITING<br/>wait/join/park 无限等待"]
    TIMED_WAITING["TIMED_WAITING<br/>sleep/wait(ms)/join(ms)"]
    TERMINATED["TERMINATED<br/>线程结束"]

    NEW -->|"start()"| RUNNABLE
    RUNNABLE -->|"竞争锁失败"| BLOCKED
    BLOCKED -->|"获取锁"| RUNNABLE
    RUNNABLE -->|"wait()/join()/park()"| WAITING
    RUNNABLE -->|"sleep(ms)/wait(ms)"| TIMED_WAITING
    WAITING -->|"notify/notifyAll/unpark"| RUNNABLE
    TIMED_WAITING -->|"超时/唤醒"| RUNNABLE
    RUNNABLE -->|"执行完毕"| TERMINATED
```

### 9.2 wait/notify 核心规则

```java
// ★ wait/notify 必须在 synchronized 块内调用!
synchronized (lock) {
    while (condition) {     // ★ 必须用 while 而不是 if!
        lock.wait();        // 释放锁 + 阻塞
    }
    // 执行业务...
}

synchronized (lock) {
    lock.notifyAll();  // ★ 优先用 notifyAll 而不是 notify
}

// 为什么用 while 而不是 if?
// 1. 虚假唤醒 (spurious wakeup): JVM 可能无缘无故唤醒线程
// 2. 多消费者竞争: A 被唤醒, 但 B 先拿到锁消费了数据
// while 会重新检查条件, if 不会
```

### 9.3 interrupt 深入

```java
// ★ interrupt() 不会直接停止线程, 而是设置中断标志!
Thread t = new Thread(() -> {
    while (!Thread.currentThread().isInterrupted()) {
        // 正常执行...
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            // ★ sleep/wait/join 收到 interrupt 时抛出异常
            // ★ 并且清除中断标志!
            Thread.currentThread().interrupt();  // ★ 重新设置中断标志
            break;
        }
    }
});
t.start();
t.interrupt();  // 设置中断标志, 如果线程在 sleep 则抛 InterruptedException
```

---

## 10. 反射与 JDK 动态代理

### 10.1 反射核心 API

```java
// ★ 获取 Class 的三种方式
Class<?> clazz1 = User.class;
Class<?> clazz2 = new User().getClass();
Class<?> clazz3 = Class.forName("com.example.User");

// 操作字段
Field field = clazz.getDeclaredField("name");
field.setAccessible(true);             // ★ 绕过 private
String name = (String) field.get(obj);
field.set(obj, "NewName");

// 操作方法
Method method = clazz.getDeclaredMethod("setName", String.class);
method.invoke(obj, "NewName");

// 操作构造器
Constructor<?> ctor = clazz.getDeclaredConstructor(String.class);
User user = (User) ctor.newInstance("Tom");
```

### 10.2 JDK 动态代理原理

```java
// ★ 接口
public interface UserService {
    void save(User user);
}

// ★ 代理处理器
public class LogInvocationHandler implements InvocationHandler {
    private final Object target;
    
    public LogInvocationHandler(Object target) {
        this.target = target;
    }
    
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("Before: " + method.getName());  // ★ 前置增强
        Object result = method.invoke(target, args);         // 委托目标方法
        System.out.println("After: " + method.getName());    // ★ 后置增强
        return result;
    }
}

// ★ 创建代理
UserService proxy = (UserService) Proxy.newProxyInstance(
    UserService.class.getClassLoader(),
    new Class[]{UserService.class},
    new LogInvocationHandler(new UserServiceImpl())
);
proxy.save(user);  // ★ 自动触发 LogInvocationHandler.invoke()
```

**底层原理**：

```java
// Proxy.newProxyInstance 内部:
// 1. 调用 ProxyGenerator.generateProxyClass() 生成字节码
// 2. 通过 native 方法 defineClass0() 加载类
// 3. 生成的代理类大致结构:
public class $Proxy0 extends Proxy implements UserService {
    private static Method m1;  // save 方法
    
    public $Proxy0(InvocationHandler h) { super(h); }
    
    public final void save(User user) {
        // ★ 调用 InvocationHandler.invoke
        h.invoke(this, m1, new Object[]{user});
    }
}
```

---

## 11. Java 泛型深度

### 11.1 类型擦除

```java
// ★ 泛型只在编译期存在, 运行时全部擦除
List<String> strList = new ArrayList<>();
List<Integer> intList = new ArrayList<>();

// 运行时: 两个都是 ArrayList (raw type)
System.out.println(strList.getClass() == intList.getClass()); // ★ true!

// 这就是为什么不能 new T(), new T[], instanceof T
```

### 11.2 PECS 原则

```
Producer Extends, Consumer Super

? extends T: 只能读 (producer), 不能写
? super T:   只能写 (consumer), 读只能读到 Object
```

```java
// ★ Producer Extends: 从集合中读取
public void printAll(List<? extends Number> list) {
    for (Number n : list) {          // ✅ 读: 知道至少是 Number
        System.out.println(n);
    }
    // list.add(10);                 // ❌ 写: 不知道具体是 Integer 还是 Double
}

// ★ Consumer Super: 往集合中写入
public void addNumbers(List<? super Integer> list) {
    list.add(1);                     // ✅ 写: 知道至少能存 Integer
    list.add(2);
    // Integer i = list.get(0);      // ❌ 读: 只知道是 Object
}

// 应用: Collections.copy()
// public static <T> void copy(List<? super T> dest, List<? extends T> src)
// dest 是 consumer (写入) → ? super T
// src 是 producer (读取) → ? extends T
```

### 11.3 协变与逆变

```java
// ★ Java 数组是协变 (Covariant)
Number[] nums = new Integer[10];  // ✅ 编译通过
nums[0] = 3.14;                   // ❌ 运行时 ArrayStoreException!

// ★ 泛型是不可变 (Invariant)
List<Number> nums = new ArrayList<Integer>();  // ❌ 编译错误!
// 正确: 用通配符
List<? extends Number> nums = new ArrayList<Integer>();  // ✅
```

---

## 12. SPI 服务发现机制

### 12.1 SPI 原理

```mermaid
flowchart TB
    Caller["调用方"] --> ServiceLoader["ServiceLoader.load(Service.class)"]
    ServiceLoader --> Config["读取 META-INF/services/<br/>com.example.Service"]
    Config --> Classes["获取所有实现类全限定名"]
    Classes --> Load["Class.forName 加载"]
    Load --> Instantiate["newInstance 实例化"]
    Instantiate --> Return["返回所有实现"]
```

**JDBC 驱动加载就是 SPI 的经典案例**：

```java
// JDBC 4.0+: 不需要 Class.forName("com.mysql.cj.jdbc.Driver")!
// 因为 DriverManager 内部使用了 SPI

// mysql-connector-java.jar 中的文件:
// META-INF/services/java.sql.Driver
// 内容: com.mysql.cj.jdbc.Driver

// ServiceLoader 自动发现并加载
ServiceLoader<Driver> loader = ServiceLoader.load(Driver.class);
for (Driver driver : loader) {
    System.out.println("Found driver: " + driver.getClass().getName());
}
```

### 12.2 SPI 使用示例

```java
// 1. 定义接口
public interface PaymentService {
    void pay(BigDecimal amount);
}

// 2. 实现类 (在另一个 JAR 中)
public class AlipayService implements PaymentService {
    @Override
    public void pay(BigDecimal amount) {
        System.out.println("支付宝支付: " + amount);
    }
}

// 3. META-INF/services/com.example.PaymentService 文件:
// com.example.AlipayService

// 4. 调用方:
ServiceLoader<PaymentService> loader = ServiceLoader.load(PaymentService.class);
for (PaymentService service : loader) {
    service.pay(new BigDecimal("100"));  // 自动发现并调用!
}
```

---

## 13. java.time 时间 API

### 13.1 核心类速查

```java
// ★ 时间线
Instant now = Instant.now();                    // UTC 时间戳: 2024-01-15T10:30:00Z

// ★ 日期(无时间)
LocalDate date = LocalDate.of(2024, 1, 15);     // 2024-01-15

// ★ 时间(无日期)
LocalTime time = LocalTime.of(10, 30, 0);       // 10:30:00

// ★ 日期+时间(无时区)
LocalDateTime dt = LocalDateTime.of(date, time); // 2024-01-15T10:30:00

// ★ 日期+时间+时区
ZonedDateTime zdt = dt.atZone(ZoneId.of("Asia/Shanghai"));
// 2024-01-15T10:30:00+08:00[Asia/Shanghai]

// ★ 计算
LocalDate nextWeek = date.plusWeeks(1);          // 下周
long days = ChronoUnit.DAYS.between(date, nextWeek); // 7
```

### 13.2 不可变 + 线程安全

```java
// ★ java.time 所有类都是 immutable, 线程安全!
LocalDate date = LocalDate.of(2024, 1, 15);
date.plusDays(1);        // 返回新对象, 不修改原对象
System.out.println(date); // 仍然是 2024-01-15

// ★ 与旧 API 互转
Date oldDate = new Date();
Instant instant = oldDate.toInstant();           // Date → Instant
LocalDateTime ldt = LocalDateTime.ofInstant(instant, ZoneId.systemDefault());
Date newDate = Date.from(ldt.atZone(ZoneId.systemDefault()).toInstant());
```

---

## 14. MethodHandle 与 invokedynamic

### 14.1 MethodHandle vs 反射

```java
// ★ 反射: JVM 黑箱, 每次调用都要安全检查
Method method = String.class.getMethod("length");
method.invoke("hello");  // 慢

// ★ MethodHandle: 直接绑定到方法, 接近字节码级别的调用
MethodHandles.Lookup lookup = MethodHandles.lookup();
MethodType mt = MethodType.methodType(int.class);  // 返回 int, 无参数
MethodHandle mh = lookup.findVirtual(String.class, "length", mt);
int len = (int) mh.invokeExact("hello");  // 快! 接近原生调用

// ★ 性能对比: MethodHandle > 反射 > 普通反射
```

### 14.2 invokedynamic 指令

```java
// Lambda 表达式编译后使用 invokedynamic
// 源码: list.forEach(x -> System.out.println(x));
//
// 编译后的字节码:
// invokedynamic #2 <accept, BootstrapMethods #0>
// BootstrapMethods:
//   0: REF_invokeStatic LambdaMetafactory.metafactory(...)
//
// ★ 运行时 JVM 调用 LambdaMetafactory 动态生成实现类
//    而不是编译时就固定
```

**为什么用 invokedynamic？**

```java
// 问题: 如果 Lambda 在编译时就生成匿名内部类:
//   1. 每个 Lambda → 一个 .class 文件 → 类爆炸
//   2. 无法改变实现策略

// invokedynamic 解决方案:
//   1. 编译时: 只记录 Bootstrap Method
//   2. 运行时: Bootstrap Method 决定如何实现 Lambda
//      - 可以生成内部类
//      - 可以使用 MethodHandle
//      - 可以缓存结果
//      - JVM 可以自由优化实现
```

### 14.3 反射/MethodHandle/VarHandle 对比

| 维度 | 反射 | MethodHandle | VarHandle (JDK 9+) |
|------|------|-------------|-------------------|
| 调用方式 | `invoke()` | `invokeExact()` / `invoke()` | `get()` / `set()` / `compareAndSet()` |
| 安全检查 | 每次都有 | 只在创建时 | 只在创建时 |
| 性能 | 慢 | **接近原生** | **接近原生** |
| 访问字段 | ✅ | ✅ | **✅ + CAS 原子操作** |
| 签名检查 | 运行时宽松 | **编译时严格(invokeExact)** | 编译时严格 |

---

*全文 14 章，基于 JDK 8/11/17/21 源码编写。*
