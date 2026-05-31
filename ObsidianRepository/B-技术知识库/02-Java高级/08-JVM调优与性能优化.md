# JVM 调优与性能优化

## 📖 本章导读

### 学习目标
通过本章学习，你将掌握：
- ✅ JVM 参数分类与常用配置
- ✅ GC 日志的解读与分析
- ✅ 内存泄漏的排查方法与工具
- ✅ CPU 高负载/线程阻塞的排查
- ✅ JIT 编译优化与内联
- ✅ 性能监控工具（jstat、jmap、jstack、MAT、Arthas）
- ✅ 系统级性能优化方法论

### 核心概念
**JVM 调优**不是简单地调整几个参数，而是系统性的性能管理过程。核心目标包括：减少 GC 停顿时间、提高吞吐量、避免 OOM、优化响应时间。**性能优化**涵盖从代码层面（算法、数据结构、并发设计）到 JVM 层面（内存分配、GC 策略）再到系统层面（OS 参数、硬件资源）的全链路优化。

### 知识地图
```
JVM 调优与性能优化
├── JVM 参数体系
│   ├── 堆内存（-Xms、-Xmx、-Xmn、-XX:MaxMetaspaceSize）
│   ├── GC 选择（-XX:+UseG1GC、-XX:+UseZGC）
│   ├── GC 日志（JDK 8 vs JDK 11+）
│   ├── 内存溢出（-XX:+HeapDumpOnOutOfMemoryError）
│   └── JIT 编译（-XX:CompileThreshold、-XX:+PrintCompilation）
├── GC 调优
│   ├── 目标：吞吐量优先 vs 延迟优先
│   ├── GC 日志分析
│   ├── G1 调优关键参数
│   ├── ZGC 调优
│   └── GC 调优案例
├── 内存泄漏排查
│   ├── 常见泄漏模式
│   ├── jmap 堆转储
│   ├── MAT 分析（Shallow/Retained Size）
│   └── OOM 类型与原因
├── 性能监控工具
│   ├── JDK 内置工具
│   │   ├── jstat（GC 统计）
│   │   ├── jmap（堆信息）
│   │   ├── jstack（线程栈）
│   │   ├── jinfo（运行时参数）
│   │   └── jcmd（多功能诊断）
│   ├── 可视化工具
│   │   ├── JConsole、VisualVM
│   │   └── GCeasy、GCViewer
│   ├── Arthas（阿里开源诊断工具）
│   └── Java Flight Recorder（JFR）
├── JIT 编译优化
│   ├── 分层编译（C1 + C2）
│   ├── 方法内联
│   ├── 逃逸分析和栈上分配
│   ├── 锁消除和锁粗化
│   └── 即时编译监控
└── 性能优化方法论
    ├── 性能指标（QPS、TP99、RT）
    ├── 性能测试（JMH 微基准）
    ├── 代码级优化
    ├── 数据库优化
    └── 架构级优化
```

### 常见误区
❌ **误区 1**：调优 JVM 参数就能解决所有性能问题
✅ **真相**：90% 的性能问题由代码逻辑和架构设计导致。先优化代码，再考虑 JVM 参数调整。

❌ **误区 2**：堆内存越大越好
✅ **真相**：堆过大导致 GC 时间更长（尤其是 Full GC）。建议堆大小控制在 4-8G（G1），或 16-32G（ZGC）。超过 32G 时指针压缩失效，反而更慢。

❌ **误区 3**：-Xms 和 -Xmx 设成不一样更好
✅ **真相**：生产环境建议 -Xms = -Xmx。避免频繁的堆扩容/缩容操作，减少系统调用开销。

❌ **误区 4**：System.gc() 会立即触发 Full GC
✅ **真相**：System.gc() 只是建议 JVM 执行 GC（通过 Runtime.getRuntime().gc()），JVM 可能忽略。可加 -XX:+DisableExplicitGC 禁用。

### 实战建议
1. **优先定位瓶颈**：用工具确定是 CPU、内存、IO 还是锁竞争导致的性能问题
2. **堆大小建议**：4-8G 用 G1，8G+ 用 ZGC（JDK 17+）
3. **保留 GC 日志**：线上必须开启 GC 日志，用于事后分析
4. **OOM 自动转储**：配置 -XX:+HeapDumpOnOutOfMemoryError
5. **JMH 做微基准**：不要用 for 循环测性能，用 JMH
6. **Arthas 是必备**：线上诊断问题首选 Arthas

