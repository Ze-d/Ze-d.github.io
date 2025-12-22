---
title: docker使用与开发
date: 2025-11-23 16:07:06
tags: Docker
---

## 1. Docker 是什么？

用一句话概括：

> **Docker = 用镜像和容器来管理“运行环境”的工具，让程序在任何机器上都以同样的方式运行。**

传统开发/部署方式的问题：

- 开发机装各种依赖：JDK / Python / MySQL / Redis / Kafka…
- 服务器上还要重复装一遍，而且版本必须对得上
- “我这跑得好好的，服务器怎么就挂了？”——典型的环境不一致问题

Docker 的思路：

- 把「程序 + 依赖 + 配置 + 启动命令」打包到一个 **镜像 (Image)** 里  
- 运行镜像的时候，启动一个 **容器 (Container)**  
- 容器之间、容器与宿主机之间是相对隔离的  
- 任何装了 Docker 的机器上，只要有这个镜像，就能得到一致的运行环境

---

## 2. Docker 核心概念

### 2.1 Image（镜像）

- 类似 “系统快照 + 应用安装包”
- 只读模板，可以从仓库拉取，也可以自己构建
- 示例：
  - `openjdk:17-jdk`
  - `python:3.11-slim`
  - `mysql:8.0`
  - `nginx:latest`

### 2.2 Container（容器）

- 镜像运行起来后的“实例”
- 本质上是一个隔离的进程空间，有自己的文件系统、网络等
- 可以启动多个容器实例，互不影响

### 2.3 Registry（仓库）

- 存放镜像的地方
  - 公共仓库：Docker Hub
  - 私有仓库：公司自建镜像仓库

### 2.4 Volume（数据卷）

- 用来把容器内部目录挂载到宿主机
- 作用：
  - 持久化数据（容器删掉数据不丢）
  - 在开发时把代码目录挂进去，实现热更新

### 2.5 Network（网络）

- 容器之间通过虚拟网络通信
- 在同一个自定义网络中，容器可以通过 **容器名** 互相访问
  - 例如在 `app` 容器中访问 `mysql:3306`

---

## 3. 开发中如何使用 Docker？

常见几种使用姿势：

1. **只用 Docker 跑依赖服务**（MySQL/Redis/Kafka/...），代码在本机跑
2. **应用也容器化**，本机和服务器用同一套镜像
3. **使用 docker-compose 管理多容器系统**（推荐）

下面逐个展开。

---

## 4. 必会基础命令

以 `nginx` 为例：

```bash
# 拉取镜像
docker pull nginx:latest

# 启动容器：-d 后台运行，--name 命名，-p 做端口映射
docker run -d --name my-nginx -p 8080:80 nginx:latest

# 查看正在运行的容器
docker ps

# 查看所有容器（包含已停止）
docker ps -a

# 进入容器内部
docker exec -it my-nginx /bin/bash

# 查看容器日志（持续输出）
docker logs -f my-nginx

# 停止容器
docker stop my-nginx

# 删除容器
docker rm my-nginx
```

端口映射 `-p 宿主端口:容器端口`：

- 容器内服务监听的是容器端口（例如 80）
- 通过 `-p 8080:80`，宿主机访问 `http://localhost:8080` 即等价访问容器内的 `80` 端口

------

## 5. 用 Docker 跑依赖，代码在本机跑（推荐起步方式）

这是日常开发里最常见、体验最好的一种方式：

- 宿主机：
  - 代码、IDE（IntelliJ / VSCode / 等）
- Docker：
  - MySQL / Redis / Kafka / RabbitMQ 等依赖服务

### 5.1 示例：使用 Docker 启动 MySQL

```bash
docker run -d \
  --name my-mysql \
  -e MYSQL_ROOT_PASSWORD=123456 \
  -e MYSQL_DATABASE=testdb \
  -p 3306:3306 \
  -v /your/path/mysql-data:/var/lib/mysql \
  mysql:8.0
```

在你的应用配置中（例如 Spring Boot）：

