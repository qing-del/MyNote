---
tags:
  - Java
  - SpringBoot
  - FileUpload
  - AliyunOSS
  - Backend
created: 2026-01-18
---

# Java Web 文件上传指南

## 1. 简介

**文件上传**是指将本地图片、视频、音频等文件上传到服务器，供其他用户浏览或下载的过程。这在现代应用中非常常见（如朋友圈发布、头像修改等）。

### 1.1 前端三要素

要在前端实现文件上传，HTML 表单必须满足以下三个关键条件：

1.  **Method**：必须设置为 `post`，因为文件数据量通常较大，GET 请求无法承载。
2.  **Enctype**：必须设置为 `multipart/form-data`。
    * 如果不设置，默认为 `application/x-www-form-urlencoded`，此时只会提交文件名，而不会提交文件内容。
3.  **Input Type**：必须包含 `<input type="file">` 组件。

**代码示例：**

```html
<form action="/upload" method="post" enctype="multipart/form-data">
    姓名：<input type="text" name="username"> <br>
    年龄：<input type="text" name="age"> <br>
    图像：<input type="file" name="file"> <br>
    <input type="submit" value="上传文件" name="submit">
</form>

```

### 1.2 服务器端处理 (Spring Boot)

在 Spring Boot 中，使用 `MultipartFile` 接口来接收上传的文件。

```java
@Slf4j
@RestController
public class UploadController {

    @PostMapping("/upload")
    // 若前端 input 的 name 属性不是 "file"，需使用 @RequestParam("img") 指定
    public Result handleFileUpload(String name, Integer age, MultipartFile file) {
        log.info("接收到文件：{}", file.getOriginalFilename());
        return Result.success();
    }
}

```

> [!INFO] 底层细节
> 在文件上传过程中，服务器会在**临时文件夹 (temp)** 中暂存上传的文件。当请求处理结束（Request 结束）后，如果未将文件转存，临时文件夹中的内容会被自动删除。

---

## 2. 本地存储方案

将文件直接存储在应用服务器的本地磁盘上。

### 2.1 核心代码

使用 Spring 提供的 `transferTo` 方法将临时文件写入磁盘：

```java
// 将文件转存到本地磁盘 D:/images/ 目录下
file.transferTo(new File("D:/images/" + file.getOriginalFilename()));

```

### 2.2 唯一文件名处理 (UUID)

为了防止不同用户上传同名文件导致**文件覆盖**，通常使用 UUID 生成唯一文件名。

```java
// 1. 获取原始文件名
String originalFilename = file.getOriginalFilename();

// 2. 构造唯一文件名：UUID + 原始后缀名
String newFileName = UUID.randomUUID().toString() + 
                     originalFilename.substring(originalFilename.lastIndexOf("."));
    
// 3. 保存文件
file.transferTo(new File("D:/images/" + newFileName));

```

### 2.3 文件大小限制

Spring Boot 默认限制单个文件大小为 **1MB**。如果超过限制会抛出异常。需在 `application.yml` 中配置：

```yaml
spring:
  servlet:
    multipart:
      # 最大单个文件大小
      max-file-size: 10MB
      # 单次请求最大总大小（即多文件上传时的总和）
      max-request-size: 100MB

```

---

## 3. 阿里云 OSS 云存储方案

### 3.1 为什么不推荐本地存储？

本地存储方案在实际生产环境中存在明显弊端：

1. **无法直接访问**：浏览器无法直接通过 URL 访问服务器本地磁盘资源（需要配置静态资源映射）。
2. **单点故障**：如果服务器磁盘损坏，数据将永久丢失。
3. **扩容困难**：服务器磁盘空间有限，随着数据量增长容易爆满。
4. **集群限制**：在分布式/集群环境中，文件存在 A 服务器，B 服务器无法读取。

### 3.2 解决方案对比

| 方案类型 | 技术选型 | 适用场景 | 说明 |
| --- | --- | --- | --- |
| **自建文件服务** | FastDFS, MinIO | 大型项目 | 需要自行维护服务器集群，可动态扩容，运维成本较高。 |
| **第三方云服务** | 阿里云 OSS, AWS S3 | 中小型项目 | 免维护，高可用，按量付费，开发便捷。 |

### 3.3 阿里云 OSS 简介

**OSS (Object Storage Service)**：对象存储服务。可以通过网络随时存储和调用包括文本、图片、音频和视频在内的各种文件。

**基本流程：**

1. 用户通过浏览器上传文件到我们的后端服务器。
2. 后端服务器接收文件后，调用阿里云 SDK 将文件上传至 OSS。
3. OSS 返回一个可公开访问的 URL。
4. 后端将该 URL 存储到数据库或返回给前端。

### 3.4 接入步骤

#### 第一步：前期准备

