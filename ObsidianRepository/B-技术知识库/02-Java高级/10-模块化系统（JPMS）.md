# Java 模块化系统（JPMS）

## 📖 本章导读

### 学习目标
通过本章学习，你将掌握：
- ✅ 模块化系统解决了什么问题
- ✅ module-info.java 的核心语法
- ✅ 模块导出、开放、服务、传递依赖
- ✅ 模块化对反射的影响与解决方案
- ✅ 未命名模块和自动模块
- ✅ 模块化迁移策略
- ✅ 模块化在 JDK 本身的应用
- ✅ 模块化与 OSGi 的对比

### 核心概念
**JPMS（Java Platform Module System）** 是 Java 9 最具影响力的特性，项目代号 **Jigsaw**。模块化系统为 Java 提供了三方面的核心能力：① 强封装——模块可以控制哪些包对外可见；② 可靠配置——模块声明依赖关系，运行时校验；③ 可扩展——服务加载机制（ServiceLoader）与模块化深度集成。

### 知识地图
```
模块化系统（JPMS）
├── 为什么需要模块化
│   ├── JDK 8 的问题（rt.jar 庞大、内部 API 暴露）
│   ├── JAR 地狱（类路径依赖不明确）
│   ├── 强封装需求（访问控制）
│   └── 运行时验证（找不到类立即报错）
├── module-info.java
│   ├── module 声明
│   ├── exports（导出包）
│   ├── requires（依赖模块）
│   ├── opens（开放反射）
│   ├── provides...with（提供服务）
│   ├── uses（使用服务）
│   └── transitive（传递依赖）
├── 模块类型
│   ├── 命名模块（module-info.java）
│   ├── 未命名模块（类路径）
│   ├── 自动模块（JAR 自动转为模块）
│   └── 平台模块（JDK 模块）
├── 模块化迁移
│   ├── 从类路径到模块路径
│   ├── --add-opens / --add-exports
│   ├── jdeps 工具分析依赖
│   ├── 多版本 JAR（Multi-Release JAR）
│   └── 库的适配策略
├── JDK 模块化
│   ├── java.base（核心模块）
│   ├── java.sql、java.xml 等
│   ├── jdk.unsupported（sun.misc.Unsafe）
│   └── 模块镜像（Module Image）
├── 高级主题
│   ├── 模块层（ModuleLayer）
│   ├── 运行时动态模块
│   └── 类加载器与模块的关系
└── 面试题
```

### 常见误区
❌ **误区 1**：Java 9+ 的模块化就是隐藏一些内部 API
✅ **真相**：模块化涉及编译期、打包期、运行期的全面变化。不仅是封装，还包括可靠的依赖配置、服务加载、运行时验证等整套体系。

❌ **误区 2**：模块化是 Maven/Gradle 的替代品
✅ **真相**：模块化关注的是 Java 平台级的结构化，Maven 关注的是项目构建和依赖管理。两者互补而非替代。在模块化项目中，Maven 管理构建和外部依赖，模块管理内部封装。

❌ **误区 3**：旧项目必须全部迁移到模块化
✅ **真相**：类路径（未命名模块）在 JDK 9+ 仍然支持。迁移不是全有或全无，可以渐进式进行。旧应用可以完全不改，新模块逐步引入。

❌ **误区 4**：所有 JDK 内部 API 都被移除了
✅ **真相**：许多内部 API（如 sun.misc.Unsafe）仍然存在，但默认不可访问。可以通过 `--add-opens` 或 `jdk.unsupported` 模块访问。但长期来看应迁移到标准替代品。

### 实战建议
1. **新项目从 Java 9+ 开始**：直接用模块化设计项目结构
2. **旧项目先整理依赖**：用 `jdeps` 分析依赖，逐步引入模块
3. **反射操作使用 opens**：框架需要访问私有成员时，在 module-info 中 open
4. **自动模块命名**：在 JAR 的 MANIFEST.MF 中设置 `Automatic-Module-Name` 控制模块名
5. **跨版本兼容**：module-info.java 放在 `src/main/java` 下，JDK 8 编译时不包含
6. **Maven 多版本 JAR**：用 `<Multi-Release>true</Multi-Release>` 支持多版本

---

## 1. 为什么需要模块化

### 1.1 JDK 8 的问题