---

## 1. JVM 参数体系

### 1.1 参数分类

```java
// JVM 参数分为 3 类：

// 1. 标准参数（所有 JVM 实现都必须支持）
//    java -version
//    java -jar myapp.jar
//    java -Dproperty=value

// 2. -X 参数（非标准化，但大部分 JVM 支持）
//    -Xms4g          初始堆大小
//    -Xmx4g          最大堆大小
//    -Xmn2g          新生代大小
//    -Xss256k        线程栈大小
//    -Xloggc:gc.log  GC 日志文件（JDK 8 及之前）
//    -XX:+PrintGCDetails（JDK 8）

// 3. -XX 参数（JVM 特定，不稳定）
//    -XX:+UseG1GC                使用 G1 收集器
//    -XX:MaxMetaspaceSize=256m  元空间最大大小
//    -XX:MaxDirectMemorySize=1g 直接内存最大大小
//    -XX:+HeapDumpOnOutOfMemoryError  OOM 时转储堆
//    -XX:HeapDumpPath=/tmp/dump.hprof  转储路径
```

### 1.2 常用 JVM 参数模板

```java
// JDK 8 + G1（4G 堆，低延迟）：
JAVA_OPTS="-Xms4g -Xmx4g \
  -XX:+UseG1GC \
  -XX:MaxGCPauseMillis=200 \
  -XX:ParallelGCThreads=4 \
  -XX:ConcGCThreads=2 \
  -XX:+PrintGCDetails \
  -XX:+PrintGCDateStamps \
  -Xloggc:/app/logs/gc.log \
  -XX:+HeapDumpOnOutOfMemoryError \
  -XX:HeapDumpPath=/app/logs/dump.hprof \
  -XX:+DisableExplicitGC \
  -Djava.awt.headless=true"

// JDK 17 + ZGC（8G+ 堆，亚毫秒暂停）：
JAVA_OPTS="-Xms8g -Xmx8g \
  -XX:+UseZGC \
  -XX:MaxGCPauseMillis=1 \
  -Xlog:gc*:file=/app/logs/gc.log:time \
  -XX:+HeapDumpOnOutOfMemoryError \
  -XX:HeapDumpPath=/app/logs/dump.hprof \
  --add-opens java.base/java.lang=ALL-UNNAMED"

// Spring Boot 应用典型配置：
JAVA_OPTS="-Xms2g -Xmx2g \
  -XX:+UseG1GC \
  -XX:+PrintGCDetails \
  -XX:+PrintGCDateStamps \
  -Xloggc:/app/logs/gc.log \
  -XX:+HeapDumpOnOutOfMemoryError \
  -Dspring.profiles.active=prod"
```

### 1.3 元空间与直接内存

```java
// 元空间（Metaspace）：
// JDK 8 替代 PermGen
// 使用本地内存，默认无上限
// -XX:MaxMetaspaceSize=256m
// -XX:MetaspaceSize=128m（触发初始 GC 的阈值）
// 元空间 OOM 原因：CGlib 动态生成类太多、Groovy 频繁编译

// 直接内存（Direct Memory）：
// ByteBuffer.allocateDirect() 使用的堆外内存
// -XX:MaxDirectMemorySize=1g
// 直接内存 OOM：Netty 分配太多 DirectBuffer 未释放
```

---

## 2. GC 调优

### 2.1 GC 日志解读

```java
// JDK 8 GC 日志（-XX:+PrintGCDetails -Xloggc:gc.log）：

// Young GC 日志：
// 2024-01-15T10:30:15.123+0800: 2.345: [GC (Allocation Failure)
//   [PSYoungGen: 1024K->512K(2048K)] 2048K->1024K(4096K), 0.0023456 secs]
// 解读：
//   时间戳: 2.345（JVM 启动后秒数）
//   PSYoungGen: 年轻代从 1024K 降到 512K（总 2048K）
//   整个堆: 从 2048K 降到 1024K（总 4096K）
//   耗时: 0.0023456 秒

// Full GC 日志：
// 2024-01-15T10:30:20.456+0800: 7.456: [Full GC (Metadata GC Threshold)
//   [PSYoungGen: 1024K->0K(2048K)]
//   [ParOldGen: 2048K->1024K(3072K)]
//   3072K->1024K(5120K), [Metaspace: 20480K->20480K(25600K)], 0.123456 secs]
// 解读：
//   Full GC 原因：Metadata GC Threshold（元空间达到阈值）
//   年轻代: 1024K → 0K
//   老年代: 2048K → 1024K
//   整个堆: 3072K → 1024K
//   耗时: 0.123 secs

// JDK 11+ GC 日志（-Xlog:gc*:file=gc.log）：
// [0.123s][info][gc,start] GC(0) Pause Young (G1 Evacuation Pause) 256M->128M(1024M)
// [0.126s][info][gc] GC(0) Pause Young (G1 Evacuation Pause) 256M->128M(1024M) 3.123ms
// [2.456s][info][gc,start] GC(1) Pause Full (System.gc()) 512M->256M(1024M)
// [2.589s][info][gc] GC(1) Pause Full (System.gc()) 512M->256M(1024M) 133ms
```

