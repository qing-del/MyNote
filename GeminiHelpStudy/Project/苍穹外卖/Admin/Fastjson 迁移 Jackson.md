# 🔄 Fastjson vs Jackson 迁移速查表

**背景**：

- **场景**：Spring Boot 默认集成 Jackson，为了避免依赖冲突（Dependency Convergence）和序列化标准不统一，需要将 Fastjson 替换为 Jackson。
- **核心对象**：`JSONObject` (Fastjson) ➡️ `ObjectNode` (Jackson)。
- **核心工具**：`JSON` (Fastjson 静态方法) ➡️ `ObjectMapper` (Jackson 实例方法)。

## 1. 核心映射表 (Cheat Sheet)

|**操作类型**|**Fastjson (旧) 🔴**|**Jackson (新) 🟢**|**⚠️ 注意事项**|
|---|---|---|---|
|**导包**|`import com.alibaba.fastjson.*;`|`import com.fasterxml.jackson.databind.*;`<br><br>  <br><br>`import com.fasterxml.jackson.databind.node.*;`|建议在类中定义 `private static final ObjectMapper objectMapper = new ObjectMapper();`|
|**对象转 JSON**|`JSON.toJSONString(obj)`|`objectMapper.writeValueAsString(obj)`|Jackson 会抛出 **Checked Exception**，必须用 `try-catch` 或 `throws`。|
|**JSON 转对象**|`JSON.parseObject(json, User.class)`|`objectMapper.readValue(json, User.class)`|同上，需处理异常。|
|**JSON 转 List**|`JSON.parseArray(json, User.class)`|`objectMapper.readValue(json, new TypeReference<List<User>>(){})`|泛型反序列化稍微复杂一点。|
|**创建 JSON 对象**|`new JSONObject()`|`objectMapper.createObjectNode()`|Jackson 不推荐直接 `new`，而是用工厂方法。|
|**添加字段**|`jsonObj.put("key", "val")`|`objectNode.put("key", "val")`|API 基本一致。|
|**获取 String**|`jsonObj.getString("key")`|`node.get("key").asText()`|**重要**：如果 key 不存在，Jackson 的 `get()` 返回 `null`，直接调用 `asText()` 会空指针。|
|**获取 Int**|`jsonObj.getIntValue("key")`|`node.get("key").asInt()`||
|**获取嵌套对象**|`jsonObj.getJSONObject("key")`|`(ObjectNode) node.get("key")`|Jackson 获取的是通用的 `JsonNode`，需要强转或继续操作。|

---

## 2. 常见场景实战

### 场景一：手动构建 JSON (用于 HttpClient 参数)

**🔴 Fastjson:**
```Java
JSONObject json = new JSONObject();
json.put("username", "admin");
json.put("password", "123456");
String param = json.toString();
```

**🟢 Jackson:**
```Java
// 建议复用 objectMapper 实例
ObjectNode json = objectMapper.createObjectNode();
json.put("username", "admin");
json.put("password", "123456");
String param = json.toString(); // 输出紧凑的 JSON 字符串
```

### 场景二：解析 JSON 字符串 (用于微信支付回调)

**🔴 Fastjson:**
```Java
String response = "{\"openid\":\"ox123\", \"code\":200}";
JSONObject obj = JSON.parseObject(response);
String openid = obj.getString("openid");
```

**🟢 Jackson:**
```Java
String response = "{\"openid\":\"ox123\", \"code\":200}";
// 1. 解析为树节点 (JsonNode 是只读的通用节点)
JsonNode node = objectMapper.readTree(response);

// 2. 安全获取值
if (node.has("openid")) {
    String openid = node.get("openid").asText();
}
```

> [!error] 扩展问题：
> 如果你想用`String.valueOf(node.get("openid"))`来平替之前的`obj.getString("openid")`
> 那就会产生一个问题，`get("key")`得到的是一个`TestNode`（假设内容是：“123456”）
> 直接使用`String.valueof()`转，会得到**带有双引号的结果**->`"123456"`
> 而可能需要的结果是`123456`

### 场景三：复杂对象序列化 (POJO 转 String)

**🔴 Fastjson:**
```Java
UserDTO user = new UserDTO("sky", 18);
String json = JSON.toJSONString(user);
```

**🟢 Jackson:**
```Java
UserDTO user = new UserDTO("sky", 18);
try {
    String json = objectMapper.writeValueAsString(user);
} catch (JsonProcessingException e) {
    e.printStackTrace();
    // 实际业务中通常抛出自定义异常，如: throw new RuntimeException("序列化失败");
}
```

---

## 3. 🚨 避坑指南 (必读)

1. **异常处理差异**：
    - Fastjson 出了问题通常静默或抛 RuntimeException。
    - Jackson 非常严谨，`writeValueAsString` 和 `readValue` 都会抛出 `JsonProcessingException`，**如果你不想满屏 `try-catch`，建议封装一个全局工具类 `JacksonUtil`**。
        
2. **`get()` 的空指针陷阱**：
    - Jackson 的 `jsonNode.get("not_exist_key")` 会返回 `null`。
    - ❌ 错误写法：`jsonNode.get("key").asText()` （如果 key 不存在直接 NulPointException）。
    - ✅ 正确写法：`jsonNode.path("key").asText()` （`path` 方法更安全，不存在时返回 MissingNode 而不是 null，转换时会给默认空值）。

3. **驼峰与下划线**：
    - 如果微信接口返回的是 `{"nick_name": "sky"}` (下划线)，而你的 Java 对象是 `nickName` (驼峰)。
    - Fastjson 有时会自动匹配。
    - Jackson 默认不匹配，需要在属性上加注解：`@JsonProperty("nick_name")`。

---

> 💡 **Obsidian 提示**：你可以利用 Obsidian 的 `Callout` 功能来高亮这些代码块，或者给 `Jackson` 和 `Fastjson` 打上不同的 Tag 方便检索。