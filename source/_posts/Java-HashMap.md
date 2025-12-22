---
title: Java HashMap
date: 2025-12-08 10:58:56
tags: [Java,Data-Construction]
---

## 1. 概述

HashMap 是 Java 集合框架中最重要的数据结构之一，它实现了 Map 接口，提供了基于哈希表的键值对存储机制。HashMap 允许使用 null 键和 null 值，且不保证元素的顺序。

## 2. 核心特性

- **键值对存储**：存储 key-value 对
- **时间复杂度**：平均 O(1) 的 get 和 put 操作
- **非线程安全**：多线程环境下需要外部同步
- **允许 null**：允许 null 键和 null 值
- **无序**：不保证元素的插入顺序或任何其他顺序

## 3. 底层数据结构

### 3.1 JDK 8 之前的实现（数组 + 链表）
```
HashMap
├── Entry<K,V>[] table (数组)
└── 每个数组元素 → Entry<K,V> (链表节点)
```

### 3.2 JDK 8 及之后的实现（数组 + 链表/红黑树）
```
HashMap
├── Node<K,V>[] table (数组)
├── 普通链表节点 → Node<K,V>
└── 当链表长度 ≥ 8 → TreeNode<K,V> (红黑树节点)
```

## 4. 核心类与接口

### 4.1 Node 类（链表节点）
```java
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;      // 键的哈希值
    final K key;         // 键
    V value;             // 值
    Node<K,V> next;      // 下一个节点
    
    Node(int hash, K key, V value, Node<K,V> next) {
        this.hash = hash;
        this.key = key;
        this.value = value;
        this.next = next;
    }
}
```

### 4.2 TreeNode 类（红黑树节点）
```java
static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {
    TreeNode<K,V> parent;   // 父节点
    TreeNode<K,V> left;     // 左子节点
    TreeNode<K,V> right;    // 右子节点
    TreeNode<K,V> prev;     // 前驱节点
    boolean red;           // 红色节点标识
}
```

## 5. 关键参数

```java
// 默认初始容量 - 必须是 2 的幂
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // 16

// 最大容量
static final int MAXIMUM_CAPACITY = 1 << 30;

// 默认负载因子
static final float DEFAULT_LOAD_FACTOR = 0.75f;

// 树化阈值：链表长度 ≥ 8 时转换为红黑树
static final int TREEIFY_THRESHOLD = 8;

// 树退化阈值：树节点 ≤ 6 时退化为链表
static final int UNTREEIFY_THRESHOLD = 6;

// 最小树化容量：容量 ≥ 64 时才进行树化
static final int MIN_TREEIFY_CAPACITY = 64;

// 哈希表数组
transient Node<K,V>[] table;

// 元素数量
transient int size;

// 修改次数（用于快速失败机制）
transient int modCount;

// 扩容阈值 = capacity * loadFactor
int threshold;

// 负载因子
final float loadFactor;
```

## 6. 哈希计算与索引定位

### 6.1 哈希扰动函数
```java
static final int hash(Object key) {
    int h;
    // 计算哈希码并扰动，减少哈希冲突
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

### 6.2 索引计算
```java
// 计算元素在数组中的位置
// 这个计算只有在n是2的整数幂的时候成立
int index = (n - 1) & hash;
// 其中 n 是数组长度，hash 是扰动后的哈希值
```

## 7. Put 操作实现

### 7.1 put 方法流程
```java
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}