```java
// 1. rt.jar 过于庞大
// JDK 8 的 rt.jar 包含数千个类，约 60MB
// 其实很多类（如 CORBA、DOM、Swing）小型应用根本不需要

// 2. 内部 API 暴露
// sun.misc.Unsafe、com.sun.* 等内部 API 被广泛使用
// Oracle 想隐藏这些 API 但不敢

// 3. 类路径（Classpath）脆弱
// - 依赖不明确：不知道 JAR 之间是什么关系
// - 运行时才发现类缺失（ClassNotFoundException）
// - 版本冲突：同一个类出现在多个 JAR 中

// 4. 强封装需求
// Java 的访问控制（public/protected/private）不够细
// public 的类一旦发布，就不能随便修改
// 模块化让"可访问但不应使用"的代码被真正隐藏
```

### 1.2 模块化解决的问题

```java
// 模块化带来的改变：
// 1. 模块路径（Module Path）替代类路径（部分）
// 2. 编译期和运行期的依赖校验
// 3. 强封装（模块可以隐藏包）
// 4. 更小的运行时（JLink 定制运行时镜像）
// 5. 服务加载的标准化（ServiceLoader）
```

---

## 2. module-info.java 核心语法

### 2.1 基本结构

```java
// module-info.java 是模块的配置文件
// 位于模块源码根目录（src/main/java/module-info.java）

// 最简单的模块声明：
module com.example.myapp {
    // 模块内容
}

// 完整的模块声明：
module com.example.myapp {
    // 依赖
    requires java.base;              // 隐式依赖（所有模块自动依赖）
    requires java.sql;               // 依赖 JDBC 模块
    requires transitive java.logging; // 传递依赖
    requires static java.xml;        // 编译期可选依赖

    // 导出
    exports com.example.myapp.api;   // 导出包（其他模块可访问 public 类型）
    exports com.example.myapp.model;

    // 开放（反射）
    opens com.example.myapp.internal; // 开放包给反射
    opens com.example.myapp.config to
        com.example.orm;              // 仅对 orm 模块开放反射

    // 服务
    uses com.example.spi.Formatter;   // 使用服务
    provides com.example.spi.Formatter
        with com.example.myapp.JsonFormatter; // 提供服务实现

    // 其他
    // 允许的模块（不常用）
    // 权限等
}
```

### 2.2 requires（依赖）

```java
// requires 声明模块依赖

module com.example.store {
    // 必须依赖（java.base 除外，所有模块隐式依赖 java.base）
    requires java.sql;

    // 传递依赖
    // 如果 A 依赖 B，B 依赖 C 时使用 requires transitive
    // 则 A 自动获得 C 的依赖
    requires transitive java.logging;
    // 场景：模块的 public API 中使用了另一个模块的类型
    // 例如：com.example.common 中的方法返回 Logger 类型
    // 那么依赖 com.example.common 的模块必须 requires transitive java.logging

    // 静态依赖（编译期需要，运行时可选的）
    requires static java.desktop;
    // 场景：只在使用特定功能时才需要的模块
    // 如果运行时没有该模块，不会报错
}

// requires 的常见依赖：
// java.base       → 核心（String、Thread、Collection 等）
// java.sql         → JDBC
// java.xml         → XML 处理
// java.logging     → 日志
// java.desktop     → Swing/AWT
// java.management  → JMX
// java.naming      → JNDI
// java.net.http    → HttpClient（JDK 11+）
```

### 2.3 exports（导出）

```java
// exports 声明模块对外暴露的包

module com.example.library {
    // 导出 api 包——其他模块可以访问其中的 public 类型
    exports com.example.library.api;

    // 导出到特定模块（限定导出）
    exports com.example.library.internal to
        com.example.plugin;
    // 只有 com.example.plugin 模块可以访问
    // 其他模块不能引用该包

    // 未导出的包：完全隐藏
    // 如 com.example.library.impl 中的类
    // 编译期不可见，运行期也不可访问
}

// 导出 vs 不导出：
// exports com.example.api：✅ 其他模块可以 import 和访问 public 类
// （不写）：❌ 其他模块无法 import，也无法通过反射访问
```

### 2.4 opens（开放）

