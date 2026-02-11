---
author: Flobby
pubDatetime: 2026-02-11T12:00:00Z
title: 深入理解 MySQL 索引：从 B+ 树到索引下推
slug: mysql-index-deep-dive
featured: true
draft: false
tags:
  - MySQL
  - 数据库
  - 索引优化
description: 以一张 1.6 亿行的真实生产表为例，从 B+ 树的物理结构出发，逐步拆解聚簇索引、二级索引、覆盖索引、最左匹配原则和索引下推（ICP）的工作原理与优化实践。
---

本文以一张 **1.6 亿行**的真实生产表 `t_mqtt_log` 为例，从 B+ 树的物理结构出发，逐步拆解 MySQL InnoDB 索引体系的核心知识点。

示例表结构：

```sql
CREATE TABLE `t_mqtt_log` (
  `pk_id` int NOT NULL AUTO_INCREMENT,
  `topic` varchar(255) NOT NULL COMMENT '话题',
  `message` longtext NOT NULL COMMENT '消息',
  `type` tinyint NOT NULL COMMENT '0-发送，1-接收',
  `del_flag` tinyint DEFAULT NULL,
  `update_time` timestamp NULL DEFAULT NULL,
  `create_time` timestamp NULL DEFAULT NULL,
  PRIMARY KEY (`pk_id`),
  KEY `topic` (`topic`),
  KEY `type` (`type`)
) ENGINE=InnoDB AUTO_INCREMENT=93121473 DEFAULT CHARSET=utf8mb4;
```

## B+ 树的"目录"存了什么

在 InnoDB 中，B+ 树的非叶子节点（目录节点）只存两样东西：**索引键值（Key）** 和 **页号（Page Number）**。

每一页（Page）默认 **16KB**。

- **索引键值**：你建立索引的那个字段的值。比如按 `pk_id` 建索引，这里存的就是 `pk_id` 的数字。
- **页号**：一个指针，指向下一层级某个 Page 的物理地址。

### 三层 B+ 树的查找过程

想象一个 3 层的 B+ 树：

1. **根节点（顶级目录）**：存的是 `(ID=1, 指向 A 页)`、`(ID=500, 指向 B 页)`。
2. **中间目录**：A 页里存的是 `(ID=1, 指向 C 页)`、`(ID=100, 指向 D 页)`。
3. **叶子节点（真实数据）**：C 页里存的就是 ID 从 1 到 99 的**完整行数据**。

### 为什么目录页不存真实数据？

这是 B+ 树优于 B 树的核心原因：

- **空间压缩**：目录页不存具体的行数据（用户名、地址等），所以非常"苗条"。
- **扇出（Fan-out）极大**：一个 16KB 的目录页可以存放成百上千个"键值 + 指针"对。

**速算一下**：假设主键是 8 字节的 `BIGINT`，指针是 6 字节，一共 14 字节。一个 16KB 的页面大约能存：

$$16 \times 1024 / 14 \approx 1170 \text{ 个目录项}$$

根节点翻一次，就能从 1170 个子页面里选一个；再翻一次，就是 $1170 \times 1170 \approx 137$ 万个页面。**这就是为什么 B+ 树只需要 3 层就能撑起千万级数据查询。**

你可以通过以下 SQL 观察索引的空间占用：

```sql
SHOW TABLE STATUS LIKE 't_mqtt_log';
```

## 聚簇索引与二级索引

通过以下 SQL 可以查看每个索引的实际大小（MySQL 8.0+）：

```sql
SELECT
    index_name,
    stat_value AS pages,
    stat_value * @@innodb_page_size / 1024 / 1024 AS size_mb
FROM mysql.innodb_index_stats
WHERE table_name = 't_mqtt_log' AND stat_name = 'size';
```

在我们的生产表上，结果大致如下：

| 索引名 | 大小 | 说明 |
|--------|------|------|
| PRIMARY | ~27 GB | 聚簇索引，包含全部行数据 |
| topic | ~2.5 GB | 二级索引，只存 topic + pk_id |
| type | ~900 MB | 二级索引，只存 type + pk_id |

### 为什么 PRIMARY 这么大（27GB）？

在 InnoDB 中，**主键索引就是表本身**。它的叶子节点存放的是**整行记录**（包括 `message` 大文本、`create_time` 等所有字段）。27GB 的数据实打实地长在 `PRIMARY` 这棵 B+ 树上。

### 为什么 topic 索引只有 2.5GB？

`topic` 是一棵**二级索引**（辅助索引），叶子节点只存两样东西：

1. `topic` 字段的值
2. 对应的主键 `pk_id`

当你通过 `topic` 查询时，MySQL 先在这棵小树上找到 `pk_id`，再回到 `PRIMARY` 大树上找完整数据——这就是**回表**。

### 前缀索引：给 topic 索引"瘦身"

1.6 亿数据，`topic` 索引占了 2.5GB。如果你的 MQTT Topic 都类似 `sys/sensor/data/1001`、`sys/sensor/data/1002`，前 11 个字符 `sys/sensor/` 全是重复的，在 B+ 树目录页里存这些重复字符极其浪费。

**用 SQL 测量区分度**：

```sql
SELECT
    COUNT(DISTINCT LEFT(topic, 10)) / COUNT(*) AS sel10,
    COUNT(DISTINCT LEFT(topic, 15)) / COUNT(*) AS sel15,
    COUNT(DISTINCT topic) / COUNT(*) AS total_sel
FROM t_mqtt_log;
```

- 如果 `sel15` 和 `total_sel` 非常接近（都达到 0.9 以上），说明前 15 个字符就能代表这个 Topic。
- 优化操作：`ALTER TABLE t_mqtt_log DROP INDEX topic, ADD KEY topic (topic(15));`
- 收益：索引体积可能从 2.5GB 缩减到 1GB 甚至更小。

