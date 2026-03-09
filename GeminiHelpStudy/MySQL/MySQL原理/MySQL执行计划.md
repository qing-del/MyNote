---
title: MySQL执行计划
tags:
 - SQL优化
 - MySQL
 - 执行计划
create_time: 2026-03-08
---

## 目录
- [[#引入案例]]
- [[#访问方法（access method）的概念]]
	- [[#const]]
	- [[#ref]]

---

## 引入案例
```sql
CREATE TABLE single_table ( 
	id INT NOT NULL AUTO_INCREMENT, 
	key1 VARCHAR(100), 
	key2 INT, 
	key3 VARCHAR(100), 
	key_part1 VARCHAR(100), 
	key_part2 VARCHAR(100), 
	key_part3 VARCHAR(100), 
	common_field VARCHAR(100), 
	PRIMARY KEY (id), KEY idx_key1 (key1), 
	UNIQUE KEY idx_key2 (key2), 
	KEY idx_key3 (key3), 
	KEY idx_key_part(key_part1, 
	key_part2, key_part3) 
) Engine=InnoDB CHARSET=utf8;
```

---

## 访问方法（access method）的概念
- InnoDB中访问方法大致分为两种：
	- **全表扫描**
	- **使用索引**
		- 主键、唯一二级索引等值查询
		- 普通二级索引的等值查询
		- 索引列的范围查询
		- 直接扫描整个索引

### const
- 可以使用主键列（或者唯一二级索引）精确定位一条记录（唯一性）
```sql
SELECT * FROM single_table WHRER id = 1438;
```
- 这个时候MySQL会利用这个**唯一性**在索引树中快速定位到数据行

> [!tip] 关联思考
> 在这个查找的==底层==是 `O(log n)` 的时间复杂度，因为是基于 B+树
> 但是在==工程思想==上，你可以是将其视作 `O(1)` 的时间复杂度，因为只需要**常数次索引查询**即可找到数据


### ref
- 使用普通二级索引的时候，有可能这个二级索引的等值搜索结果并**不是唯一**的（丢失**唯一性**）
- 就会需要进行多次回表查询
> [!warning] PS：一次回表相当于一次随机的磁盘IO