# Java 日期时间 API

## 📖 本章导读

### 学习目标
通过本章学习,你将掌握:
- ✅ 旧API(Date/Calendar)的设计缺陷和线程安全问题
- ✅ LocalDate/LocalTime/LocalDateTime的核心用法
- ✅ Instant时间戳和Duration/Period时间量计算
- ✅ ZonedDateTime时区处理和夏令时自动调整
- ✅ DateTimeFormatter格式化与解析(线程安全)
- ✅ TemporalAdjuster复杂日期调整
- ✅ 新旧API的互操作转换

### 核心概念
**java.time包(JDK 8+)**是全新的日期时间API(JSR 310),借鉴Joda-Time的优秀设计,解决了旧API的所有痛点。所有类都是**不可变且线程安全**的,采用清晰的领域驱动设计,分为日期、时间、时刻、时区、时间量等独立概念。

### 知识地图
```
日期时间API体系
├── 旧API的问题(JDK 1.0-1.1)
│   ├── Date:年份从1900开始,月份从0开始,可变性
│   ├── Calendar:API繁琐,线程不安全
│   └── SimpleDateFormat:多线程共享导致诡异结果
├── 新API核心类(java.time)
│   ├── 日期时间类(ISO标准,不可变,线程安全)
│   │   ├── LocalDate:纯日期(2024-01-15)
│   │   ├── LocalTime:纯时间(14:30:00.123)
│   │   ├── LocalDateTime:日期+时间(2024-01-15T14:30:00)
│   │   └── Instant:时间戳(纳秒精度,UTC时刻)
│   ├── 时区相关类
│   │   ├── ZoneId:时区标识("Asia/Shanghai")
│   │   ├── ZoneOffset:时区偏移(+08:00)
│   │   ├── ZonedDateTime:带时区的日期时间
│   │   └── OffsetDateTime:带偏移的日期时间
│   └── 时间量类
│       ├── Duration:时间量(秒/纳秒精度)
│       └── Period:日期量(年/月/日)
├── LocalDate用法
│   ├── 创建:now() / of() / parse()
│   ├── 计算:plusDays() / minusMonths() / with()
│   ├── 获取字段:getYear() / getMonthValue() / getDayOfWeek()
│   ├── 判断:isBefore() / isAfter() / isEqual() / isLeapYear()
│   └── TemporalAdjusters:firstDayOfMonth() / next(DayOfWeek.MONDAY)
├── LocalTime用法
│   ├── 创建:now() / of() / parse()
│   ├── 计算:plusHours() / minusMinutes() / truncatedTo()
│   └── 获取字段:getHour() / getMinute() / getSecond() / getNano()
├── LocalDateTime用法
│   ├── 创建:now() / of() / date.atTime() / time.atDate()
│   ├── 转换:toLocalDate() / toLocalTime() / toInstant()
│   └── 综合计算:plusDays().plusHours().minusMinutes()
├── Instant用法
│   ├── 创建:now() / ofEpochSecond() / ofEpochMilli()
│   ├── 转换:getEpochSecond() / toEpochMilli()
│   └── 计算:plusSeconds() / minus() / Duration.between()
├── Duration与Period
│   ├── Duration:ofSeconds() / ofHours() / between()
│   ├── Period:ofMonths() / between(birthDate, today)
│   └── ChronoUnit:DAYS.between()更准确
├── ZonedDateTime与时区
│   ├── 创建:of() / now(zoneId) / instant.atZone()
│   ├── 时区转换:withZoneSameInstant(ZoneId.of("Europe/Paris"))
│   └── 夏令时处理:自动调整不存在或重复的时间
├── 格式化与解析
│   ├── DateTimeFormatter(不可变,线程安全)
│   │   ├── 预定义:ISO_LOCAL_DATE / ISO_LOCAL_DATE_TIME
│   │   ├── 自定义:ofPattern("yyyy-MM-dd HH:mm:ss")
│   │   ├── 格式化:date.format(formatter)
│   │   └── 解析:LocalDate.parse("2024-01-15", formatter)
│   └── 常用模式:y年 M月 d日 H时 m分 s秒 E星期 a上下午
├── TemporalAdjuster
│   ├── 内置:firstDayOfMonth() / lastDayOfMonth()
│   ├── 星期调整:next() / previous() / firstInMonth()
│   └── 自定义:实现TemporalAdjuster接口
├── 性能测试
│   ├── Instant + Duration:StopWatch模式
│   └── System.nanoTime():高精度计时
└── 新旧API互操作
    ├── Date ↔ Instant:date.toInstant() / Date.from(instant)
    └── Calendar ↔ ZonedDateTime:calendar.toInstant().atZone()
```

