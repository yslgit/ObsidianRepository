# Java NIO 与网络编程

## 📖 本章导读

### 学习目标
通过本章学习，你将掌握：
- ✅ 五种 IO 模型（BIO、NIO、IO 多路复用、AIO、信号驱动）
- ✅ NIO 三大核心组件：Buffer、Channel、Selector
- ✅ 直接缓冲区与非直接缓冲区的区别与选择
- ✅ Reactor 与 Proactor 两种事件驱动模式
- ✅ Netty 核心架构与源码层面理解
- ✅ 零拷贝技术原理与实现
- ✅ 百万并发连接背后的技术原理

### 核心概念
**NIO（New IO / Non-blocking IO）** 是 JDK 1.4 引入的 IO 模型，提供了与 BIO 完全不同的工作方式。核心在于**多路复用（Multiplexing）**——一个线程可以监控多个 Channel 的 IO 事件，实现了"一个线程管理千万连接"的能力。**Netty** 是基于 NIO 封装的高性能网络框架，是 Java 网络编程的事实标准。

### 知识地图
```
NIO 与网络编程
├── IO 模型
│   ├── 阻塞 IO（BIO）
│   ├── 非阻塞 IO（NIO）
│   ├── IO 多路复用（select/poll/epoll）
│   ├── 异步 IO（AIO）
│   └── 信号驱动 IO（SIGIO）
├── NIO 核心组件
│   ├── Buffer（缓冲区）
│   │   ├── 属性：position/limit/capacity
│   │   ├── 类型：Heap vs Direct
│   │   ├── MappedByteBuffer（文件映射）
│   │   └── 内存对齐与伪共享
│   ├── Channel（通道）
│   │   ├── FileChannel（文件）
│   │   ├── SocketChannel（TCP）
│   │   ├── ServerSocketChannel（TCP 服务端）
│   │   ├── DatagramChannel（UDP）
│   │   └── transferTo/zero-copy
│   └── Selector（选择器）
│       ├── 事件类型：OP_ACCEPT/OP_READ/OP_WRITE/OP_CONNECT
│       ├── select/poll/epoll 实现
│       └── 空轮询 Bug
├── Reactor 模式
│   ├── 单线程 Reactor
│   ├── 多线程 Reactor
│   └── 主从 Reactor（Netty）
├── Netty
│   ├── 核心组件：Bootstrap/Channel/EventLoop/ChannelPipeline
│   ├── EventLoopGroup 线程模型
│   ├── ChannelPipeline 责任链
│   ├── ByteBuf（池化、零拷贝）
│   └── 编解码器与粘包/拆包
└── 零拷贝
    ├── 传统 IO 的 4 次拷贝
    ├── mmap（内存映射）
    ├── sendfile（DMA 拷贝）
    └── DirectBuffer 与堆外内存
```

### 常见误区
❌ **误区 1**：NIO 就是非阻塞 IO
✅ **真相**：NIO 中的 FileChannel 是阻塞的，只有网络 Channel（SocketChannel）可以配置为非阻塞模式。

❌ **误区 2**：AIO 比 NIO 性能好
✅ **真相**：在 Linux 上 AIO（JDK 7）使用内核 AIO 实现并不成熟，且 AIO 的回调处理在线程池中执行，上下文切换成本高。Netty 最终放弃了 AIO，NIO（多路复用）在 Linux 上是更优选择。

❌ **误区 3**：ByteBuffer.allocateDirect() 总是更快
✅ **真相**：直接缓冲区分配/回收成本高（系统调用 allocate），适合长生命周期、大缓冲区。小缓冲区或频繁分配的场景，堆缓冲区性能更好。

❌ **误区 4**：Netty 是应用层框架，与底层 IO 模型无关
✅ **真相**：Netty 在 Linux 上底层使用 epoll（通过 JNI 调用），比 Java NIO 的 Selector（select/poll）性能更好。Netty 4.0+ 提供 EpollEventLoopGroup 和 KQueueEventLoopGroup。

### 实战建议
1. **连接数少（<1000）用 BIO**：编程模型简单，易于维护
2. **高并发网络场景用 Netty**：不要直接使用 JDK NIO（空轮询 Bug、API 复杂）
3. **文件传输用零拷贝**：FileChannel.transferTo() 利用操作系统 sendfile
4. **大文件操作使用 MappedByteBuffer**：内存映射文件，避免用户态/内核态拷贝
5. **Netty 线程模型**：Boss Group 线程数 = 1，Worker Group 线程数 = CPU 核数 * 2
6. **堆外内存注意释放**：DirectBuffer 不会自动 GC，需要显式释放或等待 Full GC

---

## 1. IO 模型

### 1.1 五种 IO 模型对比

```
五种 IO 模型的内核视角：

应用程序              内核空间              硬件设备
┌─────────┐         ┌──────────┐         ┌────────┐
│ 用户进程 │         │ 内核     │         │ 磁盘/网络│
└────┬────┘         └────┬─────┘         └────┬───┘
     │                   │                    │
     │   1. 阻塞 IO（BIO）                    │
     │── syscall ──────→│─ 等待数据就绪 ────→│
     │←─── return ─────│←────────────────────│
     │（进程阻塞等待）   │                    │
     │                   │                    │
     │   2. 非阻塞 IO（NIO）                  │
     │── syscall ──────→│ 数据未就绪          │
     │← EAGAIN ────────│←──── return ───────│
     │── syscall ──────→│←──── data ────────│
     │←─── return ─────│←────────────────────│
     │（进程轮询）       │                    │
     │                   │                    │
     │   3. IO 多路复用（select/epoll）       │
     │── select ──────→│ 等待多个 fd         │
     │←─── return ─────│← 任一 fd 就绪 ─────│
     │── read ────────→│←──── data ────────│
     │←─── return ─────│←────────────────────│
     │                   │                    │
     │   4. 信号驱动 IO（SIGIO）              │
     │── sigaction ───→│ 注册信号处理        │
     │←─── SIGIO ──────│←──── data ────────│
     │── read ────────→│←────────────────────│
     │                   │                    │
     │   5. 异步 IO（AIO）                   │
     │── aio_read ────→│←──── data ────────│
     │←─── 回调 ───────│（内核拷贝完通知）   │
     │（内核完成所有操作）│                    │
     └─────────────────┴────────────────────┘
```

