---
title: Jar实现原理
date: 2025-12-18 11:58:13
tags: [Java,SpringBoot]
---

### 1. 什么是 JAR 包？

- **定义**：JAR 全称是 **J**ava **AR**chive。它是一种独立于平台的文件格式，用于将许多 Java 类文件（`.class`）、相关的元数据和资源（文本、图片等）聚合到一个文件中。
- **本质**：它本质上就是一个 **ZIP 压缩文件**。如果你把 `.jar` 后缀改成 `.zip`，你可以直接用解压软件打开它看到里面的内容。
- **目的**：
  - **安全性**：可以对内容进行数字签名。
  - **减少体积**：压缩文件，减少存储空间和网络传输时间。
  - **便携性**：将整个应用程序或库打包成单一文件，方便部署。

### 2. JAR 包的内部结构

一个标准的 JAR 包通常包含以下结构：

```text
MyApplication.jar
├── META-INF/
│   └── MANIFEST.MF   <-- 核心：清单文件
├── com/
│   └── example/
│       ├── Main.class
│       └── Utils.class
├── application.properties
└── images/
    └── logo.png
```

#### 关键文件：MANIFEST.MF

位于 `META-INF/` 目录下的 **`MANIFEST.MF` (清单文件)** 是 JAR 包的 "说明书" 或 "身份证"。它包含了关于这个 JAR 包的重要信息，例如：

- **Manifest-Version**: 清单文件的版本。
- **Main-Class**: (如果是可执行 JAR) 指定程序的入口类，即包含 `public static void main(String[] args)` 方法的类。
- **Class-Path**: 指定该 JAR 包依赖的其他 JAR 包路径。

------

### 3. JAR 包的分类

根据用途，JAR 包通常分为两类：

#### A. 普通 JAR 包 (Library JAR)

- **用途**：作为第三方库供其他程序调用（例如 `commons-lang3.jar`, `mysql-connector.jar`）。
- **特点**：通常没有 `Main-Class`，不能直接运行，只能被添加到项目的类路径（Classpath）中使用。

#### B. 可执行 JAR 包 (Executable JAR)

- **用途**：作为一个独立的应用程序运行。

- **特点**：在 `MANIFEST.MF` 中定义了 `Main-Class`。

- **运行方式**：

  ```bash
  java -jar my-app.jar
  ```

------

### 4. 现代开发中的 "Fat JAR" vs "Thin JAR"

在 Spring Boot 等现代框架流行后，这两个概念变得非常重要：

Spring Boot依赖spring-boot-maven-plugin插件将普通的jar包转化为Fat Jar

| **特性**     | **Thin JAR (瘦包)**                      | **Fat JAR / Uber JAR (胖包)**                                |
| ------------ | ---------------------------------------- | ------------------------------------------------------------ |
| **内容**     | 仅包含你自己编写的代码和资源。           | 包含你写的代码 **加上** 所有依赖的第三方库（这是 Spring Boot 的默认打包方式）。 |
| **依赖管理** | 运行时需要手动配置外部依赖路径 (`-cp`)。 | 依赖都在包里，做到了 "开箱即用"。                            |
| **优点**     | 体积小，适合分层构建镜像。               | 部署极其简单，一个文件就是整个应用。                         |
| **缺点**     | 部署时容易出现环境依赖缺失问题。         | 体积大（可能几十 MB 甚至上百 MB）。                          |

------

相较于thin jar,fat jar不仅包含自己的class文件，还包含：

1. 应用代码
2. 所有依赖的第三方库
3. 启动器

插件会在 `META-INF/MANIFEST.MF` 中添加关键信息：

```text
Manifest-Version: 1.0
Spring-Boot-Classes: BOOT-INF/classes/
Spring-Boot-Lib: BOOT-INF/lib/
Start-Class: com.github.paicoding.forum.web.QuickForumApplication
Spring-Boot-Version: 2.7.1
Main-Class: org.springframework.boot.loader.JarLauncher
```

**关键字段说明**:

- `Start-Class`: 你的实际主类（`QuickForumApplication`）
- `Main-Class`: Spring Boot 的启动器（`JarLauncher`）
- `Spring-Boot-Classes`: 应用代码位置
- `Spring-Boot-Lib`: 依赖库位置

### 5. 常用操作命令

虽然现在我们多用 Maven 或 Gradle，但了解 JDK 自带的 `jar` 命令有时很有用：

- **创建 JAR 包**：

  Bash

  ```
  jar -cvf app.jar .  # c=create, v=verbose, f=file
  ```

- **查看 JAR 包内容 (不解压)**：

  Bash

  ```
  jar -tf app.jar
  ```

- **解压 JAR 包**：

  Bash

  ```
  jar -xvf app.jar
  ```

------

### 6. 常见问题与排错

在使用 JAR 包时，你可能会遇到以下经典报错：

1. **"no main manifest attribute, in xxx.jar"**
   - **原因**：你试图运行一个 JAR，但它的 `MANIFEST.MF` 文件里没有 `Main-Class` 属性。
   - **解决**：检查构建配置（Maven/Gradle），确保配置了主类；或者确认你下载的是可执行包而不是源码包。
2. **`java.lang.ClassNotFoundException` / `NoClassDefFoundError`**
   - **原因**：运行时找不到需要的类。通常是因为缺少依赖的 JAR 包。
   - **解决**：如果是普通 JAR，需要检查 `-classpath` 参数；如果是 Fat JAR，检查打包插件是否正确将依赖打入。
3. **JAR Hell (JAR 包地狱)**
   - **原因**：类路径下存在同一个库的多个不同版本（例如同时存在 `log4j-1.2.jar` 和 `log4j-1.3.jar`），导致加载了错误的类。
   - **解决**：使用 Maven/Gradle 的依赖树命令 (`mvn dependency:tree`) 排查并排除冲突版本。

------

### 7. 与 Maven/Gradle 的集成

在现代 Java 开发中，我们很少手动打包，而是交给构建工具：

- **Maven**:
  - 运行 `mvn package` 会在 `target/` 目录下生成 JAR。
  - 常用插件：`maven-shade-plugin` 或 `spring-boot-maven-plugin` (用于打 Fat JAR)。
- **Gradle**:
  - 运行 `gradle build` 会在 `build/libs/` 目录下生成 JAR。