### 2.2 G1 调优关键参数

```java
// G1 收集器的核心目标：
// 在 MaxGCPauseMillis 时间内尽可能多地回收内存

// G1 关键参数：
-XX:+UseG1GC                   // 启用 G1
-XX:MaxGCPauseMillis=200       // 目标最大暂停时间（默认 200ms）
-XX:G1HeapRegionSize=2m        // Region 大小（1-32MB，默认根据堆大小自动）
-XX:ParallelGCThreads=4        // 并行 GC 线程数
-XX:ConcGCThreads=2            // 并发 GC 线程数（= ParallelGCThreads / 4）
-XX:InitiatingHeapOccupancyPercent=45  // IHOP：触发并发标记的堆占用率（默认 45%）
-XX:G1ReservePercent=10        // 预留空间（防止晋升失败）
-XX:+UnlockExperimentalVMOptions -XX:G1MixedGCLiveThresholdPercent=85
                                // Mixed GC 中 Region 存活对象阈值

// G1 调优思路：
// 1. MaxGCPauseMillis 设太小会增加 Young GC 频率
// 2. IHOP 设太小（30%）→ 过早触发并发标记
// 3. IHOP 设太大（70%）→ 可能发生并发模式失败（Full GC）

// G1 并发模式失败（Concurrent Mode Failure）：
// 原因：并发标记期间对象分配速度超过回收速度
// 表现：退化为 Serial Old 单线程 Full GC（非常慢）
// 解决：增加堆大小、降低 IHOP、增加 ConcGCThreads
```

### 2.3 ZGC 调优

```java
// ZGC（JDK 11 实验，JDK 15+ 正式，JDK 17+ 推荐）
// 目标：不超过 10ms 的暂停时间，与堆大小无关

// ZGC 参数：
-XX:+UseZGC                   // 启用 ZGC
-Xms8g -Xmx8g                 // 建议 8G+（ZGC 需要较大堆来发挥优势）
-XX:ZAllocationSpikeTolerance=2.0  // 分配尖峰容忍度
-XX:ConcGCThreads=2           // 并发 GC 线程数

// ZGC 的特性：
// 1. 染色指针（Colored Pointer）：64 位指针中的 4 位用于标记
// 2. 读屏障（Load Barrier）：读对象时检查指针颜色
// 3. 并发：几乎所有阶段都并发，只有极短暂的 STW
// 4. 不压缩指针：堆超过 32G 时 ZGC 不受影响（不像 G1）

// ZGC 的局限：
// 1. 需要大堆（<4G 不推荐，因为 ZGC 本身需要额外内存）
// 2. CPU 开销比 G1 高（读屏障）
// 3. JDK 11/15 版本不成熟，建议 JDK 17+
```

### 2.4 GC 调优决策树

```
GC 调优决策：

1. 堆大小 < 4G
   → 使用 Serial / Parallel（客户端应用）
   → 或 G1（但 G1 在 <4G 时优势不明显）

2. 堆大小 4-8G，延迟要求 <200ms
   → G1（默认，调参空间大）

3. 堆大小 > 8G，延迟要求 <10ms
   → ZGC（JDK 17+）

4. 堆大小 > 100G，延迟要求 <10ms
   → ZGC 或 Shenandoah

5. 吞吐量优先（批处理、数据分析）
   → Parallel Scavenge + Parallel Old
   → -XX:+UseParallelGC -XX:ParallelGCThreads=N
```

---

## 3. 内存泄漏排查

### 3.1 常见内存泄漏模式