```java
// Java 对 IO 模型的支持：
//
// 阻塞 IO（BIO）：java.io 包（InputStream/OutputStream/ServerSocket）
//    → 一个线程处理一个连接
//
// 非阻塞 IO（NIO 非多路复用模式）：java.nio（SocketChannel.configureBlocking(false)）
//    → 轮询检查数据是否就绪
//
// IO 多路复用（NIO 多路复用模式）：java.nio.channels.Selector
//    → 一个线程监控多个 Channel
//
// 异步 IO（AIO）：java.nio.channels.AsynchronousChannel（JDK 7）
//    → 内核完成操作后回调
```

### 1.2 BIO（Blocking IO）

```java
// 传统 BIO 模式：
// 一个线程处理一个连接，连接未断开则线程不释放

// 单线程 BIO（只能处理一个连接）：
ServerSocket server = new ServerSocket(8080);
while (true) {
    Socket socket = server.accept();          // 阻塞：等待连接
    InputStream in = socket.getInputStream(); // 阻塞：等待数据
    // 处理请求...
}

// 多线程 BIO（一个连接一个线程）：
ExecutorService pool = Executors.newCachedThreadPool();
ServerSocket server = new ServerSocket(8080);
while (true) {
    Socket socket = server.accept();           // 主线程阻塞等待连接
    pool.submit(() -> handle(socket));         // 每个连接分配一个线程
}

// BIO 的问题：
// 1. 线程开销大：每个连接一个线程，线程切换成本高
// 2. 连接数受限：C10K 问题（30GB 内存大约只能支持 10000 个线程）
// 3. 资源浪费：大量线程在等待 IO，不执行实际工作
// 适用场景：连接数少（<1000）、长连接少
```

### 1.3 NIO 非阻塞模式

```java
// NIO 非阻塞模式（不常用，因为没有事件通知，需要轮询）
// 返回 EAGAIN 表示数据未就绪

SocketChannel channel = SocketChannel.open();
channel.configureBlocking(false);              // 设置为非阻塞模式
channel.connect(new InetSocketAddress("localhost", 8080));

while (!channel.finishConnect()) {
    // 连接未完成，可以做其他事情
    System.out.println("等待连接中...");
}

ByteBuffer buffer = ByteBuffer.allocate(1024);
while (true) {
    int bytesRead = channel.read(buffer);      // 非阻塞，立即返回
    if (bytesRead > 0) {
        // 读取到数据
        buffer.flip();
        // 处理数据...
        buffer.clear();
    } else if (bytesRead == -1) {
        break;  // 连接关闭
    }
    // bytesRead == 0：数据未就绪，继续循环
    // 轮询：CPU 空转
}
```

### 1.4 IO 多路复用（NIO 的核心）

```java
// IO 多路复用是 NIO 的真正价值所在
// 一个线程可以同时监控多个 Channel 的 IO 事件

// 三个关键角色：
// 1. Channel：网络连接（或文件）
// 2. Selector：多路复用器（事件通知机制）
// 3. SelectionKey：Channel 在 Selector 上的注册标记

// 工作流程：
Selector selector = Selector.open();           // 1. 打开 Selector

ServerSocketChannel ssc = ServerSocketChannel.open();
ssc.configureBlocking(false);                  // 必须非阻塞
ssc.bind(new InetSocketAddress(8080));
ssc.register(selector, SelectionKey.OP_ACCEPT); // 2. 注册 Channel 和感兴趣的事件

while (true) {
    int readyCount = selector.select();         // 3. 阻塞直到有事件发生
    // 注意：select() 阻塞，不会空转

    Set<SelectionKey> keys = selector.selectedKeys();
    Iterator<SelectionKey> iter = keys.iterator();

    while (iter.hasNext()) {
        SelectionKey key = iter.next();
        iter.remove();  // 不移除会导致重复处理

        if (key.isAcceptable()) {
            // 4. 处理新连接
            ServerSocketChannel server = (ServerSocketChannel) key.channel();
            SocketChannel client = server.accept();
            client.configureBlocking(false);
            client.register(selector, SelectionKey.OP_READ);
        }
        if (key.isReadable()) {
            // 5. 处理数据读取
            SocketChannel client = (SocketChannel) key.channel();
            ByteBuffer buffer = ByteBuffer.allocate(1024);
            client.read(buffer);
            // 处理业务逻辑...
        }
    }
}

// 一个 Selector 线程可以处理成千上万个连接
// 连接数不再是瓶颈，瓶颈变为 CPU 和业务处理能力
```

### 1.5 select / poll / epoll

Java NIO 的 Selector 在不同操作系统上有不同的底层实现：

