---
author: Flobby
pubDatetime: 2024-10-09T15:00:00Z
title: 深入理解 Java Synchronized 关键字
slug: java-synchronized
featured: true
draft: false
tags:
  - Java
  - 并发编程
  - JUC
description: 从线程同步、锁机制到 JVM 层面的实现原理，全面解析 synchronized 关键字的工作原理与锁升级过程。
---

## 一、线程同步

### 并发与并行

计算机的多核时代，让我们能够同时执行多个任务，但"同时"这两个字其实暗藏玄机。

* **并发（Concurrency）**是指多个任务在同一时间段内交替执行，看似同时进行，实则在 CPU 时间片间切换。

* **并行（Parallelism）**则是真正意义上的同时执行，多个任务各占一个 CPU 核心并发运行。

Java 的多线程模型主要服务于**并发编程**。但正因为线程会"交错执行"，我们无法预测线程切换的时机，这就导致了线程间可能同时访问同一个共享资源，从而引发数据错乱。

### 线程安全问题

我们来看一个简单的计数器场景，在单线程中，count++ 这样的操作毫无问题。但是进入多线程并发后，这个操作变得不再可控

```java
public class Counter implements Runnable {
    private int count = 0;

    @Override
    public void run() {
        for (int i = 0; i < 10000; i++) {
            count++;
        }
    }

    public void print() {
        System.out.println("count = " + count);
    }

    public static void main(String[] args) {
        Counter counter = new Counter();

        Thread thread1 = new Thread(counter, "线程A");
        Thread thread2 = new Thread(counter, "线程B");

        thread1.start();
        thread2.start();

        try {
            thread1.join();
            thread2.join();
            counter.print();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

在单线程的情况下，输出结果一定是 count = 20000，但变成多线程运行后，大概率会看到远小于 20000 的结果，例如 count = 17853、count = 19210，每次运行都不一样。这正是**线程安全问题**的典型例子。

count++ 看起来是一个简单操作，实际上在 JVM 字节码层面被分解成三步：

1. 读取变量 count 的值到工作内存
2. 对该值执行加一操作
3. 将结果写回主内存

当两个线程几乎同时执行到这段代码时，可能会出现这种交错：

| **线程A**      | **线程B**      | **count值** |
| -------------- | -------------- | ----------- |
| 读取 count=100 | 读取 count=100 | 100         |
| 加1  → 101     | 加1  → 101     | 101         |
| 写回 → 101     | 写回 → 101     | 101         |

结果：count 只增加了一次。

这就是**竞态条件（Race Condition）**：多个线程竞争修改同一资源，最终结果依赖于线程的执行时序。

### 线程同步的意义

为了解决这种问题，我们需要引入"同步"机制，让多个线程在访问共享资源时**有序进行**。简单来说，就是"同一时间只能有一个线程"执行那段关键代码。

同步的核心目标：

- **互斥（Mutual Exclusion）**：防止同时访问共享资源；
- **可见性（Visibility）**：保证修改结果能被其他线程及时看到；
- **有序性（Ordering）**：防止指令重排序带来的异常执行顺序。

通过某种机制（如锁），让同一时刻只能有一个线程进入临界区（critical section），保证共享变量的一致性与正确性。

## 二、锁

### 锁的基本概念

锁（Lock）是一种**同步原语**，用于控制多个线程对共享资源的访问。当一个线程持有锁时，其他线程必须等待它释放锁后才能继续访问。

锁的本质，是**协调线程访问临界区的顺序**。而所谓**临界区（Critical Section）**，就是程序中会访问共享数据的那部分代码。

以第一章的 Counter 为例，我们可以在 count++ 前后加上锁保护：

```java
public synchronized void run() {
    for (int i = 0; i < 10000; i++) {
        count++;
    }
}
```

这行 synchronized 关键字会告诉 JVM：这个代码块是临界区，任何时刻只能有一个线程进入。

### 锁的分类

1. 按实现层次划分

| **类型**             | **实现层面**                    | **典型代表**  |
| -------------------- | ------------------------------- | ------------- |
| JVM 内置锁(监视器锁) | JVM 负责加锁与释放              | synchronized  |
| JUC 显式锁           | 由 Java 代码实现，基于 AQS 框架 | ReentrantLock |

这两类锁最终都依赖于操作系统底层的互斥机制，但控制权不同。synchronized 交给 JVM 自动管理，ReentrantLock 由开发者手动获取和释放。

2. 按特性划分

| **特性**              | **含义**                             | **示例**                                     |
| --------------------- | ------------------------------------ | -------------------------------------------- |
| 可重入锁（Reentrant） | 同一线程可多次获得同一把锁，不会死锁 | synchronized、ReentrantLock                  |
| 公平锁 / 非公平锁     | 线程获取锁的顺序是否按先来先得       | ReentrantLock 可配置                         |
| 独享锁 / 共享锁       | 是否允许多个线程同时持有             | ReentrantLock（独享）、ReadWriteLock（共享） |
| 自旋锁（Spin Lock）   | 在短期竞争中循环等待而不挂起         | JVM 轻量级锁机制                             |
| 乐观锁 / 悲观锁       | 是否假设无竞争，竞争失败再补救       | CAS 属于乐观锁思想                           |
| 阻塞锁 / 非阻塞锁     | 是否涉及线程挂起与唤醒               | synchronized 是阻塞锁，CAS 是非阻塞锁        |

## 三、Synchronized

synchronized 是 Java 世界最基础的同步手段。synchronized 是语言级的锁机制，它的真正实现者，是 JVM 的 **对象头（Object Header）** 与 **Monitor（监视器）**。

### JVM层面的实现原理

JVM基于进入和退出`Monitor对象`来实现`方法同步`和`代码块同步`，但两者的实现细节不一样。代码块同步是使用`monitorenter`和`monitorexit`指令实现的，而方法同步是使用另外一种方式实现的，细节在JVM规范里并没有详细说明。但是，方法的同步同样可以使用这两个指令来实现。

```java
public void syncMethod() {
    synchronized (this) {
        System.out.println("Hello Sync");
    }
}
```

我们来看一下synchronized 究竟做了什么，查看一下这一段代码的字节码

```
 0 aload_0
 1 dup
 2 astore_1
 3 monitorenter
 4 getstatic #7 <java/lang/System.out : Ljava/io/PrintStream;>
 7 ldc #13 <Hello Sync>
 9 invokevirtual #15 <java/io/PrintStream.println : (Ljava/lang/String;)V>
