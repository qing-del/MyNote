---
title: Docker 高级：Dockerfile 与镜像构建
tags:
  - Docker
  - Dockerfile
  - Image
create_time: 2026-01-26
---

## 1. 镜像结构与 Dockerfile

### 1.1 镜像的分层结构
镜像不是一个单一文件，而是由多层（Layer）组成的：
- **BaseImage**：基础镜像（如 CentOS, Ubuntu），提供系统函数库。
- **Layer**：中间层，每一步操作（安装依赖、拷贝文件）都会形成新的一层。
- **Entrypoint**：入口，程序启动的脚本。

![[Docker学习12.png]]

### 1.2 什么是 Dockerfile？
Dockerfile 是一个文本文件，包含一条条指令（Instruction），用于描述如何构建一个镜像。

### 1.3 常用指令详解
| 指令 | 说明 | 示例 |
| :--- | :--- | :--- |
| **`FROM`** | 指定基础镜像 | `FROM centos:7` |
| **`ENV`** | 设置环境变量 | `ENV JAVA_HOME=/usr/local/jdk` |
| **`COPY`** | 拷贝本地文件到镜像 | `COPY ./app.jar /tmp/app.jar` |
| **`RUN`** | 执行 Shell 命令 (构建时运行) | `RUN yum install -y vim` |
| **`EXPOSE`** | 声明监听端口 (仅文档作用) | `EXPOSE 8080` |
| **`ENTRYPOINT`** | 容器启动时执行的命令 | `ENTRYPOINT ["java", "-jar", "/app.jar"]` |

---

## 2. 实战：构建 Java 镜像

### 2.1 编写 Dockerfile
```dockerfile
# 1. 指定基础镜像
FROM java:8-alpine

# 2. 拷贝 Jar 包
COPY ./docker-demo.jar /app.jar

# 3. 暴露端口
EXPOSE 8080

# 4. 入口命令
ENTRYPOINT ["java", "-jar", "/app.jar"]

```

> [!tip] 对于 JDK 包的选择
> 一般都是选择`Linux x64 Compressed Archive`的版本
> **解释：** 它是一个 `.tar.gz` 压缩包，不依赖特定系统的包管理器（如 apt 或 yum）。你可以直接在 Dockerfile 中将其解压到任意目录（如 `/opt/java`），非常干净。

### 2.2 构建命令

在 Dockerfile 所在目录执行：

```bash
# -t 给镜像起名, "." 代表当前目录
docker build -t my-java-app:1.0 .
```