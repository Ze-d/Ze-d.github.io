---
title: Spring Boot事件监听
date: 2025-12-12 16:58:11
tags: SpringBoot
---

## 1. 通用设计原则

### 1.1 核心设计原则

#### 单一职责原则 (Single Responsibility)

每个事件类应该只包含一种类型的信息，每个监听器应该只处理一种业务逻辑。

```java
// ✅ 好的设计：单一职责
public class UserRegisterEvent extends ApplicationEvent {
    private Long userId;
    private String userName;
    private String email;
    // 只包含注册相关信息
}

// ❌ 错误设计：职责混乱
public class UserEvent extends ApplicationEvent {
    private Long userId;
    private String action; // REGISTER, LOGIN, LOGOUT, UPDATE...
    // 一个事件包含多种行为，违反SRP
}
```

#### 开闭原则 (Open-Closed)

对扩展开放，对修改封闭。通过事件监听器可以轻松扩展新功能，无需修改现有业务代码。

```java
// 原始业务：用户注册
public class UserService {
    public void register(UserRegisterReq req) {
        // 1. 保存用户
        userDao.save(req);
        // 2. 发布事件（无需修改）
        SpringUtil.publishEvent(new UserRegisterEvent(this, req.getUserId()));
    }
}

// 扩展功能1：发送欢迎邮件
@Component
public class WelcomeEmailListener {
    @EventListener
    @Async
    public void onUserRegister(UserRegisterEvent event) {
        emailService.sendWelcomeEmail(event.getUserId());
    }
}

// 扩展功能2：发放新人福利
@Component
public class NewUserGiftListener {
    @EventListener
    @Async
    public void onUserRegister(UserRegisterEvent event) {
        giftService.sendNewUserGift(event.getUserId());
    }
}

// 扩展功能3：同步用户数据到CRM
@Component
public class CrmSyncListener {
    @EventListener
    @Async
    public void onUserRegister(UserRegisterEvent event) {
        crmService.syncUser(event.getUserId());
    }
}
```

#### 依赖倒置原则 (Dependency Inversion)

业务代码依赖于抽象（事件），而不是具体实现（监听器）。

```java
// 业务代码只依赖ApplicationEvent抽象
public class OrderService {
    public void createOrder(OrderCreateReq req) {
        // 业务逻辑
        OrderDO order = // ... 创建订单
        // 通过事件解耦，不依赖具体监听器
        SpringUtil.publishEvent(new OrderCreateEvent(this, order));
    }
}

// 任何监听器都可以订阅这个事件，互不依赖
```

### 1.2 事件设计规范

#### 事件命名规范

- 使用**业务动作** + **Event** 后缀
- 动词用**过去时**（表示已发生的事件）

```java
// ✅ 规范命名
UserRegisterEvent       // 用户已注册
OrderPaidEvent          // 订单已支付
ArticlePublishedEvent   // 文章已发布
CommentDeletedEvent     // 评论已删除

// ❌ 不规范命名
UserRegister           // 缺少Event后缀
OrderPayEvent          // 过去时更准确
ArticlePublish         // 缺少ed
```

#### 事件字段设计

```java
// ✅ 包含完整上下文信息
public class OrderPaidEvent extends ApplicationEvent {
    private Long orderId;              // 订单ID（必须）
    private Long userId;               // 用户ID（必须）
    private BigDecimal amount;         // 支付金额（必须）
    private String payChannel;         // 支付渠道（可选）
    private Date payTime;              // 支付时间（可选）

    // 构造函数验证必填字段
    public OrderPaidEvent(Object source, Long orderId, Long userId, BigDecimal amount) {
        super(source);
        Assert.notNull(orderId, "订单ID不能为空");
        Assert.notNull(userId, "用户ID不能为空");
        Assert.notNull(amount, "支付金额不能为空");
        this.orderId = orderId;
        this.userId = userId;
        this       .amount = amount;
 this.payTime = new Date();
    }
}

// ❌ 缺少必要信息
public class OrderPaidEvent extends ApplicationEvent {
    private OrderDO order;  // 传递整个对象会增加耦合
}
```

---

## 2. 标准实现模式

### 2.1 事件定义模式

#### 模式1：简单事件（推荐）

适用于信息较少的场景。

```java
// 简单事件模板
public class SimpleEvent extends ApplicationEvent {
    private final String eventType;
    private final Long userId;
    private final Object data;

    public SimpleEvent(Object source, String eventType, Long userId, Object data) {
        super(source);
        this.eventType = eventType;
        this.userId = userId;
        this.data = data;
    }

    // Getter方法
    public String getEventType() { return eventType; }
    public Long getUserId() { return userId; }
    public Object getData() { return data; }
}
```

