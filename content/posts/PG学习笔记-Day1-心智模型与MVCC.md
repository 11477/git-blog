---
title: "PG 学习笔记 Day 1：平衡之争——Postgres 用堆表+死亡行偷懒，用 VACUUM 还债"
date: 2026-06-30
tags: ["数据库", "PostgreSQL", "MVCC", "面试"]
categories: ["学习笔记"]
summary: "MySQL 开发者上手 PG 的 Day 1：不满足于'PG 用 VACUUM 而 MySQL 不用'的表面结论，深入到磁盘布局、可见性判断算法、HOT chain 指针结构、freeze 的冻结菊花链——把 MVCC 这块彻底讲透"
draft: true
---

> 这是「MySQL 开发者快速上手 PostgreSQL」5 天笔记的 **Day 1**。今天只讲一个主题：Postgres 的 MVCC 实现——但不是在概念层面讲，而是推到**行在磁盘上到底怎么布局、可见性怎么逐行判断、VACUUM 怎么遍历页面清理**的深度。你用 MySQL InnoDB 的 undo-rollback 体系做参照系，帮你看清 PG 拿什么代价换回了什么。

## 一、用 MySQL 的命题，给 PG 开题

MySQL InnoDB 实现 MVCC 的路径可以概括为一句话：**每个事务反向拼装一个专属的"历史快照"**。

具体怎么做：

```
InnoDB 行（COMPACT 格式）
┌───┬───┬──────┬──────┬──────────┬─────────┬────────────┐
│  变长   │ NULL │ 记录头  │ trx_id │ roll_ptr │ row_id │ 列1 │ 列2 │ ...
│  字段长度 │ 位图  │ (5B)   │  (6B)  │  (7B)    │ (6B)  │     │     │
└───────────┴──────┴──────┴──────────┴─────────┴───────┴─────┴─────┘
```

- `trx_id`（6 字节）：最后修改此行的事务 ID
- `roll_ptr`（7 字节）：指向 **undo log**，undo log 里存着旧版本数据。这是个**物理指针**，不在数据页内。

修改发生时，InnoDB **原地改新值**，把旧值推到单独的 undo 段。要读旧版本？顺着 `roll_ptr` 链回 undo log 逐个版本还原。代价：回滚要把整个 undo 链逆向执行一遍——大事务回滚极慢。好处：数据页里始终只有"当前版本"最肥，历史版本在独立空间。

---

**PG 选的另一条路径**：把"旧版本"留在表数据页里不删。一个 UPDATE 动作产生两行：旧行存在但被标记为已删除（xmax 打上事务 ID），新行插在原位置或 nearby。历史版本不需要单独空间，它就在数据文件里，等你扫描。

```
PG 行（HeapTupleHeader + 数据）
┌──────────────┬──────┬──────┬──────┬──────┬─────────┬───────┐
│ t_xmin (4B)  │ t_xmax (4B) │ t_cid (4B) │ t_ctid (6B) │ ... │ 用户列 │
└──────────────┴──────┴──────┴──────┴─────────┴───────┘
```

- `t_xmin`：插入此行的事务 ID
- `t_xmax`：删除此行的事务 ID（未删除 = 0）
- `t_cid`：事务内命令 ID（同一事务多条命令的可见性判断用）
- `t_ctid`：当前版本的物理位置 (页号, 偏移)，**也被用作版本链指针**——旧行的 `t_ctid` 指向新的同逻辑行

**这个差异决定了他们后续所有运维形态的分岔**：

- MySQL 的 undo log 在独立空间，旧版本清理靠后台 **purge 线程** 异步做——对用户透明，不需要 SQL 层命令维护
- PG 的旧版本就在数据页里，清理必须**有人显式扫描、判断、回收**——这就是 **VACUUM**

现在你可以建立这个核心认知：**PG 的 MVCC 把"读"和"回滚"都做简单了，代价是"写数据页"会留下垃圾，需要定期打扫。** 要么你接受这个工程 trade-off 为你能力的一部分，PG 的运维水平永远入门级。

## 二、可见性判断——同一问题，两种答案

你知道 SQL 标准里的隔离级别定义（read committed vs repeatable read vs serializable）是概念性的，怎么落到"哪些行对我可见"需要一个确定性算法。

### 2.1 MySQL 的 ReadView

InnoDB 用一个 `ReadView` 结构来做一次性快照：