```java
// opens 用于运行时反射访问（框架最需要）

module com.example.entity {
    // 开放包给反射
    // 不 open——外部不能通过反射访问私有成员
    // open——外部可以通过反射访问私有成员（setAccessible(true)）
    opens com.example.entity;
    // 对 ORM 框架（Hibernate、MyBatis）尤其重要

    // 限定开放
    opens com.example.entity to
        org.hibernate.orm;
    // 只有 Hibernate 可以反射访问

    // 整个模块开放（不推荐，因为太宽）
    // open module com.example.entity {
    //     exports com.example.entity;
    // }
    // open module 意味着所有包都开放给反射
}

// opens vs exports：
// exports：编译期可访问 + 运行时 public 成员
// opens：编译期不可访问 + 运行时任何成员（反射）
// 大多数框架（Spring、Hibernate）需要 opens

// JDK 9+ 中 setAccessible(true) 会检查：
// 1. 目标类所在的包是否对调用者模块开放
// 2. 未开放则抛 InaccessibleObjectException

// 运行时 JVM 参数绕过（临时方案）：
// --add-opens com.example.entity/com.example.entity.internal=ALL-UNNAMED
```

### 2.5 provides...with / uses（服务）

```java
// 服务加载（基于 ServiceLoader 的 SPI）

// 服务接口定义：
// 在 com.example.spi 模块中
public interface PaymentProvider {
    boolean process(Payment payment);
}

// 服务提供者：
module com.example.alipay {
    requires com.example.spi;

    // 声明对 PaymentProvider 接口的实现
    provides com.example.spi.PaymentProvider
        with com.example.alipay.AlipayProvider;
    // AlipayProvider 必须实现 PaymentProvider 接口
    // 并且必须是 public 的
}

module com.example.wechatpay {
    requires com.example.spi;

    provides com.example.spi.PaymentProvider
        with com.example.wechatpay.WechatProvider;
}

// 服务消费者：
module com.example.store {
    requires com.example.spi;
    uses com.example.spi.PaymentProvider;
    // 使用 ServiceLoader 加载所有提供者
}

// 服务使用代码：
ServiceLoader<PaymentProvider> loader = ServiceLoader.load(PaymentProvider.class);
for (PaymentProvider provider : loader) {
    result = result || provider.process(payment);
}

// 模块化后，不再需要 META-INF/services 文件
// 模块声明本身就提供了服务注册信息
// 但仍然兼容传统的 META-INF/services 方式（自动模块）
```

### 2.6 transitive（传递依赖）

```java
// 传递依赖——解决"依赖传递"的问题

// 场景：模块 A 的公共 API 中使用了模块 B 的类型
// 任何模块依赖 A，如果不加上 B，编译会失败

// 方案 1（不优雅）：
// 每个依赖 A 的模块都要自己加 requires B

// 方案 2（requires transitive）：
module com.example.common {
    // 声明传递依赖
    requires transitive java.logging;

    // 如果 common 模块有一个公共方法返回 Logger：
    public Logger getLogger() { ... }
    // 依赖 common 的模块不需要显式 requires java.logging
}

// 使用：
module com.example.myapp {
    requires com.example.common;
    // 不需要显式 requires java.logging
    // 因为 common 使用了 transitive

    public void test() {
        Logger log = new CommonService().getLogger();  // 可用
    }
}

// transitive 的常见使用场景：
// - 公共库（如 Guava、Jackson）
// - API 中暴露了其他模块的类型
// - 父模块（如 spring.boot）
```

---

## 3. 模块类型

### 3.1 命名模块

```java
// 命名模块（Named Module）：
// 有 module-info.java 的模块
// 位于模块路径上

// 特征：
// 1. 有明确的模块名称
// 2. 控制导出和开放的包
// 3. 运行时依赖校验

// 命名模块不能访问未导出包
// 命名模块的包要么导出要么隐藏
```

### 3.2 未命名模块

```java
// 未命名模块（Unnamed Module）：
// 位于类路径（Classpath）上的代码

// 特征：
// 1. 没有 module-info.java
// 2. 位于类路径而非模块路径
// 3. 可以访问其他未命名模块的所有包
// 4. 可以访问命名模块的导出包（exports）
// 5. 不能访问命名模块的非导出包
// 6. 从未命名模块看不到模块信息

// 兼容性设计：
// 旧应用完全运行在类路径上
// 不需要做任何修改就能在 JDK 9+ 运行

// 如何选择模块路径 vs 类路径：
// --module-path /path/to/modules （新方式）
// --class-path /path/to/jars     （旧方式，兼容）

// 默认未命名模块有所有导出模块的访问权
// 但通过 --add-opens 也可以访问非导出包
```

### 3.3 自动模块

