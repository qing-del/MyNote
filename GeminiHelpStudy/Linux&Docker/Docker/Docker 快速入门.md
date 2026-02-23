---
title: Docker 快速入门
tags:
 - Docker
 - commands
 - Ubuntu
 - Learning
create_time: 2026-011-25
---

# 目录
- [[#Docker 安装]]
	- [[#卸载旧版]]
	- [[#安装步骤]]
	- [[#✅ 启动和校验]]

---

# Docker 安装
## 卸载旧版
- 如果系统中已存在旧的`Docker`，则需要先卸载
```bash
sudo apt-get remove docker \
                   docker-engine \
                   docker.io \
                   containerd.io \
                   docker-compose \
                   docker-compose-v2
```

> [!info] 💡 说明：
> - `docker`：旧版本的 Docker 包。
> - `docker-engine`：Docker 引擎（较老命名）。
> - `docker.io`：Ubuntu 上常用的 Docker 安装包名。
> - `containerd.io`：容器运行时组件（可能被其他工具依赖）。
> - `docker-compose` / `docker-compose-v2`：Docker Compose 工具（如果之前安装过）。

### 🧹 可选：清理残留配置文件和缓存
```bash
sudo apt-get autoremove --purge docker \
                       docker-engine \
                       docker.io \
                       containerd.io \
                       docker-compose \
                       docker-compose-v2
```
- 然后清理本地下载的包缓存：
```bash
sudo apt-get clean
```

### 🔍 检查是否已完全卸载
- 你可以运行以下命令确认是否还有 Docker 相关进程或包：
```bash
dpkg -l | grep docker
```
- 如果没有输出，说明已清理干净

---

## 安装步骤
### 🛠️ 步骤一：安装必要的依赖工具
- 在 Ubuntu 上，我们使用 `apt` 包管理器，并通过添加官方或阿里云的 APT 仓库来安装 Docker。
```
sudo apt update
sudo apt install -y apt-transport-https ca-certificates curl gnupg lsb-release
```
> [!info] 🔍
> 这些是 Ubuntu 中用于添加 HTTPS 仓库、验证签名和管理密钥所需的工具。

### 🌐 步骤二：添加 Docker 官方 GPG 密钥
```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
```
> [!warning] 💡 注意：
> 这个命令将 Docker 的 GPG 公钥导入系统信任库。


### 📁 步骤三：添加 Docker APT 仓库
#### ✅ 方式一：使用 **阿里云镜像源**（推荐国内用户）
```bash
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://mirrors.aliyun.com/docker-ce/linux/ubuntu $ (lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker-ce.list
```
> [!info] 🔎 说明：
> - `$ (lsb_release -cs)`：自动获取 Ubuntu 发行版代号（如 focal, jammy 等）。
> - 阿里云地址：`https://mirrors.aliyun.com/docker-ce/linux/ubuntu`

#### ✅ 方式二：使用官方源（可选）
```bash
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $ (lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker-ce.list
```

### 🔄 步骤四：更新 APT 缓存
```bash
sudo apt update
```
> [!info] ✅
> 这一步会从新添加的仓库中读取包信息

### ✅步骤五：安装Docker
```bash
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```
> [!info] 💡 说明：
> - `docker-ce`：Docker 引擎（核心）
> - `docker-ce-cli`：命令行工具
> - `containerd.io`：容器运行时
> - `docker-buildx-plugin`：支持多平台构建
> - `docker-compose-plugin`：Docker Compose v2 插件（推荐使用）
> 📌 如果你只想安装基础版本，可以只保留前三个包。

非常好！你已经完成了前面的准备工作，现在我们来将 **CentOS 上的 Docker 安装与启动流程**，完整地转换为适用于 **Ubuntu 系统** 的等价指令。

---

## ✅ 启动和校验
### 🔧 启动 Docker 服务
```bash
sudo systemctl start docker
```

### ⏹ 停止 Docker 服务

```bash
sudo systemctl stop docker
```

### 🔁 重启 Docker 服务

```bash
sudo systemctl restart docker
```

### 🚀 设置开机自启

```bash
sudo systemctl enable docker
```
> [!tip] ✅
> 这样每次系统启动都会自动运行 Docker 服务。

### ✅ 校验安装是否成功
执行以下命令测试：
```bash
docker ps
```
如果输出如下内容，说明安装成功：

```text
CONTAINER ID   IMAGE     COMMAND    CREATED   STATUS    PORTS     NAMES
```
> [!error] ❗
> 如果报错如 `command not found`，请检查是否正确安装了 `docker-ce-cli`。

### 🛠️ 可选：验证 Docker 版本

```bash
docker --version
```
或查看详细信息：
```bash
docker info
```

---

### ✅ 总结：Ubuntu 安装 Docker 完整流程

```bash
# 1. 卸载旧版（如有）
sudo apt remove docker docker-engine docker.io containerd runc

# 2. 安装依赖
sudo apt update
sudo apt install -y apt-transport-https ca-certificates curl gnupg lsb-release

# 3. 添加 GPG 密钥
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

# 4. 添加阿里云镜像源（推荐）
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://mirrors.aliyun.com/docker-ce/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker-ce.list

# 5. 更新缓存
sudo apt update

# 6. 安装 Docker
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# 7. 启动并设置开机自启
sudo systemctl start docker
sudo systemctl enable docker

# 8. 验证
docker ps
```

---