# Java JDBC 与数据库

## 📖 本章导读

### 学习目标
通过本章学习，你将掌握：
- ✅ JDBC 核心接口与完整工作流程
- ✅ 连接池原理与主流实现（HikariCP、Druid）
- ✅ 事务隔离级别与传播行为
- ✅ MyBatis 核心原理与插件机制
- ✅ Spring JDBC 模板与事务管理
- ✅ 数据库连接与 SQL 执行优化
- ✅ ORM 框架原理与设计

### 核心概念
**JDBC（Java Database Connectivity）** 是 Java 访问关系数据库的标准 API。它定义了一套接口规范，数据库厂商（MySQL、PostgreSQL、Oracle 等）提供实现。**连接池**解决了频繁创建/销毁连接的性能问题，**ORM 框架**（MyBatis、Hibernate）在 JDBC 之上提供了更便利的持久化方案。**事务管理**是数据库操作的核心保障。

### 知识地图
```
JDBC 与数据库
├── JDBC 核心
│   ├── DriverManager / DataSource
│   ├── Connection（连接）
│   ├── Statement / PreparedStatement
│   ├── ResultSet（结果集）
│   └── 完整 CRUD 示例
├── 连接池
│   ├── 为什么要连接池
│   ├── HikariCP（Spring Boot 默认）
│   ├── Druid（阿里）
│   ├── DBCP2 / Tomcat CP
│   └── 连接池参数调优
├── 事务
│   ├── ACID 特性
│   ├── 隔离级别（4 种）
│   ├── 传播行为（7 种）
│   ├── 编程式事务
│   ├── 声明式事务（@Transactional）
│   └── 分布式事务（XA / TCC / Seata）
├── MyBatis
│   ├── 核心组件
│   ├── 一级/二级缓存
│   ├── 插件机制（Interceptor）
│   └── 与 Hibernate 对比
├── Spring JDBC
│   ├── JdbcTemplate
│   ├── NamedParameterJdbcTemplate
│   └── 事务管理实现
└── 高级主题
    ├── SQL 注入与防护
    ├── 批量操作优化
    ├── 分库分表
    └── 读写分离
```

### 常见误区
❌ **误区 1**：PreparedStatement 的主要作用是防止 SQL 注入
✅ **真相**：PreparedStatement 有两大作用：① 预编译提高性能（尤其是批量操作）；② 参数化查询防 SQL 注入。性能提升在某些场景下比安全性更重要。

❌ **误区 2**：事务隔离级别越高越好
✅ **真相**：可串行化是最高的隔离级别，但性能开销最大。需要根据业务场景选择——读未提交适合计数器，读已提交适合大部分应用，可重复读适合金融交易。

❌ **误区 3**：MyBatis 的一级缓存不会出问题
✅ **真相**：MyBatis 一级缓存是 SqlSession 级别的，在 Spring 管理的事务中，同一个事务共用一个 SqlSession，如果先查询再修改再查询，一级缓存可能返回脏数据。也可能是导致"神奇的 N+1"问题。

❌ **误区 4**：数据库连接数越大越好
✅ **真相**：连接数超过 CPU 核心数后，增加的只是上下文切换开销。经验公式：`连接数 = ((核心数 × 2) + 有效磁盘数)`，高并发场景反而要控制连接数。

### 实战建议
1. **永远使用 PreparedStatement**：杜绝 SQL 注入
2. **连接池参数**：HikariCP 的 `maximumPoolSize` 一般设为 10-20（以 4 核为例）
3. **事务要短**：事务内不要执行远程调用、文件操作等慢操作
4. **批量操作**：使用 `addBatch()` + `executeBatch()`，不要逐条执行
5. **分页查询**：用 `limit :offset, :size` 而非内存分页
6. **MyBatis 大结果集**：配置 `fetchSize` 避免 OOM
7. **读写分离**：主库写、从库读，用 `@Transactional(readOnly = true)` 路由到从库

---

## 1. JDBC 核心

### 1.1 JDBC 架构

```
应用程序
  ↕  JDBC API（java.sql、javax.sql）
JDBC DriverManager / DataSource
  ↕  JDBC Driver SPI（数据库厂商实现）
MySQL Connector / PostgreSQL JDBC / Oracle JDBC
  ↕  网络协议（TCP/IP）
Database Server
```

### 1.2 完整 CRUD 示例