```java
// 自动模块（Automatic Module）：
// 位于模块路径上但没有 module-info.java 的 JAR

// 使用场景：
// 你想用模块化的方式管理依赖，但某些 JAR 还没有 module-info.java

// 特征：
// 1. 自动从 JAR 名称推导模块名
// 2. 导出所有包（相当于 exports 所有包）
// 3. 开放所有包（相当于 opens 所有包）
// 4. 依赖所有其他模块
// 5. 支持 META-INF/services 加载

// 模块名推导规则：
// JAR 文件: guava-30.1-jre.jar
// 自动模块名: guava（从文件名推导）
// 更精确：在 MANIFEST.MF 中设置 Automatic-Module-Name

// 目录结构：
// module-info.jar/META-INF/MANIFEST.MF
// Manifest-Version: 1.0
// Automatic-Module-Name: com.google.common

// 自动模块的目的：平缓迁移路径
// 从类路径 → 模块路径（自动模块）→ 添加 module-info.java（命名模块）
```

### 3.4 平台模块

```java
// 平台模块——JDK 自身的模块化

// JDK 9+ 将 JDK 拆分为约 80 个模块

// 核心模块：
java.base                // String、Collection、Thread、Class 等
java.logging             // java.util.logging
java.sql                 // JDBC
java.xml                 // XML 处理
java.desktop             // Swing/AWT
java.management          // JMX
java.security.jgss       // GSS/Kerberos
java.rmi                 // RMI
java.scripting           // 脚本引擎（Nashorn）

// 非标准模块（可能需要 --add-exports）：
jdk.unsupported          // sun.misc.Unsafe、sun.reflect
jdk.management           // 管理扩展
jdk.scripting.nashorn    // Nashorn 引擎

// 如何查看 JDK 模块：
// java --list-modules
// 输出：
// java.base@17
// java.compiler@17
// java.desktop@17
// ...

// 定制运行时（JLink）：
// jlink --module-path $JAVA_HOME/jmods:myapp \
//       --add-modules com.example.myapp \
//       --output my-runtime

// JLink 创建一个只包含所需模块的 Java 运行时
// 大小可以从 300MB 减小到 40MB（适合 Docker 镜像）
```

---

## 4. 模块化迁移

### 4.1 从类路径到模块路径

```java
// 迁移路径（渐进式）：
// 阶段 1：在 JDK 9+ 运行，类路径模式
//   → 不做任何修改，应用正常运行

// 阶段 2：用 jdeps 分析依赖
//   → 了解模块依赖关系
//   → 找出对内部 API 的依赖

// 阶段 3：把 JAR 放到模块路径（自动模块）
//   → 将依赖从类路径移到模块路径
//   → JAR 自动成为自动模块

// 阶段 4：核心模块添加 module-info.java
//   → 选择关键的模块先添加
//   → 逐步推广到所有模块

// 阶段 5：使用 JLink 定制运行时
//   → 减小镜像大小
```

### 4.2 jdeps 工具

```bash
# jdeps 是 JDK 依赖分析工具

# 1. 分析 JAR 的依赖
jdeps -summary myapp.jar
# 输出：
# com.example.myapp -> java.base
# com.example.myapp -> java.sql
# com.example.myapp -> jdk.unsupported

# 2. 找出对 JDK 内部 API 的依赖
jdeps -jdk-internals myapp.jar
# 输出：
# com.example.MyClass -> sun.misc.Unsafe
# 建议替换为：java.lang.invoke.VarHandle (JDK 9+)
# 或使用 --add-exports 访问

# 3. 生成模块化建议
jdeps --generate-module-info outdir myapp.jar
# 生成建议的 module-info.java

# 4. 分析依赖图
jdeps -verbose:class -dotoutput graph myapp.jar
# 生成 DOT 格式的依赖图

# 5. 检查类路径
jdeps -cp "lib/*" -recursive myapp.jar
```

### 4.3 --add-opens 与 --add-exports

