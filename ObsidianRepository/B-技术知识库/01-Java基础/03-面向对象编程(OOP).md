# Java 面向对象编程（OOP）

## 📖 本章导读

### 学习目标
通过本章学习,你将掌握:
- ✅ 类和对象的概念及内存布局
- ✅ 封装、继承、多态三大特性
- ✅ 抽象类与接口的区别和选择
- ✅ final和static关键字的深层含义
- ✅ Object类的核心方法(equals、hashCode、toString)
- ✅ 设计原则(组合优于继承)

### 核心概念
**面向对象编程(OOP)**是Java的核心范式,通过**封装**隐藏实现细节,**继承**实现代码复用,**多态**提供灵活扩展。理解OOP对于构建可维护、可扩展的系统至关重要。

### 知识地图
```
面向对象编程
├── 类与对象
│   ├── 类的定义: 字段、方法、构造器
│   ├── 对象创建: new关键字
│   └── 内存布局: 对象头+实例数据+对齐填充
├── 封装(Encapsulation)
│   ├── 访问控制: private, default, protected, public
│   ├── Getter/Setter: JavaBean规范
│   └── 防御性拷贝: 保护内部状态
├── 继承(Inheritance)
│   ├── extends关键字: 单继承
│   ├── super关键字: 调用父类
│   ├── 构造器链: 初始化顺序
│   └── 字段隐藏vs方法重写
├── 多态(Polymorphism)
│   ├── 编译时多态: 方法重载(Overload)
│   ├── 运行时多态: 方法重写(Override)
│   └── 动态绑定: vtable虚方法表
├── 抽象类与接口
│   ├── 抽象类: abstract class, 部分实现
│   ├── 接口: interface, JDK 8+默认方法
│   └── 选择原则: is-a用抽象类, can-do用接口
├── 关键字
│   ├── final: 不可变(类/方法/变量)
│   └── static: 类级别(字段/方法/块)
└── Object类
    ├── equals & hashCode: 相等性判断
    ├── toString: 字符串表示
    ├── clone: 对象拷贝(浅拷贝)
    └── finalize: 已废弃,用Cleaner替代
```

### 常见误区
❌ **误区1**: `Parent p = new Child(); p.field` 会访问子类的字段
✅ **真相**: 字段没有多态,访问的是编译时类型(Parent)的字段

❌ **误区2**: 子类可以继承父类的private成员
✅ **真相**: private成员不可见,但存在于子类对象中(可通过反射访问)

❌ **误区3**: equals相等则hashCode一定相等,反之亦然
✅ **真相**: equals相等→hashCode必相等; hashCode相等→equals不一定相等(哈希冲突)

❌ **误区4**: 继承总是好的,应该优先使用
✅ **真相**: 组合优于继承,继承破坏封装,仅在is-a关系时使用

### 实战建议
1. **封装原则**: 字段private,通过getter/setter访问,添加验证逻辑
2. **继承谨慎**: 优先使用组合,仅在明确is-a关系时使用继承
3. **多态编程**: 面向接口编程,提高代码灵活性
4. **final优化**: 不可变类用final,JVM可进行性能优化
5. **equals/hashCode**: 始终一起重写,使用Objects工具类
6. **接口设计**: JDK 8+使用default方法提供向后兼容
7. **防御性拷贝**: 返回可变对象时返回拷贝,防止外部修改

---

## 1. 类与对象基础

### 1.1 类的定义与对象的创建

```java
// 类：对象的模板
public class Person {
    // 字段（成员变量）
    String name;
    int age;

    // 构造器
    public Person(String name, int age) {
        this.name = name;
        this.age = age;
    }

    // 方法
    public void sayHello() {
        System.out.println("Hello, I'm " + name);
    }
}

// 对象：类的实例
Person p = new Person("Alice", 25);
p.sayHello();
```

**对象的创建过程（内存视角）：**
```java
Person p = new Person("Alice", 25);
```
```
1. 类加载检查：JVM 检查 Person 类是否已加载（方法区）
2. 分配堆内存：为对象分配连续内存（Eden 区），所有字段初始化为默认值
3. 设置对象头：Mark Word + Klass Pointer（指向方法区中的类元数据）
4. 执行实例初始化：<init> 方法（构造器），包括：
   a. 父类构造器（super()）
   b. 实例变量初始化和实例初始化块（按代码顺序）
   c. 构造器中的代码
5. 返回引用：栈中的 p 指向堆中的对象
```

