---
title: Autowired和Resource注解的区别
date: 2025-12-11 20:52:46
tags: [Java,Springboot]
---

- **两者都是用来做依赖注入的**，但
- `@Autowired` 是 **Spring 自家的注解，按类型（byType）优先**。
- `@Resource` 是 **JSR-250（标准规范）的注解，按名字（byName）优先**。

下面细一点说清楚区别。

------

## 1. 所属来源不同

- `@Autowired`
  - 包：`org.springframework.beans.factory.annotation.Autowired`
  - 只在 Spring 体系下有意义，是 Spring 自己的注解。
- `@Resource`
  - 包：`javax.annotation.Resource`（JDK8~） 或 `jakarta.annotation.Resource`（Jakarta EE）
  - 是 Java 标准（JSR-250）的一部分，各种容器（比如 Java EE 容器）都可以识别。

很多公司内部规范会说：*如果想写“与框架无关”的代码，用 @Resource 更“标准”一点。*

------

## 2. 注入规则差异（核心区别）

### 2.1 `@Autowired`：类型优先

**默认策略：byType（按类型查找）**

```java
@Autowired
private UserService userService;
```

Spring 会：

1. 先找容器里类型为 `UserService` 的 Bean。
2. 如果只有一个，直接注入。
3. 如果有多个同类型 Bean，会：
   - 看有没有按字段名匹配的 Bean 名称（`userService`）
   - 再看是否标了 `@Qualifier("xxx")`
   - 或者有没有 `@Primary` 的 Bean

配合 `@Qualifier` 的例子：

```java
@Autowired
@Qualifier("userServiceImpl1")
private UserService userService;
```

> 总之：**`@Autowired` 的本质是“先按类型找，再用其他信息（名字、@Qualifier、@Primary）消歧义”。**

------

### 2.2 `@Resource`：名字优先

**默认策略：byName（按名字查找），找不到再退回按类型。**

```java
@Resource
private UserService userService;
```

Spring 会：

1. 先按字段名 `"userService"` 去容器里找同名 Bean。
2. 如果找到了，就注入这个。
3. 如果找不到，再**按类型**找 `UserService`。
4. 如果按类型找到多个，就报错（名字又没指定，类型又不唯一）。

你也可以手动指定名称：

```java
@Resource(name = "userServiceImpl1")
private UserService userService;
```

这时候它就**只按这个 name 找**，找到就注入，找不到就报错，不会再按类型兜底。

> 总之：**`@Resource` 的本质是“先按名字找，再按类型兜底”。字段名 / name 属性特别重要。**

------

## 3. 属性差异：required / name / type

### `@Autowired`

- 有 `required` 属性：

```java
@Autowired(required = false)
private UserService userService;
```

- `required = true`（默认）：找不到就启动失败。
- `required = false`：找不到就注入 `null`，容器仍能启动。

### `@Resource`

- 没有 `required` 这个属性。
- 常用属性是：
  - `name`：指定 Bean 名称
  - `type`：指定 Bean 类型（一般不用，因为大多数情况可以自动推断）

------

## 4. 使用位置 & 构造器注入

- `@Autowired`

  - 可以用于：**字段、set 方法、构造方法、普通方法参数**。

  - 在 Spring 推荐的“构造器注入”里，很常见：

    ```java
    @Service
    public class UserService {
        private final UserRepository repo;
    
        @Autowired
        public UserService(UserRepository repo) {
            this.repo = repo;
        }
    }
    ```

- `@Resource`

  - 标准上主要是字段和 setter；
  - 在 Spring 里可以用在字段、setter 上，一般**不拿它做复杂的构造器注入**（语义上就不是这么设计的）。

------

## 5. 多个同类型 Bean 时的行为区别（易踩坑点）

假设你有两个同类型 Bean：

```java
@Service("userServiceImpl1")
public class UserServiceImpl1 implements UserService {}

@Service("userServiceImpl2")
public class UserServiceImpl2 implements UserService {}
```

### 情况 1：`@Autowired` + 接口类型字段

```java
@Autowired
private UserService userService;
```

- 容器里有两个 `UserService`，按类型匹配不唯一
- 又没有 `@Qualifier` 或 `@Primary`
- → **启动报错（NoUniqueBeanDefinitionException）**

------

### 情况 2：`@Resource` + 字段名恰好与 Bean 名匹配

```java
@Resource
private UserServiceImpl1 userServiceImpl1;
```

假设 `@Service("userServiceImpl1")`：

1. 它会先按 **字段名 `userServiceImpl1` → Bean name `userServiceImpl1`** 找到；
2. 直接注入，哪怕还有 `userServiceImpl2` 也没关系；
3. 不会报错。

所以实际项目里，如果 Bean 名和字段名“刚好对上了”，`@Resource` 会“莫名其妙地就能正常工作”，但换个字段名可能就挂了，这就是很多人感觉“它有点玄学”的原因。

------

## 6. 选用建议（简单记）

### 一句话记忆版本

- **更 Spring 化、支持构造器注入、结合 `@Qualifier/@Primary` 用：用 `@Autowired`。**
- **想走 Java 标准注解、按名字注入更直观：用 `@Resource`。**

### 实战中常见的约定

很多团队会统一约定其中一种用法：

- 要么统一用 `@Autowired + @Qualifier`（更灵活，偏 Spring）。
- 要么统一用 `@Resource` 做字段注入（名字清晰，代码稍“规范”一点）。

你可以根据**团队规范 + 个人习惯**来选，不要两种风格混乱使用就行。

------

## 7. 小总结表

| 对比项            | `@Autowired`                             | `@Resource`                              |
| ----------------- | ---------------------------------------- | ---------------------------------------- |
| 来源              | Spring 自带                              | JSR-250 标准（javax/jakarta.annotation） |
| 默认注入策略      | **按类型**（byType），再结合名称等消歧义 | **按名字**（byName），找不到再按类型     |
| 消歧义方式        | `@Qualifier` / `@Primary`                | `name` 属性 或 字段名                    |
| 是否有 `required` | 有（可选注入）                           | 没有                                     |
| 支持构造器注入    | ✅ 非常常用                               | 理论上可以，但很少这么用                 |
| 风格              | 明显 Spring 风格                         | 标准 Java 风格                           |

