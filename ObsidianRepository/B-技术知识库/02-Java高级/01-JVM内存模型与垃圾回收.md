# JVM 内存模型与垃圾回收机制

## 📖 本章导读

### 学习目标
通过本章学习，你将掌握：
- ✅ JVM 运行时数据区（堆、栈、方法区等）的结构和用途
- ✅ 对象创建过程与内存布局（对象头、指针压缩）
- ✅ GC 算法（标记-清除、标记-复制、标记-整理、分代收集）
- ✅ HotSpot 七种垃圾收集器的原理与对比
- ✅ GC 日志分析与调优思路
- ✅ JMM（Java 内存模型）与 happens-before 原则

### 核心概念
**JVM（Java Virtual Machine）** 是 Java 跨平台的基石。理解 JVM 内存模型是进行性能调优和排查内存问题的前提。**垃圾回收（GC）** 是 JVM 自动管理内存的核心机制，不同的 GC 算法和收集器适用于不同的应用场景。

### 知识地图
```
JVM 内存与 GC 体系
├── 运行时数据区
│   ├── 线程私有
│   │   ├── 程序计数器（Program Counter Register）
│   │   ├── Java 虚拟机栈（VM Stack）
│   │   │   ├── 栈帧组成：局部变量表、操作数栈、动态连接、返回地址
│   │   │   └── StackOverflowError / OutOfMemoryError
│   │   └── 本地方法栈（Native Method Stack）
│   └── 线程共享
│       ├── 堆（Heap）
│       │   ├── 新生代（Young Gen）：Eden + S0 + S1
│       │   ├── 老年代（Old Gen）
│       │   └── 元空间/永久代（Metaspace/PermGen）
│       ├── 方法区（Method Area）
│       └── 直接内存（Direct Memory）
├── 对象创建与内存布局
│   ├── 创建过程：类加载 → 分配内存 → 初始化 → 构造器
│   ├── 内存布局：对象头 + 实例数据 + 对齐填充
│   └── 对象访问：句柄 / 直接指针
├── GC 算法
│   ├── 标记-清除（Mark-Sweep）：碎片化
│   ├── 标记-复制（Mark-Copy）：浪费空间
│   ├── 标记-整理（Mark-Compact）：性能开销
│   └── 分代收集（Generational Collection）
├── 垃圾收集器
│   ├── Serial / Serial Old：单线程、暂停所有用户线程
│   ├── ParNew：Serial 的多线程版本
│   ├── Parallel Scavenge / Parallel Old：吞吐量优先
│   ├── CMS：低延迟、并发标记清除
│   ├── G1：区域化分代、可预测停顿
│   └── ZGC / Shenandoah：超低延迟（<10ms）
├── GC 调优
│   ├── 常用 JVM 参数
│   └── 日志分析与 OOM 排查
└── JMM（Java 内存模型）
    ├── 主内存与工作内存
    ├── happens-before 原则
    └── 内存屏障
```

---

## 1. JVM 运行时数据区

### 1.1 整体架构

```
┌─────────────────────────────────────────────────┐
│                   线程私有                         │
│  ┌──────────────┐  ┌──────────────┐  ┌────────┐  │
│  │ 程序计数器    │  │ 虚拟机栈     │  │ 本地栈 │  │
│  │（PC Register）│  │（VM Stack）  │  │(Native)│  │
│  └──────────────┘  └──────────────┘  └────────┘  │
├─────────────────────────────────────────────────┤
│                   线程共享                         │
│  ┌───────────────────────────────────────────┐   │
│  │                 堆（Heap）                  │   │
│  │  ┌───────┬──────────┬──────────────────┐  │   │
│  │  │ Eden  │  S0/S1   │     Old Gen      │  │   │
│  │  └───────┴──────────┴──────────────────┘  │   │
│  └───────────────────────────────────────────┘   │
│  ┌───────────────────────────────────────────┐   │
│  │              方法区（Method Area）          │   │
│  │  类信息、常量池、静态变量、JIT 编译代码      │   │
│  └───────────────────────────────────────────┘   │
│  ┌───────────────────────────────────────────┐   │
│  │             直接内存（Direct Memory）       │   │
│  │         NIO 的 ByteBuffer.allocateDirect()  │   │
│  └───────────────────────────────────────────┘   │
└─────────────────────────────────────────────────┘
```

### 1.2 程序计数器

```java
// 当前线程执行的字节码行号指示器
// 线程私有，每个线程都有一个独立的 PC

// 特点：
// - 记录当前线程正在执行的字节码指令地址
// - 如果执行 Native 方法，PC 为空（Undefined）
// - 唯一不会发生 OOM 的区域
// - 线程切换后能恢复到正确的执行位置
```

### 1.3 Java 虚拟机栈

