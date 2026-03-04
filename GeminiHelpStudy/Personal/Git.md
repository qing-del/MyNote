---
title: Git 笔记同步与 SSH 配置指北
tags:
- Git
- 效率工具
- Obsidian
- 同步
create_time: 2026-03-04
---

## 概述

- **概念**：Git 是一个开源的**分布式版本控制系统**，可以高效地处理从很小到非常大的项目版本管理。
- **目的**：在多设备（如电脑与移动端平板）之间实现文本（笔记、代码）的无缝同步、备份以及版本回溯。
- **优缺点**：
    - **优点**：数据完全掌握在自己手里；支持强大的分支和历史回退功能；通过 SSH 传输速度快且安全。
    - **缺点**：移动端（如 Android）原生不支持，需要借助 Termux 等终端模拟器环境；有一定的命令行学习成本；处理非纯文本（大体积图片、视频）时容易导致仓库臃肿。

> [!warning] 移动端避坑注意
> 在 Termux 中 clone 仓库前，**必须**先运行 `termux-setup-storage` 授权文件访问，并将仓库 Clone 到 `~/storage/shared/` 下的某个目录（例如 `~/storage/shared/Documents/Obsidian`），否则 Obsidian App 无法读取到处于 Termux 沙盒内的笔记文件。


---

## 核心配置：Termux 环境与 OpenSSH

为了安全拉取 Private（私有）仓库并且免输密码，我们需要配置 SSH 密钥对。以下操作在 Termux 和电脑端均适用。

### 1. 生成并配置 SSH Key
```bash
# 1. 生成新的 Ed25519 类型的 SSH 密钥（比 RSA 更安全高效）
ssh-keygen -t ed25519 -C "你的邮箱@example.com"

# 遇到提示一直回车即可，默认会保存在 ~/.ssh/id_ed25519

# 2. 启动 ssh-agent 并在后台运行
eval "$(ssh-agent -s)"

# 3. 将私钥添加到 ssh-agent
ssh-add ~/.ssh/id_ed25519

# 4. 查看并复制公钥内容
cat ~/.ssh/id_ed25519.pub
````
> [!warning] 安卓的文件管理器根目录：`/storage/emulated/0`

### 2. 关联远端与测试

将上一步 `cat` 打印出来的公钥内容（以 `ssh-ed25519` 开头的那一长串）复制，添加到 GitHub / GitLab / Gitee 等代码托管平台的 `SSH Keys` 设置中。

```Bash
# 测试 SSH 连接是否成功（以 GitHub 为例）
ssh -T git@github.com

# 如果看到类似于 "Hi username! You've successfully authenticated..." 说明配置成功！
```

---

## Git 核心基础指令

|**指令分类**|**具体命令**|**作用说明**|
|---|---|---|
|**仓库初始化**|`git clone git@github.com:用户名/仓库名.git`|**核心**：通过 SSH 地址将远端的私有仓库克隆到本地。|
||`git init`|在当前目录初始化一个新的本地 Git 仓库。|
|**状态查看**|`git status`|查看当前工作区和暂存区的状态（哪些文件被修改了）。|
||`git log`|查看提交历史记录。|
|**提交代码**|`git add .`|将当前目录下所有修改过的文件添加到暂存区。|
||`git commit -m "feat: 更新了xx笔记"`|将暂存区的内容提交到本地仓库，并附带说明信息。|
|**远程同步**|`git pull origin main`|**拉取**：从远程 `main` 分支拉取最新更改并与本地合并（写笔记前必做）。|
||`git push origin main`|**推送**：将本地的提交推送到远程 `main` 分支（写完笔记后必做）。|

---

## 💡 笔记同步流：最佳实践
为了避免笔记在两台设备上产生冲突（Merge Conflict），建议养成严格的**单向闭环**习惯：

### 场景一：出门前去上课（电脑转移到平板）

1. **电脑端**：写完笔记后，执行 `git add .` -> `git commit -m "电脑端更新"` -> `git push`。
2. **平板端**：打开 Termux，进入笔记目录，执行 `git pull`，获取最新笔记，然后打开 Obsidian 开始查阅或记录。

### 场景二：下课回宿舍（平板转移到电脑）

1. **平板端**：记录完毕，打开 Termux，执行 `git add .` -> `git commit -m "平板端更新"` -> `git push`。
2. **电脑端**：坐回电脑前，先执行 `git pull` 同步平板上的灵感，再继续使用电脑编辑。

> [!info] 进阶技巧：避免反复输入繁琐命令
> 可以在 Termux 的 `~/.bashrc` 或 `~/.zshrc` 中配置 Alias（别名），例如：
> `alias gsync="git add . && git commit -m 'Auto sync' && git pull && git push"`
> 以后在 Termux 里只要输入 `gsync` 就能一键完成拉取和推送的闭环。