```
ReadView {
    m_low_limit_id;   // 下一个即将分配的事务 ID（>= 此值的未开始，一定不可见）
    m_up_limit_id;    // 当前活跃事务中最小的事务 ID（< 此值的已提交，一定可见）
    m_creator_trx_id; // 创建此快照的事务 ID（自己改的当然可见）
    m_ids;            // 此快照创建时的活跃事务列表
}
```

对于行上的 `trx_id`，判断逻辑：

```
if (trx_id == m_creator_trx_id) → 可见（自己改的）
elif (trx_id < m_up_limit_id)   → 可见（快照前已提交）
elif (trx_id >= m_low_limit_id) → 不可见（快照后才开始）
else → 在 m_ids 里？是→不可见（活跃未提交） 否→可见（快照前已提交但>=up_limit）
```

对于 MySQL 的 READ COMMITTED，**每次语句重新构造 ReadView**。对于 REPEATABLE READ，**事务第一条语句构造一次**，之后复用。

### 2.2 PG 的事务快照

PG 的所谓"快照"用 `SnapshotData` 结构（源码 `src/include/utils/snapshot.h`），核心成员：

```
SnapshotData {
    xmin;       // 小于此值的事务均已提交 → 此行一定可见
    xmax;       // 大于等于此值的事务未开始 → 此行一定不可见（除了未来可能需要）
    xip[];      // 快照时刻正在运行的事务 ID 列表（in-progress）
    subxip[];   // 子事务同理
}
```

判断一行是否可见需将 `xmin/xmax` 和快照对比：

```
if (xmin 还没提交 且 xmin != my_xid)           → 不可见（插入者还在跑）
elif (xmax 已提交 或 == my_xid 且 xmax in xip) → 已删除，不可见
else                                           → 可见
```

**与 MySQL ReadView 的关键差异**：PG 的快照不是"上下限区间"为主，是**活跃事务集合**为主。PG 的 `xmin` 只是"还能有效的最老事务"，它不用作区间下界做 `< min visible` 的快捷判断，而是要查 `xip[]` 逐项匹配——因为 PG 的行上 xmax 可能是一个仍在活跃的事务（DELETE 后尚未提交），此时行虽然"被删了"但**对查询这个快照的事务仍然可见**（删它的事务还没提交）。

另外，PG 中即使 READ COMMITTED，同一事务里的不同语句的快照并**不完全相同**：commit 后 xmax 变了，可行自己更新的行可能第二句就不见了。这个现象 MySQL 开发者需要预留心理记号。

### 2.3 这个差异在运维层面意味着什么

- MySQL 的 ReadView 存储在事务结构里，没有任何"可见性清理"的维护成本。事务提交了就是提交了，交换 m_ids 数组完事。
- PG 的 xip 数组需要准确判断哪些事务还在跑、哪些提交了——因为脏行就在数据页里，你必须知道它们对每次查询是否可见。而判断 xmax 是否已提交，靠的是 **pg_clog**（commit log），记录每个事务的最终状态（IN_PROGRESS / COMMITTED / ABORTED），存在共享内存 + 磁盘。每做一次可见性判断，如果缓存未命中就要查 clog。vacuum 做的事情之一就是**把足够老的 clog 页清理掉**——如果它不清理，clog 无限增长，启动 slow checkpointer 先把所有数据刷下去。

## 三、HOT update——PG 的"更新不碰索引"数据结构

这是行格式和版本链最精彩的优化。你既然用 MySQL，知道 InnoDB 的 UPDATE 有四条路径：**原地更新**（如果空间够且没改索引列）、**删除+插入**（不归 undo log 的新语句）、**undo log 方式**、"页内重组"。PG 选了一个完全不同、也更精巧的非原地方案。

### 3.1 没有 HOT 的代价

回顾：PG 每 UPDATE 一行 → 标记旧行删除（xmax = 当前事务 ID），在同页或附近页**插新行**。假设你有 3 个索引（primary key + 2 个 secondary），这 3 个索引都需要指向新行的新物理位置 `ctid`。于是每个索引**也要插入新条目**——一场 UPDATE 产生 3 个索引插入、产生 3 个索引旧条目的 dead entries。高频 UPDATE 的场景下，这就是索引膨胀的元凶。

### 3.2 HOT 做了什么

新行如果满足两个条件：

1. **没有被索引的列发生变更**——即所有索引列的值都没变
2. **新行能放进同一个数据页**（还有空间）

则 PG 不更新索引条目。让**旧行和前行的索引条目指向同一个物理页**。页内呢？PG 在每个页里维护一个指向行的**行指针（line pointer / item pointer）数组**。旧行的行指针不是指向旧 tuple 直接，而是指向**一个重定向行指针（redirect）**，redirect 再指向新行的实际位置：

