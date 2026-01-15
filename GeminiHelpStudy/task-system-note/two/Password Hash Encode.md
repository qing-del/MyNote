## 📝 笔记


> **安全金律：**
> 
> 1. **数据库不存明文**：防止数据库泄露导致用户账号集体被盗。
>     
> 2. **Entity 保持纯净**：Getter/Setter 不写逻辑，加密逻辑属于 Service 层。
>     
> 3. **BCrypt 算法**：自带“盐值 (Salt)”，同样的密码每次生成的哈希值都不同，安全性极高。
>     

---

## 详细内容如下：

### 1. 什么是 Hash 加密？

简单来说，Hash（哈希）不是传统的“加密”，而是一种**不可逆的单向函数**。

- **明文密码**：`123456`
    
- **Hash 后的值 (BCrypt)**：`$2a$10$Eixza//n.A9vg...`（一串随机乱码）
    

**它的特点是：**

- **单向性**：你可以轻易把 `123456` 变成乱码，但即使黑客拿到了乱码，也几乎无法反向推导出它是 `123456`。
    
- **验证方式**：当用户登录时，我们将用户输入的明文再次进行 Hash 运算，如果得到的结果和数据库里的乱码一致，就证明密码正确。
    

---

### 2. 应该写在 Getter/Setter 里吗？

**绝对不要。** 这是一个初学者很容易踏入的陷阱。

在 Java 开发（尤其是 Spring Boot）中，我们遵循 **“实体类 (Entity) 应该是简单的”** 这一原则：

- **Getter/Setter 的职责**：只负责数据的存取，不应该包含任何业务逻辑。
    
- **原因**：如果你在 `setPassword` 里写了加密逻辑，那么当你从数据库查询用户时，MyBatis 调用 `setPassword` 会把已经加密过的密码**再次加密**，导致你永远无法登录成功。
    

---

### 3. 正确的做法在哪里？

根据你的 `JavaStudy.md` 路线图，我们应该在 **Service 层** 处理加密。

#### 第一步：在配置类中定义加密器

在你的 `SecurityConfig.java`（我们马上要写的）里，会定义一个 `BCryptPasswordEncoder`：

```Java
@Bean
public PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder(); // 这是目前最安全、最主流的哈希算法
}
```

#### 第二步：在注册逻辑中使用 (Service 层)

当你创建新用户时，在 `UserServiceImpl` 里手动触发加密：

```Java
@Service
public class UserServiceImpl implements UserService {
    @Autowired
    private PasswordEncoder passwordEncoder;

    @Override
    public void register(User user) {
        // 1. 获取前端传来的明文密码
        String rawPassword = user.getPassword();
        // 2. 加密
        String encodedPassword = passwordEncoder.encode(rawPassword);
        // 3. 将加密后的乱码存回对象，再交给 Mapper 存入数据库
        user.setPassword(encodedPassword);
        userMapper.insert(user);
    }
}
```

---