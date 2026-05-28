---
title: ACM输入输出模板
date: 2026-03-27 17:18:41
tags: leetcode
---

下面我给你整理一份 **Java ACM 模式输入输出用法文档**。
我会以 **`Scanner`** 和 **`System.out`** 为主线，再把 ACM 里真正高频、你刷牛客时最容易卡住的补充内容一并写进去：**`StringBuilder`、`BufferedReader + StringTokenizer`、`PrintWriter`、常见模板、常见坑**。

------

# Java ACM 模式输入输出用法文档

## 1. ACM 模式的核心特点

和 LeetCode 核心代码模式不同，ACM 模式通常要求你自己完成：

- 输入读取
- 输出格式控制
- 主类 `Main`
- 主函数 `main`
- 有时还要处理多组输入、EOF、整行字符串、数组、矩阵等

最基本的 Java ACM 模板如下：

```java
import java.util.*;

// 注意类名必须为 Main，不要写 package
public class Main {
    public static void main(String[] args) {
        
    }
}
```

------

## 2. `Scanner`：最常用的输入类

`Scanner` 适合入门，写法直观，非常适合你现在刷牛客输入输出题。

### 2.1 基本创建方式

```java
Scanner sc = new Scanner(System.in);
```

------

## 3. `Scanner` 常用方法总结

### 3.1 读取整数

```java
int x = sc.nextInt();
```

适合读取：

- `1`
- `100`
- `-5`

------

### 3.2 读取长整数

```java
long x = sc.nextLong();
```

适合数据范围较大时使用，比如：

- `10^12`
- 多个大整数求和

------

### 3.3 读取浮点数

```java
double x = sc.nextDouble();
```

适合读取：

- `3.14`
- `2.718`
- 带小数的输入

------

### 3.4 读取单个字符串

```java
String s = sc.next();
```

特点：

- 遇到空格、换行、制表符就停止
- 适合读取一个单词、一个 token

例如输入：

```text
hello world
```

执行：

```java
String s = sc.next();
```

得到的是：

```text
hello
```

------

### 3.5 读取整行字符串

```java
String s = sc.nextLine();
```

特点：

- 读取一整行
- 行中间的空格会保留
- 适合读句子、命令、整行数组字符串

例如输入：

```text
hello world java
```

执行：

```java
String s = sc.nextLine();
```

得到的是：

```text
hello world java
```

------

### 3.6 判断是否还有输入

#### 读 token 级别

```java
while (sc.hasNext()) {
    ...
}
```

#### 读整数级别

```java
while (sc.hasNextInt()) {
    ...
}
```

#### 读整行级别

```java
while (sc.hasNextLine()) {
    ...
}
```

这个在 **EOF 模式** 里特别常见。

------

## 4. `Scanner` 在 ACM 里的典型用法

------

### 4.1 单组输入

输入：

```text
3 5
```

代码：

```java
Scanner sc = new Scanner(System.in);
int a = sc.nextInt();
int b = sc.nextInt();
System.out.println(a + b);
```

------

### 4.2 T 组输入

输入：

```text
3
1 2
3 4
5 6
```

代码：

```java
Scanner sc = new Scanner(System.in);
int T = sc.nextInt();

while (T-- > 0) {
    int a = sc.nextInt();
    int b = sc.nextInt();
    System.out.println(a + b);
}
```

------

### 4.3 EOF 多组输入

输入：

```text
1 2
3 4
5 6
```

代码：

```java
Scanner sc = new Scanner(System.in);

while (sc.hasNextInt()) {
    int a = sc.nextInt();
    int b = sc.nextInt();
    System.out.println(a + b);
}
```

------

### 4.4 读取整行带空格字符串

输入：

```text
hello world java
```

代码：

```java
Scanner sc = new Scanner(System.in);
String s = sc.nextLine();
System.out.println(s);
```

------

### 4.5 读取数组

输入：

```text
5
1 2 3 4 5
```

代码：

```java
Scanner sc = new Scanner(System.in);
int n = sc.nextInt();
int[] arr = new int[n];

for (int i = 0; i < n; i++) {
    arr[i] = sc.nextInt();
}
```

------

### 4.6 读取矩阵

输入：

```text
2 3
1 2 3
4 5 6
```

代码：

```java
Scanner sc = new Scanner(System.in);
int m = sc.nextInt();
int n = sc.nextInt();
int[][] a = new int[m][n];

for (int i = 0; i < m; i++) {
    for (int j = 0; j < n; j++) {
        a[i][j] = sc.nextInt();
    }
}
```

------

## 5. `Scanner` 最容易踩的坑

### 5.1 `nextInt()` 后面直接接 `nextLine()`

这是最常见的坑。

错误示例：

```java
int n = sc.nextInt();
String s = sc.nextLine();
```

问题在于：

- `nextInt()` 只读走数字
- 行末换行符还留着
- 下一次 `nextLine()` 读到的是空串

