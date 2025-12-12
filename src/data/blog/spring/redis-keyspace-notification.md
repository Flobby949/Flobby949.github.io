---
author: Flobby
pubDatetime: 2024-08-01T22:00:00Z
title: SpringBoot 整合 Redis 键空间通知实现设备上下线监控
slug: springboot-redis-keyspace-notification
featured: false
draft: false
tags:
  - Spring Boot
  - Redis
  - 实战
description: 基于 Redis 键空间通知实现设备上下线状态感知，构建统一的设备状态管理方案。
---

在当前业务场景中，平台接入了多种类型的设备，通信协议包括 MQTT、TCP 长连接及其他自定义协议。各类设备通过心跳机制上报状态，平台统一将其写入 Redis，用于标识设备是否在线。但目前缺乏一套完整的掉线检测机制，无法准确识别设备因断网、断电等原因导致的离线状态。因此，需要基于 Redis 的事件监听能力，构建一套统一、协议无关的设备上下线状态感知方案。

## 1. 概述

键空间通知是Redis提供的一种发布/订阅（pub/sub）机制，允许客户端订阅特定事件（如键被set、过期等），当这些事件发生时，Redis会向订阅者发送消息通知。

这种机制是**被动通知**，本质上并不是为高性能而设计，而是为了**事件驱动型应用**提供辅助支持，适合实现**缓存失效监听**、**自动刷新**、**延迟任务触发**等需求。

## 2. Redis的键通知机制

它允许客户端订阅特定的事件类型（如 `set`, `expired`, `del`），一旦 Redis 中发生了这些事件，便会通过 **虚拟通道（channel）** 推送通知给所有订阅者。

| 类型       | 通道格式                    | 描述                                                       |
| ---------- | --------------------------- | ---------------------------------------------------------- |
| 键空间通知 | `__keyspace@<db>__:<key>`   | 通知某个 key 上发生了什么操作，比如 `set`、`expire`、`del` |
| 键事件通知 | `__keyevent@<db>__:<event>` | 通知某个操作（如 `expired`）作用在哪个 key 上              |

## 3. 配置方式

默认情况下，Redis是**不启用**通知功能的，需要手动配置。

### 3.1 配置文件

找到配置文件`redis.conf`，找到或添加如下配置：

```ini
notify-keyspace-events ExgKs
```

### 3.2 命令配置

```shell
CONFIG SET notify-keyspace-events ExgKs
```

### 3.3 配置参数

| 字符 | 含义                                    |
| ---- | --------------------------------------- |
| `K`  | 启用键空间通知（Keyspace channel）      |
| `E`  | 启用键事件通知（Keyevent channel）      |
| `g`  | 通用命令（`del`, `expire`, `rename`）   |
| `s`  | 字符串类型命令（`set`, `append`, `incr`）|
| `x`  | 过期事件（`expired`）                   |
| `e`  | 淘汰事件（`evicted`，因内存不足被清理） |
| `A`  | 所有事件的组合                          |

**示例配置组合**

| 配置值  | 说明                                         |
| ------- | -------------------------------------------- |
| `KEx`   | 监听过期事件，包含键空间与键事件通道         |
| `ExgKs` | 监听 set 操作 + 过期事件，推荐用于缓存场景   |

## 4. 测试

### 4.1 配置监听

```bash
CONFIG SET notify-keyspace-events KEA
```

### 4.2 开启监听

监听`键空间通知`

```bash
PSUBSCRIBE '__keyspace@0__:*'
```

### 4.3 测试

开启两个redis-cli，一个监听事件，一个执行命令，可以看到事件被正确触发。

![Redis测试1](/images/blog/redis-keyspace-1.png)

![Redis测试2](/images/blog/redis-keyspace-2.png)

## 5. SpringBoot 接入

### 5.1 接入Redis

1. 添加Redis依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

2. yml 配置

```yaml
spring:
  redis:
    host: 127.0.0.1
    password:
    port: 6379
    database: 0
```

### 5.2 SET 事件监听器

监听无法指定key，所以需要我们自己在监听器中进行过滤

```java
@Slf4j
@Component
@AllArgsConstructor
public class RedisKeySetListener implements MessageListener {
    private final RedisCache redisCache;

    @Override
    public void onMessage(Message message, byte[] pattern) {
        String redisKey = message.toString();
        log.info("Redis set 事件触发：{}", redisKey);

        if (redisKey.startsWith("device:online:")) {
            long ttl = redisCache.ttl(redisKey);
            log.info("Redis set 事件触发：{} --> ttl:{}", redisKey, ttl);
            if (ttl == -1) {
              // 首次创建，进行业务处理，例如设备上线通知
            }
        }
    }
}
```

### 5.3 过期事件监听

```java
@Slf4j
@Component
public class RedisKeyExpiredListener implements MessageListener {

    @Override
    public void onMessage(Message message, byte[] pattern) {
        String expiredKey = message.toString();

        if (expiredKey.startsWith("device:online:")) {
            // TODO 下线通知、记录日志
        }
    }
}
```

### 5.4 创建 Redis 配置

```java
@Slf4j
@Configuration
public class RedisConfig {

    @Value("${spring.redis.database:0}")
    private int redisDb;

    @Bean
    public RedisMessageListenerContainer redisMessageListenerContainer(
            RedisConnectionFactory redisConnectionFactory,
            MessageListenerAdapter keyExpiredListenerAdapter,
            MessageListenerAdapter setEventListenerAdapter
    ) {
        RedisMessageListenerContainer container = new RedisMessageListenerContainer();
        container.setConnectionFactory(redisConnectionFactory);

        // 监听过期事件
        container.addMessageListener(keyExpiredListenerAdapter,
            new PatternTopic("__keyevent@" + redisDb + "__:expired"));

        // 监听 set 事件
        container.addMessageListener(setEventListenerAdapter,
            new PatternTopic("__keyevent@" + redisDb + "__:set"));

        return container;
    }

    @Bean
    public MessageListenerAdapter keyExpiredListenerAdapter(RedisKeyExpiredListener listener) {
        return new MessageListenerAdapter(listener, "onMessage");
    }

    @Bean
    public MessageListenerAdapter setEventListenerAdapter(RedisKeySetListener listener) {
        return new MessageListenerAdapter(listener, "onMessage");
    }
}
```

通过以上配置，我们就实现了基于 Redis 键空间通知的设备上下线监控方案。当设备心跳写入 Redis 时触发上线逻辑，当 Redis key 过期时触发下线逻辑。