```
页内的 ItemId 数组
 line1 → [重定向] ──→ line3 → 新行 (t_xmin=200, ...)
 line2 → 正常行
 line3 → 实际新行

旧行的 t_ctid 字段指向新行的 (page, offset)，形成页内版本链
```

索引的叶节点里存的**仍然是 line1 的偏移**，读索引后顺着 redirect 走一步就能拿到新行。只有新行放不下同一个页，或者新行更新了索引列 → HOT 失效，重新走全索引更新逻辑。

### 3.3 HOT chain 的清理

索引不更新，意味着索引里永远指向页内某个 line pointer。那这个 chain 上的旧行什么时候能被真正删除？VACUUM。

- VACUUM 扫描到旧行，发现它的 `t_ctid` 指向同页的新行（HOT chain）
- 如果这个旧行的 xmax 已提交、且当前**没有任何快照还需要看到它**
- VACUUM 就删除这个旧行，回收行指针
- 连锁删除：删完第一个旧行轮，检查第二个重定向指针还能否用到其他有效行，不能也删
- 最终只剩一个**最新的行版本 + 一个可能的重定向指针**

这就是 VACUUM 在 HOT 下的核心作用：把重定向链上过期的行指针清扫成空洞，等待下次 INSERT 复用。**如果 VACUUM 不跑，HOT chain 上的旧行永远占着空间，页内行指针数组越长，表文件越大。**

可证伪的操作：你随便找个 PG 实例执行 `SELECT * FROM pg_stat_user_tables WHERE n_tup_hot_upd > 0`——`n_tup_hot_upd` 记录的就是通过 HOT 避免了索引更新次数。如果热表这个数很低（相对 `n_tup_upd`），说明你要么改索引列太频繁，要么页填充因子不合理导致新行常放不进。

## 四、VACUUM——到底怎么扫描、怎么回收

现在开始拆 VACUUM 的内部执行过程。省略信令和统计更新（analyze），只看**清理脏行**这一步。

### 4.1 步骤一：切 visibility map

每个表有一个 **visibility map（VM）**，实际上是单独的文件（`xxx_vm`），包含每个堆页的一个 bit：这个页上**所有行对所有人可见**？True / False。

如果 VM bit 为 True → VACUUM **跳过这个页不扫描**——因为里面一定没有 dead tuple。如果表一直在 append-only 且从不更新，VACUUM 代价极低，O(被改过的页数)，非 O(表大小)。

### 4.2 步骤二：页内扫描 loop

对 VM bit = false 的页逐页进入：

1. 从页头开始遍历 `ItemId` 数组的每一个行指针
2. 判断每一行的可见性：用当前数据库的最小活跃快照（OldestXmin，数据库当前所有活跃事务中最小的 xid）判断——如果一行对 OldestXmin 已经不可见，说明没有任何活跃事务还需要这颗死 metamask，安全回收
3. 可回收的行：标记其 `ItemId` 为 `LP_UNUSED`（行指针消耗掉），**页内空间标记为可用**
4. 如果经此回收后，**页上所有剩余行都对所有人可见** → 设置 VM bit = true

关键细节——VACUUM 不缩文件，磁盘上的数据文件大小不变。它只做**逻辑回收**：标记 dead line pointers 变成 `LP_UNUSED`，下次 INSERT 要在这个页里找空间时，可以复用这些空洞。这和 MySQL 的"表空间碎片"概念类似—— UPDATE 导致的空间永远存在，除非 VACUUM FULL 或扩展 pg_repack 重写整个表。

### 4.3 步骤三：第二遍扫描索引

这是 VACUUM 最耗时的部分。数据页回收完行指针后，**每个索引都要扫一遍**：

- 找到索引条目中指向已被回收行的那些条目
- 标记索引条目为 dead（索引也产生 dead index tuples）
- 等后续 VACUUM 的第二轮时，如果这个 dead index tuple 的行在堆表里已被清除 → 索引条目安全删除

这就是为什么一个频繁 UPDATE/DELETE 的库，索引也会跟着膨胀。清理索引的代价往往比清理堆表更大，因为堆表走顺序扫描，索引走随机读。

### 4.4 VACUUM FULL vs pg_repack

此前的区别在于：

