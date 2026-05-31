# Java 并发编程进阶（JUC）

## 📖 本章导读

### 学习目标
通过本章学习，你将掌握：
- ✅ AQS 同步器框架核心原理与模板方法
- ✅ ReentrantLock 公平/非公平锁实现与 Condition
- ✅ ConcurrentHashMap 分段锁演进与红黑树
- ✅ ThreadPoolExecutor 核心参数与任务调度机制
- ✅ CompletableFuture 异步编排与回调
- ✅ ForkJoinPool 工作窃取算法详解
- ✅ 7 种同步工具类的使用场景与原理

### 核心概念
**JUC（java.util.concurrent）** 是 Doug Lea 设计的 Java 并发工具包，提供了从基础锁、并发集合到高级同步工具和线程池的完整体系。其核心是 **AQS（AbstractQueuedSynchronizer）**，一个基于 CLH 锁变体实现的同步器框架，支撑了 ReentrantLock、Semaphore、CountDownLatch 等大部分同步工具。

### 知识地图
```
JUC 体系架构
├── AQS 同步器框架
│   ├── CLH 锁变体：双向队列 + state 状态
│   ├── 独占模式：acquire/release（ReentrantLock）
│   ├── 共享模式：acquireShared/releaseShared（Semaphore）
│   └── ConditionObject：等待/通知机制
├── 显式锁
│   ├── ReentrantLock：公平/非公平、可中断、超时
│   ├── ReadWriteLock：读写分离
│   ├── StampedLock：乐观读（JDK 8）
│   └── LockSupport：park/unpark 原语
├── 并发集合
│   ├── ConcurrentHashMap：分段锁/红黑树
│   ├── ConcurrentLinkedQueue：CAS + 无锁
│   ├── CopyOnWriteArrayList：写时复制
│   ├── ConcurrentSkipListMap：跳表
│   └── BlockingQueue 接口：7 种实现
├── 线程池
│   ├── ThreadPoolExecutor：核心参数与拒绝策略
│   ├── ScheduledThreadPoolExecutor：定时调度
│   ├── ForkJoinPool：工作窃取、分治
│   └── Executors：4 种快捷创建
├── 异步编程
│   ├── Future/Callable：阻塞获取
│   ├── CompletableFuture：回调编排
│   └── CompletionService：批量任务
├── 同步工具
│   ├── CountDownLatch：一次性栅栏
│   ├── CyclicBarrier：可循环栅栏
│   ├── Semaphore：信号量限流
│   ├── Exchanger：线程间数据交换
│   └── Phaser：分阶段同步
└── 原子类
    ├── 基础类型：AtomicInteger/AtomicLong
    ├── 引用类型：AtomicReference/AtomicStampedReference
    ├── 数组：AtomicIntegerArray
    └── 字段更新器：AtomicIntegerFieldUpdater
```

### 常见误区
❌ **误区 1**：`volatile` 可以保证线程安全
✅ **真相**：volatile 保证可见性和有序性，但不保证原子性。`count++` 这类复合操作仍需加锁或使用 AtomicInteger。

❌ **误区 2**：ConcurrentHashMap 读操作完全不加锁
✅ **真相**：JDK 8 中读操作通常不加锁，但遇到了链表树化或扩容时，`ForwardingNode` 会让读线程协助扩容或通过 `find()` 在新数组中查找。

❌ **误区 3**：`newSingleThreadExecutor()` 创建的线程池保证单线程执行
✅ **真相**：默认使用无界 LinkedBlockingQueue，如果任务提交速度超过处理速度，会导致 OOM。且如果不主动 shutdown，线程会一直存活。

❌ **误区 4**：`ForkJoinPool.commonPool()` 适用于所有并行任务
✅ **真相**：公共池的并行度默认为 `Runtime.availableProcessors() - 1`，阻塞型任务会消耗所有线程，其他任务无线程可用。阻塞型任务应使用自定义 ForkJoinPool。

### 实战建议
1. **优先使用 JUC 而非 synchronized**：ReentrantLock 支持超时、中断、公平性等特性
2. **线程池必须指定拒绝策略**：避免任务无限制堆积导致 OOM
3. **CompletableFuture 默认线程池**：使用自定义线程池而非默认的 ForkJoinPool.commonPool()
4. **ConcurrentHashMap 避免 size()**：JDK 8 的 size() 需遍历所有桶，用 mappingCount() 替代
5. **CopyOnWriteArrayList 适用读多写少**：每次写操作复制整个数组，写频繁时性能差
6. **ThreadLocal 用完 remove()**：避免线程池复用导致的内存泄漏
7. **StampedLock 乐观读不可重入**：适用读多写少的场景

---

## 1. AQS（AbstractQueuedSynchronizer）

### 1.1 AQS 核心思想

AQS 是 JUC 的基石，ReentrantLock、Semaphore、CountDownLatch、ReentrantReadWriteLock 等全部基于 AQS 实现。

```java
// AQS 的核心：
// 1. state：volatile int 类型的状态（锁重入次数、信号量许可数等）
// 2. CLH 变体队列：双向 FIFO 队列，管理等待线程
// 3. 模板方法模式：子类实现 tryAcquire/tryRelease 等方法

public abstract class AbstractQueuedSynchronizer
    extends AbstractOwnableSynchronizer {

    // 核心状态 —— volatile 保证可见性
    private volatile int state;

    // CAS 操作更新 state
    protected final boolean compareAndSetState(int expect, int update) {
        return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
    }

    // CLH 队列头尾指针
    private transient volatile Node head;
    private transient volatile Node tail;
}
```

**CLH 锁与 AQS 的 CLH 变体：**

```
原始 CLH 锁：自旋 + 单向链表（隐式队列）
                          ↓
AQS 的 CLH 变体：自旋 + 阻塞 + 双向链表 + 前驱轮询

Node 结构：
┌─────────────────────────────────────────────────┐
│  Node                                            │
│  ├── prev: Node          ← 前驱节点               │
│  ├── next: Node          → 后继节点               │
│  ├── thread: Thread      ← 等待线程               │
│  ├── waitStatus: int     ← 状态 (CANCELLED/SIGNAL/...)
│  │   - CANCELLED(1): 取消等待                       │
│  │   - SIGNAL(-1): 需要唤醒后继                       │
│  │   - CONDITION(-2): 在条件队列                      │
│  │   - PROPAGATE(-3): 共享模式传播                    │
│  └── nextWaiter: Node   ← 条件队列后继               │
└─────────────────────────────────────────────────┘
```

### 1.2 模板方法模式

AQS 使用模板方法模式，子类需要实现以下方法：

```java
// 需要子类实现的方法（AQS 中直接抛 UnsupportedOperationException）：
protected boolean tryAcquire(int arg);       // 独占模式——尝试获取
protected boolean tryRelease(int arg);       // 独占模式——尝试释放
protected int tryAcquireShared(int arg);     // 共享模式——尝试获取（返回剩余许可数）
protected boolean tryReleaseShared(int arg); // 共享模式——尝试释放
protected boolean isHeldExclusively();       // 是否独占模式持有

// 模板方法（AQS 已实现，子类不可重写）：
public final void acquire(int arg);          // 独占获取（忽略中断）
public final void acquireInterruptibly(int arg); // 独占获取（可中断）
public final boolean tryAcquireNanos(int arg, long nanos); // 超时获取
public final boolean release(int arg);       // 独占释放
public final void acquireShared(int arg);    // 共享获取
public final boolean releaseShared(int arg); // 共享释放
```

### 1.3 独占模式获取（acquire）

```java
// acquire 是 AQS 的模板方法，子类不可重写
public final void acquire(int arg) {
    if (!tryAcquire(arg)) {                    // 1. 子类实现：尝试获取
        Node node = addWaiter(Node.EXCLUSIVE); // 2. 获取失败，创建独占节点入队
        boolean interrupted = false;
        for (;;) {                             // 3. 自旋 —— 阻塞前至少尝试一次
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) { // 4. 前驱是 head，再试一次
                setHead(node);                 // 成为新的 head
                p.next = null;                 // 帮助 GC
                return;                        // 获取成功
            }
            // 5. 获取失败，进入等待状态
            //    shouldParkAfterFailedAcquire：将前驱的 waitStatus 设为 SIGNAL
            //    parkAndCheckInterrupt：调用 LockSupport.park() 阻塞
            if (shouldParkAfterFailedAcquire(p, node))
                interrupted |= parkAndCheckInterrupt();
        }
    }
}
// 注意：acquire 忽略中断，即使线程被中断也继续等待
// acquireInterruptibly 会响应中断，抛出 InterruptedException
```

**入队过程图解：**

