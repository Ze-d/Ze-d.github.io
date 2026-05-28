---
title: leetcode速查单
date: 2026-01-30 22:05:48
tags: leetcode
---

# 0. 通用方法论（口述 + 检查清单）

**面试口述 20 秒模板**

- “我先给出暴力，再找能否用哈希/双指针/单调栈/DP/二分把复杂度降下来；然后定义不变量（窗口合法、栈单调、BFS层序、check单调）；最后写边界与复杂度。”

**检查清单（最常翻车点）**

- 边界：空输入、1个元素、重复、负数、溢出（用 long）、不可达返回什么
- 更新顺序：先查后更 / 先更后查（前缀和、窗口、拓扑入度）
- visited 时机：**入队标记**（BFS）
- 递归语义：返回值是什么（树/DFS）

------

# 1. 数组 & 哈希（6 题）

**必刷题**

- 1 Two Sum（Easy）
- 49 Group Anagrams（Medium）
- 560 Subarray Sum Equals K（Medium）
- 128 Longest Consecutive Sequence（Medium）
- 238 Product of Array Except Self（Medium）
- 347 Top K Frequent Elements（Medium）

**最该背模板：前缀和 + 哈希计数**

[560. Subarray Sum Equals K](https://leetcode.com/problems/subarray-sum-equals-k/)

```java
public class Solution {
    public int subarraySum(int[] nums, int k) {
        int count = 0;
        int currSum = 0;
        // Map 存储 {前缀和 : 出现的次数}
        Map<Integer, Integer> prefixMap = new HashMap<>();
        // 初始化：前缀和为 0 已经出现过 1 次。处理了当 currSum 刚好等于 k 的情况
        prefixMap.put(0, 1);
        
        for (int num : nums) {
            // 1. 累加当前元素，得到当前前缀和
            currSum += num;
            
            // 2. 核心逻辑：
            // 如果 currSum - k 在 map 中存在，说明从某个旧位置到当前位置的子数组和为 k
            if (prefixMap.containsKey(currSum - k)) {
                count += prefixMap.get(currSum - k);
            }
            
            // 3. 将当前前缀和放入 map 或更新其出现的次数
            prefixMap.put(currSum, prefixMap.getOrDefault(currSum, 0) + 1);
        }
        
        return count;
    }
}
```

**重点/难点**

- “key 的选择”决定一切：value→index，prefix→count/firstIndex，signature→group
- 560/525 类：前缀和允许负数，滑窗不适用

**易错点**

- 用 int 存前缀和导致溢出（用 long）
- 前缀和最长区间：要 `putIfAbsent` 保留最早位置

------

# 2. 双指针 & 滑动窗口（8 题）

**必刷题**

- 3 Longest Substring Without Repeating Characters（Medium）
- 76 Minimum Window Substring（Hard）
- 11 Container With Most Water（Medium）
- 15 3Sum（Medium）
- 209 Minimum Size Subarray Sum（Medium）
- 283 Move Zeroes（Easy）
- 26 Remove Duplicates from Sorted Array（Easy）
- 844 Backspace String Compare（Easy/Medium，双指针练手）

**最该背模板：可变窗口**

[3. Longest Substring Without Repeating Characters](https://leetcode.com/problems/longest-substring-without-repeating-characters/)

这道题使用set，所以入窗的位置靠后。优化思路可以是使用数组（因为key非稀疏），或者是使用hashmap记录字符和最后出现的位置，优化跳跃。

```java
class Solution {
    public int lengthOfLongestSubstring(String s) {
        Set<Character> set = new HashSet<>();
        int left = 0;
        int res = 0;
        for (int i = 0; i < s.length(); i++) {
            // 1.入窗口
            // 2.收缩
            while(set.contains(s.charAt(i))) {
                set.remove(s.charAt(left));
                left++;
            }
            set.add(s.charAt(i));
            // 3.更新结果
            res = Math.max(res, i-left+1);
        }
        return res;
    }
}
```

**重点/难点**

- 明确“窗口代表什么”与“不变量”：合法/刚好合法
- 76：需要 `formed/required` 的思想（满足了多少种字符要求）

**易错点**

- `while` 写成 `if`（没收缩干净）
- 更新答案的位置错误（最短一般在 while 内）
- 3/76：字符计数更新顺序错导致 off-by-one

------

# 3. 栈 & 单调栈（6 题）

**必刷题**

- 20 Valid Parentheses（Easy）
- 155 Min Stack（Medium）
- 739 Daily Temperatures（Medium，Next Greater）
- 84 Largest Rectangle in Histogram（Hard，左右边界）
- 42 Trapping Rain Water（Hard，栈/双指针）
- 496 Next Greater Element I（Easy，巩固模板）

**最该背模板：Next Greater（单调递减栈存下标）**

[496. Next Greater Element I](https://leetcode.com/problems/next-greater-element-i/)

```java
class Solution {
    public int[] nextGreaterElement(int[] nums1, int[] nums2) {
        HashMap<Integer, Integer> map = new HashMap<>();
        Deque<Integer> deque = new ArrayDeque<>();
        int[] res = new int[nums1.length];
        for (int i = 0; i < nums2.length; i++) {
            while (!deque.isEmpty()&&deque.peek() < nums2[i]) {
                map.put(deque.pop(), nums2[i]);
            }
            deque.push(nums2[i]);
        }
        while (!deque.isEmpty()) {
            map.put(deque.pop(), -1);
        }
        for (int i = 0; i < nums1.length; i++) {
            res[i] = map.get(nums1[i]);
        }
        return res;
    }
}
```

**最该背模板：左右边界（84，单调递增 + 哨兵）**（todo）

```java
int[] h2 = Arrays.copyOf(h, h.length + 1); // 末尾0清栈
Deque<Integer> st = new ArrayDeque<>();
for (int i = 0; i < h2.length; i++) {
    while (!st.isEmpty() && h2[i] < h2[st.peek()]) {
        int mid = st.pop();
        int left = st.isEmpty() ? -1 : st.peek();
        int width = i - left - 1;
        ans = Math.max(ans, h2[mid] * width);
    }
    st.push(i);
}
```

**重点/难点**

- 栈里存**下标**不是值：距离/宽度都靠下标
- 弹栈瞬间：当前 i 是右边界，新栈顶是左边界

**易错点**

- 左边界计算错（`i - left - 1`）
- 忘记哨兵导致栈没清空、漏算面积
- 42 用栈时：接水高度取 `min(leftWall,rightWall)-midHeight`

------

# 4. 链表 & 指针技巧（7 题）

**必刷题**

- 206 Reverse Linked List（Easy）
- 21 Merge Two Sorted Lists（Easy）
- 19 Remove Nth Node From End（Medium）
- 141 Linked List Cycle（Easy）
- 143 Reorder List（Medium）
- 142 Linked List Cycle II（Medium，判环定位）
- 24 Swap Nodes in Pairs（Medium，指针操作强化）

**最该背模板：dummy 头 + 删除**

[82. Remove Duplicates from Sorted List II](https://leetcode.com/problems/remove-duplicates-from-sorted-list-ii/)

删除所有重复节点，（不保留重复节点）

```java
class Solution {
    public ListNode deleteDuplicates(ListNode head) {
        if (head==null||head.next==null) {
            return head;
        }
        ListNode dummy = new ListNode(0, head);
        ListNode cur = dummy;

        while (cur.next!=null&&cur.next.next!=null) {
            
            if (cur.next.val == cur.next.next.val) {
                int repeatval = cur.next.val;
                while (cur.next!=null &&cur.next.val==repeatval) {
                    cur.next = cur.next.next;
                }
            }else{
                cur=cur.next;
            }
        }
        return dummy.next;
    }
}
```

**重点/难点**

- dummy 统一“删头/插头”
- 改指针前保存 next 防丢链
- 快慢指针：判环、找中点、倒数第 k

**易错点**

- 反转时 next 保存顺序错
- 19：快指针先走 n+1 还是 n（取决于 dummy 是否使用）
- 143：分三步（找中点、反转后半、合并），顺序容易乱

------

# 5. 设计题（5 题）

**必刷题**

- 146 LRU Cache（Medium，必会）
- 155 Min Stack（Medium）
- 380 Insert Delete GetRandom O(1)（Medium）
- 622 Design Circular Queue（Medium）
- 295 Find Median from Data Stream（Hard，堆，设计味）

**最该背模板：LRU 结构骨架**

```java
class LRUCache {
    static class Node { int k,v; Node prev,next; Node(int k,int v){this.k=k;this.v=v;} }
    int cap;
    Map<Integer, Node> map = new HashMap<>();
    Node head = new Node(0,0), tail = new Node(0,0);

    public LRUCache(int capacity){
        cap = capacity;
        head.next = tail; tail.prev = head;
    }
    void remove(Node x){ x.prev.next = x.next; x.next.prev = x.prev; }
    void addFirst(Node x){ x.next = head.next; x.prev = head; head.next.prev = x; head.next = x; }

    public int get(int key){
        Node x = map.get(key);
        if (x == null) return -1;
        remove(x); addFirst(x);
        return x.v;
    }
    public void put(int key, int value){
        Node x = map.get(key);
        if (x != null) { x.v = value; remove(x); addFirst(x); return; }
        if (map.size() == cap) {
            Node lru = tail.prev;
            remove(lru); map.remove(lru.k);
        }
        Node nn = new Node(key, value);
        map.put(key, nn); addFirst(nn);
    }
}
```

**重点/难点**

- “O(1)”来自：HashMap 定位 + 双向链表移动
- 头最新、尾最旧，超容量删尾

**易错点**

- 忘记同步删 map
- remove/addFirst 指针更新遗漏
- cap=0 边界（面试偶尔会问）

------

# 6. 堆 / 优先队列（6 题）

**必刷题**

- 215 Kth Largest Element in an Array（Medium）
- 347 Top K Frequent Elements（Medium）
- 23 Merge k Sorted Lists（Hard，堆/分治）
- 295 Median from Data Stream（Hard）
- 973 K Closest Points to Origin（Medium）
- 703 Kth Largest in a Stream（Easy）

**最该背模板：TopK（小根堆保留K个）**

```java
PriorityQueue<int[]> pq = new PriorityQueue<>((a,b) -> a[0]-b[0]); // 按key最小
for (...) {
    pq.offer(new int[]{key, val});
    if (pq.size() > k) pq.poll();
}
```

**重点/难点**

- TopK：小根堆维持 K 个最大（或相反）
- K 路归并：堆里放当前最小节点

**易错点**

- 比较器溢出：用 `Integer.compare(a,b)`
- 23：弹出节点后记得把 `next` 入堆
- 295：两个堆平衡（size 差不超过 1）

------

# 7. 贪心 & 区间（6 题）

**必刷题**

- 56 Merge Intervals（Medium）
- 435 Non-overlapping Intervals（Medium）
- 452 Minimum Number of Arrows to Burst Balloons（Medium）
- 55 Jump Game（Medium）
- 45 Jump Game II（Medium）
- 134 Gas Station（Medium）

**最该背模板：区间（按右端点排序做选择）**

```java
Arrays.sort(intervals, (a,b) -> Integer.compare(a[1], b[1]));
int cnt = 0;
int end = Integer.MIN_VALUE;
for (int[] it : intervals) {
    if (it[0] >= end) { cnt++; end = it[1]; }
}
```

**重点/难点**

- 贪心要能说“为什么局部最优导出全局最优”
- 区间类：排序依据（右端点常用于“最多不重叠”）

**易错点**

- 端点是否算重叠（>= 还是 >，看题意）
- 452：用 long 防溢出（坐标可大）

------

# 8. 二叉树（8 题）

**必刷题**

- 102 Level Order Traversal（Medium）
- 104 Maximum Depth（Easy）
- 98 Validate BST（Medium）
- 236 LCA（Medium）
- 124 Binary Tree Maximum Path Sum（Hard）
- 105 Build Tree from Pre/Inorder（Medium）
- 199 Right Side View（Medium）
- 230 Kth Smallest in BST（Medium）

**最该背模板：DFS 自底向上（返回值语义）**

```java
int dfs(TreeNode node) {
    if (node == null) return 0;
    int L = dfs(node.left);
    int R = dfs(node.right);
    // combine L/R
    return ...;
}
```

**重点/难点**

- 递归函数语义必须清晰（返回高度？贡献？是否找到？）
- 124：返回“向上最大贡献”，全局更新“经过该点的最大路径”

**易错点**

- BST 校验用中序或上下界，注意 long 边界
- 105：哈希表存 inorder 位置，递归区间别写错

------

# 9. 图（8 题，覆盖 BFS/拓扑/DSU/最短路）

**必刷题**

- 200 Number of Islands（Medium，DFS/BFS）
- 994 Rotting Oranges（Medium，多源 BFS）
- 133 Clone Graph（Medium）
- 207 Course Schedule（Medium，拓扑判环）
- 210 Course Schedule II（Medium，输出序）
- 547 Number of Provinces（Medium，DSU）
- 684 Redundant Connection（Medium，DSU成环）
- 743 Network Delay Time（Medium，Dijkstra，偏算法岗常见）

**最该背模板：BFS（visited 入队标记 + 层序）**

```java
import java.util.*;

public class BFSTemplate {
    /**
     * @param graph  邻接表表示的图
     * @param start  起点
     * @param target 终点
     * @return 最短步数，若不通则返回 -1
     */
    public int minStep(Map<Integer, List<Integer>> graph, int start, int target) {
        // 1. 初始化：Queue 用于层序遍历，Set 用于入队标记
        Queue<Integer> queue = new LinkedList<>();
        Set<Integer> visited = new HashSet<>();
        // 2. 起点入队并标记（切记：入队即标记）
        queue.offer(start);
        visited.add(start);
        int step = 0; // 记录扩散的层数
        // 3. 执行 BFS
        while (!queue.isEmpty()) {
            // 【核心：层序遍历】在处理当前层之前，先记录当前层的节点数
            int size = queue.size();
            // 遍历当前层的所有节点
            for (int i = 0; i < size; i++) {
                int curr = queue.poll();
                // 判断是否到达目标
                if (curr == target) {
                    return step;
                }
                // 4. 遍历邻居节点
                List<Integer> neighbors = graph.getOrDefault(curr, new ArrayList<>());
                for (int neighbor : neighbors) {
                    if (!visited.contains(neighbor)) {
                        // 【核心：入队标记】先标记，再加入队列
                        visited.add(neighbor);
                        queue.offer(neighbor);
                    }
                }
            }
            // 处理完一层，步数自增
            step++;
        }
        return -1; // 目标不可达
    }
}
```

**注意点**

1. **分层遍历**：通过 `size = len(queue)`，你可以明确知道当前的 `step` 是什么。

2. **万能适配**：

   - 只要是**无权图**（每条边权重相同），这个模板求出来的 `step` 就是**最短路径**。
   - 如果是**二叉树**：去掉 `visited` 即可（因为树没有环）。
   - 如果是**矩阵/迷宫**：把 `graph.get(curr)` 换成 `上下左右` 四个方向坐标。

3. **绝对去重**：在 `append` 的同一行（或前一行）执行 `visited.add`。很多初学者死在“弹出才标记”，在大规模数据下会直接导致 **内存崩毁**。

**最该背模板：DSU**

```java
class DSU {
    int[] p, r;
    DSU(int n){ p=new int[n]; r=new int[n]; for(int i=0;i<n;i++)p[i]=i; }
    int find(int x){ return p[x]==x?x:(p[x]=find(p[x])); }
    boolean union(int a,int b){
        int ra=find(a), rb=find(b);
        if(ra==rb) return false;
        if(r[ra]<r[rb]) p[ra]=rb;
        else if(r[ra]>r[rb]) p[rb]=ra;
        else { p[rb]=ra; r[ra]++; }
        return true;
    }
}
```

**重点/难点**

- BFS：visited 必须入队标记；多源 BFS = 全源点先入队
- 拓扑：处理数< n 即有环
- Dijkstra：优先队列弹出的是当前最短“状态”，用 dist[] 剪枝

**易错点**

- Clone Graph：节点可能不连通？题通常从一个节点给，但 map 必须做去重
- Dijkstra：堆里可能有过期状态，要 `if (d>dist[u]) continue`

------

# 10. 二分（6 题）

**必刷题**

- 704 Binary Search（Easy）
- 34 Find First and Last Position（Medium）
- 33 Search in Rotated Sorted Array（Medium）
- 153 Find Minimum in Rotated Sorted Array（Medium）
- 875 Koko Eating Bananas（Medium，二分答案）
- 1011 Ship Within D Days（Medium，二分答案）

**最该背模板：二分答案（最小可行）**

```java
public int findLeftBoundary(int[] nums, int target) {
    int left = 0;
    int right = nums.length - 1; // 1. 闭区间 [left, right]

    while (left <= right) {      // 2. 闭区间统一使用 <=
        int mid = left + (right - left) / 2;
        
        if (nums[mid] < target) {
            left = mid + 1;      // 目标在右边，收缩左侧
        } else if (nums[mid] > target) {
            right = mid - 1;     // 目标在左边，收缩右侧
        } else if (nums[mid] == target) {
            // 找到目标了，但我不停！
            // 我要找最左边的，所以要锁定右边界，继续往左边压
            right = mid - 1; 
        }
    }

    // 3. 检查越界和是否真的找到了 target
    // 循环结束时，left 指向第一个大于等于 target 的位置
    if (left < nums.length && nums[left] == target) {
        return left;
    }
    return -1;
}
```

**重点/难点**

- 关键是证明 check 单调性
- 左/右边界二分要明确返回哪个端点

**易错点**

- mid 计算溢出（用 `lo + (hi-lo)/2`）
- 34：两个二分找左右边界，条件略不同

------

# 11. DP（8 题）

**必刷题**

- 70 Climbing Stairs（Easy）
- 198 House Robber（Medium）（记忆化搜索）
- 213 House Robber II（Medium）
- 322 Coin Change（Medium）（完全背包）
- 300 Longest Increasing Subsequence（Medium）
- 416 Partition Equal Subset Sum（Medium，01背包）
- 1143 Longest Common Subsequence（Medium）
- 62 Unique Paths（Medium，网格 DP）

**最该背模板：01背包倒序**

```java
public class Knapsack01 {
    /**
     * @param weights 物品重量数组
     * @param values  物品价值数组
     * @param W       背包最大容量
     * @return 最大价值
     */
    public int solve01Knapsack(int[] weights, int[] values, int W) {
        int n = weights.length;
        // 1. 初始化 dp 数组
        // dp[j] 表示容量为 j 的背包能获得的最大价值
        int[] dp = new int[W + 1];

        // 2. 遍历每一件物品
        for (int i = 0; i < n; i++) {
            // 3.【核心：倒序遍历】
            // 必须从背包最大容量 W 遍历到当前物品的重量 weights[i]
            for (int j = W; j >= weights[i]; j--) {
                // 4.状态转移方程：
                // 不放第 i 件物品：dp[j] (保持原值)
                // 放第 i 件物品：dp[j - weights[i]] + values[i]
                dp[j] = Math.max(dp[j], dp[j - weights[i]] + values[i]);
            }
        }

        return dp[W];
    }
}
```

**重点/难点**

- **遍历顺序和流程控制**：01背包一定要使用倒序遍历，防止重复使用元素。外层遍历物品，内层从最大容量开始遍历到当前重量，维护状态**转移方程。**
- **定义价值**：容量和价值应该如何建模，虚拟出价值，常见是重量=价值。
- **初始化：**最大价值：初始化为全0；恰好装满：除了0初始化为0，其他初始化为-∞，否则会存在未装满的情况。

**最该背模板：完全背包正序**

```java
public class Solution {
    public int coinChange(int[] coins, int amount) {
        // 1. 初始化 dp 数组
        // dp[j] 表示凑齐金额 j 所需的最少硬币数
        int[] dp = new int[amount + 1];

        // 2. 特殊初始化
        // 因为求的是“最小值”，所以除了 dp[0] 外，其他都初始化为一个“不可能的大值”
        // 使用 amount + 1 足够了，因为硬币最小面额是 1，硬币数不可能超过 amount
        int max = amount + 1;
        Arrays.fill(dp, max);
        dp[0] = 0; // 凑齐金额 0 准需要 0 个硬币

        // 3. 遍历物品（硬币）
        for (int i = 0; i < coins.length; i++) {
            // 【核心：正序遍历】
            // j 从当前硬币面值开始，直到 amount
            // 正序意味着：计算 dp[j] 时，可以使用已经更新过的 dp[j - coins[i]]
            // 这就是“无限次使用”物品的奥秘
            for (int j = coins[i]; j <= amount; j++) {
                // 状态转移：选当前硬币（硬币数+1）或不选
                dp[j] = Math.min(dp[j], dp[j - coins[i]] + 1);
            }
        }

        // 4. 结果判断
        return dp[amount] > amount ? -1 : dp[amount];
    }
}
```

**重点/难点**

1. **遍历顺序和流程控制：**外层遍历所有物品，内层从当前面值开始遍历到最大容量，维护状态转移矩阵
2. **初始化：**初始化为较大值即可，取Integer.MAX_VALUE容易溢出
3. **排列和组合：**组合数使用标准模板，排列数外层遍历容量，内存遍历物品（todo）

------

# 12. 回溯（6 题）

**必刷题**

- 78 Subsets（Medium）
- 46 Permutations（Medium）
- 39 Combination Sum（Medium）
- 40 Combination Sum II（Medium，去重）
- 22 Generate Parentheses（Medium）
- 79 Word Search（Medium）

**最该背模板：通用骨架**

```java
// 结果集
List<List<Integer>> res = new ArrayList<>();
// 路径（当前选择）
List<Integer> path = new ArrayList<>();

void backtrack(/* 状态参数 */) {
    // 1) 终止条件：满足题意就收集答案
    if (/* stop */) {
        res.add(new ArrayList<>(path));
        return;
    }

    // 2) 枚举“本层可选项”
    for (int i = /* begin */; i < /* end */; i++) {
        if (/* 剪枝/跳过 */) continue;

        // 3) 做选择
        path.add(/* choice */);

        // 4) 递归进入下一层（状态推进）
        backtrack(/* next state */);

        // 5) 撤销选择（回溯）
        path.remove(path.size() - 1);
    }
}
```

**重点/难点**

- 去重：同层去重 vs used 去重（排列）
- 剪枝：排序后可用 “超过目标就 break”

**易错点**

- path 引用复用导致结果全变（要 new ArrayList）
- Word Search：回溯后要恢复 visited/字符

------

# 13. 字符串专题（按岗位取舍，4 题）

**必刷题**

- 208 Implement Trie (Prefix Tree)（Medium）
- 211 Design Add and Search Words Data Structure（Medium）
- 5 Longest Palindromic Substring（Medium，中心扩展）
- 28 Find the Index of the First Occurrence in a String（Easy/Medium，KMP 可选）

**最该背模板：Trie 节点骨架**

```java
class Trie {
    static class Node {
        Node[] next = new Node[26];
        boolean end;
    }
    Node root = new Node();
    void insert(String w){
        Node cur = root;
        for(char ch: w.toCharArray()){
            int i = ch - 'a';
            if(cur.next[i]==null) cur.next[i]=new Node();
            cur = cur.next[i];
        }
        cur.end = true;
    }
    boolean search(String w){
        Node cur = root;
        for(char ch: w.toCharArray()){
            int i = ch - 'a';
            if(cur.next[i]==null) return false;
            cur = cur.next[i];
        }
        return cur.end;
    }
}
```

**重点/难点**

- 211：`.` 通配符需要 DFS 搜索多个分支
- 回文：中心扩展 O(n^2) 足够常见面试

**易错点**

- Trie 字符集不是 a-z 要改结构（面试会问）
- 211 DFS：递归终止条件与 end 判断

------

# 14. 位运算（4 题，加分项）

**必刷题**

- 136 Single Number（Easy）
- 191 Number of 1 Bits（Easy）
- 268 Missing Number（Easy）
- 231 Power of Two（Easy）

**最该背模板：lowbit / 去掉最低位 1**

```java
while (x != 0) {
    x &= (x - 1); // removes lowest set bit
    cnt++;
}
```

**重点/难点**

- 异或性质：a^a=0，a^0=a
- `x & (x-1)` 判 2 的幂

**易错点**

- Java int 有符号：位移与边界处理要小心（用 long 可避免）

------

# 15. 数学与杂项（3 题，低频）

**必刷题**

- 50 Pow(x, n)（Medium，快速幂）
- 169 Majority Element（Easy/Medium，投票法）
- 66 Plus One（Easy，细节模拟）

**最该背模板：快速幂**

```java
double fastPow(double x, long n){
    double ans = 1.0;
    long p = Math.abs(n);
    while (p > 0) {
        if ((p & 1) == 1) ans *= x;
        x *= x;
        p >>= 1;
    }
    return n >= 0 ? ans : 1.0 / ans;
}
```

**重点/难点**

- Pow：n 可能是 Integer.MIN_VALUE，先转 long
- 投票法：候选+计数，最后可选二次验证

**易错点**

- 溢出与符号边界（尤其是 n=-2^31）

------

## 最后：一份“面试前 1 小时必翻页”

如果你只复习最能提分的：

- 76（窗口）/ 84（单调栈）/ 146（LRU）/ 236（LCA）/ 124（树递归语义）/ 207（拓扑）/ 875（二分答案）/ 560（前缀和哈希）
