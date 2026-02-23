
---

### ⚔️ 实战演练任务：LifeGame 服务器预演

既然你还没有实践，我为你设计了一个 **“LifeGame 部署前夕”** 的模拟训练。

请打开你的 Linux 终端（VMware 虚拟机或 WSL 均可），完成以下 4 个任务。这些操作直接对应你未来在使用 Docker 部署 LifeGame 时必须掌握的技能。

---

#### 任务 A：搭建“基地”目录 (考察 `mkdir`, `cd`, 路径)
**场景**：Docker 部署时，我们需要把宿主机的文件夹挂载到容器里，用于持久化保存数据。

**动作**：
- [x] 在 `/usr/local` 下创建一个名为 `lifegame` 的文件夹。
- [x] 在里面一次性创建三个子文件夹：`data` (放数据库), `logs` (放日志), `config` (放配置)。
> [!tip] 提示_：使用 `mkdir -p`。
- [x] 进入 `logs` 目录，打印当前绝对路径 (`pwd`)。

---

#### 任务 B：模拟启动脚本 (考察 `vi`, `chmod`, `./`)
**场景**：Docker 容器启动时经常因为没有执行权限而报错，我们需要学会赋予权限。

**动作**：
- [x] 在 `lifegame` 目录下，用 `vi` 或 `vim` 创建一个文件 `start.sh`。
- [x] 输入内容：`echo "LifeGame System Starting..."` 并保存退出。
- [x] 尝试运行它：输入 `./start.sh`。
> [!warning] 预期结果_：报错 `Permission denied` (权限不够)。
- [x] **修复它**：使用 `chmod` 命令赋予它执行权限。
- [x] 再次运行，直到看到输出文本。

---

#### 任务 C：日志神探 (考察 `cat`, `grep`, `tail`)
**场景**：后端报错了，日志文件几百兆，你需要在 Linux 上快速定位错误。

**动作**：
- [x] （制造假日志）执行命令：
    ```Bash
    echo "2026-01-23 INFO: System Init" >> app.log
    echo "2026-01-23 INFO: User Login" >> app.log
    echo "2026-01-23 ERROR: NullPointerException in TaskService" >> app.log
    echo "2026-01-23 INFO: Task Created" >> app.log
    ```
    
- [x] **查看**：用 `cat` 看一眼文件内容。
- [x] **搜索**：用 `grep` 找出包含 "ERROR" 的那一行。
- [x] **监控**：打开两个终端窗口。
    - 窗口 1：输入 `tail -f app.log` (实时监控)。
    - 窗口 2：输入 `echo "New Log Entry" >> app.log`。
    - _观察_：窗口 1 是否自动弹出了新的一行？

---

#### 任务 D：清理现场 (考察 `ps`, `kill`, `rm`)
**场景**：旧的服务卡死了，需要强制关闭并清理垃圾文件。

**动作**：
- [x] 启动一个后台睡眠进程：`sleep 1000 &`。
- [x] 查看进程：使用 `ps -ef | grep sleep` 找到它的 **PID** (进程号)。
- [x] 杀掉进程：使用 `kill -9 <PID>`。
- [x] 删除刚才练习用的整个 `lifegame` 文件夹：`rm -rf ...` (小心操作！)

---