```java
// JDBC 完整工作流程：

public class JdbcDemo {
    // 数据库连接信息
    private static final String URL = "jdbc:mysql://localhost:3306/mydb"
        + "?useSSL=false&serverTimezone=Asia/Shanghai&rewriteBatchedStatements=true";
    private static final String USER = "root";
    private static final String PASSWORD = "password";

    public static void main(String[] args) {
        // 1. 加载驱动（JDBC 4.0+ 可省略，SPI 自动加载）
        // Class.forName("com.mysql.cj.jdbc.Driver");

        // 2. 获取连接
        try (Connection conn = DriverManager.getConnection(URL, USER, PASSWORD)) {

            // 3. 创建 PreparedStatement（预编译）
            String sql = "INSERT INTO user (name, age, email) VALUES (?, ?, ?)";
            try (PreparedStatement ps = conn.prepareStatement(sql,
                    Statement.RETURN_GENERATED_KEYS)) {

                // 4. 设置参数
                ps.setString(1, "Alice");
                ps.setInt(2, 25);
                ps.setString(3, "alice@example.com");

                // 5. 执行
                int affectedRows = ps.executeUpdate();

                // 6. 获取自增主键
                if (affectedRows > 0) {
                    try (ResultSet rs = ps.getGeneratedKeys()) {
                        if (rs.next()) {
                            long id = rs.getLong(1);
                            System.out.println("Inserted user id: " + id);
                        }
                    }
                }
            }
        } catch (SQLException e) {
            e.printStackTrace();
        }
        // try-with-resources 自动关闭 Connection、Statement、ResultSet
    }

    // 查询操作
    public static void query() {
        String sql = "SELECT id, name, age FROM user WHERE age > ?";
        try (Connection conn = DriverManager.getConnection(URL, USER, PASSWORD);
             PreparedStatement ps = conn.prepareStatement(sql)) {

            ps.setInt(1, 18);
            try (ResultSet rs = ps.executeQuery()) {
                while (rs.next()) {
                    long id = rs.getLong("id");
                    String name = rs.getString("name");
                    int age = rs.getInt("age");
                    System.out.printf("id=%d, name=%s, age=%d%n", id, name, age);
                }
            }
        } catch (SQLException e) {
            e.printStackTrace();
        }
    }

    // 批量操作
    public static void batchInsert(List<User> users) {
        String sql = "INSERT INTO user (name, age) VALUES (?, ?)";
        try (Connection conn = DriverManager.getConnection(URL, USER, PASSWORD);
             PreparedStatement ps = conn.prepareStatement(sql)) {

            for (User user : users) {
                ps.setString(1, user.getName());
                ps.setInt(2, user.getAge());
                ps.addBatch();                     // 添加到批量
            }

            int[] results = ps.executeBatch();     // 一次性提交
            System.out.println("Inserted: " + results.length);
        } catch (SQLException e) {
            e.printStackTrace();
        }
    }

    // 事务操作
    public static void transactionalOperation() {
        Connection conn = null;
        try {
            conn = DriverManager.getConnection(URL, USER, PASSWORD);
            conn.setAutoCommit(false);  // 关闭自动提交 → 开启事务

            // 操作 1：扣款
            try (PreparedStatement ps1 = conn.prepareStatement(
                    "UPDATE account SET balance = balance - ? WHERE id = ?")) {
                ps1.setBigDecimal(1, new BigDecimal("100"));
                ps1.setLong(2, 1L);
                ps1.executeUpdate();
            }

            // 操作 2：入账
            try (PreparedStatement ps2 = conn.prepareStatement(
                    "UPDATE account SET balance = balance + ? WHERE id = ?")) {
                ps2.setBigDecimal(1, new BigDecimal("100"));
                ps2.setLong(2, 2L);
                ps2.executeUpdate();
            }

            conn.commit();  // 提交事务
        } catch (SQLException e) {
            if (conn != null) {
                try {
                    conn.rollback();  // 回滚事务
                } catch (SQLException ex) {
                    ex.printStackTrace();
                }
            }
            e.printStackTrace();
        } finally {
            if (conn != null) {
                try {
                    conn.setAutoCommit(true);  // 恢复自动提交
                    conn.close();
                } catch (SQLException e) {
                    e.printStackTrace();
                }
            }
        }
    }
}
```

### 1.3 Statement vs PreparedStatement vs CallableStatement

```java
// ┌───────────────────┬──────────────────┬──────────────────────┬────────────────────┐
// │                   │ Statement        │ PreparedStatement    │ CallableStatement  │
// ├───────────────────┼──────────────────┼──────────────────────┼────────────────────┤
// │ 预编译             │ ❌ 不预编译       │ ✅ 预编译（性能好）   │ ✅ 调用存储过程     │
// │ SQL 注入           │ ❌ 有风险         │ ✅ 参数化防注入      │ ✅ 参数安全         │
// │ 适用场景           │ 一次性 DDL        │ 重复执行的 DML       │ 存储过程调用        │
// │ 代码可读性         │ 差（字符串拼接）   │ 好（? 占位符）       │ 中                  │
// │ 批处理性能         │ 差               │ 好                  │ 一般               │
// └───────────────────┴──────────────────┴──────────────────────┴────────────────────┘

// 永远不要使用 Statement 做 DML（增删改查）：
// ❌ SQL 注入风险
String name = "'; DROP TABLE user; --";
Statement stmt = conn.createStatement();
stmt.execute("SELECT * FROM user WHERE name = '" + name + "'");  // 注入成功！

// ✅ 使用 PreparedStatement
PreparedStatement ps = conn.prepareStatement("SELECT * FROM user WHERE name = ?");
ps.setString(1, name);  // 参数化，安全
```

### 1.4 ResultSet 与元数据

