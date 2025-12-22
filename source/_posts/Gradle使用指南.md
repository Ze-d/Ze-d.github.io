---
title: Gradle使用指南
date: 2025-11-23 16:07:19
tags: [Java,SpringBoot,Gradle]
---

## 一、Gradle 是什么？

**Gradle = 一种用“脚本”描述构建流程的自动化构建工具**，最常见于 **Java / Kotlin / Spring Boot / Android** 项目，用来做这些事情：

- 编译代码  
- 跑测试  
- 打 JAR / WAR / APK 包  
- 管理依赖（Maven 仓库里的各种库）  
- 一条命令完成“构建 → 测试 → 打包 → 发布”  

可以简单理解为：**Gradle 是新一代 Java 生态主流构建工具**，比 Ant 灵活、比 Maven 更可编程，而且构建性能更好。

### 1. Gradle 和 Maven 的简单区别

- **配置方式**：  
  - Maven 用 XML（`pom.xml`）  
  - Gradle 用 Groovy / Kotlin DSL（`build.gradle` / `build.gradle.kts`），是脚本，可写逻辑

- **构建模型**：  
  - Maven 基于“生命周期”（`compile → test → package → install → deploy`）  
  - Gradle 基于“任务（Task）”，可以自定义、组合、依赖

- **性能**：  
  - Gradle 支持增量构建、构建缓存、Gradle Daemon，通常比 Maven 更快

---

## 二、Gradle 的核心概念

### 1. Project（项目）

- 一个仓库就是一个 Gradle 项目（Project）。  
- 也可以是多模块结构：root project + 若干 subprojects。

### 2. Task（任务）

- 一切操作都是 task：`compileJava`、`test`、`bootJar`、`clean` 等。
- 你可以注册自己的任务，例如：

```groovy
task hello {
    doLast {
        println 'Hello, Gradle'
    }
}
```

执行：

```bash
./gradlew hello
```

### 3. Plugin（插件）

插件为项目提供特定领域的能力，例如：

- `java`：让项目具备 Java 编译、测试、打包能力  
- `application`：提供 `gradle run`，可直接运行 `main`  
- `org.springframework.boot`：Spring Boot 专用插件，支持 `bootRun` 和“胖 JAR”  
- `com.android.application`：Android 应用插件

在 `build.gradle` 中声明：

```groovy
plugins {
    id 'java'
}
```

### 4. Dependency（依赖）

和 Maven 一样，Gradle 也从 Maven 仓库下载依赖：

```groovy
repositories {
    mavenCentral()
}

dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web:3.3.0'
    testImplementation 'org.junit.jupiter:junit-jupiter:5.10.0'
}
```

---

## 三、如何开始使用 Gradle

### 1. 安装与使用方式

#### 方式 A：直接安装 Gradle（不太推荐新手）

- 从官网下载安装包，配置 `GRADLE_HOME` 和 `PATH`。  
- 常用于构建服务器或你经常新建项目的个人环境。

#### 方式 B：使用 Gradle Wrapper（推荐）

- 项目中包含：`gradlew` / `gradlew.bat` 和 `gradle/wrapper` 目录。  
- 无需本地预装 Gradle，直接运行：

```bash
# Linux / Mac
./gradlew build

# Windows
gradlew build
```

- Wrapper 会自动下载正确版本的 Gradle，**保证团队环境一致**。

> 通过 Spring Initializr 或 Android Studio 创建的项目，一般默认都会带 Gradle Wrapper。

---

### 2. 初始化一个 Gradle Java 项目（命令行方式）

假设已经有 Gradle（或使用现有项目的 Wrapper）：

```bash
gradle init      # 或 ./gradlew init
```

根据交互选择：

- 项目类型：`application` / `java-application` 等  
- 语言：Java / Kotlin  
- 测试框架：JUnit 等

Gradle 会生成类似结构：

```text
project/
 ├─ build.gradle        # 构建脚本（Groovy DSL）
 ├─ settings.gradle
 ├─ gradlew / gradlew.bat
 ├─ gradle/wrapper
 └─ src
     ├─ main/java
     └─ test/java
```

常用命令：

```bash
./gradlew build   # 编译 + 测试 + 打包
./gradlew test    # 只运行测试
./gradlew run     # 使用 application 插件时运行 main
```

---

## 四、常用 Gradle 命令

假设使用 Wrapper（推荐）：

