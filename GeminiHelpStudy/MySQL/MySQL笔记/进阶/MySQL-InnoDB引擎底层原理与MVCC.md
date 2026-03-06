---
title: MySQL-InnoDB引擎底层原理与MVCC
tags:
- MySQL
- InnoDB
- 事务原理
- MVCC
- 进阶
create_time: 2026-03-05
---

## 概述

- **概念**：InnoDB 是 MySQL 默认的、也是最常用的事务型存储引擎。它的核心优势在于支持事务（ACID）、行级锁和外键。
- **目的**：深入理解 InnoDB 的内存/磁盘架构、日志记录机制（Redo/Undo）以及多版本并发控制（MVCC），是解决高并发下死锁、性能瓶颈以及保证数据一致性的基石。

---

## 一、 逻辑存储结构

> [!info] 概念分解 InnoDB 的数据存储并不是直接毫无章法地堆在磁盘上，而是有着严格的层级结构 。

![[MySQL存储结构图.png]]

|**层级**|**英文全称**|**核心作用与执行逻辑**|
|---|---|---|
|**表空间**|Tablespace|InnoDB 存储结构的最高层 。一张表对应一个表空间（`ibd` 文件），存储该表的记录、索引等数据 。|
|**段**|Segment|分为数据段（B+树叶子节点）、索引段（非叶子节点）和回滚段 。由引擎自动管理 。|
|**区**|Extent|表空间的单元结构，每个区大小固定为 1MB 。为了保证页的连续性，InnoDB 每次会申请 4-5 个区 。|
|**页**|Page|**InnoDB 管理存储空间的基本单位**，默认大小 16KB 。一个区包含 64 个连续的页 。|
|**行**|Row|实际的数据行 。除了用户定义的列，还包含隐藏字段（如 `Trx_id`, `Roll_pointer`） 。|

---

## 二、 InnoDB 架构

InnoDB 的架构主要分为**内存结构**和**磁盘结构**两大块 。

### 1. 内存结构 (Memory Structures)

![[MySQL缓冲池.png]]

|**组件名称**|**核心作用与执行逻辑**|
|---|---|
|**Buffer Pool (缓冲池)**|极其重要 。缓存磁盘上经常操作的真实数据（以页为单位） 。执行增删改查时，先操作缓冲池中的数据（若无则从磁盘加载），再以一定频率刷新到磁盘 。|
|**Change Buffer (更改缓冲区)**|针对**非唯一普通索引**的优化 。当执行 DML 语句时，如果==数据页不在 Buffer Pool 中==，不会立刻引发磁盘 IO 读页，而是将变更记录在 Change Buffer 中 。等未来该页被读入内存时，再进行合并（Merge）操作 。|
|**Adaptive Hash Index (自适应哈希索引)**|InnoDB 自动监控表上各索引页的查询频率 。如果发现建立哈希索引可以带来速度提升，会自动构建哈希索引（仅限等值查询） 。**默认开启**|
|**Log Buffer (日志缓冲区)**|用来保存要写入磁盘中的 log 数据（如 redo log, undo log），默认 16MB ，可以通过`innodb_log_buffer_size`控制大小。通过 `innodb_flush_log_at_trx_commit` 参数控制刷盘时机 。|

> 可以通过`adaptive_hash_index`来控制**自适应哈希**的开启与关闭

### 2. 磁盘结构 & 后台线程

![[MySQL磁盘结构图.png]]

- **磁盘结构**：包含系统表空间（System Tablespace）、独立表空间（File-Per-Table Tablespaces）、通用表空间、Undo 表空间（存储 undo log）以及 Redo Log 等 。