```java
public class ResultSetMetaDataDemo {
    public static void printTable(Connection conn, String tableName) throws SQLException {
        String sql = "SELECT * FROM " + tableName + " LIMIT 1";
        try (Statement stmt = conn.createStatement();
             ResultSet rs = stmt.executeQuery(sql)) {

            // 获取结果集元数据
            ResultSetMetaData meta = rs.getMetaData();
            int columnCount = meta.getColumnCount();

            // 打印列信息
            for (int i = 1; i <= columnCount; i++) {
                System.out.printf("%s (%s, %s, %d)%n",
                    meta.getColumnName(i),      // 列名
                    meta.getColumnTypeName(i),  // 类型名（如 VARCHAR）
                    meta.getColumnClassName(i), // Java 类型名
                    meta.getColumnDisplaySize(i)); // 显示大小
            }

            // ResultSet 类型：
            // TYPE_FORWARD_ONLY：只能向前（默认）
            // TYPE_SCROLL_INSENSITIVE：可滚动，不感知修改
            // TYPE_SCROLL_SENSITIVE：可滚动，感知修改
            // CONCUR_READ_ONLY：只读（默认）
            // CONCUR_UPDATABLE：可更新
        }
    }

    // 批量处理大结果集
    public static void streamResult(Connection conn) throws SQLException {
        String sql = "SELECT * FROM large_table";
        PreparedStatement ps = conn.prepareStatement(sql);

        // fetchSize 告诉 JDBC 一次从数据库拉取多少行
        // 而不是一次性全部拉取到内存
        ps.setFetchSize(1000);  // 分批次拉取，避免 OOM

        try (ResultSet rs = ps.executeQuery()) {
            while (rs.next()) {
                // 每 fetchSize 行触发一次网络传输
                processRow(rs);
            }
        }
    }
}
```

---

## 2. 连接池

### 2.1 为什么要连接池

```java
// 没有连接池的问题：
// 1. 每次数据库操作都需要：创建连接（TCP 三次握手 + MySQL 认证）→ 操作 → 关闭连接
// 2. 创建连接的开销很大（约 20-100ms）
// 3. 高并发下频繁创建连接会导致数据库负载过高

// 连接池原理：
// ┌────────────────────────────────────────────┐
// │  连接池                                      │
// │  ├── 空闲连接池 [conn1] [conn2] [conn3]      │
// │  ├── 活跃连接池 [conn4(thread1)]             │
// │  └── 等待队列 [thread4] [thread5]            │
// └────────────────────────────────────────────┘
// 1. 预先创建一批连接
// 2. 使用时从池中获取
// 3. 使用后归还到池中（不关闭）
// 4. 连接不够时，创建新连接或等待
```

### 2.2 HikariCP（Spring Boot 默认连接池）

```java
// HikariCP 是目前最快的连接池
// Spring Boot 2.0+ 默认使用 HikariCP

// Spring Boot 自动配置（application.yml）：
// spring:
//   datasource:
//     url: jdbc:mysql://localhost:3306/mydb
//     username: root
//     password: password
//     hikari:
//       maximum-pool-size: 20        # 最大连接数（默认 10）
//       minimum-idle: 10              # 最小空闲连接（默认 10）
//       idle-timeout: 300000          # 空闲超时（5 分钟）
//       connection-timeout: 30000     # 获取连接超时（30 秒）
//       max-lifetime: 1800000         # 连接最大存活时间（30 分钟）
//       pool-name: MyHikariPool       # 连接池名称
//       connection-test-query: SELECT 1  # 测试查询

// 代码创建 HikariCP：
HikariConfig config = new HikariConfig();
config.setJdbcUrl("jdbc:mysql://localhost:3306/mydb");
config.setUsername("root");
config.setPassword("password");
config.setMaximumPoolSize(20);
config.setMinimumIdle(5);
config.setConnectionTimeout(30000);
config.setIdleTimeout(600000);
config.setMaxLifetime(1800000);

// 额外优化：
config.addDataSourceProperty("cachePrepStmts", "true");          // 缓存 PreparedStatement
config.addDataSourceProperty("prepStmtCacheSize", "250");        // 缓存数量
config.addDataSourceProperty("prepStmtCacheSqlLimit", "2048");   // 缓存 SQL 最大长度
config.addDataSourceProperty("useServerPrepStmts", "true");      // 服务端预编译
config.addDataSourceProperty("useLocalSessionState", "true");    // 使用本地会话状态
config.addDataSourceProperty("rewriteBatchedStatements", "true"); // 批量语句重写

HikariDataSource dataSource = new HikariDataSource(config);

// HikariCP 为什么快？
// 1. 代码优化：字节码精简，减少无意义的方法调用
// 2. 锁优化：使用 FastList 替代 ArrayList，减少范围检查
// 3. 无死锁设计：使用 ConcurrentBag 替代 BlockingQueue
// 4. 代理优化：使用 Javassist 生成代理（比 CGLIB 快）
```

### 2.3 Druid（阿里）

