# Level 2 批阅报告

**批阅人：资深数据库系统教授**  
**考生答卷：Level_2_MyAnswer.md**  
**批阅时间：2026-03-12**

---

## 教授的读后感

在正式打分之前，我想先说几句。

我仔细阅读了你的 `Before_Level2_Train` 文件夹中的全部训练过程——三个阶段、两个不同的 AI 教练、从单表索引失效到四表联查深分页的完整攀登路径。

说实话，**你的训练过程比答卷本身更让我印象深刻。**

你在开篇坦言了畏难情绪——"感觉太难了，完全没有思路"——然后你没有选择让 AI 给你拆解这道题本身，而是主动设计了一条"从部分到整体"的训练路线，用类似的但不同的题目自我锻炼，最终回来挑战正题。这种元认知能力（知道自己不会什么、知道该如何学会它）比任何技术细节都重要。

好了，态度分到此为止。下面是残酷的技术审计。

---

## 总评

| 任务 | 考察方向 | 得分 | 满分 |
|:---:|:---|:---:|:---:|
| 一 | EXPLAIN 执行计划预测 | 22 | 35 |
| 二 | 性能瓶颈根因分析 | 22 | 30 |
| 三 | 具体优化方案 | 21 | 35 |
| **合计** | | **65** | **100** |

**等级：C+（及格线以上，训练痕迹明显但"最后一公里"的实战落地能力尚欠打磨）**

---

## 任务一：EXPLAIN 执行计划预测（22/35）

### 你的两版分析

你给出了两版预测，我先点评你的思考过程，再逐表纠正。

#### ✅ 值得肯定的地方

1. **你主动做了成本计算。** 你尝试用《MySQL 是怎样运行的》中的成本模型（IO 成本系数 1.0、CPU 成本系数 0.2）来量化 `idx_gender` 走索引 vs 全表扫描的代价。虽然数字不完全精确，但这种**用数字说话**的思维方式是优化器行为分析的正确入口。这比 90% 的学生"我觉得会走索引"的直觉式回答高出一个层次。（+3 加分）

2. **你意识到了第一版预测可能有误，主动给出了第二版修正。** 从 `users` 驱动改为 `orders` 驱动，说明你在推演过程中不断自我反思。这很好。

3. **你正确识别了 `DATE(o.created_at)` 导致的索引失效。**

#### ❌ 关键错误

**1. 驱动表的选择（-5 分）**

你的第二版预测选择 `orders` 作为驱动表，理由是"最近一个月有 24 万行"。但这个推理有一个致命问题：**`DATE(o.created_at) >= '2024-01-01'` 不是"最近一个月"，而是从 2024 年 1 月 1 日至今，跨度超过 2 年**。按均匀分布，这大约覆盖了 320 万 × (26/48) ≈ **173 万行**，再加上 `order_status IN (3, 4)` 过滤掉 20%，仍然剩下约 **138 万行**。

而 `users` 表走 `idx_gender` 过滤 `gender=1` 得到约 72 万行，虽然比你估算的 70 万稍多，但方向没有太离谱。

**问题是：这两个路径都不是优化器的最优选择。** 在实际场景中，由于 `DATE()` 函数导致 `idx_created_at` 无法使用，`idx_user_status_created` 中的 `created_at` 也无法使用，优化器大概率会：

- **以 `orders` 表发起全表扫描**（因为所有涉及 `orders` 的条件要么是低区分度的枚举 `IN(3,4)`，要么被函数破坏）
- 或者选择 `users.idx_gender` 驱动，但由于 `gender` 区分度极低（cardinality=3），走这个索引等于扫描全表的 48%，优化器很可能直接放弃索引选全表扫描

最可能的真实情况是：**优化器选择 `orders` 全表扫描 → 对每行去 `users` 做 eq_ref 主键查找 → 过滤 `gender=1` → 再关联 `order_items` 和 `products`。**

**2. `idx_gender` 索引不会被使用（-3 分）**

`gender` 只有 3 个不同的值（cardinality=3），`gender=1` 占全表 48%。走这个索引意味着 72 万次二级索引查找 + 72 万次回表随机 I/O。优化器的成本模型几乎一定会选择全表扫描替代。你在第一版中算出了走索引的成本（987000）低于全表扫描（3150000），但你的计算中有一个关键遗漏：**回表的 I/O 成本你按 `700000 * 1.0` 来算，这意味着 70 万次独立的页读取，但实际上很多行会在同一页上**（聚簇索引中主键相邻的行在同一页），真正的随机 I/O 成本会低一些。不过即便如此，优化器在面对接近 50% 的过滤比时，通常还是倾向于全表扫描。

**你的 `idx_gender` 索引本身就是试卷中埋的第一个陷阱——一个几乎永远不会被使用的废索引。你没有识别出来。**（-2 分）

