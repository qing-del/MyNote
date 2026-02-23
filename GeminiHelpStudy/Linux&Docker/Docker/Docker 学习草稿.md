## 常见命令
- `docker images` 查看 Docker 中的镜像列表
- `docker ps` 查看**正在运行**的容器列表
- `docker ps -a` 查看**所有**容器
- `docker inspect [容器名]` 查看指定容器的详细信息

> [!info] 以下是[[Docker 命令图.canvas]]的详细参数
- `docker pull [镜像名称(:版本)]` 从仓库中下载镜像
- `docker save -o [保存后文件名] [镜像(:镜像版本)]`  将镜像保存为文件
- `docker rmi [镜像名字(:版本)]` 删除镜像
- `docker load -i [文件名]` 将`tar`文件加载成镜像
- `docker run -d --name [容器名] -p [外端口]:[内端口] [镜像名(:版本)]` 详见->[[Docker 运行容器]]
- `docker stop [容器名]` 停止容器的运行
- `docker start/restart [容器名]` 启动容器
- `docker exec -it [容器名] bash` 进入容器
> `-it`是分配一个终端，`bash`是进入容器后执行的命令模式
> 可以通过 `exit` 退出容器
- `docker rm [容器名]` 移除容器
> 加入`-f`属性可以强制移除。例如：`docker rm -f [容器名]/[容器id]`


## 数据卷
- **数据卷**是一个虚拟目录，是**容器内目录**与**宿主机目录**之间映射的桥梁

> [!warning] 注意：
> 在执行`docker run`命令时，使用 `-v 数据卷:完整容器目录` 形式可以完成数据卷挂载（数据卷不存在会自动创建）

### 匿名数据卷
- 用于保存数据库的数据

## 自定义镜像
- 常见命令：
> 全部指令可以去查看Docker的官网  


## 网络
- 同一个虚拟网卡下的容器，可以通过`NAME`属性来平替容易变动的`ip`地址
> [!tip] 提示
> 也可以通过`docker run -d --name [容器名] -p [外端口]:[内端口] --network [虚拟网卡] [镜像(:版本)]`来加入虚拟网卡

## DockerCompose
- 