```java
// 1. 静态集合类（最常见）
public class OOMDemo {
    private static final List<Object> CACHE = new ArrayList<>();

    public void add(Object obj) {
        CACHE.add(obj);  // 对象永远不会被 GC
    }
}

// 2. 未关闭的资源
try {
    Connection conn = dataSource.getConnection();
    // 忘记关闭 conn...
}

// 3. ThreadLocal 未 remove()
private static final ThreadLocal<BigData> TL = new ThreadLocal<>();

public void handle() {
    TL.set(new BigData());  // Web 应用线程池不销毁线程
    // 忘记 TL.remove()
    // → ThreadLocalMap 中的 value 永远不会被回收
}

// 4. 内部类持有外部类引用
class Outer {
    List<Inner> list = new ArrayList<>();
    class Inner { }  // Inner 持有 Outer 的引用
    void leak() {
        // 如果 list 越来越大，Outer 无法被 GC
    }
}

// 5. 缓存设计不当
// 使用 HashMap 做缓存 → 无限增长
// 应该用 WeakHashMap 或设置最大容量

// 6. 字符串 intern
// String.intern() 在 JDK 7+ 存在堆中
// 大量调用 intern() 会导致 GC 压力大
```

### 3.2 使用 MAT 分析堆转储

```java
// 步骤：
// 1. 配置 OOM 自动转储
//    -XX:+HeapDumpOnOutOfMemoryError
//    -XX:HeapDumpPath=/app/logs/dump.hprof

// 2. 主动转储（jmap）
//    jmap -dump:live,format=b,file=dump.hprof <pid>
//    live 表示只转储存活对象（会触发 Full GC）

// 3. 使用 Eclipse MAT（Memory Analyzer Tool）分析

// MAT 关键概念：
// - Shallow Size：对象本身占用的内存（不包含引用对象）
// - Retained Size：对象本身 + 被它引用的所有对象（GC root 路径上）
// - GC Root：堆外引用点（线程栈、静态字段、JNI）

// MAT 常用操作：
// 1. Leak Suspects Report → 自动分析泄漏嫌疑
// 2. Histogram → 按类统计实例数和占用内存
// 3. Dominator Tree → 按 Retained Size 排序
// 4. Path to GC Roots → 查看某个对象为什么不被回收
// 5. OQL → 对象查询语言（类似 SQL 查询对象图）
```

### 3.3 OOM 类型与原因

```java
// 1. java.lang.OutOfMemoryError: Java heap space
//    → 堆内存不足
//    原因：对象泄漏、堆太小、大对象（如大 List/Map）

// 2. java.lang.OutOfMemoryError: Metaspace
//    → 元空间不足
//    原因：动态生成类太多（CGLIB/Groovy）、类加载器泄漏

// 3. java.lang.OutOfMemoryError: Direct buffer memory
//    → 直接内存不足
//    原因：Netty DirectBuffer 未释放、NIO 操作不当

// 4. java.lang.OutOfMemoryError: GC overhead limit exceeded
//    → GC 占用 CPU > 98%，但回收 < 2% 的堆
//    原因：堆太小、内存泄漏严重、对象都不可回收但一直被引用

// 5. java.lang.OutOfMemoryError: unable to create new native thread
//    → 无法创建新线程
//    原因：线程数超过系统限制（ulimit -u）
//    或线程栈太大（-Xss 太大，总内存不够）
```

---

## 4. 性能监控工具

### 4.1 JDK 内置工具

```bash
# 1. jps：查看 Java 进程
jps -lvm
# 输出：进程ID 主类名 JVM参数

# 2. jstat：GC 统计（最常用）
# 查看 GC 情况（每 1 秒输出一次，共 5 次）
jstat -gcutil <pid> 1000 5

# 输出示例：
#  S0     S1     E      O      M     CCS    YGC     YGCT    FGC    FGCT    GCT
#  0.00   0.00  45.23  60.12  85.00  80.00  123    2.345    2    0.123   2.468
#  S0/S1：Survivor 区使用率
#  E：Eden 区使用率
#  O：Old 区使用率
#  M：Metaspace 使用率
#  YGC/FGC：Young/Full GC 次数
#  YGCT/FGCT：Young/Full GC 耗时（秒）

# 也可以看各区域的大小
jstat -gc <pid> 1000 5

# 3. jmap：内存映射（堆转储）
# 查看堆概况
jmap -heap <pid>
# 输出：堆配置、各代大小、使用率

# 查看堆中对象统计（触发 Full GC）
jmap -histo:live <pid> | head -30
# 输出：
# num     #instances         #bytes  class name
# ----------------------------------------------
#   1:         50000       80000000  [B           (byte[])
#   2:         40000       64000000  com.example.MyObject

# 堆转储
jmap -dump:live,format=b,file=dump.hprof <pid>

# 4. jstack：线程栈（排查死锁、CPU 高）
# 查看所有线程栈
jstack <pid>

# 查看死锁（自动检测）
jstack -l <pid> | grep -A 30 "deadlock"

# 5. jinfo：查看运行时 JVM 参数
jinfo <pid>
jinfo -flags <pid>    # 查看 JVM 参数
jinfo -sysprops <pid> # 查看 System.properties

# 6. jcmd：多功能诊断工具（JDK 7+）
jcmd <pid> help                  # 列出可用命令
jcmd <pid> VM.flags              # 查看 JVM 参数
jcmd <pid> VM.uptime             # 查看运行时间
jcmd <pid> GC.heap_info          # 堆信息
jcmd <pid> GC.class_histogram    # 类直方图
jcmd <pid> Thread.print          # 线程栈
```