#### 模式2：强类型事件（推荐）

为每种业务场景定义专门的事件类。

```java
// 强类型事件模板
public class UserActionEvent extends ApplicationEvent {
    private UserActionEnum action;    // 行为类型
    private Long userId;              // 用户ID
    private Long targetId;            // 目标对象ID（如文章ID、评论ID）
    private String description;       // 描述信息

    public UserActionEvent(Object source, UserActionEnum action, Long userId, Long targetId) {
        super(source);
        this.action = action;
        this.userId = userId;
        this.targetId = targetId;
    }
}

// 使用枚举区分行为类型
public enum UserActionEnum {
    LOGIN, LOGOUT, REGISTER,
    COMMENT, LIKE, COLLECT, SHARE,
    FOLLOW, UNFOLLOW
}
```

#### 模式3：泛型事件（高级）

支持多种内容类型的事件。

```java
// 泛型事件模板
public class GenericEvent<T> extends ApplicationEvent {
    private final String eventType;
    private final T payload;  // 载荷数据

    public GenericEvent(Object source, String eventType, T payload) {
        super(source);
        this.eventType = eventType;
        this.payload = payload;
    }

    public String getEventType() { return eventType; }
    public T getPayload() { return payload; }
}
```

### 2.2 监听器实现模式

#### 模式1：单一监听器处理多种事件类型

```java
@Component
public class MultiEventListener {

    @EventListener
    @Async
    public void handleUserEvent(UserActionEvent event) {
        switch (event.getAction()) {
            case LOGIN:
                handleLogin(event);
                break;
            case LOGOUT:
                handleLogout(event);
                break;
            case COMMENT:
                handleComment(event);
                break;
        }
    }

    @EventListener
    @Async
    public void handleOrderEvent(OrderEvent event) {
        switch (event.getType()) {
            case CREATED:
                handleOrderCreated(event);
                break;
            case PAID:
                handleOrderPaid(event);
                break;
        }
    }

    // 业务处理方法
    private void handleLogin(UserActionEvent event) {
        // 更新用户最后登录时间
        userDao.updateLastLoginTime(event.getUserId(), new Date());
        // 记录登录日志
        logService.recordLoginLog(event.getUserId(), event.getTargetId());
    }
}
```

#### 模式2：一个监听器处理一种事件类型

```java
// 用户行为监听器
@Component
public class UserActionListener {

    @EventListener
    @Async
    public void onUserAction(UserActionEvent event) {
        // 只处理用户行为相关事件
        log.info("处理用户行为: {}", event);
        // 业务逻辑...
    }
}

// 订单事件监听器
@Component
public class OrderEventListener {

    @EventListener
    @Async
    public void onOrderEvent(OrderEvent event) {
        // 只处理订单相关事件
        log.info("处理订单事件: {}", event);
        // 业务逻辑...
    }
}
```

**推荐**: 使用模式2，每个监听器职责更清晰，更易维护。

### 2.3 事件发布模式

#### 模式1：直接在业务代码中发布事件

```java
@Service
public class UserService {

    public void register(UserRegisterReq req) {
        // 1. 业务逻辑
        Long userId = saveUser(req);

        // 2. 发布事件
        SpringUtil.publishEvent(new UserRegisterEvent(this, userId, req.getUserName()));
    }
}
```

#### 模式2：使用切面自动发布事件

```java
// 定义切面注解
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface EventPublisher {
    String eventType();
}

// 切面实现
@Aspect
@Component
public class EventPublisherAspect {

    @Around("@annotation(eventPublisher)")
    public Object around(ProceedingJoinPoint pjp, EventPublisher eventPublisher) throws Throwable {
        // 执行目标方法
        Object result = pjp.proceed();

        // 获取方法参数
        Object[] args = pjp.getArgs();

        // 发布事件
        SpringUtil.publishEvent(new GenericEvent<>(pjp.getTarget(),
                eventPublisher.eventType(), args[0]));

        return result;
    }
}

// 使用示例
public class OrderService {
    @EventPublisher(eventType = "ORDER_CREATED")
    public OrderDO createOrder(OrderCreateReq req) {
        // 创建订单
        return saveOrder(req);
    }
}
```

#### 模式3：使用模板方法封装发布逻辑

