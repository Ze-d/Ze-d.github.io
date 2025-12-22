---
title: Java对象设计
date: 2025-12-03 19:12:23
tags: Java
---

## 一、6种对象的职责边界

**对象设计的本质是关注点分离** ------每个对象只做一件事，且做好它！

### 1.1 PO

它的含义是Persistent Object，即持久化对象。

* **职责** ：与数据库表严格1:1映射，**仅承载数据存储结构**
* **特征** ：
  * 属性与表字段完全对应
  * 无业务逻辑方法（仅有getter/setter）
* **代码示例** ：

```
`public` class UserPO {  
  private Long id;      // 对应表主键  
  private String name;  // 对应name字段
}  
```

### 1.2 DAO

它的含义是Data Access Object，即数据访问对象。

* **职责** ：**封装所有数据库操作** （CRUD），隔离业务与存储细节
* **特征** ：
  * 接口方法对应SQL操作
  * 返回PO或PO集合
* **代码示例** ：

```
`public` interface UserDao {  
    // 根据ID查询PO  
    UserPO findById(Long id);  
    
    // 分页查询  
    List<UserPO> findPage(@Param("offset") int offset, @Param("limit") int limit);  
}  
```

**底层原理** ：DAO模式 = 接口 + 实现类 + PO

![](https://cubox.pro/c/filters:no_upscale()?imageUrl=https%3A%2F%2Fimg2024.cnblogs.com%2Fblog%2F551042%2F202509%2F551042-20250916140602038-831079803.webp&valid=false)

### 1.3 BO

它的含义是Business Object，即业务对象。

* **职责** ：**封装核心业务逻辑** ，聚合多个PO完成复杂操作
* **特征** ：
  * 包含业务状态机、校验规则
  * 可持有多个PO引用
* **代码示例** ：订单退款BO

```
`public` class OrderBO {  
   // 主订单数据  
   private OrderPO orderPO;  
   // 子订单项  
   private List<OrderItemPO> items; 
   
   // 业务方法：执行退款  
   public RefundResult refund(String reason) {     
   if (!"PAID".equals(orderPO.getStatus())) {  
      throw new IllegalStateException("未支付订单不可退款"); 
   }  // 计算退款金额、调用支付网关等  }  
}  
```

### 1.4 DTO

它的含义是Data Transfer Object，即数据传输对象。

* **职责** ：**跨层/跨服务数据传输** ，屏蔽敏感字段
* **特征** ：
  * 属性集是PO的子集（如排除password字段）
  * 支持序列化（实现Serializable）
* **代码示例** ：用户信息DTO

```
`public` class UserDTO implements Serializable {  
    private Long id;  
    private String name;  
}  
```

### 1.5 VO

它的含义是View Object，即视图对象。

* **职责** ：**适配前端展示** ，包含渲染逻辑
* **特征** ：
  * 属性可包含格式化数据（如日期转yyyy-MM-dd）
  * 聚合多表数据（如订单VO包含用户名字）
* **代码示例** ：

```
`public` class OrderVO {  
  private String orderNo;  
  private String createTime; // 格式化后的日期   private String userName;   // 关联用户表字段    
  
  //状态码转文字描述  
  public String getStatusText() {  
     return OrderStatus.of(this.status).getDesc();  
  }  
}  
```

### 1.6 POJO

它的含义是Plain Old Java Object，即普通Java对象。

* **职责** ：**基础数据容器** ，可扮演PO/DTO/VO角色
* **特征** ：
  * 只有属性+getter/setter
  * 无框架依赖（如不继承Spring类）
* **典型实现** ：Lombok简化代码

```
`// 自动生成getter/setter  `
@Data 
public class UserPOJO {  
  private Long id;  
  private String name;  
}  
```

## 二、主流的对象流转模型

### 场景1

传统三层架构（DAO → DTO → VO）。

**适用系统** ：后台管理系统、工具类应用

**核心流程** ：

![](https://cubox.pro/c/filters:no_upscale()?imageUrl=https%3A%2F%2Fimg2024.cnblogs.com%2Fblog%2F551042%2F202509%2F551042-20250916140602073-1052544562.webp&valid=false)

**代码示例** ：用户查询服务

```
`// Service层  `
public UserDTO getUserById(Long id) {  
   UserPO userPO = userDao.findById(id); // 从DAO获取PO  
   UserDTO dto = new UserDTO();  
   dto.setId(userPO.getId());  
   dto.setName(userPO.getName()); // 过滤敏感字段  
   return dto; // 返回DTO  
}  

// Controller层  
public UserVO getUser(Long id) {  
   UserDTO dto = userService.getUserById(id);  
   UserVO vo = new UserVO();  
   vo.setUserId(dto.getId());  
   vo.setUserName(dto.getName());  
   vo.setRegisterTime(formatDate(dto.getCreateTime())); // 格式化日期  
   return vo;  
}  
```

**优点** ：简单直接，适合CRUD场景

**缺点** ：业务逻辑易泄漏到Service层

### 场景2

DDD架构（PO → DO → DTO → VO）。

**适用系统** ：电商、金融等复杂业务系统

**核心流程** ：

![](https://cubox.pro/c/filters:no_upscale()?imageUrl=https%3A%2F%2Fimg2024.cnblogs.com%2Fblog%2F551042%2F202509%2F551042-20250916140602101-1197010049.webp&valid=false)

**关键角色** ：DO（Domain Object）替代BO

**代码示例** ：订单支付域

```
`// Domain层：订单领域对象  `
publicclass OrderDO {  
   private OrderPO orderPO;  
   private PaymentPO paymentPO;  
   
   // 业务方法：支付校验  
   public void validatePayment() {  
     if (paymentPO.getAmount() < orderPO.getTotalAmount()) {  
        thrownew PaymentException("支付金额不足");  
      } 
   }  
}  

// App层：协调领域对象  
public OrderPaymentDTO pay(OrderPayCmd cmd) {  
    OrderDO order = orderRepo.findById(cmd.getOrderId());  
    order.validatePayment(); // 调用领域方法  return OrderConverter.toDTO(order); // 转DTO  
}  
```

**优点** ：业务高内聚，适合复杂规则系统

**缺点** ：转换层级多，开发成本高

## 三、高效转换工具

手动转换对象？效率低且易错！

### 3.1 MapStruct：编译期代码生成

**原理** ：APT注解处理器生成转换代码

**示例** ：PO转DTO

```
`@Mapper`  
publicinterface UserConverter {  
   UserConverter INSTANCE = Mappers.getMapper(UserConverter.class);  
   
   @Mapping(source = "createTime", target = "registerDate")  
   UserDTO poToDto(UserPO po);  
}  

// 编译后生成UserConverterImpl.java  
publicclass UserConverterImpl {  
   public UserDTO poToDto(UserPO po) {  
      UserDTO dto = new UserDTO();  
      dto.setRegisterDate(po.getCreateTime()); // 自动赋值！  
      return dto;  
  }  
}  
```

**优点** ：零反射损耗，性能接近手写代码

**开源地址** ：https://github.com/mapstruct/mapstruct

### 3.2 Dozer + Lombok：注解驱动转换

**组合方案** ：

* **Lombok** ：自动生成getter/setter
* **Dozer** ：XML/注解配置字段映射

```
`// Lombok注解`
@Data 
public class UserVO {  
   private String userId;  
   private String userName;  
}  

// 转换配置  
<field>  
  <a>userId</a>  
  <b>id</b>  
</field>  
```

**适用场景** ：字段名不一致的复杂转换

### 3.3 手动Builder模式：精细控制

**适用场景** ：需要动态构造的VO

```
`public` class OrderVOBuilder {  
   public OrderVO build(OrderDTO dto) {  
     return OrderVO.builder()  
        .orderNo(dto.getOrderNo())  
        .amount(dto.getAmount() + "元") // 动态拼接  
        .statusText(convertStatus(dto.getStatus()))  
        .build();  
    }  
}  
```

## 四、避坑指南

### 坑1：PO直接返回给前端

```
`// 致命错误：暴露数据库敏感字段！  `
public UserPO getUser(Long id) {  
  // 返回的PO包含password  
  return userDao.findById(id); 
}  
```

**解决方案** ：

* 使用DTO过滤字段
* 注解屏蔽：@JsonIgnore

### 坑2：DTO中嵌入业务逻辑

```
`public` class OrderDTO {
    // 错误！DTO不应有业务方法  
    public void validate() { 
       if (amount < 0) 
          throw new Exception();  
    }  
}  
```

**本质错误** ：混淆DTO与BO的职责

### 坑3：循环嵌套转换

```
`// OrderVO中嵌套List<ProductVO>  `
public class OrderVO {
   // 嵌套对象  
   private List<ProductVO> products; 
}  

// 转换时触发N+1查询  
orderVO.setProducts(order.getProducts()
       .stream()  
       .map(p -> convertToVO(p)) // 循环查询数据库  
       .collect(toList()));  
```

**优化方案** ：批量查询 + 并行转换

### 五、如何选择对象模型？

![](https://cubox.pro/c/filters:no_upscale()?imageUrl=https%3A%2F%2Fimg2024.cnblogs.com%2Fblog%2F551042%2F202509%2F551042-20250916140602020-2010199829.webp&valid=false)

## 总结

关于对象的4个核心原则：

1. **单一职责** ： PO只存数据，BO只管业务，VO只负责展示------**绝不越界！**
2. **安全隔离** ：
   * PO永不出DAO层（防数据库泄露）
   * VO永不出Controller（防前端逻辑污染服务）
3. **性能优先** ：
   * 大对象转换用**MapStruct** （编译期生成代码）
   * 嵌套集合用**批量查询** （杜绝N+1）
4. **适度设计** ：
   * 10张表以内的系统：可用POJO一撸到底
   * 百张表以上核心系统：**必须严格分层**
