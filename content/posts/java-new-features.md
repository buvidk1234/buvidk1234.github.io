+++
date = '2026-02-04T20:35:56+08:00'
draft = false
title = 'Java New Features'
tags = ['Java']
+++

## 前言

Java 作为一门历史悠久的编程语言，一直在不断演进。本文按照 **LTS（长期支持）版本** 组织，介绍 Java 8 到 Java 25 的重要新特性，帮助开发者快速了解每个版本可以用哪些功能。

> 💡 每个特性后标注的版本号表示**该特性正式可用的最低版本**。

---

## Java 8 LTS (2014) - 里程碑式更新

Java 8 是 Java 历史上最重要的版本，引入了函数式编程的核心概念。

### Lambda 表达式

**是什么？** 一种简洁的匿名函数写法，把"行为"像数据一样传递。

```java
// ❌ 传统写法
Collections.sort(list, new Comparator<String>() {
    @Override
    public int compare(String s1, String s2) {
        return s1.compareTo(s2);
    }
});

// ✅ Lambda 写法
Collections.sort(list, (s1, s2) -> s1.compareTo(s2));

// ✅ 方法引用
Collections.sort(list, String::compareTo);
```

**Lambda 语法：**
```java
(参数1, 参数2) -> { return 表达式; }  // 完整形式
(s1, s2) -> s1.compareTo(s2)          // 单行可省略 return 和大括号
s -> s.toUpperCase()                   // 单参数可省略括号
() -> System.out.println("Hello")     // 无参数
```

### Stream API

**是什么？** 对集合数据进行流水线式处理的 API。

```java
List<String> names = Arrays.asList("Alice", "Bob", "Charlie", "David");

List<String> result = names.stream()
    .filter(name -> name.length() > 3)  // 过滤
    .map(String::toUpperCase)           // 转换
    .sorted()                           // 排序
    .toList();                          // Java 16+
    // .collect(Collectors.toList());   // Java 8-15
// 结果: ["ALICE", "CHARLIE", "DAVID"]

// 并行处理
long count = names.parallelStream()
    .filter(name -> name.startsWith("A"))
    .count();
```

**常用操作：**
| 操作 | 作用 | 示例 |
|------|------|------|
| `filter` | 过滤 | `.filter(x -> x > 0)` |
| `map` | 转换 | `.map(String::toUpperCase)` |
| `sorted` | 排序 | `.sorted()` |
| `distinct` | 去重 | `.distinct()` |
| `limit` | 取前 N 个 | `.limit(5)` |
| `forEach` | 遍历 | `.forEach(System.out::println)` |

### Optional

**是什么？** 一个"可能有值也可能没值"的容器，避免 NullPointerException。

```java
// ❌ 传统写法
if (user != null && user.getAddress() != null) {
    return user.getAddress().getCity();
}
return "Unknown";

// ✅ Optional 写法
return Optional.ofNullable(user)
    .map(User::getAddress)
    .map(Address::getCity)
    .orElse("Unknown");
```

**常用方法：**
```java
Optional.of("value")              // 值不能为 null
Optional.ofNullable(maybeNull)    // 值可以为 null
Optional.empty()                  // 空 Optional

opt.orElse("default")             // 有值返回值，没值返回默认值
opt.orElseGet(() -> compute())    // 没值时才调用函数
opt.orElseThrow()                 // 没值抛异常
opt.ifPresent(v -> use(v))        // 有值才执行
```

### 新日期时间 API

**为什么需要？** 旧的 `Date` 和 `Calendar` 可变、线程不安全、API 难用。

```java
// 获取当前时间
LocalDate today = LocalDate.now();           // 只有日期
LocalTime now = LocalTime.now();             // 只有时间
LocalDateTime dateTime = LocalDateTime.now(); // 日期+时间
ZonedDateTime zoned = ZonedDateTime.now();   // 带时区

// 创建指定日期
LocalDate birthday = LocalDate.of(2000, 6, 15);

// 日期计算（返回新对象，原对象不变）
LocalDate nextWeek = today.plusWeeks(1);
LocalDate lastMonth = today.minusMonths(1);

// ⚠️ 边界情况
// 3月31日.minusMonths(1) = 2月28日（2月没有31号，调整为当月最后一天）

// 格式化
DateTimeFormatter fmt = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm");
String str = dateTime.format(fmt);
```

### 接口默认方法

**解决什么问题？** 让接口可以添加新方法而不破坏现有实现类。

```java
public interface Vehicle {
    void drive();  // 抽象方法
    
    default void honk() {  // 默认方法
        System.out.println("Beep!");
    }
    
    static Vehicle create() {  // 静态方法
        return () -> System.out.println("Driving");
    }
}
```