```java
// 事件发布模板
public abstract class EventPublisherTemplate {

    public final <T> void publishEvent(String eventType, T data) {
        try {
            // 1. 预处理（可选）
            beforePublish(eventType, data);

            // 2. 发布事件
            SpringUtil.publishEvent(new GenericEvent<>(this, eventType, data));

            // 3. 后处理（可选）
            afterPublish(eventType, data);

        } catch (Exception e) {
            handleError(eventType, data, e);
        }
    }

    protected void beforePublish(String eventType, Object data) {
        // 默认空实现，子类可覆盖
    }

    protected void afterPublish(String eventType, Object data) {
        // 默认空实现，子类可覆盖
    }

    protected void handleError(String eventType, Object data, Exception e) {
        // 默认日志记录，子类可覆盖
        log.error("事件发布失败: eventType={}, data={}", eventType, data, e);
    }
}

// 业务服务继承模板
@Service
public class OrderService extends EventPublisherTemplate {

    public OrderDO createOrder(OrderCreateReq req) {
        // 1. 执行业务逻辑
        OrderDO order = saveOrder(req);

        // 2. 发布事件（使用模板方法）
        publishEvent("ORDER_CREATED", order);

        return order;
    }

    @Override
    protected void afterPublish(String eventType, Object data) {
        if ("ORDER_CREATED".equals(eventType)) {
            // 可以添加特定后处理逻辑
            log.info("订单创建事件发布成功: {}", data);
        }
    }
}
```

---

## 3. 代码模板与工具

### 3.1 基础事件模板

#### Maven Archetype 模板

```xml
<!-- pom.xml 中添加模板配置 -->
<properties>
    <event.version>1.0.0</event.version>
</properties>

<dependencies>
    <!-- Spring Context -->
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-context</artifactId>
    </dependency>
</dependencies>
```

#### 事件基类

```java
package com.example.event;

import lombok.Data;
import org.springframework.context.ApplicationEvent;

/**
 * 基础事件类
 *
 * @param <T> 事件载荷类型
 */
@Data
public abstract class BaseEvent<T> extends ApplicationEvent {

    /**
     * 事件ID（唯一标识）
     */
    private final String eventId;

    /**
     * 事件发生时间
     */
    private final long timestamp;

    /**
     * 事件载荷数据
     */
    private final T payload;

    public BaseEvent(Object source, T payload) {
        super(source);
        this.eventId = UUID.randomUUID().toString();
        this.timestamp = System.currentTimeMillis();
        this.payload = payload;
    }

    /**
     * 事件是否超时（用于过期事件过滤）
     */
    public boolean isExpired(long expirationTimeMs) {
        return System.currentTimeMillis() - timestamp > expirationTimeMs;
    }
}
```

#### 事件监听器基类

```java
package com.example.event.listener;

import lombok.extern.slf4j.Slf4j;
import org.springframework.context.event.EventListener;
import org.springframework.scheduling.annotation.Async;

/**
 * 监听器基类（提供通用功能）
 */
@Slf4j
public abstract class BaseEventListener {

    /**
     * 异步处理事件的模板方法
     */
    @Async
    @EventListener
    public void handleEvent(BaseEvent<?> event) {
        try {
            log.info("开始处理事件: {}", event.getClass().getSimpleName());

            // 执行具体处理逻辑
            doHandle(event);

            log.info("事件处理完成: {}", event.getEventId());
        } catch (Exception e) {
            handleError(event, e);
        }
    }

    /**
     * 子类实现具体处理逻辑
     */
    protected abstract void doHandle(BaseEvent<?> event);

    /**
     * 错误处理（子类可覆盖）
     */
    protected void handleError(BaseEvent<?> event, Exception e) {
        log.error("事件处理失败: eventId={}, error={}", event.getEventId(), e.getMessage(), e);
    }
}
```

### 3.2 事件发布工具类

```java
package com.example.event;

import org.springframework.context.ApplicationEvent;
import org.springframework.stereotype.Component;

/**
 * 事件发布工具类
 */
@Component
public class EventPublisher {

    /**
     * 发布简单事件
     */
    public static void publish(String eventType, Object data) {
        SimpleEvent event = new SimpleEvent(getSource(), eventType, data);
        SpringUtil.publishEvent(event);
    }

    /**
     * 发布泛型事件
     */
    public static <T> void publishGeneric(String eventType, T data) {
        GenericEvent<T> event = new GenericEvent<>(getSource(), eventType, data);
        SpringUtil.publishEvent(event);
    }

    /**
     * 发布业务事件
 static <T extends     */
    public ApplicationEvent> void publishBusiness(T event) {
        SpringUtil.publishEvent(event);
    }

    private static Object getSource() {
        // 获取调用者的类名和方法名
        StackTraceElement[] stackTrace = Thread.currentThread().getStackTrace();
        StackTraceElement caller = stackTrace[2]; // 0: getStackTrace, 1: getSource, 2: 调用者
        return caller.getClassName() + "." + caller.getMethodName();
    }
}
```

