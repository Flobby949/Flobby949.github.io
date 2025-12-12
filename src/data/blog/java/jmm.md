---
author: Flobby
pubDatetime: 2024-09-03T10:00:00Z
title: Java 内存模型（JMM）详解
slug: java-memory-model
featured: true
draft: false
tags:
  - Java
  - 并发编程
  - JMM
description: 深入理解 Java 内存模型，从物理机并发到 JMM 抽象，掌握 happens-before 原则与指令重排序。
---

JMM是`Java内存模型`（Java Memory Model）的简称，是一个抽象的规范模型，定义了Java程序中线程如何与**内存交互**以及线程之间**通信**的规则。这是关于内存"如何被访问"的问题。

## 物理机并发

物理机中的并发和虚拟机有很多相似之处，JMM本质上是在物理硬件基础上构建的一层抽象，它借鉴了物理机的内存模型，并针对Java语言特点进行了调整和优化。在介绍JMM之前先了解一下物理机的并发问题。

现代CPU都是多核心架构，每个核心都是独立的处理单元，拥有自己的**算数逻辑单元**、**控制单元**、**一级缓存**、**二级缓存**。这种多核心CPU通过任务并行显著提高了处理能力。但是这种多核架构也带来了一系列新问题，其中最关键的就是**如何协调多核心对共享资源的访问**。

![CPU多核架构](/images/blog/cpu-architecture.png)

### 缓存一致性问题

为了解决CPU和主内存之间的速度差异，现代计算机采用多级缓存。每个核心都有自己的私有缓存，同时也存在多核心共享的L3缓存和主内存。所以当多个核心访问同一内存位置的时候，可能会出现缓存数据不一致的问题。

![缓存一致性问题](/images/blog/cache-consistency.png)

两个核心同时读取内存中的`x`到自己的L1缓存中，此时核心0修改自己L1缓存中的x=1，核心1此时从自己L1缓存中读取的x=0。在T5核心0写回主内存之前，核心1不知道x被修改，这就出现了脏读，导致数据不一致。

而物理机中的缓存一致性问题会直接影响到Java并发编程中的可见性问题。

```java
public class MemoryVisibilityProblem {
    private boolean flag = false;

    public void writer() {
        flag = true;
    }

    public void reader() {
        while (!flag) {
          // 空循环
        }
        System.out.println("Flag is now true");
    }

    public static void main(String[] args) throws InterruptedException {
        MemoryVisibilityProblem problem = new MemoryVisibilityProblem();

        // 启动读线程
        new Thread(problem::reader).start();

        // 确保读取线程先启动
        Thread.sleep(1000);

        // 启动写线程，修改flag
        new Thread(problem::writer).start();
    }
}
```

为了解决这些问题，处理器使用**缓存一致性协议**（如MESI）、**内存屏障**等机制。

## Java内存模型

这些硬件机制虽然强大，但对于高级语言程序员来说，它们太过底层和复杂。程序员需要一种更高层次的抽象，一种能够直接映射到编程语言的内存模型。这就是**Java内存模型(JMM)**的由来。JMM是硬件内存模型在Java语言层面的抽象和规范，它定义了一套规则，使得Java开发者可以编写正确、可移植的并发程序，而无需关心底层硬件的复杂性。

此之前，主流程序语言（如C和C++等）直接使用物理硬件和操作系统的内存模型。因此，由于不同平台上内存模型的差异，有可能导致程序在一套平台上并发完全正常，而在另外一套平台上并发访问却经常出错，所以在某些场景下必须针对不同的平台来编写程序。Java作为"一次编写，到处运行"的语言，需要提供**一致的行为保证**，无论底层硬件如何。

### 主内存与工作内存

JMM规定所有变量都存储在主内存（类比物理机的主内存）中，同时每条线程还拥有自己私有的工作内存（类比CPU核心的高速缓存），线程的工作内存中保存了该线程使用的变量的主内存副本。线程的所有操作都必须在工作内存中进行，不能直接修改主内存数据。同时不同线程也无法直接访问对方的工作内存，线程之间数据传递都需要依靠主内存。

![JMM内存模型](/images/blog/jmm-model.png)

### 内存交互操作

JMM定义了8种原子操作来完成主内存和工作内存之间的交互，JVM是实现时必须保证每一种操作都是原子的（long和double除外）：

