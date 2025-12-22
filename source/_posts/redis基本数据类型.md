---
title: redis基本数据类型
date: 2025-12-13 15:16:50
tags: Redis
---

## 1. String（字符串）

**本质**：

- 最基础、最常用的类型
- 可以存：普通字符串、JSON、数字（支持自增自减）、二进制

**典型使用场景：**

1. **缓存单个对象 / 配置**
   - 比如：`user:1001` → 用户信息 JSON
   - `system:config:xxx` → 某个配置项
2. **计数器（PV / UV / 业务计数）**
   - 页面访问量、点赞数、库存数等
   - 用 `INCR / DECR / INCRBY` 做原子自增自减
3. **分布式锁（简单版本）**
   - `SET lock:order_123 1 NX EX 30`
   - 利用 `NX`（不存在才写）+ 过期时间
4. **限流 / 防刷**
   - 某个IP某接口一分钟内访问次数：
     - `INCR ip:xxx:api:yyy` + `EXPIRE` 设置过期

**常用命令：**

```text
SET key value
GET key
INCR key
DECR key
SETEX key seconds value     # 设置并带过期时间
```

------

## 2. Hash（哈希）

**本质**：

- 类似一个小型的 `key-value` 结构
- 外层有一个 Redis key，内层很多 field-value

**典型使用场景：**

1. **存对象（结构化字段）**

   - 比如用户对象：

     `HSET user:1001 name "Tom" age 18 city "Beijing"`

   - 读写某个字段很方便：`HGET user:1001 age`

2. **存轻量级配置 / 统计信息表**

   - 一张“表”中某行数据的各字段

3. **避免把整个 JSON 取出再改再写回**

   - 需要频繁修改局部字段时，比 String 更细粒度

**常用命令：**

```text
HSET user:1001 name "Tom" age 18
HGET user:1001 name
HGETALL user:1001
HDEL user:1001 age
HLEN user:1001
```

------

## 3. List（列表）

**本质**：

- 有序、可重复
- 双端链表结构：头尾都能高效插入/弹出

**典型使用场景：**

1. **消息队列（简单版）**
   - 生产者：`LPUSH queue:order msg`
   - 消费者：`BRPOP queue:order`（阻塞式弹出）
   - 用于异步任务、日志等
2. **时间线/评论流（按时间顺序）**
   - 比如：`list:comments:article:1001`
   - 新评论用 `LPUSH` ，再 `LRANGE` 做分页
3. **任务列表 / 待办队列**
   - 按顺序处理的任务

**常用命令：**

```text
LPUSH key value1 value2     # 从左插入
RPUSH key value1 value2     # 从右插入
LPOP key                    # 从左弹出
RPOP key                    # 从右弹出
LRANGE key start stop       # 获取区间元素（做分页）
BLPOP / BRPOP               # 阻塞式获取（用于队列）
```

------

## 4. Set（集合）

**本质**：

- 无序、去重
- 基于哈希表，支持交集、并集、差集

**典型使用场景：**

1. **去重集合**
   - 比如：某天访问过的用户：`SADD uv:2025-12-12 uid1 uid2`
   - 查询是否访问：`SISMEMBER uv:2025-12-12 uidX`
2. **社交关系**
   - 用户关注列表：`follow:uid`
   - 好友关系 = 互相关注 = 两个 set 的交集
3. **标签 / 兴趣集合**
   - 某个标签下的用户集合
   - 做“共同兴趣”、差异用户等集合运算

**常用命令：**

```text
SADD key member1 member2
SREM key member
SISMEMBER key member       # 是否在集合内
SMEMBERS key               # 获取所有成员（注意大集合慎用）
SCARD key                  # 元素个数
SINTER set1 set2           # 交集
SUNION set1 set2           # 并集
SDIFF set1 set2            # 差集
```

------

## 5. Sorted Set（有序集合，ZSet）

**本质**：

- 每个元素有一个 `score`（分数）
- 按 score 排序，元素不能重复，但 score 可以相同

**典型使用场景：**

1. **排行榜（经典场景）**
   - 游戏积分排行榜、热度排行榜
   - 用用户ID作 member，积分作 score：
     - `ZINCRBY rank:game 10 user:1001`
   - 获取前 N：`ZREVRANGE rank:game 0 9 WITHSCORES`
2. **按时间排序的集合（score=时间戳）**
   - 新闻/帖子按时间排序
   - “从某时间之后的最新 N 条”
3. **延时任务 / 定时任务（简单版）**
   - score 存“执行时间戳”
   - 定时扫描 score ≤ 当前时间的任务执行
4. **权重排序/推荐系统简单实现**
   - score = 综合打分，随时调整

**常用命令：**

```text
ZADD key score1 member1 score2 member2
ZINCRBY key increment member     # 分数自增（积分增加）
ZRANGE key start stop WITHSCORES # 按 score 从小到大
ZREVRANGE key start stop WITHSCORES # 从大到小（排行榜常用）
ZREM key member
ZCARD key
ZCOUNT key min max               # 某个分数区间元素个数
```

------

## 其他常见高级类型

- **Bitmap**：做大规模布尔统计（签到、活跃、是否访问）
- **HyperLogLog**：近似去重计数（超大 UV 场景）
- **Geo**：地理位置（距离、附近的人）
- **Stream**：正式的消息队列/日志流模型