### 3.3 事件总线工具类

```java
package com.example.event;

import java.util.List;
import java.util.concurrent.CopyOnWriteArrayList;

/**
 * 简单的事件总线（内存实现）
 * 注意：这是简化版，生产环境建议使用Spring Event或消息队列
 */
public class SimpleEventBus {

    private interface EventHandler {
        void handle(Object event);
    }

    private final List<EventHandler> handlers = new CopyOnWriteArrayList<>();

    /**
     * 注册事件处理器
     */
    public void register(EventHandler handler) {
        handlers.add(handler);
    }

    /**
     * 发布事件
     */
    public void post(Object event) {
        handlers.forEach(handler -> {
            try {
                handler.handle(event);
            } catch (Exception e) {
                // 记录错误但不影响其他处理器
                System.err.println("事件处理失败: " + e.getMessage());
            }
        });
    }
}
```

---

## 4. 常见问题与解决方案

### 4.1 事务问题

#### 问题描述

事件监听器中执行数据库操作时，可能出现事务不一致的问题。

```java
// 问题代码
@Service
public class OrderService {

    @Transactional
    public void createOrder(OrderCreateReq req) {
        // 1. 创建订单（数据库操作）
        OrderDO order = saveOrder(req);

        // 2. 发布事件
        SpringUtil.publishEvent(new OrderCreateEvent(this, order));

        // 如果这里抛出异常，订单会回滚，但事件已发布
    }
}

@Component
public class OrderListener {

    @EventListener
    public void onOrderCreate(OrderCreateEvent event) {
        // 监听器中的数据库操作可能不在事务中
        inventoryService.reserveStock(event.getOrderId());
    }
}
```

#### 解决方案

**方案1：使用 `@TransactionalEventListener`**

```java
@Component
public class OrderListener {

    /**
     * 只有在事务提交后才执行监听器
     */
    @TransactionalEventListener
    public void onOrderCreate(OrderCreateEvent event) {
        // 只在主事务提交后执行
        inventoryService.reserveStock(event.getOrderId());
    }

    /**
     * 事务回滚时执行
     */
    @TransactionalEventListener(phase = TransactionPhase.AFTER_ROLLBACK)
    public void onOrderRollback(OrderCreateEvent event) {
        // 事务回滚后的清理工作
        log.warn("订单创建失败，执行清理: {}", event.getOrderId());
    }
}
```

**方案2：在业务代码中处理事务边界**

```java
@Service
public class OrderService {

    @Transactional
    public void createOrder(OrderCreateReq req) {
        // 1. 创建订单
        OrderDO order = saveOrder(req);

        // 2. 提交当前事务
        TransactionStatus status = TransactionAspectSupport.currentTransactionStatus();
        status.flush(); // 确保数据写入数据库

        // 3. 发布事件（异步）
        SpringUtil.publishEvent(new OrderCreateEvent(this, order));
    }
}
```

### 4.2 循环依赖问题

#### 问题描述

事件监听器可能依赖其他bean，导致循环依赖。

```java
// 问题代码
@Service
public class UserService {
    @Autowired
    private OrderService orderService;

    public void register(UserRegisterReq req) {
        // 注册逻辑
        SpringUtil.publishEvent(new UserRegisterEvent(this, userId));
    }
}

@Component
public class UserListener {
    @Autowired
    private UserService userService;  // 循环依赖！

    @EventListener
    public void onUserRegister(UserRegisterEvent event) {
        userService.initializeUser(event.getUserId());
    }
}
```

#### 解决方案

**方案1：使用 `@Lazy` 延迟注入**

```java
@Component
public class UserListener {
    @Lazy
    @Autowired
    private UserService userService;

    @EventListener
    public void onUserRegister(UserRegisterEvent event) {
        // 实际使用时才加载
        userService.initializeUser(event.getUserId());
    }
}
```

**方案2：重构设计，避免循环依赖**

```java
// 重构：将逻辑提取到独立的服务中
@Service
public class UserInitService {  // 独立服务，避免循环依赖

    @Autowired
    private UserService userService;

    @Autowired
    private OrderService orderService;

    public void initializeUser(Long userId) {
        // 初始化用户相关数据
        userService.initProfile(userId);
        orderService.createWelcomeOrder(userId);
    }
}

@Component
public class UserListener {

    @Autowired
    private UserInitService userInitService;

    @EventListener
    public void onUserRegister(UserRegisterEvent event) {
        userInitService.initializeUser(event.getUserId());
    }
}
```

