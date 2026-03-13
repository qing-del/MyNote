### 场景一

#### Part A
1. 答：
    - 事务A快照读读到的`amount`为`150.00`;
    - 按照题目的假，事务A会在执行快照读的时候生成一个`ReadView`，此时的`ReadView`是
        - `m_ids`=空集
        - `max_trx_id`=100
        - `min_trx_id`=100
        - `creator_trx_id`=99

> 这里我想说的是，我记得事务中如果是先用快照读的话，事务id应该是0吧
> 后续使用`update`语句更新的时候才会分配事务id，然后将`ReadView`的`creator_trx_id`改为新分配的事务id吧

2. 答：
    - 在T7时刻，依旧是读到`amount`为`150.00`
    - 在`RR`隔离等级下，每个事务由第一个快照读生成`ReadView`之后，后续所有的快照读都是使用这个`ReadView`，仅有发生修改的语句时，会修改`creator_trx_id`来使自己能读到自己做出的修改
    ```md
    ->此时T7的快照读会先读到最新的版本链->最新版本（由事务B修改的版本）(id=5, amount=999.00,trx_id=100)
    ->发现最新版本的版本链中`trx_id`>=`ReadView.max_trx_id`，这个是这个快照读`ReadView`之后才生成的事务修改的，所以不能读->通过`rollpoint`找到下一条版本链
    ->下一个版本(id=5,amount=150.00,trx_id=1)，发现`trx_id`<`ReadView.min_trx_id`，这个是`ReadView`生成前就已经不活跃的老事务完成的修改了，所以是可以读的，就会读到`amount=150.00`
    ```
    > 这里假设我们一开始插入数据时候，`insert`的事务id为1

#### Part B
3. 答：
    - 我回去看了一眼表的SQL语句，首先按照二级索引的索引树结构，我认为过程是这样的：会先通过二级索引找到聚簇索引，通过聚簇索引找到完整数据行，然后此时给`orders`表加上`IX`锁，利用聚簇索引给找到`id=3`和`id=5`的数据行加上`X`锁
    - `user_id=20`对应是叶子节点`id`为`[3,5]`这个区间，InnoDB会在这个二级索引得到的区间上加`X`锁，对应(3,5)区间，但是这里代表的是`GAP`锁，是防止别人修改这个区间和这两条数据行
    - 会通过聚簇索引给数据行加上`X`锁
    - `SELECT * FROM orders WHERE id = 3 FOR UPDATE;`会导致`id=3`的数据行被加上`X`锁，同时`orders`表会有`IX`锁；已知`IX`锁是标记锁，`X`锁是真的会锁并排斥其他锁（如`S`锁，`IS`锁，`IX`锁，当然排斥意向锁是因为锁在表上了），但是很可惜，此时事务A给`id=3`加上的`X`锁还没被释放，所以这个SQL语句是会被阻塞的；
    - 如果是在T6之后执行`INSERT INTO orders (id, user_id, order_status, amount) VALUES (4, 25, 1, 100.00);`，其实会被阻塞，然后产生*插入意向锁*，因为事务A在T6时刻加的锁，包括了`GAP`锁，会导致这个(3,5)区间也是无法被修改的，只有等到事务A结束之后，才会把锁放开，这个时候才可以被插入数据

> 回答问 d 的时候，我去查了一下，锁什么时候释放（我在犹豫是语句结束之后释放，还是事务结束之后释放），我有点模糊，然后我查完之后，我就能百分百确定我的思路了，其实有点犹豫的；
> 我想到一个问题，如果在一个（3，5）区间中出现`GAP`锁的时候，事务B和事务C都往`id=4`的位置执行`insert`，会产生两个插入意向锁，我记得插入意向锁可以并行执行，那么我想知道是哪条数据先插进去，等会去实践一下

---

### 场景二