```java
// 每个方法执行时创建一个栈帧（Stack Frame）
// 线程私有，生命周期与线程相同

// 栈帧组成：
/*
┌─────────────────────────────────┐
│         局部变量表（Local Variables）     │ ← 方法参数 + 局部变量
│              操作数栈（Operand Stack）   │ ← 字节码指令的工作区
│              动态连接（Dynamic Linking） │ ← 运行时常量池的方法引用
│              返回地址（Return Address）  │ ← 调用者的 PC
└─────────────────────────────────┘
*/

// 局部变量表：
// - 以 Slot（槽）为单位，每个 Slot 32 位
// - long 和 double 占 2 个 Slot
// - 实例方法中，第 0 个 Slot 是 this 引用

// 栈大小设置：
// -Xss256k      // 设置栈大小为 256KB
// -Xss1m        // 设置栈大小为 1MB（默认）

// 栈异常：
// - StackOverflowError：线程请求的栈深度 > 虚拟机允许的深度
//   （常见于递归无终止条件）
// - OutOfMemoryError：栈扩展时无法申请到足够内存
//   （线程数过多，例如 -Xss 设置过小且创建大量线程）
```

**StackOverflowError 示例：**
```java
public class StackOverflowDemo {
    private static int depth = 0;

    public static void main(String[] args) {
        try {
            recursive();
        } catch (StackOverflowError e) {
            System.out.println("递归深度: " + depth);  // 取决于 -Xss 设置
        }
    }

    public static void recursive() {
        depth++;
        recursive();
    }
}
// 默认 -Xss1m 时，递归深度约 10000-20000
// -Xss256k 时，递归深度约 2000-4000
```

### 1.4 堆（Heap）

```java
// 堆是 JVM 中最大的一块内存区域
// 被所有线程共享，在 JVM 启动时创建
// 唯一目的是存放对象实例

// 堆的分代结构（HotSpot）：
/*
┌─────────────────────────────────────────────────┐
│              堆（Heap）                           │
│  ┌──────────────────────┬─────────────────────┐  │
│  │    新生代（Young）    │     老年代（Old）     │  │
│  │ ┌───┬────┬────┐      │                     │  │
│  │ │E  │ S0 │ S1 │      │                     │  │
│  │ └───┴────┴────┘      │                     │  │
│  │   8:1:1 比例         │                     │  │
│  └──────────────────────┴─────────────────────┘  │
└─────────────────────────────────────────────────┘

默认比例：新生代 : 老年代 = 1 : 2（-XX:NewRatio=2）
Eden : S0 : S1 = 8 : 1 : 1（-XX:SurvivorRatio=8）
*/

// 堆参数设置：
// -Xms10g                  // 堆初始大小
// -Xmx10g                  // 堆最大大小（通常与 -Xms 相同，避免动态调整）
// -Xmn2g                   // 新生代大小
// -XX:NewRatio=2            // 新生代:老年代 = 1:2
// -XX:SurvivorRatio=8       // Eden:S0:S1 = 8:1:1

// 堆异常：
// OutOfMemoryError: Java heap space
//   → 对象无法分配到堆内存
//   常见原因：大对象、内存泄漏、堆过小
```

### 1.5 方法区（Method Area）

```java
// 方法区（JDK 8 之前）→ 元空间（JDK 8+）
// 存储类信息、常量、静态变量、JIT 编译后的代码

// JDK 7 → JDK 8 的变化：
// - 字符串常量池从永久代移到堆
// - 类元数据从永久代移到元空间（本地内存）

/*
JDK 7 及以前：
┌─────────────────────┐
│  方法区（永久代）      │
│  类信息    常量池      │
│  静态变量  JIT代码     │
└─────────────────────┘
PermGen 大小：-XX:MaxPermSize=256m

JDK 8+：
┌─────────────────────┐
│  元空间（Metaspace）  │  ← 使用本地内存
│  类信息    JIT代码    │
└─────────────────────┘
┌─────────────────────┐
│       堆             │
│  ... + 字符串常量池    │  ← 移到堆中
└─────────────────────┘
元空间大小：-XX:MaxMetaspaceSize（默认无上限，直到物理内存耗尽）
*/

// 方法区异常：
// JDK 7：OutOfMemoryError: PermGen space
//   （大量类加载/动态生成类时常见）
// JDK 8+：OutOfMemoryError: Metaspace

// 元空间改用本地内存的好处：
// 1. 减少 OOM 概率（应用有更多控制权）
// 2. 无需 GC 调优（本地内存由 OS 管理）
// 3. 不再需要设置 PermSize
```

### 1.6 直接内存（Direct Memory）

```java
// NIO 引入的 DirectByteBuffer，使用本地内存
// 不是 JVM 运行时数据区的一部分
// 通过 Unsafe.allocateMemory() 分配

ByteBuffer buffer = ByteBuffer.allocateDirect(1024 * 1024);  // 1MB 直接内存

// 直接内存优点：
// - 减少数据在用户态和内核态之间的拷贝（零拷贝）
// - 适合大块数据的读写（文件、网络）

// 直接内存缺点：
// - 分配和释放成本高
// - 需要手动管理（DirectByteBuffer 被 GC 时才会释放）
// - 仍然可能 OOM（OutOfMemoryError: Direct buffer memory）

// 限制直接内存大小：
// -XX:MaxDirectMemorySize=1g  // 默认等于 -Xmx
```

---

## 2. 对象创建与内存布局

### 2.1 对象的创建过程

```java
Object obj = new Object();

// JVM 层面执行以下步骤：
```