### 1.2 对象内存布局（HotSpot JVM）

```
┌─────────────────────────────────────┐
│         对象头（Header）              │
│  ├─ Mark Word（标记字）              │
│  │   哈希码、GC 年龄、锁状态标志等      │
│  │   64 位 JVM 下通常 8 字节           │
│  │   32 位 JVM 下通常 4 字节           │
│  │                                    │
│  └─ Klass Pointer（类指针）          │
│       指向方法区中的 Klass 对象         │
│       默认开启指针压缩（4 字节）         │
│       不压缩时 8 字节                   │
├─────────────────────────────────────┤
│      实例数据（Instance Data）          │
│  ├─ 父类声明的字段                      │
│  └─ 本类声明的字段                      │
│       按大小和类型排序（长整型优先）       │
├─────────────────────────────────────┤
│      对齐填充（Padding）                │
│       8 字节对齐（对象大小是 8 的倍数）    │
└─────────────────────────────────────┘
```

**指针压缩（Compressed OOPs）：**
```java
// -XX:+UseCompressedOops（默认开启）
// 堆大小 < 32GB 时，引用使用 4 字节表示（而非 8 字节）
// 堆大小 > 32GB 时，自动关闭指针压缩

// 验证对象大小（依赖 jol 工具）
// ClassLayout.parseClass(Person.class).toPrintable()
```

**对象头中的 Mark Word（64 位 JVM）：**
```
无锁状态：  | unused:25 | identity_hashcode:31 | unused:1 | age:4 | biased_lock:1 | lock:2 |
偏向锁：    | thread:54 | epoch:2 | unused:1 | age:4 | biased_lock:1 | lock:2 |
轻量级锁：  | ptr_to_lock_record:62 | lock:2 |
重量级锁：  | ptr_to_heavyweight_monitor:62 | lock:2 |
GC 标记：   | marking_phase | lock:2 |
```

---

## 2. 封装（Encapsulation）

### 2.1 访问控制修饰符

| 修饰符 | 本类 | 同包 | 子类 | 全局 |
|:---|:---:|:---:|:---:|:---:|
| `private` | ✅ | - | - | - |
| （默认）package-private | ✅ | ✅ | - | - |
| `protected` | ✅ | ✅ | ✅ | - |
| `public` | ✅ | ✅ | ✅ | ✅ |

```java
public class EncapsulationDemo {
    private int privateField;       // 仅本类可见
    int packageField;               // 包级私有（默认）
    protected int protectedField;   // 子类 + 同包可见
    public int publicField;         // 全部可见

    private void privateMethod() { }
    void packageMethod() { }
    protected void protectedMethod() { }
    public void publicMethod() { }
}
```

**访问控制的底层实现：**
```java
// 访问控制是编译期检查（非运行期）
// JVM 字节码层面没有 private/public 的区别
// 反射可以绕过访问控制（setAccessible(true)）

// 示例：反射突破 private
Field field = obj.getClass().getDeclaredField("privateField");
field.setAccessible(true);  // 抑制 Java 访问控制检查
field.set(obj, "hacked");
```

### 2.2 Getter/Setter 与 JavaBean 规范

```java
public class Person {
    private String name;
    private int age;

    // Getter 规范：get + 字段名（首字母大写）
    // boolean 类型：is + 字段名
    public String getName() { return name; }
    public boolean isActive() { return active; }

    // Setter 规范：set + 字段名
    public void setName(String name) { this.name = name; }

    // 封装的优势：可以在 setter 中添加验证逻辑
    public void setAge(int age) {
        if (age < 0 || age > 150) {
            throw new IllegalArgumentException("Invalid age: " + age);
        }
        this.age = age;
    }
}

// JavaBean 规范（和序列化相关）：
// 1. 有无参构造器
// 2. 属性有 getter/setter
// 3. 实现 Serializable
```

### 2.3 防御性拷贝（Defensive Copy）

```java
public class Period {
    private final Date start;
    private final Date end;

    // 构造器防御
    public Period(Date start, Date end) {
        // 防御性拷贝（在验证前拷贝）
        this.start = new Date(start.getTime());  // 拷贝而非直接赋值
        this.end = new Date(end.getTime());

        if (this.start.after(this.end)) {
            throw new IllegalArgumentException("start > end");
        }
    }

    // Getter 防御（返回拷贝而非原始引用）
    public Date getStart() {
        return new Date(start.getTime());
    }
    public Date getEnd() {
        return new Date(end.getTime());
    }
}

// 攻击代码（如果没有防御性拷贝）：
Date start = new Date();
Date end = new Date();
Period period = new Period(start, end);
end.setYear(2020);  // ❌ 修改了 period 内部状态！
```