- `lock`：锁定主内存变量，把主内存中的变量标识为某一线程独占
- `unlock`：解锁主内存变量，把主内存中锁定的变量解锁
- `read`：操作主内存，从主内存读取到工作内存
- `load`：操作工作内存，将`read`操作获得的值放入工作内存副本
- `use`：将工作内存值传给执行引擎，当虚拟机遇到一个需要使用变量的值的字节码指令时将会执行这个操作
- `assign`：将执行引擎值赋给工作内存中的变量
- `store`：操作工作内存，将工作内存值传到主内存
- `write`：操作主内存，将`store`操作获得的值写入主内存的变量中

同时这八种操作之间还有一些执行规则，例如`read`和`load`必须成对出现，不允许`read`后不进行`load`。

### happens-before原则

**Happens-Before原则**是 JMM 中最重要的概念之一，它定义了多线程环境中操作之间的可见性和有序性保证。理解这一原则对于编写正确、可靠的并发Java程序至关重要。

happens-before 是 JMM 中定义的两项操作之间的偏序关系，比如说操作A先行发生于操作B，其实就是说在发生操作B之前，操作A产生的影响能被操作B观察到，`A happens-before B`。

```
无同步情况：
线程A: a = 1; b = 2;
线程B: 可能看到b=2但a=0

使用happens-before规则：
线程A: a = 1; b = 2;
线程B: 通过规则保证，如果看到b=2，则一定看到a=1
```

JMM 定义了一组天然的happens-before规则，这些规则无需任何同步器协助，它们构成了Java并发编程的内存可见性保证基础

+ **程序顺序规则（Program Order Rule）**：在同一个线程中，按照控制流顺序，书写在前面的操作happens-before书写在后面的操作。PS：虽然JMM保证了程序顺序，但**允许指令重排序**，只要不改变单线程执行结果。

+ **管程锁定规则（Monitor Lock Rule）**：一个unlock操作happens-before后续对同一个锁的lock操作。这也解释了为什么 synchronized 保证了可见性。
+ **volatile变量规则（Volatile Variable Rule）**：对一个volatile变量的写操作happens-before后续对这个volatile变量的读操作。
+ **线程启动规则（Thread Start Rule）**：线程的`start()`方法happens-before该线程的任何操作。
+ **线程终止规则（Thread Termination Rule）**：线程中的所有操作happens-before其他线程检测到该线程已终止（我们可以通过Thread::join()方法是否结束、Thread::isAlive()的返回值等手段检测线程是否已经终止执行）。
+ **线程中断规则（Thread Interruption Rule）**：对线程`interrupt()`方法的调用happens-before被中断线程的代码检测到中断事件的发生（可以通过Thread::interrupted()方法检测到是否有中断发生）。
+ **对象终结规则（Finalizer Rule）**：一个对象的初始化完成（构造函数执行结束）先行发生于它的`finalize()`方法的开始。
+ **传递性（Transitivity）**：如果操作A happens-before 操作B，操作B happens-before 操作C，那么操作A happens-before 操作C。

### 指令重排序

从硬件架构角度来看，指令重排序是指处理器采用的一种优化技术，允许将多条指令不按程序规定的顺序分发给相应的电路单元进行处理。这种重排序并非随意的，处理器必须能够正确处理指令间的依赖关系，以确保程序能够产生正确的执行结果。

在单线程环境中，这种优化通常是有益的且对程序员不可见；例如这一段代码，在单线程场景下，处理器可能会将 `b = b * 3` 提前到 `a = a + 5` 之前执行，以提高执行效率。由于它们在单线程中互不影响，最终结果仍然是正确的。然而，在多线程环境中，它可能导致意想不到的结果。因此，深入理解指令重排序对于掌握并发编程至关重要。

```java
int a = 1;
int b = 2;
a = a + 5;
b = b * 3;
```

在Java内存模型(JMM)中，虽然允许指令重排序，但通过一系列机制对其进行了严格控制。其中，**happens-before规则**明确规定了哪些操作不能被重排序。此外，JMM还可以通过插入**内存屏障**来限制指令重排序。例如，`volatile`修饰的变量读写操作会插入特殊的内存屏障，从而禁止特定类型的指令重排序。