---

## Java 11 LTS (2018)

### Local Variable Type Inference (Java 10)

**是什么？** 让编译器自动推断局部变量类型。

```java
// ❌ 类型重复
HashMap<String, List<Integer>> map = new HashMap<String, List<Integer>>();

// ✅ 用 var
var map = new HashMap<String, List<Integer>>();
var list = List.of("a", "b", "c");

// ⚠️ 只能用于局部变量，必须有初始化值
```

### Collection Factory Methods (Java 9)

```java
// 创建不可变集合
List<String> list = List.of("a", "b", "c");
Set<String> set = Set.of("x", "y", "z");
Map<String, Integer> map = Map.of("one", 1, "two", 2);

// ⚠️ 这些集合不可变，不能 add/remove
```

### New String Methods (Java 11)

```java
"  ".isBlank();           // true（只有空白）
"  hello  ".strip();      // "hello"（去除首尾空白）
"ha".repeat(3);           // "hahaha"
"a\nb\nc".lines().count(); // 3（按行分割成 Stream）
```

### Simplified File I/O (Java 11)

```java
// 读取整个文件
String content = Files.readString(Path.of("file.txt"));

// 写入文件
Files.writeString(Path.of("out.txt"), "Hello");

// 🚀 大文件用流式读取（边读边处理，不占内存）
try (Stream<String> lines = Files.lines(Path.of("huge.txt"))) {
    lines.filter(line -> line.contains("ERROR"))
         .forEach(System.out::println);
}

// 二进制文件
byte[] data = Files.readAllBytes(Path.of("image.png"));
Files.write(Path.of("copy.png"), data);
```

### HTTP Client API (Java 11)

```java
HttpClient client = HttpClient.newBuilder()
    .connectTimeout(Duration.ofSeconds(10))
    .build();

HttpRequest request = HttpRequest.newBuilder()
    .uri(URI.create("https://api.example.com/users"))
    .header("Content-Type", "application/json")
    .POST(HttpRequest.BodyPublishers.ofString("{\"name\":\"Alice\"}"))
    .build();

// 同步请求
HttpResponse<String> response = client.send(request, 
    HttpResponse.BodyHandlers.ofString());

// 异步请求
client.sendAsync(request, HttpResponse.BodyHandlers.ofString())
    .thenApply(HttpResponse::body)
    .thenAccept(System.out::println);
```

### Java Platform Module System (Java 9)

```java
// module-info.java
module com.example.app {
    requires java.sql;
    exports com.example.api;
}
```

**实际情况：** 应用项目基本没有使用模块化，继续用传统 classpath。遇到反射报错时添加 JVM 参数即可：
```bash
java --add-opens java.base/java.lang=ALL-UNNAMED -jar app.jar
```

### Private Interface Methods (Java 9)

**是什么？** 允许在接口中定义私有方法，供默认方法调用。

**解决什么问题？** 解决接口默认方法中的代码重复问题。

```java
public interface Service {
    default void doWork() {
        init(); // 调用私有方法
        System.out.println("Working...");
    }
    
    default void doOtherWork() {
        init(); // 复用代码
        System.out.println("Other working...");
    }
    
    // ✅ 接口私有方法（Java 9+）
    private void init() {
        System.out.println("Initializing...");
    }
}
```

---

## Java 17 LTS (2021)

### Record Classes (Java 16)

**是什么？** 专门用于"只装数据"的类，自动生成构造器、getter、equals、hashCode、toString。

```java
// ❌ 传统写法：需要写很多样板代码
public class Point {
    private final int x, y;
    public Point(int x, int y) { this.x = x; this.y = y; }
    public int getX() { return x; }
    public int getY() { return y; }
    // equals, hashCode, toString...
}

// ✅ Record：一行搞定
public record Point(int x, int y) {}

// 使用
Point p = new Point(3, 4);
p.x();  // 3（注意没有 get 前缀）
p.y();  // 4
```

**添加验证和方法：**
```java
public record Point(int x, int y) {
    public Point {  // 紧凑构造器
        if (x < 0 || y < 0) throw new IllegalArgumentException();
    }
    
    public double distance() {
        return Math.sqrt(x * x + y * y);
    }
}
```

### Sealed Classes (Java 17)

**是什么？** 限制哪些类可以继承/实现某个类/接口。

```java
// 只有 Circle, Rectangle 可以继承 Shape
public sealed class Shape permits Circle, Rectangle {}

public final class Circle extends Shape {}      // final: 不能再被继承
public non-sealed class Rectangle extends Shape {} // non-sealed: 可以被任意继承
```