1. **注册与充值**：登录 [阿里云官网](https://www.aliyun.com/)，实名认证并充值。
2. **开通服务**：搜索 "对象存储 OSS" 并开通。
3. **创建 Bucket**：
* Bucket（存储空间）是存储对象的容器。
* 建议权限设置为“公共读”以便前端访问（注意敏感数据需私有）。

4. **获取 AccessKey**：
* 这是访问阿里云 API 的密钥（由 `AccessKey ID` 和 `AccessKey Secret` 组成）。
* **安全建议**：不要将 Key 直接写死在代码中，建议配置在环境变量里。
* 

> [!WARNING] 配置环境变量 (Windows CMD 示例)
> 为了安全，避免 Key 泄露到代码仓库：
> 以下是设置变量的值
> ```cmd
> set OSS_ACCESS_KEY_ID=你的AccessKeyId
> set OSS_ACCESS_KEY_SECRET=你的Secret
> ```
> 以下是让设置的变量值生效到环境变量中
> ```cmd
> setx OSS_ACCESS_KEY_ID "%OSS_ACCESS_KEY_ID%"
> setx OSS_ACCESS_KEY_SECRET "%OSS_ACCESS_KEY_SECRET%"
> ```
> 以下是验证变量值是否生效
> ```cmd
> echo %OSS_ACCESS_KEY_ID%
> echo %OSS_ACCESS_KEY_SECRET%
> ```
> 
> 配置后需重启 IDE 或命令行窗口生效。

#### 第二步：引入 SDK

参考：[Java 阿里云 SDK 官方文档](https://help.aliyun.com/zh/oss/developer-reference/oss-java-sdk/)

#### 第三步：编写工具类

将官方示例代码封装为工具类 `AliyunOSSOperator`。

*初版代码（硬编码方式）：*

```java
@Component
public class AliyunOSSOperator {
    // 这里的配置硬编码在类中，不利于维护
	private String endpoint = "...";    // bucket的域名
    private String bucketName = "...";  // bucket名称
    private String region = "...";      // bucket所属地域
    
    public String upload(byte[] bytes, String originalFilename) {
        // ... OSS 上传逻辑 ...
        // 返回拼接好的 URL
        return endpoint.split("//")[0] + "//" + bucketName + "." + endpoint.split("//")[1] + "/" + objectName;
    }
}
```

#### 第四步：Controller 调用

```java
@Autowired
private AliyunOSSOperator aliyunOSSOperator;

@PostMapping("/upload")
public Result upload(MultipartFile file) throws Exception {
    log.info("文件上传：{}", file.getOriginalFilename());
    
    // 调用工具类上传，获取网络 URL
    String url = aliyunOSSOperator.upload(file.getBytes(), file.getOriginalFilename());
    
    return Result.success(url);
}
```

---

## 4. 配置优化：从 @Value 到 @ConfigurationProperties

为了提高代码的可维护性，避免硬编码，我们通常经历以下优化过程。

### 4.1 阶段一：使用 @Value (基础)

将配置信息提取到 `application.yml`：

```yaml
aliyun:
  oss:
    endpoint: [https://oss-cn-beijing.aliyuncs.com](https://oss-cn-beijing.aliyuncs.com)
    bucketName: java-ai
    region: cn-beijing

```

在代码中使用 `@Value` 注入：

```java
@Component
public class AliyunOSSOperator {
    @Value("${aliyun.oss.endpoint}")
    private String endpoint;
    
    @Value("${aliyun.oss.bucketName}")
    private String bucketName;
    
    // ...
}

```

### 4.2 阶段二：使用 @ConfigurationProperties (进阶)

当配置项较多时，`@Value` 会导致类中充斥着大量注解，且无法复用。推荐使用配置类。
![代码关系图](aliyun_oss.jpg#200*200)

**操作步骤：**

1. 定义一个 POJO 类，对应配置文件结构。
2. 使用 `@ConfigurationProperties` 指定前缀。

```java
@Data
@Component
@ConfigurationProperties(prefix = "aliyun.oss") // 指定配置文件中的前缀
public class AliyunOSSProperties {
    private String endpoint;
    private String bucketName;
    private String region;
}

```

3. 在工具类中注入该配置对象：

```java
@Component
public class AliyunOSSOperator {
    
    @Autowired
    private AliyunOSSProperties aliyunOSSProperties; // 注入配置对象

    public String upload(byte[] bytes, String originalFilename) {
        // 通过 get 方法获取配置
		String endpoint = aliyunOSSProperties.getEndpoint();
        String bucketName = aliyunOSSProperties.getBucketName();
        String region = aliyunOSSProperties.getRegion();
        // ...
    }
}

```

---

## 5. 总结：配置注入方式对比

| 特性 | @Value | @ConfigurationProperties |
| --- | --- | --- |
| **注入方式** | 单个属性逐一注入 | 批量将属性绑定到 Bean 对象 |
| **松散绑定** | 不支持 (必须精准匹配) | 支持 (如 `user-name` 可绑定到 `userName`) |
| **适用场景** | 属性较少，无需复用的简单配置 | 属性较多，模块化配置，需要复用 |
| **推荐指数** | ⭐⭐ | ⭐⭐⭐⭐⭐ (推荐) |

---