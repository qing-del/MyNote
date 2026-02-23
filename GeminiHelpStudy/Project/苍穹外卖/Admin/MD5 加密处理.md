# 完善登录功能

> [!warning] ⚠注意
> 使用`MD5`加密是只有单向的
> 明文密码 -> 密文
> 不支持反过来计算的

---

### 步骤
1. 修改数据库中明文密码，改为`MD5`加密之后的密文
2. 修改 Java 代码，前端提交的密码进行`MD5`加密后再和数据库中密码比对

---

### 先获取原先用户的密文
- 示例函数
```java
private static void md5EncodeAdmin() {
	String password = "123456";
    password = DigestUtils.md5DigestAsHex(password.getBytes());  
    System.out.println(password);  
}
```

- 使用 SQL 语句进行修改数据
```sql
update into sky_take_out set password = ${encodePassword} where username = 'admin';
```