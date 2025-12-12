---
author: Flobby
pubDatetime: 2024-09-11T16:00:00Z
title: 掌握 Java 线程：实现原理、生命周期与线程池
slug: java-thread
featured: false
draft: false
tags:
  - Java
  - 并发编程
  - 线程池
description: Java线程是并发编程的核心，本文从底层原理、生命周期到线程池全面介绍Java线程。
---

Java线程是Java并发编程的核心，它允许程序同时执行多个任务，提高系统资源利用率和程序响应能力。本文从底层原理、生命周期到基本操作全面介绍Java线程。

## 一、线程的实现

线程是比进程更轻量级的调度执行单位，一个程序中不同的执行路径就是线程。线程的引入将进程的资源分配与执行调度进行分离，各个线程既可以共享进程资源（内存地址、文件I/O等），又可以独立调度。

**进程（Process）**：

- 操作系统进行资源分配的基本单位
- 拥有独立的地址空间和系统资源
- 进程间通信需要通过IPC（进程间通信）机制

**线程（Thread）**：

- 进程内的执行单元
- 共享进程的地址空间和资源
- 线程间通信可直接通过共享内存
- 线程切换开销远小于进程切换

操作系统实现线程主要有三种方式：使用内核线程实现（1：1实现），使用用户线程实现（1：N实现），使用用户线程加轻量级进程混合实现（N：M实现）。

**内核级线程（Kernel-Level Threads, KLT）**

直接由操作系统内核支持的线程，这种线程由内核来完成线程切换，内核通过调度器对线程进行调度，并负责将线程的任务映射到各个处理器上。每个内核线程可以视为内核的一个分身，这样操作系统就有能力同时处理多件事情，所以使用内核线程实现的方式也被称为1：1实现。

程序一般不会直接使用内核线程来实现线程，而是使用轻量级线程（LightWeight Process，LWP）实现，轻量级线程就是通常意义上的线程，每个轻量级线程都需要一个内核线程支持。

![内核级线程](/images/blog/kernel-thread.png)

由于内核线程的支持，每个轻量级线程都成为一个独立的调度单元，所以一个线程阻塞不会影响整个进程运行。但是也是因为轻量级线程基于内核线程，所以各种线程操作都需要进行系统调用，而系统调用需要在用户态（User Mode）和内核态（Kernel Mode）之间切换。其次每个轻量级线程都需要内核线程支持，所以需要消耗一定的内核资源。

**用户级线程（User-Level Threads, ULT）**

使用用户线程实现的方式被称为1：N实现。广义上来讲，一个线程只要不是内核线程，都可以认为是用户线程（User Thread，UT）的一种。而狭义上的用户线程指的是完全建立在用户空间的线程库上，也就是说系统内核无法感知用户线程的存在，用户线程的生命周期和调度完全在用户态中实现，不需要借助内核。

![用户级线程](/images/blog/user-thread.png)

因此用户线程的优势在于脱离内核，用户线程的所有都在用户态中实现，这种线程不需要切换到内核态，这种操作非常快速且节约资源，同时也支持更大的线程数量。用户线程的劣势也是在于没有内核的支持，所有线程操作都是用户需要考虑的，因此一个线程阻塞会导致整个进程阻塞，无法利用多核CPU。

**混合模型（Hybrid Model）**

线程除了上面两种实现之外，还有一种将内核线程和用户线程一起使用的方式。在这种实现下，既存在用户线程，也存在轻量级线程，用户线程还是在用户空间中，所以可以保留用户线程的优势。而操作系统支持的轻量级线程用户用户线程和内核线程之间的媒介。这样可以使用内核提供的线程调度功能及处理器映射，并且用户线程的系统调用要通过轻量级进程来完成，这大大降低了整个进程被完全阻塞的风险。

![混合线程模型](/images/blog/hybrid-thread.png)

## 二、线程的生命周期

Java线程通过`Thread.State`枚举类定义了六种状态，每个状态代表了线程执行过程中的一个阶段。

![线程生命周期](/images/blog/thread-lifecycle.png)

### 1. NEW（新建状态）

调用`new Thread()`后，线程被创建但尚未启动。此时线程不可执行，未分配系统资源，只是创建了一个线程对象。创建线程后应尽快调用`start()`，避免长时间处于NEW状态

### 2. RUNNABLE（运行状态）

调用`start()`方法后线程进入此状态，包括两种子状态：就绪 (Ready) 和运行中 (Running)。进入`RUNNABLE`状态时，JVM会创建相应的操作系统线程，分配必要的系统资源（如栈空间、线程控制块等）。

一个线程对象只能调用一次`start()`，多次调用会抛出`IllegalThreadStateException`

### 3. BLOCKED（阻塞状态）

线程等待获取排他锁时进入此状态（例如等待进入`synchronized`代码块或方法），处于阻塞状态的线程不会消耗CPU资源。获取到锁后，线程重新进入`RUNNABLE`状态

### 4. WAITING（等待状态）

线程**无限期等待**另一个线程执行特定操作，必须通过显式通知才能唤醒线程。可以通过`Object.wait()`, `Thread.join()`, `LockSupport.park()`等方法进入。调用`Object.notify()`或`Object.notifyAll()`或调用等待线程的`interrupt()`方法可以唤醒线程

