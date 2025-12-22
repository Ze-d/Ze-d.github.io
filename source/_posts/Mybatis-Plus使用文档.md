---
title: Mybatis-Plus使用文档
date: 2025-12-11 21:08:53
tags: [Java,Springboot,Database]
---

## 1. MyBatis-Plus 概述

**MyBatis-Plus（MP）** 是在 MyBatis 基础上的增强工具包，在不改变 MyBatis 原有使用方式的前提下，提供：

- 单表 CRUD 的通用封装（几乎不用写 SQL）
- 灵活的条件构造器（Wrapper）
- 内置分页插件
- 逻辑删除、字段自动填充、乐观锁、多租户、代码生成器等能力

**设计理念：**

> 少写 SQL，多写业务逻辑；不强行接管，一切都可以回退到原生 MyBatis。

------

## 2. 集成与基础配置（Spring Boot 场景）

### 2.1 Maven 依赖示例

```xml
<properties>
    <java.version>17</java.version>
    <mybatis-plus.version>3.5.15</mybatis-plus.version>
</properties>

<dependencies>
    <!-- Spring Boot Web 基础 -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <!-- MyBatis-Plus Starter（Spring Boot 3+） -->
    <dependency>
        <groupId>com.baomidou</groupId>
        <artifactId>mybatis-plus-spring-boot3-starter</artifactId>
        <version>${mybatis-plus.version}</version>
    </dependency>

    <!-- 分页等插件（3.5.9+ 通常需要） -->
    <dependency>
        <groupId>com.baomidou</groupId>
        <artifactId>mybatis-plus-jsqlparser</artifactId>
        <version>${mybatis-plus.version}</version>
    </dependency>

    <!-- 数据库驱动 -->
    <dependency>
        <groupId>mysql</groupId>
        <artifactId>mysql-connector-j</artifactId>
    </dependency>
</dependencies>
```

### 2.2 application.yml 配置示例

```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/demo?useSSL=false&serverTimezone=Asia/Shanghai
    username: root
    password: root
    driver-class-name: com.mysql.cj.jdbc.Driver

mybatis-plus:
  mapper-locations: classpath*:mapper/*.xml
  type-aliases-package: com.example.demo.domain
  configuration:
    map-underscore-to-camel-case: true
  global-config:
    db-config:
      id-type: assign_id
      logic-delete-field: deleted
      logic-delete-value: 1
      logic-not-delete-value: 0
```

### 2.3 启动类配置

```java
@SpringBootApplication
@MapperScan("com.example.demo.mapper")
public class DemoApplication {

    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }
}
```

------

## 3. 核心组件与基本用法

### 3.1 核心组件

- **实体类（Entity）**：用注解映射表名、主键、字段等；
- **Mapper 接口**：`extends BaseMapper<T>`，自动获得通用 CRUD 方法；
- **Service 抽象层**：`IService<T> + ServiceImpl` 或新版本的 `CrudRepository<T, ID>`；
- **Wrapper 条件构造器**：
  - `QueryWrapper` / `LambdaQueryWrapper`
  - `UpdateWrapper` / `LambdaUpdateWrapper`
- **分页插件 + Page 对象**：实现物理分页查询。

### 3.2 示例表结构

```sql
CREATE TABLE user (
  id          BIGINT PRIMARY KEY,
  name        VARCHAR(50),
  age         INT,
  email       VARCHAR(100),
  deleted     TINYINT DEFAULT 0,
  create_time DATETIME,
  update_time DATETIME
);
```

### 3.3 实体类示例

```java
@Data
@TableName("user")
public class User {

    @TableId(type = IdType.ASSIGN_ID)
    private Long id;

    private String name;

    private Integer age;

    private String email;

    @TableLogic
    private Integer deleted;

    @TableField(fill = FieldFill.INSERT)
    private LocalDateTime createTime;

    @TableField(fill = FieldFill.INSERT_UPDATE)
    private LocalDateTime updateTime;
}
```

### 3.4 Mapper 接口示例

```java
@Mapper
public interface UserMapper extends BaseMapper<User> {
    // 可扩展自定义 SQL 方法
}
```

------

## 4. MyBatis-Plus 实现 CRUD 的几种方式