```
1. 类加载检查：
   - 检查 Object 类是否已加载、解析、初始化
   - 如果没有，执行类加载过程（加载→验证→准备→解析→初始化）

2. 分配内存：
   - 根据类信息确定对象所需内存大小
   - 在堆中分配连续内存
   - 分配方式：
     a. 指针碰撞（Bump the Pointer）：堆规整（Serial、ParNew）
     b. 空闲列表（Free List）：堆不规整（CMS）
   - 线程安全：TLAB（Thread Local Allocation Buffer）
     - 每个线程在 Eden 区预先分配一块缓冲区
     - 对象优先在 TLAB 中分配（-XX:+UseTLAB）
     - TLAB 不够时，使用 CAS 加锁分配

3. 设置对象头：
   - Mark Word：哈希码、GC 年龄、锁状态
   - Klass Pointer：指向方法区中的类元数据
   - 数组长度（如果是数组对象）

4. 执行 <init> 方法：
   - 实例变量赋默认值（已在准备阶段完成）
   - 执行实例初始化块
   - 执行构造器
```

**TLAB 示意图：**
```
┌──────────────────────────────────────┐
│             Eden 区                    │
│  ┌─────┐┌──────┐┌────┐┌──────────┐  │
│  │TLAB1││TLAB2 ││... ││ 共享区域  │  │
│  │线程1││线程2 ││    ││   CAS    │  │
│  └─────┘└──────┘└────┘└──────────┘  │
└──────────────────────────────────────┘
// TLAB 参数：
// -XX:+UseTLAB               // 开启 TLAB（默认）
// -XX:TLABSize=2m            // TLAB 大小
// -XX:ResizeTLAB             // 允许动态调整 TLAB 大小
```

### 2.2 对象的内存布局（HotSpot）

```java
// HotSpot 虚拟机中，对象在堆中的存储布局分为三部分：
```

```
┌─────────────────────────────────────┐
│         对象头（Header）              │  ← 固定大小
│  ┌───────────────────────────────┐  │
│  │      Mark Word（标记字）        │  │  ← 8 字节（64 位）
│  │  ├─ 哈希码（25位）              │  │
│  │  ├─ GC 年龄（4位）              │  │
│  │  ├─ 锁状态标志（2位）            │  │
│  │  ├─ 偏向线程ID（23位）          │  │
│  │  └─ 偏向时间戳                  │  │
│  ├───────────────────────────────┤  │
│  │  Klass Pointer（类指针）        │  │  ← 4 字节（压缩）/ 8 字节
│  └───────────────────────────────┘  │
├─────────────────────────────────────┤
│     实例数据（Instance Data）         │  ← 各种类型字段
│  ┌───────────────────────────────┐  │
│  │   父类字段                      │  │
│  ├───────────────────────────────┤  │
│  │   本类字段                      │  │  ← 按大小排序
│  │   long/double → int → short   │  │
│  └───────────────────────────────┘  │
├─────────────────────────────────────┤
│     对齐填充（Padding）              │  ← 8 字节倍数
└─────────────────────────────────────┘
```

**指针压缩（Compressed OOPs）：**
```java
// 通过 -XX:+UseCompressedOops 控制（JDK 8+ 默认开启）
// 将 8 字节的引用压缩为 4 字节

// 原理：
// - 对象都是 8 字节对齐的（地址末 3 位为 0）
// - 将 35 位地址右移 3 位 → 用 32 位存储
// - 使用时左移 3 位 → 恢复 35 位地址

// 限制：
// - 堆大小 ≤ 32GB 时有效
// - 堆 > 32GB 时自动关闭

// 计算对象大小（用 jol 工具）：
// 依赖：org.openjdk.jol:jol-core
// ClassLayout.parseClass(Object.class).toPrintable()

/*
普通 Object（64 位 JVM，指针压缩）：
Mark Word: 8 字节
Klass Pointer: 4 字节（压缩）
Padding: 4 字节（对齐到 8）
总大小: 16 字节
*/
```

### 2.3 对象的访问定位

```java
// 两种访问方式：

// 1. 句柄访问：
/*
Java 栈                   堆                     方法区
┌─────────┐        ┌───────────────┐        ┌──────────┐
│ ref     │──────→│ 句柄池         │        │ 类信息   │
│         │        │ ┌───────────┐ │        │          │
│         │        │ │实例指针→对象│ │──────→│          │
│         │        │ │类型指针→类  │─┘        └──────────┘
└─────────┘        │ └───────────┘ │
                    └───────────────┘
优点：对象移动时只需修改句柄的实例指针，引用本身不变
缺点：两次指针访问，速度慢
*/

// 2. 直接指针（HotSpot 默认）：
/*
Java 栈                   堆                     方法区
┌─────────┐        ┌───────────────┐        ┌──────────┐
│ ref     │──────→│ 对象实例       │        │ 类信息   │
│         │        │ ┌───────────┐ │        │          │
│         │        │ │ Mark Word │ │        │          │
│         │        │ │ Klass Ptr │─┘──────→│          │
│         │        │ │ 实例数据  │ │        └──────────┘
└─────────┘        │ └───────────┘ │
                    └───────────────┘
优点：一次指针访问，速度快
缺点：对象移动时需要更新引用
*/
```