```java
// Linux 上 Selector 的实现选择：
// - JDK 5/6：select（有上限 1024，性能差）
// - JDK 7：epoll（性能大幅提升，无上限）
// - Netty：原生 EpollEventLoopGroup（JNI 调用，性能更好）

// ┌───────────┬───────────┬──────────┬──────────────┐
// │           │ select    │ poll     │ epoll         │
// ├───────────┼───────────┼──────────┼──────────────┤
// │ 数据结构  │ 位数组    │ pollfd[]│ 事件表（红黑树）│
// │ 最大连接数│ 1024      │ 无上限  │ 无上限        │
// │ 遍历方式  │ 线性遍历  │ 线性遍历│ 回调通知      │
// │ 效率      │ O(n)      │ O(n)    │ O(1)（活跃连接）│
// │ 模式      │ 水平触发  │ 水平触发│ 水平/边缘触发  │
// │ 内核版本  │ -         │ -      │ Linux 2.6+   │
// └───────────┴───────────┴────────┴──────────────┘

// 水平触发（Level-Triggered）：
//   只要 fd 可读/可写，每次 select/epoll 都会通知
//   NIO 默认使用水平触发（简单安全）

// 边缘触发（Edge-Triggered）：
//   只在状态变化时通知（如不可读→可读）
//   需要一次性读完所有数据
//   Netty 的 Epoll 可以使用边缘触发（高效但复杂）

// Java NIO 的空轮询 Bug（JDK 已知问题）：
// 在特定 Linux 内核版本中，select() 可能无事件时返回
// 导致 CPU 100% 空转
// 解决：Netty 通过统计空轮询次数，超过阈值重建 Selector
// 官方在 JDK 11 中修复了部分场景
```

### 1.6 AIO（Asynchronous IO）

```java
// AIO（JDK 7）—— 真正的异步 IO
// 与 NIO 的区别：
// NIO：调用 read() 时，数据不一定就绪，需要多路复用器通知
// AIO：调用 read() 后立即返回，数据就绪后内核自动拷贝到缓冲区，然后回调

public class AIODemo {
    public static void main(String[] args) throws Exception {
        AsynchronousServerSocketChannel server =
            AsynchronousServerSocketChannel.open()
                .bind(new InetSocketAddress(8080));

        // 异步接受连接
        server.accept(null, new CompletionHandler<AsynchronousSocketChannel, Void>() {
            @Override
            public void completed(AsynchronousSocketChannel client, Void attachment) {
                // 接受下一个连接
                server.accept(null, this);

                ByteBuffer buffer = ByteBuffer.allocate(1024);

                // 异步读取（内核完成数据拷贝后回调）
                client.read(buffer, buffer, new CompletionHandler<Integer, ByteBuffer>() {
                    @Override
                    public void completed(Integer result, ByteBuffer buf) {
                        buf.flip();
                        byte[] data = new byte[buf.limit()];
                        buf.get(data);
                        System.out.println("Received: " + new String(data));
                    }

                    @Override
                    public void failed(Throwable exc, ByteBuffer buf) {
                        exc.printStackTrace();
                    }
                });
            }

            @Override
            public void failed(Throwable exc, Void attachment) {
                exc.printStackTrace();
            }
        });

        System.out.println("Server started on 8080");
        Thread.sleep(Long.MAX_VALUE);  // 保持主线程存活
    }
}

// AIO 的问题（为什么 Netty 没有采用 AIO）：
// 1. Linux 内核 AIO 实现不成熟（主要支持文件 IO，网络 IO 仍然是模拟）
// 2. Windows 的 IOCP 实现较好但跨平台不统一
// 3. 回调代码复杂（回调地狱）
// 4. 在 Linux 上 epoll + Reactor 模式性能优于 AIO
// Netty 最终移除了 AIO 支持，专注于 NIO + epoll
```

---

## 2. NIO 核心组件

### 2.1 Buffer

```java
// Buffer 是 NIO 中数据存储的容器
// 本质是一个内存块（数组或直接内存）

// Buffer 的四个核心属性：
// capacity（容量）：缓冲区最大容量，创建后不可变
// position（位置）：当前读写位置
// limit（上限）：可读/写的范围
// mark（标记）：记忆位置（mark → reset）

// 状态转换示例：
ByteBuffer buffer = ByteBuffer.allocate(10);   // 刚创建

// 写入数据（put）：
// position = 0, limit = 10, capacity = 10
buffer.put((byte) 'H');                        // position = 1
buffer.put((byte) 'e');                        // position = 2
buffer.put((byte) 'l');                        // position = 3
buffer.put((byte) 'l');                        // position = 4
buffer.put((byte) 'o');                        // position = 5

// 切换读模式（flip）：
// position → 0, limit → 旧 position (=5)
buffer.flip();

// 读取数据（get）：
byte first = buffer.get();                     // 'H', position = 1
byte second = buffer.get();                    // 'e', position = 2

// 倒带（rewind）：position → 0（重新读一遍）
buffer.rewind();

// 压缩（compact）：保留未读数据，准备继续写入
buffer.compact();

// 标记（mark）：记住当前位置
buffer.mark();
buffer.get();  // 读取一些数据
buffer.reset(); // 回到 mark 位置

// 清空（clear）：
// position → 0, limit → capacity
buffer.clear();
```

**Heap Buffer vs Direct Buffer：**

```java
// 1. Heap Buffer（堆缓冲区）
//    - 在 JVM 堆上分配
//    - 由 GC 管理，自动回收
//    - 分配快，回收快
//    - read/write 时有一次额外的拷贝（堆 → 内核）
ByteBuffer heapBuf = ByteBuffer.allocate(1024);  // 堆上分配

// 2. Direct Buffer（直接缓冲区/堆外内存）
//    - 通过 malloc() 在堆外内存分配
//    - 不参与 GC（通过 PhantomReference 清理）
//    - 分配慢，回收不确定
//    - 避免 read/write 时的堆 ←→ 内核拷贝
//    - 适合长生命周期、大缓冲区（如网络传输）
ByteBuffer directBuf = ByteBuffer.allocateDirect(1024);  // 堆外分配

// 3. 带地址的直接缓冲区
//    - Netty 的 PooledDirectByteBuf 使用内存池管理
//    - 减少了分配和回收的开销

// 性能对比：
// ┌──────────────┬───────────────┬──────────────┬──────────────┐
// │              │ Heap Buffer   │ Direct Buffer│ Pooled Direct │
// ├──────────────┼───────────────┼──────────────┼──────────────┤
// │ 分配速度     │ 快            │ 慢（系统调用）│ 极快（池化）     │
// │ 读写 IO 性能 │ 慢（多一次拷贝）│ 快            │ 快             │
// │ 回收方式     │ GC 自动       │ PhantomRef   │ 引用计数回收    │
// │ 生命周期     │ 短            │ 长            │ 复用           │
// └──────────────┴───────────────┴──────────────┴──────────────┘
```

