# Lambda 表达式与 Stream API

## 📖 本章导读

### 学习目标
通过本章学习,你将掌握:
- ✅ Lambda表达式的语法和函数式接口
- ✅ JDK 8内置的六大函数式接口(Predicate/Consumer/Function等)
- ✅ 方法引用的四种形式
- ✅ 变量捕获和effectively final规则
- ✅ Stream API的创建、中间操作和终端操作
- ✅ Collectors工具类的常用收集器
- ✅ 并行流的使用场景和注意事项
- ✅ Optional的正确使用方式

### 核心概念
**Lambda表达式**是Java 8引入的函数式编程特性,允许将代码作为参数传递。**Stream API**提供了声明式的数据处理方式,支持串行和并行操作。两者结合使Java具备了现代函数式编程语言的能力,大幅简化集合操作。

### 知识地图
```
Lambda与Stream体系
├── Lambda表达式
│   ├── 基本语法
│   │   ├── 无参数:() -> System.out.println("Hello")
│   │   ├── 单参数:x -> x * 2(可省略括号)
│   │   └── 多参数:(x, y) -> x + y
│   ├── 函数式接口
│   │   ├── Predicate<T>:断言,输入T返回boolean
│   │   ├── Consumer<T>:消费,输入T无返回
│   │   ├── Function<T,R>:转换,输入T返回R
│   │   ├── Supplier<T>:供应,无输入返回T
│   │   ├── UnaryOperator<T>:一元操作
│   │   └── BinaryOperator<T>:二元操作
│   ├── 方法引用
│   │   ├── 静态方法:ClassName::staticMethod
│   │   ├── 实例方法:instance::method
│   │   ├── 特定类型:ClassName::method
│   │   ├── 构造器:ClassName::new
│   │   └── 数组:Type[]::new
│   ├── 变量捕获
│   │   ├── effectively final规则
│   │   ├── this指向外部类(非匿名内部类自身)
│   │   └── 修改捕获变量需用原子类或数组
│   └── 编译原理
│       ├── invokedynamic指令(非匿名内部类)
│       ├── LambdaMetafactory生成适配器
│       └── 无额外.class文件,JIT友好
└── Stream API
    ├── 创建Stream
    │   ├── 集合:list.stream() / list.parallelStream()
    │   ├── 数组:Arrays.stream(arr)
    │   ├── 值:Stream.of("a", "b")
    │   ├── 函数:Stream.generate() / Stream.iterate()
    │   └── 原始类型:IntStream.range()
    ├── 中间操作(Lazy)
    │   ├── filter:过滤
    │   ├── map:一对一转换
    │   ├── flatMap:一对多扁平化
    │   ├── limit/skip:截断/跳过
    │   ├── distinct:去重
    │   ├── sorted:排序
    │   └── peek:调试用
    ├── 终端操作(Eager)
    │   ├── collect:收集(Collectors.toList/toSet/toMap)
    │   ├── forEach/forEachOrdered:遍历
    │   ├── count/anyMatch/allMatch:计数/匹配
    │   ├── findFirst/findAny:查找
    │   ├── reduce:归约
    │   └── min/max:最值
    ├── Collectors工具类
    │   ├── toList/toSet/toMap
    │   ├── joining:字符串拼接
    │   ├── groupingBy:分组
    │   ├── partitioningBy:分区(true/false)
    │   ├── mapping:下游收集器
    │   └── summarizingInt:汇总统计
    ├── 并行流
    │   ├── ForkJoinPool.commonPool()
    │   ├── 适合:数据量大(>10000)+CPU密集型
    │   └── 注意:有状态操作可能更慢,避免副作用
    ├── 原始类型流
    │   ├── IntStream/LongStream/DoubleStream
    │   ├── 避免装箱开销
    │   └── boxed():转包装类型流
    └── Optional(Stream伙伴)
        ├── 创建:Optional.of()/ofNullable()/empty()
        ├── 取值:orElse()/orElseGet()/orElseThrow()
        ├── 转换:map()/flatMap()
        ├── 消费:ifPresent()/ifPresentOrElse()
        └── 误用:不做字段或参数,仅作返回类型
```

