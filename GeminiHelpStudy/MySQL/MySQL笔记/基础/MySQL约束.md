---
title: MySQL约束
tags:
 - MySQL
 - 约束
 - 基础
create_time: 2026-02-26
---

## 概述
- **概念**：约束是作用于表中字段上的规则，用于限制存储在表中的数据
- **目的**：保证数据库中数据的正确、有效性和完整性
- **分类**

![[MySQL约束表.png]]
> [!warning] 注意
> 约束是作用于表字段上的，可以在**创建表**/**修改表**的时候添加约束

---

### 约束示例
|字段名|字段含义|字段类型|约束条件|约束关键字|
|:---:|:---:|---|---|---|
|id|ID唯一标识|int|主键，并且自动增长|`PRIMARY KEY`,`AUTO_INCREMENT`|
|name|姓名|varchar(10)|不为空、唯一|`NOT NULL`,`UNIQUE`|
|age|年龄|int|大于0且小于等于120|`CHECK`|
|status|状态|char(1)|如果没有指定该值，默认为1|`DEFAULT`|
|gender|性别|char(1)|无||

- **示例代码**
```sql
create table user(
	id int primary key auto_increment commit '主键',
	name varchar(10) not null unique commit '姓名',
	age int check( age>0 && age<=120 ) commit '年龄',
	status char(1) default '1' commit '状态',
	gender char(1) commit '性别'
) commit '用户表', engine=InnoDB;
```

---

### 外键约束
- **作用**：让两张表建立连接，保证数据**一致性**

![[MySQL外键约束概念图.png]]
> 不进行**外键约束**，在数据库层面为建立连接，无法保证数据一致性

#### 添加外键SQL模板
```sql
-- 创建表时添加外键
CREATE TABLE 表名(
	字段名 数据类型...,
	...
	[CONSTRAINT] [外键名称] FOREIGN KEY(外键字段名) REFERENCES 主表(主表列名)
);

-- 在已有表上加入外键
ALTER TABLE 表名 ADD CONSTRAINT 外键名称 FOREIGN KEY(外键字段名) REFERENCES 主表(主表列名);
```

> [!info] 删除外键模板
> ```sql
> -- MySQL所有版本通用
> ALTER TABLE 表名 DROP FOREIGN KEY 外键名称;
>
> -- （MySQL 8.0.16+）
> ALTER TABLE 表名 DROP CONSTRAINT 外键名称;
> ```

#### 外键约束下的删除/更新行为
- 删除/更新**父表**中的数据行时，会检查**子表**中是否有数据，有数据一般会有以下几种**行为模式**；

![[MySQL外键约束行为模式.png]]

- **设置行为语句**
```sql
ALTER TABLE 表名 ADD CONSTRAINT 外键名称 FOREIGN KEY(外键字段名) REFERENCES 主表(主表列名) ON UPDATE [更新操作行为模式] ON DELETE [删除操作行为模式];
```