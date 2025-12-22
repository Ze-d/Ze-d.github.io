---
title: Docker常用指令
date: 2025-12-18 14:51:00
tags: docker
---

## 1. 镜像管理 (Image Management)

镜像（Image）是容器的只读模板。以下指令用于下载、构建和管理镜像。

### 搜索与拉取

- **`docker search <关键词>`**: 在 Docker Hub 搜索镜像。
- **`docker pull <镜像名>:<标签>`**: 拉取指定版本的镜像。
  - *示例:* `docker pull mysql:5.7` (如果不写标签，默认拉取 `:latest`)

### 查看与删除

- **`docker images`** (或 `docker image ls`): 列出本地所有镜像。
- **`docker rmi <镜像ID或名称>`**: 删除镜像。
  - `-f`: 强制删除（即使有容器在使用该镜像）。

### 构建镜像 (Build)

- **`docker build -t <镜像名>:<标签> .`**: 根据当前目录下的 Dockerfile 构建镜像。
  - `-t`: 指定镜像的名称和标签 (Tag)。
  - `-f`: 指定 Dockerfile 文件的路径（如果文件名不是默认的 `Dockerfile`）。
  - `.`: **关键点**，代表构建上下文（Context）是当前目录。

------

## 2. 容器生命周期管理 (Container Lifecycle)

容器（Container）是镜像的运行实例。

### 启动容器 (Run) - **最核心指令**

`docker run` 是最复杂的指令，因为它结合了创建和启动两个动作。

**基本语法:**

Bash

```
docker run [参数] <镜像名> [启动命令]
```

常用参数详解:

| 参数 | 含义 | 示例 |

| :--- | :--- | :--- |

| -d | 后台运行 (Detached mode)，容器在后台运行并返回容器 ID。 | docker run -d nginx |

| -p | 端口映射 (Port mapping)。格式为 宿主机端口:容器端口。 | docker run -p 8080:80 nginx (访问本地8080即访问容器80) |

| -v | 挂载数据卷 (Volume)。格式为 宿主机路径:容器路径。 | docker run -v /data/html:/usr/share/nginx/html nginx |

| --name | 指定名称。给容器起一个好记的名字，便于后续管理。 | docker run --name my-web nginx |

| -e | 环境变量 (Environment)。设置容器内的环境变量。 | docker run -e MYSQL_ROOT_PASSWORD=123 mysql |

| --restart| 重启策略。指定容器退出后的重启行为。 | docker run --restart=always nginx (开机自启或崩溃重启) |

| --network| 指定网络。将容器加入到特定的 Docker 网络中。 | docker run --network=my-net nginx |

### 停止、启动与重启

- **`docker start <容器ID/名>`**: 启动一个已停止的容器。
- **`docker stop <容器ID/名>`**: 优雅停止容器（发送 SIGTERM 信号）。
- **`docker restart <容器ID/名>`**: 重启容器。
- **`docker kill <容器ID/名>`**: 强制停止容器（发送 SIGKILL 信号）。

### 查看与删除

- **`docker ps`**: 查看当前**正在运行**的容器。
- **`docker ps -a`**: 查看**所有**容器（包括已停止的）。
- **`docker rm <容器ID/名>`**: 删除已停止的容器。
  - `-f`: 强制删除正在运行的容器。

------

## 3. 容器交互与调试 (Interaction & Debugging)

当容器在后台运行时，你需要这些工具来查看状态或排查问题。

### 查看日志

- **`docker logs <容器ID/名>`**: 查看容器的标准输出日志。
  - `-f`: **实时跟踪**日志输出 (Follow)，类似 `tail -f`。
  - `--tail N`: 仅显示最后 N 行日志。
  - *示例:* `docker logs -f --tail 100 my-web`

### 进入容器

- **`docker exec -it <容器ID/名> /bin/bash`**: 进入容器内部开启一个交互式终端。
  - `-i`: 保持 STDIN 打开。
  - `-t`: 分配一个伪终端 (TTY)。
  - *注意:* 如果容器是 Alpine Linux 版，通常没有 `bash`，需换成 `sh`。

###  查看详情

- **`docker inspect <容器ID/名>`**: 返回容器的详细配置信息（JSON 格式），包含 IP 地址、挂载路径、环境变量等。

### 文件复制

- **`docker cp <本地路径> <容器ID>:<容器路径>`**: 将宿主机文件拷贝到容器。
- **`docker cp <容器ID>:<容器路径> <本地路径>`**: 将容器文件拷贝到宿主机。

------

## 4. 系统清理与维护 (System Prune)

随着使用时间增加，Docker 会积累大量未使用的资源。

- **`docker system df`**: 查看 Docker 磁盘占用情况。
- **`docker system prune`**: 一键清理**未使用**的数据（慎用）。
  - 默认清理：已停止的容器、未被使用的网络、悬空镜像（dangling images）。
  - `-a`: 更激进，清理所有未被容器使用的镜像。
  - `--volumes`: 同时清理未被使用的数据卷（**极度危险，可能导致数据丢失**）。

------

## 5. 最佳实践小贴士

1. **尽量使用 `-d` 运行服务**: 除非是测试，否则通过 `-d` 让容器在后台守护运行。
2. **善用 `--name`**: 自动生成的随机名字（如 `focused_turing`）很难记，手动命名能极大提高运维效率。
3. **理解 `-p` 的方向**: 左边是宿主机（你访问的），右边是容器（程序监听的）。
4. **数据持久化**: 重要的数据库或文件服务，**务必**使用 `-v` 将数据挂载到宿主机，否则容器删除后数据将丢失。

