---
title: "MySQL 开发者快速上手 PostgreSQL（面试向）"
date: 2026-06-30
tags: ["数据库", "PostgreSQL", "MySQL", "面试"]
categories: ["学习笔记"]
summary: "已经懂 MySQL，如何用 3~5 天补齐 PostgreSQL 差异、达到面试水平——以差异和 Postgres 特有能力为主线的速通笔记"
draft: false
---

> 这是「学习笔记」系列的一篇。前提：你已经熟悉 MySQL。SQL 语法、索引、事务、范式这些通用知识 80% 可以直接迁移，真正要补的是 **PostgreSQL 与 MySQL 的差异** 以及 **Postgres 特有的东西**——而面试官最爱问的恰恰就是这些。

## 一、先建立心智模型：两者最根本的区别

花半天理解这几条，比刷十道题都管用——它们是后面所有差异的根源：

| 维度 | MySQL（InnoDB） | PostgreSQL |
|------|----------------|------------|
| 定位 | 偏"应用数据库"，追求简单快速 | 偏"对象-关系数据库"，追求标准与功能强大 |
| 并发控制 | MVCC + undo log（旧版本放 undo） | MVCC，但旧版本**直接留在表里**（堆表），靠 VACUUM 清理 |
| 主键存储 | **聚簇索引**（数据按主键物理排列） | **堆表**（数据无序堆放），所有索引都是"二级索引" |
| 类型系统 | 较宽松（默认会做隐式转换、静默截断） | 极严格，类型不匹配直接报错 |
| 扩展性 | 插件有限 | **扩展生态是核心卖点**（pgvector、PostGIS 等） |

一句话抓住精髓：**MySQL 像"够用就好的瑞士军刀"，Postgres 像"标准严谨、功能繁多的工程平台"。**

## 二、必须掌握的差异点（面试高频）

这些是面试官区分"真懂 Postgres"还是"只是会写 SQL"的题眼。

### 1. MVCC 与 VACUUM（最高频，必考）

- Postgres 的 `UPDATE`/`DELETE` 不是原地改，而是**标记旧行 + 插入新行**，旧行（dead tuple）留在表里。
- 所以需要 **VACUUM** 回收死元组、**autovacuum** 自动跑、**VACUUM FULL** 重建表。
- 衍生概念：**表膨胀（table bloat）**、事务 ID 回卷（**XID wraparound**）——能讲清这俩就是加分项。
- 对比 MySQL：InnoDB 的 purge 线程做类似的事，但机制不同。

### 2. 索引类型（Postgres 的强项）

- MySQL 主要就 B-Tree（加全文/空间）；Postgres 有 **B-Tree / Hash / GIN / GiST / BRIN**。
- 重点搞懂 **GIN**（用于 JSONB、数组、全文检索）——面试常问"JSONB 怎么建索引"。
- **部分索引（partial index）**、**表达式索引**——MySQL 没有，很能体现差异。

### 3. 数据类型

- **JSONB**（不是 JSON）——二进制存储、可建 GIN 索引，必问。
- **数组类型** `ARRAY`。
- `UUID`、枚举；`text`（Postgres 里 `varchar` 和 `text` 性能无差别，和 MySQL 习惯不同）。
- 时间类型 `timestamptz`（带时区）。

### 4. SQL 能力差异

- `RETURNING` 子句（`INSERT/UPDATE/DELETE ... RETURNING *`）——MySQL 8 之前没有。
- `ON CONFLICT ... DO UPDATE`（upsert，对应 MySQL 的 `ON DUPLICATE KEY UPDATE`）。
- **CTE / 递归 CTE**（`WITH RECURSIVE`）、**窗口函数**——两边都有但 Postgres 用得更普遍。
- `FOR UPDATE SKIP LOCKED`（工作队列模式的核心）。
- `LISTEN/NOTIFY`（MySQL 无对应物）。

### 5. 并发与锁

- 事务隔离级别（Postgres 默认 **Read Committed**，它的"可重复读"实现比 MySQL 更接近快照隔离）。
- 行锁、`SELECT FOR UPDATE`、咨询锁（advisory lock）。