```bash
# 临时解决方案：使用 JVM 参数开放模块

# --add-exports：编译期和运行期访问（public 类型）
# --add-opens：运行期反射访问（private 成员）

# 单个包：
--add-exports java.base/sun.security.x509=ALL-UNNAMED
--add-opens java.base/java.lang=ALL-UNNAMED

# 多个包（逗号分隔不支持，需要多个参数）：
--add-opens java.base/java.lang=ALL-UNNAMED \
--add-opens java.base/java.util=ALL-UNNAMED \
--add-opens java.base/java.lang.reflect=ALL-UNNAMED

# ALL-UNNAMED：对所有未命名模块开放
# 也可以对特定模块开放：
--add-opens java.base/java.lang=com.example.myapp

# 在 Maven/Gradle 中配置：
# <plugin>
#     <groupId>org.apache.maven.plugins</groupId>
#     <artifactId>maven-surefire-plugin</artifactId>
#     <configuration>
#         <argLine>--add-opens java.base/java.lang=ALL-UNNAMED</argLine>
#     </configuration>
# </plugin>

# 常用开放配置（Spring Boot 应用）：
--add-opens java.base/java.lang=ALL-UNNAMED
--add-opens java.base/java.util=ALL-UNNAMED
--add-opens java.base/java.lang.reflect=ALL-UNNAMED
--add-opens java.base/java.io=ALL-UNNAMED
--add-opens java.base/java.nio=ALL-UNNAMED
--add-opens java.base/java.text=ALL-UNNAMED
--add-opens java.base/java.time=ALL-UNNAMED
--add-opens java.base/sun.reflect=ALL-UNNAMED
# 注意：JDK 17+ 明确建议不要再依赖 --add-opens
# 框架应该适配模块化，用 module-info.java 声明 opens
```

### 4.4 多版本 JAR（Multi-Release JAR）

```java
// 多版本 JAR（MR JAR）允许一个 JAR 包含不同 JDK 版本的代码

// 目录结构：
// mylib.jar
// ├── META-INF/
// │   ├── MANIFEST.MF
// │   │   Main-Class: com.example.Main
// │   │   Multi-Release: true          ← 关键！
// │   └── versions/
// │       ├── 9/
// │       │   ├── module-info.java     ← JDK 9+ 用
// │       │   └── com/
// │       │       └── example/
// │       │           └── Internal.class  ← 9+ 版本
// │       └── 11/
// │           └── com/
// │               └── example/
// │                   └── Internal.class  ← 11+ 版本
// ├── com/
// │   └── example/
// │       └── Internal.class              ← JDK 8 默认版本
// └── other classes...

// 效果：
// JDK 8 运行：使用根目录的 Internal.class
// JDK 9+ 运行：使用 META-INF/versions/9/ 的 Internal.class
// JDK 11+ 运行：优先使用 META-INF/versions/11/ 的版本

// Maven 配置：
// <plugin>
//     <groupId>org.apache.maven.plugins</groupId>
//     <artifactId>maven-jar-plugin</artifactId>
//     <configuration>
//         <archive>
//             <manifestEntries>
//                 <Multi-Release>true</Multi-Release>
//             </manifestEntries>
//         </archive>
//     </configuration>
// </plugin>

// Maven 多版本编译：
// <plugin>
//     <groupId>net.forix</groupId>
//     <artifactId>forix-multi-release-jar</artifactId>
//     <version>1.0</version>
// </plugin>
// 或者手动配置多个 source 目录
```

### 4.5 库的适配策略

```java
// 1. 已经提供 module-info.java 的库（2024 年大多数主流库）：
//    Spring 6、Hibernate 6、Jackson、Guava 都不需要额外配置

// 2. 还没有 module-info.java 的库：
//    → 放在模块路径上作为自动模块
//    → JAR 名需要符合模块命名规则

// 3. 纯 JDK 8 库：
//    → 放在类路径上（未命名模块）
//    → 不需要任何修改

// Maven 模块化项目的典型配置（pom.xml）：
// <properties>
//     <java.version>17</java.version>
//     <maven.compiler.source>17</maven.compiler.source>
//     <maven.compiler.target>17</maven.compiler.target>
// </properties>

// <build>
//     <plugins>
//         <plugin>
//             <groupId>org.apache.maven.plugins</groupId>
//             <artifactId>maven-compiler-plugin</artifactId>
//             <configuration>
//                 <release>17</release>  <!-- 模块化需要 -->
//                 <source>17</source>
//                 <target>17</target>
//             </configuration>
//         </plugin>
//     </plugins>
// </build>

// module-info.java 位置：
// src/main/java/module-info.java  （Maven 默认位置）
// src/main/java/com/example/...   （其他源码）
```

---

## 5. 模块化对反射的影响

### 5.1 传统反射的失效

