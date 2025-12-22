---
title: Red-Black Tree
date: 2025-12-08 11:49:49
tags: [Data-Structure,Java]
---

## 1. 概述

红黑树是一种**自平衡的二叉查找树**，它在每个节点上增加一个存储位表示节点的颜色（红色或黑色），通过对任何一条从根到叶子的路径上各个节点的颜色约束，确保树的大致平衡。

## 2. 红黑树的性质

红黑树必须满足以下五个性质：

1. **每个节点要么是红色，要么是黑色**
2. **根节点必须是黑色**
3. **所有叶子节点（NIL节点）都是黑色**（实际实现中，通常用空指针表示叶子）
4. **红色节点的两个子节点必须是黑色**（即不能有连续的红色节点）
5. **从任一节点到其每个叶子的所有路径包含相同数量的黑色节点**

这些性质保证了红黑树的关键特性：**从根到叶子的最长路径不会超过最短路径的两倍**。

## 3. 红黑树 vs AVL树

| 特性      | 红黑树            | AVL树          |
| --------- | ----------------- | -------------- |
| 平衡标准  | 近似平衡          | 严格平衡       |
| 旋转频率  | 较低              | 较高           |
| 插入/删除 | 性能较好          | 需要更多旋转   |
| 查找      | O(log n)          | 稍快（更平衡） |
| 内存占用  | 1位颜色信息       | 高度或平衡因子 |
| 使用场景  | 需要频繁插入/删除 | 查询密集型应用 |

## 4. 红黑树的结构

### 4.1 节点结构
```java
class RBNode<K extends Comparable<K>, V> {
    K key;                 // 键
    V value;               // 值
    RBNode<K, V> left;     // 左子节点
    RBNode<K, V> right;    // 右子节点
    RBNode<K, V> parent;   // 父节点
    boolean color;         // 颜色（true=红，false=黑）
    
    public RBNode(K key, V value) {
        this.key = key;
        this.value = value;
        this.color = RED;  // 新节点默认为红色
    }
}
```

### 4.2 红黑树类结构
```java
public class RedBlackTree<K extends Comparable<K>, V> {
    private static final boolean RED = true;
    private static final boolean BLACK = false;
    
    private RBNode<K, V> root;
    private int size;
    
    // 获取节点颜色（处理null叶子节点）
    private boolean colorOf(RBNode<K, V> node) {
        return node == null ? BLACK : node.color;
    }
    
    // 设置节点颜色
    private void setColor(RBNode<K, V> node, boolean color) {
        if (node != null) {
            node.color = color;
        }
    }
}
```

## 5. 基本操作

### 5.1 旋转操作

#### 左旋（Left Rotate）
```java
private void leftRotate(RBNode<K, V> x) {
    // 1. 设置y为x的右子节点
    RBNode<K, V> y = x.right;
    
    // 2. 将y的左子树变为x的右子树
    x.right = y.left;
    if (y.left != null) {
        y.left.parent = x;
    }
    
    // 3. 连接y和x的父节点
    y.parent = x.parent;
    if (x.parent == null) {
        root = y;
    } else if (x == x.parent.left) {
        x.parent.left = y;
    } else {
        x.parent.right = y;
    }
    
    // 4. 将x设为y的左子节点
    y.left = x;
    x.parent = y;
}
```

#### 右旋（Right Rotate）
```java
private void rightRotate(RBNode<K, V> y) {
    // 1. 设置x为y的左子节点
    RBNode<K, V> x = y.left;
    
    // 2. 将x的右子树变为y的左子树
    y.left = x.right;
    if (x.right != null) {
        x.right.parent = y;
    }
    
    // 3. 连接x和y的父节点
    x.parent = y.parent;
    if (y.parent == null) {
        root = x;
    } else if (y == y.parent.left) {
        y.parent.left = x;
    } else {
        y.parent.right = x;
    }
    
    // 4. 将y设为x的右子节点
    x.right = y;
    y.parent = x;
}
```

### 5.2 插入操作

#### 插入步骤
1. 按照二叉查找树规则插入新节点（红色）
2. 根据不同的情况修复红黑树性质