- 数据库 URL：`jdbc:mysql://localhost:3306/testdb`
- 用户名：`root`
- 密码：`123456`

**优点：**

- 不用在本机安装、维护一堆数据库服务
- 想重置环境：停容器、删数据卷即可
- 换机器只要装 Docker + 这条命令就能复现依赖环境

------

## 6. 为应用构建镜像：Dockerfile

当你希望应用也容器化时，就需要写 `Dockerfile`。

### 6.1 Spring Boot 单 jar 示例

```dockerfile
# 1. 基础镜像（JDK 环境）
FROM eclipse-temurin:17-jdk-alpine

# 2. 设置工作目录
WORKDIR /app

# 3. 拷贝 jar 包
COPY target/myapp-0.0.1-SNAPSHOT.jar app.jar

# 4. 暴露端口（文档说明用）
EXPOSE 8080

# 5. 容器启动命令
ENTRYPOINT ["java", "-jar", "app.jar"]
```

构建 & 运行：

```bash
# 在项目根目录（确保 target 下已有 jar）
docker build -t myapp:latest .

docker run -d --name myapp-container -p 8080:8080 myapp:latest
```

此时：

- 应用在容器中运行
- 本机访问 `http://localhost:8080` 即可访问服务

------

## 7. 开发阶段的热更新与代码验证方式

核心问题：**我怎么一边改代码，一边在 Docker 环境中验证？**

有两种主流模式：

### 7.1 模式 A：代码在宿主机，Docker 只跑依赖（强烈推荐）

- 适合 Java / Node / Python 等常规开发
- 你在本机像平时一样：
  - `mvn spring-boot:run`
  - `npm run dev`
- Docker 只负责 MySQL / Redis / Kafka 等

**优点：**

- IDE 断点、热部署体验最佳
- Docker 负责“环境”，你专心写代码

### 7.2 模式 B：代码目录挂载进容器，容器跑 dev 命令

- 适合你想用容器统一开发环境的场景
- 镜像提供：JDK / Node / Python 等运行环境
- 代码通过 Volume 挂载

#### Node 项目开发示例：

```bash
docker run -it --rm \
  -v /your/path/my-node-app:/app \
  -w /app \
  -p 3000:3000 \
  node:20 \
  bash -c "npm install && npm run dev"
```

- `-v /your/path/my-node-app:/app`：把本机代码目录挂载到容器 `/app`
- `npm run dev`：跑前端开发服务器（带热更新）
- 在本机编辑器里改代码，容器里会自动重新编译、刷新

#### Spring Boot 开发类似：

```bash
docker run -it --rm \
  -v /your/path/demo:/app \
  -w /app \
  -p 8080:8080 \
  maven:3.9-eclipse-temurin-17 \
  mvn spring-boot:run
```

------

## 8. 使用 docker-compose 管理多服务开发环境

`docker-compose` 让你可以用一个 `yaml` 文件描述多个服务，然后一条命令拉起整个环境。

### 8.1 示例：后端 + MySQL 的开发环境

`docker-compose.yml`：

```yaml
version: "3.9"
services:
  db:
    image: mysql:8.0
    container_name: demo-mysql
    environment:
      MYSQL_ROOT_PASSWORD: 123456
      MYSQL_DATABASE: demo
    ports:
      - "3306:3306"
    volumes:
      - ./mysql-data:/var/lib/mysql
    networks:
      - demo-net

  app:
    image: eclipse-temurin:17-jdk
    container_name: demo-app
    working_dir: /app
    volumes:
      - ./app:/app
    command: ["bash", "-c", "chmod +x mvnw && ./mvnw spring-boot:run"]
    ports:
      - "8080:8080"
    depends_on:
      - db
    networks:
      - demo-net

networks:
  demo-net:
    driver: bridge
```

使用命令：

```bash
# 前台启动
docker-compose up

# 后台启动
docker-compose up -d

# 查看日志
docker-compose logs -f           # 所有服务
docker-compose logs -f app       # 指定 app 服务

# 停止并清理容器（保留数据卷）
docker-compose down
```

