
### 🟢 Level 1: 基础热身 (Command Line Drills)

_目标：验证环境，习惯 Docker 的操作手感。_

- [x] **任务 1.1：环境体检**
    - 在终端执行 `docker -v` 和 `docker compose version`，确保都输出了版本号。
    - _检查点_：如果不报错，说明安装成功。

- [x] **任务 1.2：运行第一个 Web 服务 (Nginx)**
    - 执行命令运行一个临时的 Nginx：
        ```Bash
        docker run -d --name test-nginx -p 80:80 nginx
        ```
    - 打开浏览器（或用手机），访问你那台 Ubuntu 服务器的 IP 地址。
    - _检查点_：看到 "Welcome to nginx!" 的页面。
        
- [x] **任务 1.3：清理战场**
    
    - 练习删除命令：停止并删除刚才的容器。
        ```Bash
        docker stop test-nginx
        docker rm test-nginx
        ```

---

### 🔵 Level 2: 数据持久化 (Database)

_目标：应用《Docker 进阶：数据卷与网络》和《Docker 运行容器》中的知识，搭建稳固的数据库。_

- [x] **任务 2.1：创建数据卷**
    - 为了防止删容器导致删库跑路，先创建一个 Docker 数据卷：
        ```Bash
        docker volume create mysql_data
        ```
        
- [x] **任务 2.2：启动 MySQL 8**
    - 参考你的笔记 `Docker 运行容器.md`，运行 MySQL 容器。
    - **要求**：
        1. 映射端口 `3306:3306`。
        2. 挂载数据卷 `mysql_data` 到 `/var/lib/mysql`
        3. 设置 Root 密码（建议设为简单的，如 `root`，方便测试）。
        4. **关键**：设置时区 `-e TZ=Asia/Shanghai`。
            
- [x] **任务 2.3：数据导入**
    - 用你电脑上的 Navicat 或 DBeaver，连接这台 Ubuntu 服务器的 3306 端口。
    - 连接成功后，运行你 `task-system` 的 SQL 脚本（建表语句）。
    - _检查点_：在数据库工具里能看到 `task` 表和 `user` 表。

---

### 🟠 Level 3: 后端容器化 (Dockerfile)

_目标：应用《Docker 高级：Dockerfile》笔记，把 Spring Boot 打包成镜像。_

- [x] **任务 3.1：准备 JAR 包**
    
    - 在你本地电脑（IDEA）里，执行 Maven `package`，拿到 `task-system-0.0.1-SNAPSHOT.jar`。
    - 通过 FinalShell 将这个 jar 包上传到 Ubuntu 的 `/root/lifegame/backend` 目录（需新建目录）。

- [x] **任务 3.2：编写 Dockerfile**
    - 在同级目录下新建 `Dockerfile`，内容参考你之前的学习（基础镜像 `openjdk:17-jdk-slim`）。

- [x] **任务 3.3：构建镜像**
    - 执行构建命令：
        ```Bash
        docker build -t lifegame-backend:v1.0 .
        ```
    - _检查点_：执行 `docker images` 能看到这个镜像。

- [x] **任务 3.4：试运行**
    - 尝试启动它（注意：此时它可能连不上 MySQL，因为我们还没配网络，但我们要先看它能不能跑起来）：
        ```Bash
        docker run -d --name backend -p 8080:8080 lifegame-backend:v1.0
        ```
        
    - 查看日志：`docker logs -f backend`。
    - _预期_：可能会报错“连接数据库失败”，但这说明 Java 环境是好的！

---

### 🔴 Level 4: 终极编排 (Docker Compose)

_目标：应用《Docker 高级：DockerCompose》笔记，一键部署全栈。_
这是最爽的一步，我们将解决 Level 3 中“连不上数据库”的问题。

- [ ] **任务 4.1：准备目录**
    - 在 `/root/lifegame` 下创建 `docker-compose.yml` 文件。

- [ ] **任务 4.2：编写 Compose 文件**
    - 参考你的笔记，定义三个服务：
        1. **`mysql`**：使用 `mysql:8` 镜像，配置数据卷。
        2. **`backend`**：使用刚才构建的 `lifegame-backend:v1.0` 镜像。
            - **关键修改**：你需要修改 Spring Boot 的 `application.yml`，把数据库连接地址从 `localhost` 改为 `mysql` (Docker 内部服务名)。_（或者利用 Docker 环境变量覆盖，这个比较高级，你可以先重新打包一个改了配置的 jar）_
        3. **`frontend`**：(可选) 如果你想挑战全栈，把 Vue 也打包成镜像（Nginx）；如果想省事，先只跑后端，前端在本地电脑跑。
            
- [ ] **任务 4.3：一键启动**
    - 停止并删除之前手动跑的所有容器。
    - 执行：
        ```Bash
        docker compose up -d
        ```
        
- [ ] **任务 4.4：最终验收**
    - `docker compose ps` 查看所有服务状态是否为 `Up`。
    - 用浏览器访问后端接口（如 `http://服务器IP:8080/task/list`，需配合 Token 测试）。

---

### 💡 导师的特别提示 (Cheat Sheet)

1. **关于修改数据库地址**：
    在 Docker Compose 网络中，容器之间互访**不能用 `localhost`**（因为 localhost 指的是容器自己）。
    - **错误**：`jdbc:mysql://localhost:3306/...`
    - **正确**：`jdbc:mysql://mysql:3306/...` (这里的 `mysql` 对应 `docker-compose.yml` 里的服务名)
  
2. **关于文件上传**：
    FinalShell 的下方有一个文件传输窗口，可以直接拖拽文件上传，非常方便。

**请先开始 Level 1 和 Level 2！** 遇到问题（比如数据库连不上）随时把报错截图发给我。