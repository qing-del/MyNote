
---

### 🛠 如何安全地修改数据目录（例如迁移到 `/data/mysql`）

#### 场景：你想把数据库从 `/var/lib/mysql` 移动到 `/data/mysql`

##### 步骤 1：停止 MySQL 服务

```bash
sudo systemctl stop mysql
```

##### 步骤 2：创建新目录并赋予权限

```bash
sudo mkdir -p /data/mysql
sudo chown -R mysql:mysql /data/mysql
sudo chmod 700 /data/mysql
```

##### 步骤 3：将旧数据复制过去

```bash
sudo cp -r /var/lib/mysql/* /data/mysql/
```

> ⚠️ 确保复制完整，不要遗漏任何文件。

##### 步骤 4：修改配置文件

编辑主配置文件：

```bash
sudo nano /etc/mysql/my.cnf
```

找到或添加以下内容：

```ini
[mysqld]
datadir = /data/mysql
socket = /data/mysql/mysql.sock
log_error = /data/mysql/error.log
slow_query_log_file = /data/mysql/slow-query.log
```

> 💡 提示：也可以在 `/etc/mysql/mysql.conf.d/mysqld.cnf` 里修改，取决于你的系统版本。

##### 步骤 5：更新 systemd 服务（如果需要）

如果你修改了 socket 位置，可能还需要更新服务文件：

```bash
sudo systemctl daemon-reload
```

##### 步骤 6：启动 MySQL

```bash
sudo systemctl start mysql
```

##### 步骤 7：验证是否成功

```bash
sudo systemctl status mysql
mysql -u root -p
```

进入后执行：

```sql
SHOW VARIABLES LIKE 'datadir';
```

应显示：`/data/mysql`

✅ 完成！现在数据库已迁移到自定义路径。

---

### ⚠️ 重要提醒

- **不要直接删除 `/var/lib/mysql/` 目录**，除非你已经完全迁移并验证成功。
- 如果你用的是 `systemd`，请确保权限正确，尤其是 `mysql` 用户能读写新路径。
- 建议备份数据库后再操作。

---

### ✅ 总结：关于 MySQL 的路径问题

|问题|回答|
|---|---|
|安装后必须用默认路径吗？|❌ 不是，可通过配置修改关键路径|
|能否把数据目录移到 `/home/user/data/mysql`？|✅ 可以，但需手动迁移 + 修改配置|
|能否用软链接？|✅ 可以，但不推荐用于生产环境|
|是否需要重新编译？|❌ 不需要，纯配置即可|

---

### 💡 附加建议：使用符号链接（快速方案）

如果你不想迁移数据，只是想“看起来”在另一个路径：

```bash
# 1. 创建新目录
sudo mkdir /opt/mysql-data

# 2. 把原目录链接过去
sudo mv /var/lib/mysql /opt/mysql-data/
sudo ln -s /opt/mysql-data/mysql /var/lib/mysql

# 3. 修改 my.cnf 里的 datadir 为 /opt/mysql-data/mysql
```

> ✅ 优点：简单快捷，适合测试环境。  
> ❌ 缺点：路径混乱，不利于运维，不推荐生产使用。

---

### ✅ 最佳实践总结

> **虽然 MySQL 安装后有固定路径，但你完全可以安全地通过配置文件修改数据目录、日志路径等，实现“自定义路径部署”。只要遵循迁移流程，就不会出错。**

---