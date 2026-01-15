# 基础知识

## 创建表
```sql
create table player_status (  
     id bigint primary key auto_increment comment 'id',  
     spirit int default 1 comment '精神属性',  
     body int default 1 comment '肉体属性',  
     current_exp bigint default 0 comment '当前经验',  
     total_exp bigint default 0 comment '总经验',  
     level int default 1 comment '等级',  
     update_time datetime default current_timestamp on update  current_timestamp comment '更新时间'  
) ENGINE = InnoDB comment '玩家状态';
```

## 表插入
```sql
INSERT INTO `player_status` (id, spirit, body, current_exp, total_exp, level) VALUES (1, 1, 1, 0, 0, 1);
```

## 表更新
```sql
update task set status=1 where id=1;
```

## 表查询
```sql
select * from player_status where id=1;
```

### 限制查询和排序
```sql
select * from task where user_id=1 and status=1 order by create_time limit 5;
```

---

# 表结构修改

## 表结构修改 (ALTER)

### 增加列 (Add Column)
用于给已存在的表追加新字段。
```sql
-- 语法：ALTER TABLE 表名 ADD COLUMN 列名 类型 [约束] [注释];
-- 示例：给 player_status 表增加一个 user_id 字段
ALTER TABLE player_status 
ADD COLUMN user_id BIGINT UNIQUE COMMENT '关联的用户ID';
````

### 添加外键约束 (Foreign Key)

用于限制两个表的关系，保证“从表”里的 ID 必须在“主表”里存在。

```SQL
-- 语法：ALTER TABLE 从表 ADD CONSTRAINT 约束名 FOREIGN KEY (从表列) REFERENCES 主表(主表列);
-- 示例：保证 player_status 里的 user_id 必须是 users 表里有的 id
ALTER TABLE player_status
ADD CONSTRAINT fk_player_user
FOREIGN KEY (user_id) REFERENCES users(id);
```

---