---
author: Flobby
pubDatetime: 2026-01-08T03:00:00Z
title: 一键开启虚拟线程后更慢了？JDK21 Pinning 排查全流程与工程化修复
slug: virtual-thread-pinning
featured: true
draft: false
tags:
  - Java
  - JDK21
  - 虚拟线程
  - 并发编程
  - 性能优化
description: 一个真实的JDK21虚拟线程生产事故的完整排查过程，深入剖析虚拟线程Pinning机制及解决方案。
---

一个真实的JDK21虚拟线程生产事故的完整排查过程。

## 一、项目背景：从JDK8到JDK21的技术升级

### 1.1 升级前的项目状态

- 运行环境：JDK 8 + Spring Boot 2.x
- 线程模型：传统平台线程池

### 1.2 为什么要升级到JDK21？

- Java 21 LTS版本，长期支持
- 虚拟线程（Project Loom）
  - 承诺轻量级协程，百万级并发
  - 简化异步编程，无需回调地狱
  - Spring Boot 3.2+ 原生支持
- Spring AI 加持

### 1.3 升级计划

```yaml
# 升级内容
JDK: 8 → 21
Spring Boot: 2.7 → 3.2+
改造目标: 启用虚拟线程，尽量替换自建线程池
```

## 二、改造

### 2.1 Spring Boot虚拟线程配置

这个配置会让 Spring Boot 在合适的位置自动使用虚拟线程执行器（例如默认的异步执行器等），是官方推荐的启用方式之一

```yaml
spring:
  threads:
    virtual:
      enabled: true  # 一键启用虚拟线程
```

### 2.2 线程池改造（踩坑伏笔）

```java
// 改造前：平台线程池
private static final ExecutorService THREAD_POOL =
    Executors.newFixedThreadPool(10);

// 改造后：虚拟线程池
private static final ExecutorService THREAD_POOL =
    Executors.newVirtualThreadPerTaskExecutor();
```

### 2.3 预期效果 vs 实际结果

| 预期               | 实际             |
| ------------------ | ---------------- |
| 🎯 并发能力提升10倍 | ❌ 吞吐量反而下降 |
| 🎯 资源占用减少     | ❌ CPU占用飙升    |

## 三、事故现场：CPU飙升事件

### 3.1 现象

- 现象
  - CPU使用率从20% → 700%+
  - 请求开始超时，P99 拉长
  - 服务器内存占用异常增长

### 3.2 初步排查方向

```
怀疑点1: 是否有死循环？ → 代码审查未发现
怀疑点2: 数据库连接泄漏？ → 连接池正常
怀疑点3: 外部API拖累？ → 上游服务正常
怀疑点4: JVM配置问题？ → 参数未变化
```

### 3.3 监控数据异常

- **线程数**：从100个平台线程 → 800+载体线程
- **GC频率**：从每分钟5次 → 每分钟30次
- **堆内存**：频繁Full GC，接近OOM边缘

## 四、问题排查：火焰图揭示性能黑洞

### 4.1 性能分析工具选择

+ async-profiler 抓 CPU 火焰图
+ arthas 快速定位热点与调用链

### 4.2 火焰图分析结果