特点：

- `app` 和 `db` 在同一个网络 `demo-net` 中
- 在 `app` 容器内访问数据库地址可以写 `db:3306`
- `./app` 目录挂载进容器，修改本机代码即可触发重新编译/热更新（视项目配置）

------

## 9. 日常最常用的 Docker 操作

### 9.1 日志查看

```bash
# 单个容器
docker logs -f myapp-container

# compose 中的服务
docker-compose logs -f          # 所有服务
docker-compose logs -f app      # 指定服务
```

### 9.2 进入容器调试

```bash
# 进入 Linux shell
docker exec -it myapp-container /bin/bash

# 直接连 MySQL
docker exec -it demo-mysql mysql -uroot -p123456
```

### 9.3 代码更新 & 镜像更新

如果采用“打包成 jar → 构建镜像”的方式：

```bash
# 1. 本机打包
mvn package -DskipTests

# 2. 构建镜像
docker build -t myapp:latest .

# 3. 重启容器
docker stop myapp-container
docker rm myapp-container
docker run -d --name myapp-container -p 8080:8080 myapp:latest
```

如果采用“代码挂载 + dev 命令”方式：

- 直接改代码即可，容器里 dev 进程会自动重载（视框架而定）

### 9.4 清理环境

```bash
# 删除所有已停止的容器
docker container prune

# 删除没有容器引用的镜像
docker image prune

# 删除所有未使用的容器、网络、镜像等（危险，慎用）
docker system prune -a
```

------

## 10. 一个“端到端”开发流程示例

以一个 **Spring Boot + Vue + MySQL** 项目为例：

### 阶段 1：只 Docker 化依赖

1. 用 Docker 跑 `mysql`、`redis` 等
2. 后端在本机用 IDE 跑
3. 前端在本机 `npm run dev`
4. 这一步已经极大减少了环境差异问题

### 阶段 2：用 docker-compose 做“开发版环境”

- 写一个 `docker-compose.dev.yml`：

  - `mysql` / `redis`：镜像 + 数据卷
  - `backend` / `frontend`：镜像只提供环境（JDK/Node），代码目录挂载，命令是 `mvn spring-boot:run` / `npm run dev`

- 开发时：

  ```bash
  docker-compose -f docker-compose.dev.yml up
  ```

- 一条命令拉起整套开发环境，IDE 只负责编辑代码和调试

### 阶段 3：写“生产版” Dockerfile + Compose

- 后端：
  - 使用打包好的 jar 构建瘦身镜像（基于 `jre`/`distroless`）
- 前端：
  - 打包静态文件
  - 用 `nginx` 镜像部署
- 数据库：
  - 用云数据库 / 独立 DB 服务 / 单独容器，根据实际架构决定

------

## 11. 从零开始学习 Docker 的推荐练习路径

1. **练习基础命令**
   - `docker pull/run/ps/logs/exec/stop/rm`
   - 用 `nginx`、`mysql`、`redis` 做小实验
2. **把一个现有 demo 做成镜像**
   - 给一个简单的 Spring Boot / Node 应用写 Dockerfile
   - 构建镜像、运行、访问接口
3. **用 docker-compose 拉起多服务**
   - App + MySQL / Redis
   - 学会用服务名互相访问（例如 `db:3306`）
4. **挂载代码目录实现热更新开发**
   - Node 项目：`npm run dev` + volume 挂载
   - Java 项目：`mvn spring-boot:run` + volume 挂载
5. **最终写一个“生产版”镜像**
   - 减小镜像体积
   - 把编译和运行阶段拆成多阶段构建（multi-stage build）

------

如果你之后告诉我你当前项目的技术栈（例如：Spring Boot + MySQL + Redis，或者还带前端），我可以在这个基础上给你定制一套 **适合该项目的 docker-compose + Dockerfile 模板**，直接可用。

```
::contentReference[oaicite:0]{index=0}
```
