# JDBC
## 目录
[java 程序操作数据库](#java-程序操作数据库)
[示例一](#示例一)
[示例二](#示例二)

### java 程序操作数据库
* **简介**： <br>目前主流的框架有 MyBatis、MyBatisPlus、Hibernate、Spring Data JPA 等<br>但是其实都是封装了 JDBC 的操作<br>JDBC 是 Java 数据库连接（Java Database Connectivity）的缩写，是 Java 语言中用于连接和操作数据库的 API<br>JDBC 提供了 **Mysql、Oracle、SqlServer** 等的数据库实现驱动
* **依赖**： 
    ```xml
    <!-- Maven 仓库地址 -->
    <dependency>
        <groupId>com.mysql</groupId>
        <artifactId>mysql-connector-j</artifactId>
        <version>8.0.33</version>
    </dependency>
    ```

#### 示例一
* 我们编写一个 JDBC 程序，执行 update 语句<br>内容：**update user set age = 25 where id = 1**
```java
public class JDBCDemo { 
    public void update() throws Exception {
        //1. 注册驱动
        Class.forName("com.mysql.cj.jdbc.Driver");
        //2. 获取连接
        String url = "jdbc:mysql://localhost:3306/web01";
        String user = "root";
        String password = "1234";
        Connection conn = DriverManager.getConnection(url, user, password);
        //3. 获取SQL语句执行对象
        Statement stmt = conn.createStatement();
        //4. 执行SQL
        int i = stmt.executeUpdate("update user set age = 25 where id = 1");
        //5. 释放资源
        stmt.close();
        conn.close();
    }
}
```

#### 示例二
* 需求：居于 JDBC 执行如下 select 语句，将查询姐u共封装到 User 对象中
* SQL：**select * from user where username = 'daqiao' and password = '123456'**
```java
public class Demo02JDBC { 
    public static void main(String[] args) throws Exception {
        //1. 注册驱动
        Class.forName("com.mysql.cj.jdbc.Driver");
        //2. 获取连接
        String url = "jdbc:mysql://localhost:3306/web01";
        String user = "root";
        String password = "1234";
        Connection conn = DriverManager.getConnection(url, user, password);
        //3. 获取数据库操作对象
        Statement stmt = conn.createStatement();
        //4. 执行SQL语句（采用预编译的方法）
        String sql = "select * from user where username = ? and password = ?";
        PreparedStatement pstmt = conn.prepareStatement(sql);
        pstmt.setString(1, "daqiao");
        pstmt.setString(2, "123456");
        ResultSet rs = pstmt.executeQuery();    // 这里获取到的是一个集合
        //5. 处理结果集
        while (rs.next()) {     //假设下一个内容是合法的  这个函数将会返回 true（boolean 值）
            int id = rs.getInt("id");
            ...
        }
        //6. 释放资源
        rs.close();
        pstmt.close();
        conn.close();
    }
}
```
> 注意：预编译的SQL语句可以防止SQL注入攻击。