```java
public void insert(K key, V value) {
    RBNode<K, V> newNode = new RBNode<>(key, value);
    
    // 1. 标准BST插入
    RBNode<K, V> parent = null;
    RBNode<K, V> current = root;
    
    while (current != null) {
        parent = current;
        int cmp = key.compareTo(current.key);
        if (cmp < 0) {
            current = current.left;
        } else if (cmp > 0) {
            current = current.right;
        } else {
            current.value = value; // 键已存在，更新值
            return;
        }
    }
    
    newNode.parent = parent;
    if (parent == null) {
        root = newNode;
    } else if (key.compareTo(parent.key) < 0) {
        parent.left = newNode;
    } else {
        parent.right = newNode;
    }
    
    // 2. 修复红黑树性质
    fixAfterInsertion(newNode);
    size++;
}
```

#### 插入修复（fixAfterInsertion）
有6种情况需要处理：

```java
private void fixAfterInsertion(RBNode<K, V> node) {
    // 新插入的节点默认为红色
    node.color = RED;
    
    while (node != null && node != root && node.parent.color == RED) {
        // 情况1：父节点是祖父节点的左子节点
        if (parentOf(node) == leftOf(parentOf(parentOf(node)))) {
            RBNode<K, V> uncle = rightOf(parentOf(parentOf(node)));
            
            // 情况1.1：叔叔节点是红色
            if (colorOf(uncle) == RED) {
                setColor(parentOf(node), BLACK);
                setColor(uncle, BLACK);
                setColor(parentOf(parentOf(node)), RED);
                node = parentOf(parentOf(node));
            } else {
                // 情况1.2：节点是父节点的右子节点（LR情况）
                if (node == rightOf(parentOf(node))) {
                    node = parentOf(node);
                    leftRotate(node);
                }
                // 情况1.3：节点是父节点的左子节点（LL情况）
                setColor(parentOf(node), BLACK);
                setColor(parentOf(parentOf(node)), RED);
                rightRotate(parentOf(parentOf(node)));
            }
        } else {
            // 对称情况：父节点是祖父节点的右子节点
            RBNode<K, V> uncle = leftOf(parentOf(parentOf(node)));
            
            // 情况2.1：叔叔节点是红色
            if (colorOf(uncle) == RED) {
                setColor(parentOf(node), BLACK);
                setColor(uncle, BLACK);
                setColor(parentOf(parentOf(node)), RED);
                node = parentOf(parentOf(node));
            } else {
                // 情况2.2：节点是父节点的左子节点（RL情况）
                if (node == leftOf(parentOf(node))) {
                    node = parentOf(node);
                    rightRotate(node);
                }
                // 情况2.3：节点是父节点的右子节点（RR情况）
                setColor(parentOf(node), BLACK);
                setColor(parentOf(parentOf(node)), RED);
                leftRotate(parentOf(parentOf(node)));
            }
        }
    }
    
    // 确保根节点为黑色
    root.color = BLACK;
}
```

### 5.3 删除操作

删除是红黑树最复杂的操作，有8种情况需要考虑。

#### 删除步骤
```java
public void delete(K key) {
    RBNode<K, V> node = findNode(key);
    if (node == null) return;
    
    deleteNode(node);
    size--;
}

private void deleteNode(RBNode<K, V> node) {
    // 情况1：节点有两个子节点
    if (node.left != null && node.right != null) {
        // 找到后继节点
        RBNode<K, V> successor = successor(node);
        // 用后继节点的值替换当前节点
        node.key = successor.key;
        node.value = successor.value;
        // 转而删除后继节点
        node = successor;
    }
    
    // 获取替代节点（如果存在）
    RBNode<K, V> replacement = (node.left != null) ? node.left : node.right;
    
    if (replacement != null) {
        // 情况2：节点有一个子节点
        replacement.parent = node.parent;
        if (node.parent == null) {
            root = replacement;
        } else if (node == node.parent.left) {
            node.parent.left = replacement;
        } else {
            node.parent.right = replacement;
        }
        
        // 删除节点
        node.left = node.right = node.parent = null;
        
        // 如果删除的是黑色节点，需要修复
        if (node.color == BLACK) {
            fixAfterDeletion(replacement);
        }
    } else if (node.parent == null) {
        // 情况3：节点是根节点且没有子节点
        root = null;
    } else {
        // 情况4：节点是叶子节点
        if (node.color == BLACK) {
            fixAfterDeletion(node);
        }
        
        // 从父节点移除引用
        if (node.parent != null) {
            if (node == node.parent.left) {
                node.parent.left = null;
            } else {
                node.parent.right = null;
            }
            node.parent = null;
        }
    }
}
```