---

## 3. 继承（Inheritance）

### 3.1 继承的语法与机制

```java
public class Animal {
    protected String name;
    private int age;  // 子类不可直接访问

    public Animal(String name) {
        this.name = name;
    }

    public void eat() {
        System.out.println(name + " is eating");
    }

    private void breathe() { }  // 子类不可继承
}

public class Dog extends Animal {
    public Dog(String name) {
        super(name);  // 必须调用父类构造器（第一行）
    }

    @Override
    public void eat() {
        super.eat();  // 调用父类方法
        System.out.println(name + " eats dog food");
    }
}
```

**继承的特点：**
```java
// 1. Java 单继承：一个类最多有一个直接父类
// class A extends B extends C { }  // ❌ 不可链式继承
// class A extends B, C { }          // ❌ 不支持多继承

// 2. 继承链长度不限
// Object → Animal → Mammal → Dog → Puppy

// 3. 子类可以访问父类的 protected 和 public 成员
// 4. 子类不能继承父类的 private 成员
// 5. 子类可以重写（override）父类的方法
// 6. 构造器不被继承，但必须调用 super()
```

### 3.2 super 关键字

```java
public class Parent {
    public Parent(String msg) {
        System.out.println("Parent: " + msg);
    }
    public void method() { System.out.println("parent"); }
}

public class Child extends Parent {
    public Child() {
        super("hello");  // 调用父类构造器（必须在第一行）
        System.out.println("Child");
    }

    @Override
    public void method() {
        super.method();  // 调用父类被重写的方法
        System.out.println("child");
    }
}
```

**super 的本质：**
```java
// super 不是引用（不能传参、不能赋值）
// super 是编译器关键字，用于生成 invokespecial 指令调用父类方法
// 而 this 是引用（可作为参数传递）

// super 访问父类字段（但会被遮蔽而非真正访问）
// 实际上 super.field 编译为：aload_0; getfield Parent.field
// 因为子类和父类同名字段在内存中同时存在
```

### 3.3 构造器链

```java
public class A {
    public A() { System.out.println("A"); }
}

public class B extends A {
    public B() { System.out.println("B"); }  // 隐式调用 super()
}

public class C extends B {
    public C() { System.out.println("C"); }
}

new C();  // 输出：A\nB\nC
// 构造器调用链：C() → B() → A() → Object()

// 隐式 super()：子类构造器未显式调用 super 时，编译器自动插入 super()
// 如果父类没有无参构造器，必须显式调用 super(参数)
```

**构造器执行顺序的完整流程：**
```java
public class Parent {
    private int x = initX();  // 2. 父类实例变量初始化

    { System.out.println("Parent init block"); }  // 3. 父类初始化块

    public Parent() {
        System.out.println("Parent constructor");  // 4. 父类构造器体
    }

    static { System.out.println("Parent static"); }  // 1. 父类静态块（类加载时）
}

public class Child extends Parent {
    private int y = initY();  // 5. 子类实例变量初始化

    { System.out.println("Child init block"); }  // 6. 子类初始化块

    public Child() {
        System.out.println("Child constructor");  // 7. 子类构造器体
    }

    static { System.out.println("Child static"); }  // 1. 子类静态块（类加载时）
}

new Child();
// 输出：
// Parent static
// Child static
// Parent init block
// Parent constructor
// Child init block
// Child constructor
```

### 3.4 继承中的字段隐藏（Shadowing）

```java
public class Parent {
    public int x = 10;
}

public class Child extends Parent {
    public int x = 20;  // 隐藏（hide）父类的 x，非重写

    public void printX() {
        System.out.println(x);           // 20（子类的 x）
        System.out.println(super.x);     // 10（父类的 x）
    }
}

// 多态不适用于字段！
Parent obj = new Child();
System.out.println(obj.x);  // 10（编译时类型决定，不是多态！）
Child child = (Child) obj;
System.out.println(child.x);  // 20

// 成员方法才有多态
```

### 3.5 继承的设计原则（组合优先于继承）