### 常见误区
❌ **误区1**: Lambda就是匿名内部类的简写
✅ **真相**: Lambda使用invokedynamic指令,不会生成.class文件,this指向也不同

❌ **误区2**: 并行流一定比串行流快
✅ **真相**: 小数据量或有状态操作(如sorted)并行反而更慢,需权衡线程开销

❌ **误区3**: Stream可以复用
✅ **真相**: Stream只能消费一次,再次使用会抛IllegalStateException

❌ **误区4**: Optional可以作为字段或方法参数
✅ **真相**: Optional仅作为返回类型,字段应直接使用null-safe设计

❌ **误区5**: flatMap和map一样
✅ **真相**: map是一对一转换,flatMap是一对多转换并扁平化(如展开嵌套集合)

### 实战建议
1. **优先使用Lambda替代匿名内部类**: 代码更简洁,性能更好
2. **优先使用方法引用**: 可读性优于Lambda,如String::toUpperCase
3. **Stream管道避免副作用**: 纯函数式风格,不在forEach中修改外部状态
4. **适当使用并行流**: 仅对大数据量+CPU密集型操作,IO密集型不适合
5. **Optional仅作为返回类型**: 不做字段或参数,避免过度封装
6. **用Collectors.toCollection()指定集合类型**: 比默认toList更可控
7. **复杂管道拆分**: 过长Stream链可拆分为多个方法提高可读性
8. **原始类型流避免装箱**: IntStream比Stream<Integer>性能更好
9. **Stream不可复用**: 需要时重新创建,不要缓存Stream对象
10. **用joining代替StringBuilder**: Stream.collect(Collectors.joining())更优雅

## 1. Lambda 表达式

### 1.1 基本语法

```java
// 语法：(参数) -> { 方法体 }
// 本质：函数式接口的匿名实现

// 无参数
() -> System.out.println("Hello")

// 单个参数（可省略括号）
x -> x * 2

// 多个参数
(x, y) -> x + y

// 多行方法体（需要大括号和 return）
(x, y) -> {
    int sum = x + y;
    return sum;
}

// 类型推断（参数类型可省略）
Comparator<Person> byAge = (p1, p2) -> Integer.compare(p1.getAge(), p2.getAge());
```

### 1.2 函数式接口

```java
// 函数式接口：只有一个抽象方法的接口
// 可以使用 @FunctionalInterface 注解检查

@FunctionalInterface
public interface MyFunction {
    void doSomething();
    // void doOther();  // ❌ 编译错误，只能有一个抽象方法
}

// JDK 8 内置的函数式接口（java.util.function 包）

// 1. Predicate<T>：断言，输入 T，返回 boolean
Predicate<String> isEmpty = s -> s.isEmpty();
Predicate<String> notEmpty = s -> !s.isEmpty();
Predicate<String> combined = isEmpty.negate().and(s -> s.length() > 3);

// 2. Consumer<T>：消费，输入 T，无返回
Consumer<String> printer = s -> System.out.println(s);
Consumer<String> logger = s -> log.info(s);
Consumer<String> both = printer.andThen(logger);

// 3. Function<T, R>：转换，输入 T，返回 R
Function<String, Integer> toLength = s -> s.length();
Function<Integer, String> toString = i -> "Number: " + i;
Function<String, String> composed = toLength.andThen(toString);
// composed.apply("hello") → "Number: 5"

// 4. Supplier<T>：供应，无输入，返回 T
Supplier<String> helloSupplier = () -> "Hello";
Supplier<Double> randomSupplier = Math::random;

// 5. UnaryOperator<T>：一元操作，输入 T，返回 T（Function 的特化）
UnaryOperator<String> toUpper = String::toUpperCase;

// 6. BinaryOperator<T>：二元操作，输入 (T, T)，返回 T
BinaryOperator<Integer> max = Integer::max;

// 基本类型特化（避免装箱）
IntPredicate even = i -> i % 2 == 0;
IntFunction<String> intToString = String::valueOf;
ToIntFunction<String> parse = Integer::parseInt;
```