### 常见误区
❌ **误区1**: SimpleDateFormat在多线程环境下安全
✅ **真相**: SimpleDateFormat内部维护Calendar对象,多线程共享会导致数据竞争,必须使用ThreadLocal或DateTimeFormatter

❌ **误区2**: Period.getDays()返回两个日期的总天数差
✅ **真相**: Period.getDays()只是天数部分的差值,计算总天数应用ChronoUnit.DAYS.between()

❌ **误区3**: Duration.ofDays(1)和Period.ofDays(1)一样
✅ **真相**: Duration一定是24小时,Period受夏令时影响可能不是24小时

❌ **误区4**: LocalDate包含时区信息
✅ **真相**: LocalDate是纯日期,无时区概念;需要时区用ZonedDateTime

❌ **误区5**: 月份加减会保持月末日期
✅ **真相**: 1月31日加1个月会自动调整为2月28/29日,JVM智能处理月末边界

### 实战建议
1. **优先使用java.time包**: 全面替代Date和Calendar,代码更清晰安全
2. **DateTimeFormatter声明为static final**: 不可变对象,线程安全可共享
3. **Duration用于时间度量,Period用于日期计算**: 区分精确时间和日历日
4. **时区处理使用ZonedDateTime**: 不要手动加减偏移,让JVM处理夏令时
5. **Instant表示时间戳**: 时间线上的一点,存储到数据库用Instant
6. **ChronoUnit.DAYS.between()计算天数差**: 比Period更准确直观
7. **避免使用Date和Calendar的弃用方法**: 如setYear()、getMonth()
8. **格式化模式注意M和m的区别**: M是月,m是分,大小写敏感
9. **TemporalAdjusters处理复杂日期**: 如"每月第一个周一"等场景
10. **新旧API转换用toInstant()**: Date和Calendar都提供了转Instant的方法

## 1. 日期时间 API 演进

| JDK 版本 | API |
|:---|:---|
| JDK 1.0 | `java.util.Date`（设计缺陷，大部分方法已弃用） |
| JDK 1.1 | `java.util.Calendar` + `DateFormat`（线程不安全，API 繁琐） |
| JDK 8+ | `java.time` 包（JSR 310，借鉴 Joda-Time，推荐使用） |

**旧 API 的问题：**
```java
// 1. Date 的设计缺陷
Date date = new Date(2024, 0, 1);  // 年从 1900 开始，月从 0 开始！
// 实际是 3924 年 1 月？（不要这样用！）

// 2. 可变性
Date date = new Date();
date.setYear(100);  // 可以修改内部状态（不可变更安全）

// 3. 线程不安全
SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd");
// 多个线程共享 sdf.parse() 会导致诡异结果

// 4. 月份从 0 开始，年份从 1900 开始
Calendar cal = Calendar.getInstance();
cal.set(2024, Calendar.JANUARY, 1);  // Calendar.JANUARY = 0
```

---

## 2. 新日期时间 API（java.time）

### 2.1 核心类

```
ISO 标准日期时间类（不可变、线程安全）：

日期：     LocalDate          (2024-01-15)
时间：     LocalTime          (14:30:00.123)
日期时间： LocalDateTime      (2024-01-15T14:30:00)
时刻：     Instant            (时间戳，纳秒精度)
时区日期： ZonedDateTime      (2024-01-15T14:30:00+08:00[Asia/Shanghai])
时段：     Duration           (时间量：秒/纳秒)
周期：     Period             (日期量：年/月/日)
```

### 2.2 LocalDate（日期）

```java
// 创建
LocalDate today = LocalDate.now();
LocalDate date = LocalDate.of(2024, 1, 15);
LocalDate parsed = LocalDate.parse("2024-01-15");  // 默认 ISO 格式

// 日期计算
LocalDate tomorrow = today.plusDays(1);
LocalDate nextWeek = today.plusWeeks(1);
LocalDate nextMonth = today.plusMonths(1);
LocalDate lastYear = today.minusYears(1);
LocalDate nextMonday = today.with(TemporalAdjusters.next(DayOfWeek.MONDAY));

// 获取字段
int year = date.getYear();
int month = date.getMonthValue();    // 1~12
Month monthEnum = date.getMonth();   // Month.JANUARY
int day = date.getDayOfMonth();
DayOfWeek dow = date.getDayOfWeek(); // DayOfWeek.WEDNESDAY
int doy = date.getDayOfYear();
boolean leap = date.isLeapYear();    // 是否闰年

// 判断与比较
date.isBefore(today);
date.isAfter(today);
date.isEqual(today);
date.compareTo(today);

// 月份加减（跨年安全）
LocalDate jan31 = LocalDate.of(2024, 1, 31);
LocalDate feb28 = jan31.plusMonths(1);  // 2024-02-29（2024 是闰年）
LocalDate mar1 = jan31.plusMonths(1).plusDays(1);  // 2024-03-01
```