```java
// Druid 是阿里开源的连接池，功能最丰富
// 除了连接池，还提供：监控、SQL 统计、防火墙、日志

// Spring Boot 配置：
// spring:
//   datasource:
//     druid:
//       initial-size: 5           # 初始连接数
//       min-idle: 5               # 最小空闲
//       max-active: 20             # 最大活跃
//       max-wait: 30000            # 获取连接超时
//       time-between-eviction-runs-millis: 60000  # 检查间隔
//       min-evictable-idle-time-millis: 300000    # 最小空闲时间
//       validation-query: SELECT 1
//       test-while-idle: true      # 空闲时检测（推荐）
//       test-on-borrow: false      # 获取时检测（不推荐，影响性能）
//       test-on-return: false      # 归还时检测

// Druid 监控：
// @Bean
// public ServletRegistrationBean<StatViewServlet> statViewServlet() {
//     return new ServletRegistrationBean<>(new StatViewServlet(), "/druid/*");
// }
// 访问 http://localhost:8080/druid 查看监控面板

// Druid 的 SQL 防火墙：
// @Bean
// public FilterRegistrationBean<WallFilter> wallFilter() {
//     WallFilter wall = new WallFilter();
//     wall.setConfig(WallConfigFactory.load("wall-config.xml"));
//     // ...
// }
```

### 2.4 连接池对比

```java
// ┌──────────────┬──────────┬──────────┬──────────┬──────────────┐
// │              │ HikariCP │ Druid    │ DBCP2    │ Tomcat CP    │
// ├──────────────┼──────────┼──────────┼──────────┼──────────────┤
// │ 性能         │ ⭐⭐⭐⭐⭐ │ ⭐⭐⭐⭐  │ ⭐⭐⭐   │ ⭐⭐⭐⭐     │
// │ 监控         │ 无       │ ⭐⭐⭐⭐⭐ │ 无       │ 简单          │
// │ SQL 防火墙   │ 无       │ ⭐⭐⭐⭐⭐ │ 无       │ 无           │
// │ 配置复杂度   │ 简单     │ 中      │ 简单     │ 简单          │
// │ Spring Boot  │ 默认     │ 需引入   │ 需引入   │ Tomcat 环境   │
// │ 文件大小     │ 小       │ 大      │ 中       │ 小           │
// └──────────────┴──────────┴──────────┴──────────┴──────────────┘

// 选型建议：
// - 一般场景：HikariCP（性能极致，简单）
// - 需要监控/防火墙：Druid
// - Tomcat 容器：Tomcat CP（集成好）
// - 遗留项目：DBCP2
```

---

## 3. 事务管理

### 3.1 ACID 特性

```java
// 数据库事务的四大特性（ACID）：

// 原子性（Atomicity）：事务中的所有操作要么全部成功，要么全部回滚
//   实现：UNDO LOG（记录修改前的值，回滚时恢复）

// 一致性（Consistency）：事务执行前后数据保持合法状态
//   实现：由开发者的业务逻辑保障 + 数据库约束

// 隔离性（Isolation）：并发事务之间互不干扰
//   实现：锁 + MVCC（多版本并发控制）

// 持久性（Durability）：事务提交后数据永久保存
//   实现：REDO LOG（崩溃恢复时重放）
```

### 3.2 隔离级别

```java
// SQL 标准定义的 4 种隔离级别：

// ┌────────────────┬──────────┬──────────┬──────────┐
// │ 隔离级别        │ 脏读     │ 不可重复读 │ 幻读     │
// ├────────────────┼──────────┼──────────┼──────────┤
// │ READ UNCOMMITTED│ ✅ 可能  │ ✅ 可能  │ ✅ 可能  │
// │ READ COMMITTED  │ ❌ 避免  │ ✅ 可能  │ ✅ 可能  │
// │ REPEATABLE READ │ ❌ 避免  │ ❌ 避免  │ ✅ 可能  │
// │ SERIALIZABLE    │ ❌ 避免  │ ❌ 避免  │ ❌ 避免  │
// └────────────────┴──────────┴──────────┴──────────┘

// 脏读（Dirty Read）：读到另一个事务未提交的数据
// 不可重复读（Non-repeatable Read）：同一事务中两次读取同一行数据结果不同
// 幻读（Phantom Read）：同一事务中两次查询返回不同的行集

// MySQL 默认：REPEATABLE READ（通过 MVCC 解决了幻读）
// PostgreSQL/Oracle 默认：READ COMMITTED

// 设置隔离级别：
Connection conn = dataSource.getConnection();
conn.setTransactionIsolation(Connection.TRANSACTION_READ_COMMITTED);

// Spring 中设置：
@Transactional(isolation = Isolation.READ_COMMITTED)
public void doSomething() { }
```

### 3.3 事务传播行为（Spring）