#### 删除修复（fixAfterDeletion）
```java
private void fixAfterDeletion(RBNode<K, V> node) {
    while (node != root && colorOf(node) == BLACK) {
        if (node == leftOf(parentOf(node))) {
            // 情况1：节点是左子节点
            RBNode<K, V> sibling = rightOf(parentOf(node));
            
            // 情况1.1：兄弟节点是红色
            if (colorOf(sibling) == RED) {
                setColor(sibling, BLACK);
                setColor(parentOf(node), RED);
                leftRotate(parentOf(node));
                sibling = rightOf(parentOf(node));
            }
            
            // 情况1.2：兄弟节点的两个子节点都是黑色
            if (colorOf(leftOf(sibling)) == BLACK && 
                colorOf(rightOf(sibling)) == BLACK) {
                setColor(sibling, RED);
                node = parentOf(node);
            } else {
                // 情况1.3：兄弟节点的右子节点是黑色
                if (colorOf(rightOf(sibling)) == BLACK) {
                    setColor(leftOf(sibling), BLACK);
                    setColor(sibling, RED);
                    rightRotate(sibling);
                    sibling = rightOf(parentOf(node));
                }
                // 情况1.4：兄弟节点的右子节点是红色
                setColor(sibling, colorOf(parentOf(node)));
                setColor(parentOf(node), BLACK);
                setColor(rightOf(sibling), BLACK);
                leftRotate(parentOf(node));
                node = root;
            }
        } else {
            // 对称情况：节点是右子节点
            RBNode<K, V> sibling = leftOf(parentOf(node));
            
            if (colorOf(sibling) == RED) {
                setColor(sibling, BLACK);
                setColor(parentOf(node), RED);
                rightRotate(parentOf(node));
                sibling = leftOf(parentOf(node));
            }
            
            if (colorOf(rightOf(sibling)) == BLACK && 
                colorOf(leftOf(sibling)) == BLACK) {
                setColor(sibling, RED);
                node = parentOf(node);
            } else {
                if (colorOf(leftOf(sibling)) == BLACK) {
                    setColor(rightOf(sibling), BLACK);
                    setColor(sibling, RED);
                    leftRotate(sibling);
                    sibling = leftOf(parentOf(node));
                }
                setColor(sibling, colorOf(parentOf(node)));
                setColor(parentOf(node), BLACK);
                setColor(leftOf(sibling), BLACK);
                rightRotate(parentOf(node));
                node = root;
            }
        }
    }
    
    setColor(node, BLACK);
}
```

## 6. 红黑树在HashMap中的实现

### 6.1 TreeNode类
```java
static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {
    TreeNode<K,V> parent;  // 父节点
    TreeNode<K,V> left;    // 左子节点
    TreeNode<K,V> right;   // 右子节点
    TreeNode<K,V> prev;    // 前驱节点（链表结构）
    boolean red;           // 颜色
    
    // 构建红黑树
    final void treeify(Node<K,V>[] tab) {
        TreeNode<K,V> root = null;
        // 遍历链表，插入红黑树
        for (TreeNode<K,V> x = this, next; x != null; x = next) {
            next = (TreeNode<K,V>)x.next;
            x.left = x.right = null;
            if (root == null) {
                x.parent = null;
                x.red = false;
                root = x;
            } else {
                // 插入节点到红黑树
                K k = x.key;
                int h = x.hash;
                Class<?> kc = null;
                for (TreeNode<K,V> p = root;;) {
                    int dir, ph;
                    K pk = p.key;
                    if ((ph = p.hash) > h)
                        dir = -1;
                    else if (ph < h)
                        dir = 1;
                    else if ((kc == null &&
                              (kc = comparableClassFor(k)) == null) ||
                             (dir = compareComparables(kc, k, pk)) == 0)
                        dir = tieBreakOrder(k, pk);
                    
                    TreeNode<K,V> xp = p;
                    if ((p = (dir <= 0) ? p.left : p.right) == null) {
                        x.parent = xp;
                        if (dir <= 0)
                            xp.left = x;
                        else
                            xp.right = x;
                        // 插入后修复红黑树
                        root = balanceInsertion(root, x);
                        break;
                    }
                }
            }
        }
        moveRootToFront(tab, root);
    }
}
```