**配合 Record 使用：**
```java
public sealed interface Result<T> permits Success, Failure {}
public record Success<T>(T value) implements Result<T> {}
public record Failure<T>(Exception error) implements Result<T> {}

// Java 17 中通常配合 instanceof 模式匹配使用
if (result instanceof Success<String> s) {
    System.out.println("OK: " + s.value());
} else if (result instanceof Failure<String> f) {
    System.out.println("Error: " + f.error());
}

// 💡 注意：switch 的模式匹配（case Success s -> ...）在 Java 17 还是预览特性
// 直到 Java 21 才正式转正（请看后文 Java 21 章节）
```

### Pattern Matching with instanceof (Java 16)

```java
// ❌ 传统写法
if (obj instanceof String) {
    String s = (String) obj;
    System.out.println(s.length());
}

// ✅ 新写法
if (obj instanceof String s) {
    System.out.println(s.length());
}

// 配合条件
if (obj instanceof String s && s.length() > 5) {
    System.out.println(s.toUpperCase());
}
```

### Switch Expressions and Statements (Java 14)

```java
// ❌ 传统 switch：容易忘 break
String result;
switch (day) {
    case MONDAY: result = "Start"; break;
    case FRIDAY: result = "End"; break;
    default: result = "Mid";
}

// ✅ 新语法：直接返回值
String result = switch (day) {
    case MONDAY, FRIDAY -> "Weekend adjacent";
    case TUESDAY, WEDNESDAY, THURSDAY -> "Midweek";
    case SATURDAY, SUNDAY -> "Weekend";
};

// 多行用 yield
int value = switch (input) {
    case "a" -> 1;
    default -> {
        log("Unknown");
        yield -1;
    }
};
```

### Text Blocks (Java 15)

```java
// ❌ 传统：转义地狱
String json = "{\n  \"name\": \"Alice\"\n}";

// ✅ 文本块
String json = """
    {
        "name": "Alice",
        "age": 30
    }
    """;

String sql = """
    SELECT * FROM users
    WHERE status = 'active'
    ORDER BY created_at
    """;
```

---

## Java 21 LTS (2023)

### Virtual Threads (Java 21)

**是什么？** 轻量级线程，由 JVM 管理。传统线程约 1MB，虚拟线程只需几 KB。

**适合场景：** I/O 密集型（网络请求、数据库查询）。

```java
// 创建虚拟线程
Thread.ofVirtual().start(() -> {
    System.out.println("Running in virtual thread");
});

// 虚拟线程池（推荐）
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    for (int i = 0; i < 10_000; i++) {
        executor.submit(() -> {
            Thread.sleep(Duration.ofSeconds(1));
            return "done";
        });
    }
}
// 10000 个虚拟线程，只需几十 MB 内存！

// Spring Boot 3.2+ 开启虚拟线程
// application.properties:
// spring.threads.virtual.enabled=true
```

### Record Patterns (Java 21)

**是什么？** 在模式匹配中解构 Record。

```java
record Point(int x, int y) {}
record Line(Point start, Point end) {}

// 嵌套解构
if (obj instanceof Line(Point(int x1, int y1), Point(int x2, int y2))) {
    System.out.println("From " + x1 + "," + y1 + " to " + x2 + "," + y2);
}

// 在 switch 中
String describe(Object obj) {
    return switch (obj) {
        case Point(int x, int y) when x == y -> "对角线点: " + x;
        case Point(int x, int y) -> "点: (" + x + ", " + y + ")";
        default -> "其他";
    };
}
```

### Pattern Matching with switch (Java 21)

```java
String analyze(Object obj) {
    return switch (obj) {
        case null -> "空值";
        case String s when s.isEmpty() -> "空字符串";
        case String s -> "字符串: " + s;
        case Integer i when i < 0 -> "负数";
        case Integer i -> "正数: " + i;
        case int[] arr -> "数组长度: " + arr.length;
        default -> "其他";
    };
}
```

### Sequenced Collections (Java 21)

```java
// 统一的首尾操作 API
SequencedCollection<String> list = new ArrayList<>();
list.addFirst("first");
list.addLast("last");
String first = list.getFirst();
String last = list.getLast();

// 反向视图
SequencedCollection<String> reversed = list.reversed();
```

### Unnamed Variables and Patterns (Java 22)

```java
// 忽略不需要的变量
try {
    // ...
} catch (IOException _) {
    System.out.println("IO error");  // 不需要使用异常对象
}

// 在 switch 中
case Point(int x, _) -> "x = " + x;  // 不关心 y

// Lambda 中
map.forEach((_, value) -> System.out.println(value));
```

---

## Java 25 LTS (2025)

### Compact Source Files and Instance Main Methods (Java 25)