```java
// Spring 定义了 7 种事务传播行为：

// @Transactional(propagation = Propagation.REQUIRED)  // 默认
// 有事务则加入，没有则新建
// 最常用，99% 的场景用这个

// @Transactional(propagation = Propagation.REQUIRES_NEW)
// 必须新建事务，挂起当前事务
// 适用：日志记录、审计（不受主事务回滚影响）

// @Transactional(propagation = Propagation.NESTED)
// 嵌套事务（保存点），内层回滚不影响外层
// 适用：复杂业务中的部分可回滚操作

// @Transactional(propagation = Propagation.SUPPORTS)
// 有事务则加入，没有则以非事务方式运行

// @Transactional(propagation = Propagation.NOT_SUPPORTED)
// 必须以非事务方式运行，挂起当前事务

// @Transactional(propagation = Propagation.MANDATORY)
// 必须在已有事务中运行，否则抛异常

// @Transactional(propagation = Propagation.NEVER)
// 必须非事务运行，如果存在事务则抛异常

// 示例：
@Service
public class OrderService {
    @Autowired
    private AuditService auditService;

    @Transactional
    public void createOrder(Order order) {
        // 主事务
        saveOrder(order);
        try {
            auditService.logAudit(order);  // REQUIRES_NEW，独立事务
        } catch (Exception e) {
            // 审计失败不影响主事务
            log.error("Audit failed", e);
        }
    }
}

@Service
public class AuditService {
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void logAudit(Order order) {
        // 即使外层事务回滚，审计日志也被提交
    }
}
```

### 3.4 Spring 声明式事务

```java
// @Transactional 工作原理：
// 1. Spring 通过 AOP 为目标 Bean 创建代理
// 2. 代理在方法调用前获取/创建事务
// 3. 方法执行完成后提交/回滚事务

// 关键细节：

// 1. 自调用失效
// ❌ 同一个类中方法调用，@Transactional 不生效
public class UserService {
    @Transactional
    public void outerMethod() {
        innerMethod();  // 自调用，事务不生效！
    }

    @Transactional
    public void innerMethod() { }
}

// 2. rollbackFor
// @Transactional 默认只在 RuntimeException 和 Error 时回滚
// 受检异常（Exception 子类但不是 RuntimeException）不回滚！
@Transactional(rollbackFor = Exception.class)  // 所有异常都回滚
public void doSomething() throws Exception { }

// 3. readOnly
@Transactional(readOnly = true)  // 只读事务优化
public List<User> findAll() {
    // 一些数据库会优化只读事务（如 MySQL 不会加锁）
    // 注意：如果在此方法中修改数据会抛异常
}

// 4. timeout
@Transactional(timeout = 5)  // 超时 5 秒，超时自动回滚
public void slowOperation() { }
```

### 3.5 分布式事务

```java
// 分布式事务：跨多个数据库/服务的事务

// 方案 1：XA（两阶段提交，2PC）
// - 准备阶段（Prepare）：所有资源准备好
// - 提交阶段（Commit）：所有资源提交
// - 问题：性能差，协调者单点故障，阻塞

// 方案 2：TCC（Try-Confirm-Cancel）
// - Try：预留资源
// - Confirm：确认执行
// - Cancel：取消（回滚）
// - 问题：需要业务代码支持 Try/Confirm/Cancel

// 方案 3：SAGA
// - 每个操作都有对应的补偿操作
// - 失败后执行反向补偿
// - 适用于长事务

// 方案 4：Seata（阿里开源，推荐）
// AT 模式（Automatic Transaction）：
// 自动生成 UNDO SQL，对业务代码无侵入
// 性能优于 XA，一致性弱于 XA（最终一致性）

// Spring 中的分布式事务：
// @GlobalTransactional  // Seata 注解
// public void createOrder() {
//     // 跨服务、跨库操作
// }
```

---

## 4. MyBatis

### 4.1 核心组件

```java
// MyBatis 核心组件：
// 1. SqlSessionFactoryBuilder → 构建工厂
// 2. SqlSessionFactory → 创建 SqlSession
// 3. SqlSession → 会话（数据库操作入口）
// 4. Mapper → 映射器（接口 + XML）

// MyBatis 工作流程：
// 1. 解析配置文件（mybatis-config.xml + Mapper XML）
// 2. 构建 Configuration 对象
// 3. 打开 SqlSession（获取连接）
// 4. 获取 Mapper 代理（JDK 动态代理）
// 5. 执行 SQL（通过 SqlSession 调用 Executor）
// 6. 结果映射（ResultSet → 对象）

// Mapper 的工作原理：
// MyBatis 为每个 Mapper 接口生成 JDK 动态代理
// MapperProxy.invoke() → MapperMethod.execute() → SqlSession → Executor

// MyBatis 配置文件解析过程：
// XMLConfigBuilder → Configuration
//   ├── settings（全局设置）
//   ├── typeAliases（类型别名）
//   ├── environments（环境：开发/测试/生产）
//   │   └── transactionManager（事务管理器）
//   │   └── dataSource（数据源）
//   ├── mappers（映射器）
//   │   └── XMLMapperBuilder → MappedStatement
//   │       ├── SQL 语句（select/insert/update/delete）
//   │       ├── 参数映射（parameterMap）
//   │       └── 结果映射（resultMap）
//   └── plugins（插件）
```

### 4.2 一级缓存和二级缓存