### 4.3 事件丢失问题

#### 问题描述

监听器执行失败时，事件丢失，无法重试。

#### 解决方案

**方案1：添加重试机制**

```java
@Component
public class RetryableEventListener {

    @EventListener
    @Async
    public void handleWithRetry(Event event) {
        int maxRetries = 3;
        int retryCount = 0;
        Exception lastException = null;

        while (retryCount < maxRetries) {
            try {
                // 执行处理逻辑
                doHandle(event);
                return; // 成功则退出
            } catch (Exception e) {
                lastException = e;
                retryCount++;
                log.warn("事件处理失败，第{}次重试: {}", retryCount, e.getMessage());

                try {
                    Thread.sleep(1000 * retryCount); // 指数退避
                } catch (InterruptedException ie) {
                    Thread.currentThread().interrupt();
                    break;
                }
            }
        }

        // 最终失败，记录日志或发送到死信队列
        log.error("事件处理最终失败: {}", event, lastException);
        sendToDeadLetterQueue(event, lastException);
    }
}
```

**方案2：使用消息队列持久化**

```java
// 生产者：发布事件到消息队列
@Service
public class OrderService {

    public void createOrder(OrderCreateReq req) {
        OrderDO order = saveOrder(req);
        // 发布到RabbitMQ
        rabbitTemplate.convertAndSend("order.exchange", "order.created", order);
    }
}

// 消费者：监听消息队列
@Component
public class OrderMessageListener {

    @RabbitListener(queues = "order.created.queue")
    public void handleOrderCreated(OrderDO order) {
        // 处理订单创建事件
        inventoryService.reserveStock(order.getId());
    }
}
```

### 4.4 性能问题

#### 问题描述

大量事件堆积，监听器执行缓慢。

#### 解决方案

**方案1：异步处理 + 线程池**

```java
@Configuration
@EnableAsync
public class AsyncConfig {

    @Bean("eventExecutor")
    public TaskExecutor eventExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(10);
        executor.setMaxPoolSize(50);
        executor.setQueueCapacity(200);
        executor.setKeepAliveSeconds(60);
        executor.setThreadNamePrefix("Event-");
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.initialize();
        return executor;
    }
}

@Component
public class EventListener {

    @Async("eventExecutor")  // 指定线程池
    @EventListener
    public void handleEvent(Event event) {
        // 异步处理
    }
}
```

**方案2：事件批处理**

```java
@Component
public class BatchEventListener {

    private final List<Event> buffer = new CopyOnWriteArrayList<>();
    private final int batchSize = 100;

    @EventListener
    public void onEvent(Event event) {
        buffer.add(event);

        if (buffer.size() >= batchSize) {
            processBatch();
        }
    }

    @Scheduled(fixedDelay = 5000) // 5秒处理一次
    public void processBatch() {
        if (!buffer.isEmpty()) {
            List<Event> batch = new ArrayList<>(buffer);
            buffer.clear();
            doProcessBatch(batch);
        }
    }

    private void doProcessBatch(List<Event> batch) {
        // 批量处理
        for (Event event : batch) {
            // 处理每个事件
        }
    }
}
```

---

## 5. 测试策略

### 5.1 单元测试

#### 测试事件发布

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

    @Mock
    private OrderDao orderDao;

    @InjectMocks
    private OrderService orderService;

    @Spy
    private ApplicationEventPublisher eventPublisher = new SimpleApplicationEventPublisher();

    @Test
    public void testCreateOrderShouldPublishEvent() {
        // Given
        OrderCreateReq req = new OrderCreateReq();
        req.setUserId(1L);
        req.setAmount(BigDecimal.valueOf(100));

        // When
        orderService.createOrder(req);

        // Then - 验证事件已发布
        ArgumentCaptor<OrderCreateEvent> captor =
            ArgumentCaptor.forClass(OrderCreateEvent.class);
        verify(eventPublisher).publishEvent(captor.capture());

        OrderCreateEvent event = captor.getValue();
        assertEquals(1L, event.getUserId());
        assertEquals(BigDecimal.valueOf(100), event.getAmount());
    }
}
```

#### 测试事件监听

```java
@ExtendWith(MockitoExtension.class)
class UserListenerTest {

    @Mock
    private EmailService emailService;

    @InjectMocks
    private UserListener userListener;

