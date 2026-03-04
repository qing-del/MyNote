---
title: MySQL进阶：SQL优化实战
tags:
- MySQL
- SQL优化
- 进阶
create_time: 2026-03-02
---

## 概述
- **概念**：在掌握了索引的使用规则后，日常开发中还需要针对具体的 SQL 语句（如插入、排序、分组、分页等）进行针对性的性能调优 。
- **目的**：通过合理的 SQL 编写和索引配合，降低数据库的 CPU、内存和磁盘 I/O 消耗，提升系统整体的并发能力和响应速度。

---

## 一、 插入数据优化 (Insert)

> [!info] 分析思路 频繁的单条插入会导致频繁的事务提交和网络请求。优化核心在于**批量化**和**顺序化** 。

### 核心优化策略

|**优化方式**|**执行逻辑说明**|**适用场景**|
|---|---|---|
|**批量插入**|将多条 `INSERT` 语句合并为一条，减少网络开销和解析时间。|一次性插入多条（建议 500-1000 条左右）。|
|**手动提交事务**|避免 MySQL 每执行一条插入就自动开启和提交一次事务。|循环执行多条 `INSERT` 语句时。|
|**主键顺序插入**|按照主键的顺序进行插入，避免底层 B+ 树频繁产生页分裂。|绝大场景下的数据写入。|

- **示例SQL**：

```SQL
-- ❌ 慢：单条单条插入
INSERT INTO user(id, name) VALUES (1, 'Tom');
INSERT INTO user(id, name) VALUES (2, 'Jerry');

-- ✅ 快：批量插入
INSERT INTO user(id, name) VALUES (1, 'Tom'), (2, 'Jerry'), (3, 'Spike');

-- ✅ 快：手动提交事务
START TRANSACTION;
INSERT INTO user(id, name) VALUES (1, 'Tom');
INSERT INTO user(id, name) VALUES (2, 'Jerry');
COMMIT;
```

> [!tip] 大批量数据加载 (`LOAD DATA`)
> 如果一次性需要插入几百万条数据，使用 `INSERT` 性能极低，推荐使用 MySQL 提供的 `LOAD DATA` 指令，直接读取本地文件加载到数据库。
> ```txt
> 1,jacolp,123456,Jaoclp,2006-06-06,1
> 2,yuke,369369,YuKe,2005-05-05,2
> ...
> ```
> - 对应表结构
> 
> |id|username|password|name|birthday|sex|
> |---|---|---|---|---|---|
> |1|jacolp|123456|Jacolp|2006-06-06|1|
> 
> - 执行流程
> ```sql
> -- 客户端连接服务器时，加上 --local-infile
> mysql --local-infile -u root -p
> 
> -- 设置全局参数local_infile为1，开启从本地加载文件导入数据开关
> set global local_infile=1;
> 
> -- 执行load命令将准备好的数据，加载到表结构中
> load data local infile '[文件完整路径]' into table '[表名]' fields terminated by ',' line terminated by '\n';
> -- 第一个','可以看作是'[字段分隔符]'，第二个'\n'可以看作是'[数据行分隔符]'
> ```

---

## 二、 主键优化

> [!info] 核心概念 在 InnoDB 存储引擎中，表数据都是根据主键顺序组织存放的，这种存储方式的表称为**索引组织表 (Index Organized Table, IOT)** 。

### 页分裂与页合并

- **页分裂**：主键乱序插入时，如果当前数据页已满，为了保持 B+ 树的有序性，MySQL 会申请一个新的数据页，并将满页的数据**挪一部分**过去，改变指针指向。这个过程极其消耗性能 。
> 相当于 `B+Tree` 上的节点分裂，会挪动原本页节点一半的数据到新的页上面
- **页合并**：当删除一条记录时，数据并不会立刻物理删除，而是打上标记。当页中删除的记录达到**阈值**（**默认为 50%**）时，InnoDB 会尝试将该页与相邻的兄弟页进行合并，以优化空间利用率 。

### 主键设计原则