```java
// 一级缓存（SqlSession 级别，默认开启）：
// - 同一个 SqlSession 中两次相同的查询，第二次直接从缓存取
// - 执行 INSERT/UPDATE/DELETE 会清空缓存
// - 跨 SqlSession 不共享

// 问题场景：
SqlSession session = sqlSessionFactory.openSession();
UserMapper mapper = session.getMapper(UserMapper.class);

User u1 = mapper.selectById(1);  // 查询数据库
User u2 = mapper.selectById(1);  // 从缓存取（命中）

// 另一个 SqlSession 修改了数据
SqlSession session2 = sqlSessionFactory.openSession();
UserMapper mapper2 = session2.getMapper(UserMapper.class);
mapper2.updateName(1, "NewName");  // 更新
session2.commit();

User u3 = mapper.selectById(1);   // ❌ 仍然从缓存取（脏数据！）
// 注意：MyBatis 默认不会跨 SqlSession 广播缓存失效

// 一级缓存失效的场景：
// 1. 使用不同的 SqlSession
// 2. 手动清空缓存（session.clearCache()）
// 3. 两次查询之间执行了 DML 操作
// 4. 查询参数不同
// 5. 查询范围不同（RowBounds 分页）

// 二级缓存（namespace 级别，跨 SqlSession）：
// - 需要在 Mapper XML 中配置 <cache/>
// - 所有 SqlSession 共享
// - 更新操作会清空对应 namespace 的缓存
// - 注意：多表关联查询容易造成脏数据

// <mapper namespace="com.example.UserMapper">
//     <cache eviction="LRU" flushInterval="60000" size="512" readOnly="true"/>
// </mapper>

// MyBatis 缓存总结：
// 1. 一级缓存：慎用！（Spring 集成时，事务内共享 SqlSession，可能导致脏数据）
// 2. 二级缓存：不推荐！（关联查询脏数据、分布式场景问题多）
// 3. 更好的方案：Redis 外部缓存
```

### 4.3 插件机制

```java
// MyBatis 插件基于责任链模式 + 动态代理
// 可以拦截 4 种核心组件：

// @Intercepts({
//     @Signature(type = Executor.class, method = "update",
//               args = {MappedStatement.class, Object.class}),
//     @Signature(type = Executor.class, method = "query",
//               args = {MappedStatement.class, Object.class, RowBounds.class, ResultHandler.class})
// })
// public class MyPlugin implements Interceptor {
//     @Override
//     public Object intercept(Invocation invocation) throws Throwable {
//         // 前置处理
//         System.out.println("Before: " + invocation.getMethod().getName());
//         Object result = invocation.proceed();  // 执行目标方法
//         // 后置处理
//         System.out.println("After: " + result);
//         return result;
//     }
//
//     @Override
//     public Object plugin(Object target) {
//         return Plugin.wrap(target, this);  // 生成代理
//     }
//
//     @Override
//     public void setProperties(Properties properties) {
//         // 读取配置参数
//     }
// }

// 可拦截的 4 种对象：
// 1. Executor：执行器（增删改查、缓存维护）
// 2. ParameterHandler：参数处理（设置参数值）
// 3. ResultSetHandler：结果集处理（结果映射）
// 4. StatementHandler：语句处理（SQL 编译、执行）

// 实际应用：
// - 分页插件（PageHelper）→ 拦截 Executor，重写 SQL 添加 limit
// - 数据脱敏 → 拦截 ResultSetHandler
// - 读写分离 → 拦截 Executor，根据 SQL 类型路由
// - 慢 SQL 监控 → 拦截 Executor，统计执行时间
```

### 4.4 MyBatis vs Hibernate

```java
// ┌──────────────┬──────────────────────┬──────────────────────┐
// │              │ MyBatis              │ Hibernate            │
// ├──────────────┼──────────────────────┼──────────────────────┤
// │ 性质         │ 半 ORM（SQL 映射）    │ 全 ORM（对象关系映射） │
// │ 开发效率     │ 中（需写 SQL）        │ 高（自动生成 SQL）    │
// │ 性能         │ 高（SQL 精准可控）    │ 中（N+1 问题，需优化） │
// │ 灵活性       │ 极高（任何 SQL）      │ 低（复杂查询困难）    │
// │ 学习曲线     │ 低（会 SQL 即可）      │ 高（HQL、级联、缓存） │
// │ 移植性       │ 差（SQL 可能不兼容）  │ 好（自动适配方言）    │
// │ 缓存         │ 简单（一级+二级）     │ 完善（三级缓存）      │
// │ 动态 SQL     │ 强大（OGNL 表达式）   │ 一般（Criteria）     │
// └──────────────┴──────────────────────┴──────────────────────┘

// 选型建议：
// - 团队 SQL 能力强的 → MyBatis
// - 复杂报表、多表关联 → MyBatis（SQL 可控）
// - 简单的 CRUD → Hibernate（开发效率高）
// - 国内互联网公司 → 大多数用 MyBatis

// MyBatis 大量用于国内的原因：
// 1. 复杂查询场景多（互联网业务 SQL 复杂）
// 2. 性能要求高（精准控制 SQL）
// 3. 团队成员对 SQL 熟悉
```

---

## 5. Spring JDBC

### 5.1 JdbcTemplate