```bash
# 查看所有可用任务
./gradlew tasks

# 基本构建流程
./gradlew clean      # 清理 build 目录
./gradlew build      # 编译 + 测试 + 打包（常用）
./gradlew assemble   # 只构建产物，不跑测试
./gradlew test       # 单独运行测试

# 自定义 Task
./gradlew hello

# Spring Boot 项目
./gradlew bootRun    # 启动应用
./gradlew bootJar    # 打可执行 JAR
```

---

## 五、最小可用 Java 项目示例：build.gradle

下面是一个简单可用的 Java 项目 `build.gradle` 示例（Groovy DSL）：

```groovy
plugins {
    id 'java'
    id 'application'
}

group = 'com.example'
version = '1.0.0'

java {
    sourceCompatibility = JavaVersion.VERSION_17
    targetCompatibility = JavaVersion.VERSION_17
}

application {
    // 你的 main 方法类全名
    mainClass = 'com.example.App'
}

repositories {
    mavenCentral()
}

dependencies {
    implementation 'com.google.guava:guava:33.3.0-jre'

    testImplementation 'org.junit.jupiter:junit-jupiter:5.10.0'
}

test {
    useJUnitPlatform()
}
```

你需要：

1. 在 `src/main/java/com/example/App.java` 中实现 `public static void main(String[] args)`；  
2. 在项目根目录执行：

```bash
./gradlew run
./gradlew build
```

即可完成运行和构建。

---

## 六、在 IDE 中使用 Gradle（IntelliJ IDEA / Android Studio）

一般步骤：

1. 创建项目时选择 **Gradle Project**（或在 Spring Initializr 中选择 Gradle）。  
2. IDE 会读取 `build.gradle` / `build.gradle.kts`，生成 Gradle 工程视图。  
3. 在 Gradle 工具窗口中可以看到所有 Task，双击即可执行。  
4. 修改依赖后刷新 Gradle 项目，IDE 会自动更新 classpath 和代码提示。

---

## 七、Ant / Maven / Gradle 对比

### 1. 一句话总结

- **Ant**：最早的 Java 构建工具，**完全脚本式，没统一规范，全靠自己写 target**。  
- **Maven**：约定优于配置的代表，**强规范、强约束，XML 声明式配置 + 生命周期**，适合传统 Java 企业项目。  
- **Gradle**：结合两者优点，**既有 Maven 的约定，也有 Ant 脚本的灵活性，用 Groovy/Kotlin DSL 可编程**，性能更好，是现在主流（Spring Boot / Android 默认）。

### 2. 核心维度对比表

| 维度              | Ant                                | Maven                                                | Gradle                                                    |
| ----------------- | ---------------------------------- | ---------------------------------------------------- | --------------------------------------------------------- |
| 出现时间 / 代际   | 最早（上一代）                     | 第二代                                               | 第三代                                                    |
| 配置方式          | XML 脚本（`build.xml`）            | XML 声明（`pom.xml`）                                | Groovy/Kotlin DSL（`build.gradle` / `build.gradle.kts`）  |
| 构建模型          | 无统一生命周期，自己写 `target`    | 标准生命周期：`clean → compile → test → package ...` | 基于 Task 的依赖图，也可实现类似生命周期                  |
| 约定优于配置      | 几乎没有，目录结构全靠自己约束     | 非常强（标准目录结构 / 约定好源码路径等）            | 有默认约定，也可灵活覆盖                                  |
| 依赖管理          | 原生不支持（常用 Apache Ivy）      | 内置依赖管理（GAV + 仓库 + 传递依赖）                | 内置依赖管理，语法更简洁、更灵活                          |
| 灵活性 / 可编程性 | 高，但 XML 写逻辑比较痛苦          | 中等，XML 不适合写复杂逻辑                           | 非常高：DSL + 真正编程能力（变量、条件、循环、函数等）    |
| 性能、增量构建    | 每次都按脚本执行，缺少系统级优化   | 有部分增量构建，但整体性能一般                       | 支持增量构建、构建缓存、Gradle Daemon，性能优势明显       |
| 多模块大项目支持  | 依赖手写脚本组织，管理复杂         | 成熟、稳定，传统大型企业项目广泛使用                 | 尤其适合多模块、微服务、Android 等复杂工程                |
| 生态 / 插件       | 生态基本停滞，主要在遗留项目中出现 | 插件生态巨大、文档丰富                               | 生态快速发展，尤其在 Spring、Android 等现代技术栈中占主流 |
| 典型使用场景      | 维护遗留项目、自定义自动化脚本     | 传统 Java Web / 企业级应用                           | Spring Boot / Android / 微服务 / CI/CD 等现代项目         |