---

## 3. GC 算法

### 3.1 可达性分析算法

```java
// 主流的 GC 算法都是通过"可达性分析"来判断对象是否存活

// 基本思路：
// 1. 从 GC Roots 开始向下搜索
// 2. 搜索经过的路径称为"引用链"
// 3. 不在引用链上的对象 → 不可达 → 判定为可回收

// GC Roots 包括：
// - 虚拟机栈中引用的对象（局部变量表中的引用）
// - 静态属性引用的对象（方法区中的静态字段）
// - 方法区中常量引用的对象（如字符串常量池引用）
// - 本地方法栈中 JNI 引用的对象
// - 活跃线程对象
```

```
GC Roots → 可达对象 → 存活
              ↓
           不可达对象 → 标记为可回收
```

### 3.2 引用类型

```java
// Java 提供四种引用类型（java.lang.ref 包）

// 1. 强引用（Strong Reference）：最常见的引用
Object obj = new Object();           // 强引用
// 只要强引用还在，GC 永远不会回收

// 2. 软引用（SoftReference）：内存不足时回收
SoftReference<Object> softRef = new SoftReference<>(new Object());
Object obj = softRef.get();          // 可能为 null（被回收）
// 用途：缓存（直到内存不足前都可用）
// -XX:SoftRefLRUPolicyMSPerMB=1000  // 软引用存活时间

// 3. 弱引用（WeakReference）：下次 GC 即回收
WeakReference<Object> weakRef = new WeakReference<>(new Object());
Object obj = weakRef.get();          // GC 后可能为 null
// 用途：ThreadLocalMap（key 是弱引用）、WeakHashMap

// 4. 虚引用（PhantomReference）：无法通过 get() 获取对象
ReferenceQueue<Object> queue = new ReferenceQueue<>();
PhantomReference<Object> phantomRef = new PhantomReference<>(new Object(), queue);
// phantomRef.get() 永远返回 null
// 用途：对象被回收后的通知（DirectByteBuffer 的回收清理）
```

### 3.3 常见的 GC 算法

**标记-清除（Mark-Sweep）：**
```
算法步骤：
1. 标记所有可达对象
2. 清除所有不可达对象（将空闲内存加入空闲列表）

优点：简单、不需要额外空间
缺点：产生内存碎片、标记和清除都需要暂停线程

图解：
标记前：[A][B][C][ ][D][E][ ][F]
标记后：[A][B][C][X][D][E][X][F]  (X 表示可回收)
清除后：[A][B][C][  ][D][E][  ][F]  (碎片化！)
```

**标记-复制（Mark-Copy）：**
```
算法步骤：
1. 将内存分为两块等大小区域 A 和 B
2. 对象在 A 中分配
3. 回收时，将 A 中存活对象复制到 B
4. 清空 A，然后交换 A 和 B 的角色

优点：无碎片、分配高效（指针碰撞）
缺点：浪费 50% 空间、对象存活率高时复制开销大

图解（HotSpot 新生代收集）：
┌──────┬─────┬─────┐      ┌──────┬─────┬─────┐
│ Eden │ S0  │ S1  │  →   │ Eden │ S0  │ S1  │
│      │     │     │      │      │     │     │
│      │活对象│     │      │ 空   │ 空  │活对象│
└──────┴─────┴─────┘      └──────┴─────┴─────┘
   from  to         交换后  to   from
```

**标记-整理（Mark-Compact）：**
```
算法步骤：
1. 标记所有可达对象
2. 将所有存活对象向一端移动
3. 清理边界以外的内存

优点：无碎片、空间利用率高
缺点：对象移动需要暂停线程，开销大

图解：
标记前：[A][B][C][ ][D][E][ ][F]
标记后：[A][B][C][X][D][E][X][F]
整理后：[A][B][C][D][E][F][  ][  ]
```

**分代收集（Generational Collection）：**
```
将堆分为新生代和老年代，使用不同的算法：

新生代（Young Gen）：标记-复制算法
  - 对象存活率低，复制开销小
  - 只需少量空间（Eden:S0:S1 = 8:1:1，浪费 10%）

老年代（Old Gen）：标记-整理 / 标记-清除算法
  - 对象存活率高，复制代价大
  - 使用整理算法或并发清除（CMS）

分代假设（弱分代假说）：
  绝大多数对象朝生夕死（分配后很快就不可达）
```

### 3.4 GC 触发条件

```java
/*
Minor GC (Young GC)：新生代垃圾回收
  - 触发条件：Eden 区满
  - 特点：速度快（几十毫秒），频繁

Major GC / Full GC：整个堆的回收
  - 触发条件：
    a. 老年代空间不足
    b. System.gc()（建议性）
    c. 晋升失败（担保失败）
    d. 元空间/Metaspace 不足
    e. CMS 并发模式失败（Concurrent Mode Failure）
  - 特点：速度慢（几百毫秒到几秒），尽量少触发

空间分配担保：
  - Minor GC 前，JVM 检查老年代最大可用连续空间
  - 如果 > 新生代所有对象总大小 → 安全（可 Minor GC）
  - 否则检查是否允许担保失败（-XX:-HandlePromotionFailure）
  - 如果不允许 → 直接 Full GC
*/
```