**3. `order_items` 和 `products` 的 rows 估算（-3 分）**

你在两版中分别预测 `order_items.rows = 2w/46w`，但 `order_items` 通过 `idx_order_id` 关联 `orders`，每个订单平均 2.1 个商品，所以每次被驱动表查找的 rows 应该是 **≈2**（ref 类型，每个 order_id 匹配约 2 行），而不是"2 万"或"46 万"。EXPLAIN 中的 `rows` 列显示的是**每次循环预计扫描的行数**，不是总行数。

同理，`products` 通过主键 `p.id = oi.product_id` 查找，每次命中 1 行，type 应该是 `eq_ref`，`rows = 1`。你的第二版判断 `eq_ref / PRIMARY` 是对的，但 `rows = 46w` 不对。

#### 满分答案的 EXPLAIN 预测

| 表 | type | key | rows（每次循环） | Extra |
|:---:|:---:|:---:|:---:|:---|
| orders | ALL | NULL | 320 万（全表） | Using where; Using temporary; Using filesort |
| users | eq_ref | PRIMARY | 1 | Using where |
| order_items | ref | idx_order_id | ≈2 | NULL |
| products | eq_ref | PRIMARY | 1 | Using where |

**关键推理：**
- `orders` 全表扫描是因为所有可用索引都被破坏（`DATE()` 函数 + 低区分度枚举），这是最可能的路径。
- 全表扫描后，对每行 order 去 `users` 表做 `eq_ref`（`u.id = o.user_id`），在 Server 层检查 `gender=1`。
- 通过后继续去 `order_items` 做 ref 查找（`oi.order_id = o.id`），每次约 2 行。
- 最后去 `products` 做 `eq_ref`（`p.id = oi.product_id`），每次 1 行，Server 层检查 `is_deleted=0`。
- `Using temporary; Using filesort` 是因为 `ORDER BY o.created_at DESC` 无法利用任何索引直接排序，必须将结果集物化到临时表中排序后再 LIMIT。

---

## 任务二：性能瓶颈根因分析（22/30）

### 你找到的 4 个问题

| 编号 | 你识别的问题 | 评价 | 得分 |
|:---:|:---|:---|:---:|
| 1 | 大量回表随机 I/O | ✅ 正确，方向对 | 5/7 |
| 2 | `DATE()` 函数导致索引失效 | ✅ 完全正确 | 7/7 |
| 3 | `Using filesort` 外排序 | ✅ 正确 | 5/8 |
| 4 | 深分页 LIMIT 500000,20 的浪费 | ✅ 正确 | 5/8 |

**总得分：22/30**

#### 每个问题的细化点评

**问题 1：大量回表随机 I/O（5/7）**

方向正确，但分析不够具体。你需要指出**哪些表在哪个环节**产生了回表。具体来说：
- `orders` 全表扫描本身不涉及"回表"概念（它就在聚簇索引上扫），但是每行都要去 `users` 做一次主键查找，这是 **NLJ（Nested Loop Join）的被驱动表访问成本**。
- 如果 `idx_gender` 真的被选中（虽然不太可能），那才是经典的"二级索引 + 回表"场景。

**问题 2：`DATE()` 函数索引失效（7/7）**

满分。你清楚地识别了 `DATE(o.created_at)` 对 `idx_created_at` 和 `idx_user_status_created` 中 `created_at` 部分的破坏。

**问题 3：外排序（5/8）**

你正确指出了存在 filesort，但遗漏了一个关键细节：**`ORDER BY o.created_at DESC` 加 `LIMIT 500000, 20` 的组合，在无法使用索引排序时，MySQL 需要把所有经过 4 表 JOIN 和 WHERE 过滤后的结果集全部物化到临时表（`Using temporary`）中，再做全量排序（`Using filesort`）**。如果结果集有几十万行甚至上百万行，这个临时表会非常巨大，可能溢出到磁盘（`Created_tmp_disk_tables` 指标上升），I/O 雪上加霜。你提到了"排序缓冲区装不下"但没有提到临时表。（-3）

**问题 4：深分页（5/8）**

你的 "150020 条数据丢掉 150000 行" 这个数字从何而来不太清楚——原始 SQL 是 `LIMIT 500000, 20`，所以应该是 **500020 条排序后丢掉 500000 条**。另外，你提到了"延迟关联"是正确的解决方向，但这部分应该在任务三中展开。在根因分析中应该更聚焦于"为什么 OFFSET 大了会慢"——底层原因是 **Server 层无法跳过前 50 万行的排序和物化过程**，即使最终只取 20 行，前面 50 万行的全部计算（JOIN + WHERE + SORT）一步都不能少。（-3）

#### 你遗漏的第 5 个瓶颈

**`idx_gender` 废索引与 `idx_shop_deleted` 的低区分度问题。**