### 2.3 LocalTime（时间）

```java
// 创建
LocalTime now = LocalTime.now();
LocalTime time = LocalTime.of(14, 30, 15, 123_000_000);  // 14:30:15.123
LocalTime parsed = LocalTime.parse("14:30:15");

// 时间计算
LocalTime later = time.plusHours(2);
LocalTime earlier = time.minusMinutes(30);
LocalTime truncated = time.truncatedTo(ChronoUnit.MINUTES);  // 截断到分钟

// 获取字段
int hour = time.getHour();
int minute = time.getMinute();
int second = time.getSecond();
int nano = time.getNano();  // 纳秒

// 判断
time.isBefore(LocalTime.NOON);       // 是否在中午之前
time.isAfter(LocalTime.MIDNIGHT);
```

### 2.4 LocalDateTime（日期+时间）

```java
// 创建
LocalDateTime now = LocalDateTime.now();
LocalDateTime dt = LocalDateTime.of(2024, 1, 15, 14, 30);
LocalDateTime fromDate = date.atTime(14, 30);   // LocalDate + 时间
LocalDateTime fromTime = time.atDate(date);      // LocalTime + 日期
LocalDateTime parsed = LocalDateTime.parse("2024-01-15T14:30:00");

// 转换
LocalDate datePart = dt.toLocalDate();
LocalTime timePart = dt.toLocalTime();
Instant instant = dt.toInstant(ZoneOffset.UTC);  // 转时间戳

// 日期时间计算（综合了 LocalDate 和 LocalTime 的方法）
dt.plusDays(1).plusHours(2).minusMinutes(30);
```

### 2.5 Instant（时间戳）

```java
// 时刻：从 1970-01-01T00:00:00Z 开始的纳秒计数

Instant now = Instant.now();             // 当前 UTC 时刻
Instant epoch = Instant.EPOCH;           // 1970-01-01T00:00:00Z
Instant fromEpoch = Instant.ofEpochSecond(1_700_000_000);  // 从秒
Instant fromEpochMs = Instant.ofEpochMilli(System.currentTimeMillis());

// 转换
long epochSecond = now.getEpochSecond();  // 秒
int nano = now.getNano();                 // 纳秒部分
long epochMs = now.toEpochMilli();        // 毫秒（可能丢失纳秒）

// 计算
Instant later = now.plusSeconds(60);
Instant earlier = now.minus(10, ChronoUnit.MINUTES);
Duration diff = Duration.between(earlier, now);
System.out.println(diff.getSeconds());  // 600
```

### 2.6 Duration 与 Period

```java
// Duration：时间量（秒/纳秒精度）
Duration fiveSeconds = Duration.ofSeconds(5);
Duration twoHours = Duration.ofHours(2);
Duration oneDay = Duration.ofDays(1);      // 24 小时（与日历日不同！）

Duration between = Duration.between(startTime, endTime);
System.out.println(between.toMinutes());   // 转换为分钟

// Period：日期量（年/月/日）
Period twoMonths = Period.ofMonths(2);
Period oneYear = Period.of(1, 6, 15);      // 1 年 6 月 15 日

Period age = Period.between(birthDate, today);
System.out.println(age.getYears() + "岁" + age.getMonths() + "月");

// 两者区别：
// Duration：基于秒/纳秒，精确时间差
// Period：基于日历日期（年/月/日），受时区和夏令时影响

// 使用 ChronoUnit 比较
long daysBetween = ChronoUnit.DAYS.between(startDate, endDate);
long hoursBetween = ChronoUnit.HOURS.between(start, end);
```

### 2.7 ZonedDateTime 与时区