    @Test
    public void testOnUserRegisterShouldSendWelcomeEmail() {
        // Given
        UserRegisterEvent event = new UserRegisterEvent(this, 1L, "test@example.com");

        // When
        userListener.onUserRegister(event);

        // Then
        verify(emailService).sendWelcomeEmail(1L);
    }

    @Test
    public void testOnUserRegisterWhenEmailFailsShouldNotThrowException() {
        // Given
        UserRegisterEvent event = new UserRegisterEvent(this, 1L, "test@example.com");
        doThrow(new RuntimeException("邮件发送失败")).when(emailService).sendWelcomeEmail(anyLong());

        // When & Then - 不应抛出异常
        assertDoesNotThrow(() -> userListener.onUserRegister(event));
    }
}
```

### 5.2 集成测试

#### 测试事件流程

```java
@SpringBootTest
@TestMethodOrder(OrderAnnotation.class)
class EventIntegrationTest {

    @Autowired
    private ApplicationEventPublisher eventPublisher;

    @Autowired
    private UserService userService;

    @Autowired
    private UserListener userListener;

    @Spy
    private List<UserRegisterEvent> receivedEvents = new ArrayList<>();

    @BeforeEach
    public void setup() {
        receivedEvents.clear();
    }

    @Test
    @Order(1)
    public void testUserRegisterFlow() throws InterruptedException {
        // When - 用户注册
        userService.register(new UserRegisterReq("test", "test@example.com"));

        // Then - 验证事件被触发
        await().atMost(Duration.ofSeconds(5))
               .until(() -> receivedEvents.size() > 0);

        assertEquals(1, receivedEvents.size());
        assertEquals("test@example.com", receivedEvents.get(0).getEmail());
    }

    @EventListener
    public void captureEvent(UserRegisterEvent event) {
        receivedEvents.add(event);
    }
}
```

### 5.3 测试工具类

```java
/**
 * 事件测试工具类
 */
public class EventTestUtils {

    /**
     * 阻塞等待事件触发
     */
    public static <T> T waitForEvent(Class<T> eventClass, Duration timeout) {
        // 实现等待逻辑
    }

    /**
     * 获取所有发布的事件
     */
    public static List<ApplicationEvent> getPublishedEvents() {
        // 实现获取逻辑
    }

    /**
     * 清空事件列表
     */
    public static void clearEvents() {
        // 实现清空逻辑
    }
}
```

---

## 6. 性能优化

### 6.1 异步优化

#### 线程池配置

```java
@Configuration
@EnableAsync
public class AsyncEventConfig {

    /**
     * 通用事件处理线程池
     */
    @Bean("eventExecutor")
    public TaskExecutor eventExecutor() {
        return ThreadPoolTaskExecutorBuilder.newBuilder()
                .corePoolSize(10)
                .maxPoolSize(50)
                .queueCapacity(200)
                .keepAliveSeconds(60)
                .threadNamePrefix("Event-")
                .rejectedExecutionHandler(new ThreadPoolExecutor.AbortPolicy())
                .build();
    }

    /**
     * 慢事件处理线程池（用于耗时操作）
     */
    @Bean("slowEventExecutor")
    public TaskExecutor slowEventExecutor() {
        return ThreadPoolTaskExecutorBuilder.newBuilder()
                .corePoolSize(5)
                .maxPoolSize(20)
                .queueCapacity(100)
                .keepAliveSeconds(120)
                .threadNamePrefix("SlowEvent-")
                .rejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy())
                .build();
    }
}
```

#### 异步监听器示例

```java
@Component
public class OptimizedEventListener {

    /**
     * 快速事件（使用通用线程池）
     */
    @Async("eventExecutor")
    @EventListener
    public void handleFastEvent(FastEvent event) {
        // 快速处理，如更新缓存
        cacheService.put(event.getKey(), event.getValue());
    }

    /**
     * 慢事件（使用专用线程池）
     */
    @Async("slowEventExecutor")
    @EventListener
    public void handleSlowEvent(SlowEvent event) {
        // 慢速处理，如发送邮件、同步数据
        emailService.sendEmail(event.getTo(), event.getContent());
        crmService.syncData(event.getUserId());
    }
}
```

### 6.2 内存优化

#### 事件对象池

```java
/**
 * 事件对象池（避免频繁创建对象）
 */
public class EventObjectPool {

    private static final BlockingQueue<UserRegisterEvent> POOL =
        new LinkedBlockingQueue<>(1000);

    /**
     * 获取事件对象
     */
    public static UserRegisterEvent acquire() {
        UserRegisterEvent event = POOL.poll();
        return event != null ? event : new UserRegisterEvent(null, 0L, null);
    }

