---
title: Java 中的 Redis (Spring Data Redis)
tags:
 - Redis
 - Java
 - SpringBoot
create_time: 2026-02-02
---

# Java 中的 Redis 使用指南

本笔记记录了如何在 Spring Boot 项目中整合 Redis，使用了 Spring Data Redis 模块。

## 1. 引入依赖 (Maven)

根据教程步骤，首先需要在 `pom.xml` 中引入 `spring-boot-starter-data-redis`。

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
<dependency>
    <groupId>org.apache.commons</groupId>
    <artifactId>commons-pool2</artifactId>
</dependency>

```

## 2. 编写配置 (YAML)
在 `application.yml` 中配置 Redis 数据源信息。

```yaml
spring:
  redis:
    host: localhost
    port: 6379
    password: 123456 # 如果没有密码则留空
    database: 0 # 指定使用的数据库索引，默认是0
```

> [!tip] 💡 提示：关于 Database
> * `spring.redis.database` 用于指定连接的数据库。
> * Redis 默认会自动创建 **16** 个数据库（索引 0-15）。
> * 数据库之间的数据是**相互隔离**的。


## 3. 编写配置类 (解决序列化问题)
默认的 `RedisTemplate` 使用 JDK 序列化，会导致在 Redis 客户端（如 GUI 工具或命令行）中看到的 Key 是乱码（如 `\xac\xed\x00`）。为了方便管理和调试，通常需要自定义配置类，使用 `StringRedisSerializer`。

```java
@Configuration
@Slf4j
public class RedisConfiguration {
    @Bean
    public RedisTemplate redisTemplate(RedisConnectionFactory redisConnectionFactory){
        log.info("开始创建redis模板类...");
        RedisTemplate redisTemplate = new RedisTemplate();
        
        // 设置连接工厂
        redisTemplate.setConnectionFactory(redisConnectionFactory);
        
        // 核心步骤：设置 Key 的序列化器
        // 默认为 JdkSerializationRedisSerializer，这里改为 StringRedisSerializer
        redisTemplate.setKeySerializer(new StringRedisSerializer());
        
        // 可选扩展：设置 Value 的序列化器 (例如使用 JSON 序列化)
        // redisTemplate.setValueSerializer(new GenericJackson2JsonRedisSerializer());
        
        return redisTemplate;
    }
}
```

## 4. 五种数据类型的操作详解
Spring Data Redis 将不同数据类型的操作封装到了不同的 Operation 接口中。

### 4.1 String 类型 (ValueOperations)
* **对应接口**：`ValueOperations`
* **获取方式**：`redisTemplate.opsForValue()`
* **应用场景**：
* **缓存**：存储对象 JSON 字符串、页面片段。
* **计数器**：视频播放量、点赞数。
* **Session共享**：存储用户登录 Token。

**常用方法与示例：**
```java
@Autowired
private RedisTemplate redisTemplate;

