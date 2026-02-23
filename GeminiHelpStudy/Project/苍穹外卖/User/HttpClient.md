## 导入依赖

> [!tip] 💡提示
> 在之前的项目中导入了`阿里云OSS`的依赖，其中就已经带有`HttpClient`的依赖了
> 其实可以不用修改`pom.xml`文件来导入依赖

### 入门案例
- `Get`请求入门案例
```Java
@Test  
public void testGet() throws IOException {  
    // 1. 获取 HttpClient 对象  
    CloseableHttpClient httpClient = HttpClients.createDefault();  
    // 2. 创建 HttpGet 请求，设置url路径  
    HttpGet httpGet = new HttpGet("http://localhost:8080/user/shop/status");  
    // 3. 执行请求  
    CloseableHttpResponse response = httpClient.execute(httpGet);  
    // 4. 解析一下响应状态码  
    int code = response.getStatusLine().getStatusCode();  
    System.out.println("响应状态码：" + code);  
    // 5.解析响应数据  
    HttpEntity entity = response.getEntity();  
    String body = EntityUtils.toString(entity);  
    System.out.println("响应数据：" + body);  
    // 6.关闭流  
    response.close();  
    httpClient.close();  
}
```
- `Post`请求入门
```Java
@Test  
public void testPost() throws IOException {  
    // 创建 HttpClient 对象  
    CloseableHttpClient httpClient = HttpClients.createDefault();  
    // 创建 httpPost 请求，设置 url 路径  
    HttpPost httpPost = new HttpPost("http://localhost:8080/admin/employee/login");  
  
    // 创建请求参数  
    ObjectNode objectNode = objectMapper.createObjectNode();  
    objectNode.put("username", "admin");  
    objectNode.put("password", "123456");  
  
  
    StringEntity entity = new StringEntity(objectNode.toString(), "utf-8");  
    entity.setContentType("application/json");  // 设置请求编码 不设置会返回415状态码  
    httpPost.setEntity(entity);  
  
    // 执行请求  
    CloseableHttpResponse response = httpClient.execute(httpPost);  
  
    // 解析响应数据  
    int code = response.getStatusLine().getStatusCode();  
    System.out.println("响应状态码：" + code);  
    // 解析响应数据  
    HttpEntity responseEntity = response.getEntity();  
    String body = EntityUtils.toString(responseEntity);  
    System.out.println("响应数据：" + body);  
  
    // 关闭流  
    response.close();  
    httpClient.close();  
}
```