```java
// JdbcTemplate 是 Spring 对 JDBC 的轻量封装
// 消除样板代码（连接管理、异常处理、资源释放）

@Repository
public class UserDao {
    @Autowired
    private JdbcTemplate jdbcTemplate;

    // 查询单个对象
    public User findById(Long id) {
        return jdbcTemplate.queryForObject(
            "SELECT * FROM user WHERE id = ?",
            new BeanPropertyRowMapper<>(User.class),
            id);
    }

    // 查询列表
    public List<User> findByName(String name) {
        return jdbcTemplate.query(
            "SELECT * FROM user WHERE name LIKE ?",
            new BeanPropertyRowMapper<>(User.class),
            "%" + name + "%");
    }

    // 查询单值
    public int count() {
        return jdbcTemplate.queryForObject(
            "SELECT COUNT(*) FROM user", Integer.class);
    }

    // 更新
    public int update(User user) {
        return jdbcTemplate.update(
            "UPDATE user SET name = ?, age = ? WHERE id = ?",
            user.getName(), user.getAge(), user.getId());
    }

    // 批量更新
    public int[] batchInsert(List<User> users) {
        return jdbcTemplate.batchUpdate(
            "INSERT INTO user (name, age) VALUES (?, ?)",
            new BatchPreparedStatementSetter() {
                @Override
                public void setValues(PreparedStatement ps, int i) throws SQLException {
                    ps.setString(1, users.get(i).getName());
                    ps.setInt(2, users.get(i).getAge());
                }

                @Override
                public int getBatchSize() {
                    return users.size();
                }
            });
    }

    // NamedParameterJdbcTemplate（支持命名参数）
    @Autowired
    private NamedParameterJdbcTemplate namedJdbc;

    public User findByEmail(String email) {
        return namedJdbc.queryForObject(
            "SELECT * FROM user WHERE email = :email",
            new MapSqlParameterSource("email", email),
            new BeanPropertyRowMapper<>(User.class));
    }
}

// BeanPropertyRowMapper 自动将 ResultSet 映射到对象
// 要求：列名与字段名要匹配（支持下划线 → 驼峰转换）
```

---

## 6. 高级主题

### 6.1 SQL 注入与防护

```java
// SQL 注入原理：恶意用户通过输入构造恶意 SQL

// ❌ 错误示范（字符串拼接）：
String name = "'; DROP TABLE user; --";
String sql = "SELECT * FROM user WHERE name = '" + name + "'";
// 实际执行的 SQL：
// SELECT * FROM user WHERE name = ''; DROP TABLE user; --'

// ✅ 防护方式 1：PreparedStatement 参数化（最有效）
// 参数值被作为数据处理，不是 SQL 的一部分
PreparedStatement ps = conn.prepareStatement(
    "SELECT * FROM user WHERE name = ?");
ps.setString(1, userInput);

// ✅ 防护方式 2：使用 MyBatis #{}（而不是 ${}）
// #{} 生成 PreparedStatement 占位符
// ${} 直接拼接字符串（有风险，仅用于表名/列名等动态标识符）

// ✅ 防护方式 3：输入验证
// 白名单检查
```

### 6.2 批量操作优化

```java
// 批量插入 10000 条数据的对比：

// 方式 1：逐条插入（最慢）—— 10000 次网络往返
for (User user : users) {
    jdbcTemplate.update("INSERT INTO user (name) VALUES (?)", user.getName());
}

// 方式 2：JDBC 批量（快 10-100 倍）
String sql = "INSERT INTO user (name) VALUES (?)";
PreparedStatement ps = conn.prepareStatement(sql);
for (User user : users) {
    ps.setString(1, user.getName());
    ps.addBatch();
    if (batchCount % 1000 == 0) {
        ps.executeBatch();  // 每 1000 条提交一次
    }
}
ps.executeBatch();

// 方式 3：rewriteBatchedStatements（MySQL 优化）
// JDBC URL 添加：?rewriteBatchedStatements=true
// MySQL 会将多条 INSERT 重写为一条多值 INSERT
// 优化前：INSERT INTO user VALUES (?); INSERT INTO user VALUES (?);
// 优化后：INSERT INTO user VALUES (?), (?), (?), ...
// 性能提升：约 50-100 倍！

// 方式 4：MyBatis 批量
// <insert id="batchInsert">
//     INSERT INTO user (name) VALUES
//     <foreach collection="list" item="user" separator=",">
//         (#{user.name})
//     </foreach>
// </insert>
// 注意：MySQL 对 SQL 长度有限制（max_allowed_packet），需要控制每批数量
```

### 6.3 分库分表

```java
// 当单表数据量超过千万级时，需要考虑分库分表

// 分库分表方案：

// 1. 垂直分库：按业务模块拆分到不同数据库
//    用户库、订单库、商品库...

// 2. 水平分表：同一张表按某个字段拆分
//    user_0, user_1, user_2...user_15

// 分片规则：
// - 取模：user_id % 16 → 均匀分布，但扩容困难
// - 范围：user_id 1-1000 → table_0, 1001-2000 → table_1
// - 一致性哈希：扩容时影响小

// 主流中间件：
// Apache ShardingSphere（推荐）
// MyCat
// TDDL（阿里）

// ShardingSphere 配置示例：
// spring:
//   shardingsphere:
//     datasource:
//       names: ds0, ds1
//       ds0:
//         url: jdbc:mysql://localhost:3306/db0
//       ds1:
//         url: jdbc:mysql://localhost:3306/db1
//     sharding:
//       tables:
//         user:
//           actual-data-nodes: ds$->{0..1}.user_$->{0..15}
//           database-strategy:
//             inline:
//               sharding-column: user_id
//               algorithm-expression: ds$->{user_id % 2}
//           table-strategy:
//             inline:
//               sharding-column: user_id
//               algorithm-expression: user_$->{user_id % 16}
```