public void testString() {
    ValueOperations ops = redisTemplate.opsForValue();

    // 1. 设置值 set(key, value)
    ops.set("username", "zhangsan");

    // 2. 设置值并指定过期时间 (常用!)
    // set(key, value, timeout, TimeUnit)
    ops.set("code", "1234", 60, TimeUnit.SECONDS);

    // 3. 如果 key 不存在才设置 (分布式锁的基础)
    // setIfAbsent(key, value)
    Boolean isLocked = ops.setIfAbsent("lock", "1"); 

    // 4. 获取值
    Object value = ops.get("username");
}
```


### 4.2 Hash 类型 (HashOperations)
* **对应接口**：`HashOperations`
* **获取方式**：`redisTemplate.opsForHash()`
* **结构**：`Key -> (Field, Value)`
* **应用场景**：
* **购物车**：Key 为用户ID，Field 为商品ID，Value 为数量。
* **存储对象**：Key 为对象ID，Field 为属性名，Value 为属性值（修改单个属性方便）。

**常用方法与示例：**
```java
public void testHash() {
    HashOperations ops = redisTemplate.opsForHash();

    // 1. 存储单个字段 put(key, hashKey, hashValue)
    ops.put("user:1001", "name", "Jack");
    ops.put("user:1001", "age", "20");

    // 2. 获取单个字段 get(key, hashKey)
    String name = (String) ops.get("user:1001", "name");

    // 3. 获取该 Key 下所有的字段名 keys(key) -> 返回 Set
    Set<Object> keys = ops.keys("user:1001");

    // 4. 获取该 Key 下所有的值 values(key) -> 返回 List
    List<Object> values = ops.values("user:1001");
    
    // 5. 删除字段
    ops.delete("user:1001", "age");
}
```


### 4.3 List 类型 (ListOperations)
* **对应接口**：`ListOperations`
* **获取方式**：`redisTemplate.opsForList()`
* **结构**：双向链表。
* **应用场景**：
* **消息队列**：简单的生产者消费者模型。
* **最新列表**：朋友圈的时间线（Timeline），新数据 `LeftPush`，旧数据右侧挤出。

**常用方法与示例：**
```java
public void testList() {
    ListOperations ops = redisTemplate.opsForList();

    // 1. 从左侧推入 leftPushAll(key, values...)
    ops.leftPushAll("mylist", "a", "b", "c"); // 结果: c, b, a

    // 2. 从左侧推入单个 leftPush(key, value)
    ops.leftPush("mylist", "d");

    // 3. 范围查询 range(key, start, end) -> 返回 List
    // 0 到 -1 代表查询所有
    List<Object> range = ops.range("mylist", 0, -1);

    // 4. 从右侧弹出 (出队) rightPop(key)
    Object popValue = ops.rightPop("mylist");
    
    // 5. 获取长度
    Long size = ops.size("mylist");
}
```


### 4.4 Set 类型 (SetOperations)
* **对应接口**：`SetOperations`
* **获取方式**：`redisTemplate.opsForSet()`
* **特点**：无序、去重。
* **应用场景**：
* **点赞用户**：一个用户只能点赞一次，自动去重。
* **共同好友**：利用交集操作（Intersect）。
* **抽奖**：利用 `randomMember` 随机获取元素。

**常用方法与示例：**
```java
public void testSet() {
    SetOperations ops = redisTemplate.opsForSet();

    // 1. 添加元素 add(key, values...)
    ops.add("set1", "a", "b", "c", "d");
    ops.add("set2", "c", "d", "e", "f");

    // 2. 获取所有元素 members(key) -> 返回 Set
    Set<Object> members = ops.members("set1");

    // 3. 求交集 (共同好友) intersect(key1, key2)
    Set<Object> intersect = ops.intersect("set1", "set2"); // 结果: c, d

    // 4. 求并集 union(key1, key2)
    Set<Object> union = ops.union("set1", "set2");
    
    // 5. 删除元素
    ops.remove("set1", "a");
}
```


### 4.5 ZSet 类型 (ZSetOperations)
* **对应接口**：`ZSetOperations` (Sorted Set)
* **获取方式**：`redisTemplate.opsForZSet()`
* **特点**：有序、去重、每个元素带有一个 `double` 类型的 `score`。
* **应用场景**：
* **排行榜**：游戏分数排行、热搜话题排行（Score 为热度值）。
* **带权重的消息队列**。

**常用方法与示例：**
```java
public void testZSet() {
    ZSetOperations ops = redisTemplate.opsForZSet();

    // 1. 添加元素 (带分数) add(key, value, score)
    ops.add("ranking", "PlayerA", 90);
    ops.add("ranking", "PlayerB", 100);
    ops.add("ranking", "PlayerC", 85);

    // 2. 给指定元素增加分数 incrementScore
    // PlayerA 加 5 分，变为 95
    ops.incrementScore("ranking", "PlayerA", 5.0);

    // 3. 获取排行 (从小到大) range(key, start, end)
    Set<Object> range = ops.range("ranking", 0, -1);

    // 4. 获取排行 (从大到小，常用作TopN) reverseRange
    Set<Object> topList = ops.reverseRange("ranking", 0, 2); // 前3名
}
```


## 5. 通用操作 (RedisTemplate)
有些命令不属于特定数据类型，而是针对 Key 本身的操作，直接通过 `redisTemplate` 调用。

* **判断 Key 是否存在**：`Boolean hasKey = redisTemplate.hasKey("key");`
* **删除 Key**：`redisTemplate.delete("key");`
* **查看 Key 类型**：`DataType type = redisTemplate.type("key");`
* **查找 Key (模式匹配)**：`Set<String> keys = redisTemplate.keys("user:*");` (注意：生产环境禁用 keys 命令，容易阻塞)
* **设置过期时间**：`redisTemplate.expire("key", 10, TimeUnit.MINUTES);`

## 6. 扩展：StringRedisTemplate

> [!abstract] 扩展知识
> 在实际开发中，如果你的 Key 和 Value 都是 String 类型（例如存 JSON 串），Spring 提供了一个预配置好的类：`StringRedisTemplate`。

* **区别**：
* `RedisTemplate`：默认 Key/Value 是 Object，序列化稍显麻烦。
* `StringRedisTemplate`：继承自 RedisTemplate，Key/Value 默认就是 `String`，序列化器默认就是 `StringRedisSerializer`。

* **建议**：如果是简单的字符串读写，优先注入 `StringRedisTemplate` 使用，省去了自定义配置类的步骤。
```java
@Autowired
private StringRedisTemplate stringRedisTemplate;

public void test() {
    stringRedisTemplate.opsForValue().set("simple", "hello");
}
```