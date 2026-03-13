### 1. 成本常数表概览

MySQL 执行语句分为 Server 层和存储引擎层，因此成本常数也分两表存储 。

|**表名**|**存储内容描述**|**包含的列**|
|---|---|---|
|**`mysql.server_cost`**|存储在 Server 层进行的操作（如连接管理、查询优化、临时表创建等）对应的成本常数 。|`cost_name`, `cost_value`, `last_update`, `comment`|
|**`mysql.engine_cost`**|存储依赖于具体存储引擎的操作（如读取磁盘页、读取内存页）对应的成本常数 。|`engine_name`, `device_type`, `cost_name`, `cost_value`, `last_update`, `comment`|

---

### 2. `mysql.server_cost` 详细常数

该表中的 `cost_value` 若为 `NULL`，则表示优化器使用该项的**默认值** 。

|**成本常数名称**|**默认值**|**描述与影响**|
|---|---|---|
|**`disk_temptable_create_cost`**|40.0|创建基于**磁盘**的临时表的成本。增大此值会让优化器尽量避免使用磁盘临时表 。|
|**`disk_temptable_row_cost`**|1.0|向基于磁盘的临时表写入或读取一条记录的成本 。|
|**`memory_temptable_create_cost`**|2.0|创建基于**内存**的临时表的成本。增大此值会减少内存临时表的使用 。|
|**`memory_temptable_row_cost`**|0.2|向基于内存的临时表写入或读取一条记录的成本 。|
|**`key_compare_cost`**|0.1|两条记录做**比较操作**的成本（多用于排序）。增大此值会提升 filesort 的成本，使其更倾向于使用索引排序 。|
|**`row_evaluate_cost`**|0.2|**检测一条记录是否符合搜索条件**的成本。增大此值可能使优化器更倾向于使用索引而非全表扫描 。|

---

### 3. `mysql.engine_cost` 详细常数

该表增加了 `engine_name` 列（如 `default` 表示适用于所有引擎）和 `device_type` 列（目前仅 0 有效） 。

|**成本常数名称**|**默认值**|**描述与影响**|
|---|---|---|
|**`io_block_read_cost`**|1.0|从**磁盘**读取一个块（页）的成本 。|
|**`memory_block_read_cost`**|0.25|从**内存**（Buffer Pool）读取一个块（页）的成本 。|

---

### 4. 调节步骤与注意事项笔记

#### 如何修改成本常数？
调节过程分为两步，修改后必须刷新才能生效 ：
1. **更新数值**：通过标准的 SQL `UPDATE` 语句修改。
    - _示例_：将检测记录的成本改为 0.4：
        `UPDATE mysql.server_cost SET cost_value = 0.4 WHERE cost_name = 'row_evaluate_cost';`
2. **重新加载**：执行刷新指令。
    - `FLUSH OPTIMIZER_COSTS;`

#### 关键笔记点
- **恢复默认值**：将 `cost_value` 设置为 `NULL` 并执行 `FLUSH` 操作，即可恢复为系统默认的成本值 。
- **临时表成本**：MySQL 在处理 `DISTINCT`、`GROUP BY`、`UNION` 等查询时常需建立内部临时表 。数据量大时用磁盘（MyISAM/InnoDB），小时用内存（Memory） 。
- **多表连接影响**：在多表连接查询中，被驱动表的连接列是否有索引（如 `eq_ref` 或 `ref` 访问方法）会极大影响成本计算 。