---
title: Linux 基础指令与前后端环境部署
tags:
  - Linux
  - Frontend
  - Backend
  - DevOps
  - CentOS
  - Learning
  - Commands
create_time: 2026-01-23
---

## 目录 (Table of Contents)
- [[Linux 基础指令与前后端环境部署#1. Linux 系统概述与环境搭建| Linux 系统概述与环境搭建]]
- [[Linux 基础指令与前后端环境部署#2. Linux 文件系统与目录结构| Linux 文件系统与目录结构]]
- [[Linux 基础指令与前后端环境部署#3. Linux 常用命令详解| Linux 常用命令详解]]
    - [[Linux 基础指令与前后端环境部署#3.1 基础文件操作| 基础文件操作]]
    - [[Linux 基础指令与前后端环境部署#3.2 文件内容查看| 文件内容查看⭐]]
    - [[Linux 基础指令与前后端环境部署#3.3 复制、移动与删除| 复制、移动与删除]]
    - [[Linux 基础指令与前后端环境部署#3.4 搜索与查找| 搜索与查找⭐]]
    - [[Linux 基础指令与前后端环境部署#3.5 压缩与解压 (tar)| 压缩与解压]]
    - [[Linux 基础指令与前后端环境部署#3.6 文本编辑器 (vi/vim)| 文本编辑器 (vi/vim)]]
	- [[Linux 基础指令与前后端环境部署#3.7 扩展：权限管理 (chmod/chown)| 扩展：权限管理 (chmod)]]
	- [[Linux 基础指令与前后端环境部署#3.8 扩展：进程与网络 (ps/netstat)| 扩展：进程与网络 (ps/netstat)]]
	- [[Linux 基础指令与前后端环境部署#4. 软件安装与环境部署| 软件安装与环境部署]]
    - [[Linux 基础指令与前后端环境部署#4.1 安装方式综述| 安装方式综述]]
    - [[Linux 基础指令与前后端环境部署#4.2 JDK 安装| JDK 安装]]
    - [[Linux 基础指令与前后端环境部署#4.3 MySQL 安装| MySQL 安装]]
	    - [[#远程数据库连接]]
    - [[Linux 基础指令与前后端环境部署#4.4 Nginx 安装| Nginx 安装]]
	    - [[Linux 基础指令与前后端环境部署#4.4.1 Nginx 常用操作命令| Nginx 常用操作命令]]
- [[Linux 基础指令与前后端环境部署#5. 防火墙配置| 防火墙配置]]
- [[Linux 基础指令与前后端环境部署#6. 实战：前后端分离项目部署| 前后端分离项目部署]]
	- [[Linux 基础指令与前后端环境部署#6.1 后端项目部署 (Jar 包)| 后端项目部署（Jar 包）]]
	- [[Linux 基础指令与前后端环境部署#6.2 前端项目部署 (Nginx)| 前端项目部署（Nginx）]]
	- [[Linux 基础指令与前后端环境部署#6.3 Nginx 核心原理：反向代理与重写详解]| 反向代理与重写详解]]
		- [[Linux 基础指令与前后端环境部署#💡 扩展：面试高频题 (Interview Q&A)| 原理面试题]]

---

## 1. Linux 系统概述与环境搭建

### 1.1 操作系统简介
-   **定义**：Linux 是一套免费使用和自由传播的操作系统。
-   **分类**：
    -   **内核版**：由 Linux 核心团队开发维护，免费开源，负责控制硬件。
    -   **发行版**：基于内核版扩展，由各厂商开发（如 Ubuntu, RedHat, CentOS）。
-   **CentOS**：RedHat 的社区版，免费且稳定，是后端服务器部署的主流选择。

### 1.2 虚拟机安装与配置
> [!info] 工具说明
> -   **VMware Workstation**：虚拟化软件，模拟完整硬件系统。课程推荐版本：`16.1.0`。
> -   **CentOS 7**：推荐使用的 Linux 镜像版本。

**网络配置步骤 (NAT模式)**：
1.  **设置子网 IP**：在 VMware 的“虚拟网络编辑器”中，选择 NAT 模式，将子网 IP 设置为 `192.168.100.0`。
2.  **配置 Windows 网络**：确保主机虚拟适配器 (VMnet8) 已启用。
3.  **常用网络命令**：
    -   `ip addr`：查看当前 IP 地址。
    -   `./start-ens33.sh`：如果没 IP，可执行此脚本手动激活网卡（视具体环境而定）。

### 1.3 远程连接工具
为了方便操作，通常使用 SSH 工具远程连接服务器。
-   **常用工具**：Putty, Xshell, SecureCRT, **FinalShell** (推荐，自带上传下载功能)。
-   **连接参数**：主机 IP、端口 (默认 22)、用户名 (root)、密码。

[[Linux 基础指令与前后端环境部署#目录 (Table of Contents)| 回到目录]]

---

## 2. Linux 文件系统与目录结构

Linux 的文件系统是一个倒挂的树状结构，`/` 是所有目录的顶点。

### 核心目录说明
| 目录 | 作用 | 备注 |
| :--- | :--- | :--- |
| **`/bin`** | 存放二进制可执行文件 | 基础命令如 `ls`, `cp` 等 |
| **`/etc`** | **存放系统配置文件** | 如 `/etc/profile` (环境变量) |
| **`/home`** | 存放普通用户的主目录 | 类似 Windows 的 `Users` |
| **`/root`** | **超级用户目录** | 管理员 root 的家 |
| **`/usr`** | **存放系统应用程序** | 类似 Windows 的 `Program Files`，常用 `/usr/local` |
| **`/var`** | 存放经常变化的文件 | 如日志文件 `/var/log` |

[[Linux 基础指令与前后端环境部署#目录 (Table of Contents)| 回到目录]]

---

## 3. Linux 常用命令详解

**命令通用格式**：`command [-options] [parameter]`
> [!tip] 可以通过 `cat /etc/os-release` 来查看当前Linux是什么版本的

### 3.1 基础文件操作

-   **`ls` (List)**：显示指定目录下的内容。
    -   `ls -a`：显示所有文件（含 `.` 开头的隐藏文件）。
    -   `ls -l`：以列表形式显示详细信息（权限、大小、时间等）。
    -   **`ll`**：`ls -l` 的简写（CentOS 中常用）。
-   **`cd` (Change Directory)**：切换当前目录。
    -   `cd ..`：返回上一级。
    -   `cd ~`：返回当前用户的 Home 目录。
    -   `cd -`：切换回上一次所在的目录。
-   **`pwd`**：查看当前所在路径 (Print Working Directory)。
-   **`mkdir` (Make Directory)**：创建目录。
    -   `mkdir -p a/b/c`：创建多级目录（如果父目录不存在自动创建）。

> [!tip] 💡 实战场景：快速搭建项目结构 后端开发时，经常需要一次性建立多层目录结构。
> **场景**：你要部署一个名为 `myapp` 的项目，里面需要存放日志和配置。 **操作**：
> ```Bash
> # 一次性创建父子目录
> mkdir -p /usr/local/myapp/logs /usr/local/myapp/conf
> 
> # 快速进入该目录 
> cd /usr/local/myapp
> ```



### 3.2 文件内容查看

| 命令 | 作用 | 适用场景 | 常用参数 |
| :--- | :--- | :--- | :--- |
| **`cat`** | 查看所有内容 | **小文件** | `-n` (显示行号) |
| **`more`** | 分页显示内容 | **大文件** | 回车(下一行), 空格(下一页), q(退出) |
| **`head`** | 查看文件**开头** | 确认文件头部 | `head -10 filename` (看前10行) |
| **`tail`**⭐ | 查看文件**结尾** | **查看日志** | **`tail -f filename`** (动态实时监控日志) |

> [!example] 💡 实战场景：生产环境日志排查 **背景**：你的 Java 后端项目报错了，你需要实时监控日志文件，看最新的错误信息。 **操作**：
> 1. 使用 `tail -f` (follow) 挂起终端，实时输出新增内容。
>  ```Bash
>  tail -f /var/log/nginx/error.log
>  ```
>  > 退出的话可以使用`ctrl + C`就可以退出实时监控模式
>  
> 2. 结合 `grep` 过滤只看报错信息（配合管道符 `|`）。
>  ```Bash
>  # 实时监控，且只显示包含 "Error" 的行
>  tail -f catalina.out | grep "Error"
>  ```

### 3.3 复制、移动与删除

-   **`cp` (Copy)**：复制文件或目录。
    -   `cp source dest`：复制文件。
    -   `cp -r sourceDir destDir`：递归复制**目录**。
-   **`mv` (Move)**：移动或重命名。
    -   `mv a.txt b.txt`：**重命名** (如果 b.txt 不存在)。
    -   `mv a.txt /usr/local/`：**移动** (如果目标是目录)。
-   **`rm` (Remove)**：删除。
    -   `rm -r`：递归删除目录。
    -   `rm -f`：强制删除，无需确认。
    -   **`rm -rf *`**：**高危操作**，强制删除当前目录下所有内容。

> [!warning] 💡 最佳实践：修改配置前的备份 在修改 Nginx 或 MySQL 核心配置文件前，**永远先备份**。 **场景**：修改 `nginx.conf`。 **操作**：
> ```Bash
> # 1. 备份 (习惯在文件名后加 .bak 或 .日期)
> cp nginx.conf nginx.conf.bak.20240123
> 
> # 2. 如果改坏了，快速恢复
> cp nginx.conf.bak.20240123 nginx.conf
> ```



### 3.4 搜索与查找

> [!question] `find` 与 `grep` 的区别？
> - **`find`**：根据文件的**属性**（如文件名）查找文件。
> - **`grep`**：根据指定的**关键字**查找**文件内容**。

-   **`find`**：
    -   语法：`find [路径] -name "文件名模式"`
    -   示例：`find / -name "*.log"` (全盘查找后缀为 .log 的文件)。
-   **`grep`**：⭐
    -   语法：`grep [选项] "关键字" 文件名`
    -   常用参数：
        -   `-i`：忽略大小写。
        -   `-n`：显示行号。

> [!example] 💡 实战场景：查找“失踪”的配置文件 **背景**：你接手了一个老项目，不知道 Nginx 安装在哪里，只知道它肯定叫 `nginx.conf`。 **操作**：
> ```Bash
> # 全盘查找名为 nginx.conf 的文件
> find / -name "nginx.conf"
> ```

> [!example] 💡 实战场景：排查代码或进程 **背景**：你想确认当前服务器有没有运行 Java 程序。 **操作**：
> ```Bash
> # ps -ef 列出所有进程，grep 过滤关键词
> ps -ef | grep java
> ```

### 3.5 压缩与解压 (tar)
Linux 中常用 `.tar.gz` 格式。

-   **常用参数**：
    -   `z`：使用 gzip 压缩/解压。
    -   `c`：创建 (Create) 包。
    -   `x`：解包 (Extract)。
    -   `v`：显示过程 (Verbose)。
    -   `f`：指定文件名 (File)。
-   **核心命令**：
    -   **打包压缩**：`tar -zcvf test.tar.gz file1 dir1`
    -   **解压**：`tar -zxvf test.tar.gz`
    -   **解压到指定目录**：`tar -zxvf test.tar.gz -C /usr/local` (注意 `-C` 参数)。

### 3.6 文本编辑器 (vi/vim)
`vim` 是 `vi` 的增强版，支持颜色显示。

**三种模式切换**：
1.  **命令模式** (默认)：按 `Esc` 进入。可进行删除 (`dd`)、复制 (`yy`)、粘贴 (`p`)。
2.  **插入模式**：在命令模式按 `i`、`a` 或 `o` 进入。可以正常编辑文本。
3.  **底行模式**：在命令模式按 `:` 进入。
    -   `:wq`：保存并退出。
    -   `:q!`：不保存强制退出。
    -   `:set nu`：显示行号。

### 3.7 扩展：权限管理 (chmod/chown)
Linux 对文件的权限控制非常严格。
- `r` (read, 4): 读
- `w` (write, 2): 写
- `x` (execute, 1): 执行

**常用命令**：
- `chmod` (Change Mode)：修改权限。
- `chown` (Change Owner)：修改拥有者。

> [!example] 💡 实战场景：脚本无法执行 **问题**：你上传了一个启动脚本 `start.sh`，输入 `./start.sh` 提示 `Permission denied`。 **原因**：文件默认没有“执行(x)”权限。 **解决**：
> ```Bash
> # 方式1：赋予所有用户执行权限 (推荐脚本使用)
> chmod +x start.sh
> 
> # 方式2：赋予最高权限 (7=4+2+1，慎用)
> chmod 777 start.sh
> ```



### 3.8 扩展：进程与网络 (ps/netstat)
后端排查问题的核心：**进程在不在？端口通不通？**
1. **进程管理**
    - **`nohup`**：可以挂起服务器 使其在后台运行
    - **`ps -ef`**：查看全格式进程信息。
    - **`kill -9 [PID]`**：强制杀死进程（PID 是进程号）。
    - **`top`**：类似 Windows 任务管理器，实时查看 CPU/内存占用。
    
    > [!tip] 组合技：杀掉卡死的 Java 进程
    > ```Bash
    > # 1. 挂起进程
    > nohup java -jar xxxxx.jar &> tlias.log &
    > 
    > # 2. 查找进程号 (假设找到 PID 为 12345)
    > ps -ef | grep java
    > 
    > # 3. 强制关闭
    > kill -9 12345
    > ```
    
1. **网络监听**
    - **`netstat -ntlp`**：查看当前所有监听端口及对应的程序。
    - **`lsof -i :端口号`**：查看谁在占用这个端口。
    
    > [!example] 💡 实战场景：启动服务报错“端口被占用” **问题**：启动 Tomcat 报错 `Address already in use: 8080`。 **解决**：
    > ```Bash
    > # 1. 查谁占了 8080
    > netstat -ntlp | grep 8080
    > # 或者
    > lsof -i :8080
    > 
    > # 2. 根据查到的 PID 杀掉旧进程
    > kill -9 [PID]
    > ```

> [!tip] Linux 中的特殊符号：
> - `|` 管道符。将前面命令的输出，作为后面指令的输入。
> 例如：
> ```bash
> ps -ef | grep java
> ```
> - `>`与`>>` 重定向符号。将前面的文本内容，输出到后面的文件中
> 	- `>` **覆盖**重定向
> 	- `>>` **追加**重定向
> 例如：
> ```bash
> echo 'Hello Linux' > 1.log
> ```

[[Linux 基础指令与前后端环境部署#目录 (Table of Contents)| 回到目录]]

---

## 4. 软件安装与环境部署

### 4.1 安装方式综述
1.  **二进制发布包**：解压即用，需手动配置 (如 JDK)。
2.  **RPM 包**：按照规范打包，无法自动解决依赖。
3.  **Yum 在线安装**：自动下载并安装，**自动解决依赖** (方便)。
4.  **源码编译安装**：需手动编译打包 (如 Nginx, Redis)。

### 4.2 JDK 安装
1.  **上传**：使用 FinalShell 将 `.tar.gz` 包上传至服务器。
2.  **解压**：`tar -zxvf jdk-xxx.tar.gz -C /usr/local`。
3.  **配置环境变量**：
    -   编辑配置文件：`vim /etc/profile`。
    -   末尾追加：
        ```bash
        export JAVA_HOME=/usr/local/jdk-17.0.10
        export PATH=$JAVA_HOME/bin:$PATH
        ```
4.  **生效与验证**：
    -   `source /etc/profile`
    -   `java -version`

### 4.3 MySQL 安装
1.  **清理旧环境**：检查并卸载自带的 mariadb。
    -   `rpm -qa | grep mariadb`
    -   `rpm -e --nodeps mariadb-libs-xxx`
2.  **上传与解压**：解压后移动到 `/usr/local/mysql`。
3.  **配置环境**：在 `/etc/profile` 中添加 `MYSQL_HOME`。
4.  **初始化**：
    -   创建用户组：`groupadd mysql`, `useradd -r -g mysql ...`
    -   初始化数据：`mysqld --initialize ...` (**记下生成的临时 root 密码**)。
5.  **启动与配置**：
    -   `systemctl start mysql`
    -   登录并修改密码，开启远程访问权限：
        ```sql
        ALTER USER 'root'@'localhost' IDENTIFIED BY '1234';
        GRANT ALL PRIVILEGES ON *.* TO 'root'@'%';
        FLUSH PRIVILEGES;
        ```


### 4.4 Nginx 安装
1.  **安装依赖**：
    ```bash
    yum install -y pcre pcre-devel zlib zlib-devel openssl openssl-devel
    ```
2.  **配置与编译**：
    -   解压源码包。
    -   执行配置：`./configure --prefix=/usr/local/nginx`
    -   编译安装：`make` 然后 `make install`。
3.  **启动**：进入 `/usr/local/nginx/sbin` 执行 `./nginx`。

#### 4.4.1 Nginx 常用操作命令
进入 Nginx 安装目录的 `sbin` 目录下执行：
- **启动**：`./nginx`
- **停止**：`./nginx -s quit`
- **重新加载配置文件** (修改配置后必须执行)：`./nginx -s reload`

[[Linux 基础指令与前后端环境部署#目录 (Table of Contents)| 回到目录]]

---

## 5. 防火墙配置

为了让外部（如 Windows 浏览器）能访问 Linux 上的服务（MySQL: 3306, Nginx: 80），必须配置防火墙。

-   **查看状态**：`systemctl status firewalld`
-   **关闭防火墙** (生产环境不建议)：`systemctl stop firewalld`
-   **开放指定端口** (推荐)：
    ```bash
    # 开放 8080 端口
    firewall-cmd --zone=public --add-port=8080/tcp --permanent
    
    # 立即生效 (必须执行)
    firewall-cmd --reload
    ```
-   **查看已开放端口**：`firewall-cmd --zone=public --list-ports`

> [!info] 💡 场景扩展：云服务器的安全组 如果你使用的是阿里云/腾讯云/AWS 等云服务器，仅仅在 Linux 内部关闭防火墙或开放端口可能**无效**。

> **注意**：必须同时在云厂商的**网页控制台 -> 安全组 (Security Group)** 中开放对应的端口（如 80, 3306, 8080），否则外网无法访问。


#### 远程数据库连接
- 需要访问服务器数据库的主机，先执行下列命令查看是否能连通服务器
```bash
ping [服务器地址]
```
> 如果连得通，说明是MySQL的配置有问题

- 配置 Linux 中 MySQL 的配置
```bash
# 1. 修改配置
sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf
# 确保: bind-address = 0.0.0.0

# 2. 重启服务
sudo systemctl restart mysql

# 3. 再次检查
sudo ss -tuln | grep 3306
```
> 后续再次尝试`mysql -h[服务器ip] -u [用户名] -p`
> 后续输入密码即可

[[Linux 基础指令与前后端环境部署#目录 (Table of Contents)| 回到目录]]

---

## 6. 实战：前后端分离项目部署

### 6.1 后端项目部署 (Jar 包)
后端部署的核心是将 Java 项目打包并在服务器上后台运行
1. **打包**：
    在 Maven 父工程中执行 `package` 生命周期，确保本地测试通过后，生成 `.jar` 包。
2. **上传**：
    将 Jar 包上传至服务器目录，例如 `/usr/local/app`。
3. **启动服务**：
    - **前台启动** (测试用)：
        ```Bash
        java -jar xxxxx.jar
        ```
         > [!tip] 缺点：占用当前窗口，窗口关闭服务即停止。
        
    - **后台启动** (生产用) ⭐：
        使用 `nohup` 命令让服务在后台运行，并将日志输出到指定文件。
        ```  Bash
        nohup java -jar xxxxx.jar &> tlias.log &
        ```
        - `nohup`：不挂断地运行命令。
        - `&>`：将标准输出和错误输出同时重定向到日志文件。
        - 最后的 `&`：表示在后台运行。

4. **进程检查与停止**：
    - **查看进程**：`ps -ef | grep java`
    - **停止服务**：`kill -9 [进程PID]`

### 6.2 前端项目部署 (Nginx)
前端部署的核心是将静态资源托管到 Nginx，并配置反向代理以连接后端。

1. **上传资源**：
    将前端打包好的静态资源（通常是 `dist` 文件夹下的内容）上传到 Nginx 安装目录下的 `html` 目录中。

2. **修改配置**：
    编辑配置文件 `conf/nginx.conf`，主要配置**静态资源路径**和**反向代理**。
    ```Nginx
    server {
        listen       80;
        server_name  localhost;
    
        # 前端静态资源配置
        location / {
            root   html;
            index  index.html index.htm;
            # 解决单页应用(SPA)刷新404问题
            try_files $uri $uri/ /index.html; 
        }
    
        # 反向代理配置：将 /api/ 开头的请求转发到后端 Tomcat
        location ^~ /api/ {
            # 路径重写：去掉 /api 前缀 (根据后端接口实际情况决定是否需要)
            rewrite ^/api/(.*)$ /$1 break; 
            # 转发到后端接口地址
            proxy_pass http://localhost:8080; 
        }
    }
    ```
    
![[Linux项目部署学习1.png]]
    
2. **生效配置**：
    修改完配置文件后，必须执行重载命令：
    
    ```Bash
    ./sbin/nginx -s reload
    ```

### 6.3 Nginx 核心原理：反向代理与重写详解
在前后端分离项目中，`nginx.conf` 中的这两行配置是最核心、也最容易混淆的部分：

```Nginx
location ^~ /api/ {
    rewrite ^/api/(.*)$ /$1 break;   # 路径重写
    proxy_pass http://localhost:8080; # 反向代理
}
```

#### 1. 为什么要这么配置？(业务背景)

- **前端诉求**：为了区分“静态页面请求”和“后端接口请求”，前端代码通常会在所有接口路径前加上 `/api` 前缀
	- 例如：`http://localhost/api/users`
- **后端现状**：Java 的 Controller 接口定义通常没有 `/api`
	- 例如：`@GetMapping("/users")`
- **矛盾**：如果直接转发 `/api/users` 给后端，Tomcat 会报 404。
- **解决**：Nginx 需要在转发前，**把 `/api` 这个前缀“切掉”**。

#### 2. 指令深度解析
|**指令**|**作用**|**详细解读**|
|---|---|---|
|**`proxy_pass`**|**反向代理**|相当于“传话筒”。凡是匹配到 `/api/` 的请求，Nginx 不自己处理，而是转交给 `http://localhost:8080` (你的 Java 服务)。|
|**`rewrite`**|**路径重写**|相当于“整容”。通过正则表达式修改请求的 URL 路径。|

> [!info] `proxy_pass`的作用
> - **含义**：凡是匹配到 `/api/` 的请求，Nginx 不自己处理了，而是转交给 `http://localhost:8080` (你的 Java 服务) 去处理。
> - **形象比喻**：Nginx 是餐厅门口的接待员，Tomcat 是后厨。客人（浏览器）找接待员点菜，接待员把需求传给后厨。

> [!tip] 💡 正则表达式 `rewrite ^/api/(.*)$ /$1 break;` 拆解：
> - `^/api/`：匹配以 `/api/` 开头的内容。
> - `(.*)`：**捕获组**。括号表示“抓取”，意思就是把 `/api/` 后面剩下的所有字符都抓出来，存入变量 `$1` 中。
> - `/$1`：**替换**。用刚才抓到的 `$1` (即不带 api 的路径) 替换掉整个原路径。
> - `break`：停止处理后续重写规则，直接执行下面的代理。

#### 3. 请求流转全过程图解
假设浏览器请求地址为：`http://localhost/api/system/login`

|**步骤**|**处理者**|**动作**|**URL 路径变化**|
|---|---|---|---|
|1|**浏览器**|发起请求|`/api/system/login`|
|2|**Nginx**|拦截请求 (`location /api/`)|`/api/system/login`|
|3|**Nginx**|**执行 rewrite** (截取 `/api` 后的内容)|变为 `/system/login`|
|4|**Nginx**|执行 `proxy_pass`|转发 `/system/login`|
|5|**Tomcat**|接收请求|收到 `/system/login` -> **匹配成功** ✅|

#### 💡 扩展：面试高频题 (Interview Q&A)
> [!question] Q1: 为什么前端写 `/api/login`，后端写 `/login` 也能调通？
> **A:** 因为 Nginx 做了**路径重写 (Rewrite)**。Nginx 截获请求后，利用正则去掉了 `/api` 前缀，再将处理后的路径转发给后端，所以后端收到的实际上就是 `/login`。

> [!question] Q2: `proxy_pass` 结尾加 `/` 和不加 `/` 有什么区别？
> - `proxy_pass http://localhost:8080;` (不加斜杠)：Nginx 会把匹配到的路径（含 `/api`）**完整**拼接到后面。-- **不加入`/`就需要写`rewrite`方法来进行重写，重写`rewrite`在实际开发中更推荐✅** 
> - `proxy_pass http://localhost:8080/;` (加斜杠)：Nginx 会**自动去掉**匹配到的 `/api` 部分（类似一种**隐式的 rewrite**，但在复杂场景下建议手动写 rewrite 更清晰）。
> > [!error] 注意：
> > 如果写了 `rewrite` 又加入了 `/` 会导致二次重写然后出现错误！！！


[[Linux 基础指令与前后端环境部署#目录 (Table of Contents)| 回到目录]]

---