final V putVal(int hash, K key, V value, boolean onlyIfAbsent, boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    
    // 步骤1：如果表为空或长度为0，进行初始化
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    
    // 步骤2：计算索引位置，如果该位置为空
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {
        // 步骤3：处理哈希冲突
        Node<K,V> e; K k;
        if (p.hash == hash && 
            ((k = p.key) == key || (key != null && key.equals(k))))
            // 情况3.1：键已存在（首节点）
            e = p;
        else if (p instanceof TreeNode)
            // 情况3.2：红黑树节点
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            // 情况3.3：遍历链表
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {
                    // 在链表尾部插入新节点
                    p.next = newNode(hash, key, value, null);
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);  // 尝试树化
                    break;
                }
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;  // 键已存在
                p = e;
            }
        }
        
        // 步骤4：替换已存在键的值
        if (e != null) {
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    
    // 步骤5：更新修改次数和元素数量
    ++modCount;
    if (++size > threshold)
        resize();  // 扩容
    afterNodeInsertion(evict);
    return null;
}
```

### 7.2 树化过程（treeifyBin）
当链表长度达到 8 且数组长度 ≥ 64 时，链表转换为红黑树：
```java
final void treeifyBin(Node<K,V>[] tab, int hash) {
    int n, index; Node<K,V> e;
    if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
        resize();  // 如果数组太小，先扩容
    else if ((e = tab[index = (n - 1) & hash]) != null) {
        // 将链表转换为双向链表，再转换为红黑树
        TreeNode<K,V> hd = null, tl = null;
        do {
            TreeNode<K,V> p = replacementTreeNode(e, null);
            if (tl == null)
                hd = p;
            else {
                p.prev = tl;
                tl.next = p;
            }
            tl = p;
        } while ((e = e.next) != null);
        if ((tab[index] = hd) != null)
            hd.treeify(tab);  // 转换为红黑树
    }
}
```

## 8. Get 操作实现

```java
public V get(Object key) {
    Node<K,V> e;
    return (e = getNode(hash(key), key)) == null ? null : e.value;
}