#### Part A
1. 答：
    - 这里会加入`X`类型的`GAP`锁（如果用查看锁的语句查看的话）
    - 加锁区间就是(5,8)，如果用查看锁的SQL语句查看，区间就是这样的，但是会显示是挂在`id=8`上，但是实际上并没有锁住；（类比`id=5`和`id=8`的两条数据行是实体杆子，然后要封住`(5,8)`区间，就要挂载杆子上拉条围栏这样）
    - 因为InnoDB的B+树的叶子节点是链表，其实锁住的是链表两个节点之间的空间，使其无法进行插入操作
    ```md
    3<->5<->8<->10
    > 那么上锁之后就是这样的
    3<->5<-GAP锁->8<->10
    > 这个时候就锁住(5,8)区间了
    ```

2. 答：
    - 加锁区间是(10,15)这个区间
    - 没有重叠

#### Part B
3. 答：
    - 会的，因为此时事务Y还没有结束，所以在(10,15)区间上的锁还没有被释放，无法插入，事务X的插入语句会被阻塞
4. 答：
    - 这个是不会的，因为插入的区间是(8,10)，X事务锁住的是(5,8)这个区间，所以事务Y的插入语句不会被阻塞
5. 答：    
    - 并不会形成死锁，因为事务X虽然被阻塞了，但是不影响事务Y完成插入并最后提交释放锁，在事务Y提交/回滚之后，事务Y锁住的(10,15)区间会释放，然后事务X中被阻塞的插入语句就会拿到锁并完成插入，之后事务X就可以进行提交/回滚操作了；

> 如果将`INSERT INTO orders (id, user_id, order_status, amount) VALUES (9, 35, 1, 120.00);`改成`id=6`，我就不会做了
> 说真的，是形成死锁，这道题我还真不会答，因为我不是很清楚 InnoDB 是怎么处理死锁的，但是我能辨别出他形成了死锁

---

### 实践时间

#### 场景二的Part B
- 对于**场景二的Part B**的阻塞情况确实和我想的一样
```txt
-- 这是在T3之后进行查询得到的锁数据
mysql> select object_schema,object_name,index_name,lock_type,lock_mode,lock_data from performance_schema.data_locks;
+---------------+-------------+------------+-----------+-----------+-----------+
| object_schema | object_name | index_name | lock_type | lock_mode | lock_data |
+---------------+-------------+------------+-----------+-----------+-----------+
| advance_test  | orders      | NULL       | TABLE     | IX        | NULL      |
| advance_test  | orders      | NULL       | TABLE     | IX        | NULL      |
| advance_test  | orders      | PRIMARY    | RECORD    | X,GAP     | 15        |
| advance_test  | orders      | PRIMARY    | RECORD    | X,GAP     | 8         |
+---------------+-------------+------------+-----------+-----------+-----------+
```
- 不过可惜的是没查出来是锁住哪个区间，但是和我显示的东西预测的一样

#### 场景一的Part B
- 在T6之后查询锁情况：
```txt
mysql> select object_schema,object_name,index_name,lock_type,lock_mode,lock_data from performance_schema.data_locks;
+---------------+-------------+-------------+-----------+---------------+-----------+
| object_schema | object_name | index_name  | lock_type | lock_mode     | lock_data |
+---------------+-------------+-------------+-----------+---------------+-----------+
| advance_test  | orders      | NULL        | TABLE     | IX            | NULL      |
| advance_test  | orders      | idx_user_id | RECORD    | X,GAP         | 30, 8     |
| advance_test  | orders      | idx_user_id | RECORD    | X             | 20, 3     |
| advance_test  | orders      | idx_user_id | RECORD    | X             | 20, 5     |
| advance_test  | orders      | PRIMARY     | RECORD    | X,REC_NOT_GAP | 3         |
| advance_test  | orders      | PRIMARY     | RECORD    | X,REC_NOT_GAP | 5         |
+---------------+-------------+-------------+-----------+---------------+-----------+
```
- 跟我想得有点不太一样，GAP锁咋是锁`((20,5), (30,8))`这个二级索引区间，防止插入吗？那干鸡毛不防止插入中间，而是防止插入尾部？
- 对于问题`d`，查询语句确实被阻塞了；插入语句也被阻塞了；
- 然后就是我的实践了，看看都插入同一个位置，看看到底是插入谁的：最后是插入先加锁的`事务B'`的