---

### 3. 三者的构建思路差异

#### Ant：命令式脚本（类似写 Shell）

- 你要告诉 Ant：先干什么、再干什么，全是步骤：
  - 编译哪些目录  
  - 复制哪些文件  
  - 打成什么包  
- **没有统一项目模型**，你说什么它做什么。

优点：  

- 非常灵活，几乎可以做任何事。

缺点：  

- 大项目脚本难维护、可读性差、复用性差。

> 类比：像自己写一堆 `bash` 脚本，从 0 到 1 手撸构建流程。

#### Maven：声明式 + 生命周期 + 约定目录

- 通过约定好的目录结构和生命周期，让构建标准化：
  - 源代码：`src/main/java`  
  - 资源：`src/main/resources`  
  - 测试：`src/test/java`  
- 你只需要声明：
  - 这是个什么打包类型（`jar` / `war`）  
  - 有哪些依赖  
  - 用了哪些插件  

剩下工作交给 Maven 的生命周期：

- `validate → compile → test → package → verify → install → deploy`

优点：

- **标准化程度高，团队协作成本低**  
- XML 声明式配置，结构清晰

缺点：

- 灵活性偏弱，做“特殊构建逻辑”比较难受  
- XML 配置一旦复杂起来，维护成本较高  
- 性能相较 Gradle 一般

> 类比：像一条标准化流水线，你按规范放东西，它就按流程处理。

#### Gradle：DSL + Task 依赖图 + 可编程

- 使用 Groovy / Kotlin DSL 编写脚本：
  - 可以定义变量、函数、条件逻辑、循环等
  - 任务间是 DAG（有依赖关系的有向无环图）
- 结合了：
  - Maven 的项目模型、依赖管理、仓库和部分约定；
  - Ant 的“任务 + 灵活脚本”的思想；
  - 现代工程的性能优化（增量构建、缓存、守护进程）。

优点：

- 灵活、可编程，能应对复杂构建需求  
- 性能相对更好，适合频繁构建的项目（如大型工程、CI/CD）  
- 与 Android、Spring Boot 等现代技术深度集成

缺点：

- 上手门槛稍高，团队需要熟悉 DSL  
- 配置复杂时，如果没有统一规范，容易“太自由”而变乱

---

### 4. 实际选型建议

**何时会用 Ant？**

- 主要是两类场景：  
  1. 维护历史遗留项目  
  2. 某些非常定制化、很久以前写好的构建脚本

> 新项目基本不建议从 Ant 起步。

**何时选 Maven？**

- 公司里已有大量 Maven 模板和私服，大家都习惯 pom.xml  
- 项目偏传统 Java EE / Spring + Tomcat / 单体系统，构建流程比较朴素  
- 希望配置“更规则、更保守”，不强调复杂逻辑

**何时选 Gradle？（推荐）**

- 新项目，尤其是：
  - Spring Boot / Spring Cloud 微服务  
  - Android App  
  - 多模块工程、大型项目、CI/CD 频繁构建场景  
  - 需要灵活构建逻辑、自定义任务、自动化脚本

> 如果你现在在做 **Spring Boot + Docker + CI/CD** 或 **Android**，那 Gradle 通常是更优选择。

---

## 八、上手学习路线建议

1. **快速过一遍概念**  
   - 知道 Ant / Maven / Gradle 做什么，用在哪里。  

2. **重点实战 Gradle**  
   - 用 Spring Initializr 创建一个 Gradle + Spring Boot 项目；  
   - 理解 `build.gradle` 中 `plugins` / `repositories` / `dependencies` / `tasks`；  
   - 熟悉 `./gradlew bootRun / build / test / bootJar` 整体流程。  

3. **再学习 Maven（如果必要）**  
   - 看一个简单的 Maven 项目的 `pom.xml`，了解基本结构；  
   - 对比 Gradle 的 `build.gradle`，理解两者只是“构建描述语言不同”。  

如果你有具体项目（例如：一个 Spring Boot / Java 库 / Android App），可以基于这篇文档的内容：

- 先写出一个最小可用的 Gradle 脚本；  
- 再逐步加入依赖、多模块、Docker 打包、CI 脚本等。  