```java
// ❌ 不恰当的继承（破坏封装）
public class InstrumentedHashSet<E> extends HashSet<E> {
    private int addCount = 0;

    @Override
    public boolean add(E e) {
        addCount++;
        return super.add(e);
    }

    @Override
    public boolean addAll(Collection<? extends E> c) {
        addCount += c.size();  // ❌ 错误！addAll 内部调用了 add
        return super.addAll(c);
    }
    // 实际 addCount 被加了两次（addAll → add 每次也 ++）
}

// ✅ 组合优于继承
public class InstrumentedSet<E> {
    private final Set<E> set;    // 组合
    private int addCount = 0;

    public InstrumentedSet(Set<E> set) { this.set = set; }

    public boolean add(E e) {
        addCount++;
        return set.add(e);
    }

    public boolean addAll(Collection<? extends E> c) {
        addCount += c.size();
        return set.addAll(c);
    }

    public int getAddCount() { return addCount; }
}
```

**何时使用继承？**
```
使用继承的条件（同时满足）：
1. 子类是父类的"is-a"关系
2. 父类的 API 设计为继承（有文档说明可以重写哪些方法）
3. 不违反父类的封装
4. 不会因为父类修改导致子类损坏

否则优先使用组合（Composition）
```

---

## 4. 多态（Polymorphism）

### 4.1 多态的三种形式

```java
public class Animal {
    public void sound() { System.out.println("..."); }
}

public class Dog extends Animal {
    @Override
    public void sound() { System.out.println("Woof"); }
}

public class Cat extends Animal {
    @Override
    public void sound() { System.out.println("Meow"); }
}

// 1. 编译时多态（方法重载 Overload）
public class Calculator {
    public int add(int a, int b) { return a + b; }
    public double add(double a, double b) { return a + b; }
}

// 2. 运行时多态（方法重写 Override）—— 实际指的是动态绑定
Animal animal = new Dog();
animal.sound();  // "Woof"（运行时决定调用 Dog 的 sound）
animal = new Cat();
animal.sound();  // "Meow"

// 3. 参数多态（泛型 Generics）
List<String> list = new ArrayList<>();
```

### 4.2 动态绑定（Dynamic Dispatch）的底层机制

```java
Animal animal = new Dog();
animal.sound();  // 如何找到 Dog.sound()？
```

```
JVM 方法调用的字节码指令：

1. invokestatic：调用静态方法（编译时确定）
2. invokespecial：调用构造器、private 方法、父类方法（编译时确定）
3. invokevirtual：调用 public/protected 方法（运行时动态分派）
4. invokeinterface：调用接口方法（运行时动态分派）
5. invokedynamic：JDK 7+，动态语言支持（Lambda 底层使用）

animal.sound() 编译为 invokevirtual

invokevirtual 的分派流程：
1. 从操作数栈顶获取对象引用（animal）
2. 通过对象引用找到堆中的对象
3. 通过对象头中的 Klass Pointer 找到方法区中的类元数据（Dog 的 vtable）
4. 在 vtable（虚方法表）中查找 sound() 方法的索引
5. 执行对应的方法体
```

**虚方法表（vtable）：**
```
Animal vtable（方法区）:
┌───────────────────────┐
│ Object.finalize()     │ ← 继承自 Object
│ Object.toString()     │
│ Object.equals()       │
│ ├─────────────────────┤
│ Animal.sound()        │ → Animal.class 中的字节码
│ Animal.eat()          │
└───────────────────────┘

Dog vtable（方法区）:
┌───────────────────────┐
│ Object.finalize()     │
│ Object.toString()     │
│ Object.equals()       │
│ ├─────────────────────┤
│ Dog.sound()           │ → Dog.class 中的字节码（重写了 Animal.sound）
│ Animal.eat()          │ → 继承自 Animal，未被重写
└───────────────────────┘

多态的性能开销：
- invokevirtual 比 invokespecial 多一次 vtable 查找
- JIT 会进行内联缓存（Inline Cache）优化
- 单态调用点（Monomorphic）：直接内联 → 无开销
- 双态/多态调用点：退化为 vtable 查找
```

### 4.3 方法重写（Override）的规则