```
初始化 head = tail = dummy node（延迟初始化）
  head
   ↓
  Node(thread=null, waitStatus=0)
   ↑
  tail

第一个竞争线程 T1 入队：
  head                  tail
   ↓                     ↓
  Node(dummy) ←───→ Node(T1, waitStatus=0)
                        ↑
                     predecessor=head → tryAcquire → 成功则出队

第二个竞争线程 T2 入队：
  head                                        tail
   ↓                                           ↓
  Node(dummy) ←───→ Node(T1) ←───→ Node(T2)
                                    ↑
                              predecessor=Node(T1) 非 head → park()
```

### 1.4 独占模式释放（release）

```java
public final boolean release(int arg) {
    if (tryRelease(arg)) {                     // 1. 子类实现：尝试释放
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);                // 2. 唤醒 head 的后继
        return true;
    }
    return false;
}

private void unparkSuccessor(Node node) {
    int ws = node.waitStatus;
    if (ws < 0) compareAndSetWaitStatus(node, ws, 0);
    // 从 tail 向前找第一个未被取消的节点
    // 为什么从 tail 向前？——入队时是先设置 prev 再 CAS tail，从后往前保证遍历完整
    Node s = node.next;
    if (s == null || s.waitStatus > 0) { // CANCELLED
        s = null;
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0) s = t;
    }
    if (s != null)
        LockSupport.unpark(s.thread);          // 唤醒线程
}
```

### 1.5 ConditionObject

```java
// Condition 提供了类似 Object.wait/notify 的等待通知机制
// 但 Object 的等待集只有 1 个，Condition 可以创建多个

public class ConditionObject implements Condition {
    // 条件队列是单向链表（使用 Node.nextWaiter）
    private transient Node firstWaiter;
    private transient Node lastWaiter;

    // await() —— 释放锁并等待
    public final void await() throws InterruptedException {
        // 1. 创建 CONDITION 状态节点，加入条件队列
        // 2. 完全释放锁（保存释放前的 state 值）
        // 3. 阻塞（park）
        // 4. 被 signal 唤醒后，重新竞争锁（acquireQueued）
    }

    // signal() —— 转移一个等待节点到同步队列
    public final void signal() {
        // 1. 检查当前线程是否持有锁（isHeldExclusively）
        // 2. 将条件队列头节点转移到同步队列
        // 3. 唤醒该线程（unpark）
    }
}

// Object.wait/notify vs Condition.await/signal
// ┌───────────────────┬──────────────────────────────┐
// │ Object            │ Condition                     │
// ├───────────────────┼──────────────────────────────┤
// │ 只有一个等待集     │ 可以创建多个 Condition          │
// │ 必须在 synchronized 块内 │ 必须持有对应的 Lock         │
// │ 中断响应不可控     │ 支持超时、可中断、不可中断等多种形式 │
// │ notify 随机唤醒    │ signal 可以选择特定条件队列     │
// └───────────────────┴──────────────────────────────┘
```

---

## 2. ReentrantLock

### 2.1 核心结构

```java
public class ReentrantLock implements Lock {
    // 内部 Sync 继承 AQS
    private final Sync sync;

    // Sync 抽象类 —— 定义 tryRelease 的共同逻辑
    abstract static class Sync extends AbstractQueuedSynchronizer {
        abstract void lock();  // 子类实现不同

        // 非公平 tryAcquire 的通用实现
        final boolean nonfairTryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                // CAS 抢锁 —— 非公平的关键：不管队列是否为空，先抢再说
                if (compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            } else if (current == getExclusiveOwnerThread()) {
                // 重入
                int nextc = c + acquires;
                if (nextc < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }

        // 释放锁 —— 公平/非公平共享同一实现
        protected final boolean tryRelease(int releases) {
            int c = getState() - releases;
            if (Thread.currentThread() != getExclusiveOwnerThread())
                throw new IllegalMonitorStateException();
            boolean free = false;
            if (c == 0) {     // 完全释放（重入计数归零）
                free = true;
                setExclusiveOwnerThread(null);
            }
            setState(c);      // 非完全释放只更新 state
            return free;
        }
    }

    // 公平锁
    static final class FairSync extends Sync {
        final void lock() {
            acquire(1);  // 直接调用 AQS 的 acquire
        }

        protected final boolean tryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                // 公平锁关键：hasQueuedPredecessors() 检查队列中是否有等待线程
                // 如果有等待线程且不是当前线程，则不能抢锁
                if (!hasQueuedPredecessors() &&
                    compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            } else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0) throw new Error();
                setState(nextc);
                return true;
            }
            return false;
        }
    }

    // 非公平锁
    static final class NonfairSync extends Sync {
        final void lock() {
            // 非公平锁一上来先 CAS 抢一次锁（插队）
            if (compareAndSetState(0, 1))
                setExclusiveOwnerThread(Thread.currentThread());
            else
                acquire(1);  // 抢失败才走正常 acquire
        }

        protected final boolean tryAcquire(int acquires) {
            return nonfairTryAcquire(acquires);  // 调用 Sync 的通用实现
        }
    }
}
```

### 2.2 公平锁 vs 非公平锁

```java
// 公平锁（FairSync）：
// 1. lock() 直接调用 acquire(1)
// 2. tryAcquire 中先检查 hasQueuedPredecessors()
// 3. 线程严格按照到达顺序获取锁（FIFO）
// 吞吐量：低（约 1/3 ~ 1/2 非公平锁）

// 非公平锁（NonfairSync）：
// 1. lock() 先 CAS 抢一次锁（插队）
// 2. 失败后再走 acquire → tryAcquire(nonfairTryAcquire)
// 3. nonfairTryAcquire 中不检查 hasQueuedPredecessors()
// 吞吐量：高（避免线程挂起/唤醒的上下文切换）

// 为什么非公平锁吞吐量更高？
// 假设线程 A 持有锁，线程 B 在等待队列中 park
// 线程 A 释放锁 → unpark 线程 B → 线程 B 需要上下文切换来 resume
// 在 线程 B 恢复运行的这个时间窗口内，线程 C 来插队
// 线程 C 不需要 park/unpark，直接 CAS 获取锁 → 减少了一次上下文切换

// 适用场景：
// 公平锁：业务要求严格公平，避免线程饥饿（如交易系统）
// 非公平锁：大多数场景，性能优先（默认选择）
```

### 2.3 Condition 使用示例

```java
public class BoundedBuffer {
    private final ReentrantLock lock = new ReentrantLock();
    private final Condition notFull  = lock.newCondition();
    private final Condition notEmpty = lock.newCondition();

    private final Object[] buffer = new Object[100];
    private int putIndex, takeIndex, count;

    public void put(Object x) throws InterruptedException {
        lock.lock();
        try {
            while (count == buffer.length) {
                notFull.await();    // 缓冲区满，等待"非满"条件
            }
            buffer[putIndex] = x;
            if (++putIndex == buffer.length) putIndex = 0;
            count++;
            notEmpty.signal();     // 唤醒等待"非空"条件的线程
        } finally {
            lock.unlock();
        }
    }

    public Object take() throws InterruptedException {
        lock.lock();
        try {
            while (count == 0) {
                notEmpty.await();  // 缓冲区空，等待"非空"条件
            }
            Object x = buffer[takeIndex];
            if (++takeIndex == buffer.length) takeIndex = 0;
            count--;
            notFull.signal();      // 唤醒等待"非满"条件的线程
            return x;
        } finally {
            lock.unlock();
        }
    }
}

// 为什么用 while 而不是 if 检查条件？
// 虚假唤醒（spurious wakeup）：
// 线程从 await 返回时，条件可能仍不满足
// 用 while 循环保证在条件满足之前不会继续执行
```

### 2.4 ReentrantLock vs synchronized 对比

```java
// 1. 显式 vs 隐式
//    synchronized：JVM 内置，自动加解锁（异常时自动释放）
//    ReentrantLock：API 层，需手动 lock/unlock（必须在 finally 中 unlock）

// 2. 功能对比
//    ┌─────────────────┬──────────────────┬──────────────────────┐
//    │ 特性             │ synchronized     │ ReentrantLock        │
//    ├─────────────────┼──────────────────┼──────────────────────┤
//    │ 可中断          │ ❌ 不能中断等待   │ ✅ lockInterruptibly  │
//    │ 超时            │ ❌ 不支持         │ ✅ tryLock(timeout)   │
//    │ 公平性          │ ❌ 非公平         │ ✅ 可选公平/非公平    │
//    │ 多个条件        │ ❌ 一个等待集     │ ✅ 多个 Condition     │
//    │ 性能            │ JDK 8+ 已优化     │ 高（不同场景各有优势） │
//    │ 便捷性          │ ✅ 简洁           │ ❌ 需手动释放         │
//    └─────────────────┴──────────────────┴──────────────────────┘

// 3. JDK 6 之后 synchronized 经过锁升级优化（偏向锁 → 轻量级锁 → 重量级锁）
//    在低竞争场景下 synchronized 性能甚至优于 ReentrantLock
//    ReentrantLock 的优势在高竞争和复杂同步场景

// 选择建议：
// - 简单同步：优先 synchronized（简洁、不易出错）
// - 需要超时/中断/公平：ReentrantLock
// - 多个等待条件：ReentrantLock 的 Condition
// - 高竞争且追求吞吐量：ReentrantLock + 非公平模式
```

