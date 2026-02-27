---
author: Flobby
pubDatetime: 2026-02-27T10:00:00Z
title: JVM 内存布局详解：从线程私有到共享区域
slug: jvm-memory-layout
featured: true
draft: false
tags:
  - Java
  - JVM
  - 内存管理
description: 深入理解 JVM 运行时数据区（Runtime Data Areas），掌握线程私有区与共享区的职责划分，为学习垃圾回收机制打下坚实基础。
---

我们可以把 JVM 内存想象成一个大型办公园区，它被划分为**线程私有**和**线程共享**两大区域。理解这个布局是掌握 JVM 调优和垃圾回收的基础。

## 线程私有区：每个线程的"独立工位"

这部分内存随着线程的创建而产生，随着线程的结束而销毁。因为是私有的，所以**不需要垃圾回收**，也不存在线程安全问题。

### 程序计数器（Program Counter Register）

程序计数器是当前线程执行的字节码的行号指示器。你可以把它理解为执行引擎的"GPS"，告诉 CPU 下一步该执行哪行代码。

它是 JVM 规范中唯一没有规定任何 `OutOfMemoryError` 情况的区域。如果线程正在执行 Java 方法，计数器记录的是正在执行的虚拟机字节码指令地址；如果执行的是 Native 方法，则计数器值为空（Undefined）。

### Java 虚拟机栈（JVM Stack）

虚拟机栈描述的是 Java 方法执行的线程内存模型。它存放了**局部变量表**、操作数栈、动态链接、方法出口等信息。

每调用一个方法，就会压入一个"栈帧"（Stack Frame）。方法执行完毕后，栈帧出栈。你平时遇到的 `StackOverflowError` 就是这里满了——通常是递归调用层次过深导致的。

```java
// 典型的 StackOverflowError 场景
public void infiniteRecursion() {
    infiniteRecursion(); // 无限递归，栈帧不断压入
}
```

### 本地方法栈（Native Method Stack）

本地方法栈为虚拟机使用到的 `native` 方法服务。当 Java 代码调用 C/C++ 实现的本地方法时，就会用到这个区域。

HotSpot 虚拟机直接将本地方法栈和虚拟机栈合二为一。

## 线程共享区：园区的"公共区域"

这里是 GC 真正发挥作用的主战场。所有线程共享这些区域，因此需要考虑线程安全和垃圾回收问题。

### 堆（Heap）——"对象的仓库"

堆是 JVM 管理的**最大**一块内存，几乎所有的**对象实例**和**数组**都在这里分配。这也是垃圾收集器管理的主要区域，因此也被称为"GC 堆"。

为了方便垃圾回收，堆内部被划分成了不同的"社区"：

**新生代（Young Generation）**

- **Eden 区**：绝大多数对象在这里诞生。当 Eden 区满时，会触发 Minor GC。
- **Survivor 区（S0/S1）**：经历过一次 Minor GC 幸存下来的对象会待在这里。两个 Survivor 区交替使用。

**老年代（Old Generation）**

大对象或者在新生代"熬过"多次 GC（默认 15 次）的长寿对象会晋升到这里。老年代的 GC 频率较低，但每次耗时更长。

![堆内存布局](/images/blog/heap-memory.png)

### 方法区（Method Area / Metaspace）——"蓝图库"

方法区存储已被加载的**类信息、常量、静态变量**、即时编译器编译后的代码缓存等数据。

**历史演变**：

- **JDK 7 及之前**：叫做"永久代"（PermGen），放在堆里，使用 `-XX:PermSize` 和 `-XX:MaxPermSize` 配置。
- **JDK 8 及之后**：改名为"元空间"（Metaspace），直接使用**本地内存（Native Memory）**，不再占用堆内存。使用 `-XX:MetaspaceSize` 和 `-XX:MaxMetaspaceSize` 配置。

这个改变解决了永久代容易溢出的问题，因为本地内存通常比堆内存大得多。

## 为什么理解内存布局如此重要？

理解了布局，你就能秒懂 GC 的两个核心命题：

### 1. 哪些对象该回收？（可达性分析）

GC 会从一系列被称为 **GC Roots** 的根对象开始，向下搜索引用链。如果一个对象到 GC Roots 之间没有任何引用链相连，说明它已经不可达，可以被回收。

**GC Roots 主要包括**：

- 虚拟机栈中引用的对象（方法里的局部变量）
- 方法区中静态属性引用的对象（类的 `static` 变量）
- 方法区中常量引用的对象（字符串常量池里的引用）
- 本地方法栈中 JNI 引用的对象

### 2. 在哪个区域回收？

- **Minor GC**：专门清理 Eden 和 Survivor 区，触发频繁但速度快。
- **Major GC**：只回收老年代，目前只有 CMS 收集器有这种行为。
- **Full GC**：清理整个堆（包括老年代）甚至方法区，代价最大。

## 常见内存异常

| 异常类型 | 发生区域 | 典型原因 |
|---------|---------|---------|
| `StackOverflowError` | 虚拟机栈 | 递归过深、栈帧过大 |
| `OutOfMemoryError: Java heap space` | 堆 | 对象过多、内存泄漏 |
| `OutOfMemoryError: Metaspace` | 元空间 | 加载类过多、动态代理滥用 |

理解 JVM 内存布局是学习垃圾回收机制的前提。下一篇文章我们将深入探讨垃圾回收的判定算法和收集策略。