**MappedByteBuffer（内存映射文件）：**

```java
// MappedByteBuffer 将文件全部或部分映射到内存
// 读写映射内存等于读写文件（操作系统负责同步）
// 底层使用 mmap 系统调用

public class MappedByteBufferDemo {
    public static void main(String[] args) throws Exception {
        RandomAccessFile file = new RandomAccessFile("bigfile.dat", "rw");
        FileChannel channel = file.getChannel();

        // 映射文件前 1GB 到内存
        MappedByteBuffer buffer = channel.map(
            FileChannel.MapMode.READ_WRITE, 0, 1024L * 1024 * 1024);

        // 直接通过内存操作文件
        buffer.put(0, (byte) 'A');      // 修改文件第 1 个字节
        buffer.putInt(100, 42);         // 修改文件第 101 字节（int）

        // 强制写回磁盘
        buffer.force();

        // 注意事项：
        // 1. 大文件映射容易导致 OutOfMemoryError（超过虚拟地址空间限制）
        // 2. 用完需要调用 Cleaner 方法释放（MappedByteBuffer 没有 unmap API）
        // 3. 32 位系统单个映射最大约 1.5GB（地址空间限制）
        // 4. 适合大文件读写的场景
    }
}
```

### 2.2 Channel

```java
// Channel 是 NIO 的数据传输通道
// 与 InputStream/OutputStream 的对比：
// - 双向：既可以读也可以写
// - 异步：支持非阻塞模式
// - 传输：可以直接与 Buffer 交互

// 主要 Channel 类型：
// FileChannel：文件 IO（阻塞，无法非阻塞）
// SocketChannel：TCP 客户端
// ServerSocketChannel：TCP 服务端
// DatagramChannel：UDP

// FileChannel 基本操作：
FileChannel fc = new RandomAccessFile("data.txt", "rw").getChannel();

// 写入
ByteBuffer buffer = ByteBuffer.allocate(1024);
buffer.put("Hello".getBytes());
buffer.flip();
fc.write(buffer);          // 从 buffer 写入文件

// 读取
fc.position(0);            // 重置位置
buffer.clear();
fc.read(buffer);           // 从文件读到 buffer

// 数据拷贝（不需要经过用户态）
// 不同 Channel 之间直接传输数据
fc.transferTo(0, fc.size(), socketChannel);    // 文件 → 网络（零拷贝）
socketChannel.transferFrom(fc, 0, fc.size());  // 文件 → 网络（零拷贝）
```

### 2.3 Selector 详解

```java
// Selector 是 NIO 多路复用的核心

// 事件类型（SelectionKey 中的 op 常量）：
// OP_ACCEPT（16）  → 服务端接受新连接
// OP_CONNECT（8）  → 客户端连接建立
// OP_READ（1）     → 数据可读
// OP_WRITE（4）    → 数据可写（注意：通常一直可写，需要注意）

// 经典服务端代码结构：
public class NIOServer {
    private Selector selector;
    private ServerSocketChannel serverChannel;

    public void start() throws IOException {
        selector = Selector.open();

        serverChannel = ServerSocketChannel.open();
        serverChannel.configureBlocking(false);
        serverChannel.bind(new InetSocketAddress(8080));

        // 注册 ACCEPT 事件
        serverChannel.register(selector, SelectionKey.OP_ACCEPT);

        System.out.println("Server started on 8080");

        while (true) {
            // select() 阻塞直到有事件发生
            // 返回值为发生的就绪事件数量
            selector.select();

            Set<SelectionKey> selectedKeys = selector.selectedKeys();
            Iterator<SelectionKey> iter = selectedKeys.iterator();

            while (iter.hasNext()) {
                SelectionKey key = iter.next();
                iter.remove();  // 必须移除

                if (!key.isValid()) continue;

                try {
                    if (key.isAcceptable()) {
                        handleAccept(key);
                    } else if (key.isReadable()) {
                        handleRead(key);
                    } else if (key.isWritable()) {
                        handleWrite(key);
                    }
                } catch (Exception e) {
                    key.cancel();  // 异常时取消注册
                    key.channel().close();
                }
            }
        }
    }

    private void handleAccept(SelectionKey key) throws IOException {
        ServerSocketChannel server = (ServerSocketChannel) key.channel();
        SocketChannel client = server.accept();
        client.configureBlocking(false);
        client.register(selector, SelectionKey.OP_READ);
        System.out.println("Accepted connection from " + client.getRemoteAddress());
    }

    private void handleRead(SelectionKey key) throws IOException {
        SocketChannel client = (SocketChannel) key.channel();
        ByteBuffer buffer = ByteBuffer.allocate(1024);
        int bytesRead = client.read(buffer);

        if (bytesRead == -1) {
            // 连接关闭
            key.cancel();
            client.close();
            return;
        }

        buffer.flip();
        byte[] data = new byte[buffer.limit()];
        buffer.get(data);
        String message = new String(data);
        System.out.println("Received: " + message);

        // 注册 WRITE 事件，准备发送响应
        key.interestOps(SelectionKey.OP_WRITE);
        key.attach(ByteBuffer.wrap(("Echo: " + message).getBytes()));
    }

    private void handleWrite(SelectionKey key) throws IOException {
        SocketChannel client = (SocketChannel) key.channel();
        ByteBuffer buffer = (ByteBuffer) key.attachment();

        if (buffer.hasRemaining()) {
            client.write(buffer);
        } else {
            // 写完取消 WRITE 事件
            key.interestOps(SelectionKey.OP_READ);
        }
    }
}

// ⚠️ OP_WRITE 的危险：
// Socket 的发送缓冲区通常有空闲空间，所以 OP_WRITE 通常都是就绪的
// 如果注册了 OP_WRITE 但不处理，select() 几乎每次都立即返回，导致 CPU 100%
// 正确做法：只在需要发送数据时才注册 OP_WRITE，发送完后立即取消
```

