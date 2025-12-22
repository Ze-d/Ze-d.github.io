---
title: Spring AOP
date: 2025-12-11 11:41:04
tags: [Spring,Java]
---

我直接从三个层次来讲：**是什么 → 能干嘛 → Spring 具体怎么做（含代码示例）**。看完你基本就能自己写一个简单的 AOP 了。

------

## 一、AOP 是什么？跟 OOP 有啥关系？

- **OOP（面向对象）**：通过类、继承、多态，把**“核心业务逻辑”**封装起来，比如下单、支付、发货。
- **AOP（面向切面）**：通过“切面”，把一些跟核心业务“横着切过去”的功能**抽出来统一处理**，比如：
  - 事务（Transactional）
  - 日志
  - 权限校验
  - 性能统计
  - 异常监控

这些东西都有特点：

> 不属于某一个具体业务，但又“到处都要用”，如果每个方法里手写一遍会非常恶心。

AOP 的核心思想一句话：

> **在不改业务代码的前提下，在方法调用“前后、抛异常时”等时机，自动插入一段公共逻辑。**

------

## 二、AOP 里的几个关键概念（Spring 里也就是这些）

你在 Spring AOP 会经常看到这些词：

1. **Join Point（连接点）**

   - 程序中**可以被“切入”的点**。
   - 在 Spring AOP 里，**只能是方法的执行**（method execution）。
   - 比如：`UserService.createUser()` 的每次调用，就是一个连接点。

2. **Pointcut（切入点）**

   - 从一堆连接点中，**筛选出我要处理的那一部分**。

   - 用表达式来写，常见形式：

     ```java
     @Pointcut("execution(* com.example.service.*.*(..))")
     public void serviceMethods() {}
     ```

   解释一下这个表达式：

   - `execution`：拦截“方法执行”
   - `* com.example.service.*.*(..)`：
     - `*`：任意返回值
     - `com.example.service.*`：该包下任意类
     - 第二个 `*`：类中的任意方法
     - `(..)`：任意参数列表

3. **Advice（通知 / 增强）**
    就是你要“插进去的那段逻辑”，按执行时机分几种：

   - `@Before`：方法执行**之前**

   - `@AfterReturning`：方法**正常返回之后**

   - `@AfterThrowing`：方法**抛异常之后**

   - `@After`：方法结束之后（不管是否异常，有点类似 finally）

   - `@Around`：**环绕通知**，包住整个方法调用，最强大，也最常用：

     ```java
     @Around("serviceMethods()")
     public Object around(ProceedingJoinPoint pjp) throws Throwable {
         // 方法前
         Object result = pjp.proceed(); // 调用原方法
         // 方法后
         return result;
     }
     ```

4. **Aspect（切面）**

   - 把“切入点 + 通知代码”合在一起，就是一个切面。
   - 用 `@Aspect` 注解的类就是一个切面类。

5. **Target Object（目标对象）**

   - 被增强的业务 Bean，比如 `UserServiceImpl`。

6. **AOP Proxy（代理对象）**

   - Spring 不直接把 `UserServiceImpl` 给你，而是给你一个“代理对象”。
   - 所有调用经过“代理 → 再转给真实对象”，在代理里就可以做文章（织入增强逻辑）。
   - Spring 主要通过两种技术造代理：
     - JDK 动态代理（接口）
     - CGLIB 代理（子类）

7. **Weaving（织入）**

   - 把“切面增强”插入到目标对象执行流程中的过程就叫“织入”。
   - Spring AOP 是**运行期通过代理织入**，不改字节码。

------

## 三、Spring AOP 的工作机制（重要）

### 1. Spring AOP 是“代理驱动”的

一个典型过程：

1. Spring 启动时，扫描到你的 `@Aspect` 切面类；
2. 根据切点表达式，知道哪些 Bean 的哪些方法要被增强；
3. 为这些 Bean 生成“代理对象”：
   - 如果这个类 **实现了接口** → 默认用 **JDK 动态代理**
   - 否则 → 用 **CGLIB 动态继承生成子类代理**
4. 你在代码里从容器拿到的是**代理对象**（不是原始对象）；
5. 当你调用方法时：
   - 先进入代理 → 执行你配置的各类 Advice
   - 再通过反射调用原始方法。

### 2. 一个简单的图示（抽象理解）

```text
你写的代码：
   controller -> userService.createUser()

实际调用链：
   controller -> (代理对象 userServiceProxy) -> [AOP逻辑] -> (真实对象 userServiceImpl.createUser)
```

------

## 四、核心用法示例（@Aspect 基于注解的 AOP）

下面写一套你实际项目里可以直接抄的例子：**统计 service 方法执行耗时**。

### 1. 引导配置

在 Spring Boot 中，只要引了 `spring-boot-starter-aop`，自动就开启了 AOP。
 在非 Boot 环境，可以手动配置：