| 动作 | VACUUM | VACUUM FULL | pg_repack |
|------|--------|-------------|-----------|
| 做什么 | 标记行指针 LP_UNUSED | 完全重写整个表和索引到新文件 | 重写表+索引，用触发器同步变更 |
| 锁 | ShareUpdateExclusiveLock（不阻塞读写） | **AccessExclusiveLock**（整个表不可用） | AccessExclusiveLock **仅在最终切换瞬间** |
| 磁盘空间 | 不缩 | 缩 | 缩 |

VACUUM FULL 底层是先建一个全新的数据文件，把活行逐行拷贝过去，最后原子替换 `pg_class.relfilenode`。拷贝期间持有 AccessExclusiveLock——对表的一切操作等待。

pg_repack 扩展绕开了这个限制：拷贝大量数据时不拿排他锁，只在最后切换 relfilenode 的瞬间拿一次极短的排他锁。

## 五、XID wraparound 与 freeze——这真的是"运维坑"不是"面试题"

### 5.1 32 位的事务 ID 为什么是真实限制

PG 的事务 ID 是 32 位无符号整数 `uint32`，只有约 42.9 亿个。对于交易系统（每秒几百个事务），这个数耗得飞快。

但更关键的是可见性判断的**循环比较**。PG 把事务 ID 看作**一个环**，`xid` 从 0 一直增长到 2^32-1，然后回卷到 0。判断两个事务 ID 的先后关系（"T1 在 T2 之前吗"）不能用简单的 `T1 < T2`，必须用**模比较**，且限定比较范围在当前 XID 的一半空间内（约 21 亿个事务）。

```
如果 current_xid = 100
那么 50-149 之间的事务被认为是"未来"（因为 99+50 > 环的一半）
49 以下被认为是"过去"（因为 100-50 < 环的一半）

回卷到 0 后，0 对前面的 4B-1 来说是"未来"，但 half-space 比较认不出
```

当 xid 跑了一圈回到老 xid，旧行的 xmin 突然不再是正常比较的"过去"，而变成了**不可比较的同环值**。这时所有正常可见性判断都失效——Postgres 的唯一选择是停机保护数据。

### 5.2 freeze 干了什么——物理操作

VACUUM freeze 的模式：

1. 对足够的旧行，把 `t_xmin` 替换为 `FrozenTransactionId = 2`（0 和 1 保留为无效/引导）
2. 基于此，替换后在可见性判断时，一行 t_xmin=2 永远是"过去的已提交"，不再需要查 xid 环比较
3. freeze 同时更新 `pg_class.relfrozenxid`：记下"此表所有行至少在 xid < X 之前已被冻结"

anti-wraparound autovacuum 在上述模比较发现 xid 跨过半环时自动触发，优先级高于普通 autovacuum。它会**冻结全表所有能冻的行**，不管表变不变量。

监控的核心 SQL：

```sql
SELECT datname, age(datfrozenxid) AS xid_age, 
       current_setting('autovacuum_freeze_max_age') AS freeze_limit
FROM pg_database;
```

`age()` 函数做模比较，返回当前的 xid 和冻结点的距离。当 `xid_age` 超过 `autovacuum_freeze_max_age`（默认 2 亿），对应数据库强制触发 anti-wraparound vacuum。超过 15 亿，数据库强制 shutdown。

**长事务是最大的加速器**：一个事务保持不提交，OldestXmin 一直在这个事务的 xid。freeze 判断"这个行还能被任何事务看见吗"时，答案永远是"这个长事务可能看见"——所以 freeze 不能冻这些行，且死行无法被清理。事务运行越久，未被 freeze 的行越多，xid 消耗越向停机方向推进。

## 六、动手实验——看到磁盘结构而不是 `pg_stat` 的数

### 实验 1：直接看 xmin/xmax/ctid 三元组 + HOT chain

```bash
docker run --name pg-day1 -e POSTGRES_PASSWORD=pass -p 5432:5432 -d postgres:17
docker exec -it pg-day1 psql -U postgres
```