---

## 3. Reactor 模式

### 3.1 Reactor 架构

```java
// Reactor 模式是 NIO 多路复用的经典设计模式
// 核心思想：将 IO 事件分发给对应的 Handler 处理

// ┌──────────────────────────────────────────────────────┐
// │  Reactor                                            │
// │  （Selector：监听事件）                                │
// │     ↓ dispatch                                       │
// │  ┌──────────────────────────────────────────────────┐│
// │  │  Handler  │  Handler  │  Handler                 ││
// │  │  (read)   │  (write)  │  (accept)                ││
// │  └──────────────────────────────────────────────────┘│
// └──────────────────────────────────────────────────────┘
```

### 3.2 三种 Reactor 模型

```java
// 1. 单线程 Reactor（一个线程做所有事情）
// Reactor 线程 = 监听事件 + 处理 IO + 处理业务
// 问题：业务逻辑阻塞会导致所有连接卡顿
// 适用：简单、耗时短的场景

// 2. 多线程 Reactor（IO 和业务分离）
// Reactor 线程 = 监听事件 + 处理 IO（read/write）
// Worker 线程池 = 处理业务逻辑
// Java NIO Selector 常见实现方式

// 3. 主从 Reactor（Netty 使用）
// Main Reactor = 处理 ACCEPT 事件
// Sub Reactor = 处理 READ/WRITE 事件
// Worker 线程池 = 处理业务逻辑（ChannelHandler 中执行）

// ┌─────────────────────────────────────────────────────────┐
// │  Main Reactor（1 个线程）                                │
// │  ├── Selector(OP_ACCEPT)                                │
// │  │     ↓ 新连接分配                                     │
// │  └──────────────────────────────────────────────────    │
// │                    ↓                                    │
// │  ┌──────────────────────────────────────────────────┐  │
// │  │  Sub Reactor 1  │  Sub Reactor 2  │  Sub Reactor 3│  │
// │  │  Selector       │  Selector       │  Selector     │  │
// │  │  (read/write)   │  (read/write)   │  (read/write) │  │
// │  └────────┬──────────────────────────────┬───────────┘  │
// │           ↓                              ↓              │
// │  ┌──────────────────────────────────────────────────┐  │
// │  │  Worker ThreadPool（业务处理）                     │  │
// │  └──────────────────────────────────────────────────┘  │
// └─────────────────────────────────────────────────────────┘
```

---

## 4. Netty

### 4.1 Netty 核心组件

```java
// Netty 是 Java 网络编程的事实标准
// 主要组件：
// - Bootstrap/ServerBootstrap：启动器
// - EventLoopGroup：线程组（Reactor 线程池）
// - EventLoop：Reactor 线程（处理 IO 事件）
// - Channel：网络连接抽象
// - ChannelPipeline：处理链（责任链模式）
// - ChannelHandler：处理 IO 事件的回调
// - ChannelFuture：异步结果
// - ByteBuf：优化后的缓冲区
```

### 4.2 Netty 服务端示例

```java
// Netty 服务端示例（Echo Server）
public class EchoServer {
    public static void main(String[] args) {
        // Boss Group：处理 ACCEPT 事件（1 个线程）
        EventLoopGroup bossGroup = new NioEventLoopGroup(1);
        // Worker Group：处理 READ/WRITE 事件（默认 CPU 核数 * 2）
        EventLoopGroup workerGroup = new NioEventLoopGroup();

        try {
            ServerBootstrap b = new ServerBootstrap();
            b.group(bossGroup, workerGroup)
                .channel(NioServerSocketChannel.class)     // 使用 NIO
                // .channel(EpollServerSocketChannel.class) // Linux epoll（性能更好）
                .option(ChannelOption.SO_BACKLOG, 1024)     // TCP 全连接队列大小
                .childOption(ChannelOption.SO_KEEPALIVE, true)
                .childOption(ChannelOption.TCP_NODELAY, true) // 禁用 Nagle 算法
                .childHandler(new ChannelInitializer<SocketChannel>() {
                    @Override
                    protected void initChannel(SocketChannel ch) {
                        ch.pipeline()
                            .addLast("codec", new StringDecoder())  // 解码器
                            .addLast("handler", new EchoHandler());  // 业务处理
                    }
                });

            ChannelFuture f = b.bind(8080).sync();          // 绑定端口
            System.out.println("Server started on 8080");
            f.channel().closeFuture().sync();               // 等待关闭
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            bossGroup.shutdownGracefully();
            workerGroup.shutdownGracefully();
        }
    }

    // 业务处理器
    static class EchoHandler extends SimpleChannelInboundHandler<String> {
        @Override
        protected void channelRead0(ChannelHandlerContext ctx, String msg) {
            System.out.println("Received: " + msg);
            ctx.writeAndFlush("Echo: " + msg + "\n");
        }

        @Override
        public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
            cause.printStackTrace();
            ctx.close();
        }
    }
}

// Netty 客户端
public class EchoClient {
    public static void main(String[] args) {
        EventLoopGroup group = new NioEventLoopGroup();
        try {
            Bootstrap b = new Bootstrap();
            b.group(group)
                .channel(NioSocketChannel.class)
                .handler(new ChannelInitializer<SocketChannel>() {
                    @Override
                    protected void initChannel(SocketChannel ch) {
                        ch.pipeline()
                            .addLast(new StringDecoder())
                            .addLast(new StringEncoder())
                            .addLast(new ChannelInboundHandlerAdapter() {
                                @Override
                                public void channelActive(ChannelHandlerContext ctx) {
                                    ctx.writeAndFlush("Hello Netty");
                                }

                                @Override
                                public void channelRead(ChannelHandlerContext ctx, Object msg) {
                                    System.out.println("Server response: " + msg);
                                }
                            });
                    }
                });

            ChannelFuture f = b.connect("localhost", 8080).sync();
            f.channel().closeFuture().sync();
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            group.shutdownGracefully();
        }
    }
}
```