### 2.5 StampedLock（JDK 8）

```java
// StampedLock 是 JDK 8 引入的新锁，三种模式：
// 1. 写锁（独占）—— 类似 ReentrantLock
// 2. 读锁（共享）—— 类似 ReadWriteLock 的读锁
// 3. 乐观读（关键特性）—— 不加锁，后续验证

// 核心优势：乐观读完全不加锁，读读之间不阻塞，读写也不阻塞

public class StampedLockDemo {
    private double x, y;
    private final StampedLock sl = new StampedLock();

    // 写操作 —— 获取写锁
    void move(double deltaX, double deltaY) {
        long stamp = sl.writeLock();        // 获取写锁
        try {
            x += deltaX;
            y += deltaY;
        } finally {
            sl.unlockWrite(stamp);           // 释放写锁
        }
    }

    // 读操作 —— 乐观读（完全无锁）
    double distanceFromOrigin() {
        long stamp = sl.tryOptimisticRead(); // 1. 获取时间戳（乐观读标记）
        double currentX = x;
        double currentY = y;
        if (!sl.validate(stamp)) {          // 2. 验证期间是否有写操作
            stamp = sl.readLock();           // 3. 被写过，升级为悲观读锁
            try {
                currentX = x;
                currentY = y;
            } finally {
                sl.unlockRead(stamp);
            }
        }
        return Math.sqrt(currentX * currentX + currentY * currentY);
    }
}

// StampedLock 注意事项：
// 1. 不可重入（每次获取都返回新的 stamp）
// 2. 不支持 Condition
// 3. 悲观读锁和写锁都不会进行条件队列的管理
// 4. 适用场景：读多写少，读操作耗时短

// JDK 8 引入的另一个改良：LongAdder（替代 AtomicLong 高竞争场景）
// LongAdder 将单一 value 拆分为多个 cell，减少 CAS 冲突
```

---

## 3. ConcurrentHashMap

### 3.1 JDK 7 实现（分段锁）

```java
// 分段锁（Segment）：
// HashTable 对整个表加锁 → 并发度低
// ConcurrentHashMap 将表分成多个 Segment，每个 Segment 独立加锁

// JDK 7 结构：
// ┌──────────────────────────────────────────────┐
// │  ConcurrentHashMap                            │
// │  ├── segments: Segment[]     ← 默认 16 个     │
// │  │   ├── Segment[0]:                         │
// │  │   │   ├── count                           │
// │  │   │   ├── lock (继承 ReentrantLock)        │
// │  │   │   └── HashEntry[] table  ← 真正存储    │
// │  │   ├── Segment[1]:                         │
// │  │   └── ...                                 │
// │  └── concurrencyLevel: 16    ← 最大并发度    │
// └──────────────────────────────────────────────┘

// 写操作：只锁对应的 Segment（其他 Segment 可以并行写）
// size()：先无锁累加，如果结果不一致再加锁
// 缺点：定位两次（先找 Segment，再找 HashEntry）
```

### 3.2 JDK 8 实现（CAS + synchronized）

```java
// JDK 8 重写了 ConcurrentHashMap，放弃 Segment，使用：
// 1. 数组 + 链表 + 红黑树（与 HashMap 一致）
// 2. CAS + synchronized 保证线程安全
// 3. 锁的粒度从 Segment 级降为 桶（bucket）级

// 核心成员变量：
// transient volatile Node<K,V>[] table;  // 哈希数组（volatile）
// private transient volatile int sizeCtl;
//   sizeCtl < 0  ：-1 表示正在初始化，-(1+n) 表示正在扩容
//   sizeCtl = 0  ：默认值
//   sizeCtl > 0  ：阈值（capacity * loadFactor）

// put 流程：
final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null) throw new NullPointerException();
    int hash = spread(key.hashCode());   // 计算 hash（扰动）
    int binCount = 0;

    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        if (tab == null || (n = tab.length) == 0)
            tab = initTable();           // 延迟初始化（CAS 保证线程安全）

        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            // 该桶为空 → CAS 直接插入（无锁）
            if (casTabAt(tab, i, null, new Node<K,V>(hash, key, value, null)))
                break;
        }

        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);  // 正在扩容，协助迁移

        else {
            V oldVal = null;
            synchronized (f) {           // 仅锁当前桶的头节点
                if (tabAt(tab, i) == f) {
                    if (fh >= 0) {       // 链表
                        for (Node<K,V> e = f; e != null; e = e.next) {
                            // 遍历链表插入或覆盖...
                        }
                    } else if (f instanceof TreeBin) {
                        // 红黑树插入
                    }
                }
            }
            if (binCount >= TREEIFY_THRESHOLD)
                treeifyBin(tab, i);      // 链表转红黑树（≥8）
        }
    }
    addCount(1L, binCount);
    return null;
}

// get 流程（完全无锁）：
public V get(Object key) {
    Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
    int h = spread(key.hashCode());
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (e = tabAt(tab, (n - 1) & h)) != null) {
        if ((eh = e.hash) == h) {
            // 检查头节点
            if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                return e.val;
        } else if (eh < 0)
            // 特殊节点（TreeBin 或 ForwardingNode）
            return (p = e.find(h, key)) != null ? p.val : null;
        while ((e = e.next) != null) {
            // 遍历链表
            if (e.hash == h && ((ek = e.key) == key || (ek != null && key.equals(ek))))
                return e.val;
        }
    }
    return null;
}
// get 无锁是因为 Node.val 和 Node.next 都是 volatile

// size() 实现：
// JDK 8 使用 CounterCell 数组来维护计数，避免单一计数器的竞争
// sumCount() 累加 baseCount + 所有 CounterCell 的值
// 不再需要 JDK 7 那种无锁尝试 → 加锁重试的策略
```

### 3.3 扩容机制

```java
// JDK 8 ConcurrentHashMap 支持多线程并发扩容（transfer）

// 触发条件：
// 1. put 后总元素数超过 sizeCtl（阈值）
// 2. 链表转树时，如果数组长度 < 64，优先扩容

// 扩容过程：
// 1. 生成新的哈希数组（nextTable），容量 2 倍
// 2. 将旧数组的节点迁移到新数组
// 3. 多线程协作迁移，每个线程负责一个 stride 区间

// 扩容时的 ForwardingNode：
// 迁移完一个桶后，将旧桶头节点替换为 ForwardingNode
// hash = MOVED（-1）
// 其他线程 put/get 遇到 ForwardingNode：
//   - put：先协助扩容（helpTransfer），再在新数组中插入
//   - get：通过 ForwardingNode.find() 到新数组中查找

// addCount 中的扩容触发逻辑（简化）：
private final void addCount(long x, int check) {
    CounterCell[] as; long b, s;
    // 计数...
    if (check >= 0) {
        for (;;) {
            if (s > (long)sizeCtl && table.length < MAXIMUM_CAPACITY) {
                int rs = resizeStamp(n);
                if (sc < 0) {
                    // 正在扩容，协助
                    if (...)
                        transfer(tab, nt);
                } else {
                    // 触发扩容
                    U.compareAndSwapInt(this, SIZECTL, sc, (rs << RESIZE_STAMP_SHIFT) + 2);
                    transfer(tab, null);
                }
            }
        }
    }
}
```

### 3.4 并发集合对比

```java
// ┌──────────────────────┬──────────────────┬──────────────────┬──────────────────────┐
// │ 集合                  │ 线程安全          │ 锁粒度            │ 适用场景              │
// ├──────────────────────┼──────────────────┼──────────────────┼──────────────────────┤
// │ HashMap              │ ❌               │ 无               │ 单线程                │
// │ Hashtable            │ ✅               │ 整表锁           │ ❌ 不推荐             │
// │ Collections.syncMap()│ ✅               │ 整表锁           │ 简单包装              │
// │ ConcurrentHashMap    │ ✅               │ 桶级锁+CAS      │ 高并发读写 ✅首选     │
// ├──────────────────────┼──────────────────┼──────────────────┼──────────────────────┤
// │ ArrayList            │ ❌               │ 无               │ 单线程                │
// │ Vector               │ ✅               │ 整表锁           │ ❌ 不推荐             │
// │ CopyOnWriteArrayList │ ✅               │ 写时复制         │ 读多写极少 ✅          │
// ├──────────────────────┼──────────────────┼──────────────────┼──────────────────────┤
// │ HashSet              │ ❌               │ 无               │ 单线程                │
// │ ConcurrentSkipListSet│ ✅               │ 无锁(CAS)       │ 有序并发集合          │
// ├──────────────────────┼──────────────────┼──────────────────┼──────────────────────┤
// │ TreeMap              │ ❌               │ 无               │ 单线程排序            │
// │ ConcurrentSkipListMap│ ✅               │ 无锁(CAS)       │ 高并发排序            │
// │                      │                  │                  │ ✅ O(log n) 无锁     │
// └──────────────────────┴──────────────────┴──────────────────┴──────────────────────┘
```