### 4.2 Arthas（阿里开源诊断工具）

```bash
# Arthas 是线上诊断利器，不用重启就能诊断问题

# 启动
java -jar arthas-boot.jar
# 选择需要诊断的 Java 进程

# 常用命令：

# 1. dashboard：查看系统实时数据面板
#    线程、内存、GC、系统信息

# 2. thread：查看线程信息
thread                   # 列出所有线程
thread -n 3              # 前 3 个最忙的线程
thread -b                # 查看当前阻塞的线程
thread <id>              # 查看指定线程栈

# 3. jvm：查看 JVM 信息
#    包括：加载类数、线程数、GC 情况、内存使用

# 4. memory：查看内存使用
#    堆、非堆、各代使用情况

# 5. ognl：在线执行代码（非常强大）
ognl '@java.lang.System@getProperty("java.version")'
ognl '#ctx=@com.example.SpringContext@getContext(), #ctx.getBean("userService")'

# 6. sc/sm：查看类/方法信息
sc -d com.example.UserService   # 查看类信息
sm com.example.UserService      # 查看方法列表

# 7. watch：观测方法调用（最常用）
watch com.example.UserService getUser "{params,returnObj,throwExp}" -x 2
# -x 2 表示展开 2 层

# 8. trace：方法调用链路耗时
trace com.example.UserService getUser
# 输出每个方法的调用耗时，定位慢方法

# 9. tt：TimeTunnel（时空隧道，记录方法调用）
tt -t com.example.UserService getUser
tt -l                           # 列出记录
tt --play -i 1000               # 重放记录

# 10. vmtool：直接获取 JVM 对象（Arthas 3.5+）
vmtool --action getInstances --className com.example.UserService --limit 1
```

### 4.3 JFR（Java Flight Recorder）

```bash
# JFR 是 Oracle JDK 内置的低开销性能监控工具
# JDK 11+ 开源可用

# 启动时开启 JFR 记录：
-XX:StartFlightRecording=name=myrecording,filename=record.jfr,dumponexit=true,settings=profile

# 运行时开启：
jcmd <pid> JFR.start name=myrecording settings=profile
jcmd <pid> JFR.dump name=myrecording filename=record.jfr
jcmd <pid> JFR.stop name=myrecording

# JFR 可以分析的内容：
# - GC 暂停（详细到每个 phase）
# - 锁竞争
# - IO 等待
# - 方法采样（热点方法）
# - 类加载
# - 分配热点

# 使用 JDK Mission Control（JMC）打开 .jfr 文件分析
```

### 4.4 CPU 高负载排查流程

```bash
# CPU 100% 排查步骤：

# 1. 使用 top 找到 CPU 最高的 Java 进程
top -c  # 显示进程列表（按 CPU 排序）

# 或使用 Arthas
thread -n 3  # 前 3 个最忙的线程

# 2. 使用 top -H 或 ps 找到 CPU 最高的线程
top -H -p <pid>

# 3. 将线程 ID 转换为十六进制
printf "%x\n" <tid>  # 如 12345 → 3039

# 4. 使用 jstack 找到对应线程
jstack <pid> | grep -A 30 "0x3039"

# 或在 Arthas 中直接
thread <tid>

# 5. 分析代码
# - 线程状态：RUNNABLE（在执行业务代码）
# - 是否在 GC（VMThread）
# - 是否在死循环
# - 是否在频繁的锁自旋
```

---

## 5. JIT 编译优化

### 5.1 分层编译