---

## 4. 垃圾收集器

### 4.1 收集器总览

```
                     新生代                     老年代
                ┌────────────┐          ┌────────────┐
                │  Serial    │          │ Serial Old │
                │（单线程）   │          │（单线程+MSC）│
                ├────────────┤          ├────────────┤
                │  ParNew    │          │   CMS      │
                │（并行）     │          │（低延迟）   │
                ├────────────┤          ├────────────┤
                │ Parallel   │          │Parallel Old│
                │（吞吐量优先）│          │（吞吐量优先）│
                └────────────┘          └────────────┘

                全功能区域化收集器：
                ┌───────────────────────────┐
                │     G1（GC 预测停顿）       │
                ├───────────────────────────┤
                │  ZGC（超低延迟 < 10ms）    │
                └───────────────────────────┘
连接线表示可组合使用（JDK 8 及以前）：
  Serial + Serial Old
  ParNew + CMS
  Parallel Scavenge + Parallel Old
  G1 独立使用（不需要组合）
  ZGC 独立使用
```

### 4.2 新生代收集器

**Serial 收集器：**
```java
// -XX:+UseSerialGC          // 启用 Serial + Serial Old

// 特点：
// - 单线程收集（一个 CPU 或一条线程完成 GC）
// - 需要暂停所有用户线程（Stop The World）
// - 简单高效，没有线程交互开销
// - Client 模式下的默认收集器

// 适用场景：单核 CPU、小内存（<100MB）、桌面应用
```

**ParNew 收集器：**
```java
// -XX:+UseParNewGC          // 启用 ParNew + CMS（JDK 9 前）

// 特点：
// - Serial 的多线程版本（并行收集）
// - 必须配合 CMS 使用（JDK 9 之后被移除）
// - 仍然会 STW（Stop The World）
// - -XX:ParallelGCThreads=4  // 并行线程数

// 适用场景：多核 CPU + CMS 的组合
```

**Parallel Scavenge 收集器：**
```java
// -XX:+UseParallelGC        // 启用 Parallel Scavenge + Parallel Old

// 特点：
// - 关注吞吐量（Throughput）
// - 自适应调节（-XX:+UseAdaptiveSizePolicy）
// - 可控制 GC 时间

// 关键参数：
// -XX:MaxGCPauseMillis=50   // 最大 GC 停顿（毫秒），默认无限制
// -XX:GCTimeRatio=99        // 吞吐量 = 1 / (1 + 1/99) = 99%
// 注意：MaxGCPauseMillis 设得太小会导致频繁 GC

// 适用场景：后台计算、批处理、数据分析
```

### 4.3 CMS 收集器

```java
// -XX:+UseConcMarkSweepGC   // 启用 ParNew + CMS + Serial Old（后备）
// -XX:CMSInitiatingOccupancyFraction=75  // 老年代 75% 时触发

// 特点：并发收集、低延迟、不压缩

// 工作流程（四步）：
/*
1. 初始标记（Initial Mark）—— STW
   标记 GC Roots 直接关联的对象 → 快（只标记一层）

2. 并发标记（Concurrent Mark）
   从 GC Roots 开始遍历整个对象图
   和用户线程同时运行 → 时间长但不停顿

3. 重新标记（Remark）—— STW
   修正并发标记期间因用户程序运行而变动的标记
   使用增量更新（Incremental Update）

4. 并发清除（Concurrent Sweep）
   清除标记为不可达的对象
   和用户线程同时运行
*/

// CMS 的缺点：
// 1. 浮动垃圾（Floating Garbage）：并发阶段新产生的垃圾只能在下次 GC 回收
// 2. 并发模式失败（Concurrent Mode Failure）：
//    老年代空间不够 → 退化为 Serial Old（Full GC，长时间 STW）
// 3. 内存碎片（不压缩）：老年代使用标记-清除 → 碎片化
//    碎片化 → 无法分配大对象 → 提前 Full GC
//    解决方案：-XX:+UseCMSCompactAtFullCollection

// JDK 9 标记为弃用，JDK 14 正式移除
```

### 4.4 G1 收集器