|**组件类型**|**英文名称**|**具体功能说明**|**相关参数 / 默认文件名**|
|---|---|---|---|
|**系统表空间**|System Tablespace|InnoDB 数据字典（MySQL 8.0 前）、双写缓冲区（MySQL 8.0.20 前）、Change Buffer 和 Undo Logs（早期版本）的默认存储区域。如果关闭了独立表空间，所有的表数据和索引也会存放在这里。|**参数:** `innodb_data_file_path`<br><br>  <br><br>**文件名:** 默认为 `ibdata1`|
|**独立表空间**|File-Per-Table Tablespaces|用于存放单张表的数据（聚簇索引）和二级索引。每个表对应一个独立的物理文件，优点是 `DROP` 或 `TRUNCATE` 表时可以快速释放磁盘空间给操作系统。|**参数:** `innodb_file_per_table` (MySQL 5.6+ 默认开启)<br><br>  <br><br>**文件名:** `表名.ibd`|
|**通用表空间**|General Tablespaces|共享的表空间，通过 `CREATE TABLESPACE` 语法手动创建。可以将多张表的数据存储在同一个通用表空间文件中，相比独立表空间能减少元数据内存开销，但比系统表空间更灵活。|**语法:** `CREATE TABLESPACE`<br><br>  <br><br>**文件名:** 自定义，通常以 `.ibd` 结尾|
|**撤销表空间**|Undo Tablespaces|专门用于存储 Undo Log（撤销日志）。记录数据修改前的旧版本状态，用于实现 **事务回滚（Rollback）** 和 **多版本并发控制（MVCC）**，保证读写不冲突。|**参数:** `innodb_undo_directory`<br><br>  <br><br>**文件名:** `undo_001`, `undo_002` 等 (MySQL 8.0 默认创建两个独立文件)|
|**重做日志**|Redo Log|Write-Ahead Logging (WAL) 机制的核心。记录了对数据页的物理修改。主要用于 **崩溃恢复（Crash Recovery）**，确保事务的持久性（ACID中的D）。|**参数:** `innodb_redo_log_capacity` (8.0.30+), `innodb_log_file_size` (旧版)<br><br>  <br><br>**文件名:** `ib_logfile0`, `ib_logfile1` 或 `#innodb_redo` 目录|
|**临时表空间**|Temporary Tablespaces|分为会话临时表空间和全局临时表空间。用于存储用户创建的临时表（`CREATE TEMPORARY TABLE`）以及优化器在执行复杂查询（如 `GROUP BY`, `ORDER BY`）时创建的内部临时表。|**参数:** `innodb_temp_data_file_path`<br><br>  <br><br>**文件名:** 全局默认为 `ibtmp1`|
|**双写缓冲区文件**|Doublewrite Buffer Files|MySQL 8.0.20 引入的独立文件。在数据页真正写入数据文件前，先顺序写入此处。用于防止系统崩溃导致的 **部分写失效（Partial Page Write）** 问题，保证数据页的完整性。|**参数:** `innodb_doublewrite_dir`, `innodb_doublewrite`<br><br>  <br><br>**文件名:** 以 `.dblwr` 结尾的文件|

> [!warning] 💡核心版本差异提示 (MySQL 8.0 的重要演进)
> - **Undo Log 的剥离：** 在早期版本中，Undo log 默认混在系统表空间 (`ibdata1`) 中，导致大事务会撑爆系统表空间且难以收缩。MySQL 8.0 之后，Undo log 强制使用独立的 Undo 表空间，并支持自动截断收缩（Truncate）。  
> - **Redo Log 的动态扩容：** MySQL 8.0.30 之后，废弃了旧的 `innodb_log_files_in_group` 等参数，引入了 `#innodb_redo` 文件夹和 `innodb_redo_log_capacity`，Redo Log 变得可以更加灵活地动态调整大小。

- **后台线程**：负责将内存结构中的数据异步刷新到磁盘，保证数据的一致性 。

