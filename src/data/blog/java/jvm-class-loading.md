---
author: Flobby
pubDatetime: 2026-03-04T11:00:00Z
title: JVM 类加载机制：从字节码到可运行对象的完整旅程
slug: jvm-class-loading
featured: true
draft: false
tags:
  - Java
  - JVM
  - 类加载
  - 双亲委派
description: 深入理解 JVM 类加载的五个生命周期阶段、三层类加载器家族，以及双亲委派模型的设计哲学与实际打破场景。
---

如果说 JVM 的内存布局是"办公园区"，垃圾回收是"保洁保安"，那么**类加载机制**就是这个园区的"人力资源部"——负责把磁盘上的 `.class` 字节码文件，转换成内存中可以运行的 `java.lang.Class` 对象。

这个过程不是一蹴而就的，而是一个严谨的五步旅程。

## 类加载的生命周期：五部曲

当你在代码里第一次用到某个类（比如 `new User()`），HR 部门就开始干活了。整个过程分为五个阶段：

### 第一步：加载（Loading）

- 通过类的**全限定名**（如 `com.me.User`）找到对应的字节码数据（通常是 `.class` 文件，也可以来自网络、数据库、动态生成）。
- 将字节流代表的静态存储结构转化为方法区的运行时数据结构。
- 在堆中生成一个对应的 `java.lang.Class` 对象，作为访问方法区数据的入口。

### 第二步：验证（Verification）

**安检环节**。JVM 不相信任何外来字节码，必须核验它是否符合规范、是否有安全隐患。验证包括四个层次：

1. **文件格式验证**：是否以魔数 `0xCAFEBABE` 开头？版本号是否合法？
2. **元数据验证**：类的继承关系是否合法？是否继承了不可继承的类（`final` 类）？
3. **字节码验证**：指令是否合法？类型转换是否安全？
4. **符号引用验证**：引用的类、方法、字段是否真实存在？是否有访问权限？

> 如果你对字节码文件的合法性有足够信心（例如是自己可信的代码），可以用 `-Xverify:none` 关闭验证来加快启动速度。

### 第三步：准备（Preparation）

**分配工位**。为类中的**静态变量**（`static` 字段）分配内存，并赋予**零值**。

```java
public class Example {
    public static int count = 100;  // 准备阶段后，count = 0（零值，不是 100）
    public static final int MAX = 50;  // 这是常量，准备阶段直接赋值 50
}
```

这里有两个容易混淆的细节：
- `static` 变量：准备阶段只赋零值，真正赋值 `100` 是在初始化阶段。
- `static final` 常量：如果是编译期可知的字面量，则在准备阶段直接赋真实值。

### 第四步：解析（Resolution）

把符号引用替换为直接引用。

字节码中对其他类、方法、字段的引用，最初都是以符号（名字字符串）的形式存在的——因为编译时还不知道它们最终在内存中的地址。解析阶段就是把这些"名字"替换成真实的内存指针。

### 第五步：初始化（Initialization）

**正式入职**。执行类的初始化方法 `<clinit>`——这是编译器收集所有静态变量赋值语句和 `static {}` 代码块，按顺序合并生成的。

```java
public class Database {
    static {
        System.out.println("1. 静态代码块执行");
        url = "jdbc:mysql://localhost:3306/db";  // 可以访问后面声明的静态变量
    }

    public static String url;
    public static int maxPoolSize = 10;

    // 编译器生成的 <clinit> 等价于：
    // url = "jdbc:mysql://...";
    // maxPoolSize = 10;
}
```

`<clinit>` 是线程安全的——JVM 保证多线程并发初始化时，只有一个线程能执行它，其他线程会阻塞等待。这也是**静态内部类单例模式**线程安全的根本原因。

```java
// 利用 <clinit> 线程安全的经典单例
public class Singleton {
    private static class Holder {
        static final Singleton INSTANCE = new Singleton();
    }
    public static Singleton getInstance() {
        return Holder.INSTANCE;  // 第一次访问时，Holder 类被初始化
    }
}
```

## 类加载器：三层"面试官"

谁来执行上述加载过程？JVM 设置了三个级别的类加载器。

### Bootstrap ClassLoader（启动类加载器）

- **地位**：最高，由 **C++** 实现，是 JVM 的一部分。
- **职责**：加载 Java 核心类库（JDK 8 的 `rt.jar`，JDK 9+ 的 `java.base` 等模块）。
- **特殊性**：在 Java 代码中是不可见的，直接调用返回 `null`。

```java
// Bootstrap 加载了 String，但在 Java 里拿不到它
System.out.println(String.class.getClassLoader());  // 输出：null
```

### Platform/Extension ClassLoader（平台/扩展类加载器）

- **地位**：中层，Java 实现。
- **职责**：加载扩展库。JDK 8 中是 `JAVA_HOME/lib/ext` 目录，JDK 9+ 更名为 Platform ClassLoader，负责加载平台模块。

### Application ClassLoader（应用程序类加载器）

- **地位**：最常用，Java 实现。
- **职责**：加载你项目 `classpath` 下的所有类——即你写的代码、引入的第三方 jar 包，默认都是它负责的。
- **获取方式**：`ClassLoader.getSystemClassLoader()`

## 双亲委派模型：逐级请示，不行再自己上

三个加载器是如何协作的？遵循一个关键规则：**双亲委派（Parent Delegation）**。

### 工作流程

