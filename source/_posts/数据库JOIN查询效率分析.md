---
title: 数据库JOIN查询效率分析
date: 2025-12-15 20:50:17
tags: Database
---

## 1. JOIN 为什么会慢：本质是“中间结果 + 匹配代价”

一条 JOIN 查询的耗时通常由下面几类成本叠加决定：

- **扫描成本**：读多少行、是否走索引、是否顺序读还是随机读（回表）。
- **匹配成本**：用什么连接算法（Nested Loop / Hash / Merge）。
- **中间结果膨胀**：一对多、多对多会让行数“乘法增长”，后续的 JOIN/排序/聚合都会被放大。
- **内存与落盘（spill）**：Hash Join 或排序（Merge Join 前置排序）内存不足会落盘，IO 暴涨。
- **优化器误判（基数估计不准）**：导致选错 JOIN 顺序、选错算法、选错驱动表（常见性能灾难来源）。
- **数据倾斜**：少数热门 key 导致 Hash Join 桶/分区极不均衡，局部变慢。

------

## 2. 三种主流物理 JOIN 算法：适用场景与效率特征

### 2.1 Nested Loop Join（NLJ，嵌套循环）

**思想**：外表每一行去内表找匹配行。

- **无索引**：近似 `O(|A| * |B|)`（非常慢）
- **内表 join key 有索引**：近似 `O(|A| * log|B|)`（外表过滤后很小会很快）

**适用**：

- 外表经过 `WHERE` 过滤后行数很少
- 内表在 join key 上有高选择性索引（减少回表）

> MySQL 官方文档中直接描述了 NLJ 的执行方式：逐表嵌套循环读取。([dev.mysql.com](https://dev.mysql.com/doc/refman/8.4/en/nested-loop-joins.html?utm_source=chatgpt.com))

------

### 2.2 Hash Join（哈希连接）

**思想**：先用较小的一侧建立哈希表（build），再扫描另一侧探测（probe）。

- 典型等值连接下近似 `O(|A| + |B|)`
- **强依赖内存**：内存不足会“on-disk hash join / spill”，性能下降明显

**适用**：

- **等值连接**（`A.k = B.k`）的大表 JOIN
- join key 上缺索引或索引无法有效使用时

> MySQL 8.0.18 起实现 Hash Join，用于 **inner equi-join**；并说明多数情况下比“无索引时的 block-nested-loop”更高效。([dev.mysql.com](https://dev.mysql.com/doc/relnotes/mysql/8.0/en/news-8-0-18.html))

------

### 2.3 Merge Join（Sort-Merge Join，排序合并）

**思想**：两边按 join key 有序后“拉链式”合并匹配。

- 若两边已天然有序（索引有序扫描 / 上游结果已排序）：近似 `O(|A| + |B|)`
- 否则需要排序：额外 `O(n log n)`，大表排序可能落盘

**适用**：

- 两边已经按 join key 排序（或容易得到有序输出）
- 某些范围连接/需要有序输出的场景更友好（取决于优化器实现）

------

## 3. MySQL 的 JOIN 方法：它到底用哪一种？

MySQL 典型会用以下几类物理算法/优化：

- **Nested Loop Join（NLJ）**：经典执行方式。([dev.mysql.com](https://dev.mysql.com/doc/refman/8.4/en/nested-loop-joins.html?utm_source=chatgpt.com))
- **BKA（Batched Key Access）**：一种“批量索引探测 + join buffer”的优化策略，可用于 inner/outer/semijoin 等（减少随机 IO、提升局部性）。([dev.mysql.com](https://dev.mysql.com/doc/refman/8.4/en/bnl-bka-optimization.html))
- **Hash Join**：MySQL 8.0.18 起支持，用于 **inner equi-joins**；默认在满足条件时可被选择，并受 `join_buffer_size` 等影响（内存不足会转为磁盘方式）。([dev.mysql.com](https://dev.mysql.com/doc/relnotes/mysql/8.0/en/news-8-0-18.html))

> 结论：MySQL **不是只有一种** JOIN；老版本更偏 NLJ +（BNL/BKA 这类 join buffer 优化），8.0.18 之后补上了 Hash Join。([dev.mysql.com](https://dev.mysql.com/doc/relnotes/mysql/8.0/en/news-8-0-18.html))

------

## 4. 主流数据库各自常用的 JOIN 方法（对照表）

| 数据库              | 常见物理 JOIN 方法                          | 备注                                                         |
| ------------------- | ------------------------------------------- | ------------------------------------------------------------ |
| **MySQL**           | NLJ、BKA、Hash Join（8.0.18+）              | Hash Join 用于 inner equi-join；BKA 依赖 join buffer + 索引批量访问 ([dev.mysql.com](https://dev.mysql.com/doc/relnotes/mysql/8.0/en/news-8-0-18.html)) |
| **PostgreSQL**      | Nested Loop / Hash Join / Merge Join        | 官方文档明确三种 join strategy ([PostgreSQL](https://www.postgresql.org/docs/current/planner-optimizer.html?utm_source=chatgpt.com)) |
| **SQL Server**      | Nested Loops / Hash / Merge + Adaptive Join | Adaptive Join 可在 Hash 与 NL 之间运行时选择 ([Microsoft Learn](https://learn.microsoft.com/en-us/sql/relational-databases/performance/joins?view=sql-server-ver17&utm_source=chatgpt.com)) |
| **Oracle Database** | Nested Loops / Hash Join / Sort Merge       | 官方文档说明优化器会在三种 join method 上算 cost ([Oracle Docs](https://docs.oracle.com/en/database/oracle/oracle-database/21/tgsql/query-optimizer-concepts.html?source=%3Aso%3Ali%3Aor%3Aawr%3Aocl%3A%3A%3ANews&utm_source=chatgpt.com)) |
| **SQLite**          | Nested Loops（为主/基本就是它）             | SQLite 文档明确：joins using nested loops ([SQLite](https://www.sqlite.org/queryplanner-ng.html?utm_source=chatgpt.com)) |
| **MariaDB**         | 多种 block-based join（BNL/BNLH/BKA 等）    | 官方内部文档列出 block-based join algorithms ([MariaDB](https://mariadb.com/docs/general-resources/development-articles/mariadb-internals/mariadb-internals-documentation-query-optimizer/block-based-join-algorithms?utm_source=chatgpt.com)) |

------

## 5. 提升 JOIN 性能的“工程要点清单”

1. **让过滤尽量早发生**（先把驱动表变小，再 JOIN）。
2. **给 join key 建对索引**（尤其是被驱动表；并确保类型一致避免隐式转换）。
3. **减少回表**：能覆盖索引就覆盖索引（少取列、或让索引包含常用列）。
4. **避免函数包裹 join key**（如 `CAST(col)` / `LOWER(col)` 让索引失效）。
5. **控制中间结果**：能先聚合再 JOIN（例如先按外键 group，再连主表）。
6. **关注内存与落盘**：Hash Join/Sort 相关参数不足会 spill。
7. **统计信息要新**：优化器估错基数，JOIN 顺序/算法就容易选错。

------

## 6. 排查建议：你应该看什么

- **MySQL**：`EXPLAIN FORMAT=TREE` / `EXPLAIN ANALYZE`（MySQL 8.0.18+ 引入并强化）([dev.mysql.com](https://dev.mysql.com/doc/relnotes/mysql/8.0/en/news-8-0-18.html))
- **SQLite**：`EXPLAIN QUERY PLAN`（每个 nested loop 会输出一条 SCAN/SEARCH 记录）([SQLite](https://www.sqlite.org/eqp.html?utm_source=chatgpt.com))