```java
// JIT（Just-In-Time）编译是 JVM 性能的关键

// 分层编译（Tiered Compilation，JDK 8+ 默认）：
// Level 0：解释执行（Interpreter）
// Level 1：C1 编译器（简单优化，无 profiling）
// Level 2：C1 编译器（部分 profiling）
// Level 3：C1 编译器（完全 profiling）—— 默认 C1 编译级别
// Level 4：C2 编译器（激进优化，适合长期运行的热点）

// 编译阈值（-XX:CompileThreshold）：
// 方法调用 + 循环回边次数达到阈值后触发编译
// 分层编译下默认阈值：C1 编译 2000 次，C2 编译 15000 次

// 查看 JIT 编译：
-XX:+PrintCompilation  // 打印编译日志
// 输出示例：
// 123  1 % com.example.UserService::getUser @ 5 (45 bytes)
//   123：编译 ID
//   1：编译级别
//   %：OSR（栈上替换，循环编译）
//   @ 5：字节码偏移量
```

### 5.2 方法内联

```java
// 方法内联是 JIT 最重要的优化之一
// 将方法调用展开为实际代码，消除调用开销

public class InlineDemo {
    private int add(int a, int b) {
        return a + b;
    }

    public int compute() {
        // 如果没有内联：
        int r1 = add(1, 2);  // 实际调用 add 方法
        int r2 = add(3, 4);  // 实际调用 add 方法
        return r1 + r2;      // C2 编译后：return 10

        // 内联后等价于：
        // return 1 + 2 + 3 + 4;
        // → 进一步常量折叠：return 10;
    }
}

// 内联条件（C2 编译器）：
// 1. 方法体 < 325 字节（-XX:MaxInlineSize）
// 2. 热方法（调用频繁）
// 3. final/private/static 方法更容易内联
// 4. 接口方法（需要去虚拟化）

// 查看内联结果：
-XX:+PrintInlining
// 输出：
// @ 15   com.example.Service::method (12 bytes)   inline (hot)
// @ 22   com.example.Util::helper (45 bytes)   inline (intrinsic)

// 影响内联的因素：
// 1. 方法太大 → 不会被内联
// 2. 接口多态 → 只有单实现时才内联（去虚拟化）
// 3. 编译阈值 → 需要足够热
```

### 5.3 逃逸分析和栈上分配

```java
// 逃逸分析（Escape Analysis）是 C2 编译器的关键优化

public class EscapeAnalysisDemo {
    // 对象不逃逸——可以在栈上分配
    public long sum() {
        Point p = new Point(1, 2);  // p 只在方法内使用
        return p.x + p.y;
        // 优化后：Point 对象不会在堆上分配
        // 直接在栈上分配，方法结束后自动销毁
        // 进一步优化：标量替换
        // long result = 1 + 2;
    }

    // 对象逃逸——必须在堆上分配
    private Point cache;
    public void store() {
        cache = new Point(1, 2);  // 赋值给成员变量 → 逃逸
    }

    // 方法返回对象 → 逃逸
    public Point create() {
        return new Point(1, 2);  // 被外部使用 → 逃逸
    }

    // 参数传递 → 可能逃逸（取决于调用方）
    public void process(Point p) {
        // p 可能被其他线程访问
    }
}

// 逃逸分析的 3 个优化：
// 1. 栈上分配（Stack Allocation）：对象在栈上分配，不进入堆
// 2. 标量替换（Scalar Replacement）：将对象拆为基本类型
// 3. 锁消除（Lock Elimination）：如果对象不逃逸，同步可以消除

// 验证锁消除：
public String concat(String a, String b) {
    // StringBuffer.append 是 synchronized 方法
    // 但 buf 不逃逸 → JIT 会消除锁
    StringBuffer buf = new StringBuffer();
    buf.append(a);
    buf.append(b);
    return buf.toString();
}

// 开启逃逸分析（默认开启）：
-XX:+DoEscapeAnalysis
-XX:+EliminateAllocations    // 标量替换
-XX:+EliminateLocks          // 锁消除
```

### 5.4 编译日志解读

```bash
# -XX:+PrintCompilation 输出示例：
#
# 时间戳  编译ID 方法标识  方法名
#  123    1      b        java.lang.String::hashCode (55 bytes)
#         ↑              ↑
#   编译ID  方法类型：
#           b: 阻塞编译（blocking）
#           !: 有异常处理器
#           s: synchronized
#           n: native 方法
#           %: OSR（栈上替换）
#
# 关键观察点：
# 1. 如果看到太多 "made not entrant" / "made zombie"
#    → 表示频繁去优化（可能是类重新加载或方法被内联后被取消）
# 2. 如果编译数量很少
#    → 可能代码不够热，或 -XX:CompileThreshold 太高
```

