# 数据库系统原理核心考核 - Level 2

**考核人：[你的学号/姓名]**  
**命题人：资深数据库系统教授**  
**考察范围：SQL 性能调优、EXPLAIN 执行计划解读、索引失效、Join 算法、深度分页优化**

---

## 教授前言

进入 Level 2，考察的不再是你能否背出底层机制的名字，而是你能否在一个真实的业务场景中，**从一段有问题的 SQL 出发，靠 EXPLAIN 提供的线索，沿着执行链路反向定位出性能的根源，并给出可落地的优化方案。**

以下是一个真实电商系统中的四张业务核心表。我已在注释和说明中埋下了若干陷阱。你的任务不仅是发现它们，还要解释"为什么这是一个陷阱"。

---

## 业务背景

某中型电商平台的订单履约系统，日均订单量约 8 万笔，系统运行 4 年，核心表数据量已进入百万级。以下是四张表的完整 DDL 及数据分布说明。

---

## 表结构定义（完整 DDL）

### 表一：用户表 `users`

```sql
CREATE TABLE `users` (
  `id`          BIGINT      NOT NULL AUTO_INCREMENT COMMENT '用户ID（主键）',
  `username`    VARCHAR(64) NOT NULL                COMMENT '用户名',
  `phone`       VARCHAR(20) NOT NULL                COMMENT '手机号',
  `gender`      TINYINT     NOT NULL DEFAULT 0      COMMENT '性别：0-未知 1-男 2-女',
  `city`        VARCHAR(32) NOT NULL DEFAULT ''     COMMENT '注册城市',
  `status`      TINYINT     NOT NULL DEFAULT 1      COMMENT '账号状态：1-正常 2-封禁 3-注销',
  `register_at` DATETIME    NOT NULL                COMMENT '注册时间',
  `last_login`  DATETIME        NULL                COMMENT '最后登录时间（未登录过为NULL）',
  PRIMARY KEY (`id`),
  UNIQUE KEY  `uk_phone` (`phone`),
  INDEX       `idx_gender` (`gender`),
  INDEX       `idx_city_status` (`city`, `status`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

**数据分布特征：**
- 总行数：约 **150 万行**
- `gender`：0（未知）占 **5%**，1（男）占 **48%**，2（女）占 **47%**，**区分度极低（cardinality ≈ 3）**
- `status`：1（正常）占 **97%**，2（封禁）占 **2%**，3（注销）占 **1%**，**区分度极低**
- `last_login`：有 **约 8 万行为 NULL**（约 5%），为从未登录的僵尸账号
- `city`：全国约 300 个城市，区分度尚可
- `register_at`：4 年内均匀分布

---

### 表二：订单表 `orders`

```sql
CREATE TABLE `orders` (
  `id`            BIGINT         NOT NULL AUTO_INCREMENT COMMENT '订单ID（主键）',
  `order_no`      VARCHAR(32)    NOT NULL               COMMENT '订单编号（业务唯一）',
  `user_id`       BIGINT         NOT NULL               COMMENT '下单用户ID',
  `shop_id`       BIGINT         NOT NULL               COMMENT '店铺ID',
  `total_amount`  DECIMAL(12, 2) NOT NULL               COMMENT '订单总金额',
  `order_status`  TINYINT        NOT NULL DEFAULT 1     COMMENT '订单状态：1-待支付 2-已支付 3-已发货 4-已完成 5-已取消',
  `created_at`    DATETIME       NOT NULL               COMMENT '下单时间',
  `paid_at`       DATETIME           NULL               COMMENT '支付时间（未支付为NULL）',
  `remark`        VARCHAR(512)       NULL               COMMENT '订单备注',
  PRIMARY KEY (`id`),
  UNIQUE KEY  `uk_order_no` (`order_no`),
  INDEX       `idx_user_id` (`user_id`),
  INDEX       `idx_shop_status` (`shop_id`, `order_status`),
  INDEX       `idx_created_at` (`created_at`),
  INDEX       `idx_user_status_created` (`user_id`, `order_status`, `created_at`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

**数据分布特征：**
- 总行数：约 **320 万行**
- `order_status`：4（已完成）占 **72%**，5（已取消）占 **15%**，3（已发货）占 **8%**，其余合计 **5%**
- `paid_at`：约 **15%（约 48 万行）为 NULL**（对应待支付或已取消的订单）
- `created_at`：4 年内均匀分布，最近 1 个月约有 **24 万行**
- `shop_id`：约 **2000 家店铺**，头部 50 家店铺贡献了 **60%** 的订单量（数据严重倾斜）

---

### 表三：商品表 `products`

```sql
CREATE TABLE `products` (
  `id`          BIGINT         NOT NULL AUTO_INCREMENT COMMENT '商品ID（主键）',
  `shop_id`     BIGINT         NOT NULL               COMMENT '所属店铺ID',
  `name`        VARCHAR(256)   NOT NULL               COMMENT '商品名称',
  `category_id` BIGINT         NOT NULL               COMMENT '分类ID',
  `price`       DECIMAL(10, 2) NOT NULL               COMMENT '商品单价',
  `stock`       INT            NOT NULL DEFAULT 0     COMMENT '库存数量',
  `is_deleted`  TINYINT        NOT NULL DEFAULT 0     COMMENT '逻辑删除：0-正常 1-已删除',
  `created_at`  DATETIME       NOT NULL               COMMENT '创建时间',
  PRIMARY KEY (`id`),
  INDEX `idx_shop_id` (`shop_id`),
  INDEX `idx_category` (`category_id`),
  INDEX `idx_shop_deleted` (`shop_id`, `is_deleted`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

**数据分布特征：**
- 总行数：约 **80 万行**
- `is_deleted`：0（正常）占 **91%**，1（已删除）占 **9%**，**区分度极低（cardinality = 2）**
- `category_id`：共约 500 个分类，分布较均匀

---

### 表四：订单明细表 `order_items`

```sql
CREATE TABLE `order_items` (
  `id`           BIGINT         NOT NULL AUTO_INCREMENT COMMENT '明细ID（主键）',
  `order_id`     BIGINT         NOT NULL               COMMENT '关联订单ID',
  `product_id`   BIGINT         NOT NULL               COMMENT '关联商品ID',
  `quantity`     INT            NOT NULL               COMMENT '购买数量',
  `unit_price`   DECIMAL(10,2)  NOT NULL               COMMENT '下单时单价（快照）',
  `subtotal`     DECIMAL(12,2)  NOT NULL               COMMENT '小计金额',
  PRIMARY KEY (`id`),
  INDEX `idx_order_id`   (`order_id`),
  INDEX `idx_product_id` (`product_id`),
  INDEX `idx_order_product` (`order_id`, `product_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

**数据分布特征：**
- 总行数：约 **680 万行**（平均每订单约 2.1 个商品）

---

## 考核目标：一段致命的慢查询

以下是某后台报表系统实际跑出的一条 SQL（原汁原味，未做任何修改）。这条 SQL 在数据量达到现有规模后，从原来的 0.3 秒膨胀到了目前的 **47 秒**，已导致多次报表超时告警。  

```sql
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
FROM orders o
JOIN users  u  ON u.id = o.user_id
JOIN order_items oi ON oi.order_id = o.id
JOIN products    p  ON p.id = oi.product_id
WHERE
    u.gender = 1
    AND o.order_status IN (3, 4)
    AND DATE(o.created_at) >= '2024-01-01'
    AND p.is_deleted = 0
ORDER BY o.created_at DESC
LIMIT 500000, 20;
```

---

## 你的任务

### 任务一：EXPLAIN 执行计划预测

请预测这条 SQL 在 EXPLAIN 执行计划中，**每张表各自的关键字段**可能呈现的内容，并逐一解释**为什么**会出现这样的结果。

请以下表格式作答（至少填写 type / key / rows / Extra 四列，并在表格下方附上你的推理过程）：

| 表 | 预测 type | 预测 key | 预测 rows（量级） | 预测 Extra |
|:---:|:---:|:---:|:---:|:---|
| orders | | | | |
| users | | | | |
| order_items | | | | |
| products | | | | |

---

### 任务二：性能瓶颈根因分析

请指出这条 SQL 存在的**所有性能瓶颈**（至少找出 4 个独立问题），并对每个问题说明：
- 它具体是什么问题？
- 它从底层原理上如何导致了性能恶化？

---

### 任务三：具体优化方案

请给出可落地的优化方案，要求：
- 如需修改 SQL 写法，请给出修改后的完整 SQL；
- 如需新建或删除索引，请给出完整的 `ALTER TABLE` 语句；
- 如有其他手段（如数据类型调整、表设计建议），请说明原因；
- 对每项优化，解释其能解决哪个具体瓶颈，以及预期的性能收益方向。

---

**【作答要求】**

请将你的完整推演和方案写入答卷文件 `Level_2_MyAnswer.md` 中，完成后告知我。我将逐一审视你的逻辑链条是否严密。

**现在，请开始你的作答。**