![image-20260108103939473](https://www.flobby.top/imagebed/i/1767839984261453973_HZYC0oGW.png)

**定位到关键热点**：

代码中的`CircularFifoQueue.contains()`是最最最宽的平顶山

### 4.3 启用虚拟线程诊断

```bash
# 添加JVM参数，监控Pinning事件
-Djdk.tracePinnedThreads=full
```

**控制台输出（触目惊心）**：

```
VirtualThread[#156] pinned for 0.003s
    at CircularFifoQueue.contains(...)
    at EzvizWebhook.processMessage(EzvizWebhook.java:104)
    [repeated 2000+ times in 1 minute]
```

**问题确认**：虚拟线程**Pinning（钉死）**导致性能灾难！

## 五、深入剖析：虚拟线程Pinning机制

### 5.1 虚拟线程的正常工作原理

```
[虚拟线程A] --阻塞--> 卸载 ---> [载体线程空闲]
[虚拟线程B] <--挂载-- 调度 <--- [载体线程执行]

优势：8个载体线程 → 支撑10万虚拟线程
```

### 5.2 什么是Pinning（钉死）？

```
[虚拟线程A] --阻塞--> ❌无法卸载❌ ---> [载体线程被占用]
                                      其他虚拟线程无法使用
```

**Pinning的本质**：虚拟线程无法从载体线程卸载，导致载体线程被独占。

### 5.3 导致Pinning的两大元凶

1. **在`synchronized`块内阻塞**
2. **执行native方法时阻塞**

### 5.4 为什么Pinning会导致性能灾难？

| 正常调度                     | Pinning状态                    |
| ---------------------------- | ------------------------------ |
| 8载体线程 → 10万虚拟线程并发 | 8载体线程 → 仅8个虚拟线程并发  |
| 载体线程利用率100%           | 载体线程利用率100%（但浪费）   |
| 吞吐量高                     | 吞吐量暴跌，性能不如平台线程池 |

## 六、案例分析与解决方案

### 案例1: 高频短时Pinning - 性能黑洞

#### 6.1 问题代码分析

```java
// 第43行：数据结构选择不当
private static final CircularFifoQueue<String> MESSAGE_UNIQUE_QUEUE =
    new CircularFifoQueue<>(20);

// 第104行：CPU杀手（火焰图热点）
if (!MESSAGE_UNIQUE_QUEUE.contains(messageId)) {  // ⚠️ 问题点
    MESSAGE_UNIQUE_QUEUE.add(messageId);
    // 处理业务逻辑
}
```

#### 6.2 三重致命缺陷

**缺陷1：O(n)时间复杂度**

```java
// CircularFifoQueue源码
public synchronized boolean contains(Object o) {
    for (int i = 0; i < size; i++) {  // 线性遍历
        if (elements[i].equals(o)) return true;
    }
    return false;
}
```

- 每次webhook请求都要遍历整个队列
- 调用频率：200次/秒
- 单次耗时：0.1-1ms
- **累积CPU消耗**：200 × 0.5ms = 100ms/秒 → 10% CPU

**缺陷2：synchronized锁竞争**

```java
public synchronized boolean contains(Object o) { }
public synchronized boolean add(Object o) { }
```

- 高并发下锁竞争
- 导致虚拟线程排队等待

**缺陷3：虚拟线程Pinning放大器**

```
萤石webhook: 200请求/秒

时间轴推演：
0-1秒：
  ├─ 创建200个虚拟线程
  ├─ 全部在contains()处pinning (synchronized块内)
  └─ JVM被迫创建200个载体线程

1-2秒：
  ├─ 又来200请求 → 再创建200虚拟线程
  ├─ synchronized锁竞争，前200个未处理完
  └─ 载体线程累积至400+

2-3秒：
  ├─ 载体线程突破1000+
  ├─ 内存：1000线程 × 1MB栈 = 1GB+
  ├─ CPU: 100% (锁竞争 + 上下文切换)
  └─ 💥 系统崩溃
```

#### 6.3 解决方案：Caffeine Cache替换

```java
// ✅ 替换为无锁、O(1)的缓存
private static final Cache<String, Boolean> MESSAGE_UNIQUE_CACHE =
    Caffeine.newBuilder()
        .maximumSize(1000)          // 容量提升50倍
        .expireAfterWrite(5, TimeUnit.MINUTES)
        .build();

// ✅ 修改后：O(1)查找 + 无锁
if (MESSAGE_UNIQUE_CACHE.getIfPresent(messageId) == null) {
    MESSAGE_UNIQUE_CACHE.put(messageId, Boolean.TRUE);
    // 业务逻辑
}
```

#### 6.4 优化效果对比

| 指标        | CircularFifoQueue | Caffeine Cache | 提升       |
| ----------- | ----------------- | -------------- | ---------- |
| 时间复杂度  | O(n) 线性         | O(1) 哈希      | ⚡ 常量时间 |
| 线程安全    | synchronized      | 无锁CAS        | 🔓 消除竞争 |
| 容量        | 20                | 1000           | 📈 50倍     |
| Pinning风险 | ❌ 会pin           | ✅ 不会pin      | 🎯 根除     |
| CPU占用     | 45%               | <1%            | 🚀 45倍提升 |

### 案例2: 低频长时Pinning - 同步等待

#### 6.5 问题代码分析

```java
public static Object sendSyncMessage(String ctxId, String requestId, Object msg) {
    CompletableFuture<Object> future = createFuture(requestId);
    ……

    // ⚠️ 致命：阻塞10秒，pin住载体线程
    return future.get(10, TimeUnit.SECONDS);
}
```

#### 6.6 并发崩溃推演

**场景**：8核CPU服务器（载体线程数=8），100个并发请求

```
传统平台线程池（100线程）：
├─ 100个请求同时处理
├─ 每个阻塞10秒
└─ 总耗时：10秒 ✅

虚拟线程（有Pinning）：
├─ 前8个请求占满载体线程，pin住10秒
├─ 剩余92个请求排队等待
├─ 10秒后处理下一批8个
├─ 需要：100 ÷ 8 = 13批
└─ 总耗时：13 × 10 = 130秒 ❌
```

**结论**：虚拟线程遇到长时间Pinning，性能反而**不如**传统线程池！

#### 6.7 解决方案：平台线程池隔离

**核心思路**：将阻塞操作委托给专用平台线程池，避免pin虚拟线程。

```java
// ✅ 创建专用平台线程池（用于阻塞操作）
private static final ExecutorService BLOCKING_EXECUTOR =
    Executors.newFixedThreadPool(10, r -> {
        Thread t = new Thread(r, "netty-sync-blocking");
        t.setDaemon(true);
        return t;
    });

// ✅ 修改后：阻塞在平台线程，虚拟线程立即释放
public static Object sendSyncMessage(String ctxId, String requestId, Object msg) {
    CompletableFuture<Object> future = createFuture(requestId);
    ……
    // 关键：在平台线程中执行阻塞等待
    return CompletableFuture.supplyAsync(() -> {
        try {
            return future.get(10, TimeUnit.SECONDS);
        } catch (TimeoutException e) {
            throw new CompletionException(new ServerException("同步消息超时"));
        } catch (Exception e) {
            throw new CompletionException(e);
        }
    }, BLOCKING_EXECUTOR).join();  // 虚拟线程在join时不会pin
}
```

## 七、经验总结与最佳实践

### 7.1 核心教训

```
教训1：虚拟线程 ≠ 性能银弹
├─ 必须理解Pinning机制
├─ 必须识别代码中的阻塞点
└─ 必须评估是否真的适合虚拟线程

教训2：synchronized是隐形杀手
├─ 短时锁 + 高频调用 = CPU热点
├─ 长时锁 + 阻塞 = 吞吐量崩溃
└─ 虚拟线程放大了这些问题

教训3：监控是生命线
├─ 必须启用-Djdk.tracePinnedThreads
├─ 必须定期抓取火焰图
└─ 必须建立性能基线对比
```

### 7.2 Pinning风险等级分类

| 类型       | Pin时长        | 调用频率     | 风险等级 | 处理策略                  |
| ---------- | -------------- | ------------ | -------- | ------------------------- |
| 长时间阻塞 | 秒级(10-30s)   | 低频         | 🔴 极高   | 必须使用平台线程池隔离    |
| 短时间锁   | 毫秒级(1-10ms) | 高频(100+/s) | 🟠 高     | 必须替换数据结构/无锁算法 |
| 瞬时锁     | 微秒级(<1ms)   | 高频         | 🟡 中     | 监控，考虑优化            |
| 瞬时锁     | 微秒级(<1ms)   | 低频         | 🟢 低     | 可接受，如HikariCP        |

### 7.3 虚拟线程适用场景判断

| 场景                 | 是否适合   | 原因                 | 示例               |
| -------------------- | ---------- | -------------------- | ------------------ |
| 纯异步IO             | ✅ 强烈推荐 | 无阻塞，完美发挥优势 | HTTP、数据库查询   |
| 偶尔同步调用(<100ms) | ✅ 可以使用 | 配合平台线程池隔离   | RPC调用、缓存查询  |
| 频繁synchronized     | ❌ 不推荐   | 产生大量pinning      | 集合遍历、锁竞争   |
| 长时间阻塞(>1s)      | ❌ 不推荐   | 性能不如传统线程池   | 同步消息等待       |
| CPU密集型计算        | ❌ 无意义   | 虚拟线程优势在IO     | 图像处理、加密计算 |

### 7.4 代码审查清单

**🔍 必须排查的危险模式**：

```java
// ❌ 反模式1：虚拟线程中使用future.get()
CompletableFuture<String> future = asyncCall();
String result = future.get(10, TimeUnit.SECONDS);  // 危险！

// ✅ 推荐：委托给平台线程池
CompletableFuture.supplyAsync(() ->
    future.get(10, TimeUnit.SECONDS), platformThreadPool
).join();


// ❌ 反模式2：synchronized + 集合遍历
synchronized (list) {
    for (String item : list) {  // O(n) + 锁 = 灾难
        if (item.equals(target)) return true;
    }
}

// ✅ 推荐：无锁数据结构
ConcurrentHashMap<String, Boolean> set = new ConcurrentHashMap<>();
set.putIfAbsent(target, Boolean.TRUE);  // O(1) + 无锁


// ❌ 反模式3：synchronized + 循环等待
synchronized (monitor) {
    while (!condition) {
        monitor.wait(1000);  // 长时间pin
    }
}

// ✅ 推荐：使用显式锁
Lock lock = new ReentrantLock();
Condition condition = lock.newCondition();
lock.lock();
try {
    condition.await(1, TimeUnit.SECONDS);
} finally {
    lock.unlock();
}


// ❌ 反模式4：循环调用synchronized方法
for (int i = 0; i < 1000; i++) {
    syncMethod();  // 每次都pin
}

// ✅ 推荐：批量处理或异步
List<Task> tasks = prepareTasks();
asyncBatchProcess(tasks);  // 一次性处理
```

### 7.5 虚拟线程配置建议

```yaml
# application.yml
spring:
  threads:
    virtual:
      enabled: true

# JVM参数
-Djdk.virtualThreadScheduler.parallelism=16    # 载体线程数 = CPU核数 * 2
-Djdk.virtualThreadScheduler.maxPoolSize=256   # 最大载体线程数
-Djdk.tracePinnedThreads=short                 # 生产环境监控pinning
-XX:+UseZGC                                    # 推荐ZGC，低延迟GC
```

### 7.6 监控策略

```bash
# 启动时监控
-Djdk.tracePinnedThreads=full  # 开发环境用full，生产环境用short

# 火焰图抓取
```

### 7.7 问题解决决策树

```
发现阻塞操作？
│
├─ 能否改为异步？
│   └─ 是 → CompletableFuture、响应式编程 ✅ 最优方案
│
├─ 必须同步等待？
│   ├─ 阻塞时间 > 100ms？
│   │   └─ 是 → 平台线程池隔离 ✅ 推荐
│   │
│   └─ 阻塞时间 < 100ms？
│       ├─ 调用频率低（< 10/s）？
│       │   └─ 是 → 可接受，监控即可 ⚠️
│       │
│       └─ 调用频率高（> 100/s）？
│           └─ 必须优化：替换数据结构/无锁算法 🔴
│
└─ 使用synchronized？
    └─ 替换为ReentrantLock、Caffeine、ConcurrentHashMap ✅
```

## 八、总结

### 8.1 虚拟线程不是银弹

```
虚拟线程的适用场景：
✅ IO密集型应用（HTTP、数据库、消息队列）
✅ 高并发长连接（WebSocket、SSE）
✅ 异步编程简化

虚拟线程不适合：
❌ CPU密集型计算
❌ 大量synchronized代码
❌ 长时间阻塞等待
```

### 8.2 Java生态的适配进度

| 框架/库          | 虚拟线程支持  | 备注             |
| ---------------- | ------------- | ---------------- |
| Spring Boot 3.2+ | ✅ 原生支持    | 需要配置启用     |
| Tomcat 10.1+     | ✅ 支持        | 自动适配         |
| Netty            | ⚠️ 部分支持    | 某些场景会pin    |
| HikariCP         | ⚠️ 可用但会pin | 影响可控         |
| Jedis            | ❌ 不推荐      | 大量synchronized |
| Lettuce          | ✅ 推荐        | 异步客户端       |
