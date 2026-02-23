

## ✅ 卸载通过 `apt` 安装的 MySQL

### 1️⃣ 卸载 MySQL 服务及相关包

```bash
sudo apt remove --purge mysql-server mysql-client mysql-common mysql-server-core-* mysql-client-core-*
```

> 💡 说明：
> 
> - `--purge`：不仅删除软件包，还会删除配置文件。
> - `mysql-server`：主服务包。
> - `mysql-client`：客户端工具。
> - `mysql-common`：共享组件。
> - `mysql-server-core-*` / `mysql-client-core-*`：核心依赖包（版本相关）。

---

### 2️⃣ 清理残留的依赖和缓存

```bash
sudo apt autoremove --purge
sudo apt clean
```

> ✅ 这会移除不再需要的依赖包，并清理 APT 缓存。

---

### 3️⃣ 检查是否还有残留文件或目录

有时配置文件或数据目录未被自动删除。建议手动检查：

```bash
ls /etc/mysql/
ls /var/lib/mysql/
```

> ⚠️ 如果你确定不再需要这些数据，请手动删除（谨慎操作！）：

```bash
sudo rm -rf /etc/mysql/
sudo rm -rf /var/lib/mysql/
```

> 📌 特别注意：`/var/lib/mysql/` 是数据库数据默认存储路径，删除后所有数据将永久丢失！

---

### 4️⃣ 验证是否已完全卸载

```bash
dpkg -l | grep mysql
```

> ✅ 如果没有任何输出，说明已彻底卸载。

---

## 🧩 小贴士：不同命名方式

在 Ubuntu 中，常见的 MySQL 相关包名包括：

|包名|作用|
|---|---|
|`mysql-server`|MySQL 服务器|
|`mysql-client`|MySQL 客户端|
|`mariadb-server`|MariaDB（常被误认为 MySQL）|
|`percona-server-server`|Percona MySQL|

> ❗ 建议确认你安装的是哪个数据库。如果是 MariaDB，应使用 `mariadb-server` 作为卸载目标。

---

### ✅ 最终建议

如果你不确定安装了什么，可以用：

```bash
dpkg -l | grep -i mysql
```

来查看系统中所有与 MySQL 相关的包。

---