```java
// JDK 9+ 之前：
// setAccessible(true) 可以访问任何类的私有成员

// JDK 9+：
// 如果目标类在某模块中，且调用者模块没有访问权限
// setAccessible 会抛 InaccessibleObjectException

// 示例：
public class ReflectionTest {
    public static void main(String[] args) throws Exception {
        // JDK 8：正常
        // JDK 9+：可能抛 InaccessibleObjectException

        // 尝试访问 ArrayList 的私有字段
        Field field = ArrayList.class.getDeclaredField("elementData");
        field.setAccessible(true);  // JDK 9+：❌ InaccessibleObjectException

        // 原因：java.base 模块没有对调用者开放 java.util 包
    }
}

// 报错信息：
// java.lang.reflect.InaccessibleObjectException:
// Unable to make field transient java.lang.Object[]
//                 java.util.ArrayList.elementData accessible:
// module java.base does not "opens java.util" to unnamed module
```

### 5.2 模块化对框架的影响

```java
// 受影响最大的框架：
// 1. Spring（依赖注入需要反射）
// 2. Hibernate（ORM 需要反射访问字段）
// 3. MyBatis（结果映射需要反射）
// 4. Jackson/Gson（序列化需要反射）
// 5. 动态代理（CGLIB/ByteBuddy）

// 框架的适配方案：
// - Spring 6：要求 module-info.java 中 open 相应包给 Spring
// - Hibernate 6：同样需要 open 包
// - Jackson：需要 open 包

// 对于应用开发者：
// 在自己的 module-info.java 中为框架开放包：
module com.example.myapp {
    opens com.example.entity to
        org.hibernate.orm,
        com.fasterxml.jackson.databind;

    opens com.example.controller to
        spring.beans,
        spring.context,
        spring.web;
}

// 或者更简单地（如果不在乎安全性）：
open module com.example.myapp {
    // 完全开放所有包
}
```

---

## 6. 高级主题

### 6.1 模块层（ModuleLayer）

```java
// ModuleLayer 是模块化系统的运行时层面概念
// 用于动态创建模块（如应用服务器加载多个应用）

// 模块层结构：
// - Boot Layer：启动层（JDK 模块 + 应用模块）
// - Custom Layer：自定义层（插件模块）

// 创建自定义模块层：
public class ModuleLayerDemo {
    public static void main(String[] args) {
        ModuleLayer bootLayer = ModuleLayer.boot();

        // 配置模块查找器（从哪里加载模块）
        ModuleFinder finder = ModuleFinder.of(Path.of("plugins"));

        // 配置根模块
        Set<String> rootModules = Set.of("com.example.plugin");

        // 父层
        Set<ModuleLayer> parentLayers = Set.of(bootLayer);

        // 创建自定义模块层
        Configuration config = ModuleLayer
            .resolve(finder, parentLayers, rootModules);

        ModuleLayer pluginLayer = bootLayer.defineModulesWithOneLoader(
            config, ClassLoader.getSystemClassLoader());

        // 从自定义层加载类
        Class<?> pluginClass = pluginLayer
            .findLoader("com.example.plugin")
            .loadClass("com.example.plugin.Main");

        pluginClass.getMethod("run").invoke(
            pluginClass.getConstructor().newInstance());
    }
}

// 应用场景：
// - IDE 加载插件（如 IntelliJ IDEA 的插件系统）
// - 应用服务器隔离多个应用
// - OSGi 风格的模块化
```

### 6.2 类加载器与模块

```java
// JDK 9+ 类加载器架构的变化：

// JDK 8：
// Bootstrap ClassLoader → Extension → Application

// JDK 9+：
// Bootstrap ClassLoader → Platform ClassLoader → Application
// （Extension 改为 Platform，逻辑变化）

// 模块与类加载器的关系：
// - 大多数平台模块由 Platform ClassLoader 加载
// - 应用模块由 Application ClassLoader 加载
// - 类加载器仍然是树状结构，但模块系统增加了额外的封装层

// 关键变化：
// JDK 9+ 中，类加载器不再直接决定类的可见性
// 模块的 exports/opens 声明才是可见性的决定因素
```

### 6.3 模块化 vs OSGi

