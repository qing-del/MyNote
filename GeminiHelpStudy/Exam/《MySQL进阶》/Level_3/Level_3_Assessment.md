# 数据库系统原理核心考核 - Level 3

**考核人：[你的学号/姓名]**  
**命题人：资深数据库系统教授**  
**考察范围：MVCC 快照读版本选择、行级锁的加锁边界、Gap Lock / Next-Key Lock、死锁分析**

---

## 教授前言

欢迎进入 Level 3。

这一关考察的不是你背出了多少锁的名词，而是你能否在一个具体的索引结构和时间线场景中，**精确地画出每把锁的加锁区间边界，并沿着事务的等待链路判断是否会形成死锁**。

我特意在两道场景题中分别设置了：
- **场景一**：辅助索引加锁时回溯聚簇索引的联动机制 + MVCC 快照读版本寻址
- **场景二**：Gap Lock 边界的精确判断 + 经典死锁构造（但结论未必是你以为的那个）

我强烈建议你将推演写完之后，在**本地 MySQL 终端开两个 Session**，实际执行 `BEGIN` + 具体 SQL，然后在第三个 Session 运行：

```sql
SELECT ENGINE_LOCK_ID, ENGINE_TRANSACTION_ID, LOCK_TYPE, LOCK_MODE, LOCK_STATUS, LOCK_DATA
FROM performance_schema.data_locks;
```

用真实结果验证你的推演是否正确。如果真实结果和你的推演不一致，请记录下来并分析原因。

---

## 基础表结构

以下两道场景题共用同一张表：

```sql
CREATE TABLE `orders` (
  `id`           BIGINT      NOT NULL AUTO_INCREMENT  COMMENT '主键 / 聚簇索引',
  `user_id`      BIGINT      NOT NULL                 COMMENT '用户ID',
  `order_status` TINYINT     NOT NULL DEFAULT 1       COMMENT '订单状态 1-5',
  `amount`       DECIMAL(10,2) NOT NULL               COMMENT '金额',
  PRIMARY KEY (`id`),
  INDEX `idx_user_id` (`user_id`)
) ENGINE=InnoDB;
```

表中现有以下 **7 行数据**（写入后无其他修改，表已稳定）：

| id | user_id | order_status | amount |
|:---:|:---:|:---:|:---:|
| 1  | 10 | 1 | 100.00 |
| 3  | 20 | 2 | 200.00 |
| 5  | 20 | 3 | 150.00 |
| 8  | 30 | 4 | 300.00 |
| 10 | 30 | 1 | 250.00 |
| 15 | 40 | 2 | 180.00 |
| 20 | 50 | 5 | 500.00 |

> **注意**：`id` 不连续，这是关键条件。`idx_user_id` 是一个非唯一的二级索引。

---

## 场景一：辅助索引加锁 + MVCC 版本链追踪

**隔离级别：`REPEATABLE READ`（默认）**

### 时间线

| 时刻 | 事务 A（Session 1）| 事务 B（Session 2）|
|:---:|:---|:---|
| T1 | `BEGIN;` | |
| T2 | `SELECT amount FROM orders WHERE id = 5;` | |
| T3 | | `BEGIN;` |
| T4 | | `UPDATE orders SET amount = 999.00 WHERE id = 5;` |
| T5 | | `COMMIT;` |
| T6 | `UPDATE orders SET order_status = 9 WHERE user_id = 20;` | |
| T7 | `SELECT amount FROM orders WHERE id = 5;` | |
| T8 | `COMMIT;` | |

### 你的任务

**Part A：MVCC 快照读**

1. **T2 时刻**，事务 A 执行快照读，读到的 `amount` 是多少？此时 Read View 是如何生成的？（假设当前系统下一个事务 id 将被分配为 100，事务 A 被分配的 trx_id 暂定为 99，且此时无其他活跃事务）