### 4.3 Netty 线程模型

```java
// NioEventLoopGroup 构造参数说明：
// 1. NioEventLoopGroup() → 默认线程数 = CPU 核数 * 2
// 2. NioEventLoopGroup(1) → 单线程
// 3. NioEventLoopGroup(0) → 使用 DEFAULT_EVENT_LOOP_THREADS（CPU 核数 * 2）

// NioEventLoop 的核心工作：
// 1. 轮询 Selector 上的 IO 事件（select）
// 2. 处理 IO 事件（processSelectedKeys）
// 3. 处理异步任务队列（runAllTasks，包括用户提交的普通任务和定时任务）

// 一个 NioEventLoop 可以处理多个 Channel，但所有操作都在同一个线程中执行
// 这意味着：在一个 NioEventLoop 中的 ChannelHandler 不需要考虑并发

// EventLoop 的任务队列：
// eventLoop.execute(() -> { ... });   // 提交到 EventLoop 的任务队列
// eventLoop.schedule(() -> { ... }, 5, TimeUnit.SECONDS); // 定时任务

// 线程模型的特点：
// 1. 一个 Channel 只绑定一个 NioEventLoop（保证线程安全）
// 2. 一个 NioEventLoop 可以管理多个 Channel
// 3. 所有 ChannelHandler 在所属的 NioEventLoop 线程中执行
// 4. 不要在 ChannelHandler 中执行耗时操作（阻塞会拖慢其他连接）
// 5. 耗时操作应提交到业务线程池
```

### 4.4 ChannelPipeline

```java
// ChannelPipeline 是责任链模式的实现
// 数据在 pipeline 中依次经过所有处理器

// ┌─────┬───────────┬───────────┬───────────┬───────────┐
// │     │ Inbound   │ Inbound   │           │ Outbound  │
// │ IO  │ Handler 1 │ Handler 2 │ ...       │ Handler 1 │
// │ →───┼────→──────┼────→──────┼────→──────┼────→──────│
// │     │ head      │           │           │ tail      │
// │ ←───┼────←──────┼────←──────┼────←──────┼────←──────│
// │     │ Outbound  │ Outbound  │           │           │
// └─────┴───────────┴───────────┴───────────┴───────────┘

// Inbound 事件处理（数据流入）：
// channelRegistered → channelActive → channelRead → channelReadComplete
// → exceptionCaught（异常时）

// Outbound 事件处理（数据流出）：
// bind → connect → write → flush → close

// Handler 的添加顺序：
pipeline.addLast("decoder", new StringDecoder());         // Inbound
pipeline.addLast("encoder", new StringEncoder());         // Outbound
pipeline.addLast("handler", new EchoHandler());            // Inbound
// 数据流：decoder(head) → handler(tail)
// 出站流：handler(后添加，先处理) → encoder → ...

// ChannelHandlerContext：
// ctx.write()：从当前位置向前（tail 方向）找 Outbound Handler
// ctx.channel().write()：从 pipeline 尾部开始找
// ctx.fireChannelRead()：向后传递事件（head 方向）
```

### 4.5 ByteBuf

```java
// Netty 的 ByteBuf 解决了 JDK ByteBuffer 的以下问题：
// 1. 读写分别用两个指针（readerIndex / writerIndex），不需要 flip()
// 2. 支持自动扩容
// 3. 支持池化（PooledByteBufAllocator）
// 4. 支持零拷贝（CompositeByteBuf、Unpooled.wrappedBuffer）

// ByteBuf 的三种类型：
// 1. Heap Buffer：堆上分配
ByteBuf heapBuf = Unpooled.buffer(1024);
if (heapBuf.hasArray()) {
    byte[] array = heapBuf.array();  // 可以直接获取数组
}

// 2. Direct Buffer：堆外分配
ByteBuf directBuf = Unpooled.directBuffer(1024);
if (!directBuf.hasArray()) {
    int len = directBuf.readableBytes();
    byte[] array = new byte[len];
    directBuf.getBytes(0, array);  // 需要拷贝
}

// 3. Composite Buffer：组合多个 ByteBuf（零拷贝）
CompositeByteBuf compBuf = Unpooled.compositeBuffer();
ByteBuf header = Unpooled.wrappedBuffer("HEAD".getBytes());
ByteBuf body = Unpooled.wrappedBuffer("BODY".getBytes());
compBuf.addComponents(true, header, body);
// 没有拷贝，只是逻辑组合

// 池化 ByteBuf（Netty 默认，可以通过 -Dio.netty.allocator.type=unpooled 关闭）：
ByteBufAllocator alloc = PooledByteBufAllocator.DEFAULT;
ByteBuf buf = alloc.buffer(1024);  // 从对象池获取
// 使用完后，引用计数减 1，归还到池中
buf.release();

// 引用计数：
ByteBuf buf = alloc.buffer();
buf.retain();   // refCnt = 2
buf.release();  // refCnt = 1
buf.release();  // refCnt = 0，归还池中

// 零拷贝操作（不需要移动数据）：
// 1. slice()：共享同一块内存的切片
ByteBuf original = Unpooled.wrappedBuffer("Hello World".getBytes());
ByteBuf slice = original.slice(0, 5);  // "Hello"，与 original 共享内存
slice.retain();  // 需要增加引用计数

// 2. duplicate()：共享整个缓冲区
ByteBuf duplicate = original.duplicate();

// 3. CompositeByteBuf：组合多个缓冲区
```