```java
// @Override 注解：编译器验证是否真的重写了父类方法（推荐使用）

public class Parent {
    public Object method() throws IOException { return null; }
}

public class Child extends Parent {
    // 1. 方法签名必须相同
    @Override
    public Object method() throws IOException { return null; }

    // 2. 返回值类型可以更具体（协变返回类型 Covariant Return Type）
    @Override
    public String method() { return "specific"; }  // ✅ String 是 Object 的子类

    // 3. 访问权限不能更严格
    @Override
    public Object method() { return null; }  // ✅ 一致
    // @Override private Object method() { return null; }  // ❌ 更严格

    // 4. 异常类型可以更少或更具体（但不能更宽泛）
    @Override
    public String method() { return "no exception"; }  // ✅ 不抛异常也行
    // @Override public String method() throws Exception { }  // ❌ 更宽泛
}

// 静态方法不能重写（只能隐藏）
// private 方法不能重写
// final 方法不能重写
// 构造器不能重写
```

### 4.4 重载（Overload）的解析规则

```java
public class OverloadExample {
    // 重载：方法名相同，参数列表不同（个数、类型、顺序）

    public void method(int a) { }
    public void method(double a) { }
    public void method(int a, String b) { }

    // 返回类型不同不构成重载
    // public int method(int a) { return a; }  // ❌ 编译错误
}

// 重载解析的优先级（选择最具体的版本）：
public class OverloadResolution {
    public void method(Object o) { System.out.println("Object"); }
    public void method(String s) { System.out.println("String"); }
    public void method(Integer i) { System.out.println("Integer"); }

    public static void main(String[] args) {
        OverloadResolution r = new OverloadResolution();

        r.method("hello");      // String（最精确匹配）
        r.method(42);           // Integer（自动装箱后最精确）
        r.method(null);         // ❌ 编译错误！String 和 Integer 均可匹配
        // 编译器无法确定 null 的具体类型
    }
}
```

**重载解析的完整规则链：**
```
方法调用匹配流程（由编译器在编译期完成）：

1. 第一阶段：不涉及自动装箱/拆箱、不涉及可变参数
   → 寻找精确匹配

2. 第二阶段：自动类型转换（基本类型 widening）
   → int 可以匹配 long、float、double 等

3. 第三阶段：自动装箱/拆箱
   → int 匹配 Integer，Integer 匹配 int

4. 第四阶段：可变参数
   → method(int...) 匹配任意数量 int 参数

如果有多个匹配，选择最具体的版本。
```

---

## 5. 抽象类与接口

### 5.1 抽象类

```java
// 抽象类：不能实例化，可以包含抽象方法和具体方法
public abstract class Shape {
    protected String color;

    public Shape(String color) { this.color = color; }

    // 抽象方法：只有声明，没有实现
    public abstract double getArea();

    // 具体方法
    public String getColor() { return color; }

    // 可以包含字段、构造器、静态方法
    public static String getDescription() { return "A geometric shape"; }
}

public class Circle extends Shape {
    private double radius;

    public Circle(String color, double radius) {
        super(color);  // 抽象类也有构造器
        this.radius = radius;
    }

    @Override
    public double getArea() {
        return Math.PI * radius * radius;
    }
}

// 抽象类也可以没有抽象方法（但很少见）
public abstract class UtilityBase { }
```

### 5.2 接口（Interface）

```java
// JDK 7 及以前：纯抽象
public interface Flyable {
    // 字段：必须是 public static final
    int MAX_SPEED = 1000;

    // 方法：必须是 public abstract
    void fly();
}

// JDK 8+：默认方法和静态方法
public interface Flyable {
    double MAX_SPEED = 1000;  // 自动是 public static final

    void fly();  // 抽象方法

    // 默认方法（JDK 8）
    default void takeOff() {
        System.out.println("Taking off...");
        log("takeOff");
    }

    // 静态方法（JDK 8）
    static boolean isFlying(Object obj) {
        return obj instanceof Flyable;
    }
}

// JDK 9+：私有方法（辅助默认方法）
public interface Flyable {
    default void takeOff() {
        preFlightCheck();
        System.out.println("Taking off...");
    }

    default void land() {
        preFlightCheck();
        System.out.println("Landing...");
    }

    // 私有方法：辅助默认方法复用代码
    private void preFlightCheck() {
        System.out.println("Check complete");
    }

    // 私有静态方法
    private static void log(String msg) {
        System.out.println("[LOG] " + msg);
    }
}
```

### 5.3 接口的默认方法冲突