### 1.3 方法引用

```java
// 方法引用是 Lambda 的简写形式

// 1. 静态方法引用：ClassName::staticMethod
Function<String, Integer> parser = Integer::parseInt;
// 等价于：s -> Integer.parseInt(s)

// 2. 实例方法引用：instance::method
Consumer<String> printer = System.out::println;
// 等价于：s -> System.out.println(s)

// 3. 特定类型的实例方法引用：ClassName::method
Function<String, String> toUpper = String::toUpperCase;
// 等价于：s -> s.toUpperCase()

// 4. 构造器引用：ClassName::new
Supplier<List<String>> listCreator = ArrayList::new;
// 等价于：() -> new ArrayList<>()

Function<String, StringBuilder> sbCreator = StringBuilder::new;
// 等价于：s -> new StringBuilder(s)

// 5. 数组构造器引用：Type[]::new
IntFunction<int[]> arrayCreator = int[]::new;
// 等价于：size -> new int[size]
```

### 1.4 变量捕获

```java
// Lambda 可以捕获外部变量，但变量必须是 effectively final

// 可以捕获
String prefix = "User: ";
Function<String, String> addPrefix = name -> prefix + name;
// prefix 没有被重新赋值（effectively final）

// 不能修改捕获的变量
int count = 0;
Runnable task = () -> {
    // count++;  // ❌ 编译错误！捕获的变量不能修改
};

// 要修改，使用原子类或数组
int[] counter = {0};
Runnable task2 = () -> counter[0]++;  // ✅ 数组引用是 effectively final

// this 的指向
// Lambda 中的 this 指向外部类（而非匿名内部类那样指向 Lambda 自身）
public class LambdaThis {
    private String value = "outer";

    public void test() {
        Runnable r = () -> System.out.println(this.value);
        // this 指向 LambdaThis 实例
        r.run();  // "outer"
    }
}

// 对比匿名内部类
public class AnonymousThis {
    private String value = "outer";

    public void test() {
        Runnable r = new Runnable() {
            private String value = "inner";
            @Override
            public void run() {
                System.out.println(this.value);  // "inner"（指向匿名类自身）
            }
        };
        r.run();
    }
}
```

### 1.5 Lambda 的编译原理

```java
// Lambda 不是匿名内部类！
// 使用 invokedynamic 指令 + java.lang.invoke.LambdaMetafactory

// 源码：
Runnable r = () -> System.out.println("Hello");

// 编译后等价于（伪代码）：
// 1. 编译器生成一个静态方法：lambda$0()
// private static void lambda$0() {
//     System.out.println("Hello");
// }
//
// 2. 通过 invokedynamic + LambdaMetafactory 转换为 Runnable 实例
// 3. invokedynamic 在首次调用时生成 Runnable 的适配器类
//    （不会像匿名内部类那样每次 new 一个新类）

// 优势：
// 1. 没有匿名内部类的类文件生成
// 2. 首次调用后直接内联（JIT 友好）
// 3. 不捕获变量时，Runnable 实例是单例
```

---

## 2. Stream API

### 2.1 创建 Stream