```sql
-- 建表 + 非索引列（让 HOT 失效条件明确）
CREATE TABLE ht (id serial PRIMARY KEY, name text, n int);
CREATE INDEX ON ht(n);          -- n 有索引，name 没有
INSERT INTO ht (name, n) SELECT 'x', g FROM generate_series(1,5) g;

-- 查初始行：xmin/xmax/ctid
SELECT xmin, xmax, ctid, name, n FROM ht ORDER BY id;
-- 5 行，xmax 全 0，ctid 为 (0,1)..(0,5) ——同页

-- 只改 name（name 无索引）→ 应走 HOT
UPDATE ht SET name = 'yy' WHERE id = 2;
SELECT xmin, xmax, ctid, name, n FROM ht ORDER BY id;
-- 现在你能看到：
--   旧行 (0,2) xmax=当前xid ctid=(0,6) → 旧行逻辑删除但 ctid 指向新行(0,6)
--   新行 (0,6) xmin=当前xid xmax=0 → 新行就在同页且索引没被更新

-- 改 n（n 有索引）→ 不走 HOT
UPDATE ht SET n = 999 WHERE id = 3;
SELECT xmin, xmax, ctid, name, n FROM ht ORDER BY ctid;
-- 旧行 (0,3) xmax=当前xid，新行在同页但索引也更新了

-- 验证索引页未被 HOT updated
SELECT n_tup_upd, n_tup_hot_upd FROM pg_stat_user_tables WHERE relname = 'ht';
```

注意 `n_tup_hot_upd` 和 `n_tup_upd` 的差值——HOT 失效的 UPDATE 会增加 `n_tup_upd` 而 `n_tup_hot_upd` 不动。

### 实验 2：通过 pageinspect 看 page 内部的行指针

```sql
CREATE EXTENSION pageinspect;

SELECT * FROM heap_page_items(get_raw_page('ht', 0));
-- lp_off：该行在页内的起始偏移
-- lp_flags：0=unused 1=normal 2=redirect 3=dead
-- t_xmin/t_xmax/t_ctid：就是刚才直接查到的系统列
-- t_infomask2：包含 HOT chain 相关的 HEAP_HOT_UPDATED / HEAP_ONLY_TUPLE 标志
```

注意那一行 `lp_flags=2` 的行——这就是 HOT 的 redirect line pointer。VACUUM 做完后这些 redirect 会被清扫回收。

### 实验 3：VACUUM 回收后的行指针变化

```sql
DELETE FROM ht WHERE id = 1;
SELECT lp, lp_flags FROM heap_page_items(get_raw_page('ht', 0));
-- DELETE 后的行 lp_flags 变为 3 (dead)，但空间未回收

VACUUM ht;
SELECT lp, lp_flags FROM heap_page_items(get_raw_page('ht', 0));
-- 回收后的行指针变为 lp_flags=0 (unused)，可被下次 INSERT 复用
```

同时 check VM 文件的是否标记该页：

```sql
SELECT * FROM pg_visibility_map('ht', 0);
-- all_visible=true 说明此页被标记，后续 VACUUM 将跳过
```

### 实验 4：长事务卡 freeze 的完整性验证

```sql
-- 窗口 A：保持事务不提交
BEGIN;
SELECT txid_current();
-- 假设返回 780

-- 窗口 B：大量修改
UPDATE ht SET name = 'a' WHERE id <= 100;

-- 窗口 B：执行 VACUUM，但 freeze 不会生效（OldestXmin 是 780，旧行对它可见）
VACUUM FREEZE ht;

-- 检查 freeze 效果
SELECT age(relfrozenxid) FROM pg_class WHERE relname = 'ht';
-- 因为窗口 A 的长事务还在，freeze 跳过了那些更新的行
-- 窗口 A COMMIT 后再跑一次 VACUUM FREEZE 才能冻干净
```

## 七、自测——不是背概念，而是说清楚为什么

1. 如果一个 MySQL DBA 问你"为什么 PG 不在 undo log 里存旧版本，而非要塞在数据页里"——你怎么用少于 5 句话解释设计 choice，并且能讲清利弊？
2. HOT update 的条件有几个？如果 UPDATE 改了索引列，HOT 会怎么退化？
3. VACUUM 回收 dead tuple 以后，表的文件大小没变。下一次 INSERT 进来怎么找到这些空洞？空洞在什么条件下会被复用？
4. VACUUM FULL 和 pg_repack 的根本区别不是"会不会锁表"，是内部用什么方式实现空表重写。这两个的拷贝逻辑区别是什么？
5. freeze 把 t_xmin 改成了什么特殊值？为什么改了这个值后，不再需要做 xid 环比较？
6. 一个长事务的存在为什么同时阻止 VACUUM 和 freeze？阻止的逻辑在源码哪一层？

能在纸上或白板上画出问题 1-3 的磁盘/页布局，Day 1 才算真正吃透。

---

**Day 2 预告**：索引类型与 JSONB。不列罗类型，而是讲 GIN 的倒排索引结构（和 MySQL FULLTEXT 的对比）、B-Tree 的 page split 在 PG 和 InnoDB 中的实现差异。以及 JSONB 到底为什么需要 GIN 索引，而不是直接 B-Tree。