    /**
     * 释放事件对象
     */
    public static void release(UserRegisterEvent event) {
        // 重置对象状态
        event.setUserId(0L);
        event.setUserName(null);

        // 放回对象池
        POOL.offer(event);
    }
}

// 使用示例
public class UserService {
    public void register(UserRegisterReq req) {
        UserRegisterEvent event = EventObjectPool.acquire();
        try {
            event.setSource(this);
            event.setUserId(req.getUserId());
            event.setUserName(req.getUserName());

            SpringUtil.publishEvent(event);
        } finally {
            EventObjectPool.release(event);
        }
    }
}
```

#### 事件压缩

```java
/**
 * 事件压缩器（合并重复事件）
 */
@Component
public class EventCompressor {

    private final Map<String, Deque<Event>> eventBuffer = new ConcurrentHashMap<>();

    /**
     * 压缩事件（合并相同类型的事件）
     */
    public void compressAndPublish(Event event) {
        String key = event.getClass().getSimpleName() + ":" + event.getUserId();

        eventBuffer.computeIfAbsent(key, k -> new ConcurrentLinkedDeque<>()).add(event);

        // 如果缓冲达到阈值，发布压缩后的事件
        Deque<Event> deque = eventBuffer.get(key);
        if (deque.size() >= 10) {
            Event compressed = compress(deque);
            SpringUtil.publishEvent(compressed);
            deque.clear();
        }
    }

    private Event compress(Deque<Event> events) {
        // 简单示例：合并为批处理事件
        return new BatchEvent(events.toArray(new Event[0]));
    }
}
```

### 6.3 监控与度量

#### 事件监控

```java
/**
 * 事件监控指标
 */
@Component
public class EventMetrics {

    private final MeterRegistry meterRegistry;
    private final Counter eventPublishedCounter;
    private final Timer eventHandleTimer;

    public EventMetrics(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
        this.eventPublishedCounter = Counter.builder("events.published.total")
                .description("Total published events")
                .register(meterRegistry);
        this.eventHandleTimer = Timer.builder("events.handle.duration")
                .description("Event handling duration")
                .register(meterRegistry);
    }

    public void recordEventPublished(String eventType) {
        eventPublishedCounter.increment(
            Tags.of("event.type", eventType)
        );
    }

    public void recordEventHandleTime(String eventType, Duration duration) {
        eventHandleTimer.record(duration,
            Tags.of("event.type", eventType)
        );
    }
}
```

#### 事件监听器增强

```java
@Component
public class MonitoredEventListener {

    @Autowired
    private EventMetrics metrics;

    @EventListener
    @Async
    public void handleEvent(Event event) {
        String eventType = event.getClass().getSimpleName();
        metrics.recordEventPublished(eventType);

        Timer.Sample sample = Timer.start(meterRegistry);
        try {
            doHandle(event);
        } finally {
            sample.stop(Timer.builder("events.handle.duration")
                    .description("Event handling duration")
                    .register(meterRegistry, eventType));
        }
    }
}
```

---

## 7. 扩展场景

### 7.1 分布式事件

#### 使用 Redis 实现跨JVM事件

```java
/**
 * Redis事件发布器
 */
@Component
public class RedisEventPublisher {

    @Autowired
    private RedisTemplate<String, Object> redisTemplate;

    public void publish(String channel, Object event) {
        redisTemplate.convertAndSend(channel, event);
    }
}

/**
 * Redis事件监听器
 */
@Component
public class RedisEventListener {

    @Autowired
    private RedisTemplate<String, Object> redisTemplate;

    @PostConstruct
    public void init() {
        redisTemplate.getConnectionFactory()
            .getConnection()
            .subscribe(message -> {
                if ("user.events".equals(message.getChannel())) {
                    UserEvent event = (UserEvent) message.getData();
                    handleEvent(event);
                }
            }, "user.events".getBytes());
    }

    private void handleEvent(UserEvent event) {
        // 处理事件
    }
}
```

### 7.2 事件溯源 (Event Sourcing)

```java
/**
 * 事件存储
 */
@Entity
@Table(name = "events")
public class EventEntity {

    @Id
    private String eventId;

    private String aggregateId;  // 聚合根ID
    private String eventType;    // 事件类型
    private String eventData;    // 事件数据（JSON）
    private long timestamp;      // 时间戳
}

/**
 * 事件存储仓库
 */
@Repository
public interface EventRepository extends JpaRepository<EventEntity, String> {