下文的 CRUD 指：

- C: Create（新增）
- R: Read（查询）
- U: Update（更新）
- D: Delete（删除/逻辑删除）

### 4.1 方式一：基于 BaseMapper 的 CRUD（DAO 级）

**定义：**
 `Mapper extends BaseMapper<Entity>`，在 Service 或 Controller 中直接使用 `mapper.xxx()` 调用。

```java
@Service
@RequiredArgsConstructor
public class UserAppService {

    private final UserMapper userMapper;

    public void crudWithMapper() {
        // === C: 新增 ===
        User u = new User();
        u.setName("张三");
        u.setAge(20);
        userMapper.insert(u);

        // === R: 主键查询 ===
        User user = userMapper.selectById(u.getId());

        // === R: 条件查询 ===
        List<User> list = userMapper.selectList(
                new LambdaQueryWrapper<User>()
                        .ge(User::getAge, 18)
                        .like(User::getName, "张")
        );

        // === U: 主键更新 ===
        user.setEmail("test@example.com");
        userMapper.updateById(user);

        // === D: 主键删除（若启用逻辑删除 → UPDATE）===
        userMapper.deleteById(user.getId());
    }
}
```

**特点：**

- 优点：直观、贴近 MyBatis 原始用法，控制粒度细；
- 缺点：业务层需要写较多 mapper 调用，通用逻辑需要自己封装。

------

### 4.2 方式二：基于 Service 抽象层的 CRUD（推荐）

MP 在 BaseMapper 之上，提供了通用 Service 抽象层：

- 经典写法：`IService<T>` + `ServiceImpl<M, T>`
- 新写法：`CrudRepository<T, ID>`（3.5.9+ 推荐，更像 Spring Data 风格）

**示例（以 IService 为例）**

```java
public interface UserService extends IService<User> {
    // 可以在这里扩展业务方法
}

@Service
public class UserServiceImpl extends ServiceImpl<UserMapper, User>
        implements UserService {
}
```

**使用：**

```java
@Service
@RequiredArgsConstructor
public class UserAppService {

    private final UserService userService;

    public void crudWithService() {
        // === C: 新增（单条 / 批量）===
        User u = new User();
        u.setName("李四");
        u.setAge(22);
        userService.save(u);

        List<User> batch = List.of(
                new User(), new User()
        );
        userService.saveBatch(batch);

        // === R: 单条 / 列表 / 分页 ===
        User byId = userService.getById(u.getId());

        List<User> adults = userService.list(
                new LambdaQueryWrapper<User>().ge(User::getAge, 18)
        );

        Page<User> page = userService.page(
                new Page<>(1, 10),
                new LambdaQueryWrapper<User>().ge(User::getAge, 18)
        );

        // === U: 更新 ===
        u.setEmail("lisi@example.com");
        userService.updateById(u);

        // 条件更新
        userService.update(
                new LambdaUpdateWrapper<User>()
                        .lt(User::getAge, 20)
                        .set(User::getEmail, "xxx@xxx.com")
        );

        // === D: 删除 ===
        userService.removeById(u.getId());
        userService.remove(
                new LambdaQueryWrapper<User>().lt(User::getAge, 18)
        );
    }
}
```

**特点：**

- 提供丰富的批量方法：`saveBatch`、`saveOrUpdate`、`removeByIds` 等；
- 业务层代码更整洁，**99% 业务建议通过 Service 层调用**；
- 新项目可以优先考虑 `CrudRepository` 方案，老项目继续使用 `IService` 问题不大。

------

### 4.3 方式三：ActiveRecord 模式（实体自带 CRUD）

**思想：** 实体继承 `Model<T>`，通过实体对象直接执行 CRUD。

```java
@Data
@TableName("user")
public class User extends Model<User> {

    @TableId(type = IdType.ASSIGN_ID)
    private Long id;
    private String name;
    private Integer age;
    private String email;
}
```

**使用：**

```java
public void crudWithActiveRecord() {
    // === C ===
    User u = new User();
    u.setName("王五");
    u.setAge(25);
    u.insert();

    // === R ===
    User u1 = u.selectById();    // 按自身 id
    User u2 = new User().selectById(1L);

    // === U ===
    u1.setEmail("ww@example.com");
    u1.updateById();

    // === D ===
    u1.deleteById();
}
```