```java
// -XX:+UseG1GC              // 启用 G1（JDK 9+ 默认）
// -XX:MaxGCPauseMillis=200  // 目标停顿时间（默认 200ms）
// -XX:G1HeapRegionSize=4m   // Region 大小（1MB~32MB，默认堆/2048）

// 核心特点：
// - 区域化堆结构（Region）
// - 可预测停顿时间
// - 不再区分新生代/老年代物理区域
// - 整体基于标记-整理，局部基于复制（无碎片）

// Region 划分：
/*
堆被划分为 2048 个 Region（默认），每个 Region 大小 1MB~32MB
┌──────────────────────────────────────────┐
│  E │  E │  E │  S │  O │  O │  O │  H  │
│  E │  E │  E │  E │  O │  O │  H │  O  │
│  E │  E │  S │  E │  O │  O │  O │  H  │
│  E │  E │  S │  O │  O │  O │  O │  H  │
└──────────────────────────────────────────┘
E = Eden Region
S = Survivor Region
O = Old Region
H = Humongous Region（大对象，大小 ≥ Region 的 50%）
*/

// G1 的 GC 类型：
/*
1. Young GC（Minor GC）：
   选择所有 Eden Region，将存活对象复制到 Survivor Region

2. Mixed GC（混合 GC）：
   选择所有 Eden Region + 部分高收益 Old Region
   基于停顿预测模型选择回收哪些 Region

3. Full GC：
   如果对象分配过快，Mixed GC 来不及回收
   退化为单线程 Full GC（Serial Old）
*/

// Remembered Set（RSet）：
// - 记录其他 Region 对当前 Region 的引用
// - 避免全堆扫描
// - 每个 Region 都有一个 RSet
// - RSet 占用堆内存的 5%~20%

// G1 工作流程：
/*
1. 初始标记 —— STW（伴随 Young GC 一起）
2. 并发标记
3. 最终标记 —— STW
4. 筛选回收 —— STW（按收益排序 Region，分批回收）
*/

// G1 调优建议：
// - 不要设置 -Xmn（G1 自动调整新生代大小）
// - 目标停顿时间酌情设置（100ms~500ms）
// - Region 大小不要手动设置，让 G1 自动决定
// - 监控 Full GC 次数，如果频繁说明需要调优
```

### 4.5 ZGC 收集器

```java
// -XX:+UseZGC              // 启用 ZGC（JDK 11+ 实验，JDK 15+ 正式）
// -XX:ZAllocationSpikeTolerance=2.0  // 分配尖峰容忍度

// 特点：超低延迟（<10ms）、与堆大小无关

// 核心技术：
// 1. 染色指针（Colored Pointer）：
//    在 64 位指针中借 4 位存储标记信息
//    无需对象头标记，无需写屏障

// 2. 读屏障（Load Barrier）：
//    从堆加载引用时检查指针颜色
//    如果被标记为需要修正，则修正指针
//    比 G1 的写屏障影响面小

// 3. 并发整理：
//    ZGC 的整理操作是并发的（不 STW）
//    使用转发表（Forward Table）处理并发移动

// ZGC 的限制：
// - 不支持 32 位平台
// - 需要多核 CPU
// - 堆不能太大（目前不支持 >16TB，理论限制 4TB）
```

### 4.6 收集器对比总结

| 收集器 | 类型 | 并行 | 并发 | STW | 适用场景 |
|:---|:---|:---:|:---:|:---:|:---|
| Serial | 新生代 | ❌ | ❌ | ⬛⬛⬛ | 单核、小内存 |
| ParNew | 新生代 | ✅ | ❌ | ⬛⬛ | 多核+CMS |
| Parallel | 新生代 | ✅ | ❌ | ⬛⬛ | 吞吐量优先 |
| Serial Old | 老年代 | ❌ | ❌ | ⬛⬛⬛ | CMS后备 |
| Parallel Old | 老年代 | ✅ | ❌ | ⬛⬛ | 吞吐量 |
| CMS | 老年代 | ✅ | ✅ | ⬛ | 低延迟 |
| G1 | 全区域 | ✅ | ✅ | ⬛ | 大堆（4GB+） |
| ZGC | 全区域 | ✅ | ✅ | 🔲 | 超低延迟 |

---

## 5. GC 调优

### 5.1 常用 JVM 参数

```java
// 堆设置
// -Xms4g                     // 堆初始大小（= -Xmx 避免动态调整）
// -Xmx4g                     // 堆最大大小
// -Xmn2g                     // 新生代大小（G1 下不推荐设置）
// -XX:SurvivorRatio=8         // Eden:S0:S1 = 8:1:1
// -XX:MaxTenuringThreshold=15 // 晋升年龄阈值（默认 15）
// -XX:PretenureSizeThreshold  // 大对象直接进入老年代的阈值

// 元空间设置
// -XX:MetaspaceSize=256m      // 元空间初始大小
// -XX:MaxMetaspaceSize=256m   // 元空间最大大小

// GC 选择
// -XX:+UseG1GC
// -XX:+UseParallelGC
// -XX:+UseZGC
// -XX:MaxGCPauseMillis=200

// GC 日志（JDK 8）
// -XX:+PrintGCDetails
// -XX:+PrintGCDateStamps
// -XX:+PrintHeapAtGC
// -Xloggc:gc.log

// GC 日志（JDK 11+）
// -Xlog:gc*:file=gc.log:time,uptime,level,tags

// OOM 时堆转储
// -XX:+HeapDumpOnOutOfMemoryError
// -XX:HeapDumpPath=/path/to/dump.hprof

// 其他
// -XX:+DisableExplicitGC      // 禁用 System.gc()
// -XX:+PrintCommandLineFlags  // 打印最终生效的参数
```

### 5.2 GC 日志分析

