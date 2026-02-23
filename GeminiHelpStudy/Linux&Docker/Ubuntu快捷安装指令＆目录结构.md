# Nginx
### `sudo apt install nginx`
- **目录结构**：
	- |路径|说明|
	|---|---|
	|`/etc/nginx/`|配置文件目录（核心配置在 `nginx.conf`）|
	|`/var/www/html/`|默认网页根目录（存放网站文件）|
	|`/usr/sbin/nginx`|Nginx 可执行文件|
	|`/var/log/nginx/`|日志文件目录（access.log, error.log）|
> 修改`nginx.conf`必须去指定配置文件目录修改

- **自定义项目路径** : 通过修改`nginx.conf`来实现
```nginx
# /etc/nginx/nginx.conf （或 include 其他配置文件）
http {
    # ...
    server {
        listen 80;
        server_name localhost;

        # ✅ 关键：这里可以写任意路径！
        root ${ 你的项目目录 };   # 你自己的项目目录
        index index.html;

        location / {
            try_files  $ uri  $ uri/ =404;
        }
    }
}
```
- [[Nginx 卸载 | 在 Ubuntu 中卸载 Nginx 的示例]]

# MySQL 安装
### `sudo apt install mysql`
- **目录结构：**
	- |路径|说明|
	|---|---|
	|`/etc/mysql/`|配置文件目录（主配置 `my.cnf`）|
	|`/usr/sbin/mysqld`|MySQL 服务可执行程序|
	|`/var/lib/mysql/`|默认数据目录（最重要的！）|
	|`/var/log/mysql/`|日志文件（错误日志、慢查询日志等）|
	|`/run/mysqld/`|运行时 socket 文件（PID、socket）|
	|`/usr/share/mysql/`|系统数据库表结构、字符集、SQL 脚本|

> [!warning] ⚠️ 特别注意：  
> **`/var/lib/mysql/` 是默认存放所有数据库文件的地方**，包括 `.ibd`、`.frm`、

- `mysql/` 系统库等。
	- **可修改**路径的目录：
		- |路径|是否可改|修改方式|
		|---|---|---|
		|数据目录 (`datadir`)|✅ 可以|修改 `my.cnf` 中的 `datadir`|
		|错误日志路径|✅ 可以|设置 `log_error`|
		|慢查询日志路径|✅ 可以|设置 `slow_query_log_file`|
		|socket 文件路径|✅ 可以|设置 `socket`|
		|临时目录|✅ 可以|设置 `tmpdir`|

> [!tip] 附加信息
> 修改实例详见 -> [[MySQL 修改数据目录]]

- 停止`MySQL`服务
```bash
sudo systemctl stop mysql
```
> [!warning] 卸载实例
> [[MySQL 卸载 | Ubuntu 中卸载 MySQL]]