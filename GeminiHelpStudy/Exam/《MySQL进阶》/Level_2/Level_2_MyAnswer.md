> 做前思想：
> 刚看到的Level2的完整时候，整个人是有点畏难情绪的，感觉太难了，因为这是我学完第一次进行实战，感觉完全没有思路，还特别怕做烂，我还特意去AI那里做了一波分析，用的是下面这段提示词
> ```txt
> 请不要涉及任何提示和知识点的透露，这是我目前大二的用来考验自己对MySQL进阶篇知识掌握程度考卷，你帮我分析一下这张考卷的难度（给出难度系数）；如果你想说的具体一点，你可以利用高中物理的角度来分析，出一道类比的题目，并且做分析，这里可以用高中物理知识点来联合说明；再说一次，不要透露任何跟MySQL相关的知识点和解题提示；
> ```
> 然后我还想了一下，要不要让AI来给我先拆解，给我一下类似的但**不是**要拆这道level2，就是先从部分（别的类似的相关的题目）解决开始，然后再过渡到这个整体
> 但是后面过了一段时间，我想了一下毕竟畏难，就等于我走了普通人的路，为了能走到顶端，我还是选择沉下心来一张一张表去分析，然后尝试去寻找思路了，尽管可能我会做得不及格，但是我也不能做，做了还能对，不做就等于全错，不做就不知道自己到底哪里不行
> 好了，后面发现分析了也毫无思路，只能先慢慢锻炼一下工程思维了(在`Before_Level2_Train`文件夹中保留了我训练时使用的题目和追问过程)

> 练习时：
> 感觉单表优化还是挺简单的，结合一些简单的理论知识就可以搞定了
> 等到简单的多表莲藕查询，我发现了，不对劲，怎么多了个莲藕过程和整体关系分析之后，好像是并不是那么的简单，感觉难度像是坐火箭一样飞上去了
> 我感觉多表“莲藕”查询的优化，我还得是先认真学习一下SQL优化器以及成本计算，这个执行计划到底是怎么来的（跑去学习SQL优化器了）

> 练习的内容我丢在`Before_Level2_Train`文件夹中了，教授可以看一眼

---

> 我在草稿纸上对表进行了分析
> 而且我个人分析，这个是深分页搜索20条**订单**数据
> 所以重点在分出最新创建的20个有效订单，而不是20个订单物品

### 任务一

| 表 | 预测 type | 预测 key | 预测 rows（量级） | 预测 Extra |
|:---:|:---:|:---:|:---:|:---|
| orders | ref | idx_user_status_created | 1w | using index condition, using where |
| users | ref | idx_gender | 70w | null |
| order_items | ref | idx_order_product | 2w | null |
| products | eq_ref | primary key | 2w | using where |

#### 分析
1. 首先按照我的个人分析，虽然是`products`表的数据量小，扇出会更小，但很可惜，它的`where`过滤条件中并没有能使用索引的，然后`users`表中使用`gender`过滤之后数据量比`product`表总量小，而且不是“猜”出来的数据量，SQL优化器大概率会将`users`当作第一个驱动表
2. 判断`users`表的`gender`二级索引会走全表还是索引呢？不知道，所以我选择大致算一下
    - **全表扫描**
        - 如果可以使用`SHOW TABLE STATUS`就好了，可惜没有，所以我这里打算猜一下
        - 按照《MySQL是怎样运行的》这本书中的例子，一页可能有100行，所以我们这里有1.5w页
        - IO成本：15000 * 1.0 + 微调值 ≈ 15000.0
        - CPU成本：1500000 * 0.2 + 微调值 ≈ 300000.0
        - 总成本：3150000.0（实际应该会低一点）
    - **使用索引**
        - 范围读取索引，只有一个区间，所以IO成本是：1.0
        - 我们这里可以估摸着有70w条符合的数据行，按照我们上面说的规定
        - 读取所有符合条件的数据行的页IO成本：7000 * 1.0 = 7000.0
        - 读取所有符合条件的二级索引得到的数据行CPU成本：700000 * 0.2 = 140000.0
        - 回表查询的IO成本：700000 * 1.0 = 700000.0
        - 读取所有回表得到数据行的CPU成本：700000 * 0.2 = 140000.0
        - 总成本：987000.0
    - 所以这里走索引扫描，得到大约70w的扇出量
3. 第一个被驱动表我个人认为是`orders`表，虽然是`products`表数据量小，但是无法直接关联
    - 这里进行查找的时候会使用`idx_user_status_created`这个联合索引，但很可惜，只能用前面两个，后面最后一个用不了，因为它的匹配条件使用了`DATE`函数进行了运算，导致索引失效，最后一点的过滤需要扔到`Service层`执行
> 这里发现不对劲，似乎是`orders`作为驱动的话扇出会更小....
    - 由于查询所需要返回的结果中，无法使用索引下推，这里会导致大量的回表查询
4. 第二个被驱动表我认为是`order_items`表，原因也很简单，因为无法直接关联`products`表
    - 不难发现，也是`二级索引 + 回表`的组合