2. **T7 时刻**，事务 A 再次执行快照读，按照 RR 隔离级别，读到的 `amount` 是多少？请结合版本链推演：执行器会看到 `id=5` 这行的最新物理版本（由事务 B 写入，`amount=999.00`），它是如何判断这个版本"不可见"并回溯找到正确版本的？

**Part B：当前读与加锁分析**

3. **T6 时刻**，事务 A 执行 `UPDATE orders SET order_status = 9 WHERE user_id = 20;`。这是一个**当前读（Current Read）**，会触发加锁。请回答：

   a. MySQL 会优先使用哪个索引来执行这条 UPDATE？走的是 `idx_user_id` 还是聚簇索引？

   b. 在 `idx_user_id` 这棵二级索引树上，`user_id = 20` 对应的叶子节点区间是什么？InnoDB 会在该二级索引上加什么类型的锁，加锁区间是多少？

   c. 加完二级索引上的锁之后，InnoDB 还会做什么额外的操作（提示：回想辅助索引加锁时的"回表"联动）？在**聚簇索引**上，又会加哪些锁？

   d. 综合你的加锁分析：如果此时事务 C 在 T6 之后执行 `SELECT * FROM orders WHERE id = 3 FOR UPDATE;`，会被阻塞吗？如果执行 `INSERT INTO orders (id, user_id, order_status, amount) VALUES (4, 25, 1, 100.00);`，会被阻塞吗？请分别给出结论和底层理由。

---

## 场景二：Gap Lock 边界精确判断 + 死锁分析

**隔离级别：`REPEATABLE READ`（默认）**

### 时间线

| 时刻 | 事务 X（Session 1）| 事务 Y（Session 2）|
|:---:|:---|:---|
| T1 | `BEGIN;` | `BEGIN;` |
| T2 | `SELECT * FROM orders WHERE id = 7 FOR UPDATE;` | |
| T3 | | `SELECT * FROM orders WHERE id = 12 FOR UPDATE;` |
| T4 | `INSERT INTO orders (id, user_id, order_status, amount) VALUES (11, 35, 1, 120.00);` | |
| T5 | | `INSERT INTO orders (id, user_id, order_status, amount) VALUES (9, 35, 1, 120.00);` |

### 你的任务

**Part A：加锁区间推演**

1. **T2 时刻**，事务 X 执行 `SELECT * FROM orders WHERE id = 7 FOR UPDATE`。注意：`id=7` 这行**不存在**于表中。请问：
   - InnoDB 会加什么类型的锁？
   - 加锁区间是什么（请用开区间 / 闭区间精确表示）？
   - 为什么是这个区间，而不是别的？（请从 B+ 树叶子节点的物理结构来解释）

2. **T3 时刻**，事务 Y 执行 `SELECT * FROM orders WHERE id = 12 FOR UPDATE`。同样，`id=12` 也**不存在**。请问：
   - 加锁区间是什么？
   - 和 T2 的锁区间相比，两者有没有重叠？

**Part B：死锁判断**

3. **T4 时刻**，事务 X 尝试插入 `id=11`。
   - 这个插入操作是否会被事务 Y 持有的锁阻塞？为什么？

4. **T5 时刻**，事务 Y 尝试插入 `id=9`。
   - 这个插入操作是否会被事务 X 持有的锁阻塞？为什么？

5. **综合 T4 和 T5**：此时是否形成了死锁？如果形成了，请画出事务等待图（Wait-for Graph），并说明 InnoDB 的死锁检测器会如何处理。如果没有形成，请说明理由。

---

## 作答要求

1. 请将完整的推演过程和结论写入 `Level_3_MyAnswer.md`。
2. 对于 **场景二的 Part B**，强烈建议你在本地 MySQL 中实际执行并观察 `performance_schema.data_locks` 的加锁情况，将截图或输出内容附在答卷的对应部分下方。
3. 完成后告知我，我将对你的每个推演步骤进行逐项审核。

**现在，请开始你的作答。**
