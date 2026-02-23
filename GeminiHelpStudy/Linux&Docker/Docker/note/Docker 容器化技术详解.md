---
title: Docker 容器化技术详解
tags:
  - Docker
  - Basics
  - Commands
create_time: 2026-01-26
---

## 1. Docker 核心概念

### 1.1 镜像与容器
Docker 的运行机制类似“类”与“对象”的关系：
- **镜像 (Image)**：将应用所需的运行环境、配置文件、系统函数库等与应用一起打包得到的文件包。
- **容器 (Container)**：为每个镜像的应用进程创建的隔离运行环境。
- **镜像仓库 (Repository)**：存储和管理镜像的平台，官方仓库为 DockerHub。

![[Docker学习1.png]]

### 1.2 架构图解
Docker 采用 C/S 架构：
- **Client**：发送命令（如 `docker run`）。
- **Docker Daemon**：守护进程，接收命令并操作镜像与容器。
- **Registry**：镜像仓库，下载镜像。

---

## 2. Docker 环境安装 (Ubuntu)

> [!info] 提示
> 本节基于 Ubuntu 系统安装 Docker CE。

1. **卸载旧版本**（如果存在）：
```bash
sudo apt-get remove docker docker-engine docker.io containerd.io
```

2. **安装依赖**：
	- 安装必要的系统工具
```bash
sudo apt update
sudo apt-get install ca-certificates curl gnupg
```

3. **添加 GPG 密钥与仓库**：
```bash
# 添加阿里云镜像源（推荐）
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] [https://mirrors.aliyun.com/docker-ce/linux/ubuntu](https://mirrors.aliyun.com/docker-ce/linux/ubuntu) $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker-ce.list
```
```bash
# 添加 Docker 官方 GPG 秘钥
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
```

4. **安装 Docker**：
- 设置软件源
```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```
> [!info] 注意
> 看上面命令里的 `"$VERSION_CODENAME"`，它会自动读取你的系统版本名（也就是 `noble`），这样就不会装错成 `jammy` 了。
- 安装最新版 Docker
```bash
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

5. **启动与校验**：
```bash
sudo systemctl start docker
docker ps  # 输出表头说明安装成功
```

> [!tip] 验证版本
> ```bash
> docker --version
> ```

---

## 3. Docker 常用命令详解

> [!tip] 开始前建议先配置一下加速器
> 文件路径：`/etc/docker/daemon.json`
> ```json
> {
>         "registry-mirrors": [
> 			"https://docker.m.daocloud.io",
>				"https://huecker.io",
> 					"https://dockerhub.timeweb.cloud",
> 						"https://noohub.ru",
> 							"https://p2cr7377.mirror.aliyuncs.com"
>                                ]
> }
> ```

### 3.1 镜像命令 (Images)

* **查看镜像**：`docker images`
* **拉取镜像**：`docker pull [镜像名(:版本)]`
* **删除镜像**：`docker rmi [镜像名(:版本)]`
* **保存/加载**：
* `docker save -o [文件名.tar] [镜像名]`
* `docker load -i [文件名.tar]`

> [!tip] 命名规范
> 镜像名称结构为 `[Repository]:[Tag]`。如果没有指定 Tag，默认为 `latest`。

### 3.2 容器命令 (Containers)

* **运行容器**：`docker run` (详见下文)
* **查看容器**：
* `docker ps`：查看**运行中**的容器。
* `docker ps -a`：查看**所有**容器（含已停止）。

* **停止/启动**：`docker stop [容器名]` / `docker start [容器名]`
* **进入容器**：`docker exec -it [容器名] bash`
* **⭐查看日志**：`docker logs [容器名]` (`-f` 可持续跟踪)
* **删除容器**：`docker rm [容器名]` (加 `-f` 强制删除运行中的容器)

![[Docker 命令图.canvas]]
[[Docker 命令图.canvas | 点击跳转]]

### 3.3 网络问题


---

## 4. 实战：运行容器 (MySQL 为例)

### 4.1 命令结构拆解

运行 MySQL 容器的完整命令如下：

```bash
docker run -d \
  --name mysql \
  -p 3307:3306 \
  -e TZ=Asia/Shanghai \
  -e MYSQL_ROOT_PASSWORD=123 \
  mysql:8
```

| 参数 | 含义 |
| --- | --- |
| **`-d`** | 让容器在**后台运行** |
| **`--name`** | 给容器起一个唯一的名字 |
| **`-p`** | **端口映射** `宿主机端口:容器端口` (如 3307:3306) |
| **`-e`** | 设置**环境变量** (如时区、密码) |
| **`mysql:8`** | 指定镜像名称和版本 |

![[Docker学习3.png]]

### 4.2 常见问题

* **端口冲突**：如果宿主机 3306 被占用，修改 `-p` 左侧的端口即可。
* **版本问题**：建议明确指定 Tag（如 `:8` 或 `:5.7`），避免使用 `latest` 导致生产环境不一致。
