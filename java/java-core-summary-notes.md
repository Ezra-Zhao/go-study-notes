# Java 基础核心学习笔记

> 资料定位：本笔记依据一份 70 多页的 Java SE 核心知识总结的章节结构整理而成，适合已经学过 Java 语法的同学快速复习、查漏补缺。正文全部用自己的语言重写，代码示例均为手写标准写法。

---

## 一、Java 语言概览

Java 是一门面向对象、跨平台的编程语言。"跨平台"靠的是虚拟机这层中间层：我们写的源代码先被编译成字节码（.class 文件），再由各操作系统的 JVM（Java 虚拟机）执行。一份字节码在 Windows、Linux、macOS 上都能跑，这就是所谓的"一次编写，到处运行"。

环境里两个缩写要分清：

- **JDK**（开发工具包）：包含编译器 javac、打包工具等，是开发者用的全套工具。
- **JRE**（运行环境）：只有运行字节码需要的类库和虚拟机，只能运行程序，不能编译。

安装 JDK 后需要配置环境变量，主要是把 `bin` 目录放进 PATH，让命令行能找到 `javac` 和 `java`。写好第一个 `HelloWorld.java` 后，用 `javac` 编译、`java` 运行，记住运行时不带 `.class` 后缀。

## 二、基本语法

### 数据类型

Java 是强类型语言，变量用之前必须声明类型。基本类型共 8 种：

| 类别 | 类型 | 说明 |
|------|------|------|
| 整数 | byte、short、int、long | 默认 int，长整型字面量加后缀 L |
| 浮点 | float、double | 默认 double，float 字面量加后缀 f |
| 字符 | char | 单个字符，用单引号，Unicode 编码 |
| 布尔 | boolean | 只有 true / false 两个值 |

除此之外一切皆对象。`String` 是引用类型，不可变——一旦创建，内容就不能改，任何"修改"操作实际上是产生了一个新对象。

```java
String s = "hello";
s = s + " world";   // 不是改旧对象，而是生成新对象再重新指向
int n = 100L;       // 错误：long 字面量不能直接赋给 int
```

### 运算符

算术、关系、逻辑运算符和 C 语言大体一致，几个 Java 特有的细节：

- `+` 既做加法也做字符串拼接，拼接时其他类型会被转成字符串。
- 移位运算符 `<<`、`>>`、`>>>`：`>>` 是算术右移（高位补符号位），`>>>` 是逻辑右移（高位补 0）。
- 短路逻辑 `&&`、`||`：左边的结果能确定整体结果时，右边不再求值；`&`、`|` 作为逻辑运算符时则两边都算。

### 流程控制

`if-else`、`switch`、`while`、`do-while`、`for` 与常见语言一致。Java 7 起 switch 支持 `String`（Java 5 引入的是枚举支持），注意 case 分支别忘了 `break`，否则会贯穿执行。

```java
for (String arg : args) {       // 增强 for：遍历数组或集合最简洁的写法
    System.out.println(arg);
}
```

`break` 跳出本层循环，`continue` 结束本次迭代。多层循环需要时可给循环加标签：`break outer;` 能直接跳出外层循环。

## 三、面向对象

### 类、属性与方法

类是对象的模板。属性描述状态，方法描述行为。一个类的成员包含：字段（属性）、构造器、普通方法。

```java
public class Person {
    private String name;                    // 属性：习惯用 private 封装
    public Person(String name) {            // 构造器：名字与类同名，没有返回类型
        this.name = name;
    }
    public String getName() {               // 方法：封装后对外提供访问
        return this.name;
    }
}
```

### 方法重载与重写

- **重载（overload）**：同一个类里，方法名相同、参数列表不同。编译器按参数匹配调用哪个。跟返回值类型无关。
- **重写（override）**：子类重新实现父类的方法，要求方法签名一致，访问权限不能更窄，也不能抛出更宽的受检异常。重写是实现多态的基础。

### 初始化顺序

对象创建时初始化顺序是固定的：先父类静态块 → 子类静态块 → 父类实例块和构造器 → 子类实例块和构造器。静态部分只在类加载时跑一次。

数组初始化有三种常见写法：