### 6.4 读写分离

```java
// 主库（Master）：处理写操作（INSERT/UPDATE/DELETE）
// 从库（Slave）：处理读操作（SELECT）

// 读写分离的实现：

// 1. Spring 配置多数据源
// 2. 通过 AOP 拦截方法，根据 readOnly 注解路由

// @Transactional(readOnly = true) → 路由到从库
// @Transactional → 路由到主库

// Spring 中的实现方式：
// 1. AbstractRoutingDataSource（Spring 内置）
// 2. 通过 @Transactional(readOnly) 判断

// ShardingSphere 自动支持读写分离：
// spring:
//   shardingsphere:
//     masterslave:
//       name: ms
//       master-data-source-name: master
//       slave-data-source-names: slave0, slave1
//       load-balance-algorithm-type: round_robin  # 从库负载均衡
```

---

## 7. 面试题

### Q1：PreparedStatement 相比 Statement 的优势？

```java
// 1. 预编译：SQL 被预编译并缓存，重复执行效率高
// 2. 防注入：参数化查询，用户输入不会被当作 SQL 执行
// 3. 可读性：? 占位符比字符串拼接清晰
// 4. 二进制安全：可以传输二进制数据（如图片）
// 5. 批量操作：addBatch() + executeBatch() 批量提交
```

### Q2：数据库连接池的参数如何设置？

```java
// maximumPoolSize（最大连接数）：
// 公式 = ((core_count * 2) + effective_spindle_count)
// 核心数 4，有效磁盘数 1 → (4*2) + 1 = 9，取整 10

// 误区：连接数不是越大越好
// 连接过多会导致：上下文切换、锁竞争、数据库负载高

// HikariCP 经验值（4 核 8G）：
// maximumPoolSize: 10-20
// minimumIdle: 5-10
// connectionTimeout: 30000（30 秒）
// idleTimeout: 600000（10 分钟）
// maxLifetime: 1800000（30 分钟）
```

### Q3：@Transactional 注解的失效场景？

```java
// 1. 自调用（同一类中方法调用）
// 2. 修饰非 public 方法
// 3. 异常被 try-catch 捕获未抛出
// 4. 异常类型不是 RuntimeException（需配置 rollbackFor）
// 5. 数据库不支持事务（如 MySQL MyISAM 引擎）
// 6. 在异步线程中执行（事务与线程绑定）
// 7. 传播行为配置错误（如 REQUIRES_NEW 但事务管理器没配好）
```

### Q4：MyBatis 的 #{} 和 ${} 区别？

```java
// #{}：预编译占位符（PreparedStatement ?）
//   自动加引号
//   防 SQL 注入
//   使用：所有用户输入的值

// ${}：直接字符串拼接
//   不加引号
//   有 SQL 注入风险
//   使用：表名、列名等数据库标识符
//   ORDER BY ${columnName}

// 示例：
// SELECT * FROM user WHERE name = #{name}
// → SELECT * FROM user WHERE name = ?
// → SELECT * FROM user WHERE name = 'Alice'

// SELECT * FROM ${tableName}
// → SELECT * FROM user_2024
// ${} 用于动态表名（如分表查询）
```

### Q5：什么是 N+1 查询问题？

```java
// N+1 问题：先查 1 次获取结果列表，再循环查 N 次获取关联数据

// MyBatis 中的 N+1：
// <resultMap id="blogMap" type="Blog">
//     <association property="author" column="author_id"
//                  select="selectAuthor"/>  <!-- 每次循环都执行 -->
// </resultMap>
// <select id="selectBlog" resultMap="blogMap">
//     SELECT * FROM blog
// </select>
// 1 次 blog 查询 + N 次 author 查询 = N+1 次

// 解决方法：
// 1. 使用 JOIN 查询（推荐）
// 2. 使用 @Many 的 fetchType=LAZY（延迟加载，按需查询）
// 3. 使用 MyBatis 的 collection 进行嵌套查询（left join）
// 4. 使用 Hibernate 的 batch fetching

// 另一种 N+1 场景：
// for (Order order : orderList) {          // 1 次查询
//     for (OrderItem item : order.getItems()) { }  // N 次查询（延迟加载触发）
// }
```

---

## 📚 参考资料

- 《高性能 MySQL》—— 数据库优化经典
- 《MyBatis 技术内幕》—— MyBatis 源码分析
- 《深入理解 Apache ShardingSphere》
- [HikariCP 官方 Wiki](https://github.com/brettwooldridge/HikariCP/wiki)
- [MyBatis 官方文档](https://mybatis.org/mybatis-3/)