```java
// 1. 从集合
Stream<String> fromList = list.stream();
Stream<String> parallelStream = list.parallelStream();  // 并行流

// 2. 从数组
Stream<String> fromArray = Arrays.stream(arr);
Stream<String> fromArrayPart = Arrays.stream(arr, 0, 2);  // arr[0..2)

// 3. 从值
Stream<String> of = Stream.of("a", "b", "c");
Stream<Integer> single = Stream.of(42);
Stream<Object> empty = Stream.empty();

// 4. 从文件
try (Stream<String> lines = Files.lines(Paths.get("file.txt"))) { }

// 5. 从函数生成
Stream<Double> randoms = Stream.generate(Math::random);      // 无限流
Stream<Integer> iterate = Stream.iterate(0, n -> n + 2);     // 无限流
Stream<Integer> bounded = Stream.iterate(0, n -> n < 100, n -> n + 2);  // JDK 9+ 有界

// 6. 从其他 API
IntStream intStream = IntStream.range(0, 10);    // [0, 1, ..., 9]
IntStream closed = IntStream.rangeClosed(0, 9);  // [0, 1, ..., 9]
IntStream.range(0, 10).boxed();  // IntStream → Stream<Integer>

// 7. 字符串字符
IntStream chars = "hello".chars();   // 每个字符的 Unicode 码点
```

### 2.2 中间操作（Lazy）

```java
// 中间操作返回新的 Stream，延迟执行（终端操作时才执行）

List<String> list = Arrays.asList("apple", "banana", "cherry", "date");

// filter：过滤
list.stream()
    .filter(s -> s.startsWith("a"))
    .collect(Collectors.toList());

// map：转换（一对一）
list.stream()
    .map(String::toUpperCase)
    .map(s -> s.length())
    .collect(Collectors.toList());  // [5, 6, 6, 4]

// flatMap：扁平化（一对多）
List<List<String>> nested = Arrays.asList(
    Arrays.asList("a", "b"),
    Arrays.asList("c", "d")
);
nested.stream()
    .flatMap(Collection::stream)           // 展开子集合
    .collect(Collectors.toList());         // [a, b, c, d]

// flatMap 的实际用途：空 Optional 过滤
List<Optional<String>> optionals = Arrays.asList(
    Optional.of("a"), Optional.empty(), Optional.of("b")
);
optionals.stream()
    .flatMap(Optional::stream)             // JDK 9+ 的 Optional.stream()
    .collect(Collectors.toList());         // [a, b]

// limit：截断
list.stream().limit(2).collect(Collectors.toList());  // [apple, banana]

// skip：跳过
list.stream().skip(2).collect(Collectors.toList());   // [cherry, date]

// distinct：去重
Arrays.asList(1, 2, 2, 3, 3, 3).stream()
    .distinct()
    .collect(Collectors.toList());  // [1, 2, 3]

// sorted：排序
list.stream()
    .sorted()
    .sorted(Comparator.comparingInt(String::length))
    .collect(Collectors.toList());

// peek：调试用（中间操作的不动点）
list.stream()
    .peek(s -> System.out.println("Processing: " + s))
    .map(String::toUpperCase)
    .collect(Collectors.toList());
```

### 2.3 终端操作（Eager）

```java
List<String> list = Arrays.asList("apple", "banana", "cherry");

// collect：收集到集合
List<String> listResult = list.stream().collect(Collectors.toList());
Set<String> setResult = list.stream().collect(Collectors.toSet());
ArrayList<String> customList = list.stream()
    .collect(Collectors.toCollection(ArrayList::new));

// toList（JDK 16+，不可变）
List<String> immutable = list.stream().toList();

// forEach：遍历
list.stream().forEach(System.out::println);

// forEachOrdered：保证顺序（并行流时）
list.parallelStream().forEachOrdered(System.out::println);

// count：计数
long count = list.stream().filter(s -> s.length() > 5).count();

// anyMatch / allMatch / noneMatch：匹配检查
boolean hasApple = list.stream().anyMatch("apple"::equals);
boolean allLong = list.stream().allMatch(s -> s.length() > 3);
boolean noneEmpty = list.stream().noneMatch(String::isEmpty);

// findFirst / findAny：查找
Optional<String> first = list.stream().findFirst();
Optional<String> any = list.parallelStream().findAny();  // 并行流中 findAny 更快

// reduce：归约
Optional<String> concat = list.stream().reduce((a, b) -> a + ", " + b);
int sum = IntStream.range(0, 10).reduce(0, Integer::sum);  // 45
int sumWithIdentity = IntStream.range(0, 10).reduce(0, Integer::sum);
// 有 identity 时，返回类型不是 Optional（因为一定有默认值）

// min / max
Optional<String> shortest = list.stream()
    .min(Comparator.comparingInt(String::length));
```