### 6.2 平衡插入（balanceInsertion）
```java
static <K,V> TreeNode<K,V> balanceInsertion(TreeNode<K,V> root,
                                            TreeNode<K,V> x) {
    x.red = true;
    for (TreeNode<K,V> xp, xpp, xppl, xppr;;) {
        if ((xp = x.parent) == null) {
            x.red = false;
            return x;
        }
        else if (!xp.red || (xpp = xp.parent) == null)
            return root;
        if (xp == (xppl = xpp.left)) {
            if ((xppr = xpp.right) != null && xppr.red) {
                xppr.red = false;
                xp.red = false;
                xpp.red = true;
                x = xpp;
            }
            else {
                if (x == xp.right) {
                    root = rotateLeft(root, x = xp);
                    xpp = (xp = x.parent) == null ? null : xp.parent;
                }
                if (xp != null) {
                    xp.red = false;
                    if (xpp != null) {
                        xpp.red = true;
                        root = rotateRight(root, xpp);
                    }
                }
            }
        }
        else {
            if (xppl != null && xppl.red) {
                xppl.red = false;
                xp.red = false;
                xpp.red = true;
                x = xpp;
            }
            else {
                if (x == xp.left) {
                    root = rotateRight(root, x = xp);
                    xpp = (xp = x.parent) == null ? null : xp.parent;
                }
                if (xp != null) {
                    xp.red = false;
                    if (xpp != null) {
                        xpp.red = true;
                        root = rotateLeft(root, xpp);
                    }
                }
            }
        }
    }
}
```

## 7. 红黑树的性能分析

### 7.1 时间复杂度
| 操作 | 平均情况 | 最坏情况 |
| ---- | -------- | -------- |
| 查找 | O(log n) | O(log n) |
| 插入 | O(log n) | O(log n) |
| 删除 | O(log n) | O(log n) |

### 7.2 空间复杂度
- 每个节点需要额外存储颜色信息（1位）
- 整体空间复杂度：O(n)

### 7.3 高度保证
红黑树的高度 h ≤ 2log₂(n+1)，其中 n 是节点数。

**证明思路**：
1. 设 bh(x) 为从节点 x 到任意叶子节点的黑色节点数（不包括 x）
2. 根据性质5，所有路径的 bh 相同
3. 根据性质4，红色节点不能连续，所以红色节点数 ≤ 黑色节点数
4. 因此总节点数 n ≥ 2^{bh(root)} - 1
5. 树高 h ≤ 2 * bh(root) ≤ 2log₂(n+1)

## 8. 红黑树的应用场景

### 8.1 在Java集合框架中
- **HashMap**：JDK 8+中，链表转红黑树
- **TreeMap**：基于红黑树的有序Map
- **TreeSet**：基于TreeMap实现

### 8.2 在操作系统和数据库中
- **Linux内核**：进程调度、内存管理
- **文件系统**：ext3文件系统的目录索引
- **数据库索引**：某些数据库的B+树变种

### 8.3 在实时系统中
由于红黑树的平衡性较好，查找、插入、删除的时间复杂度稳定在O(log n)，适合实时应用。

## 9. 完整示例：红黑树实现