但在我们的实际数据中，结果是 `0.0000 | 0.0000 | 0.0006`——前缀索引完全无效，整体区分度也极低。每个不同的 Topic 平均有约 $1 / 0.0006 \approx 1666$ 条重复记录。

## 索引查询速度与选择性

### 主键索引：一步到位

主键索引是聚簇索引，数据就长在树上：

- 查到 ID → 直接拿到数据
- 代价：$O(\log N)$ 次磁盘 I/O
- 没有任何多余动作

### 二级索引：回表代价与选择性

命中行数越少，效率越高——这在数据库里叫**选择性（Selectivity）**。

以 `type = 1` 查询为例，假设一半数据是 `type = 1`：

1. 在 `type` 索引树里找到所有 `type = 1` 的 `pk_id`——**8000 万个 ID**
2. 拿着 8000 万个 ID 去主键索引里找完整行数据——**8000 万次随机 I/O**
3. MySQL 优化器发现回表 8000 万次比全表扫描还慢，于是**放弃索引，直接全表扫描**

**这就是为什么低区分度的 `type` 索引在数据量大到一定程度后会失效。**

### 覆盖索引：避免回表的杀招

只要 B+ 树的叶子节点里已经包含了 `SELECT` 需要的所有字段，MySQL 就不需要回表。

二级索引的叶子节点包含：索引字段值 + 主键 ID。

```sql
-- 覆盖索引：树里全都有
SELECT pk_id FROM t_mqtt_log WHERE topic = 'A';
SELECT topic FROM t_mqtt_log WHERE topic = 'A';

-- 不是覆盖索引：message 在主键树里，得回表
SELECT message FROM t_mqtt_log WHERE topic = 'A';
```

## 最左匹配原则

### 前缀索引的教训

回到我们的区分度数据 `0.0000`，它告诉我们：**在当前 `topic` 结构下，前缀索引不仅没用，反而是自寻死路。**

前缀索引的初衷是给 B+ 树"瘦身"：

- 不带前缀：`topic` 平均 100 字节，一个目录页只能存约 150 个键值对
- 带前缀（10 字节）：一个目录页能存约 1000 个键值对
- 树高从 4 层降到 3 层，少一次磁盘 I/O

但当区分度为 0 时，前缀索引会导致 MySQL 匹配到 1.6 亿行，然后疯狂回表——比全表扫描慢无数倍。

**前缀索引还有两个致命副作用**：

1. **无法使用覆盖索引**：即使只查 `topic`，MySQL 也必须回表，因为索引树里只存了前缀，无法确认后面的部分是否匹配。
2. **无法用于排序**：`ORDER BY topic` 无法利用前缀索引，因为前缀有序不代表整个字符串有序。

### 从前缀索引到复合索引

前缀索引是"单列版"的最左匹配，最左匹配原则是"多列版"的前缀逻辑。B+ 树排列数据时都遵循同一个准则：**从左往右排**。

- **前缀索引**：`topic` 的前 15 位做索引，B+ 树先按第 1 个字符排，相同则按第 2 个排……
- **复合索引 `(type, topic)`**：B+ 树先按 `type` 排，`type` 相同再按 `topic` 排

如果你建了 `(A, B, C)` 的索引：

- 查询有 `A` → 能用索引
- 跳过 `A` 直接查 `B` → 索引失效
- `LIKE 'sys/sensor%'` → 能用索引（从左开始匹配）
- `LIKE '%sensor'` → 索引失效（不是从左开始）

## 索引下推（Index Condition Pushdown）

MySQL 5.6 引入的核心优化，专门解决**频繁回表导致的磁盘 I/O 爆炸**。

假设有复合索引 `INDEX(topic, type)`，执行：

```sql
SELECT * FROM t_mqtt_log WHERE topic LIKE 'iot/sensor/%' AND type = 1;
```

### 没有 ICP 的时代（MySQL 5.6 之前）

1. **存储引擎层**：用最左匹配找到所有 `topic` 以 `iot/sensor/` 开头的记录，假设 **100 万条**
2. **回表**：把 100 万个主键 ID 对应的整行记录全部从磁盘读出，交给 Server 层
3. **Server 层过滤**：逐行检查 `type` 是否等于 1，最后只有 100 条符合
4. **结论**：为了 100 条数据，做了 **99.99 万次**多余的回表

### 有了 ICP 之后

1. **存储引擎层**：同样找到 100 万条 `topic` 匹配的记录
2. **原地判断**：虽然 `LIKE` 范围查询导致 `type` 无法走索引排序，但 **`type` 的值就在复合索引的叶子节点里**
3. **过滤**：存储引擎直接在 B+ 树里判断 `type = 1`，只有符合的 100 条才回表
4. **结论**：磁盘 I/O 从 100 万次骤减到 100 次，效率提升 **10000 倍**

> 以前就像快递员只看地址就把 100 万个包裹搬到你家，让你自己拆开看哪个是你买的；现在是你告诉快递员："地址是这一片的，且包裹上贴了红色标签的才给我送过来"。

## 总结

| 知识点 | 核心要点 |
|--------|---------|
| B+ 树目录 | 只存键值 + 页号，扇出大，3 层撑千万级数据 |
| 聚簇索引 | 主键索引 = 表本身，叶子节点存完整行 |
| 二级索引 | 只存索引字段 + 主键，查完整数据需回表 |
| 覆盖索引 | SELECT 的字段都在索引里，免回表 |
| 前缀索引 | 低区分度时无效，且无法覆盖索引、无法排序 |
| 最左匹配 | 查询必须从索引最左列开始，否则索引失效 |
| 索引下推 | 存储引擎层提前过滤，大幅减少回表次数 |