```java
int[] a = new int[5];           // 长度固定，元素默认 0
int[] b = new int[]{1, 2, 3};   // 显式初始化
int[] c = {1, 2, 3};            // 简化写法，只在声明时可用
```

### this 和 super

`this` 指当前对象：构造器里用 `this(name)` 调用本类另一个构造器，区分同名字段与参数。`super` 指父类部分：`super()` 调用父类构造器（必须写在构造器第一行），`super.method()` 调用父类被重写的方法。

### 访问权限

四种，从严到宽：`private`（本类）→ 缺省（本包）→ `protected`（本包 + 子类）→ `public`（所有）。设计原则是能小不大，字段优先 private。

## 四、继承、多态、组合

**继承**是"is-a"关系：`class Dog extends Animal`。子类获得父类非私有成员，可以重写行为。Java 只支持单继承，接口可以多实现，这避免了多继承的菱形问题。

**多态**是面向对象的核心：父类引用指向子类对象，调用被重写的方法时，实际执行的是子类的版本（动态绑定）。多态让代码对扩展开放：新增子类不影响使用父类引用的老代码。

```java
Animal a = new Dog();   // 向上转型：把子类对象当父类用，安全且隐式
a.speak();              // 运行时执行 Dog 的 speak()，这就是多态
```

**组合**是"has-a"关系：在类里持有另一个类的对象来复用功能。能组合就不继承——继承是强耦合，组合更灵活。

**向上转型**：子类引用赋给父类类型变量，自动完成，只能看到父类声明的成员。**向下转型**需要显式强制转换，且必须先用 `instanceof` 检查，否则抛 `ClassCastException`。

```java
if (a instanceof Dog) {
    Dog d = (Dog) a;    // 向下转型前先检查
    d.fetch();
}
```

**代理**：一个对象持有另一个对象的引用并把调用转发过去，典型应用是动态代理，在不改原类代码的情况下加日志、事务等横切逻辑。

## 五、static 与 final

`static` 属于类而不是某个对象：静态字段所有对象共享，静态方法只能访问静态成员（没有 this）。工具类的方法常做成 static，直接用类名调用。

`final` 有三层含义：修饰变量表示常量（必须初始化，引用不可改指向）；修饰方法表示不能被重写；修饰类表示不能被继承（如 String）。

## 六、抽象类与接口

- **抽象类**：用 abstract 修饰，可以有抽象方法也可以有具体实现，用来抽取子类的公共部分。子类用 extends 继承，一个类只能继承一个抽象类。
- **接口**：用 interface 定义，Java 8 起可以有 default 方法和 static 方法。接口定义的是"能做什么"的契约，类用 implements 实现，一个类可以实现多个接口。

选择原则：如果要定义行为规范、允许多重实现，用接口；如果要在层级中共享代码和状态，用抽象类。实际开发中接口用得更多，因为它解耦更彻底。

## 七、异常处理

异常体系的根是 `Throwable`，分两大分支：

- **Error**：系统级严重错误，如内存溢出，程序一般不处理。
- **Exception**：可处理的异常。又分受检异常（编译器强制处理，如 IOException）和运行时异常 RuntimeException（如空指针、数组越界，编译器不强制）。

```java
try {
    readFile(path);
} catch (FileNotFoundException e) {
    // 针对性处理
} catch (IOException e) {
    // 兜底处理
} finally {
    // 一定执行：关流、释放资源
}
```

`throw` 是在代码里主动抛异常，`throws` 是在方法签名上声明"这个方法可能抛什么异常，调用者你看着办"。Java 7 起 try-with-resources 能自动关闭实现了 AutoCloseable 的资源，写 IO 代码时优先用它。

异常设计原则：不要吞异常（catch 里空着什么都不做），至少记日志；不要用异常做正常流程控制；自定义业务异常时继承 RuntimeException 可以减少调用方的模板代码。

## 八、内部类

内部类是定义在另一个类内部的类，分四种：成员内部类（可访问外部类所有成员，持有外部类引用）、静态内部类（不持有外部引用，推荐用这个）、局部内部类（定义在方法里）、匿名内部类（没有名字，常用于一次性实现接口）。匿名内部类在事件监听、回调里很常见，Java 8 的 Lambda 表达式让这类写法更简洁。