final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
    
    // 步骤1：检查表是否为空
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (first = tab[(n - 1) & hash]) != null) {
        
        // 步骤2：检查首节点
        if (first.hash == hash &&
            ((k = first.key) == key || (key != null && key.equals(k))))
            return first;
        
        // 步骤3：遍历链表或红黑树
        if ((e = first.next) != null) {
            if (first instanceof TreeNode)
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);
            do {
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    return e;
            } while ((e = e.next) != null);
        }
    }
    return null;
}
```

## 9. 扩容机制（Resize）

```java
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
    
    // 计算新容量和阈值
    if (oldCap > 0) {
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1; // 双倍阈值
    }
    else if (oldThr > 0)
        newCap = oldThr;  // 使用构造函数的阈值作为初始容量
    else {
        newCap = DEFAULT_INITIAL_CAPACITY;  // 默认容量
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
    
    threshold = newThr;
    
    // 创建新数组并重新哈希
    @SuppressWarnings({"rawtypes","unchecked"})
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    
    if (oldTab != null) {
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;
                if (e.next == null)
                    // 单个节点直接重新定位
                    newTab[e.hash & (newCap - 1)] = e;
                else if (e instanceof TreeNode)
                    // 红黑树节点分裂
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                else {
                    // 链表节点重新分配
                    Node<K,V> loHead = null, loTail = null;
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    do {
                        next = e.next;
                        // 关键优化：判断节点是否需要移动到新位置
                        if ((e.hash & oldCap) == 0) {
                            // 保持原索引位置
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        } else {
                            // 移动到新位置（原索引 + oldCap）
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    
                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
                    if (hiTail != null) {
                        hiTail.next = null;
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    return newTab;
}
```

## 10. 移除操作实现

```java
public V remove(Object key) {
    Node<K,V> e;
    return (e = removeNode(hash(key), key, null, false, true)) == null ?
        null : e.value;
}

final Node<K,V> removeNode(int hash, Object key, Object value,
                           boolean matchValue, boolean movable) {
    Node<K,V>[] tab; Node<K,V> p; int n, index;
    
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (p = tab[index = (n - 1) & hash]) != null) {
        
        Node<K,V> node = null, e; K k; V v;
        
        // 查找要删除的节点
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            node = p;
        else if ((e = p.next) != null) {
            if (p instanceof TreeNode)
                node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
            else {
                do {
                    if (e.hash == hash &&
                        ((k = e.key) == key ||
                         (key != null && key.equals(k)))) {
                        node = e;
                        break;
                    }
                    p = e;
                } while ((e = e.next) != null);
            }
        }
        
        // 删除节点
        if (node != null && (!matchValue || (v = node.value) == value ||
                             (value != null && value.equals(v)))) {
            if (node instanceof TreeNode)
                ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
            else if (node == p)
                tab[index] = node.next;
            else
                p.next = node.next;
            ++modCount;
            --size;
            afterNodeRemoval(node);
            return node;
        }
    }
    return null;
}
```

## 11. 性能优化特性

### 11.1 容量设计为 2 的幂次
- 原因：`(n - 1) & hash` 等价于 `hash % n`，但位运算效率更高
- 好处：提高索引计算速度

### 11.2 负载因子优化
- 默认 0.75：平衡时间成本和空间成本
- 负载因子过高 → 哈希冲突增加，性能下降
- 负载因子过低 → 空间浪费，扩容频繁

### 11.3 链表转红黑树
- 阈值：链表长度 ≥ 8
- 最小容量要求：数组长度 ≥ 64
- 退化阈值：树节点 ≤ 6 时退化为链表

## 12. 线程安全问题

### 12.1 并发问题表现
- **数据不一致**：多线程 put 操作可能导致数据丢失
- **死循环**：JDK 1.7 中扩容可能导致链表成环
- **fast-fail**：迭代时修改会抛出 ConcurrentModificationException

### 12.2 线程安全方案
```java
// 方案1：使用 Collections.synchronizedMap
Map<String, String> syncMap = Collections.synchronizedMap(new HashMap<>());

// 方案2：使用 ConcurrentHashMap
Map<String, String> concurrentMap = new ConcurrentHashMap<>();
```

## 13. 使用建议

### 13.1 初始化容量选择
```java
// 预估元素数量，避免频繁扩容
int expectedSize = 100;
Map<String, String> map = new HashMap<>(expectedSize * 4 / 3 + 1);
```

### 13.2 键的设计要求
- 键对象必须正确实现 `equals()` 和 `hashCode()` 方法
- 好的哈希函数应均匀分布哈希值
- 使用不可变对象作为键（如 String、Integer）

### 13.3 迭代方式
```java
// 推荐：使用 EntrySet 迭代
for (Map.Entry<K, V> entry : map.entrySet()) {
    K key = entry.getKey();
    V value = entry.getValue();
}

// 不推荐：先获取 keySet 再 get
for (K key : map.keySet()) {
    V value = map.get(key);  // 额外哈希计算
}
```

## 14. 与相关类的比较

| 特性       | HashMap          | LinkedHashMap             | TreeMap                  | ConcurrentHashMap    |
| ---------- | ---------------- | ------------------------- | ------------------------ | -------------------- |
| 数据结构   | 数组+链表/红黑树 | 数组+链表/红黑树+双向链表 | 红黑树                   | 分段数组+链表/红黑树 |
| 顺序       | 无序             | 插入顺序或访问顺序        | 键的自然顺序或自定义顺序 | 无序                 |
| 线程安全   | 否               | 否                        | 否                       | 是                   |
| 允许 null  | 键值均可为 null  | 键值均可为 null           | 键不能为 null            | 键值均不能为 null    |
| 时间复杂度 | O(1) 平均        | O(1) 平均                 | O(log n)                 | O(1) 平均            |

## 15. 注意事项

1. **不要在多线程环境下直接使用 HashMap**
2. **选择合适的初始容量**以减少扩容次数
3. **确保键对象的不可变性**或在修改后重新放入
4. **重写 equals 必须重写 hashCode**
5. **关注内存使用**，及时清理不再使用的 HashMap

---

## 附录：完整示例代码

```java
import java.util.HashMap;
import java.util.Map;

public class HashMapDemo {
    public static void main(String[] args) {
        // 创建 HashMap
        Map<String, Integer> scores = new HashMap<>(16, 0.75f);
        
        // 添加元素
        scores.put("Alice", 95);
        scores.put("Bob", 88);
        scores.put("Charlie", 92);
        
        // 获取元素
        Integer aliceScore = scores.get("Alice");
        System.out.println("Alice's score: " + aliceScore);
        
        // 检查键是否存在
        boolean hasBob = scores.containsKey("Bob");
        System.out.println("Has Bob: " + hasBob);
        
        // 遍历所有元素
        System.out.println("All scores:");
        for (Map.Entry<String, Integer> entry : scores.entrySet()) {
            System.out.println(entry.getKey() + ": " + entry.getValue());
        }
        
        // 移除元素
        scores.remove("Charlie");
        System.out.println("After removal, size: " + scores.size());
        
        // 清空
        scores.clear();
        System.out.println("After clear, is empty: " + scores.isEmpty());
    }
}
```
