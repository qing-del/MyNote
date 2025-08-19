# MyBatis
## 目录
- [简介](#简介)
- [依赖和配置](#依赖和配置)
- [数据库连接池](#数据库连接池)

### 简介
* **MyBatis** 是一款优秀的持久层框架，用于**简化 JDBC** 的开发
* **官网：** https://mybatis.org/mybatis-3/zh_CN/index.html

### 依赖和配置
1. 准备工作：
    1. 创建 SpringBoot 项目、引入 MyBatis 相关依赖
    2. 准备数据库表 user、实体类 User
    ```sql
    CREATE TABLE `user` (
        id int(11) NOT NULL AUTO_INCREMENT comment '主键，用户ID',
        username varchar(255) NOT NULL comment '用户名',
        password varchar(255) NOT NULL comment '密码',
        name varchar(255) NOT NULL comment '姓名',
        age int(11) NOT NULL comment '年龄',
    ) comment '用户表';
    ```
    ```java
    public class User { 
        private Integer id;
        private String username;
        private String password;
        private String name;
        private Integer age;
    }
    ```
    3. 配置 MyBatis（在application.properties 中数据库连接信息）
    ```properties
    spring.datasource.url=jdbc:mysql://localhost:3306/web
    spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver
    spring.datasource.username=root
    spring.datasource.password=1234
    ```
    ```java
    @Mapper
    public interface UserMapper {
        @Select("select * from user")
        public List<User> findAll();
    }
    ```
    > 提示：Mybatis的持久层接口命名规范为 XxxMapper，也称为 Mapper 接口。

### MyBatis 日志输出
* 在默认情况下，MyBatis 执行 SQL 语句时，不会将 SQL 语句打印在控制台<br>可以在配置文件中加入如下配置
```properties
mybatis.configuration.log-impl=org.apache.ibatis.logging.stdout.StdOutImpl
```

### 数据库连接池
* MyBatis 默认使用 Hikari 作为数据库连接池，但也可以使用其他连接池，如 C3P0、DBCP、Druid、HikariCP 等
* 下面以 Druid 为例
    1. 添加依赖
    ```xml
    <dependency>
        <groupId>com.alibaba</groupId>
        <artifactId>druid-spring-boot-starter</artifactId>
        <version>1.2.19</version>
    </dependency>
    ```
    2. 配置 **application.properties**
    ```properties
    #更换数据库连接池
    spring.datasource.type=com.alibaba.druid.pool.DruidDataSource
    ```