```java
// G1 GC 日志示例（JDK 11+）：
/*
[2024-01-15T10:30:00.123+0800] GC(0) Pause Young (Normal) (G1 Evacuation Pause)
  {Heap before GC: ...}
  {Heap after GC: ...}
  [Eden: 1024.0M(1024.0M)->0.0B(1024.0M)
   Survivors: 0.0B->16.0M
   Heap: 2048.0M(4096.0M)->1024.0M(4096.0M)]
  [Times: user=0.12 sys=0.01, real=0.03 secs]
                                      ↑
                                      real 是实际停顿时间
                                      user 是 CPU 时间（多线程会累加）

Full GC 日志（危险信号）：
[2024-01-15T10:31:00.456+0800] GC(5) Pause Full (G1 Evacuation Pause)
  Times: user=2.50 sys=0.10, real=1.20 secs
  ↑ real=1.2s → 业务线程停顿 1.2 秒！
*/

// GC 日志关键指标：
// - Eden 区回收效率（每次回收 90%+ 正常）
// - Full GC 频率（尽可能为 0）
// - GC 停顿时间（是否在目标范围内）
// - 晋升到老年代的对象大小
// - 老年代使用率增长趋势
```

### 5.3 内存泄漏排查

```java
// 典型的 OOM 场景与排查思路：

// 1. Java heap space
//    - 排查：jmap -heap [pid] / jhat / Eclipse MAT
//    - 命令：jmap -dump:format=b,file=heap.hprof [pid]
//    - 分析：MAT 中的 Leak Suspects Report
//    - 常见原因：
//      a. 集合类泄漏（HashMap 一直 add 不 remove）
//      b. 内部类/匿名类持有外部类引用
//      c. 连接未关闭（数据库、文件、网络）
//      d. 静态集合（static List 无限增长）

// 2. Metaspace
//    - 排查：是否动态生成了大量类（CGLIB、动态代理）
//    - 常见原因：AOP 框架使用不当、热部署未清理

// 3. Direct buffer memory
//    - 排查：NIO/DirectByteBuffer 使用不当
//    - 常见原因：ByteBuffer.allocateDirect() 未释放

// 4. GC overhead limit exceeded
//    - JVM 花大量时间做 GC 但效果很小（>98% 时间在 GC，堆回收 <2%）
//    - 基本确定是内存泄漏或堆过小

// jdk.jcmd 工具链：
// jps / jstat / jmap / jstack / jinfo
// jcmd [pid] GC.heap_info
// jcmd [pid] Thread.print
```

---

## 6. JMM（Java 内存模型）

### 6.1 主内存与工作内存

```java
// JMM 定义了线程与主内存之间的关系：
// - 所有变量存储在主内存（堆）
// - 每个线程有自己的工作内存（栈/寄存器）
// - 线程不能直接操作主内存，必须先把变量复制到工作内存

/*
┌──────────────┐          ┌──────────────┐
│   线程 A      │          │   线程 B      │
│ ┌──────────┐ │          │ ┌──────────┐ │
│ │ 工作内存  │ │          │ │ 工作内存  │ │
│ │ x = 0    │ │          │ │ x = 0    │ │
│ └──────────┘ │          │ └──────────┘ │
└──────┬───────┘          └──────┬───────┘
       ↓                         ↓
┌─────────────────────────────────────┐
│              主内存                   │
│          x = 0 → 1 → 2               │
└─────────────────────────────────────┘

线程 A 修改 x = 1，但线程 B 可能仍然看到 x = 0
除非 A 把 x 刷回主内存，B 从主内存重新加载
这就是"可见性"问题
*/
```

### 6.2 happens-before 原则

```java
// 如果一个操作 happens-before 另一个操作，第一个操作对第二个操作可见

// JMM 定义的 8 条 happens-before 规则：

// 1. 程序次序规则：同一线程中，前一个操作 happens-before 后一个操作
// 2. volatile 规则：volatile 写 happens-before volatile 读
// 3. 锁规则：解锁 happens-before 加锁
// 4. 传递性：A->B, B->C ⇒ A->C
// 5. 线程启动规则：Thread.start() happens-before 线程中任何操作
// 6. 线程终止规则：线程中所有操作 happens-before Thread.join()
// 7. 中断规则：interrupt() happens-before 检测到中断
// 8. 对象终结规则：构造器结束 happens-before finalize() 开始

// 示例：
volatile boolean flag = false;
int x = 0;

// 线程 A
x = 42;          // 1
flag = true;     // 2（volatile 写）

// 线程 B
if (flag) {      // 3（volatile 读）
    System.out.println(x);  // 4 → 一定是 42！
}
// 因为：
// 1 happens-before 2（程序次序）
// 2 happens-before 3（volatile 规则）
// 3 happens-before 4（程序次序）
// 传递性：1 → 2 → 3 → 4，所以 x = 42 对线程 B 可见
```

### 6.3 内存屏障