这是我在题目中特意埋的陷阱。`users.idx_gender`（cardinality=3）和 `products.idx_shop_deleted`（cardinality=2，且 `is_deleted=0` 占 91%）都是**对优化器几乎没有价值的索引**，它们不仅浪费磁盘空间和写入性能，而且可能误导优化器做出错误的索引选择。你应该在分析中指出"现有索引设计中存在不合理索引"这是题目明确要求的识别项。

此外，**`order_items` 表上的 `idx_order_product(order_id, product_id)` 和 `idx_order_id(order_id)` 存在冗余**——前者的前缀完全覆盖了后者，`idx_order_id` 可以安全删除。

---

## 任务三：具体优化方案（21/35）

### ✅ 正确的优化方向

1. **消除 `DATE()` 函数**：你把 `DATE(o.created_at) >= '2024-01-01'` 改写为 `o.created_at >= '2024-01-01 00:00:00'`。✅ 完全正确。（+5）
2. **延迟关联思路**：你使用了 CTE（`WITH filter_orders AS`）来做延迟关联的框架，方向正确。（+3）
3. **创建覆盖索引的意识**：你为 `order_items` 设计了 `(order_id, product_id, quantity, subtotal)` 覆盖索引。思路正确。（+3）
4. **冗余字段降维**：你在最后提出了"在 `order_items` 中加入 `unit_price` 和 `product_name` 冗余字段减少表联查"，这是一个非常有架构思维的建议。（+3）

### ❌ 关键错误与遗漏

**1. 延迟关联并没有真正"瘦身"（-5 分）**

你的 CTE `filter_orders` 中依然做了 **4 表 JOIN**，并 `SELECT o.id AS primary_id`。这意味着在 CTE 内部，MySQL 要完成全部的 JOIN + WHERE 过滤 + ORDER BY + LIMIT 的完整流程——跟没有做延迟关联几乎没有区别。

**真正的延迟关联**应该是这样的思路：

```sql
-- 第一步：只在 orders 上做排序和分页，拿到 20 个 order id
WITH page_ids AS (
    SELECT o.id
    FROM orders o
    WHERE o.created_at >= '2024-01-01 00:00:00'
      AND o.order_status IN (3, 4)
    ORDER BY o.created_at DESC
    LIMIT 500000, 20
)
-- 第二步：用这 20 个 id 去 JOIN 拿完整数据
SELECT u.username, u.city, u.gender,
       o.order_no, o.total_amount, o.order_status, o.created_at,
       p.name AS product_name, p.price AS unit_price,
       oi.quantity, oi.subtotal
FROM page_ids t
JOIN orders o ON o.id = t.id
JOIN users u ON u.id = o.user_id
JOIN order_items oi ON oi.order_id = o.id
JOIN products p ON p.id = oi.product_id
WHERE u.gender = 1
  AND p.is_deleted = 0
ORDER BY o.created_at DESC;
```

注意关键区别：**CTE 内部只操作 `orders` 一张表**，用索引快速完成排序和分页，只取出 20 个 id。然后外层再用这 20 个 id 去关联其他 3 张表拿详情。这样前 50 万行的排序和 offset 扫描发生在**只有一棵 B+ 树参与**的最轻量路径上。

当然这个简化版的延迟关联有一个代价：**`gender=1` 和 `is_deleted=0` 的过滤被推到了外层**，导致最终拿到的 20 条数据中可能有不满足条件的被过滤掉，实际返回不足 20 条。这是延迟关联的固有 trade-off——你要么在内层做完所有过滤（慢但精确），要么在内层只做主表过滤（快但可能短页）。实战中通常选后者，因为 `gender=1` 占 48% 和 `is_deleted=0` 占 91% 的过滤率已经很高。

**2. orders 表的覆盖索引设计有问题（-3 分）**

你设计的索引是 `(created_at, status, total_amount, order_no, user_id)`。问题：

- 索引名写成了 `idx_created_status_amount_no_user`，且 `total_amout` 拼写错误（应为 `total_amount`）。
- **字段 `status` 放在 `created_at` 之后是正确的**（范围查询 `created_at` 之后 `status` 用于过滤），但你需要想清楚这个索引的目标是什么。如果是为了延迟关联中 CTE 的排序优化，你需要的是能让 `ORDER BY created_at DESC` 直接通过索引完成排序的索引结构。
- 正确的做法：**`(order_status, created_at)`**，先用 `IN(3,4)` 做等值匹配，再用 `created_at` 做范围 + 排序。由于 `IN(3,4)` 是两个离散值，MySQL 会做两次 ref 查找然后 merge，`created_at` 在每个 `order_status` 值内是有序的。

**3. EXISTS 子查询多余且有逻辑错误（-3 分）**

