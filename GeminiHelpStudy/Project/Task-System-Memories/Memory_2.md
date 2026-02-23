# 🚩 LifeGame Project - Phase 2 Final Summary

> [!abstract] 阶段概述
> **核心目标**: 完成多用户系统、商业化闭环（商店）、全栈交互优化，并实现容器化生产环境部署。
> **当前版本**: v0.3.0 (Dockerized)
> **状态**: ✅ 已完结 (All Green)
> **部署环境**: Ubuntu 24.04 LTS + Docker Compose

## 1. 核心技术栈演进 (Tech Stack Evolution)

| 领域 | Phase 1 (基础) | Phase 2 (当前) | 关键升级点 |
| :--- | :--- | :--- | :--- |
| **后端** | SpringBoot单体 | **Spring Security + JWT** | 实现了无状态认证与数据隔离 |
| **数据** | CRUD | **Transactional + MyBatis高级** | 保证了商店交易的原子性与数据一致性 |
| **前端** | 简单HTML | **Vue3 + Element Plus** | 组件化开发，Axios 拦截器封装 |
| **存储** | 本地文件 | **阿里云 OSS** | 自定义 Starter 封装，实现云端图床 |
| **运维** | 本地运行 | **Docker Compose** | 镜像构建、容器编排、网络隔离 |

## 2. 关键业务模块成果

### 🛡️ 用户与安全中心 (User & Security)
- **JWT 鉴权体系**:
    - 实现了 `JwtAuthenticationFilter`，对请求头 `Bearer Token` 进行拦截解析。
    - 解决了 Token 过期后的前端自动登出与路由守卫问题。
- **个人中心**:
    - 支持头像上传（对接阿里云 OSS）、修改昵称、修改密码。
    - 解决了 MySQL `head_photo` 字段长度不足导致的 Truncation 报错。

### 🛒 商店交易系统 (Shop System)
- **原子化交易**: 使用 `@Transactional` 确保“扣钱-发货-扣库存”三大步骤同生共死。
- **动态库存**: 实现了 `-1` 无限库存与普通库存的逻辑区分。
- **前端交互**: 完成了从“表格展示”到“卡片式货架”的 UI 升级，解决了 Axios POST 参数 `Content-Type` 传递错误的问题。

### 📊 任务与属性系统 (Refined)
- **数据隔离**: 重构 SQL，确保用户只能查询到自己的任务 (`WHERE user_id = ?`)。
- **分页查询**: 集成 `PageHelper` 优化了大量任务数据的加载体验。

## 3. DevOps 运维与部署 (New!)

> *注：此部分补全了日志中未记录的 Linux/Docker 实践*

### 🐧 Linux 环境 (Ubuntu 24.04)
- **环境准备**: 熟练掌握 SSH 连接 (FinalShell)，文件权限管理 (`chmod`, `chown`)。
- **故障排查**:
    - 解决了 `docker.io` 国内拉取超时 (`i/o timeout`) 问题，配置了镜像加速器。
    - 解决了容器挂载本地目录时的权限问题 (MySQL 10061/Permission Denied)，掌握了 `chmod 777` 修复挂载点权限。

### 🐳 Docker 容器化
- **Dockerfile 构建**:
    - 编写了后端 `Dockerfile`，基于 `openjdk:17` 构建业务镜像。
    - 解决了容器时区问题 (`-e TZ=Asia/Shanghai`)。
- **数据持久化**:
    - 理解了 `Bind Mount` (覆盖) 与 `Volume` (继承) 的区别。
    - 解决了 Nginx 挂载空目录导致 403 的问题。
- **Docker Compose 编排**:
    - 编写 `docker-compose.yml` 一键拉起 MySQL + Backend + Nginx。
    - 实现了容器内部网络互联（Backend 连接 DB 使用服务名而非 localhost）。

## 4. 待解决/遗留问题 (Future Scope)
- [ ] **代码规范**: 部分 Controller 逻辑可以进一步下沉到 Service。
- [ ] **性能优化**: 目前大量图片直接回源 OSS，未来可加入 CDN。
- [ ] **高可用**: 只有一个后端实例，无 Redis 缓存，高并发下数据库压力大。

---
**存档时间**: 2026-01-28
**开发者**: Jacolp