    List<EventEntity> findByAggregateIdOrderByTimestamp(String aggregateId);
}

/**
 * 事件溯源服务
 */
@Service
public class EventSourcingService {

    @Autowired
    private EventRepository eventRepository;

    /**
     * 保存事件
     */
    public void saveEvent(DomainEvent event) {
        EventEntity entity = new EventEntity();
        entity.setEventId(event.getEventId());
        entity.setAggregateId(event.getAggregateId());
        entity.setEventType(event.getEventType());
        entity.setEventData(JSON.toJSONString(event));
        entity.setTimestamp(event.getTimestamp());

        eventRepository.save(entity);
    }

    /**
     * 获取聚合根的所有事件
     */
    public List<DomainEvent> getEvents(String aggregateId) {
        return eventRepository.findByAggregateIdOrderByTimestamp(aggregateId)
            .stream()
            .map(this::toDomainEvent)
            .collect(Collectors.toList());
    }
}
```

### 7.3 事件驱动架构 (EDA)

```java
/**
 * 事件路由器
 */
@Component
public class EventRouter {

    private final Map<String, List<EventHandler>> routes = new ConcurrentHashMap<>();

    /**
     * 注册事件处理器
     */
    public void register(String eventType, EventHandler handler) {
        routes.computeIfAbsent(eventType, k -> new CopyOnWriteArrayList<>()).add(handler);
    }

    /**
     * 路由事件
     */
    public void route(Event event) {
        String eventType = event.getClass().getSimpleName();
        List<EventHandler> handlers = routes.get(eventType);

        if (handlers != null) {
            handlers.forEach(handler -> {
                try {
                    handler.handle(event);
                } catch (Exception e) {
                    // 处理失败
                    log.error("事件处理失败: {}", eventType, e);
                }
            });
        }
    }
}

/**
 * 事件处理器接口
 */
@FunctionalInterface
public interface EventHandler {
    void handle(Event event) throws Exception;
}
```

---

## 8. 最佳实践清单

### 8.1 设计阶段

- [ ] **明确事件边界** - 事件应代表业务领域的已完成行为
- [ ] **定义清晰的事件类** - 每个事件类都有明确的职责
- [ ] **设计事件字段** - 包含必要的上下文信息，但不传递整个对象
- [ ] **考虑事件顺序** - 有序事件需要考虑时间戳或版本号
- [ ] **规划扩展点** - 为未来可能的监听器预留扩展空间

### 8.2 开发阶段

- [ ] **使用强类型事件** - 避免使用字符串类型的事件名
- [ ] **验证事件数据** - 在构造函数中验证必填字段
- [ ] **添加日志记录** - 记录关键业务事件
- [ ] **处理异常** - 监听器中不要抛出未处理的异常
- [ ] **使用泛型** - 提高事件系统的复用性

### 8.3 测试阶段

- [ ] **编写单元测试** - 测试事件发布和监听逻辑
- [ ] **编写集成测试** - 测试完整的事件流
- [ ] **测试异步处理** - 验证异步监听器的执行
- [ ] **测试事务边界** - 确保事件在正确的事务点触发
- [ ] **测试错误处理** - 验证异常情况下的行为

### 8.4 性能阶段

- [ ] **配置线程池** - 为异步监听器设置合理的线程池
- [ ] **监控事件量** - 跟踪事件发布和处理的性能
- [ ] **避免内存泄漏** - 及时清理监听器中的资源
- [ ] **批处理** - 对高频事件进行批处理
- [ ] **压缩事件** - 合并重复或相似的事件

### 8.5 运维阶段

- [ ] **监控事件失败** - 设置告警，当事件处理失败时及时通知
- [ ] **记录事件日志** - 保留关键事件的审计日志
- [ ] **定期清理** - 清理过期或已完成的事件数据
- [ ] **容量规划** - 根据事件量规划系统容量
- [ ] **文档维护** - 维护事件列表和监听器文档

---

## 9. 总结

通用的事件监听设计需要遵循以下核心原则：

1. **单一职责** - 每个事件和监听器只负责一种业务逻辑
2. **松散耦合** - 通过事件实现业务代码与副作用的分离
3. **异步处理** - 使用 `@Async` 提升系统性能
4. **可观测性** - 添加日志、监控和度量
5. **可测试性** - 编写全面的测试用例
6. **可维护性** - 清晰的命名和规范

通过遵循这些原则和最佳实践，可以构建一个健壮、可扩展的事件驱动系统。

---

**文档版本**: v2.0
**最后更新**: 2025-12-12
**作者**: zjy