```java
public interface A {
    default void hello() { System.out.println("A"); }
}

public interface B {
    default void hello() { System.out.println("B"); }
}

// 冲突：必须手动解决
public class C implements A, B {
    // 必须重写，否则编译错误
    @Override
    public void hello() {
        A.super.hello();  // 选择 A 的默认实现
        // 或 B.super.hello();
        // 或完全重新实现
    }
}

// 接口 vs 类冲突：类优先
public interface D {
    default void hello() { System.out.println("D"); }
}

public class E {
    public void hello() { System.out.println("E"); }
}

public class F extends E implements D {
    // 这里没有冲突，类 E 的 hello 优先于接口 D 的默认方法
    public static void main(String[] args) {
        new F().hello();  // 输出 "E"
    }
}
```

### 5.4 接口与抽象类的选择

```java
// 接口 vs 抽象类对比：
//       特性        │    接口    │  抽象类
// ─────────────────┼───────────┼─────────
//  多继承           │    ✅     │    ❌
//  字段             │ static final │ 任意
//  构造器           │    ❌     │    ✅
//  方法实现         │ default/static/private │ 任意
//  访问修饰符       │ public（默认）│ 任意
//  语义             │ "capability" │ "is-a"

// 接口：强调行为/能力（Can Do）
public interface Comparable<T> { int compareTo(T o); }
public interface Runnable { void run(); }
public interface Serializable { }  // 标记接口

// 抽象类：强调层次关系（Is A）
public abstract class AbstractList<E> { }  // 骨架实现
```

**骨架实现（Skeletal Implementation）：**
```java
// 接口 + 抽象类配合使用的经典模式
// 接口定义契约
public interface List<E> {
    int size();
    E get(int index);
    void add(E e);
}

// 骨架抽象类提供默认实现
public abstract class AbstractList<E> implements List<E> {
    protected int size = 0;

    public boolean isEmpty() { return size() == 0; }  // 默认实现

    // 子类只需要实现 size()、get()、add()
}

// 具体子类
public class ArrayList<E> extends AbstractList<E> {
    // 只需要实现核心方法
}

// 这种方式称为"Template Method"模式
```

---

## 6. final 关键字

```java
// 1. final 类：不可被继承
public final class String { }  // String、Integer 等都是 final 类
// public class MyString extends String { }  // ❌

// 2. final 方法：不可被重写
public class Parent {
    public final void templateMethod() { }
}
public class Child extends Parent {
    // @Override public void templateMethod() { }  // ❌
}

// 3. final 变量：必须初始化，不可重新赋值
final int x = 10;              // 基本类型：值不可变
final StringBuilder sb = new StringBuilder();  // 引用不可变！
sb.append("hello");             // ✅ 可以修改对象内部状态
// sb = new StringBuilder();    // ❌ 引用不能重新赋值

// 4. 空白 final（blank final）：声明时不初始化，构造器中初始化
public class Person {
    private final int id;       // 空白 final

    public Person(int id) {
        this.id = id;           // 必须在构造器中赋值
    }
}

// 5. 方法参数 final
public void process(final int param) {
    // param = 10;  // ❌ 不能修改
}
```

**final 的内存语义：**
```java
// JMM（Java 内存模型）中 final 的特殊保证：
// 1. 在构造器中写 final 字段的操作，与后续将该对象引用赋值给变量之间，
//    不能重排序（保证 final 字段在对象构造完成后立即可见）
// 2. 对所有线程可见，不需要额外的同步

public class FinalFieldExample {
    final int x;
    int y;
    static FinalFieldExample instance;

    public FinalFieldExample() {
        x = 1;    // final 字段的写
        y = 2;    // 普通字段的写
    }

    // 如果 instance 不为 null，instance.x == 1 一定成立（JMM 保证）
    // 但 instance.y 可能仍然是 0（没有同步保证）
}
```

---

## 7. static 关键字

```java
public class StaticDemo {
    // 静态字段（类变量）：所有实例共享
    public static int counter = 0;

    // 实例字段：每个实例独立
    public int instanceCounter = 0;

    // 静态方法：不能访问实例成员
    public static int getCounter() {
        return counter;  // ✅ 静态方法可以访问静态字段
        // return instanceCounter;  // ❌ 不能访问实例字段
    }

    // 静态初始化块：类加载时执行（仅一次）
    static {
        counter = 100;
        System.out.println("Static init");
    }

    // 实例初始化块：每次创建对象时执行
    {
        instanceCounter = 50;
        System.out.println("Instance init");
    }
}
```