---

## 6. 性能优化方法论

### 6.1 性能指标

```java
// 关键性能指标：

// 1. QPS（Queries Per Second）：每秒查询数
// 2. TPS（Transactions Per Second）：每秒事务数
// 3. RT（Response Time）：响应时间
// 4. TP99：99% 的请求在多少毫秒内完成
//    TP50（中位数）、TP90、TP99、TP999

// 性能优化的目标组合：
// - 高 QPS + 低 RT → 最佳体验
// - 高 QPS + 高 RT → 资源瓶颈
// - 低 QPS + 低 RT → 正常
// - 低 QPS + 高 RT → 严重问题

// Little's Law：QPS = 并发数 / RT
// 例如：100 并发，RT=200ms → QPS = 100 / 0.2 = 500
```

### 6.2 JMH 微基准测试

```java
// JMH（Java Microbenchmark Harness）是 JDK 官方的微基准测试框架
// 不要用 for 循环测性能，

// 添加依赖：
// <dependency>
//     <groupId>org.openjdk.jmh</groupId>
//     <artifactId>jmh-core</artifactId>
//     <version>1.36</version>
// </dependency>

import org.openjdk.jmh.annotations.*;
import java.util.concurrent.TimeUnit;

@BenchmarkMode(Mode.Throughput)        // 测试吞吐量
@OutputTimeUnit(TimeUnit.SECONDS)       // 每秒
@State(Scope.Thread)
@Warmup(iterations = 3, time = 1)       // 预热 3 轮
@Measurement(iterations = 5, time = 3)  // 正式测试 5 轮
@Fork(1)                                // 一个 JVM 进程
public class StringBenchmark {

    private String str = "hello, world!";

    @Benchmark
    public String substring() {
        return str.substring(0, 5);
    }

    @Benchmark
    public String replaceAll() {
        return str.replace("world", "java");
    }

    public static void main(String[] args) throws Exception {
        org.openjdk.jmh.Main.main(args);
    }
}

// 输出：
// Benchmark                     Mode  Cnt     Score    Error   Units
// StringBenchmark.substring     thrpt    5  5000.000 ± 10.000  ops/s
// StringBenchmark.replaceAll    thrpt    5  2000.000 ± 5.000   ops/s

// JMH 必须注意的陷阱：
// 1. 必须预热（让 JIT 完成编译）
// 2. 防止死代码消除（返回结果或用 Blackhole）
// 3. 测试代码不要包含 IO 操作
// 4. 一个 Benchmark 只测一个变量
```

### 6.3 代码级优化建议

```java
// 1. 减少对象创建
// ❌
String s = new String("hello");  // 创建了两个对象
// ✅
String s = "hello";              // 常量池

// 2. 使用原始类型
// ❌
Integer sum = 0;
for (int i = 0; i < 10000; i++) sum += i;  // 大量装箱
// ✅
int sum = 0;
for (int i = 0; i < 10000; i++) sum += i;

// 3. 合理使用集合预分配
// ❌
List<String> list = new ArrayList<>();  // 默认 10，频繁扩容
// ✅
List<String> list = new ArrayList<>(1000);

// 4. 避免在循环中调用方法
// ❌
for (int i = 0; i < list.size(); i++) { }  // size() 每次调用
// ✅
for (int i = 0, len = list.size(); i < len; i++) { }

// 5. 流操作注意
// ❌ 两次遍历
long count = list.stream().filter(x -> x > 0).count();
long sum = list.stream().filter(x -> x > 0).mapToInt(x -> x).sum();
// ✅ 一次遍历
IntSummaryStatistics stats = list.stream()
    .filter(x -> x > 0)
    .collect(Collectors.summarizingInt(x -> x));

// 6. 使用 StringBuilder 替代 +
// 循环中拼接字符串一定要用 StringBuilder
// ❌
String s = "";
for (String item : items) {
    s += item;  // 每次创建新 StringBuilder
}

// 7. 合理使用缓存
// 热点数据缓存到内存（Guava Cache / Caffeine）
// 避免重复计算
```

### 6.4 系统级优化