### 4.6 粘包与拆包

```java
// TCP 是流式协议，没有消息边界
// 应用层需要自己处理粘包/拆包

// 粘包：发送方发送多个小包，接收方一次收到多个（Nagle 算法合并）
// 拆包：发送方发送一个大包，接收方分多次收到（MSS 限制分段）

// Netty 内置的解码器：

// 1. FixedLengthFrameDecoder：固定长度
ch.pipeline().addLast(new FixedLengthFrameDecoder(10));

// 2. LineBasedFrameDecoder：换行符分割
ch.pipeline().addLast(new LineBasedFrameDecoder(1024));

// 3. DelimiterBasedFrameDecoder：自定义分隔符
ch.pipeline().addLast(new DelimiterBasedFrameDecoder(1024,
    Unpooled.wrappedBuffer("END".getBytes())));

// 4. LengthFieldBasedFrameDecoder：长度字段（最常用，协议解析）
// 协议格式：[length(4字节)][content(length字节)]
ch.pipeline().addLast(new LengthFieldBasedFrameDecoder(
    1024,      // maxFrameLength：最大帧长度
    0,         // lengthFieldOffset：长度字段偏移
    4,         // lengthFieldLength：长度字段长度
    0,         // lengthAdjustment：长度调整
    4          // initialBytesToStrip：跳过的字节数（去掉长度字段）
));
```

---

## 5. 零拷贝（Zero-Copy）

### 5.1 传统 IO 的 4 次拷贝

```java
// 传统 IO 方式：从磁盘读取数据，通过 Socket 发送
// File.read(file, buf, len);
// Socket.send(socket, buf, len);

// 数据在内存中的流动：
// 磁盘 → 内核缓冲区（DMA 拷贝）              ① DMA 拷贝
// 内核缓冲区 → 用户缓冲区（CPU 拷贝）          ② CPU 拷贝
// 用户缓冲区 → Socket 缓冲区（CPU 拷贝）       ③ CPU 拷贝
// Socket 缓冲区 → 网卡（DMA 拷贝）             ④ DMA 拷贝
//
// 注：DMA = Direct Memory Access（硬件直接访问内存，不需 CPU 参与）

// 4 次拷贝 + 4 次上下文切换（用户态 ←→ 内核态）
// 大文件传输时性能极差
```

### 5.2 mmap 优化

```java
// mmap（内存映射文件）将文件映射到内存
// 用户态和内核态共享同一块内存，减少一次 CPU 拷贝

// 流程：
// 磁盘 → 内核缓冲区（DMA 拷贝）              ① DMA 拷贝
// 内核缓冲区 → Socket 缓冲区（CPU 拷贝）      ② CPU 拷贝
// Socket 缓冲区 → 网卡（DMA 拷贝）            ③ DMA 拷贝

// 减少了 1 次 CPU 拷贝（不需要从内核拷贝到用户缓冲区）
// 但仍是 3 次拷贝 + 4 次上下文切换

// Java 中的 mmap：FileChannel.map() + Channel.write()
MappedByteBuffer mmap = fileChannel.map(FileChannel.MapMode.READ_ONLY, 0, size);
socketChannel.write(mmap);  // 从映射内存写入 Socket
```

### 5.3 sendfile 优化

```java
// sendfile（Linux 2.6.33+）—— 真正的零拷贝
// 数据不需要在用户态和内核态之间拷贝
// 应用程序不参与数据搬运

// 流程：
// 磁盘 → 内核缓冲区（DMA 拷贝）              ① DMA 拷贝
// 内核缓冲区 → 网卡（DMA 拷贝）              ② DMA 拷贝

// 只有 2 次 DMA 拷贝，0 次 CPU 拷贝！
// 0 次上下文切换（不需要切换到用户态）

// 如果网卡支持 SG-DMA（Scatter-Gather DMA）：
// 磁盘 → 内核缓冲区（DMA）→ 网卡直接从内核缓冲区读取（SG-DMA）
// 同样 2 次拷贝，但都发生在硬件层面

// Java 中的 sendfile：FileChannel.transferTo()
public class ZeroCopyDemo {
    public static void main(String[] args) throws Exception {
        FileChannel fileChannel = new RandomAccessFile("bigfile.dat", "r").getChannel();
        SocketChannel socketChannel = SocketChannel.open(
            new InetSocketAddress("localhost", 8080));

        // 一行代码实现零拷贝
        // 底层调用 Linux sendfile 系统调用
        long position = 0;
        long count = fileChannel.size();
        long transferred = fileChannel.transferTo(position, count, socketChannel);

        System.out.println("Transferred: " + transferred + " bytes");
    }
}

// transferTo 的限制：
// 1. 只能从 FileChannel 到 SocketChannel（文件 → 网络）
// 2. 操作系统支持：Linux、Windows（但实现不同）
// 3. 最大传输大小可能受限（取决于操作系统）
```

### 5.4 Netty 中的零拷贝