**静态成员的内存模型：**
```java
// 静态字段存储在方法区（元空间/Metaspace）中
// 不属于任何对象实例（不存储在堆中）

// 静态方法的调用：编译为 invokestatic
// 不需要对象引用，直接通过类名调用
StaticDemo.getCounter();

// 静态方法的"重写"误解：
public class Parent {
    public static void staticMethod() {
        System.out.println("Parent static");
    }
}

public class Child extends Parent {
    public static void staticMethod() {  // 这是隐藏（hide），不是重写
        System.out.println("Child static");
    }
}

Parent p = new Child();
p.staticMethod();      // "Parent static"（编译时决定，非多态！）
Child c = (Child) p;
c.staticMethod();      // "Child static"
```

---

## 8. Object 类详解

Object 是所有类的根父类，以下是其核心方法：

### 8.1 equals 与 hashCode

```java
public class User {
    private String name;
    private int age;

    // equals 的约定（自反、对称、传递、一致）
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;                          // 1. 自反性
        if (o == null || getClass() != o.getClass()) return false;  // 2. 类型检查
        User user = (User) o;
        return age == user.age && Objects.equals(name, user.name);
    }

    // hashCode 约定：
    // 1. 同一个对象多次调用返回相同整数
    // 2. equals 相等的对象，hashCode 必须相等
    // 3. equals 不相等的对象，hashCode 可以相等（但应尽量不相等，提高哈希表性能）
    @Override
    public int hashCode() {
        return Objects.hash(name, age);
    }
}
```

**equals 的常见错误：**
```java
// ❌ 错误 1：用 getClass() 与 instanceof 的选择
// getClass() 校验：要求类型完全相同（反子类化）
// instanceof 校验：允许子类实例通过（符合 Liskov 替换）

// 通常用 instanceof（可以有效处理子类比较）
@Override
public boolean equals(Object o) {
    if (this == o) return true;
    if (!(o instanceof User)) return false;  // instanceof 包含 null 检查
    User user = (User) o;
    return ...
}

// ❌ 错误 2：没有覆盖 hashCode（HashMap 中会出 bug）
// ❌ 错误 3：使用可变的字段作为 equals 的判断依据
// ❌ 错误 4：equals 中调用可能抛异常的方法
```

### 8.2 toString

```java
@Override
public String toString() {
    return "User{name='" + name + "', age=" + age + "}";
}

// 最佳实践：
// - 提供有意义的 toString（便于调试、日志）
// - 包含所有有意义的信息
// - 避免循环引用（互相引用的对象）
```

### 8.3 clone

```java
// clone 是 protected 方法，且是"浅拷贝"
// 实际不推荐使用（有缺陷的 API），推荐使用拷贝构造器或工厂方法

// 正确使用 clone 的方式：
@Override
public User clone() {
    try {
        return (User) super.clone();  // 调用 Object 的 native clone
    } catch (CloneNotSupportedException e) {
        throw new AssertionError();  // 永远不会发生
    }
}

// 更好的选择：
// 1. 拷贝构造器
public User(User other) {
    this.name = other.name;
    this.age = other.age;
}

// 2. 静态工厂方法
public static User copyOf(User other) {
    return new User(other.name, other.age);
}
```

### 8.4 finalize（已废弃）

```java
// JDK 9+ 标记为 deprecated
// 不保证执行时机，不保证执行
// 推荐使用 try-with-resources 或 Cleaner

// @Override protected void finalize() throws Throwable {
//     // 资源清理
//     super.finalize();
// }
```

---

## 9. 设计原则（SOLID）

### 9.1 单一职责原则（SRP）

```java
// ❌ 违反 SRP：一个类负责多个功能
public class Employee {
    public void calculatePay() { }    // 薪资计算
    public void saveToDatabase() { }  // 持久化
    public void printReport() { }     // 报表打印
}

// ✅ 遵循 SRP
public class Employee { public void work() { } }
public class PayCalculator { public void calculatePay(Employee e) { } }
public class EmployeeRepository { public void save(Employee e) { } }
public class ReportPrinter { public void print(Employee e) { } }
```

### 9.2 开闭原则（OCP）

```java
// 对扩展开放，对修改关闭
public abstract class Shape {
    public abstract double getArea();
}

public class Circle extends Shape {
    private double radius;
    @Override public double getArea() { return Math.PI * radius * radius; }
}

public class Rectangle extends Shape {
    private double width, height;
    @Override public double getArea() { return width * height; }
}

// 新增形状不需要修改现有代码
public class Triangle extends Shape { ... }
```

### 9.3 Liskov 替换原则（LSP）