12 aload_1
13 monitorexit
14 goto 22 (+8)
17 astore_2
18 aload_1
19 monitorexit
20 aload_2
21 athrow
22 return
```

关键就在这两条指令：

- **monitorenter**：尝试获取锁，在编译后插入到同步代码块的开始位置
- **monitorexit**：释放锁，插入到方法结束处和异常处

可以看到，编译器在同步代码块的前后自动插入了这两条指令，并且有两个monitorexit，这是因为在异常路径也插入了 monitorexit，以保证即使抛异常也能释放锁。这就是 synchronized 能自动防止死锁（未释放锁）的原因。

### 对象头中的MarkWord结构

JVM 中每个对象在内存中都由三部分组成：

|                   | **说明**                              |
| ----------------- | ------------------------------------- |
| **Mark Word**     | 存储锁状态、哈希码、GC 分代年龄等信息 |
| **Klass Pointer** | 指向对象的类型元数据（类信息）        |
| **实例数据区**    | 存储对象字段值                        |

**Mark Word** 是锁的核心。JVM 会在其中记录锁的状态、持有线程 ID、偏向信息等内容。

![MarkWord结构](/images/blog/markword.png)

### Monitor 机制

Monitor（监视器）是 JVM 层面的**锁对象**，本质上是一个同步工具结构，用于管理线程的进入与等待。HotSpot 的 ObjectMonitor 结构核心包含以下成员（伪代码简化）：

```cpp
ObjectMonitor {
    Object* object;       // 对应的锁对象
    Thread* owner;        // 当前持有锁的线程
    int entryCount;       // 重入次数
    EntryList waiters;    // 等待队列
}
```

当线程执行到 `monitorenter`：

1. 如果 Monitor 的 owner 字段为空，则当前线程设置为 owner，获取锁成功；
2. 如果已有线程持有锁，则当前线程进入 EntryList 等待队列；
3. 当锁释放（monitorexit）时，会唤醒等待队列中的线程尝试重新竞争。

这就是 Java 中"线程阻塞"和"唤醒"的底层机制。

### 锁升级

synchronized并不是一开始就是**重量级锁**，JVM 会根据竞争情况动态地升级或降级锁，以提高性能。这个过程称为**锁优化（Lock Optimization）**。

1. **无锁状态（No Lock）**：对象刚创建，未被任何线程加锁。
2. **偏向锁（Biased Lock）**：假设锁长期被同一线程持有，避免频繁 CAS。
3. **轻量级锁（Lightweight Lock）**：出现少量竞争时，通过 CAS 操作自旋等待。
4. **重量级锁（Heavyweight Lock）**：竞争激烈，线程被挂起并排队等待。

这是一种**逐级升级、不可逆的锁状态变化**。偏向锁能快速进入临界区，轻量级锁用自旋避免阻塞，只有在竞争严重时才膨胀为重量级锁。这一策略让 synchronized 在 JDK 1.6 之后性能大幅提升。

## 四、锁状态的细节变化

在 HotSpot 虚拟机中，锁并不是固定的结构，而是一个**可动态演化的状态机**。虚拟机会根据**竞争程度**和**线程行为模式**自动调整锁的类型，从而在"性能"和"安全"之间做出权衡。

### 锁状态

|                                  | **竞争程度** | **特点**                        | **适用场景**           |
| -------------------------------- | ------------ | ------------------------------- | ---------------------- |
| **无锁（No Lock）**              | 无竞争       | 对象未被任何线程持有锁          | 对象刚创建或从未加锁   |
| **偏向锁（Biased Lock）**        | 极低         | 偏向第一个获得锁的线程          | 同一线程反复进入同步块 |
| **轻量级锁（Lightweight Lock）** | 低           | 使用 CAS 自旋，避免线程挂起     | 短时间小规模竞争       |
| **重量级锁（Heavyweight Lock）** | 高           | 线程阻塞，进入 Monitor 等待队列 | 高并发、长时间竞争     |

锁的状态保存在对象头的 Mark Word 中，通过其中的标志位控制状态转换。

整个锁升级的路径如下：

> 无锁 → 偏向锁 → 轻量级锁 → 重量级锁

###  偏向锁

偏向锁（Biased Locking）基于HotSpot的作者经过研究发现：大多数情况下，锁不仅不存在多线程竞争，而且总是由同 一线程多次获得。

因此，当一个线程第一次获得锁时，虚拟机会在对象头中**记录这个线程的 ID**，并将锁标记为偏向锁。下次同一线程进入同步块时，无需任何同步操作，只需判断对象头中的线程 ID 是否匹配即可直接进入。

**偏向锁的撤销：**

当另一线程尝试获得同一把锁时，JVM 会触发偏向锁撤销过程（这涉及 Safepoint 停顿），然后升级为轻量级锁。这种设计让单线程同步场景几乎无锁成本。

### 轻量级锁

轻量级锁（Lightweight Lock）采用 CAS （Compare-And-Swap）操作来竞争锁。

- 当一个线程进入同步块时，JVM 在栈帧中创建一个**锁记录（Lock Record）**，并尝试用 CAS 将对象头中的 Mark Word 替换为指向这个锁记录的指针。
- 如果替换成功，说明当前线程获得锁；
- 如果替换失败，说明有竞争，线程会自旋（spin）等待锁释放，而不是立即挂起。

自旋的好处是：**避免线程上下文切换**（比挂起和唤醒快几个数量级）。但如果竞争激烈，自旋次数过多会浪费 CPU，JVM 会根据情况自动升级为重量级锁。

### 重量级锁

当自旋竞争持续失败时，JVM 会将锁升级为重量级锁。此时，锁的实现依托于 ObjectMonitor，线程会进入阻塞队列 (EntryList) 中等待唤醒。

特点：

- 线程挂起与唤醒由操作系统调度；
- 性能开销大；
- 能保证任何竞争下的正确性。

这也是 synchronized 早期被认为"慢"的原因：在 JDK 1.5 之前，是重量级锁。

### 锁优化技术

除了锁升级，HotSpot 还引入了几项重要优化：

- **锁消除（Lock Elimination）**：编译器通过逃逸分析发现某锁对象不会被共享，直接去掉锁。
- **锁粗化（Lock Coarsening）**：将多次连续加锁操作合并为一次，减少加锁次数。
- **自适应自旋（Adaptive Spinning）**：JVM 根据历史竞争记录动态调整自旋次数。

这些机制让现代 synchronized 的性能在多数场景下与 ReentrantLock 相差无几。