**特点：**

- 写起来最像「面向对象操作数据库」；
- 但实体会耦合持久化逻辑，一些 DDD / 分层架构不推荐这种模式；
- 实战中更多用于 demo / 工具 / 小项目。

------

### 4.4 方式四：配合 Wrapper 的“花式 CRUD”

Wrapper 条件构造器既可以搭配 BaseMapper 使用，也可以搭配 Service 使用，是 MyBatis-Plus 核心“拼积木”工具。

#### 4.4.1 条件删除

```java
// 删除年龄 < 18 的用户
userService.remove(
        new LambdaQueryWrapper<User>().lt(User::getAge, 18)
);
```

#### 4.4.2 条件更新

```java
// 年龄 < 20 的用户，邮箱统一置空
userService.update(
        new LambdaUpdateWrapper<User>()
                .lt(User::getAge, 20)
                .set(User::getEmail, null)
);
```

#### 4.4.3 条件查询 + 分页

```java
Page<User> page = userService.page(
        new Page<>(1, 10),
        new LambdaQueryWrapper<User>()
                .between(User::getAge, 18, 30)
                .like(User::getName, "张")
                .orderByDesc(User::getId)
);
```

**建议重点掌握：**

- `LambdaQueryWrapper` / `LambdaUpdateWrapper`（避免字段写字符串，防止写错）
- 条件方法的“是否生效开关”：如 `like(condition, column, value)`，`condition` 为 `false` 时该条件自动忽略，便于根据前端入参动态拼装查询。

------

## 5. 分页插件与配置示例

### 5.1 分页插件配置

```java
@Configuration
public class MybatisPlusConfig {

    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor() {
        MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
        // 分页插件
        interceptor.addInnerInterceptor(
                new PaginationInnerInterceptor(DbType.MYSQL)
        );
        return interceptor;
    }
}
```

### 5.2 分页查询示例

```java
public Page<User> pageUsers(int pageNo, int pageSize) {
    Page<User> page = new Page<>(pageNo, pageSize);
    return userService.page(
            page,
            new LambdaQueryWrapper<User>()
                    .ge(User::getAge, 18)
    );
}
```

------

## 6. 进阶：逻辑删除、自动填充、乐观锁（简要）

### 6.1 逻辑删除

- 实体字段加 `@TableLogic`
- 在 `global-config.db-config` 里配置删除标记和未删除标记

之后：

- `deleteById` 会变成 `UPDATE xxx SET deleted = 1 WHERE id = ?`
- 所有查询都会自动带上 `deleted = 0`

### 6.2 字段自动填充

- 在实体上标注 `@TableField(fill = FieldFill.INSERT/INSERT_UPDATE)`
- 编写 `MetaObjectHandler` 实现，在 `insertFill`/`updateFill` 中填充时间或用户信息。

### 6.3 乐观锁

- 实体上添加 `@Version` 字段（如 `Integer version`）
- 注册 `OptimisticLockerInnerInterceptor`

更新时会携带 `version` 条件，成功后自动 `+1`，可用于控制并发写入。

------

## 7. 项目落地实践建议

1. **默认组合**（推荐）
   - DAO 层：`Mapper extends BaseMapper<T>`
   - Service 层：`IService<T> + ServiceImpl`（或 CrudRepository）
   - 业务层：优先通过 Service 调用，配合 `LambdaQueryWrapper`/`LambdaUpdateWrapper` 和分页插件。
2. **统一约定**
   - 表字段统一使用下划线命名，实体统一驼峰；
   - 所有表带 `deleted`、`create_time`、`update_time` 字段；
   - 所有列表接口默认分页。
3. **谨慎项**
   - 禁用“无条件全表更新/删除”：调用 `update`/`remove` 时务必检查 Wrapper 条件；
   - 大表查询必分页；
   - Wrapper 拼接条件时尽量使用 Lambda 方式，避免手写字符串字段名。

------

如果你愿意，可以告诉我你现在项目里的一张业务表（字段+含义），我可以基于这份文档，再给你补一套**完整的示例代码目录结构 + CRUD Demo**，直接贴到你项目里就能跑。
