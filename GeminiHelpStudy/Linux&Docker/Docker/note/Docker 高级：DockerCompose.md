---
title: Docker 高级：DockerCompose
tags:
  - Docker
  - Compose
  - DevOps
create_time: 2026-01-26
---

## 1. 什么是 Docker Compose
Docker Compose 是一个用于定义和运行多容器 Docker 应用程序的工具。它使用 YAML 文件来配置应用程序的服务，通过一个命令即可创建并启动所有服务。

![[Docker学习19.png]]

## 2. YAML 文件结构
文件通常命名为 `docker-compose.yml`。

```yaml
version: "3.8"

services:
  mysql:
    image: mysql:5.7
    container_name: mysql
    ports:
      - "3306:3306"
    environment:
      MYSQL_ROOT_PASSWORD: 123
    volumes:
      - ./mysql_data:/var/lib/mysql
    networks:
      - itheima

  web:
    build: .
    ports:
      - "8080:8080"
    depends_on:
      - mysql
    networks:
      - itheima

networks:
  itheima:
    name: itheima

```

## 3. 常用命令

| 命令 | 说明 |
| --- | --- |
| `docker compose up -d` | 创建并后台启动所有服务 |
| `docker compose down` | 停止并移除容器、网络 |
| `docker compose logs -f` | 查看日志 |
| `docker compose ps` | 查看运行状态 |
| `docker compose restart` | 重启服务 |