当应用类加载器收到"请加载 `com.myapp.User`"的请求时：

```
Application ClassLoader 收到请求
        ↓（向上委派）
Platform ClassLoader 收到请求
        ↓（向上委派）
Bootstrap ClassLoader 收到请求
        ↓（在核心库里找，找不到 User）
返回给 Platform ClassLoader
        ↓（在扩展库里找，找不到 User）
返回给 Application ClassLoader
        ↓（在 classpath 里找，找到了！）
Application ClassLoader 加载 User
```

用一句话概括：**收到请求不自己先做，先往上推；父类搞不定了，才轮到自己**。

### 为什么要这么设计？

**原因一：安全性——防止核心类被伪造**

假设有人写了一个假的 `java.lang.String`，里面藏着窃取密码的代码：

```java
package java.lang;
public class String {
    // 恶意代码...
    static { stealPasswords(); }
}
```

- **没有双亲委派**：JVM 直接加载了这个假 String，账号密码全丢了。
- **有了双亲委派**：加载请求传到 Bootstrap，Bootstrap 说"我早有一个正版的了"，直接返回。恶意代码没有任何机会进内存。

**原因二：唯一性——避免重复加载**

通过往上请示，保证同一个类在 JVM 中只有一份。不然 Bootstrap 和 Application 各自加载一次，JVM 会把它们当成两个完全不同的类（即使字节码一样），造成类型系统混乱。

### 四步法记忆

面试时被追问"详细说说双亲委派流程"，记住这四步：

1. **检查缓存**：加载器先检查自己是否已经加载过此类，有则直接返回。
2. **向上委派**：没有则把请求发给父加载器。
3. **父类递归**：父加载器重复上述过程，一直到 Bootstrap。
4. **向下兜底**：父类都加载不了，子加载器才自己从磁盘读取字节码进行加载。

## 打破双亲委派：必要时的"越权"

双亲委派很安全，但在某些场景下太死板了，实际上三次著名的"打破"推动了 Java 生态的发展。

### 第一次：SPI 机制（JDBC 等）

Java 定义了数据库连接接口（`java.sql.Driver`），由 Bootstrap 加载。但具体的驱动实现（`com.mysql.jdbc.Driver`）在用户 jar 包里，属于 Application ClassLoader 的地盘。

问题来了：**Bootstrap 无法向下"请求" Application ClassLoader 去加载子类**，委派只能往上，不能往下。

解决方案：引入**线程上下文类加载器（Thread Context ClassLoader）**，允许父类加载器请求子类加载器去加载。本质上是一条"绕过委派的后门通道"。

```java
// JDBC 驱动的加载（简化版）
ServiceLoader<Driver> loader = ServiceLoader.load(Driver.class);
// ServiceLoader 内部会用线程上下文类加载器去加载具体实现
```

### 第二次：Tomcat 的类隔离

Tomcat 里同时运行着多个 Web 应用，它们可能依赖同一个库的不同版本：

- 应用 A：`spring-5.3.jar`
- 应用 B：`spring-6.0.jar`

如果遵守双亲委派，父类加载器只会加载其中一个版本，另一个应用就挂了。

Tomcat 的方案：给每个 Web 应用分配一个独立的 **WebAppClassLoader**，让它**优先自己搜索**，搜不到再向父类委派——与双亲委派方向相反。这实现了不同应用之间的**类隔离**，同时也支持了**热部署**（扔掉旧的 ClassLoader，创建新的加载更新后的类）。

### 第三次：OSGi 模块化

OSGi 框架实现了更复杂的**网状类加载**，不同模块之间可以横向共享类，形成一张依赖图而非单一的树状委派链。这是 Java 模块化系统（JPMS）出现之前最成熟的模块化方案。

### 打破方式对比

| 场景 | 打破方式 | 核心目的 |
|-----|---------|---------|
| JDBC / SPI | 线程上下文类加载器 | 父类加载器能调用子类来加载实现 |
| Tomcat | 子类优先加载 | 不同 Web 应用的类版本隔离 |
| OSGi | 网状委派 | 复杂的模块化与插件化 |
| 热部署 | 丢弃旧 ClassLoader | 不停机更新业务代码 |

## 最后一道防线

无论怎么打破，有一条线是不可逾越的：**核心 Java 类（`java.lang.*`）绝对不能被自定义加载器覆盖**。

如果你强行写一个 `java.lang.String` 并尝试加载，JVM 在验证阶段会直接抛出安全异常。这是双亲委派留下的最后一道安全护城河。

## 总结

| 阶段 | 关键动作 | 常见面试考点 |
|-----|---------|-----------|
| 加载 | 读取字节码，生成 Class 对象 | 加载来源可以多样化 |
| 验证 | 四层合法性校验 | `-Xverify:none` 可跳过 |
| 准备 | 静态变量赋零值 | 注意区分 static 和 static final |
| 解析 | 符号引用 → 直接引用 | 动态分派在此阶段之后 |
| 初始化 | 执行 `<clinit>` | 线程安全，一次性执行 |
| **双亲委派** | 向上请示，向下兜底 | 安全性、唯一性 |
| **打破委派** | 三次历史性打破 | SPI、Tomcat、OSGi |

理解类加载机制，不仅能让你更清楚一行 Java 代码从源文件到可执行状态的完整旅程，更能让你在面对 ClassNotFoundException、ClassCastException 等疑难杂症时，具备从底层找到根因的能力。