### 2.4 Collectors 工具类

```java
List<String> list = Arrays.asList("apple", "banana", "cherry", "date");

// toList / toSet / toMap
Map<Integer, String> map = list.stream()
    .collect(Collectors.toMap(
        String::length,           // key mapper
        Function.identity(),      // value mapper
        (v1, v2) -> v1            // key 冲突处理（取第一个）
    ));

// joining：字符串拼接
String joined = list.stream()
    .collect(Collectors.joining(", ", "[", "]"));
// "[apple, banana, cherry, date]"

// groupingBy：分组
Map<Integer, List<String>> byLength = list.stream()
    .collect(Collectors.groupingBy(String::length));
// {4=[date], 5=[apple], 6=[banana, cherry]}

Map<Integer, Set<String>> byLengthSet = list.stream()
    .collect(Collectors.groupingBy(
        String::length,
        Collectors.toSet()
    ));

Map<Integer, Long> countByLength = list.stream()
    .collect(Collectors.groupingBy(
        String::length,
        Collectors.counting()
    ));
// {4=1, 5=1, 6=2}

// partitioningBy：分区（逻辑分组，只有 true/false 两组）
Map<Boolean, List<String>> partitioned = list.stream()
    .collect(Collectors.partitioningBy(s -> s.length() > 5));
// {false=[apple, date], true=[banana, cherry]}

// mapping：下游收集器
Map<Integer, List<String>> upperByLength = list.stream()
    .collect(Collectors.groupingBy(
        String::length,
        Collectors.mapping(String::toUpperCase, Collectors.toList())
    ));

// summarizingInt：汇总统计
IntSummaryStatistics stats = list.stream()
    .collect(Collectors.summarizingInt(String::length));
stats.getSum();
stats.getAverage();
stats.getMax();
```

### 2.5 并行流

```java
// 并行流使用 ForkJoinPool.commonPool()
// 默认并行度 = Runtime.getRuntime().availableProcessors() - 1

// 创建并行流
list.parallelStream();
list.stream().parallel();

// 操作本身无需改动
int sum = IntStream.range(0, 10_000_000)
    .parallel()
    .sum();

// 并行流需要注意的问题：
// 1. 有状态的操作（sorted、distinct、limit）在并行流中可能更慢
// 2. 共享可变状态（如 ArrayList.add）线程不安全
// 3. 小数据量不适合并行（线程创建开销 > 并行加速）

// 并行流的正确使用：
// - 数据量大（>10,000 元素）
// - CPU 密集型操作
// - 操作可分解（无状态、无依赖）

// ❌ 错误：有副作用
List<Integer> result = new ArrayList<>();
IntStream.range(0, 100)
    .parallel()
    .forEach(result::add);  // 线程不安全！

// ✅ 正确：无状态收集
List<Integer> result2 = IntStream.range(0, 100)
    .parallel()
    .boxed()
    .collect(Collectors.toList());
```

### 2.6 原始类型流

```java
// IntStream、LongStream、DoubleStream：避免装箱

IntStream ints = IntStream.of(1, 2, 3);
IntStream range = IntStream.range(0, 10);
IntStream randomInts = new Random().ints(10);  // 10 个随机数

int sum = ints.sum();                  // 求和
OptionalDouble avg = ints.average();   // 平均值
OptionalInt max = ints.max();          // 最大值
IntSummaryStatistics stats = ints.summaryStatistics();  // 全部统计

// 类型转换
IntStream intStream = IntStream.range(0, 10);
Stream<Integer> boxed = intStream.boxed();   // → Stream<Integer>
Stream<String> mapped = intStream.mapToObj(String::valueOf);  // → Stream<String>
Stream<Double> doubleStream = intStream.asDoubleStream();      // → DoubleStream
```