5. 最后利用从`oi`表得到的`product_id`去`products`表进行聚簇索引查询，但是`where`条件要在`Service`层进行过滤

#### 补充
- 上面是我直接分析的
我分析完之后，我更加感觉应该是`orders`->`users`->`order_items`->`products`

| 表 | 预测 type | 预测 key | 预测 rows（量级） | 预测 Extra |
|:---:|:---:|:---:|:---:|:---|
| orders | ref | idx_user_status_created | 24w | using index condition, using where |
| users | eq_ref | primary key | 12w | using where |
| order_items | ref | idx_order_product | 46w | null |
| products | eq_ref | primary key | 46w | using where |

- 因为根据题目给出的数据分布，最近一个月总共有`24w`行订单数据
- 利用订单表的低扇出，去用户表中使用主键查询，放到`Service层`过滤一下，都不用回表，何乐而不为
- 剩下的就是老套路了

---

### 任务二
- 从上面分析的结果来看，我们已经分析了一部分前置信息了
- 再结合整个SQL来看，其实还涉及了`using filesort`外排序，以及是深分页的情况
- 总的来说就是这四个主要问题
    - 大量回表随机IO
    - 索引失效
    - 外排序
    - 深分页

#### 大量回表随机IO
- 每一次随机IO都代表着磁盘上的磁头要进行物理偏移，这个过程是很慢的

#### 索引失效
- 因为使用了`DATE`函数进行运算，所以最后一个索引不能下推
- 使得联合索引中使用了前面两个索引进行过滤之后，还得把这部分数据通过IO送到`Service层`进行最后过滤
- 会产生无效IO

#### 外排序
- 由于最后面产生的数据集是无序的
- 必须要使用一次外排序来进行
- 而且有可能排序缓冲区会装不下，就会导致需要内存与磁盘互动
- 又是出现大量的IO

#### 深分页
- 这里通过IO拿到最后数据并进行排序之后
- Service层会扫描这些完整的150020条数据
- 然后丢掉150000行数据，导致浪费和无效回表IO
- 需要使用"**延迟关联**"来解决

---

### 任务三
```sql
-- 先建立联合索引 -- 解决回表问题
create index idx_created_status_amount_no_user on orders(created_at, status, total_amout, order_no, user_id);
create index idx_order_product_quantity_subtotal on order_items(order_id, product_id, quantity, subtotal);

-- 给数据瘦身（解决一下深分页大浪费的问题）（解决深分页问题）
WITH filter_orders AS (
    SELECT
        o.id AS primary_id
    FROM orders o
    JOIN users  u  ON u.id = o.user_id
    JOIN order_items oi ON oi.order_id = o.id
    JOIN products    p  ON p.id = oi.product_id
    WHERE
        u.gender = 1
        AND o.created_at >= '2024-01-01 00:00:00' -- 解决索引失效、外排序问题
        AND o.order_status IN (3, 4)
        AND p.is_deleted = 0
        -- 防止数据膨胀
        AND EXISTS (
            SELECT 1 FROM order_items oi WHERE oi.order_id = o.id
        )
    ORDER BY o.created_at DESC
    LIMIT 500000, 20;
)
SELECT
    u.username,
    u.city,
    u.gender,
    o.order_no,
    o.total_amount,
    o.order_status,
    o.created_at,
    p.name        AS product_name,
    p.price       AS unit_price,
    oi.quantity,
    oi.subtotal
FROM filter_orders t
JOIN orders o ON o.id = t.primary_id
JOIN users  u  ON u.id = o.user_id
JOIN order_items oi ON oi.order_id = o.id
JOIN products    p  ON p.id = oi.product_id
ORDER BY o.created_at DESC;  -- 其实我感觉这一行可以不加，因为我感觉本来其实应该就是有序的了，加了只是我作为保底一下
```
> 由于用户表和商品表中在这个联查循序下会使用各自的聚簇索引，也就可以是没有**索引覆盖**这一个说法了，不追求极致性能，让其必须走**索引覆盖**的话，可以再建立联合索引

- 还有就是外排序`using filesort`，分析一下B+树联合索引的结构
```cpp
bool cmp(node& a, node& b) {
    if(a.user_id != b.user_id) return a.user_id < b.user_id;
    if(a.created_at != b.created_at) return a.created_at < b.created_at;
    if(a.status != b.status) return a.status < b.status;
    if(a.total_amout != b.total_amout) return a.total_amout < b.total_amout;
    return a.order_no < b.order_no;
}

signed main() {
    node arr[]; // 假设是B+树叶子节点的数组
    sort(arr, arr+n, cmp);  // 叶子节点的链表顺序是这样的
    return 0;
}
```
- 按照上述代码的排序方式，利用联合索引扫描出来的数据行，就是在`created_at`有序的情况下，范围查询`status`这个这一列，这样的话，我感觉其实就是**有序的**
- 其实我感觉还有一种就是修改一下表结构，比如：
    - 在`order_items`中加入`unit_price`列和`product_name`这两个冗余字段，这样可以将四表联查变成三表联查，这样在伪代码层面，就相当于四层for变成了三层for循环了