---

## 4. ThreadPoolExecutor

### 4.1 核心参数与状态管理

```java
public class ThreadPoolExecutor extends AbstractExecutorService {

    // 核心参数
    private final int corePoolSize;       // 核心线程数（一直存活，除非 allowCoreThreadTimeOut）
    private final int maximumPoolSize;    // 最大线程数（核心 + 临时）
    private final long keepAliveTime;     // 临时线程空闲存活时间
    private final BlockingQueue<Runnable> workQueue;  // 任务队列
    private final ThreadFactory threadFactory;        // 线程工厂
    private final RejectedExecutionHandler handler;   // 拒绝策略

    // 运行状态（高 3 位存储状态，低 29 位存储线程数）
    // ctl = 线程数 + 运行状态（AtomicInteger）
    private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));

    // 状态编码：
    // RUNNING    : 111 → 接受新任务并处理队列任务
    // SHUTDOWN   : 000 → 不接受新任务，但处理队列任务
    // STOP       : 001 → 不接受新任务，不处理队列任务，中断进行中的任务
    // TIDYING    : 010 → 所有任务终止，即将执行 terminated()
    // TERMINATED : 011 → terminated() 执行完毕

    private static final int RUNNING    = -1 << COUNT_BITS;
    private static final int SHUTDOWN   =  0 << COUNT_BITS;
    private static final int STOP       =  1 << COUNT_BITS;
    private static final int TIDYING    =  2 << COUNT_BITS;
    private static final int TERMINATED =  3 << COUNT_BITS;

    // 状态流：
    // RUNNING → SHUTDOWN（shutdown()）
    // RUNNING → STOP（shutdownNow()）
    // SHUTDOWN → TIDYING（队列和线程池都为空）
    // STOP → TIDYING（线程池为空）
    // TIDYING → TERMINATED（terminated() 执行完毕）
}
```

### 4.2 任务调度流程

```java
// execute 流程（核心逻辑）：
public void execute(Runnable command) {
    int c = ctl.get();
    // 1. workerCount < corePoolSize
    //    → 创建核心线程执行任务
    if (workerCountOf(c) < corePoolSize) {
        if (addWorker(command, true))
            return;
        c = ctl.get();  // 重新获取（addWorker 可能失败）
    }

    // 2. 线程数 ≥ corePoolSize
    //    → 尝试加入阻塞队列
    if (isRunning(c) && workQueue.offer(command)) {
        int recheck = ctl.get();
        // 双重检查：线程池是否已关闭
        if (!isRunning(recheck) && remove(command))
            reject(command);               // 关闭了 → 拒绝
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);        // 没有线程了 → 添加空任务 Worker
    }

    // 3. 队列满了
    //    → 尝试创建临时线程（maximumPoolSize）
    else if (!addWorker(command, false))
        reject(command);                   // 线程数已达上限 → 拒绝

    // 总结：
    // corePoolSize → workQueue → maximumPoolSize → 拒绝策略
}
```

**线程池参数调优经验公式：**

```
// CPU 密集型：corePoolSize = CPU 核数 + 1
//   理由：CPU 密集型线程几乎不等待，线程数超过 CPU 核数只会增加切换开销
//   +1 是为了补偿页缺失等偶尔的等待

// IO 密集型：corePoolSize = CPU 核数 * 2（或更多）
//   理由：IO 密集型线程大部分时间在等待，更多线程可以提高 CPU 利用率
//   经验值：CPU 核数 * (1 + 平均等待时间 / 平均计算时间)

// 混合型：拆分 CPU 密集和 IO 密集的任务到不同的线程池
//   避免 IO 等待占用了 CPU 任务的线程
```

### 4.3 Worker 机制

```java
// Worker 继承了 AQS，实现了 Runnable
// 每个 Worker 对应一个工作线程

private final class Worker extends AbstractQueuedSynchronizer
    implements Runnable {

    final Thread thread;       // 工作线程（由 ThreadFactory 创建）
    Runnable firstTask;        // 第一个任务（可为 null，从队列取）
    volatile long completedTasks;  // 完成任务数

    Worker(Runnable firstTask) {
        setState(-1);          // 防止在 runWorker 之前被中断
        this.firstTask = firstTask;
        this.thread = getThreadFactory().newThread(this);
    }

    public void run() {
        runWorker(this);       // 核心的 Worker 运行循环
    }
}

// runWorker 核心循环：
final void runWorker(Worker w) {
    Thread wt = Thread.currentThread();
    Runnable task = w.firstTask;
    w.firstTask = null;
    w.unlock();                 // 允许中断（state = 0）
    boolean completedAbruptly = true;
    try {
        // 循环取任务执行（getTask 从 workQueue 取）
        while (task != null || (task = getTask()) != null) {
            w.lock();           // 加锁 = 标记 Worker 为"繁忙"

            // 检查线程池是否 STOP，如果是则中断
            if ((runStateAtLeast(ctl.get(), STOP) ||
                 (Thread.interrupted() && runStateAtLeast(ctl.get(), STOP))) &&
                !wt.isInterrupted())
                wt.interrupt();

            try {
                beforeExecute(wt, task);     // 前置钩子
                try {
                    task.run();              // 真正执行任务
                    afterExecute(task, null); // 后置钩子
                } catch (Throwable ex) {
                    afterExecute(task, ex);
                    throw ex;
                }
            } finally {
                task = null;
                w.completedTasks++;
                w.unlock();       // 解锁
            }
        }
        completedAbruptly = false;
    } finally {
        // 线程退出处理
        processWorkerExit(w, completedAbruptly);
    }
}

// getTask —— 从队列取任务（含超时逻辑）
private Runnable getTask() {
    boolean timedOut = false;
    for (;;) {
        int c = ctl.get();
        if (runStateAtLeast(c, SHUTDOWN) &&       // 关闭了
            (runStateAtLeast(c, STOP) || workQueue.isEmpty()))
            return null;  // 减少 Worker 数

        int wc = workerCountOf(c);

        // allowCoreThreadTimeOut || wc > corePoolSize
        boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;

        if ((wc > maximumPoolSize || (timed && timedOut)) &&
            (wc > 1 || workQueue.isEmpty())) {
            if (compareAndDecrementWorkerCount(c))
                return null;  // 减少 Worker 数
            continue;
        }

        try {
            Runnable r = timed ?
                workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :  // 超时等待
                workQueue.take();                                      // 阻塞等待
            if (r != null) return r;
            timedOut = true;    // poll 超时，下次循环减少 Worker
        } catch (InterruptedException retry) {
            timedOut = false;
        }
    }
}
```

### 4.4 拒绝策略

```java
// JDK 内置 4 种拒绝策略：

// 1. AbortPolicy（默认）：直接抛 RejectedExecutionException
public static class AbortPolicy implements RejectedExecutionHandler {
    public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
        throw new RejectedExecutionException("Task " + r.toString() +
                                             " rejected from " + e.toString());
    }
}

// 2. CallerRunsPolicy：由调用者线程执行任务（反压）
//    提交任务的线程自己执行，放慢提交速度
public static class CallerRunsPolicy implements RejectedExecutionHandler {
    public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
        if (!e.isShutdown()) {
            r.run();  // 调用者线程直接运行
        }
    }
}

// 3. DiscardPolicy：静默丢弃
//    直接扔掉，不抛异常（容易导致任务丢失，不推荐）
public static class DiscardPolicy implements RejectedExecutionHandler {
    public void rejectedExecution(Runnable r, ThreadPoolExecutor e) { }
}

// 4. DiscardOldestPolicy：丢弃队列中最老的任务，重试提交
//    适用于可以丢弃旧任务的场景
public static class DiscardOldestPolicy implements RejectedExecutionHandler {
    public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
        if (!e.isShutdown()) {
            e.getQueue().poll();   // 丢一个最老的
            e.execute(r);          // 重试
        }
    }
}

// 自定义策略（推荐，如记录日志或写入消息队列）：
public class LoggingRejectedHandler implements RejectedExecutionHandler {
    public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
        log.warn("Task rejected: {}", r.toString());
        // 可写入 MQ 或数据库，稍后重试
    }
}
```

### 4.5 Executors 工厂方法（不推荐）