```java
// JMM 通过内存屏障（Memory Barrier）禁止特定类型的指令重排序

// 四种内存屏障（JSR-133）：
/*
屏障类型        指令示例          说明
LoadLoad       Load1; LoadLoad; Load2    Load1 先于 Load2
StoreStore     Store1; StoreStore; Store2  Store1 先于 Store2
LoadStore      Load1; LoadStore; Store2   Load1 先于 Store2
StoreLoad      Store1; StoreLoad; Load2    Store1 先于 Load2（最重，全局刷新）
*/

// volatile 写的语义：
// - 插入 StoreStore 屏障（禁止前面的写与 volatile 写重排序）
// - 插入 StoreLoad 屏障（禁止 volatile 写与后续读写重排序）

// volatile 读的语义：
// - 插入 LoadLoad 屏障（禁止 volatile 读与后续读重排序）
// - 插入 LoadStore 屏障（禁止 volatile 读与后续写重排序）

// 锁的语义：
// - 加锁：读取主内存数据（相当于插入 LoadLoad + LoadStore）
// - 解锁：刷新工作内存到主内存（相当于插入 StoreStore + StoreLoad）
```

---

## 7. 面试题

### Q1: 对象在堆中的生命周期？
```
1. 对象在 Eden 区分配
2. 第一次 Minor GC：如果存活，进入 S0（年龄 = 1）
3. 第二次 Minor GC：S0 中存活对象复制到 S1（年龄++），清空 S0
4. 每次 Minor GC 交换 S0/S1，年龄达到阈值（默认 15）→ 晋升老年代
5. 动态年龄判定：如果 S0/S1 中相同年龄对象总和 > 区域 50%，大的年龄作为晋升阈值

特殊情况：
- 大对象（大于 PretenureSizeThreshold）直接进入老年代
- 如果 Survivor 区放不下，提前进入老年代
```

### Q2: 如何选择垃圾收集器？
```java
// 堆 < 4GB，单机单核：Serial（客户端默认）
// 堆 < 4GB，多核，高吞吐：Parallel（服务端默认 JDK 8 以前）
// 堆 4GB+，延迟敏感（<200ms）：G1
// 堆 4GB+，超低延迟（<10ms）：ZGC（JDK 15+）
// 应用场景参考：
//   - 批处理/离线计算 → Parallel（吞吐量优先）
//   - Web 服务/中间件 → G1（延迟可控）
//   - 高频交易/实时系统 → ZGC/Shenandoah
```

### Q3: 什么是 Stop-The-World？
```java
// JVM 停止所有用户线程执行 GC 的现象
// 原因：GC 过程中对象引用关系会变，必须"冻结"快照

// STW 的危害：
// - 业务线程暂停（毫秒到秒级）
// - 用户体验下降（卡顿、超时）

// 减少 STW 的策略：
// - CMS/G1/ZGC：并发 GC，尽量缩短 STW
// - GC 调优：减少 GC 频率和单次 GC 时间
// - 大堆 + 高效收集器：G1/ZGC
```

### Q4: 方法区和元空间的区别？
```java
// JDK 7 → JDK 8 的变化：
// 1. 永久代被移除，元空间替代
// 2. 字符串常量池移到堆
// 3. 类元数据使用本地内存

// 影响：
// - 不用再担心 PermGen OOM（元空间默认无上限）
// - 类加载泄漏仍然会耗尽元空间
// - 本地内存不够时仍然会 OOM
```

### Q5: 什么是 TLAB？
```java
// Thread Local Allocation Buffer
// 每个线程在 Eden 区的一块私有分配缓冲区
// 目的：避免多线程分配对象时的锁竞争

// 流程：
// 1. 线程从 Eden 区分配一块 TLAB
// 2. 对象分配在 TLAB 内（无锁）
// 3. TLAB 不够 → 重新分配 TLAB 或 CAS 在 Eden 分配
```

### Q6: JVM 参数 -Xms 和 -Xmx 为什么建议设为相同？
```java
// 避免 JVM 运行时动态调整堆大小（GC 压力转移为内存调整）
// 动态调整会导致额外的系统调用和内存申请/释放
// 生产环境：-Xms = -Xmx 减少运行时波动
```

### Q7: 如何判断对象已死？
```java
// 1. 引用计数法（Python、PHP）：互相引用时无法回收
// 2. 可达性分析（Java 使用）：GC Roots 不可达 → 可回收

// 即使可达性分析标记为不可达，也不是立即回收
// 对象回收需要经过两次标记：
// 第一次标记：不可达
// 第二次标记：finalize() 未被覆盖或已调用过
// 如果 finalize() 中让 this 重新可达 → 对象复活（不推荐使用）
```

---

## 8. 最佳实践

1. **-Xms = -Xmx**：避免运行时动态调整堆大小
2. **选择适合的 GC 收集器**：G1 适合大堆（4GB+），Parallel 适合吞吐优先
3. **启用 GC 日志**：生产环境保留 GC 日志用于排查问题
4. **堆转储**：配置 `-XX:+HeapDumpOnOutOfMemoryError` 自动保存堆快照
5. **监控老年代使用率**：增长趋势过快说明潜在内存泄漏
6. **避免 System.gc()**：`-XX:+DisableExplicitGC` 禁用手动 GC
7. **合理设置元空间大小**：`-XX:MaxMetaspaceSize` 兜底限制
8. **调优 G1 仅设置目标停顿时间**：不要手动调整 Region 大小或新生代比例
9. **关注 Full GC 频率**：Full GC 应该极少发生，否则需要排查
10. **TLAB 默认开启**：无需特殊配置，对象分配性能已优化