```java
// Netty 的零拷贝包含多个层面：
// 1. 堆外内存（Direct Buffer）：减少堆 ←→ 内核的拷贝
// 2. CompositeByteBuf：逻辑组合多个缓冲区，不进行物理拷贝
// 3. FileRegion：包装 FileChannel.transferTo()
// 4. Unpooled.wrappedBuffer：包装 byte[] 为 ByteBuf，不拷贝

// FileRegion 示例：
public class FileServerHandler extends ChannelInboundHandlerAdapter {
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        // 使用 FileRegion 实现零拷贝文件发送
        FileRegion region = new DefaultFileRegion(
            new FileInputStream("bigfile.dat").getChannel(),
            0,
            fileSize);
        ctx.writeAndFlush(region);
    }
}
```

---

## 6. 面试题

### Q1：BIO、NIO、AIO 的区别？

```java
// BIO（Blocking IO）：线程阻塞等待数据
// 适用连接数少、架构简单的场景

// NIO（Non-blocking IO）：IO 多路复用
// 适用连接数多、连接时间短的场景（如即时通讯）

// AIO（Asynchronous IO）：异步非阻塞，数据就绪后回调
// 适用连接数多、连接时间长的场景（如直播、文件传输）
// 但 Linux 上 AIO 不成熟，实际仍是 NIO 更优

// ┌────────┬─────────────┬───────────────┬──────────────────┐
// │        │ BIO         │ NIO           │ AIO              │
// ├────────┼─────────────┼───────────────┼──────────────────┤
// │ 阻塞   │ 阻塞        │ 非阻塞        │ 非阻塞           │
// │ 线程   │ 1 连接:1 线程│ 1 连接:1 线程 │ N 连接:1 线程    │
// │ 调用者 │ 阻塞等待    │ 轮询或通知    │ 回调             │
// │ 使用   │ 简单        │ 复杂          │ 复杂             │
// │ 场景   │ 低并发      │ 高并发        │ 极高并发         │
// └────────┴─────────────┴───────────────┴──────────────────┘
```

### Q2：select、poll、epoll 的区别？

```java
// select：1024 上限（fd_set 位数组），线性遍历 O(n)
// poll：无上限（链表），线性遍历 O(n)
// epoll：无上限（红黑树 + 链表），回调机制 O(1)

// epoll 优势：
// 1. 注册回调：注册 fd 时绑定回调函数，事件就绪时触发
// 2. mmap 共享内存：避免用户态 ↔ 内核态的数据拷贝
// 3. 只返回就绪的 fd：不需要遍历所有 fd
// 4. 支持边缘触发（ET）：减少事件通知次数

// epoll 的系统调用：
// epoll_create()：创建 epoll 实例
// epoll_ctl()：注册/修改/删除 fd 和事件
// epoll_wait()：等待事件（返回就绪的 fd）
```

### Q3：Netty 的 EventLoop 与线程的关系？

```java
// 一个 NioEventLoop 对应一个永远不会改变的 Thread
// 一个 NioEventLoop 处理多个 Channel
// 一个 Channel 只属于一个 NioEventLoop

// EventLoop 内部循环：
// while (true) {
//     select();           // 1. 检查 IO 事件
//     processSelectedKeys();  // 2. 处理 IO 事件
//     runAllTasks();      // 3. 执行任务队列中的任务
// }

// 为什么所有 Handler 在同一个线程？
// 避免并发问题，不需要加锁
// 但耗时操作会阻塞该 EventLoop 的所有 Channel

// 解决耗时操作：
// 1. 在 Handler 中提交到业务线程池
// 2. 使用 EventExecutorGroup 将 Handler 分配到指定线程池
pipeline.addLast(businessGroup, new BusinessHandler());
```

### Q4：Netty 如何解决 JDK 空轮询 Bug？

```java
// JDK NIO Selector 空轮询 Bug：
// 在特定 Linux 内核中，select() 可能无事件返回
// 导致 CPU 100% 空转

// Netty 的解决方案：
// 1. 统计空轮询次数
// 2. 如果达到阈值（默认 512 次），重建 Selector
// 3. 将旧 Selector 上注册的 Channel 转移到新 Selector
// 4. 关闭旧 Selector

// 关键代码（io.netty.channel.nio.NioEventLoop）：
private static final int SELECTOR_AUTO_REBUILD_THRESHOLD = 512;

int select() throws IOException {
    long timeout = selectDelayMillis;
    int selectCnt = 0;

    for (;;) {
        long startTime = System.nanoTime();
        int selectedKeys = selector.select(timeout);
        long time = System.nanoTime() - startTime;

        if (selectedKeys != 0 || time >= timeout) {
            // 正常返回
            return selectedKeys;
        }

        selectCnt++;
        if (selectCnt >= SELECTOR_AUTO_REBUILD_THRESHOLD) {
            // 空轮询次数过多，重建 Selector
            rebuildSelector();
            selectCnt = 0;
            break;
        }
    }
    return selector.select(timeout);
}
```

### Q5：Netty 如何实现高并发？

```java
// 1. 主从 Reactor 模型（Main Reactor + Sub Reactor）
// 2. epoll 多路复用（事件驱动，非阻塞）
// 3. 零拷贝（Direct Buffer + FileRegion + 池化）
// 4. 内存池（PooledByteBufAllocator 减少 GC）
// 5. 串行无锁设计（一个 Channel 在一个 EventLoop 中处理）
// 6. 高效的编解码（Recycling ArrayList、FastThreadLocal）
// 7. 批量 IO 处理（每次 select 后批量处理，减少系统调用）
// 8. MPSC 队列（多生产者单消费者，无锁队列）
```

---

## 📚 参考资料

- 《Java NIO》（Ron Hitchens）—— NIO 经典著作
- 《Netty 权威指南》（李林锋）—— Netty 中文权威
- 《Netty 源码剖析》
- [Netty 官方文档](https://netty.io/wiki/)
- [Linux 零拷贝原理](https://www.kernel.org/doc/html/latest/core-api/index.html)