1. 满足业务需求的情况下，尽量**降低主键的长度**（主键越长，二级索引占用的空间越大）。
2. 插入数据时，尽量选择**顺序插入**，选择使用 `AUTO_INCREMENT` 自增主键。
3. 尽量**不要使用 UUID** 或其他自然主键（如身份证号）作为主键（长度过长、且无序会导致频繁的页分裂）。
4. 业务操作时，避免对主键进行修改。

---

## 三、 ORDER BY 优化

- **作用场景**：针对包含排序逻辑的查询进行调优 。

### Extra 字段解析

|**Extra 信息**|**含义简述**|**优化目标**|
|---|---|---|
|`Using filesort`|通过表索引或全表扫描读取满足条件的数据，然后在**排序缓冲区 (sort buffer)** 中完成排序。|**需要消除**，性能较低。|
|`Using index`|通过有序索引顺序扫描直接返回有序数据，不需要额外排序。|**终极目标**，性能极高。|

> 为什么`Using filesort`会性能低？ -> [[MySQL索引原理#Extra 列]]的`Using filesort`节

### 优化规则

- 根据排序字段建立合适的索引。
- 联合索引在做排序时，同样遵循**最左前缀法则**。如果排序涉及多个字段，建议建立联合索引，且关注创建索引时的**排序方式（ASC/DESC）**。
- **示例代码**：

```SQL
-- 假设联合索引 idx_user_age_phone 创建为: (age ASC, phone DESC)
CREATE INDEX idx_user_age_phone ON user(age ASC, phone DESC);

-- ✅ 走索引 (Using index)：完全匹配索引的字段和升降序
EXPLAIN SELECT id, age, phone FROM user ORDER BY age ASC, phone DESC;

-- ❌ 文件排序 (Using filesort)：顺序颠倒或升降序不一致
EXPLAIN SELECT id, age, phone FROM user ORDER BY phone DESC, age ASC;
```

> [!info] 💡思考
> - `order by age asc, phone desc`为什么不是扫描到`age`之后用下一个`age`进行`Backward index scan`？
> - `order by phone asc, age`中涵盖`Using index`，是不是可以说明先用链表组装组装结果，在组装完之后再进行别的操作？
> 	- 就是用每个用age排序好的phone都是升序的，那么就是扫描不同的age值下的phone时，就用链表来组装结果
> 	- 根据phone组装链表完毕之后，就进行`sort(a, b, a.phone==b.phone? return a.age < b.age; return a.phone < b.phone;)`

> [!warning] 警告
> 如果不可避免出现`Using filesort`，并且数据量过大，就会出现需要使用磁盘文件来进行IO操作；
> 详见 -> [[MySQL索引原理#Extra 列]]的`Using filesort`节

---

## 四、 GROUP BY 优化

- **作用场景**：分组统计查询的性能调优 。
- **核心逻辑**：在没有索引的情况下，`GROUP BY` 会产生临时表（`Using temporary`），性能消耗极大。优化关键依然是**利用索引**。
- **使用规则**：
    - 分组操作可以通过建立和利用索引来大幅提升效率。
    - 同样满足联合索引的最左前缀法则。

> [!tip] 案例
> 例如存在`(profession, age, status)`联合索引
> `select age,count(*) where profession='软件工程' group by age;`
> 其实这条SQL语句也是`Using index`，因为满足了**最左前缀法则**

---

## 五、 LIMIT 优化 (深度分页)

- **概念**：在执行类似 `LIMIT 9000000, 10` 的深度分页时，MySQL 需要扫描前 9000010 条记录，然后丢弃前 9000000 条，返回最后的 10 条，这会导致全表扫描式的 I/O 灾难 。

> [!warning] 优化方案：覆盖索引 + 子查询/表连接
> MySQL 并不支持在 `IN` 语句中直接使用带 `LIMIT` 的子查询。标准的解决思路是：**先通过覆盖索引（主键查询）快速定位出需要的数据的主键 id，然后再用这批 id 去和原表做 INNER JOIN（内连接）**。

- **示例SQL**：

```SQL
-- ❌ 慢查询：直接深度分页回表
SELECT * FROM user LIMIT 9000000, 10;

-- ✅ 快查询：覆盖索引 + 表连接
SELECT t.* FROM user t
INNER JOIN (SELECT id FROM user ORDER BY id LIMIT 9000000, 10) p
ON t.id = p.id;
```

> [!info] 为什么会快？
> 除了**索引覆盖**，其实还可以联想到用`IO`的开销，数据量大小，SQL优化器的暗中操作
> 详见 -> [[MySQL的limit关键词优化原理]]

---

## 六、 COUNT 优化

- **概念**：`COUNT()` 是一个聚合函数，对于返回的结果集，一行一行地判断，如果参数不是 `NULL`，累计值就加 1 。

### COUNT 变体性能对比

|**语法**|**执行逻辑说明**|**性能评级**|
|---|---|---|
|`count(字段)`|如果没有 `NOT NULL` 约束，InnoDB 需要逐行取出字段值判断是否为 `NULL`。|最低|
|`count(主键 id)`|InnoDB 会遍历整张表，把每行的主键 id 取出返回给服务层，服务层拿到主键后直接按行累加（主键不可能为 `NULL`）。|较低|
|`count(1)`|InnoDB 遍历整张表，但不取值。服务层对于返回的每一行，放一个数字“1”进去，直接累加。|高|
|`count(*)`|InnoDB 并不会把全部字段取出来，而是专门做了优化，不取值，服务层直接按行累加。|**最高**|

> [!tip] 结论 按照效率排序：`count(字段)` < `count(主键 id)` < `count(1)` ≈ `count(*)`。所以在开发中，**强烈建议直接使用 `count(*)`** 。

> - 当数据量达到上亿的时候，尽管是再怎么优化，都会很慢
> 	- ①创建一张表来保存这张表的总数据量
> 	- ②加入`Redis`来处理

---

## 七、 UPDATE 优化

> [!warning] 避免行锁升级为表锁 InnoDB 引擎的行锁是**针对索引加的锁**，而不是针对记录加的锁。如果 `UPDATE` 的条件字段没有索引，或者索引失效，**行锁将会升级为表锁**，严重影响系统的并发写性能 。

- **示例SQL**：

```SQL
-- 假设 name 字段没有建立任何索引

-- ❌ 危险操作：当前事务未提交前，整张表都会被锁住（表锁），其他客户端的任何 UPDATE 都会被阻塞
UPDATE user SET status = '1' WHERE name = 'Tom';

-- ✅ 安全操作：给 name 添加索引后，只会锁住 name='Tom' 的这一行（行锁），不影响其他记录的更新
```

---

💡 避坑与进阶指导

### 实际开发常见坑

1. **批量插入未控制单批次大小**：一次性往 MySQL 塞入十万条数据会导致数据包过大（超出 `max_allowed_packet` 限制）并阻塞数据库，必须分批次处理（如每批 500-1000 条）。
2. **深分页直接甩给 MySQL**：C 端接口如果允许用户无限制往后翻页，极易被爬虫或恶意请求打挂数据库。需在业务侧限制最大翻页深度，或强制配合前置条件过滤。
3. **无索引字段执行并发 UPDATE**：业务线突然发生大面积锁等待超时（Deadlock/Timeout），排查后大概率是因为 `WHERE` 后的条件没走索引，触发了整表锁定。

### 性能注意点

1. **UUID做主键的灾难**：不仅占用存储大，在海量数据写入时，无序的 UUID 会导致 B+ 树页分裂极其严重，引发插入性能断崖式下跌。
2. **大表 COUNT 估算**：如果只是为了在前端展示“大约有 1000 万+ 条记录”，不要在实时核心接口中使用 `SELECT COUNT(*)` 扫大表，应该考虑缓存（Redis 计数器）或者利用系统视图拉取估算值。

### 面试高频问题

- 面试题：什么是深分页问题？如何解决 MySQL 的深分页慢查询？
- 面试题：`count(*)`, `count(1)`, `count(id)` 有什么区别？哪个性能最好？
- 面试题：什么情况下 InnoDB 的行锁会升级为表锁？这会对系统造成什么影响？
- 面试题：为什么要建议大家使用自增 ID 作为主键，而不是 UUID？请从底层数据结构的角度（页分裂）分析。