|**线程名称**|**英文名称**|**具体功能说明**|**相关参数**|
|---|---|---|---|
|**主线程**|Master Thread|InnoDB 的核心后台线程，具有最高的调度优先级。它负责协调各个后台线程，早期版本中包揽了脏页刷新、合并插入缓冲（Change Buffer）、回收 Undo 页等几乎所有工作。随着版本演进，为了提高并发性能，它的许多核心工作（如 Purge 和 Flush）已经被剥离给了其他专用线程，现在主要起统筹和兜底作用。|无特定开关参数，系统核心线程|
|**输入输出线程**|I/O Threads|负责处理 InnoDB 中大量的异步 I/O (AIO) 请求，极大提高了数据库的读写性能。主要分为四种类型的 IO 线程：<br><br>  <br><br>1. **Read thread**: 负责数据页的异步读取操作。<br><br>  <br><br>2. **Write thread**: 负责数据页的异步写回操作。<br><br>  <br><br>3. **Log thread**: 负责将 Redo Log 缓冲刷新到磁盘。<br><br>  <br><br>4. **Insert Buffer thread**: 负责合并 Change Buffer。|**读线程数:** `innodb_read_io_threads` (默认 4)<br><br>  <br><br>**写线程数:** `innodb_write_io_threads` (默认 4)|
|**清理线程**|Purge Threads|专门负责回收已经分配并使用的 Undo Log，以及清理那些带有“删除标记 (delete-marked)”的数据行。事务提交后，其对应的 Undo Log 可能不再需要（如果没有其他事务的 MVCC 依赖它），Purge 线程就会将其释放。|**参数:** `innodb_purge_threads` (MySQL 5.6+ 默认开启并设为 4，最大 32)|
|**页面清理线程**|Page Cleaner Threads|专门负责脏页（修改过但尚未写入磁盘的数据页）的刷新操作。将脏页从缓冲池 (Buffer Pool) 刷新到磁盘的数据文件中。这是从 Master Thread 中剥离出来的重要功能，极大减轻了 Master Thread 的负担，减少了系统因为刷脏页导致的性能抖动。|**参数:** `innodb_page_cleaners` (MySQL 5.7+ 引入，默认值为 4，通常建议与 `innodb_buffer_pool_instances` 数量一致)|
|**错误监控线程**|Error Monitor Thread|负责监控数据库运行时的错误情况，一旦发现某些致命错误或者由于资源不足导致的挂起，它会尝试进行恢复或者将数据库安全地宕机，以保护数据的一致性。|系统内部管理|
|**锁监控线程**|Lock Monitor Thread|负责监控死锁和长事务。当开启相应的监控后，它会定期扫描 InnoDB 的事务锁表，如果检测到死锁（Deadlock），会根据策略（通常是回滚 `undo` 代价最小的事务）自动介入打破死锁。|**参数:** `innodb_deadlock_detect` (默认开启)|

> [!warning] 💡 核心演进提示 (性能优化的主线)
> 在理解 MySQL 后台线程时，有一条非常清晰的历史演进主线——**“为主线程减负，走向高度并发”**：
> - **单核时代的 Master Thread：** 在 MySQL 5.5 及更早的版本中，`Master Thread` 是个“大管家”，既要刷脏页，又要清理 Undo log，还要合并 Change Buffer。在高并发读写时，主线程迅速成为单点瓶颈。 
> - **多线程剥离（5.6 ~ 8.0）：** * MySQL 5.5 增加了多个 IO 线程。
>     - MySQL 5.6 独立出了 `Purge Thread`，主线程不再负责回收 Undo log。
>     - MySQL 5.7 独立出了 `Page Cleaner Thread`，主线程不再负责刷脏页。 到现在，InnoDB 已经是一个高度细分、各司其职的现代化多线程架构，能更好地榨干多核 CPU 的性能。

---

## 三、 事务原理 (底层日志)

> [!tip] 核心考点 事务的 ACID 特性中，**原子性（A）、一致性（C）、持久性（D）** 是由底层日志（Redo/Undo）保证的；而**隔离性（I）** 是由锁机制和 MVCC 共同保证的 。

### 1. Redo Log (重做日志) -> 保证持久性 (D)

- **概念**：记录的是数据页的**物理修改**（例如：在某数据页的某偏移量处修改了什么值） 。
- **执行逻辑 (WAL机制)**：
    1. 事务修改数据时，先修改 Buffer Pool 中的内存页（此时变为脏页） 。
    2. 同时将物理修改记录到 Redo Log Buffer，并在事务提交时**顺序追加写**到磁盘上的 Redo Log 文件中 。
    3. 如果数据库宕机导致脏页未刷入磁盘，重启时 InnoDB 会读取 Redo Log 进行重做，恢复数据 。


### 2. Undo Log (回滚日志) -> 保证原子性 (A)

- **概念**：记录的是数据的**逻辑修改** 。当事务需要回滚时，通过 Undo Log 执行相反的操作来撤销修改 。
- **执行逻辑**：
    - 如果执行了 `INSERT`，Undo Log 中会记录一条对应的 `DELETE` 语句 。
    - 如果执行了 `UPDATE`，Undo Log 中会记录一条将数据改回原样的 `UPDATE` 语句 。
- **额外作用**：除了回滚，Undo Log 也是实现 **MVCC** 的关键 。

