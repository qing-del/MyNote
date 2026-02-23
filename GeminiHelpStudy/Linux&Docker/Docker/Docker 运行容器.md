---
title: Docker 运行容器
tags:
 - Docker
 - commands
 - MySQL
 - Learning
create_time: 2026-01-25
---

# 目录
- [[#安装MySQL]]
	- [[#🧩 指令结构说明]]
	- [[#🔍 逐个参数详解]]
- [[#🎯 总结：这条命令做了什么？]]
	- [[#✅ 如何验证是否成功？]]
	- [[#💡 小贴士]]

---


> [!tip] 这里使用 MySQL 作为示例
## 安装MySQL
- 先停掉虚拟机中的 MySQL ，确保你的虚拟机已经安装 Docker，且网络开通
```bash
docker run -d \
  --name mysql \
  -p 3307:3306 \
  -e TZ=Asia/Shanghai \
  -e MYSQL_ROOT_PASSWORD=123 \
  mysql:8
```


### 🧩 指令结构说明

```bash
docker run [选项] [镜像名]
```

- `docker run`：创建并启动一个新的容器。
- `-d`：后台运行（detached mode）。
- `--name mysql`：为容器指定名称。
- `-p 3307:3306`：端口映射。
- `-e TZ=...` 和 `-e MYSQL_ROOT_PASSWORD=...`：设置环境变量。
- `mysql:8`：使用的镜像名称和版本。

---

### 🔍 逐个参数详解

#### 1. `docker run`

> 创建并启动一个新容器。这是 Docker 最基本的命令之一。

---

#### 2. `-d`

> **后台运行模式**  
> 表示容器启动后在后台运行，不会占用当前终端。  
> 如果不加 `-d`，你会直接进入容器内部（通常用于调试）。

✅ 示例：

```bash
docker run -d ...   # 容器在后台运行
docker run ...      # 会阻塞当前终端
```

---

#### 3. `--name mysql`

> **为容器命名**  
> 给这个容器起一个名字叫 `mysql`，方便后续管理。

📌 作用：

- 可以用 `docker stop mysql` 停止它
- 用 `docker logs mysql` 查看日志
- 避免使用随机生成的长 ID

> ⚠️ 注意：不能使用空格或特殊字符，且名字必须唯一。

---

#### 4. `-p 3307:3306`

> **端口映射（Port Mapping）**

格式：`宿主机端口:容器内端口`

- 宿主机（你的 Linux 服务器）监听 **3307 端口**
- 容器内的 MySQL 服务运行在 **3306 端口**
- 所有访问 `3307` 的请求都会转发到容器的 `3306`

💡 为什么这样设计？

- 防止与本地已有的 MySQL 冲突（比如你本机也有 MySQL）
- 多个服务可以用不同端口暴露（如 3307、3308）

---

#### 5. `-e TZ=Asia/Shanghai`

> **设置环境变量：时区**

- `TZ` 是 Time Zone 的缩写
- 设置容器内部的时区为 **上海/中国标准时间（CST）**
- 这样数据库记录的时间就不会显示成 UTC 时间（如 `2025-04-05T08:00:00Z`）

✅ 效果：

```sql
SELECT NOW();  -- 输出类似：2025-04-05 16:00:00 （北京时间）
```

> 🌍 其他常见时区：
> 
> - `America/New_York`
> - `Europe/London`
> - `Asia/Tokyo`

---

#### 6. `-e MYSQL_ROOT_PASSWORD=123`

> **设置 MySQL 的 root 用户密码**

- `MYSQL_ROOT_PASSWORD` 是 MySQL 官方镜像支持的环境变量
- 容器启动时会自动创建 root 用户，并设置该密码
- 密码在这里是明文写的（**生产环境建议使用 `.env` 文件或 secrets**）

⚠️ 警告：

- `123` 是非常弱的密码，仅用于学习演示
- 实际项目中请使用强密码，例如：`MySecurePass@2025`

---

#### 7. `mysql:8`

> **指定要使用的镜像**

- `mysql`：Docker Hub 上官方的 MySQL 镜像
- `:8`：表示使用 **MySQL 8.0 版本**

📌 可选版本：

```bash
mysql:5.7    # MySQL 5.7
mysql:latest # 最新版本（不推荐用于生产）
mysql:8.0.33 # 特定小版本
```

> ✅ 推荐使用明确版本号，避免因更新导致兼容问题。

---

## 🎯 总结：这条命令做了什么？

|功能|说明|
|---|---|
|🐳 启动容器|使用 `mysql:8` 镜像创建一个 MySQL 容器|
|🖥️ 后台运行|用 `-d` 让容器在后台运行|
|🏷️ 命名容器|名字叫 `mysql`，便于管理|
|🔗 映射端口|将宿主机 3307 映射到容器 3306|
|🕒 设置时区|使用上海时区，避免时间错乱|
|🔐 设置密码|root 用户密码设为 `123`|
|📦 自动初始化|官方镜像会自动完成数据库初始化|

---

### ✅ 如何验证是否成功？

```bash
# 查看容器是否运行
docker ps

# 查看日志（确认 MySQL 是否启动成功）
docker logs mysql

# 测试连接（需安装 mysql-client）
mysql -h 127.0.0.1 -P 3307 -u root -p
```

> 输入密码：`123`

---

### 💡 小贴士

- 如果你想改密码，只需修改 `-e MYSQL_ROOT_PASSWORD=xxx` 即可
- 若想持久化数据，请添加 `-v /data/mysql:/var/lib/mysql` 挂载数据卷
- 生产环境建议使用配置文件 + 加密方式管理密码

---