### 5. TIMED_WAITING（超时等待状态）

线程在**指定时间**内等待，超时后会自动唤醒，也可以被提前唤醒。可以通过`Thread.sleep()`, `Object.wait(timeout)`, `Thread.join(timeout)`, `LockSupport.parkNanos()`等方法进入

### 6. TERMINATED（终止状态）

线程生命周期结束，线程执行完毕或被异常终止。此时线程对象仍然存在，但不能再被启动，可以通过`isAlive()`方法判断线程是否处于活跃状态（非TERMINATED状态）

## 三、线程的基本操作

### 1. 线程创建与启动

**继承Thread类**

```Java
public class MyThread extends Thread {
    @Override
    public void run() {
        System.out.println("Thread is running");
    }

    public static void main(String[] args) {
        MyThread thread = new MyThread();
        thread.start();
    }
}
```

**实现Runnable接口**

```java
public class MyRunnable implements Runnable {
    @Override
    public void run() {
        System.out.println("Thread is running");
    }

    public static void main(String[] args) {
        Thread thread = new Thread(new MyRunnable());
        thread.start();
    }
}
```

**实现Callable接口配合Future**

```java
import java.util.concurrent.*;

public class MyCallable implements Callable<String> {
    @Override
    public String call() throws Exception {
        return "Callable result";
    }

    public static void main(String[] args) {
        FutureTask<String> futureTask = new FutureTask<>(new MyCallable());
        Thread thread = new Thread(futureTask);
        thread.start();

        try {
            String result = futureTask.get();
            System.out.println(result);
        } catch (InterruptedException | ExecutionException e) {
            e.printStackTrace();
        }
    }
}
```

除了这三种方式之外，还可以通过**线程池**创建启动线程。此外在**JDK21**之后，Java还基于**混合模型**实现了虚拟线程。

### 2. 线程基本操作

**线程休眠**

```java
try {
    Thread.sleep(1000); // 线程休眠1000毫秒
} catch (InterruptedException e) {
    e.printStackTrace();
}
```

**线程让步**

```java
Thread.yield(); // 暂停当前线程，让其他线程有机会执行
```

**等待线程结束**

```java
Thread thread = new Thread(new MyRunnable());
thread.start();
try {
    thread.join(); // 无限等待
} catch (InterruptedException e) {
    e.printStackTrace();
}
```

**中断线程**

```java
Thread thread = new Thread(() -> {
    while (!Thread.currentThread().isInterrupted()) {
        // 执行任务
    }
});
thread.start();
thread.interrupt();
```

## 四、线程池

线程池是一种池化技术，它维护一组可重用的线程，用于执行提交的任务。当有新任务到达时，线程池会从池中取出一个空闲线程来执行任务，而不是创建一个新线程；当任务执行完毕后，线程不会被销毁，而是返回线程池中等待下一个任务。

**使用线程池的优势**

1. **降低资源消耗**：通过复用已存在的线程，减少线程创建和销毁造成的开销
2. **提高响应速度**：任务到达时无需创建新线程，可以立即执行
3. **提高线程的可管理性**：统一管理线程，可以设置线程的最大数量，避免系统因创建过多线程而导致资源耗尽
4. **提供更多功能**：如定时执行、定期执行、线程中断等功能

### 1. 线程池核心参数

线程池的核心实现在`java.util.concurrent.ThreadPoolExecutor`类中：

```java
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler)
```

**corePoolSize（核心线程数）**

- 线程池中保持活跃不会回收的线程数量
- 当有新任务到达时，如果当前线程数小于核心线程数，会创建新线程执行任务

**maximumPoolSize（最大线程数）**

- 线程池中允许存在的最大线程数
- 当工作队列已满且当前线程数小于最大线程数时，线程池会创建新线程执行任务

**keepAliveTime & TimeUnit（线程空闲时间）**

- 当线程数大于核心线程数时，多余的线程在空闲时间超过指定值后会被终止

**workQueue（工作队列）**

常见的队列类型：
- `ArrayBlockingQueue`：基于数组的有界阻塞队列
- `LinkedBlockingQueue`：基于链表的无界阻塞队列
- `SynchronousQueue`：不存储元素的阻塞队列

**handler（拒绝策略）**

- `AbortPolicy`：直接抛出异常（默认）
- `CallerRunsPolicy`：由提交任务的线程执行该任务
- `DiscardPolicy`：直接丢弃任务
- `DiscardOldestPolicy`：丢弃队列中最老的任务

### 2. 线程池调度流程

```java
public void execute(Runnable command) {
    int c = ctl.get();
    // 工作线程小于核心线程数量，创建工作线程
    if (workerCountOf(c) < corePoolSize) {
        if (addWorker(command, true))
            return;
        c = ctl.get();
    }
    // 工作线程大于核心线程数量，把任务offer到阻塞队列
    if (isRunning(c) && workQueue.offer(command)) {
        int recheck = ctl.get();
        if (! isRunning(recheck) && remove(command))
            reject(command);
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);
    }
    // 如果队列已满，创建新线程
    else if (!addWorker(command, false))
        // 如果创建失败，说明线程数达到最大线程数，执行拒绝策略
        reject(command);
}
```
