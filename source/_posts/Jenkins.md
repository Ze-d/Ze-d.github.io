---
title: Jenkins
date: 2025-12-17 17:16:03
tags: [Jenkins,CI/CD]
---

## 1. 概述

Jenkins 是目前全球最流行的开源自动化服务器（Automation Server）。它基于 Java 开发，拥有超过 1800 个插件的生态系统。它不仅是持续集成（CI）和持续交付（CD）的标准工具，更是现代 DevOps 流程中的“调度中枢”。

---

## 2. 核心原理 (Core Principles)

Jenkins 的运行机制并不神秘，理解其底层架构有助于更好地进行性能优化和故障排查。

### 2.1 Master-Slave (Controller-Agent) 架构

Jenkins 采用分布式架构来分担负载：

* **Controller (原 Master)**：
* **大脑**：负责管理配置、调度任务、监控 Agent 状态、提供 Web UI。
* **职责**：它通常不直接运行构建任务（为了保证系统稳定性），而是将任务分发给 Agent。


* **Agent (原 Slave)**：
* **手脚**：实际执行构建任务的节点。可以是物理机、虚拟机，也可以是动态生成的 Docker 容器/Kubernetes Pod。
* **跨平台**：Agent 可以是 Windows, Linux, macOS 等不同系统，从而支持在不同环境下编译代码。



### 2.2 Pipeline 运行机制

Jenkins 2.0 引入的核心概念是 **Pipeline**。

1. **触发 (Trigger)**：通过 Webhook（代码提交）、定时任务（Cron）或上游任务触发。
2. **调度 (Queue)**：任务进入构建队列，Controller 寻找空闲的 Agent。
3. **工作区 (Workspace)**：Agent 在本地创建目录，拉取代码。
4. **执行 (Execution)**：按照定义的步骤（Steps）执行 Shell 脚本、Maven 命令等。
5. **归档 (Artifacts)**：构建生成的产物（如 `.jar`, `.war`, `.apk`）回传给 Controller 或上传至制品库（Nexus/Artifactory）。

---

## 3. 解决的核心问题 (Solving Problems)

在引入 Jenkins 之前，传统的软件交付面临四大痛点，Jenkins 提供了针对性的解决方案：

| 痛点         | 传统模式                                                     | Jenkins 解决方案                                             |
| ------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| **集成地狱** | 开发人员各自开发，合并代码时才发现大量冲突和逻辑错误。       | **持续集成 (CI)**：每次 Commit 自动触发构建和测试，**5分钟内**反馈结果，Bug 不过夜。 |
| **环境差异** | “在我的电脑上是好的，服务器上怎么不行？”                     | **标准化构建**：统一编译环境和命令，消除“环境配置漂移”带来的不一致。 |
| **部署风险** | 手动 SSH 登录服务器，手动替换文件，容易误操作（删库跑路风险）。 | **自动化部署 (CD)**：一键发布，流程固化。支持蓝绿部署、金丝雀发布等高级策略。 |
| **流程黑盒** | 只有运维知道怎么发布，发布过程不透明，日志难追溯。           | **可视化与审计**：谁在什么时间发布了什么版本，日志、测试报告全记录，清晰可查。 |

---

## 4. 应用场景 (Applications)

虽然 Jenkins 最常用于 CI/CD，但它的能力远不止于此：

### 4.1 标准 CI/CD 流水线

* **开发阶段**：代码静态扫描 (SonarQube) -> 单元测试 -> 编译打包。
* **测试阶段**：自动部署到测试环境 -> 运行接口自动化测试。
* **生产阶段**：人工审批 -> 自动部署到生产环境 -> 冒烟测试。

### 4.2 自动化运维任务 (Ops Automation)

* **定时任务**：替代 Linux Crontab，执行数据库备份、日志清理，且拥有更好的日志界面和失败报警。
* **数据迁移**：编写 Pipeline 执行复杂的数据清洗或迁移脚本。

### 4.3 多环境与多平台构建

* 利用 Agent 机制，同一套代码可以同时在 Windows 上构建 `.exe`，在 macOS 上构建 iOS App，在 Linux 上构建 Docker 镜像。

---

## 5. 工程化实践 (Engineering Best Practices)

要在大规模团队中高效使用 Jenkins，必须遵循“工程化”原则，避免将其通过简单的“点击配置”变成维护噩梦。

### 5.1 Pipeline as Code (流水线即代码)

**严禁**在 Jenkins 网页 UI 上手动配置构建步骤。

* **做法**：在项目根目录创建 `Jenkinsfile`。
* **优势**：
* **版本控制**：构建流程随代码一起提交，可追溯修改历史。
* **灾难恢复**：Jenkins 挂了，换台机器拉取代码即可重建流程，无需重新配置。



**示例 (Declarative Pipeline):**

```groovy
pipeline {
    agent any
    stages {
        stage('Build') {
            steps { sh 'mvn clean package' }
        }
        stage('Test') {
            steps { sh 'mvn test' }
        }
    }
    post {
        failure {
            // 构建失败发送钉钉/邮件通知
            echo 'Build Failed'
        }
    }
}

```

### 5.2 弹性构建节点 (Docker/K8s Integration)

不要依赖静态的物理机 Agent，容易产生环境污染（上一个任务留下的临时文件影响下一个任务）。

* **最佳实践**：使用 Kubernetes 插件或 Docker 插件。
* **原理**：任务开始时，Jenkins 动态启动一个全新的 Docker 容器作为 Agent；任务结束后，容器自动销毁。保证每次构建都是纯净环境。

### 5.3 共享库 (Shared Libraries)

当你有 50 个微服务，每个服务的 `Jenkinsfile` 90% 都一样时，如何维护？

* **解决方案**：使用 **Shared Libraries**。
* 将通用的逻辑（如：发送通知、标准的 Docker 构建命令）提取出来，封装成 Groovy 函数库。各个项目的 `Jenkinsfile` 只需引用该库，即可复用代码，极大降低维护成本。

### 5.4 权限与安全 (Security)

* **RBAC (Role-Based Access Control)**：严格控制权限。开发人员只有“构建”权限，运维人员才有“配置”权限。
* **Credentials Management**：绝对不要在脚本里明文写密码。使用 Jenkins 的 "Credentials Binding" 插件，将 SSH Key、API Token 等敏感信息加密存储。

---

## 6. 总结

Jenkins 不仅仅是一个工具，它是**软件交付流水线的基石**。

* **初级阶段**：利用 Jenkins 自动化替代手动打包。
* **中级阶段**：实施 Pipeline as Code，接入代码质量扫描和自动化测试。
* **高级阶段**：基于 K8s 实现弹性构建，开发共享库实现流水线的标准化和模块化。