```java
public class RedBlackTreeExample<K extends Comparable<K>, V> {
    private static final boolean RED = true;
    private static final boolean BLACK = false;
    
    private class Node {
        K key;
        V value;
        Node left, right, parent;
        boolean color;
        
        Node(K key, V value) {
            this.key = key;
            this.value = value;
            this.color = RED;
        }
    }
    
    private Node root;
    
    // 插入
    public void put(K key, V value) {
        Node node = new Node(key, value);
        if (root == null) {
            root = node;
            root.color = BLACK;
            return;
        }
        
        // 标准BST插入
        Node parent = null;
        Node current = root;
        while (current != null) {
            parent = current;
            int cmp = key.compareTo(current.key);
            if (cmp < 0) {
                current = current.left;
            } else if (cmp > 0) {
                current = current.right;
            } else {
                current.value = value;
                return;
            }
        }
        
        node.parent = parent;
        if (key.compareTo(parent.key) < 0) {
            parent.left = node;
        } else {
            parent.right = node;
        }
        
        fixAfterInsertion(node);
    }
    
    // 查找
    public V get(K key) {
        Node node = getNode(key);
        return node == null ? null : node.value;
    }
    
    private Node getNode(K key) {
        Node current = root;
        while (current != null) {
            int cmp = key.compareTo(current.key);
            if (cmp < 0) {
                current = current.left;
            } else if (cmp > 0) {
                current = current.right;
            } else {
                return current;
            }
        }
        return null;
    }
    
    // 中序遍历（有序输出）
    public void inOrderTraversal() {
        inOrder(root);
        System.out.println();
    }
    
    private void inOrder(Node node) {
        if (node != null) {
            inOrder(node.left);
            System.out.print(node.key + "(" + (node.color ? "R" : "B") + ") ");
            inOrder(node.right);
        }
    }
    
    // 验证红黑树性质
    public boolean isValid() {
        if (root == null) return true;
        
        // 性质2：根节点必须是黑色
        if (root.color != BLACK) {
            System.out.println("违反性质2：根节点不是黑色");
            return false;
        }
        
        // 检查所有路径的黑色节点数
        int blackCount = -1;
        return checkNode(root, 0, new int[]{blackCount});
    }
    
    private boolean checkNode(Node node, int blackCount, int[] expected) {
        if (node == null) {
            // 到达叶子节点（NIL）
            if (expected[0] == -1) {
                expected[0] = blackCount;
                return true;
            }
            return blackCount == expected[0];
        }
        
        // 计算当前路径的黑色节点数
        if (node.color == BLACK) {
            blackCount++;
        }
        
        // 性质4：红色节点不能有红色子节点
        if (node.color == RED) {
            if ((node.left != null && node.left.color == RED) ||
                (node.right != null && node.right.color == RED)) {
                System.out.println("违反性质4：红色节点有红色子节点");
                return false;
            }
        }
        
        return checkNode(node.left, blackCount, expected) &&
               checkNode(node.right, blackCount, expected);
    }
    
    // 辅助方法（旋转和修复，与前面代码类似）
    // ... 省略旋转和修复方法的实现 ...
}
```

## 10. 红黑树的优缺点

### 优点：
1. **平衡性较好**：最坏情况下的性能有保证
2. **旋转较少**：插入/删除时需要的旋转操作比AVL树少
3. **实现相对简单**：比AVL树等其他平衡树实现简单
4. **广泛应用**：经过充分测试和验证

### 缺点：
1. **实现复杂**：比普通二叉查找树复杂很多
2. **严格约束**：需要维护多个性质
3. **内存开销**：每个节点需要存储颜色信息

## 总结

红黑树是一种非常优秀的自平衡二叉查找树，它通过在节点中添加颜色信息并强制执行五个性质，确保了树的大致平衡。虽然实现相对复杂，但它在插入、删除和查找操作之间提供了很好的平衡，特别适合需要频繁更新的场景。在Java的HashMap中，红黑树用于处理哈希冲突严重的情况，将性能从O(n)提升到O(log n)。

## 与 2-3-4 树的关系