正确写法：

```java
int n = sc.nextInt();
sc.nextLine(); // 吃掉换行
String s = sc.nextLine();
```

更稳的写法：

```java
int n = Integer.parseInt(sc.nextLine());
String s = sc.nextLine();
```

------

### 5.2 把 `next()` 当成整行输入

```java
String s = sc.next();
```

这只能读到空格前。

要读整行带空格内容，必须用：

```java
String s = sc.nextLine();
```

------

### 5.3 输入格式和读取格式不一致

比如题目实际输入是：

```text
abc def
```

你却写：

```java
int n = sc.nextInt();
```

就会报：

```text
InputMismatchException
```

本质原因就是：**你以为当前位置是整数，实际却是字符串。**

------

## 6. `System.out`：最常用的输出方式

Java ACM 输出最常见的就是：

- `print`
- `println`
- `printf`

------

### 6.1 `print`

输出后不换行。

```java
System.out.print(123);
System.out.print(456);
```

输出：

```text
123456
```

------

### 6.2 `println`

输出后换行。

```java
System.out.println(123);
System.out.println(456);
```

输出：

```text
123
456
```

------

### 6.3 `printf`

格式化输出，ACM 里非常重要。

```java
System.out.printf("%.3f%n", 3.14159);
```

输出：

```text
3.142
```

------

## 7. `printf` 常见格式控制

### 7.1 保留 3 位小数

```java
System.out.printf("%.3f%n", x);
```

例如：

```java
double x = 3.1;
System.out.printf("%.3f%n", x);
```

输出：

```text
3.100
```

------

### 7.2 补前导零到 9 位

```java
System.out.printf("%09d%n", n);
```

例如：

```java
int n = 123;
System.out.printf("%09d%n", n);
```

输出：

```text
000000123
```

------

### 7.3 普通整数输出

```java
System.out.printf("%d%n", n);
```

------

### 7.4 字符串输出

```java
System.out.printf("%s%n", s);
```

------

## 8. 数组标准输出

ACM 中最稳妥的数组输出方式，是自己控制空格。

### 8.1 一维数组标准输出

```java
for (int i = 0; i < arr.length; i++) {
    if (i > 0) System.out.print(" ");
    System.out.print(arr[i]);
}
System.out.println();
```

输出效果：

```text
1 2 3 4 5
```

------

### 8.2 二维数组标准输出

```java
for (int i = 0; i < a.length; i++) {
    for (int j = 0; j < a[i].length; j++) {
        if (j > 0) System.out.print(" ");
        System.out.print(a[i][j]);
    }
    System.out.println();
}
```

------

### 8.3 调试时查看数组内容

#### 一维数组

```java
System.out.println(Arrays.toString(arr));
```

#### 二维数组

```java
System.out.println(Arrays.deepToString(arr));
```

注意：这两个更适合调试，不一定符合题目要求。

------

## 9. 为什么还要补充 `StringBuilder`

因为大量输出时，频繁 `print/println` 可能较慢。
ACM 里经常会先把结果拼起来，最后一次性输出。

### 9.1 基本写法

```java
StringBuilder sb = new StringBuilder();
sb.append(1).append(" ").append(2).append(" ").append(3);
System.out.println(sb.toString());
```

------

### 9.2 数组输出时配合使用

```java
StringBuilder sb = new StringBuilder();

for (int i = 0; i < arr.length; i++) {
    if (i > 0) sb.append(" ");
    sb.append(arr[i]);
}

System.out.println(sb.toString());
```

优点：

- 写法清晰
- 性能更稳
- 竞赛里很常见

------

## 10. 为什么还要补充 `BufferedReader + StringTokenizer`

因为 `Scanner` 虽然方便，但性能偏慢。
在数据量很大时，可能超时。这个时候 ACM 里常用更快的组合：

- `BufferedReader`：读输入
- `StringTokenizer`：拆分 token

------

### 10.1 快速输入模板

```java
import java.io.*;
import java.util.*;

public class Main {
    public static void main(String[] args) throws Exception {
        BufferedReader br = new BufferedReader(new InputStreamReader(System.in));
        StringTokenizer st = new StringTokenizer(br.readLine());

        int a = Integer.parseInt(st.nextToken());
        int b = Integer.parseInt(st.nextToken());

        System.out.println(a + b);
    }
}
```

------

### 10.2 为什么它快

因为：

- `BufferedReader` 按行读取，底层效率更高
- `StringTokenizer` 拆 token 比 `Scanner` 更轻量

------

### 10.3 什么时候优先用它

你现在阶段：

- 牛客基础输入输出题：优先 `Scanner`
- 大数据量竞赛题：考虑 `BufferedReader`

也就是说，`Scanner` 是你当前主力，`BufferedReader` 是后续进阶。

------

## 11. 为什么还要补充 `PrintWriter`