### 2.7 Optional（Stream 的伙伴）

```java
// 创建
Optional<String> empty = Optional.empty();
Optional<String> of = Optional.of("hello");          // 非 null
Optional<String> ofNullable = Optional.ofNullable(null);  // 可为 null

// 检查
opt.isPresent();    // JDK 11+ 已弃用，用 isEmpty() 代替
opt.isEmpty();      // JDK 11+

// 取值
opt.get();                     // 空则 NoSuchElementException
opt.orElse("default");         // 默认值
opt.orElseGet(() -> compute());  // 延迟计算
opt.orElseThrow();             // JDK 10+
opt.orElseThrow(RuntimeException::new);

// 转换
opt.map(String::toUpperCase);
opt.flatMap(s -> Optional.of(s.length()));

// 过滤
opt.filter(s -> s.length() > 3);

// 消费
opt.ifPresent(System.out::println);
opt.ifPresentOrElse(
    System.out::println,
    () -> System.out.println("empty")
);

// 流式操作（JDK 9+）
opt.stream()  // Optional → Stream（0 或 1 个元素）
    .filter(...)
    .collect(...);

// Optional 的误用（不要作为字段或参数）
public class User {
    // ❌
    private Optional<String> name;
    // ✅
    private String name;
}
```

---

## 3. 面试题

### Q1: Lambda 和匿名内部类的区别？
```java
// 1. this 指向：Lambda 指向外部类，匿名内部类指向自身
// 2. 编译方式：Lambda 用 invokedynamic，匿名内部类生成 .class 文件
// 3. 性能：Lambda 无额外类加载开销，JIT 易内联
// 4. 变量捕获：两者都要求 effectively final
```

### Q2: Stream 的中间操作和终端操作的区别？
```java
// 中间操作：lazy（延迟执行），返回 Stream，可以链式调用
// 终端操作：eager（立即触发计算），消费 Stream 后关闭
//
// 延迟执行的意义：
// 1. 可以优化操作顺序（filter → map 合并遍历）
// 2. 短路操作（limit、anyMatch）无需处理全部数据
```

### Q3: 并行流一定更快吗？
```java
// 不一定。并行流有额外开销：
// 1. 线程创建和上下文切换
// 2. 数据分割和结果合并
// 3. 内存局部性影响
//
// 适合并行流的场景：
// - 数据量大（>10,000）
// - 操作是 CPU 密集型
// - 操作无状态、可分解
//
// 不适合：
// - 数据量小
// - 操作是 IO 密集型
// - 有顺序要求
```

### Q4: Stream 和集合的区别？
```java
// 集合：数据存储结构（内存中的数据结构）
// Stream：数据计算管道（不存储数据，延迟计算）
//
// 1. Stream 不修改源数据（产生新结果）
// 2. Stream 只能消费一次（没有复用）
// 3. Stream 是函数式管道，集合是数据结构
```

### Q5: flatMap 和 map 的区别？
```java
// map：一对一转换（每个元素映射为一个结果）
// flatMap：一对多转换，然后将结果扁平化
//
// map：Stream<T> → Stream<R>（元素数量不变）
// flatMap：Stream<T> → Stream<R>（元素数量可增可减）
```

---

## 4. 最佳实践

1. **优先使用 Lambda 替代匿名内部类**：代码更简洁
2. **优先使用方法引用**：可读性更好
3. **Stream 管道避免副作用**：纯函数式风格
4. **适当使用并行流**：仅对大数据量 + CPU 密集型
5. **Optional 仅作为返回类型**：不做字段或参数
6. **用 `Collectors.toCollection()` 指定集合类型**：比默认更可控
7. **复杂管道拆分**：过长管道可拆分为多个方法
8. **避免在 forEach 中修改外部状态**：用 collect 代替
9. **Stream 不可复用**：需要时重新创建
10. **原始类型流避免装箱**：IntStream、LongStream、DoubleStream