```java
// 虽然 Executors 提供了便捷方法，但《Java 开发手册》强制不允许使用：

// ❌ 问题：newFixedThreadPool 使用无界 LinkedBlockingQueue
//    任务堆积会导致 OOM
ExecutorService fixed = Executors.newFixedThreadPool(10);
//    workQueue = new LinkedBlockingQueue<Runnable>()  // 容量 Integer.MAX_VALUE

// ❌ 问题：newSingleThreadExecutor 同样使用无界队列
ExecutorService single = Executors.newSingleThreadExecutor();

// ❌ 问题：newCachedThreadPool 最大线程数为 Integer.MAX_VALUE
//    大量任务会导致创建过多线程，耗尽系统资源
ExecutorService cached = Executors.newCachedThreadPool();
//    maximumPoolSize = Integer.MAX_VALUE
//    SynchronousQueue（直接传递，不存储）

// ❌ 问题：newScheduledThreadPool 无界 DelayedWorkQueue
ScheduledExecutorService scheduled = Executors.newScheduledThreadPool(10);

// ✅ 正确方式：通过 ThreadPoolExecutor 构造函数手动指定参数
ThreadPoolExecutor executor = new ThreadPoolExecutor(
    10,                      // corePoolSize
    20,                      // maximumPoolSize
    60L, TimeUnit.SECONDS,   // keepAliveTime
    new ArrayBlockingQueue<>(1000),  // 有界队列
    new ThreadFactoryBuilder().setNameFormat("biz-pool-%d").build(),
    new ThreadPoolExecutor.CallerRunsPolicy()
);
```

### 4.6 ScheduledThreadPoolExecutor

```java
// ScheduledThreadPoolExecutor 继承 ThreadPoolExecutor
// 核心是 schedule 方法的延迟执行机制

public class ScheduledThreadPoolExecutor extends ThreadPoolExecutor {
    // schedule：延迟执行一次
    public ScheduledFuture<?> schedule(Runnable command,
                                       long delay, TimeUnit unit);

    // scheduleAtFixedRate：固定频率（任务开始时间间隔）
    // 如果任务执行时间超过 period，下一次不会并行，而是等当前任务结束立即开始
    public ScheduledFuture<?> scheduleAtFixedRate(Runnable command,
                                                  long initialDelay,
                                                  long period,
                                                  TimeUnit unit);

    // scheduleWithFixedDelay：固定延迟（任务结束后开始计时）
    // 无论任务执行多久，间隔始终从执行完后开始计算
    public ScheduledFuture<?> scheduleWithFixedDelay(Runnable command,
                                                     long initialDelay,
                                                     long delay,
                                                     TimeUnit unit);
}

// 内部实现：
// 1. 使用 DelayedWorkQueue（优先级队列）存储任务
// 2. 每次从队列取出最近到期的任务执行
// 3. 循环任务执行完后重新计算下一次执行时间，重新入队

// scheduleAtFixedRate vs scheduleWithFixedDelay：
// scheduleAtFixedRate:   ████░░████░░████     执行固定间隔（包括执行时间）
// scheduleWithFixedDelay: ████░░░░████░░░░     执行间隔从结束开始算
```

---

## 5. ForkJoinPool

### 5.1 分治模型与工作窃取

```java
// ForkJoinPool 是 JDK 7 引入的线程池，专为分治任务设计
// 核心：将大任务拆分为小任务，直到足够小后直接执行

// 工作窃取（Work-Stealing）：
// 每个工作线程有自己的双端队列（deque）
// 自己从队尾取任务执行（LIFO）→ 减少竞争
// 空闲线程从其他线程的队首偷取任务（FIFO）→ 窃取的是最老的任务
// 窃取时才有竞争（CAS），自己取任务无竞争

// ┌─────Worker-1─────┐    ┌─────Worker-2─────┐
// │  ┌───────────┐   │    │  ┌───────────┐   │
// │  │ Task A    │ ←─┼────┼──│ Task D    │   │
// │  │ Task B    │   │    │  │ Task E    │   │
// │  │ Task C ◄──┼───┼────┼──│           │   │ ← Worker-2 空闲
// │  └───────────┘   │    │  └───────────┘   │   从队首偷 Task C
// │  取任务从队尾(C)  │    │  偷任务从队首(C)  │
// └──────────────────┘    └──────────────────┘
// 队尾无竞争（自己取）   队首有竞争（CAS）

// 工作窃取的优点：
// 1. 充分利用 CPU 资源
// 2. 减少线程调度开销
// 3. 天然负载均衡（忙的线程做自己的任务，闲的线程帮别人）
```

### 5.2 RecursiveTask 和 RecursiveAction

```java
// ForkJoinTask 的两种子类：
// RecursiveTask<V>：有返回值
// RecursiveAction：无返回值

// 示例：并行计算数组和
public class SumTask extends RecursiveTask<Long> {
    private static final int THRESHOLD = 1000;
    private final long[] array;
    private final int start;
    private final int end;

    public SumTask(long[] array, int start, int end) {
        this.array = array;
        this.start = start;
        this.end = end;
    }

    @Override
    protected Long compute() {
        int length = end - start;
        if (length <= THRESHOLD) {
            // 足够小，直接计算
            long sum = 0;
            for (int i = start; i < end; i++) {
                sum += array[i];
            }
            return sum;
        }

        // 拆分子任务
        int mid = start + length / 2;
        SumTask leftTask = new SumTask(array, start, mid);
        SumTask rightTask = new SumTask(array, mid, end);

        // fork 子任务（异步）
        leftTask.fork();
        rightTask.fork();

        // join 获取结果（阻塞等待）
        long leftResult = leftTask.join();
        long rightResult = rightTask.join();

        return leftResult + rightResult;

        // 或者使用 forkJoin 更高效：
        // invokeAll(leftTask, rightTask);  // fork 两个子任务
        // return leftTask.join() + rightTask.join();
    }

    public static void main(String[] args) {
        long[] array = new long[100_000];
        // 填充数据...

        ForkJoinPool pool = new ForkJoinPool(); // 默认并行度 = CPU 核数
        long result = pool.invoke(new SumTask(array, 0, array.length));
        System.out.println("Sum: " + result);
    }
}

// 优化提示：
// 1. 不要在 compute() 中执行阻塞操作（ForkJoinPool 默认没有 Compensation）
// 2. 使用 invokeAll(forkJoinTask...) 比显式 fork/join 更高效（内部优化了调度）
// 3. 阈值选择：经验值为 100-10000，取决于任务复杂度
```

### 5.3 ForkJoinPool 公共池

```java
// ForkJoinPool 提供全局公共池 commonPool
// 注意：commonPool 是全局共享的，JDK 8 的许多地方默认使用它

// 获取公共池
ForkJoinPool commonPool = ForkJoinPool.commonPool();

// 公共池参数：
// 并行度 = Runtime.availableProcessors() - 1
//   （如果机器是 4 核，并行度为 3）

// 扩展机制：
// ForkJoinPool 实现 ManagedBlocker 接口来处理阻塞任务
// 如果任务阻塞（如等待 IO），可以通过 ForkJoinPool.managedBlock() 补偿
// 但 commonPool 的补偿能力有限

// 问题场景：
// parallelStream 默认使用 commonPool
// 如果 parallelStream 中有阻塞操作（如远程调用），会消耗所有公共线程
// 导致其他使用 commonPool 的地方也卡住

// ✅ 阻塞任务应使用自定义 ForkJoinPool
ForkJoinPool myPool = new ForkJoinPool(parallelism);
try {
    myPool.submit(() -> {
        // 可以包含阻塞操作的自定义 ForkJoinTask
    }).get();
} finally {
    myPool.shutdown();
}

// 或者将 parallelStream 提交到自定义池：
myPool.submit(() ->
    list.parallelStream().forEach(item -> {
        // 阻塞操作
    })
).get();
```

---

## 6. CompletableFuture

### 6.1 核心概念

```java
// CompletableFuture 是 JDK 8 引入的异步编程工具
// 实现了 Future 和 CompletionStage 接口
// 支持：回调、编排、异常处理、组合

// 与 Future 的对比：
// Future：
//   - 只能通过 get() 阻塞获取结果
//   - 无法主动设置结果
//   - 无法编排多个异步任务
//   - 没有异常处理机制（get 抛 ExecutionException）
// CompletableFuture：
//   - 回调驱动（thenApply/thenAccept）
//   - 可以手动完成（complete/completeExceptionally）
//   - 支持编排（thenCompose/thenCombine）
//   - 丰富的异常处理（exceptionally/handle）
```

### 6.2 创建 CompletableFuture

```java
// 1. 直接创建已完成的结果
CompletableFuture<String> completed = CompletableFuture.completedFuture("done");

// 2. 异步执行任务（默认 ForkJoinPool.commonPool()）
CompletableFuture<Void> future = CompletableFuture.runAsync(() -> {
    System.out.println("Running in " + Thread.currentThread().getName());
});
// runAsync：无返回值

CompletableFuture<String> future2 = CompletableFuture.supplyAsync(() -> {
    return "Hello";
});
// supplyAsync：有返回值

// 3. 指定线程池（推荐）
ExecutorService executor = Executors.newFixedThreadPool(10);
CompletableFuture<String> future3 = CompletableFuture.supplyAsync(() -> {
    return "Hello";
}, executor);
```

