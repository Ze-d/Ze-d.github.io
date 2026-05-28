---
title: LeetCode常见集合以及接口
date: 2026-02-02 21:41:20
tags: [leetcode,java]
---

在 Java 的集合框架（Collection Framework）中，这些数据结构是构建任何应用程序的基石。我们可以把它们分为**线性结构**（List, Stack, Queue）和**非线性结构**（Set, Map/Hash Table）。

------

## 1. List（列表）：

List 是最常用的接口，它保证了元素的**插入顺序**。

### 1.1**常用接口方法：** 

### `add(e)`, `get(index)`, `remove(index)`, `size()`, `contains(e)`。

- 增：`add(e)`、`add(index, e)`
- 查：`get(index)`、`indexOf(e)`、`contains(e)`
- 删：`remove(index)`、`remove(Object o)`
- 改：`set(index, e)`
- 其他：`size()`、`isEmpty()`、`clear()`、`subList(l, r)`

> [!CAUTION]
>
> **注意 remove 的重载坑：**
>  `list.remove(1)` 是按下标删；
>  `list.remove(Integer.valueOf(1))` 才是删除元素值为 1 的对象（LeetCode 很常见坑）。

### 1.2**主流实现：**

- **ArrayList：** 基于**动态数组**。查询快（根据下标直接定位），增删慢（涉及数组扩容和位移）。它是开发中的首选。
- **LinkedList：** 基于**双向链表**。增删快（只需更改指针），查询慢（需要从头遍历）。它同时也实现了 `Deque` 接口，可以当队列用。



------

## 2. Stack（栈）：

栈就像一叠盘子，最后放上去的总是最先被拿走。

#### 2.1**常用接口方法：**

 `push(e)` (入栈), `pop()` (出栈并移除), `peek()` (查看栈顶)。

```java
Deque<Integer> st = new ArrayDeque<>();
st.push(1);      // 入栈（等价于 addFirst）
st.pop();        // 出栈并移除（等价于 removeFirst）
st.peek();       // 看栈顶（等价于 peekFirst）
```

#### 2.2**实现建议：**

- **Stack 类：** Java 早期提供的实现，继承自 `Vector`。它是**线程安全**的，但因为加锁开销大，现在已**不推荐使用**。

- **Deque 接口实现：** 官方推荐使用 `ArrayDeque` 来实现栈的功能，性能更优：

  `Deque<Integer> stack = new ArrayDeque<>();`

------

## 3. Queue（队列）：

队列就像排队买票，先来的先走。

#### **3.1常用接口方法：** 

`offer(e)` (入队), `poll()` (出队并移除), `peek()` (查看队首)。

```java
Deque<Integer> q = new ArrayDeque<>();
q.offer(1);   // 入队（尾部）
q.poll();     // 出队（头部）
q.peek();     // 看队首
```

#### **3.2主流实现：**

- **ArrayDeque：** 基于循环数组的高效双端队列。
- **LinkedList：** 同样实现了 Queue 接口。
- **PriorityQueue：** **优先级队列**。元素不是按顺序排，而是按“优先级”（自然顺序或自定义比较器）出队。底层是**二叉堆**。

------

## 4. Set（集合）：

Set 的核心特性是**不允许重复元素**。

#### **4.1常用接口方法：** 

`add(e)`, `remove(e)`, `contains(e)`。

- add(e)：返回 boolean（是否真的加入）
- contains(e)
- remove(e)
- size() / isEmpty() / clear()

#### 4.2**主流实现：**

- **HashSet：** 底层其实就是一个 `HashMap`。它是**无序**的，查找和插入的时间复杂度都是 $O(1)$。
- **LinkedHashSet：** 维护了一个双向链表来记录插入顺序，所以它是**有序**的。
- **TreeSet：** 底层是**红黑树**。元素会自动排序（从小到大），但性能比 HashSet 略慢 $O(\log n)$。

------

## 5. Hash Table（哈希表/散列表）

在 Java 中，哈希表主要通过 `Map` 接口体现，存储的是 **Key-Value（键值对）**。

#### **常用接口方法：** 

#### `put(k, v)`, `get(k)`, `containsKey(k)`, `keySet()`。

- 增/改：`put(k,v)`、`putIfAbsent(k,v)`
- 查：`get(k)`、`getOrDefault(k, defaultValue)`
- 判：`containsKey(k)`
- 删：`remove(k)`
- 遍历：`entrySet()`、`keySet()`、`values()`
- 进阶：`computeIfAbsent(k, ...)`（分组题超好用）

**计数统计**

```java
map.put(x, map.getOrDefault(x, 0) + 1);
```

**分组**

```java
map.computeIfAbsent(key, k -> new ArrayList<>()).add(val);
```

索引表

```java
if (map.containsKey(target - x)) ...
map.put(x, i);
```

#### **主流实现：**

- **HashMap：** 最常用的实现。基于“数组+链表+红黑树”。当发生哈希冲突时，数据存在链表中；链表过长（超过 8）时转为红黑树以优化查询。
- **Hashtable：** 古老的实现，线程安全，但不支持 `null` 键值。**基本被废弃**。
- **ConcurrentHashMap：** 线程安全的高性能版本，是多线程环境下的首选。

------

### 总结对比表

| **数据结构** | **接口** | **重复性**  | **顺序性** | **核心实现**                   |
| ------------ | -------- | ----------- | ---------- | ------------------------------ |
| **列表**     | `List`   | 可重复      | 插入顺序   | `ArrayList`, `LinkedList`      |
| **栈**       | `Deque`  | 可重复      | LIFO       | `ArrayDeque`, `Stack`          |
| **队列**     | `Queue`  | 可重复      | FIFO       | `ArrayDeque`, `PriorityQueue`  |
| **集合**     | `Set`    | 不可重复    | 无序/排序  | `HashSet`, `TreeSet`           |
| **哈希表**   | `Map`    | Key不可重复 | 无序       | `HashMap`, `ConcurrentHashMap` |