```java
// 1. OS 参数
// - 文件描述符限制：ulimit -n 65535
// - 进程数限制：ulimit -u 65535
// - TCP 参数：
//   net.ipv4.tcp_tw_reuse = 1      # 重用 TIME_WAIT 连接
//   net.ipv4.tcp_fin_timeout = 30   # 缩短 FIN 超时
//   net.core.somaxconn = 1024      # 连接队列大小

// 2. 线程池优化
// - 核心线程数 = CPU 核数 * 2（IO 密集）
// - 核心线程数 = CPU 核数 + 1（CPU 密集）
// - 队列大小：根据 QPS * RT 计算

// 3. 数据库优化
// - SQL 索引
// - 连接池大小（参考前一章）
// - 慢查询日志
// - 读写分离

// 4. 网络优化
// - 连接复用（HTTP Keep-Alive）
// - 压缩（Gzip）
// - 序列化（Protobuf / Kryo 替代 JSON）
```

---

## 7. 面试题

### Q1：JVM 调优的目标是什么？

```java
// 1. 降低 GC 暂停时间（STW 时间）
// 2. 提高吞吐量（GC 占用 CPU 比例 < 5%）
// 3. 避免 OOM
// 4. 提高启动速度（开发环境）

// 调优步骤：
// 1. 设定目标：延迟要求多少，吞吐量要求多少
// 2. 收集数据：GC 日志、线程栈、堆转储
// 3. 定位瓶颈：是 GC 问题还是代码问题
// 4. 调整参数：一次只改一个参数
// 5. 验证效果：A/B 对比
```

### Q2：你们的 JVM 参数怎么配置的？

```java
// 标准答案结构：
// 1. 堆大小：-Xms4g -Xmx4g
// 2. GC 选择：G1（JDK 8/11）/ ZGC（JDK 17+）
// 3. GC 日志：开启并保留
// 4. OOM 转储：HeapDumpOnOutOfMemoryError
// 5. 其他：Metaspace 限制、直接内存限制
// 6. 原因：根据业务场景选择（低延迟还是高吞吐）
```

### Q3：CPU 100% 怎么排查？

```java
// 1. top -c 找到 CPU 最高的 Java 进程
// 2. top -H -p <pid> 找到 CPU 最高的线程
// 3. printf "%x\n" <tid> 转十六进制
// 4. jstack <pid> | grep -A 30 "0xtid"
// 5. 分析线程栈中的代码位置

// 常见原因：
// - 死循环
// - 频繁 GC（GC Thread 占满 CPU）
// - 锁自旋（尤其是 ConcurrentHashMap 扩容）
// - 正则匹配回溯
// - 序列化/反序列化
```

### Q4：线上 OOM 怎么处理？

```java
// 1. 保留现场：不要重启（先 dump）
// 2. 获取堆转储：
//    jmap -dump:live,format=b,file=dump.hprof <pid>
// 3. 分析堆转储：
//    - MAT Leak Suspects Report
//    - 查看大对象（Dominator Tree）
//    - 查看 GC Roots 路径
// 4. 定位泄漏点
// 5. 临时措施：重启，增大堆
// 6. 修复：代码层面修复泄漏

// 如果没有开启 HeapDumpOnOutOfMemoryError：
// 下次发布一定要加！

// 预防措施：
// 1. 配置 -XX:+HeapDumpOnOutOfMemoryError
// 2. 配置健康检查（监控内存使用率）
// 3. 压测 + 容量规划
// 4. 代码审查（关注集合类、ThreadLocal）
```

### Q5：G1 和 ZGC 怎么选择？

```java
// G1：
// - JDK 9+ 默认
// - 堆 < 8G 时推荐
// - 暂停目标 100-200ms
// - 成熟稳定
// - 堆 > 32G 时指针压缩失效

// ZGC：
// - JDK 17+ 正式
// - 堆 > 8G 时推荐
// - 暂停 < 10ms
// - CPU 开销比 G1 高（读屏障）
// - 4T 以内堆大小

// 选择建议：
// JDK 8：G1
// JDK 11 且堆 > 8G：可以尝试 ZGC
// JDK 17+：堆 > 8G 用 ZGC，< 8G 用 G1
```

---

## 📚 参考资料

- 《深入理解 Java 虚拟机》（周志明）—— JVM 调优权威参考
- 《Java 性能权威指南》（Scott Oaks）
- [Arthas 官方文档](https://arthas.aliyun.com/)
- [JMH 官方示例](https://github.com/openjdk/jmh)
- [GCeasy（在线 GC 日志分析）](https://gceasy.io/)