2-3-4 树是 4 阶 B 树，与一般的 B 树一样，2-3-4 树可以实现在
<math xmlns="http://www.w3.org/1998/Math/MathML" data-latex="O(\log n)"> O ( log ⁡ n ) </math> 时间内进行搜索、插入和删除操作。2-3-4 树的节点分为三种，2 节点、3 节点和 4 节点，分别包含一个、两个或三个数据元素。所有的叶子节点都处于同一深度（最底层），所有数据都有序存储。

2-3-4 树和红黑树是同构的，任意一棵红黑树都唯一对应一棵 2-3-4 树。在 2-3-4 树上的插入和删除操作导致节点的扩展、分裂和合并，相当于红黑树中的变色和旋转。下图是 2-3-4 树的 2 节点、3 节点和 4 节点对应的红黑树节点。注意到 2-3-4 树的 3 节点对应红黑树中红色节点左偏和右偏两种情况，所以一棵红黑树可能对应多棵 2-3-4 树。

可以通过对比 2-3-4 树来理解红黑树的插入和删除操作。^[2](https://oi-wiki.org/ds/rbtree/#fn:234-vs-rbt)^

## 实际工程项目中的使用

由于红黑树是目前主流工业界综合效率最高的内存型平衡树，其在实际的工程项目中有着广泛的使用，这里列举几个实际的使用案例并给出相应的源码链接，以便读者进行对比学习。

### Linux

源码：

* [`linux/lib/rbtree.c`](https://elixir.bootlin.com/linux/latest/source/lib/rbtree.c)

Linux 中的红黑树所有操作均使用循环迭代进行实现，保证效率的同时又增加了大量的注释来保证代码可读性，十分建议读者阅读学习。Linux 内核中的红黑树使用非常广泛，这里仅列举几个经典案例。

* [CFS 非实时任务调度](https://www.kernel.org/doc/html/latest/scheduler/sched-design-CFS.html)

  Linux 的稳定内核版本在 2.6.24 之后，使用了新的调度程序 CFS，所有非实时可运行进程都以虚拟运行时间为键值用一棵红黑树进行维护，以完成更公平高效地调度所有任务。CFS 弃用 active/expired 数组和动态计算优先级，不再跟踪任务的睡眠时间和区别是否交互任务，而是在调度中采用基于时间计算键值的红黑树来选取下一个任务，根据所有任务占用 CPU 时间的状态来确定调度任务优先级。

* [epoll](https://man7.org/linux/man-pages/man7/epoll.7.html)

  epoll 全称 event poll，是 Linux 内核实现 IO 多路复用 (IO multiplexing) 的一个实现，是原先 poll/select 的改进版。Linux 中 epoll 的实现选择使用红黑树来储存文件描述符。

### Nginx

源码：

* [`nginx/src/core/ngx_rbtree.h`](https://github.com/nginx/nginx/blob/master/src/core/ngx_rbtree.h)
* [`nginx/src/core/ngx_rbtree.c`](https://github.com/nginx/nginx/blob/master/src/core/ngx_rbtree.c)

nginx 中的用户态定时器是通过红黑树实现的。在 nginx 中，所有 timer 节点都由一棵红黑树进行维护，在 worker 进程的每一次循环中都会调用 `ngx_process_events_and_timers` 函数，在该函数中就会调用处理定时器的函数 `ngx_event_expire_timers`，每次该函数都不断的从红黑树中取出时间值最小的，查看他们是否已经超时，然后执行他们的函数，直到取出的节点的时间没有超时为止。

关于 nginx 中红黑树的源码分析公开资源很多，读者可以自行查找学习。

### C++

源码：

* GNU libstdc++

  * [`libstdc++-v3/include/bits/stl_tree.h`](https://github.com/gcc-mirror/gcc/blob/master/libstdc%2B%2B-v3/include/bits/stl_tree.h)
  * [`libstdc++-v3/src/c++98/tree.cc`](https://github.com/gcc-mirror/gcc/blob/master/libstdc%2B%2B-v3/src/c%2B%2B98/tree.cc)

  另外，`libstdc++` 在 `<ext/rb_tree>` 中提供了 [`__gnu_cxx::rb_tree`](https://github.com/gcc-mirror/gcc/blob/master/libstdc%2B%2B-v3/include/ext/rb_tree)，其继承了 `std::_Rb_tree`，可以认为是供外部使用的类型别名。需要注意的是，该头文件 **不是** C++ 标准的一部分，所以非必要不推荐使用。

  `libstdc++` 的 [`pb_ds`](https://oi-wiki.org/lang/pb-ds/tree/) 中也提供了红黑树。

* LLVM libcxx

  * [`libcxx/include/__tree`](https://github.com/llvm/llvm-project/blob/main/libcxx/include/__tree)

* Microsoft STL

  * [`stl/inc/xtree`](https://github.com/microsoft/STL/blob/main/stl/inc/xtree)

大多数 STL 中的 `std::set` 和 `std::map` 的内部数据结构就是红黑树（例如上面提到的这些）。不过值得注意的是，C++ 标准并未规定必须以红黑树实现 `std::set` 和 `std::map`，所以不应该在工程项目中直接使用 `std::set` 和 `std::map` 的内部数据结构。

### OpenJDK

源码：

* [`java.util.TreeMap<K, V>`](https://github.com/openjdk/jdk/blob/master/src/java.base/share/classes/java/util/TreeMap.java)
* [`java.util.TreeSet<K, V>`](https://github.com/openjdk/jdk/blob/master/src/java.base/share/classes/java/util/TreeSet.java)
* [`java.util.HashMap<K, V>`](https://github.com/openjdk/jdk/blob/master/src/java.base/share/classes/java/util/HashMap.java)

JDK 中的 `TreeMap` 和 `TreeSet` 都是使用红黑树作为底层数据结构的。同时在 JDK 1.8 之后 `HashMap` 内部哈希表中每个表项的链表长度超过 8 时也会自动转变为红黑树以提升查找效率。

## 参考资料

* Cormen, T. H., Leiserson, C. E., Rivest, R. L., \& Stein, C. (2022).*Introduction to algorithms*. MIT press.
* [Red-Black Tree - Wikipedia](https://en.wikipedia.org/wiki/Red%E2%80%93black_tree)
* [Red-Black Tree Visualization](https://www.cs.usfca.edu/~galles/visualization/RedBlack.html)

*** ** * ** ***

1. L. J. Guibas and R. Sedgewick, "A dichromatic framework for balanced trees,"*19^th^ Annual Symposium on Foundations of Computer Science (sfcs 1978)* , Ann Arbor, MI, USA, 1978, pp. 8-21, doi:[10.1109/SFCS.1978.3](https://doi.org/10.1109%2FSFCS.1978.3). [↩](https://oi-wiki.org/ds/rbtree/#fnref:gilbas1978 "Jump back to footnote 1 in the text")

2. [这篇博文](https://www.cnblogs.com/zhenbianshu/p/8185345.html) 提供了详细的描述。 [↩](https://oi-wiki.org/ds/rbtree/#fnref:234-vs-rbt "Jump back to footnote 2 in the text")

3. [https://en.wikipedia.org/wiki/Red--black_tree#cite_note-Cormen2009-18](https://en.wikipedia.org/wiki/Red%E2%80%93black_tree#cite_note-Cormen2009-18) [↩](https://oi-wiki.org/ds/rbtree/#fnref:cite_note-cormen2009-18 "Jump back to footnote 3 in the text")

4. [https://en.wikipedia.org/wiki/Red--black_tree#cite_note-Mehlhorn2008-17](https://en.wikipedia.org/wiki/Red%E2%80%93black_tree#cite_note-Mehlhorn2008-17) [↩](https://oi-wiki.org/ds/rbtree/#fnref:cite_note-mehlhorn2008-17 "Jump back to footnote 4 in the text")

5. [https://en.wikipedia.org/wiki/Red--black_tree#cite_note-Algs4-16](https://en.wikipedia.org/wiki/Red%E2%80%93black_tree#cite_note-Algs4-16): 432--447 [↩](https://oi-wiki.org/ds/rbtree/#fnref:cite_note-algs4-16 "Jump back to footnote 5 in the text")

