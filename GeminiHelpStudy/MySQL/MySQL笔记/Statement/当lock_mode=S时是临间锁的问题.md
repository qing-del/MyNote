> [!warning] Question❓
> 在一张id为主键，附带两列（name，age）中
> 我们令age为二级索引
> 
> 其中有
> 1,Java,1
> 3,PHP,3
> 7,Ruby,7
> 8,rose,8
> 
> 然后我们开启事务，并在事务先进行如下操作：
> `select * from stu where age = 3 lock in share mode;`
> 但是我们不提交
> 
> 然后在另一个窗口中执行：
> `select object_schema,object_name,index_name,lock_type,lock_mode,lock_data from performance_schema.data_locks;`
> 会得到以下内容（部分）
> 一条index_name为`index_stu_age`，`lock_mode`=`S`，`lock_data`=3,3
> 一条index_name为`Primary`，`lock_mode`=`S,REC_NOT_GAP`，`lock_data`=3
> 一条index_name为`index_stu_age`，`lock_mode`=`S,GAP`，`lock_data`=7,7
> 
> 我想知道为什么第一个锁行信息中的`lock_mode`=`S`，然后课程中的老师说这个是临间锁？我有点不太懂，这里的S不是行级读锁吗？

---

这是一个非常棒的问题！你不仅观察仔细，而且确实抓住了 MySQL 在展示锁信息时一个非常容易让人误解的“障眼法”。
你课程里的老师说得完全正确：**这里的 `S` 确实代表临键锁（Next-Key Lock）。**
你之所以感到困惑，是因为你把**锁的读写属性**（S/X）和**锁的范围算法**（Record/Gap/Next-Key）混淆了。让我为你直接点破这层窗户纸：

### 1. `S` 是读锁，但这不代表它“只是”个记录锁
在 InnoDB 中，锁有两个维度的特征：
1. **读写模式 (Lock Mode)**：是共享锁（`S`，读锁）还是排他锁（`X`，写锁）？
2. **锁定范围 (Lock Type/Algorithm)**：是仅仅锁一条记录？还是锁一个间隙？还是两者都锁？

在 `performance_schema.data_locks` 表中，**InnoDB 的默认行锁算法就是临键锁（Next-Key Lock）**。为了简化显示，MySQL 采取了一种“做减法”的展示逻辑：

|**lock_mode 显示值**|**真实的锁范围类型**|**物理含义**|
|---|---|---|
|**`S`** 或 **`X`**|**临键锁 (Next-Key Lock)**|默认状态。既锁定记录本身，又锁定记录前面的间隙（Gap + Record）。|
|**`S, REC_NOT_GAP`**|**记录锁 (Record Lock)**|退化状态。明确告诉你：只锁记录，**不包含**间隙。|
|**`S, GAP`**|**间隙锁 (Gap Lock)**|退化状态。明确告诉你：只锁间隙，**不包含**记录本身。|

所以，当你只看到一个干干净净的 `S` 时，它其实代表的是**“完全体”**——一个共享的临键锁（Shared Next-Key Lock）。

---

### 2. 结合你的实验数据详细复盘
你执行了 `select * from stu where age = 3 lock in share mode;`，由于 `age` 是二级索引（允许有重复值），InnoDB 为了防止幻读，必须把 `age=3` 周围可能插入新数据的位置都锁死。
我们来逐一拆解你查出来的这三条锁信息 ：
- **第一条：`index_stu_age`，`lock_mode`=S，lock_data=3,3**
    - **解析**：这是一个**共享临键锁**。
    - **作用范围**：它不仅锁住了 `age=3` 这条二级索引记录本身，还锁住了它前面的间隙。结合你的数据，前面一条记录是 `age=1`，所以它锁住的范围是 `(1, 3]` 。

- **第二条：`Primary`，`lock_mode`=S,REC_NOT_GAP，lock_data=3**
    - **解析**：这是一个**共享记录锁**。
    - **作用范围**：因为你查的是 `select *`，InnoDB 需要回表去查主键索引 。主键是唯一的，直接精准锁定 `id=3` 这一行即可，不需要锁间隙。所以它明确加上了 `REC_NOT_GAP`，告诉你它只锁这一条聚簇索引记录。

- **第三条：`index_stu_age`，`lock_mode`=S,GAP，lock_data=7,7**
    - **解析**：这是一个**共享间隙锁**。
    - **作用范围**：只锁前面的间隙不够，万一有其他事务在 `age=3` 的后面插入了一个新的 `age=3` 的数据怎么办？为了彻底防止幻读，InnoDB 会向右继续找，找到下一条记录 `age=7` 。然后在这条记录前加上间隙锁（`GAP`）。所以它锁住的范围是 `(3, 7)`，注意这里不包含 7 本身。
 

### 总结

简单来说，在二级索引的等值查询中，为了防止幻读，InnoDB 采用了经典的**“左闭右开”**包围圈：
1. 用一个 **临键锁 (`S`)** 锁住前面到目标值的区间：`(1, 3]`。
2. 用一个 **间隙锁 (`S, GAP`)** 锁住目标值到后面一个值的区间：`(3, 7)`。
    这两者合在一起，就把 `age=3` 附近所有可能产生幻读的缺口全部封死了。