## 九、集合框架

集合是 Java 最常用的类库。核心接口和实现：

- **List**（有序、可重复）：`ArrayList` 底层是数组，随机访问快，增删中间元素慢；`LinkedList` 底层是链表，增删快但随机访问慢。大多数场景用 ArrayList。
- **Set**（无序、不重复）：`HashSet` 底层是哈希表，O(1) 查找；`TreeSet` 底层是红黑树，元素自动排序。
- **Map**（键值对）：`HashMap` 最常用，键不允许重复；`TreeMap` 按键排序；`LinkedHashMap` 保持插入顺序。注意 HashMap 不是线程安全的，多线程用 ConcurrentHashMap。

```java
List<String> list = new ArrayList<>();
list.add("a");
Map<String, Integer> map = new HashMap<>();
map.put("key", 1);
```

`Collections` 工具类提供排序、二分查找、同步包装等静态方法。遍历优先用增强 for 或迭代器，在遍历中删除元素必须用 `Iterator.remove()`，直接调 `list.remove()` 会报 `ConcurrentModificationException`。

## 十、泛型

泛型让类和方法可以对多种类型工作，同时保留编译期类型检查，省掉强制转型：

```java
public class Box<T> {           // 泛型类：T 是类型参数
    private T value;
    public void set(T value) { this.value = value; }
    public T get() { return value; }
}
Box<String> box = new Box<>();  // 菱形语法，编译器推断类型
```

通配符 `?` 表示未知类型：`List<? extends Number>` 只能读不能写（生产者），`List<? super Integer>` 可以写入（消费者）。记住口诀"PECS：Producer Extends，Consumer Super"。

泛型的类型信息在编译后会被擦除，所以不能 `new T()`，也不能用 `instanceof` 判断泛型类型。

## 十一、反射

反射让程序在运行时检查和操作类：从类名拿到 Class 对象，再获取字段、方法、构造器并调用。框架（Spring、MyBatis）大量用反射做依赖注入和映射。

```java
Class<?> clazz = Class.forName("com.example.Person");
Object obj = clazz.getDeclaredConstructor().newInstance();
Method m = clazz.getMethod("getName");
String name = (String) m.invoke(obj);
```

反射的代价是性能损耗和破坏封装（能访问私有成员），业务代码里能不用就不用。`ClassLoader` 负责把字节码加载进 JVM，理解双亲委派模型有助于排查"类找不到"和jar 包冲突问题。

## 十二、枚举

枚举是固定常量的类型安全表示，比一堆 `public static final int` 好得多：编译期检查、可带属性和方法、能用在 switch 里。

```java
public enum Level {
    LOW(1), MEDIUM(5), HIGH(10);
    private final int code;
    Level(int code) { this.code = code; }
    public int getCode() { return code; }
}
```

## 常见坑

1. `==` 比较的是引用（对象是否同一个），字符串内容比较必须用 `equals()`。
2. 在 `HashMap` 的 key 或 `HashSet` 的元素类里重写了 `equals` 却没重写 `hashCode`，会导致查找失效。
3. 自动装箱的 `Integer` 缓存只覆盖 -128~127，超出范围用 `==` 比较会翻车。
4. 字符串拼接在循环里用 `+` 会产生大量临时对象，大循环用 `StringBuilder`。
5. 忘记关闭 IO 流/数据库连接导致资源泄漏，用 try-with-resources。
6. `SimpleDateFormat` 不是线程安全的，多线程共用会出怪问题。
7. 浮点数 `double` 做金额运算有精度误差，金额用 `BigDecimal`。

## 学习建议

- 先把集合、异常、IO 这三块练熟，这是日常开发 80% 的代码。
- 每个知识点都亲手敲一遍，光看不写记不住；多态和泛型是理解门槛，值得多花几天。
- 源码阅读顺序建议：ArrayList → HashMap → HashSet，看懂了这三个，集合框架就通了。
- 学完做个小项目巩固，比如学生管理系统或文件批量处理工具，把 IO+集合+异常串起来。

本文为学习笔记（编纂），用自己的话重写；