和 `StringBuilder` 类似，`PrintWriter` 更适合大规模输出。

### 11.1 基本写法

```java
import java.io.*;
import java.util.*;

public class Main {
    public static void main(String[] args) throws Exception {
        PrintWriter out = new PrintWriter(new BufferedWriter(new OutputStreamWriter(System.out)));
        out.println("hello");
        out.println("world");
        out.flush();
    }
}
```

注意最后一定要：

```java
out.flush();
```

------

## 12. ACM 模式最常用模板汇总

### 12.1 单组输入

```java
Scanner sc = new Scanner(System.in);
int a = sc.nextInt();
int b = sc.nextInt();
System.out.println(a + b);
```

------

### 12.2 T 组输入

```java
Scanner sc = new Scanner(System.in);
int T = sc.nextInt();

while (T-- > 0) {
    int a = sc.nextInt();
    int b = sc.nextInt();
    System.out.println(a + b);
}
```

------

### 12.3 EOF 输入

```java
Scanner sc = new Scanner(System.in);

while (sc.hasNextInt()) {
    int a = sc.nextInt();
    int b = sc.nextInt();
    System.out.println(a + b);
}
```

------

### 12.4 整行带空格输入

```java
Scanner sc = new Scanner(System.in);
String s = sc.nextLine();
System.out.println(s);
```

------

### 12.5 先读数字，再读整行

```java
Scanner sc = new Scanner(System.in);
int n = sc.nextInt();
sc.nextLine();
String s = sc.nextLine();
```

------

### 12.6 读数组

```java
Scanner sc = new Scanner(System.in);
int n = sc.nextInt();
int[] arr = new int[n];

for (int i = 0; i < n; i++) {
    arr[i] = sc.nextInt();
}
```

------

### 12.7 读矩阵

```java
Scanner sc = new Scanner(System.in);
int m = sc.nextInt();
int n = sc.nextInt();
int[][] a = new int[m][n];

for (int i = 0; i < m; i++) {
    for (int j = 0; j < n; j++) {
        a[i][j] = sc.nextInt();
    }
}
```

------

### 12.8 保留 3 位小数

```java
System.out.printf("%.3f%n", x);
```

------

### 12.9 补前导零到 9 位

```java
System.out.printf("%09d%n", n);
```

------

### 12.10 标准输出数组

```java
for (int i = 0; i < arr.length; i++) {
    if (i > 0) System.out.print(" ");
    System.out.print(arr[i]);
}
System.out.println();
```

------

## 13. 常见错误清单

### 13.1 类名写错

必须写：

```java
public class Main
```

------

### 13.2 写了 `package xxx`

ACM 平台一般不允许。

------

### 13.3 `nextInt()` 和 `nextLine()` 混用出错

记得吃掉换行。

------

### 13.4 直接 `System.out.println(arr)`

打印出来不是数组内容，而是地址信息。

------

### 13.5 输出多余空格或少换行

ACM 判题可能会严格比较格式。

------

### 13.6 题目要求多组输入，你却只读一组

看到 “多组输入” 一定先判断：

- 是不是 `T` 组
- 是不是 EOF

------

## 14. 你现阶段最推荐的使用策略

对于你现在刷牛客 ACM 输入输出题，我建议这样分层：

### 第一层：先彻底掌握 `Scanner + System.out`

先把下面这些练熟：

- `nextInt`
- `next`
- `nextLine`
- `hasNextInt`
- `print / println / printf`

这是当前最重要的。

### 第二层：补 `StringBuilder`

尤其是数组输出、多组结果输出。

### 第三层：了解 `BufferedReader + StringTokenizer + PrintWriter`

先会看懂，后面数据量大再正式上手。

------

## 15. 一句话速记版

你可以把这份文档最后压缩成下面这组口令：

- 读数字：`nextInt / nextLong / nextDouble`
- 读单词：`next()`
- 读整行：`nextLine()`
- 读到结束：`while (hasNext...)`
- 普通输出：`print / println`
- 格式化输出：`printf`
- 大量拼接输出：`StringBuilder`
- 更快输入：`BufferedReader + StringTokenizer`
- 更快输出：`PrintWriter`

------

## 16. 最后给你一份最实用的 ACM 入门总模板

```java
import java.util.*;

// 注意类名必须为 Main
public class Main {
    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        StringBuilder sb = new StringBuilder();

        int n = sc.nextInt();
        int[] arr = new int[n];

        for (int i = 0; i < n; i++) {
            arr[i] = sc.nextInt();
        }

        for (int i = 0; i < n; i++) {
            if (i > 0) sb.append(" ");
            sb.append(arr[i]);
        }

        System.out.println(sb.toString());
    }
}
```

这份模板已经把你现阶段最需要的三件事放一起了：

- `Scanner` 输入
- 数组处理
- `StringBuilder` 控制输出格式

需要下载版的话，我再把这份整理成一份 **Markdown 文档**。