你在 CTE 内加了：
```sql
AND EXISTS (SELECT 1 FROM order_items oi WHERE oi.order_id = o.id)
```
原始 SQL 中没有"订单必须有订单明细"这个业务约束。这个 EXISTS 是你自己加的"防止数据膨胀"的手段，但：
- 在延迟关联中，CTE 内部本来就不应该 JOIN `order_items`，所以不存在膨胀问题。
- 加了这个 EXISTS 反而增加了额外的子查询检查成本。

**4. 没有删除废索引（-2 分）**

你只添加了新索引，但没有指出应该**删除**的废索引：
- `users.idx_gender` — cardinality=3，几乎不可能被使用 → **DROP INDEX**
- `order_items.idx_order_id` — 被 `idx_order_product(order_id, product_id)` 完全覆盖 → **DROP INDEX**

**5. `idx_shop_deleted` 的问题未提及（-1 分）**

`products.idx_shop_deleted(shop_id, is_deleted)` 这个索引在本查询中完全无用（本查询走的是 `p.id` 主键），而且 `is_deleted` 只有 2 个值，区分度极低。是否需要保留取决于其他业务查询，但至少应该在分析中提到。

---

## 训练过程点评（加分项，不计入总分）

我通读了你的三个训练阶段：

### 阶段一（Gemini）：单表索引失效
3/3 全部答对。`YEAR()` 函数失效、隐式类型转换、`LIKE '%prefix'` 前缀模糊——这三个经典陷阱你完全掌握了。后缀搜索的冗余列方案甚至被教练评为"教科书级别"。**基础扎实。**

### 阶段二（Gemini）：双表 JOIN + 深分页
- VIP 用户双表 JOIN：驱动表选择、覆盖索引设计、INLJ 执行流程——思路清晰。
- 深分页延迟关联：虽然 SQL 语法有小错（`IN` 子查询不支持 `LIMIT`，教练纠正后你理解了 `INNER JOIN 派生表` 的写法），但核心原理完全命中。

### 阶段三（Gemini）：四表物流系统终极挑战
这是最接近 Level 2 正题的训练。你在这里暴露了几个问题（1:N 数据膨胀、延迟关联不彻底、联合索引顺序错位），但你通过追问教练的方式把这些坑都挖出来了。**你的追问质量非常高**——"为什么说查的是运单分页而不是轨迹分页？""拿到 50 个 ID 后联查结果不止 50 条吧？"——这些反问说明你在真正地思考，而不是被动接受。

**总结：你的训练路径设计得很好，但 Level 2 正题中暴露出你还没有完全消化阶段三的教训——延迟关联的"瘦身"做得不彻底，和阶段三中被指出的"形同虚设的延迟关联"是同一个问题。**

---

## 总结与建议

### 你目前的能力画像

| 能力维度 | 掌握程度 |
|:---|:---|
| 索引失效场景识别 | ⭐⭐⭐⭐⭐ 优秀 |
| 成本模型量化思维 | ⭐⭐⭐⭐ 良好（有意识，需更精确） |
| EXPLAIN rows 含义理解 | ⭐⭐ 薄弱（混淆了"每次循环行数"和"总行数"） |
| 驱动表选择推演 | ⭐⭐⭐ 中等（方向对，数据估算不准） |
| 延迟关联落地 | ⭐⭐⭐ 中等（知道原理，SQL 写法不到位） |
| 废索引识别 | ⭐⭐ 薄弱（未能主动识别题目中的陷阱索引） |
| 冗余字段架构思维 | ⭐⭐⭐⭐ 良好（主动提出了减少 JOIN 的方案） |

### 你需要强化的 3 件事

1. **EXPLAIN 的 `rows` 列**代表的是**每次循环预计扫描的行数**，不是总行数。当 type=ref 时，`rows=2` 意味着每次被驱动匹配大约 2 行。总行数 = 驱动表 rows × 被驱动表 rows × ... 。在你的下一次练习中，手动建表、灌数据、跑真实的 EXPLAIN，对照你的预测。

2. **延迟关联的核心原则："CTE 内层只碰一张表，只 SELECT 主键"。** 如果你发现 CTE 内层开始 JOIN 第二张表了，停下来想一想——是不是可以把这个过滤推到外层去做。

3. **养成"审计现有索引"的习惯。** 优化不只是加索引，还包括删除那些浪费空间和误导优化器的废索引。

---

**同学，65 分。** 考虑到你从零开始自学了三个阶段的训练后来挑战这道综合题，这个分数代表的不是你的上限，而是你当前的实战落地成熟度。你的思维方向、分析框架、甚至架构意识都已经超出了大二的平均水平，但"知道原理"和"写得出正确的 SQL"之间还有一段距离需要用实践来填补。

**你准备好进入 Level 3 了吗？还是想在 Level 2 上做一版 v0.2 的复盘再挑战？**