```java
// ┌──────────────┬──────────────────────┬──────────────────────┐
// │              │ JPMS（Java 模块化）   │ OSGi                  │
// ├──────────────┼──────────────────────┼──────────────────────┤
// │ 规范制定     │ Oracle（JCP）        │ OSGi Alliance        │
// │ 引入时间     │ Java 9（2017）       │ 2003                 │
// │ 粒度         │ 包级导出             │ 包级导出             │
// │ 动态加载     │ ModuleLayer（有限）   │ 完善（Bundle 可热插拔）│
// │ 版本管理     │ 不支持               │ 支持版本化依赖         │
// │ 类加载器     │ 3 个内置             │ 每个 Bundle 一个      │
// │ 学习成本     │ 中                   │ 高                   │
// │ 市场采用     │ JDK 内置，广泛使用   │ Eclipse 生态          │
// └──────────────┴──────────────────────┴──────────────────────┘

// 简单总结：
// JPMS：平台级模块化，JDK 内置，适合静态模块管理
// OSGi：应用级模块化，功能更强大（动态、版本），但更复杂

// 两者不是替代关系，OSGi 可以在 JDK 9+ 上运行
```

---

## 7. 面试题

### Q1：什么是模块化？为什么需要模块化？

```java
// 模块化（JPMS / Jigsaw）是 Java 9 引入的模块系统
// 核心特性：
// 1. module-info.java 声明模块
// 2. exports/opens 控制封装
// 3. requires 声明依赖（编译期和运行期校验）
// 4. provides/uses 支持服务加载

// 为什么需要：
// 1. JDK 自身太大（rt.jar 60MB），需要拆分为小型模块
// 2. 隐藏内部 API（sun.misc.Unsafe）
// 3. 解决 JAR 依赖不明确的问题
// 4. 支持 JLink 定制运行时（减少镜像大小）
```

### Q2：exports 和 opens 的区别？

```java
// exports：导出包给编译器 + 运行时（public 成员）
// opens：开放包给运行时反射（所有成员）

// exports com.example.api
//   → 编译期可 import com.example.api 中的 public 类
//   → 运行时可访问 public 成员

// opens com.example.internal
//   → 编译期不可访问
//   → 运行时可以通过反射访问所有成员（包括 private）

// 一般场景：
// - API 包：exports（公开 API）
// - 实体类/内部实现：opens（给 ORM/JSON 框架反射使用）
```

### Q3：JDK 9+ 模块化对现有项目有什么影响？

```java
// 1. 零影响——如果在类路径运行（完全兼容）
//    旧项目不做任何改动就能在 JDK 9+ 运行

// 2. 内部 API 依赖——需要 --add-exports
//    如果使用了 sun.misc.Unsafe 等内部 API
//    需要添加 JVM 参数或者迁移到标准 API

// 3. 反射访问——需要 --add-opens
//    如果框架（Spring、Hibernate）需要反射访问 JDK 内部类

// 4. 库作者——需要适配
//    提供 module-info.java 或配置 Automatic-Module-Name
```

### Q4：迁移到模块化有什么好处？

```java
// 1. 更好的封装性：只暴露需要暴露的 API
// 2. 明确的依赖关系：编译期就知道缺少什么
// 3. 更小的运行时：JLink 可以只打包需要的模块
// 4. 更好的安全性：减少攻击面（未导出的包不可访问）
// 5. 更快的启动：模块路径加载比类路径快
// 6. 更好的 IDE 支持：自动补全模块声明
```

### Q5：浅谈你使用的框架对模块化的支持？

```java
// Spring Boot 3.x：完全支持模块化
// - module-info.java 已提供
// - 需要应用在自己的 module-info.java 中 opens 相关包

// Hibernate 6.x：支持模块化
// - module-info.java 已提供
// - 实体类所在的包需要 open 给 Hibernate

// Jackson 2.x：支持模块化
// - module-info.java 已提供
// - 被序列化的类需要 open 给 Jackson

// MyBatis 3.x：部分支持
// - 主要依赖反射，需要 opens

// 总结：2024 年主流框架基本都支持模块化
// 应用开发者只需要在自己的 module-info.java 中正确配置 opens 即可
```

---

## 📚 参考资料

- 《Java 模块化系统》—— Jigsaw 官方指南
- 《Java 9 模块化开发》（Sander Mak 著）
- [OpenJDK Jigsaw 项目](https://openjdk.org/projects/jigsaw/)
- [Oracle 模块化官方文档](https://www.oracle.com/corporate/features/understanding-java-9-modules.html)
- [JLink 用户指南](https://docs.oracle.com/en/java/javase/17/docs/specs/man/jlink.html)