### 6. 主键自增（小但常被问）

- MySQL：`AUTO_INCREMENT`
- Postgres：旧写法 `SERIAL`，新标准写法 `GENERATED ALWAYS AS IDENTITY`

## 三、运维/架构概念（中高级岗会问）

- **WAL**（Write-Ahead Log）——对应 MySQL 的 redo log，是复制和恢复的基础。
- **流复制**（streaming replication）、主从、逻辑复制。
- **连接模型**：Postgres 是**每连接一个进程**（不是线程），连接数一高就吃内存——必须配 **PgBouncer 连接池**。这是和 MySQL 很不同、面试爱问的点。
- `EXPLAIN ANALYZE` 读执行计划（和 MySQL 的 `EXPLAIN` 类似但输出格式不同）。
- `psql` 常用元命令：`\dt`（列表）、`\d 表名`（看结构）、`\di`（索引）。

## 四、动手计划（边学边练才记得住）

```bash
# 用 Docker 起一个，5 分钟搞定
docker run --name pg -e POSTGRES_PASSWORD=pass -p 5432:5432 -d postgres:17
docker exec -it pg psql -U postgres
```

然后亲手做这几件事（每件都对应一个面试考点）：

1. 建表时故意用 `SERIAL` 和 `GENERATED ... IDENTITY` 各来一次
2. 写一个 `INSERT ... ON CONFLICT DO UPDATE ... RETURNING *`
3. 建一张带 `JSONB` 列的表，给它建 GIN 索引，用 `@>`、`->>` 查询
4. `UPDATE` 一批数据后，用 `pg_stat_user_tables` 看 dead tuple，手动 `VACUUM`
5. 开两个 `psql` 窗口，演示 `SELECT FOR UPDATE SKIP LOCKED` 抢不同的行
6. `EXPLAIN ANALYZE` 一条查询，对比加索引前后

## 五、推荐资源（按性价比排序）

1. **官方文档的 "Tutorial" 和 "The SQL Language" 两章**——Postgres 官方文档质量极高，是业界标杆。
2. **《PostgreSQL 修炼之道》/《PostgreSQL 实战》**（中文）——系统补全。
3. 搜 **"PostgreSQL vs MySQL interview questions"**——直接刷差异题。
4. 读真实生产项目的 SQL——比看教程更有体感，能看到 `FOR UPDATE SKIP LOCKED`、JSONB、迁移写法等真实用法。

## 六、时间分配建议

SQL 基础扎实的话，**集中 3~5 天**足够应付大多数面试：

- **Day 1**：心智模型 + MVCC/VACUUM（吃透这个就赢一半）
- **Day 2**：索引类型（重点 GIN/JSONB）+ 数据类型
- **Day 3**：SQL 差异（RETURNING / upsert / CTE / 窗口函数）+ 动手练
- **Day 4**：并发锁 + WAL/复制 + 连接池
- **Day 5**：刷差异面试题 + 读真实项目 SQL

## 七、核心策略

面试官问 Postgres，十有八九想听的是"**它和 MySQL 哪里不一样、为什么**"。所以全程带着"这个在 MySQL 里是怎样的？Postgres 为什么这么设计？"的对比视角去学——这既是最快的学习路径，也正好是面试的最佳答法。

### 附：一张速查对照表

| 场景 | MySQL | PostgreSQL |
|------|-------|------------|
| 自增主键 | `AUTO_INCREMENT` | `GENERATED ALWAYS AS IDENTITY` |
| upsert | `ON DUPLICATE KEY UPDATE` | `ON CONFLICT DO UPDATE` |
| 取回受影响行 | （8.0 前无） | `... RETURNING *` |
| JSON 索引 | 有限 | `JSONB` + GIN 索引 |
| 大小写不敏感查询 | 默认排序规则不敏感 | 默认敏感，需 `ILIKE` 或 `citext` |
| 字符串类型 | `varchar(n)` 更省 | `text`/`varchar` 性能无差别 |
| 清理旧版本 | purge 线程 | `VACUUM` / autovacuum |
| 连接 | 线程 | 进程（需连接池） |
