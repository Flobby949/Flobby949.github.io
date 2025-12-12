---
author: Flobby
pubDatetime: 2024-09-12T11:00:00Z
title: 从线程封闭到 ThreadLocal：理解并发中的私有空间
slug: java-threadlocal
featured: false
draft: false
tags:
  - Java
  - 并发编程
  - ThreadLocal
description: 深入理解线程封闭与 ThreadLocal 的实现原理，掌握并发编程中数据隔离的最佳实践。
---

在并发编程中，**线程安全**一直是一个绑不开的话题。通常我们会使用 `synchronized`、锁机制来保证多线程下的数据一致性。但其实，有一种"天然"的方式也能避免线程安全问题，那就是——**线程封闭（Thread Confinement）**。

## 一、线程封闭

当访问共享的可变数据时，通常需要使用同步机制，而同步操作会带来一定的性能开销。一种避免同步的有效策略是不共享数据，即仅在单线程内访问数据，从而无需同步。这种技术被称为**线程封闭（Thread Confinement）**。

线程封闭是一种重要的并发设计模式，其核心思想是**将数据严格限制在单个线程的上下文中**，确保该数据仅能被一个线程访问。通过这种方式，我们无需使用同步机制即可保证线程安全，因为数据根本不会被多个线程共享。线程封闭是实现线程安全的最简单方式之一。

### 1. 栈封闭

栈封闭是一种最自然、最纯粹的线程封闭实现方式，它利用Java方法的局部变量特性来确保数据访问的线程安全性。**栈封闭的核心思想是将数据存储在方法栈帧的局部变量表中，使数据仅在创建它的方法内可见和访问**。

在Java中，每个线程拥有独立的Java虚拟机栈，其中包含多个栈帧。每个栈帧对应一个方法调用，包含该方法的局部变量表、操作数栈、动态链接和方法返回地址。局部变量表存储了方法中声明的所有局部变量，这些变量仅在该方法执行期间存在，且仅对当前线程可见。

![栈帧结构](/images/blog/stack-frame.png)

### 2. ThreadLocal

ThreadLocal（线程局部变量）是Java中提供的一种特殊机制，用于实现线程封闭，允许每个线程拥有自己独立的数据副本。**ThreadLocal的核心是为每个线程提供变量的独立副本，使得多线程环境下访问变量时互不干扰**。

ThreadLocal通过为每个线程维护一个ThreadLocalMap来实现线程局部变量。当线程访问ThreadLocal变量时，它实际上是在访问自己线程对应的ThreadLocalMap中的值。

### 3. Ad-hoc 线程封闭

委托封闭（Ad-hoc 线程封闭）指，维护线程封闭性的职责完全由程序实现来控制，把一个对象的"所有权"**交给单个线程**，由这个线程**独占**使用，其他线程不访问它。这种方式非常脆弱，因为没有任何一种语言特性可以将对象封闭到目标线程上。

![Ad-hoc线程封闭](/images/blog/adhoc-confinement.png)

## 二、ThreadLocal

ThreadLocal是一种线程局部变量机制，它为每个使用该变量的线程提供独立的变量副本，每个线程都可以独立修改自己的副本而不会影响其他线程的副本。

### 1. 为什么需要 ThreadLocal

线程封闭中其他两种方式各有局限：

- 栈封闭 → 只能用在局部变量，不能跨方法共享数据。
- 委托封闭 → 依赖开发者约束，容易被破坏。

于是，Java 提供了 **ThreadLocal**：

- 它不是局部变量，但效果类似于"每线程一份"。
- 它把变量存放在一个 **ThreadLocalMap** 中，**每个线程都有自己的副本**，互不干扰。

![ThreadLocal原理](/images/blog/threadlocal-map.png)

```java
public class ThreadLocalExample {
    private static ThreadLocal<Integer> threadLocal = ThreadLocal.withInitial(() -> 0);

    public static void main(String[] args) {
        Runnable task = () -> {
            for (int i = 0; i < 3; i++) {
                threadLocal.set(threadLocal.get() + 1);
                System.out.println(Thread.currentThread().getName() +
                                   " -> " + threadLocal.get());
            }
        };

        new Thread(task, "Thread-A").start();
        new Thread(task, "Thread-B").start();
    }
}

// 输出
Thread-A -> 1
Thread-A -> 2
Thread-A -> 3
Thread-B -> 1
Thread-B -> 2
Thread-B -> 3
```

### 2. ThreadLocal 实现

**Thread 类**

ThreadLocalMap是Thread类中的变量，也就是说每一个线程都有一个自己的 ThreadLocalMap

```java
public class Thread implements Runnable {
    ThreadLocal.ThreadLocalMap threadLocals = null;
}
```

**ThreadLocal 类**

```java
public class ThreadLocal<T> {
  private final int threadLocalHashCode = nextHashCode();
  private static AtomicInteger nextHashCode = new AtomicInteger();
  private static final int HASH_INCREMENT = 0x61c88647;

  public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
      ThreadLocalMap.Entry e = map.getEntry(this);
      if (e != null) {
        return (T)e.value;
      }
    }
    return setInitialValue();
  }

  public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
      map.set(this, value);
    } else {
      createMap(t, value);
    }
  }

  ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
  }
}
```

观察源码中的`set()`方法，当我们调用 `threadLocal.set(value)` 时，首先会调用一个`getMap()`方法，这个方法会把当前线程的`ThreadLocalMap`取出来，然后把 `(this, value)` 存入当前线程的 `ThreadLocalMap`。

**ThreadLocalMap 类**

```java
static class ThreadLocalMap {
  static class Entry extends WeakReference<ThreadLocal<?>> {
    Object value;
    Entry(ThreadLocal<?> k, Object v) {
      super(k);
      value = v;
    }
  }
  private Entry[] table;
}
```

这里`Entry` 的 **key 是弱引用 (WeakReference)**，value 是强引用。

**为什么 key 要用 WeakReference 呢？**如果 key 用的是强引用，那么即使外部不再持有 ThreadLocal 对象，它依然会被 `ThreadLocalMap` 强引用着，永远无法被 GC 回收，导致 **ThreadLocal 对象本身泄漏**。JDK 选择弱引用的好处是，当外部没有引用时，ThreadLocal 对象可以被 GC 回收。

### 3. ThreadLocal 的风险与陷阱

**1. 内存泄漏**

即使 key 被回收变成 `null`，**value 依然是强引用**，并不会自动回收。如果线程（特别是线程池中的工作线程）长期存活，而我们又忘记调用 `remove()`，那么这些 value 就会一直残留在内存里，导致 **value 泄漏**。

解决方法：**用完必须调用 `remove()`**。

**2. 数据隔离被误解**

- ThreadLocal 保证的是 **每个线程的数据独立**，而不是数据安全。
- 如果一个对象被多个线程共享并存入 ThreadLocal，就无法达到线程封闭的效果。

**3. ThreadLocal 滥用**

- ThreadLocal 常被用来"偷懒"，把数据隐式地传递。
- 这样代码耦合度高，维护成本大，阅读时不容易理解变量的来龙去脉。