```java
// 子类对象必须可以替换父类对象而不改变程序的正确性

// ❌ 违反 LSP
public class Rectangle {
    private int width, height;
    public void setWidth(int w) { this.width = w; }
    public void setHeight(int h) { this.height = h; }
    public int getArea() { return width * height; }
}

public class Square extends Rectangle {
    @Override
    public void setWidth(int w) {
        super.setWidth(w);
        super.setHeight(w);  // 副作用！违反 LSP
    }
}

// ✅ 或设计一个不可变的 Shape，或使用组合
```

### 9.4 接口隔离原则（ISP）

```java
// ❌ 胖接口
public interface Worker {
    void work();
    void eat();
    void sleep();
}

// ✅ 细粒度接口
public interface Workable { void work(); }
public interface Eatable { void eat(); }
public interface Sleepable { void sleep(); }

public class Robot implements Workable { }  // 不需要 eat 和 sleep
public class Human implements Workable, Eatable, Sleepable { }
```

### 9.5 依赖倒置原则（DIP）

```java
// 依赖于抽象（接口）而非具体实现

// ❌ 依赖具体实现
public class LightBulb {
    public void turnOn() { }
    public void turnOff() { }
}
public class Switch {
    private LightBulb bulb;  // 依赖具体类
    public Switch() { this.bulb = new LightBulb(); }
}

// ✅ 依赖抽象
public interface Switchable {
    void turnOn();
    void turnOff();
}
public class LightBulb implements Switchable { }
public class Switch {
    private Switchable device;
    public Switch(Switchable device) { this.device = device; }
}
```

---

## 10. 面试题

### Q1: Java 中是否有多继承？
```java
// 类不支持多继承，但接口支持
public class A extends B, C { }  // ❌ 编译错误
public interface D extends E, F { }  // ✅ 接口可以多继承
public class G implements H, I { }   // ✅ 类可以实现多个接口
```

### Q2: 什么是菱形问题（Diamond Problem）？Java 如何处理？
```java
// C++ 的菱形问题：B 和 C 都继承 A，D 多继承 B 和 C
// 导致 D 中有两份 A 的成员

// Java 类不能多继承，所以没有此问题
// Java 接口的默认方法冲突需要手动解决（见 5.3 节）
```

### Q3: 静态方法可以被重写吗？
```java
// 不能。静态方法只能被隐藏（hide）
// 调用时由编译时类型决定，不是运行时多态
```

### Q4: 构造器可以调用被重写的方法吗？
```java
public class Parent {
    public Parent() {
        overrideMe();  // ❌ 危险！
    }
    public void overrideMe() {
        System.out.println("Parent.overrideMe");
    }
}

public class Child extends Parent {
    private final String value;

    public Child(String value) {
        this.value = value;
    }

    @Override
    public void overrideMe() {
        System.out.println("Child.overrideMe: " + value);
    }
}

new Child("hello");
// 输出："Child.overrideMe: null"（不是 "hello"！）
// 原因：父类构造器执行时，子类的 value 尚未初始化（仍为 null）
// 原则：构造器中不要调用可被重写的方法
```

### Q5: equals 和 hashCode 为什么要同时覆盖？
```java
// HashMap 的运作机制：
// 1. 用 hashCode 找到桶（bucket）
// 2. 用 equals 在桶内比较

// 只覆盖 equals 不覆盖 hashCode：
// → 两个"相等"的对象 hashCode 不同 → 进入不同桶 → HashMap 认为不相等
// → 违反 hashCode 一致性约定
```

### Q6: 内部类的字节码文件命名？
```java
// Outer$Inner.class（内部类）
// Outer$1.class（匿名内部类）
// Outer$1Local.class（局部内部类）
```

---

## 11. 最佳实践

1. **优先组合而非继承**：继承破坏封装，组合更灵活
2. **覆盖 equals 必须同时覆盖 hashCode**：否则在集合中行为异常
3. **构造器不要调用可被重写的方法**：子类字段尚未初始化
4. **使用 @Override 注解**：编译期检查是否正确重写
5. **接口优于抽象类**：除非有共同的字段或构造器
6. **最小化类和成员的可见性**：信息隐藏
7. **优先使用不可变类**：线程安全，简单可靠（如 String、Integer）
8. **防御性拷贝**：避免内部状态被外部修改
9. **用标记接口（Serializable）而非标记注解**：如果用于类型判断
10. **equals 中的自反性、对称性、传递性检查**：与第三方库交互时特别小心