```java
// 时区相关类：
// ZoneId：时区标识（如 "Asia/Shanghai"）
// ZoneOffset：时区偏移（如 +08:00）
// ZonedDateTime：带时区的日期时间
// OffsetDateTime：带偏移的日期时间

// 获取可用时区
Set<String> zones = ZoneId.getAvailableZoneIds();
// System.out.println(zones.size());  // 600+ 个时区

// 常用时区
ZoneId shanghai = ZoneId.of("Asia/Shanghai");
ZoneId utc = ZoneId.of("UTC");
ZoneId systemZone = ZoneId.systemDefault();

// 创建 ZonedDateTime
ZonedDateTime zdt = ZonedDateTime.of(2024, 1, 15, 14, 30, 0, 0, shanghai);
ZonedDateTime nowWithZone = ZonedDateTime.now(shanghai);
ZonedDateTime fromInstant = Instant.now().atZone(shanghai);

// 时区转换
ZonedDateTime parisTime = zdt.withZoneSameInstant(ZoneId.of("Europe/Paris"));
// 2024-01-15T07:30+01:00[Europe/Paris]（上海 14:30 = 巴黎 7:30）

// 夏令时处理
ZoneId usEastern = ZoneId.of("America/New_York");
ZonedDateTime dstDate = ZonedDateTime.of(2024, 3, 10, 2, 30, 0, 0, usEastern);
// 2024-03-10 美国进入夏令时，2:00 AM 跳到 3:00 AM
// 2:30 AM 这个时间不存在！
// ZonedDateTime 会自动调整到正常时间
```

---

## 3. 格式化与解析

### 3.1 DateTimeFormatter

```java
// 预定义的格式化器
DateTimeFormatter isoDate = DateTimeFormatter.ISO_LOCAL_DATE;       // 2024-01-15
DateTimeFormatter isoDateTime = DateTimeFormatter.ISO_LOCAL_DATE_TIME;  // 2024-01-15T14:30:00
DateTimeFormatter isoInstant = DateTimeFormatter.ISO_INSTANT;  // 2024-01-15T06:30:00Z

// 自定义格式
DateTimeFormatter formatter = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");
DateTimeFormatter chinese = DateTimeFormatter.ofPattern("yyyy年MM月dd日 EEEE");

// 格式化
LocalDateTime now = LocalDateTime.now();
String formatted = now.format(formatter);                     // "2024-01-15 14:30:00"
String cnFormatted = now.format(chinese);                     // "2024年01月15日 星期一"

// 解析
LocalDateTime parsed = LocalDateTime.parse("2024-01-15 14:30:00", formatter);
LocalDate date = LocalDate.parse("2024/01/15",
    DateTimeFormatter.ofPattern("yyyy/MM/dd"));

// 线程安全！DateTimeFormatter 是不可变的，可以定义为 static final
public static final DateTimeFormatter FORMATTER =
    DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");
```

**常用格式模式：**
```
y    = year（年）         M = month（月）     d = day（日）
H    = hour（小时0-23）   m = minute（分）    s = second（秒）
S    = fraction of second（毫秒/纳秒）
E    = day of week（星期几）
a    = AM/PM 标记
z    = time zone name（时区名）
XXX  = time zone offset（时区偏移 ±08:00）

示例：
"yyyy-MM-dd"              → "2024-01-15"
"yyyy/MM/dd HH:mm:ss"     → "2024/01/15 14:30:00"
"yyyy-MM-dd'T'HH:mm:ss'Z'"  → "2024-01-15T06:30:00Z"
"yyyy年M月d日 E a hh:mm"    → "2024年1月15日 星期一 下午 02:30"
```

### 3.2 旧 API 与新 API 的互操作

```java
// Date ↔ Instant
Date date = new Date();
Instant instant = date.toInstant();
Date fromInstant = Date.from(instant);

// Calendar ↔ ZonedDateTime
Calendar calendar = Calendar.getInstance();
ZonedDateTime zdt = calendar.toInstant().atZone(ZoneId.systemDefault());
Calendar fromZdt = GregorianCalendar.from(zdt);

// SimpleDateFormat ↔ DateTimeFormatter 无直接转换
// 需要手动匹配模式
```

---

## 4. TemporalAdjuster