---

## 四、 MVCC (多版本并发控制)

- **概念**：Multi-Version Concurrency Control。指维护一个数据的多个版本，使得读写操作没有冲突 。

### 1. 前置概念：当前读 vs 快照读

- **当前读**：读取的是记录的**最新版本**，读取时还要保证其他并发事务不能修改当前记录，会对读取的记录进行加锁 （如 `FOR UPDATE`, `LOCK IN SHARE MODE`, `UPDATE`, `DELETE`）。
- **快照读**：简单的 `SELECT`（不加锁）就是快照读，读取的是记录的**可见版本**（可能是历史版本），**不加锁，非阻塞** 。

### 2. MVCC 实现的三大基石

#### ① 隐藏字段

InnoDB 在每行数据后隐式添加了三个字段 ：

- `DB_TRX_ID`：最近修改（或插入）该行数据的**事务 ID** 。
- `DB_ROLL_PTR`：**回滚指针**，指向该行数据上一个版本的 Undo Log 记录 。
- `DB_ROW_ID`：隐藏主键（如果表没有主键且没有合适的唯一索引，才会生成） 。

#### ② Undo Log 版本链

- 多个事务对同一行数据进行修改时，旧版本的数据会被记录在 Undo Log 中 。
- 这些旧版本通过行记录中的 `DB_ROLL_PTR`（回滚指针）连接起来，形成一条单向链表，这就是**版本链** 。链表头部是最新数据，尾部是最老数据 。

#### ③ ReadView (读视图)

- **概念**：快照读 SQL 执行时由 InnoDB 提取的一个系统当前状态的“快照” 。它决定了当前事务能看到版本链中的哪个版本 。
- **核心字段** ：
    - `m_ids`：当前系统中活跃的（未提交的）事务 ID 集合 。
    - `min_trx_id`：`m_ids` 中最小的事务 ID 。
    - `max_trx_id`：系统分配给下一个事务的 ID 。
    - `creator_trx_id`：生成该 ReadView 的当前事务 ID 。

### 3. RC 与 RR 级别的本质区别

MVCC 到底怎么判断哪个版本可见？依据就是 ReadView 的匹配规则 。不同的隔离级别，**生成 ReadView 的时机不同**：

|**隔离级别**|**生成 ReadView 的时机**|**底层效果差异**|
|---|---|---|
|**RC (读已提交)**|**每一次**执行快照读（`SELECT`）时，都会生成一个全新的 ReadView 。|同一个事务中两次查询可能得到不同结果（不可重复读） 。|
|**RR (可重复读)**|仅在事务中**第一次**执行快照读时生成 ReadView，后续查询复用该 ReadView 。|保证了同一个事务中无论查询多少次，看到的数据快照都是一致的（可重复读） 。|

> 更详细的原理内容 -> [[MySQL锁与事务#机制解析]]

---

## 💡 避坑与进阶指导

### 实际开发常见坑

1. **长事务导致 Undo Log 暴涨**：如果系统中存在执行时间极长的事务，会导致该事务关联的 Undo Log 版本链迟迟无法被 Purge 线程清理。这不仅严重占用磁盘空间，还会导致查询遍历版本链时性能极度劣化。
2. **以为快照读能读取最新数据**：在 RR 隔离级别下，如果你在事务 A 中先 `SELECT` 了一次，此时事务 B 修改了数据并提交。事务 A 再次 `SELECT`（快照读）依然是旧数据。如果业务强依赖最新数据计算，必须使用 `SELECT ... FOR UPDATE`（当前读）强制获取最新数据并加锁。

### 面试高频问题

- 面试题：什么是 WAL 机制？为什么不直接把修改的数据写入磁盘，而是先写 Redo Log？（答：顺序 IO 远快于随机 IO）
- 面试题：解释一下 MySQL 中的脏页和刷盘机制？
- 面试题：简述 Undo Log 和 Redo Log 的区别是什么？
- 面试题：什么是 MVCC？它解决了什么问题？（答：读写无阻塞并发）
- 面试题：详细描述一下 MVCC 的底层实现原理。（必考题：隐藏字段 + Undo Log 版本链 + ReadView）
- 面试题：RC 和 RR 隔离级别在 MVCC 的实现上有什么核心区别？（答：ReadView 生成的时机不同）