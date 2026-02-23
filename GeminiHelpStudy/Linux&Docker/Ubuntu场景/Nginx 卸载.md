
### ✅ 一、核心结论

> **卸载 `apt` 安装的 Nginx 有三种方式，根据你的需求选择：**
>
> - `apt remove nginx`：只删除程序，保留配置和数据（推荐用于测试或临时移除）
> - `apt purge nginx`：彻底删除，包括配置文件和数据（推荐用于完全清理）
> - `apt autoremove`：自动清理依赖包（可选）

---

## 📌 二、卸载命令详解

| 命令 | 效果 | 是否保留配置？ |
|------|------|----------------|
| `sudo apt remove nginx` | 卸载 Nginx 程序 | ✅ 保留 `/etc/nginx/`、`/var/www/html/` 等 |
| `sudo apt purge nginx` | 彻底卸载，删除所有相关文件 | ❌ 不保留任何配置和数据 |
| `sudo apt autoremove` | 清理不再需要的依赖包 | ⚠️ 仅清理依赖，不处理 Nginx 本身 |

> 🔥 **强烈建议：如果你要重新安装或彻底重置，使用 `purge`。**

---

### ✅ 三、完整卸载步骤（推荐操作）

```bash
# 步骤 1：停止 Nginx 服务
sudo systemctl stop nginx

# 步骤 2：彻底卸载 Nginx（包括配置和数据）
sudo apt purge nginx

# 步骤 3：清理残留的依赖包（可选但推荐）
sudo apt autoremove

# 步骤 4：确认是否还有残留文件（可选检查）
# 检查关键路径是否还存在：
ls /etc/nginx/
ls /var/www/html/
ls /usr/sbin/nginx
```

> ✅ 这样做后，你将完全移除 Nginx 及其所有痕迹。

---

### 🛠 四、如果想保留配置文件怎么办？

如果你只是想“停用”Nginx，但想以后继续用：

```bash
sudo apt remove nginx
```

> ✅ 这会保留 `/etc/nginx/` 配置文件、`/var/www/html/` 的网页内容等。

你可以之后通过以下命令快速恢复：
```bash
sudo apt install nginx
sudo systemctl start nginx
```

---

### ⚠️ 五、常见问题与注意事项

| 问题 | 解决方法 |
|------|----------|
| 卸载后仍提示 `nginx: command not found`？ | 可能是环境变量问题，运行 `source ~/.bashrc` |
| `/etc/nginx/` 目录还在？ | 是正常的，`remove` 不会删它，除非用 `purge` |
| 想清空网站文件？ | 手动删除：`sudo rm -rf /var/www/html/*` |
| 想删除自定义站点配置？ | 删除：`/etc/nginx/sites-available/xxx` 和软链接 |

---

### ✅ 六、额外：如何查看已安装的 Nginx 包？

```bash
dpkg -l | grep nginx
```

> 会列出所有与 Nginx 相关的包，如：
> ```
> ii  nginx                  1.25.3-0ubuntu1.1    amd64        high performance web server
> ii  nginx-common           1.25.3-0ubuntu1.1    all          small utilities for nginx
> ```

---

### ✅ 总结一句话：

> **如果你想彻底卸载 Nginx，确保执行：**
> ```bash
> sudo apt purge nginx && sudo apt autoremove
> ```
> 这将清除程序、配置、数据和依赖，让你从零开始。

---