```java
// 用于复杂的日期调整
import java.time.temporal.TemporalAdjusters;

LocalDate date = LocalDate.of(2024, 1, 15);  // 周一

// 常用调整器
date.with(TemporalAdjusters.firstDayOfMonth());        // 2024-01-01
date.with(TemporalAdjusters.lastDayOfMonth());          // 2024-01-31
date.with(TemporalAdjusters.next(DayOfWeek.MONDAY));   // 下周一
date.with(TemporalAdjusters.previous(DayOfWeek.MONDAY)); // 上周一
date.with(TemporalAdjusters.firstInMonth(DayOfWeek.MONDAY));  // 当月的第一个周一
date.with(TemporalAdjusters.lastInMonth(DayOfWeek.SUNDAY));   // 当月的最后一个周日
date.with(TemporalAdjusters.nextOrSame(DayOfWeek.FRIDAY));    // 下一个周五或当天

// 自定义调整器
TemporalAdjuster nextPayDay = temporal -> {
    LocalDate d = LocalDate.from(temporal);
    if (d.getDayOfMonth() < 15) {
        return d.withDayOfMonth(15);
    } else {
        return d.plusMonths(1).withDayOfMonth(1);
    }
};

LocalDate payDay = date.with(nextPayDay);
```

---

## 5. 时间度量与性能测试

```java
// 使用 Instant + Duration 做性能测试
public class StopWatch {
    private Instant start;

    public void start() {
        start = Instant.now();
    }

    public Duration stop() {
        return Duration.between(start, Instant.now());
    }
}

// 使用
StopWatch sw = new StopWatch();
sw.start();
doSomething();
Duration elapsed = sw.stop();
System.out.printf("耗时: %d ms%n", elapsed.toMillis());
System.out.printf("耗时: %.3f s%n", elapsed.toMillis() / 1000.0);

// 或更直接
long start = System.nanoTime();
doSomething();
long elapsed = System.nanoTime() - start;
System.out.println("耗时: " + elapsed / 1_000_000 + "ms");
```

---

## 6. 面试题

### Q1: SimpleDateFormat 为什么线程不安全？如何解决？
```java
// SimpleDateFormat 内部维护 Calendar 对象，多个线程共享会导致数据竞争
// 解决方案：
// 1. 每次使用时创建新实例（性能差）
// 2. 加锁同步（ThreadLocal 方式也不错）
// 3. 使用 DateTimeFormatter（线程安全，推荐）
```

### Q2: LocalDate 和 Date 的区别？
```java
// LocalDate：只包含日期，不可变，线程安全
// Date：包含日期+时间，可变，大部分方法弃用
// 
// LocalDate 没有时区概念（纯日期）
// Date 内部是时间戳（有时区含义）
```

### Q3: Period 和 Duration 的区别？
```java
// Period：基于日期（年/月/日），处理日历日
// Duration：基于时间（秒/纳秒），处理精确时间
//
// Period.ofDays(1) 不一定是 24 小时（夏令时）
// Duration.ofDays(1) 一定是 24 小时
```

### Q4: 时区转换中的夏令时如何处理？
```java
// ZonedDateTime 自动处理夏令时
// 在夏令时转换时（春季提前、秋季回退），ZonedDateTime 会调整
// - 不存在的日期时间自动前进
// - 重复的日期时间（秋季回退）取第一次出现
```

### Q5: 如何计算两个日期之间的天数？
```java
LocalDate start = LocalDate.of(2024, 1, 1);
LocalDate end = LocalDate.of(2024, 12, 31);

// 方式 1：ChronoUnit
long days = ChronoUnit.DAYS.between(start, end);

// 方式 2：Period
Period period = Period.between(start, end);
// period.getDays() 只是天数部分的差值，不是总天数！
// 需要：period.getYears() * 365 + period.getMonths() * 30 + period.getDays()（不准确）

// 方式 3：until
long days2 = start.until(end, ChronoUnit.DAYS);
```

---

## 7. 最佳实践

1. **优先使用 `java.time` 包**：替代 `Date` 和 `Calendar`
2. **使用 `DateTimeFormatter`**：替代 `SimpleDateFormat`（线程安全）
3. **`DateTimeFormatter` 声明为 `static final`**：不可变，安全共享
4. **`Duration` 用于时间度量，`Period` 用于日期计算**
5. **时区处理使用 `ZonedDateTime`**：而非手动加减偏移
6. **使用 `Instant` 表示时间戳**：时间线上的一点，无时区
7. **`LocalDate` 使用 `ChronoUnit.DAYS.between()` 计算天数差**
8. **避免使用 `Date` 和 `Calendar` 的弃用方法**
9. **格式化模式中的 M（月）和 m（分）不要混淆**
10. **使用 `TemporalAdjusters` 处理复杂日期调整**