```java
@Configuration
@EnableAspectJAutoProxy(exposeProxy = true) // exposeProxy 后面讲“自调用”问题会用到
public class AopConfig {
}
```

### 2. 写一个切面

```java
@Aspect
@Component
public class ServiceLogAspect {

    // 1. 定义切点：所有 service 包下的 public 方法
    @Pointcut("execution(public * com.example.service..*(..))")
    public void serviceMethods() {}

    // 2. 环绕通知：统计执行时间
    @Around("serviceMethods()")
    public Object logExecutionTime(ProceedingJoinPoint pjp) throws Throwable {
        long start = System.currentTimeMillis();

        // 获取方法信息
        String className = pjp.getTarget().getClass().getSimpleName();
        String methodName = pjp.getSignature().getName();

        try {
            Object result = pjp.proceed();  // 调用原方法
            long cost = System.currentTimeMillis() - start;
            System.out.println("[AOP] " + className + "." + methodName + " 耗时 " + cost + " ms");
            return result;
        } catch (Throwable ex) {
            long cost = System.currentTimeMillis() - start;
            System.out.println("[AOP] " + className + "." + methodName + " 异常，耗时 " + cost + " ms");
            throw ex;
        }
    }
}
```

你啥都不用改，只要在 `com.example.service` 包下的 Bean 方法被调用，就会自动输出耗时日志。

### 3. 事务就是典型的 AOP

Spring 的 `@Transactional` 本质上也是通过 AOP 做的：

- 在方法前：开启事务 / 绑定连接
- 正常返回：提交事务
- 抛异常：回滚事务

你不需要自己写 `try { ... commit } catch { rollback }`，全部交给切面去做。

------

## 五、Spring AOP 常见“坑”和限制

### 1. 只能拦截 Spring 管理的 Bean

**前提条件：**

- 只有被 Spring 容器托管的 Bean，Spring 才会给它生成代理。
- 你如果自己 `new` 出来的对象（而不是通过 `@Autowired` / `ApplicationContext.getBean()` 获取），**AOP 不会生效**。

### 2. 自调用问题（调用自身方法时 AOP 失效）

```java
@Service
public class UserService {

    @Transactional
    public void createUser() {
        // ...
        this.innerLogic();  // 自调用
    }

    @Transactional
    public void innerLogic() {
        // ...
    }
}
```

上面这种写法，`innerLogic()` 上的 `@Transactional` 很可能**不生效**。
 原因是：`this.innerLogic()` 是在同一个对象内部直接调用，没有经过“代理对象”。

解决方式之一（常见手法）：

- 把需要增强的方法放到**另一个 Bean** 里，通过注入调用；
- 或者使用：`((UserService) AopContext.currentProxy()).innerLogic();`（需要 `@EnableAspectJAutoProxy(exposeProxy = true)`）。

### 3. 只支持方法级别的 AOP（Spring AOP）

- Spring AOP 只拦截“方法执行”。
- 如果想拦截更细，比如构造函数、字段访问，需要用到完整的 AspectJ 编译时织入 / 类加载期织入，这就不是单纯“Spring AOP”了。

------

## 六、JDK 动态代理 vs CGLIB：大致区别

Spring 默认策略：

- **如果 Bean 有接口**：用 **JDK 动态代理**
  - 代理对象是接口的实现类，类型是 `Proxy`。
- **如果 Bean 没有接口**：用 **CGLIB 生成子类代理**
  - 原理：继承目标类，重写方法，在里面加上增强逻辑。

可以通过配置强制使用 CGLIB：

```java
@EnableAspectJAutoProxy(proxyTargetClass = true)
```

一般只要知道**两者都是为 AOP 提供动态代理基础**，在业务开发层面不必太纠结。

------

## 七、AOP 典型应用场景小结

你以后看到这类需求，可以直接想到“AOP 来搞”：

1. **事务管理**：`@Transactional`
2. **统一日志**：记录入参、出参、耗时
3. **权限控制**：检查当前用户是否有权限调用某方法
4. **接口限流**：比如加 `@RateLimit` 注解，在切面里统计 QPS 后拒绝超限请求
5. **统一异常包装**：把底层异常转成统一的业务异常或统一响应格式
6. **审计日志**：记录操作行为（谁在何时对什么资源做了什么）

------

## 八、总结一句话版

- **OOP**：按“对象/模块”垂直切分系统；
- **AOP**：按“横向共性逻辑”水平切一刀，把“日志/事务/权限”这种“到处都有”的东西抽到切面里；
- **Spring AOP**：用动态代理在运行时包住目标 Bean 的方法执行，在方法的前后/异常等时机自动执行你的增强逻辑。

------

如果你愿意，下一步我可以帮你写一个完整的小 demo：
 比如“在所有 `@Controller` 的接口调用前后打印请求信息 + 耗时”，顺便拆解一下每一行代码的含义。