### 6.3 回调与转换

```java
// 串行转换（类似 Stream 的 map）
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> "Hello")
    .thenApply(s -> s + " World")       // 同步转换（Function）
    .thenApply(String::toUpperCase);    // "HELLO WORLD"

// 消费结果（无返回值）
future.thenAccept(System.out::println);  // Consumer

// 只执行（不关心结果）
future.thenRun(() -> System.out.println("Done"));  // Runnable

// 异步回调（指定线程执行回调）
future.thenApplyAsync(s -> s + " World", executor);
```

### 6.4 异常处理

```java
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
    if (Math.random() > 0.5) throw new RuntimeException("Error");
    return "Success";
});

// 1. exceptionally：异常时返回默认值
String result1 = future.exceptionally(ex -> {
    System.err.println("Caught: " + ex);
    return "Default";
}).join();

// 2. handle：无论正常还是异常都处理，可根据 Throwable 是否为 null 判断
String result2 = future.handle((res, ex) -> {
    if (ex != null) {
        System.err.println("Error: " + ex);
        return "Fallback";
    }
    return res;
}).join();

// 3. whenComplete：消费结果或异常，不改变结果
future.whenComplete((res, ex) -> {
    if (ex != null) System.err.println("Error: " + ex);
    else System.out.println("Result: " + res);
    // 不返回值，不影响链式调用
}).thenApply(s -> s + " processed");
```

### 6.5 组合与编排

```java
// 1. thenCompose：任务依赖（flatMap）
//    第一个任务的结果作为第二个任务的输入
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> "Hello")
    .thenCompose(s -> CompletableFuture.supplyAsync(() -> s + " World"));
// "Hello World"

// 2. thenCombine：合并两个独立任务的结果
CompletableFuture<String> f1 = CompletableFuture.supplyAsync(() -> "Hello");
CompletableFuture<String> f2 = CompletableFuture.supplyAsync(() -> "World");
CompletableFuture<String> combined = f1.thenCombine(f2, (a, b) -> a + " " + b);
// "Hello World"

// 3. allOf：等待所有任务完成（无返回值）
CompletableFuture<String> f1 = CompletableFuture.supplyAsync(() -> "A");
CompletableFuture<String> f2 = CompletableFuture.supplyAsync(() -> "B");
CompletableFuture<String> f3 = CompletableFuture.supplyAsync(() -> "C");
CompletableFuture<Void> all = CompletableFuture.allOf(f1, f2, f3);
all.join();  // 阻塞直到所有完成

// allOf 的痛点：无返回值，需要手动收集
// 解决方法：
List<String> results = Stream.of(f1, f2, f3)
    .map(CompletableFuture::join)
    .collect(Collectors.toList());

// 或者自定义工具方法：
@SuppressWarnings("unchecked")
public static <T> CompletableFuture<List<T>> sequence(
        List<CompletableFuture<T>> futures) {
    return CompletableFuture.allOf(futures.toArray(new CompletableFuture[0]))
        .thenApply(v -> futures.stream()
            .map(CompletableFuture::join)
            .collect(Collectors.toList()));
}

// 4. anyOf：任意一个完成就返回
CompletableFuture<Object> any = CompletableFuture.anyOf(f1, f2, f3);
Object firstResult = any.join();
```

### 6.6 实际应用示例

```java
// 模拟微服务编排
public class OrderService {
    private final ExecutorService executor = Executors.newFixedThreadPool(20);

    public CompletableFuture<OrderResult> getOrderDetail(String orderId) {
        // 并行查询：订单信息、用户信息、商品信息、物流信息
        CompletableFuture<OrderInfo> orderInfo =
            CompletableFuture.supplyAsync(() -> queryOrder(orderId), executor);
        CompletableFuture<UserInfo> userInfo =
            CompletableFuture.supplyAsync(() -> queryUser(orderId), executor);
        CompletableFuture<ProductInfo> productInfo =
            CompletableFuture.supplyAsync(() -> queryProduct(orderId), executor);
        CompletableFuture<LogisticsInfo> logisticsInfo =
            CompletableFuture.supplyAsync(() -> queryLogistics(orderId), executor);

        // 合并所有结果
        return orderInfo.thenCombine(userInfo, (order, user) ->
                new OrderResult(order, user, null, null))
            .thenCombine(productInfo, (result, product) -> {
                result.setProduct(product);
                return result;
            })
            .thenCombine(logisticsInfo, (result, logistics) -> {
                result.setLogistics(logistics);
                return result;
            })
            .exceptionally(ex -> {
                log.error("Failed to get order detail", ex);
                return new OrderResult(null, null, null, null);
            });
    }
}

// 超时处理（JDK 9+）：
// future.orTimeout(1, TimeUnit.SECONDS);
// future.completeOnTimeout("default", 1, TimeUnit.SECONDS);

// JDK 8 中模拟超时：
CompletableFuture<String> futureWithTimeout = CompletableFuture
    .supplyAsync(() -> {
        try { Thread.sleep(5000); } catch (InterruptedException e) {}
        return "result";
    }, executor);

// 通过 completeOnTimeout 模拟
futureWithTimeout.completeOnTimeout("timeout", 1, TimeUnit.SECONDS);
```

---

## 7. 同步工具类

### 7.1 CountDownLatch（一次性栅栏）

```java
// 场景：一个线程等待多个线程完成
// 类似于"发令枪"的反向——等待所有选手就位

// 核心方法：
// CountDownLatch(int count)：指定计数
// countDown()：计数减 1
// await()：阻塞直到计数为 0
// await(long timeout, TimeUnit unit)：超时等待

public class CountDownLatchDemo {
    public static void main(String[] args) throws InterruptedException {
        int threadCount = 5;
        CountDownLatch latch = new CountDownLatch(threadCount);

        for (int i = 0; i < threadCount; i++) {
            new Thread(() -> {
                try {
                    System.out.println(Thread.currentThread().getName() + " working");
                    Thread.sleep((long) (Math.random() * 1000));
                    System.out.println(Thread.currentThread().getName() + " done");
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                } finally {
                    latch.countDown();  // 必须在 finally 中执行
                }
            }).start();
        }

        System.out.println("Main waiting for all threads...");
        latch.await();  // 阻塞，直到 count 为 0
        System.out.println("All threads completed. Main continues.");
    }
}

// 特点：
// 1. 一次性：计数归零后不可重用
// 2. 计数在构造时设定，不可更改
// 3. await 的线程可以多个（所有等待线程同时被唤醒）

// 原理：基于 AQS 共享模式
// count = AQS.state
// countDown() → releaseShared(1) → tryReleaseShared 减少 state
// await() → acquireShared(1) → tryAcquireShared 检查 state == 0
```

### 7.2 CyclicBarrier（可循环栅栏）

```java
// 场景：多个线程互相等待，都到达某个屏障点后继续执行
// 类似于"凑齐一桌开饭"

// 核心方法：
// CyclicBarrier(int parties)：需要等待的线程数
// CyclicBarrier(int parties, Runnable barrierAction)：到达屏障后优先执行 barrierAction
// await()：到达屏障点等待
// reset()：重置屏障（可重用）

public class CyclicBarrierDemo {
    public static void main(String[] args) {
        int workerCount = 4;
        // 所有 worker 都到达屏障后，优先执行合并操作
        CyclicBarrier barrier = new CyclicBarrier(workerCount, () -> {
            System.out.println("=== All workers arrived, merging results ===");
        });

        for (int i = 0; i < workerCount; i++) {
            new Thread(() -> {
                try {
                    System.out.println(Thread.currentThread().getName() + " stage 1 done");
                    barrier.await();  // 等待其他线程

                    System.out.println(Thread.currentThread().getName() + " stage 2 done");
                    barrier.await();  // 等待其他线程（可重用）

                    System.out.println(Thread.currentThread().getName() + " stage 3 done");
                    barrier.await();
                } catch (InterruptedException | BrokenBarrierException e) {
                    Thread.currentThread().interrupt();
                }
            }).start();
        }
    }
}

// CountDownLatch vs CyclicBarrier：
// ┌──────────────────┬────────────────────────┬────────────────────────────┐
// │                  │ CountDownLatch         │ CyclicBarrier              │
// ├──────────────────┼────────────────────────┼────────────────────────────┤
// │ 可重用           │ ❌ 一次性               │ ✅ reset() 后可重用        │
// │ 角色             │ 一个等多个               │ 多个互等                   │
// │ 计数操作         │ 调用 countDown() 减      │ await() 隐式减             │
// │ 核心区别         │ 是"事件"                │ 是"屏障点"                 │
// │ 线程关系         │ 等待线程 ≠ 工作线程      │ 等待线程 = 工作线程         │
// └──────────────────┴────────────────────────┴────────────────────────────┘

// CyclicBarrier 原理：
// 内部使用 ReentrantLock + Condition（Generation 概念）
// 每次 barrier 被冲破（最后一个线程到达）后进入下一 Generation
// 如果某个线程等待超时或被中断，BrokenBarrierException 通知所有其他线程
```