```java
// ❌ 传统写法
public class Hello {
    public static void main(String[] args) {
        System.out.println("Hello");
    }
}

// ✅ 简化写法
void main() {
    IO.println("Hello"); // Java 25 变更：必须使用 IO.println 或显式导入
}
```

### Flexible Constructor Bodies (Java 25)

```java
public class Child extends Parent {
    public Child(int value) {
        if (value < 0) throw new IllegalArgumentException();  // 可以在 super() 前执行
        super(value);
    }
}
```

### Module Import Declarations (Java 25)

**是什么？** 一次性导入整个模块的所有公开类。

**类比理解：** 以前 `import java.util.*` 只能导入一个包。现在 `import module java.base` 相当于把 `java.util.*`, `java.io.*`, `java.nio.*` 等几百个包一次性全导入了。

```java
import module java.base;  // 导入基础模块的所有类

void main() {
    // List, Map, Path, Files 等都不用单独 import 了
    List<String> list = List.of("a", "b");
    Path path = Path.of("file.txt");
}
```

### Scoped Values (Java 25)

**是什么？** **ThreadLocal 的现代替代品**。它是一种在方法调用链中隐式传递数据的机制。

**为什么要替代 ThreadLocal？**
- **ThreadLocal**：像全局变量，谁都能改（可变），生命周期长（容易内存泄漏），内存开销大。
- **Scoped Values**：像方法参数，**不可变**（安全），**生命周期严格绑定代码块**（出了块就失效），对虚拟线程特别优化（省内存）。

```java
// 1. 定义一个"隐式参数"（全局常量）
static final ScopedValue<User> CURRENT_USER = ScopedValue.newInstance();

void handleRequest(Request req) {
    User user = authenticate(req);
    
    // 2. 绑定值并执行代码块
    // 在 runWhere 的花括号内，CURRENT_USER 有值；出了花括号，它就失效了
    ScopedValue.runWhere(CURRENT_USER, user, () -> {
        processOrder();  // 不需要显式传递 user 参数
    });
}

// 3. 在深层方法中获取
void processOrder() {
    // 就像从空气中抓到了这个参数
    User user = CURRENT_USER.get();
    System.out.println("Processing for " + user.name());
}
```

### Structured Concurrency (Java 25)

**是什么？** 结构化并发。把一组并行任务看作一个整体，要么一起成功，要么一起失败。

**解决什么问题？**
以前多线程通过 `ExecutorService` 提交任务后，不仅难以管理（任务可能还在跑，主线程已经异常退出了），而且如果有任务失败，其他兄弟任务还在浪费资源继续跑。

**Structured Concurrency 就像给线程加了 try-with-resources：**
1. **自动等待**：代码块结束前，必须等所有子任务完成（不用手动 latch.await）。
2. **短路机制**：如果一个任务失败，自动取消其他正在跑的任务（省资源）。
3. **异常传播**：子任务的异常也会正确抛出给主线程。

```java
Response fetchData() throws Exception {
    // 开启一个"任务作用域"
    try (var scope = StructuredTaskScope.open()) {
        
        // 衍生出两个并行任务
        SubTask<User> userTask = scope.fork(() -> fetchUser());
        SubTask<Order> orderTask = scope.fork(() -> fetchOrders());
        
        // 等待所有任务完成（或者其中一个失败导致短路）
        scope.join(); 
        
        // 组装结果
        return new Response(userTask.get(), orderTask.get());
    } 
    // 离开 try 块时，保证所有线程都已结束（不会有"泄漏"的僵尸线程）
}
```

### 性能优化（无需改代码）

- **紧凑对象头**：对象内存减少 10-20%（`-XX:+UseCompactObjectHeaders`）
- **AOT 预热优化**：启动时间减少 30-50%

---

## 总结

| 版本 | 必须掌握的特性 |
|------|----------------|
| Java 8 | Lambda, Stream, Optional, 新日期 API |
| Java 11 | var, 集合工厂, HTTP Client, Files 简化 |
| Java 17 | Record, Sealed Classes, instanceof 模式匹配, 文本块 |
| Java 21 | Virtual Threads, Switch 模式匹配, Sequenced Collections |
| Java 25 | 简化入口点, Scoped Values, Structured Concurrency |

### 版本选择

| 场景 | 推荐版本 |
|------|----------|
| 新项目 | Java 21 或 25 |
| 企业稳定项目 | Java 21 |
| 遗留项目升级 | 17 → 21 → 25 |

---

## 参考资料

- [Oracle Java Documentation](https://docs.oracle.com/en/java/)
- [Java Language Changes by Release](https://docs.oracle.com/en/java/javase/25/language/java-language-changes-release.html)