### 7.3 Semaphore（信号量）

```java
// 场景：限流（控制同时访问某个资源的线程数）
// 类似于"停车场显示剩余车位"

// 核心方法：
// Semaphore(int permits)：许可数
// Semaphore(int permits, boolean fair)：是否公平
// acquire()：获取许可（可中断，可多个）
// acquireUninterruptibly()：获取许可（不可中断）
// tryAcquire()：尝试获取，不阻塞
// release()：释放许可

public class SemaphoreDemo {
    // 连接池限流
    private static final int MAX_CONNECTIONS = 3;
    private final Semaphore semaphore = new Semaphore(MAX_CONNECTIONS, true);

    public void connect() {
        try {
            semaphore.acquire();  // 获取连接许可
            // 使用数据库连接...
            System.out.println(Thread.currentThread().getName() + " connected");
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        } finally {
            semaphore.release();  // 释放许可（必须在 finally 中）
        }
    }
}

// 注意：
// 1. acquire 和 release 没有强制配对要求（可以 acquire(2) 然后 release(2)）
// 2. release 可以在没有 acquire 的情况下调用，会增加许可数（不当使用会导致问题）
// 3. 永远不要在 try 中 release，必须在 finally 中

// 原理：基于 AQS 共享模式
// permits = AQS.state
// acquire() → acquireSharedInterruptibly(1) → tryAcquireShared (state - 1)
// release() → releaseShared(1) → tryReleaseShared (state + 1)
```

### 7.4 Exchanger（交换器）

```java
// 场景：两个线程交换数据
// 类似于"接头暗号交换"

// 核心方法：
// exchange(V x)：等待另一个线程也到达交换点
// exchange(V x, long timeout, TimeUnit unit)：超时版本

public class ExchangerDemo {
    public static void main(String[] args) {
        Exchanger<String> exchanger = new Exchanger<>();

        new Thread(() -> {
            try {
                String data = "Data from Thread A";
                System.out.println("Thread A sending: " + data);
                String received = exchanger.exchange(data);
                System.out.println("Thread A received: " + received);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }).start();

        new Thread(() -> {
            try {
                String data = "Data from Thread B";
                Thread.sleep(100);  // 模拟 B 稍晚到达
                System.out.println("Thread B sending: " + data);
                String received = exchanger.exchange(data);
                System.out.println("Thread B received: " + received);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }).start();
    }
}

// 原理：内部使用 CAS + 自旋 + 阻塞（LockSupport）
// 第一个线程到达：创建 Node 并自旋等待
// 第二个线程到达：CAS 交换数据，unpark 第一个线程
// 支持多对多（内部使用 Slot 数组）
```

### 7.5 Phaser（分阶段同步器）

```java
// Phaser（JDK 7）是 CountDownLatch + CyclicBarrier 的增强版
// 特点：支持分阶段同步、动态增减参与者

public class PhaserDemo {
    public static void main(String[] args) {
        Phaser phaser = new Phaser(3) {  // 3 个参与者
            // 每次阶段推进时自动执行
            @Override
            protected boolean onAdvance(int phase, int registeredParties) {
                System.out.printf("Phase %d completed (%d parties)%n",
                    phase, registeredParties);
                return registeredParties == 0;  // true 结束 phaser
            }
        };

        for (int i = 0; i < 3; i++) {
            new Thread(() -> {
                for (int phase = 0; phase < 3; phase++) {
                    System.out.println(Thread.currentThread().getName()
                        + " executing phase " + phase);
                    phaser.arriveAndAwaitAdvance();  // 到达并等待
                }
            }).start();
        }
    }
}

// Phaser 核心方法：
// register() / bulkRegister(int)：注册参与者
// arrive()：到达，不等待
// arriveAndAwaitAdvance()：到达并等待其他人
// arriveAndDeregister()：到达并注销（不再参与后续阶段）
// getPhase()：当前阶段编号
// onAdvance()：阶段推进时的回调（类似 CyclicBarrier 的 barrierAction）

// 适用场景：多阶段任务
// 如：文件批处理 → Phase 1: 读取 → Phase 2: 处理 → Phase 3: 写入
```

---

## 8. 原子类（Atomic）

### 8.1 原子类体系

```java
// JUC 提供了一组原子操作类，基于 CAS（Compare-And-Swap）实现

// 基本类型：
//   AtomicBoolean、AtomicInteger、AtomicLong

// 引用类型：
//   AtomicReference<V>、AtomicStampedReference<V>、AtomicMarkableReference<V>
//   AtomicStampedReference：带版本号（解决 ABA 问题）
//   AtomicMarkableReference：带布尔标记

// 数组：
//   AtomicIntegerArray、AtomicLongArray、AtomicReferenceArray<V>

// 字段更新器（通过反射更新对象的 volatile 字段）：
//   AtomicIntegerFieldUpdater<T>、AtomicLongFieldUpdater<T>
//   AtomicReferenceFieldUpdater<T,V>

// JDK 8 新增：
//   DoubleAccumulator、DoubleAdder、LongAccumulator、LongAdder
```

### 8.2 CAS 原理

```java
// CAS（Compare-And-Swap）是乐观锁的核心实现

// CAS 操作包含三个操作数：
// 内存位置 V、预期值 A、新值 B
// 当且仅当 V 的值等于 A 时，将 V 更新为 B，否则不更新
// 无论是否更新，返回 V 的旧值

// Unsafe 类提供 CAS 操作：
public final class Unsafe {
    // 对象字段的 CAS
    public final native boolean compareAndSwapObject(
        Object o, long offset, Object expected, Object x);

    public final native boolean compareAndSwapInt(
        Object o, long offset, int expected, int x);

    public final native boolean compareAndSwapLong(
        Object o, long offset, long expected, long x);
}

// AtomicInteger 的 incrementAndGet 实现：
public class AtomicInteger {
    private volatile int value;
    private static final long valueOffset;

    static {
        try {
            // 获取 value 字段的内存偏移量
            valueOffset = unsafe.objectFieldOffset(
                AtomicInteger.class.getDeclaredField("value"));
        } catch (Exception ex) { throw new Error(ex); }
    }

    public final int incrementAndGet() {
        for (;;) {
            int current = get();
            int next = current + 1;
            if (compareAndSet(current, next))  // CAS 尝试
                return next;
            // 失败则重试（自旋）
        }
    }
}
```

### 8.3 ABA 问题

```java
// ABA 问题描述：
// 线程 1 读取 A → 线程 2 将 A 改为 B 又改回 A → 线程 1 CAS 成功
// 线程 1 认为值没有变化，但实际上已经变了两次

// 解决方法：版本号

// AtomicStampedReference 内部维护 [reference, stamp] 对
// 每次修改 stamp + 1，即使 reference 回到旧值，stamp 也不同

public class ABADemo {
    // 带版本号的原子引用
    private static final AtomicStampedReference<String> ref =
        new AtomicStampedReference<>("A", 0);

    public static void main(String[] args) {
        int[] stampHolder = new int[1];

        // 获取当前值和版本号
        String value = ref.get(stampHolder);
        int stamp = stampHolder[0];

        // 其他线程修改了 value：A → B → A

        // CAS 检查：值匹配但版本号不匹配 → CAS 失败
        boolean success = ref.compareAndSet("A", "C", stamp, stamp + 1);
        System.out.println("CAS result: " + success);  // false
    }
}
```

### 8.4 LongAdder（JDK 8）

```java
// 高竞争场景下 AtomicLong 的性能问题：
// 所有线程 CAS 同一个 value → 大量 CAS 失败 → 频繁重试

// LongAdder 的优化：
// 将单一 value 拆分为 base + Cell[] 数组
// 每个线程通过 hash 分配到不同的 Cell
// 最终 sum() = base + sum(Cell[])

// ┌────────────────────────────────────────────┐
// │  LongAdder                                  │
// │  ├── base: long          ← 低竞争时使用     │
// │  └── cells: Cell[]       ← 高竞争时扩容     │
// │      ├── Cell[0]: value                    │
// │      ├── Cell[1]: value                    │
// │      ├── Cell[2]: value                    │
// │      └── ...                               │
// └────────────────────────────────────────────┘

// 适用场景：
// AtomicLong：需要精确的数值（计数器、序列生成器）
// LongAdder：只关心最终结果，不关心中间值（统计、监控指标）

public class LongAdderDemo {
    private final LongAdder count = new LongAdder();

    public void increment() {
        count.increment();  // CAS 冲突比 AtomicLong 小得多
    }

    public long total() {
        return count.sum();  // 非原子快照
    }

    public void reset() {
        count.reset();  // 重置为零
    }

    public long sumThenReset() {
        return count.sumThenReset();  // 获取并重置（适合定时上报的场景）
    }
}

// 高竞争场景性能对比（粗略）：
// 1 线程：AtomicLong ≈ LongAdder
// 4 线程：LongAdder 约快 3-5 倍
// 16 线程：LongAdder 约快 10 倍
```

---

## 9. LockSupport

```java
// LockSupport 是线程阻塞/唤醒的基础工具
// 底层调用 Unsafe.park() / unpark()

// 核心方法：
// park()：阻塞当前线程（可设置 blocker 对象用于诊断）
// parkNanos(long nanos)：超时阻塞
// parkUntil(long deadline)：阻塞到绝对时间
// unpark(Thread thread)：唤醒指定线程

// 与 Object.wait/notify 的区别：
// 1. 不需要 synchronized 块
// 2. unpark 可以先于 park 调用（许可证机制）
// 3. park 不会释放锁
// 4. 响应中断但不抛异常（需手动检查 Thread.interrupted()）

public class LockSupportDemo {
    public static void main(String[] args) throws InterruptedException {
        Thread t = new Thread(() -> {
            System.out.println("Thread starting...");
            LockSupport.park();  // 阻塞等待许可
            System.out.println("Thread unparked!");
            // 检查是否被中断
            if (Thread.interrupted()) {
                System.out.println("Thread was interrupted");
            }
            // park 不响应中断（但标记中断状态）
            LockSupport.park();
            System.out.println("Thread unparked again!");
        });
        t.start();

        Thread.sleep(1000);
        LockSupport.unpark(t);  // 发放许可（先于 park 也可以）
        // 或者 t.interrupt();  // 中断也能使 park 返回
    }
}

// 许可证机制（类比为 1 位的信号量）：
// 每个线程关联一个许可（permit），默认为 0
// unpark：设置 permit = 1（如果已经是 1，则什么也不做）
// park：检查 permit，如果为 1 则消费并返回；如果为 0 则阻塞

// AQS 中使用 LockSupport.park() 实现线程阻塞
// Condition.await() 中同样使用 LockSupport.park()
```

---

## 10. 面试题

### Q1：AQS 的原理是什么？

```java
// AQS（AbstractQueuedSynchronizer）是一个基于 CLH 锁变体的同步器框架。
//
// 核心：state（volatile int）+ CLH 双向队列
//
// 工作流程：
// 1. 子类通过 tryAcquire/tryRelease 等模板方法定义同步语义
// 2. 获取失败时，AQS 将线程封装为 Node 加入等待队列
// 3. 队列中的线程通过自旋 + park() 等待前驱节点唤醒
// 4. 释放时 unpark 后继节点
//
// 两种模式：
// - 独占模式：ReentrantLock（一次只有一个线程能获取）
// - 共享模式：Semaphore/CountDownLatch（多个线程可以同时获取）
//
// Condition：每个 Condition 有自己的等待队列
// await = 释放锁 + 加入条件队列 + park
// signal = 转移条件队列节点到同步队列 + unpark
```

### Q2：synchronized 和 ReentrantLock 怎么选择？

```java
// 简单场景优先 synchronized（代码简洁，不易出错）
//
// 需要以下特性时使用 ReentrantLock：
// 1. 可中断等待：lockInterruptibly()
// 2. 超时：tryLock(timeout, unit)
// 3. 公平锁：new ReentrantLock(true)
// 4. 多个等待条件：lock.newCondition() 创建多个
//
// JDK 6+ synchronized 经过锁升级优化（偏向 → 轻量 → 重量）
// 低竞争时性能不输 ReentrantLock
// 高竞争时 ReentrantLock 通常更优（可配置 + CAS 非公平插队）
```

### Q3：ConcurrentHashMap JDK 7 和 JDK 8 的区别？

```java
// JDK 7（分段锁）：
// - 数据结构：Segment[] + HashEntry[]
// - 并发度：默认 16（固定的 segment 数量）
// - 锁粒度：Segment 级
// - 查找：两次 hash（先定位 Segment，再定位 HashEntry）
// - size()：无锁累加尝试 → 失败则加锁
//
// JDK 8（CAS + synchronized）：
// - 数据结构：Node[] + 链表/红黑树
// - 并发度：数组长度（桶数量，可扩容）
// - 锁粒度：桶（bucket）级，更细
// - 查找：一次 hash
// - size()：CounterCell 数组（分散计数，近似无锁）
// - 新增：扩容时多线程协助、链表树化、红黑树退化
```

### Q4：线程池的 execute 和 submit 的区别？

```java
// execute(Runnable)：
// - 无返回值
// - 异常直接抛到任务之外（如果没捕获，线程池捕获后重新创建线程）
// - 异常无法追踪
//
// submit(Callable/Runnable)：
// - 返回 Future
// - 异常被封装在 Future 中，get() 时抛 ExecutionException
// - 可以追踪异步结果
//
// 如果只关心执行不关心结果，用 execute
// 如果需要结果或捕获异常，用 submit + Future.get()
```

### Q5：ForkJoinPool 工作原理？

```java
// 1. 分治：大任务递归拆分为小任务，直到阈值（THRESHOLD）
// 2. 工作窃取：每个工作线程有双端队列，自己从队尾取，空闲线程从其他线程队首偷
// 3. 任务调度：fork() 将任务放入工作队列，join() 等待结果
// 4. commonPool 默认并行度 = CPU 核数 - 1
// 5. 不区分核心/临时线程，所有线程同等（目标并行度 parallelism）
//
// 自适应调整：
// 如果池中没有足够的任务队列，ForkJoinPool 会创建新的工作线程
// 如果任务长时间空闲，线程会逐步回收
//
// 使用注意：任务不应包含阻塞操作（IO、等待锁）
// 阻塞任务应使用 managedBlock() 或自定义池
```

### Q6：ThreadLocal 的原理和内存泄漏问题？

```java
// ThreadLocal 原理：
// 每个 Thread 维护一个 ThreadLocalMap
// map 的 key 是 ThreadLocal 对象（弱引用），value 是存储的值
//
// 内存泄漏原因：
// ThreadLocal 被 GC 回收后，key 变为 null，但 value 仍然存在
// 如果线程一直存活（线程池），value 永远不会回收
//
// ┌──────────────────────┐
// │  Thread              │
// │  ├── threadLocals ───┼──→ ThreadLocalMap
// │  └── ...             │     ├── Entry(key: WeakReference(threadLocal), value)
// │                      │     ├── Entry(key: null, value) ← 内存泄漏
// │                      │     └── ...
// └──────────────────────┘
//
// 解决方法：
// 1. 每次使用完调用 remove() —— 最佳实践
// 2. ThreadLocal 声明为 private static（强引用防止 GC）
// 3. 使用 try-finally 确保 remove
//
// ThreadLocalMap 对 null key 的处理：
// get/set 操作时会探测清除（expungeStaleEntry）null key 的 entry
// 但不调用 get/set 的过期 entry 会导致泄漏
```

### Q7：什么是伪共享（False Sharing）？如何解决？

```java
// CPU 缓存行（Cache Line）通常为 64 字节
// 如果两个变量在同一个缓存行中，且被不同线程修改
// 会导致缓存行失效，每次访问都需要重新从内存加载

// ┌─── 缓存行 64 字节 ───┐
// │ 变量 A (线程 1 修改)   │
// │ 变量 B (线程 2 修改)   │ ← 线程 1 修改 A 导致 B 的缓存也失效
// └───────────────────────┘

// Jed 8 中使用 @Contended 注解（需要 JVM 参数：-XX:-RestrictContended）
@sun.misc.Contended
public class VolatileLong {
    public volatile long value = 0L;
}

// 手动填充（JDK 8 之前）：
public class VolatileLongPadded {
    public volatile long value = 0L;
    // 填充 6 个 long（64 - 8 - 8 = 48 字节，再加对象头等）
    public long p1, p2, p3, p4, p5, p6;  // 旧方法
}

// 实际案例：LongAdder 的 Cell 类使用 @Contended 防止伪共享
// 伪共享对高并发计数器影响巨大（性能可差 10 倍以上）
```

---

## 📚 参考资料

- 《Java 并发编程的艺术》—— AQS 原理、锁优化
- 《Java 并发编程实战》—— Doug Lea 经典著作
- JUC 源码：java.util.concurrent 包
- [Oracle 官方文档：Concurrency](https://docs.oracle.com/javase/tutorial/essential/concurrency/)

---

> 📝 **提示**：本章篇幅较长，建议按"AQS 原理 → 显式锁 → 并发集合 → 线程池 → 异步编程 → 同步工具"的顺序学习，每个部分都可以独立掌握。
