
# 笔记：Swagger 2 升级 OpenAPI 3 迁移速查表 (Spring Boot 3 + Knife4j 4.x)

**背景**：
由于课程使用的 Spring Boot 版本较旧（配合 Swagger 2/Springfox），而我的环境是 Spring Boot 3（必须配合 OpenAPI 3/SpringDoc）。因此，当课程老师使用 Swagger 注解时，我需要实时转换为 OpenAPI 3 的注解。

---

## 1. 核心概念对比

| 特性 | 旧版 (Swagger 2) | 新版 (OpenAPI 3) |
| :--- | :--- | :--- |
| **底层依赖** | `springfox-boot-starter` | `springdoc-openapi-starter-webmvc-ui` (或 `knife4j-openapi3-...`) |
| **核心包名** | `io.swagger.annotations` | `io.swagger.v3.oas.annotations` |
| **配置类** | `Docket` | `OpenAPI` Bean |
| **描述标准** | Swagger 2.0 | OpenAPI 3.0 |

---

## 2. 注解映射速查表 (常用)

### 2.1 实体类 (DTO/VO/Entity)
用于描述数据模型及其字段。

| 作用 | 旧注解 (Swagger 2) | 新注解 (OpenAPI 3) | 示例 (新版) |
| :--- | :--- | :--- | :--- |
| **类描述** | `@ApiModel("描述")` | **`@Schema(description = "...")`** | `@Schema(description = "员工登录DTO")` |
| **字段描述** | `@ApiModelProperty("描述")` | **`@Schema(description = "...")`** | `@Schema(description = "用户名")` |
| **必填标记** | `@ApiModelProperty(required=true)` | `@Schema(requiredMode = Schema.RequiredMode.REQUIRED)` | |

### 2.2 控制层 (Controller)
用于描述接口分组和具体接口功能。

| 作用 | 旧注解 (Swagger 2) | 新注解 (OpenAPI 3) | 示例 (新版) |
| :--- | :--- | :--- | :--- |
| **接口分组(类上)** | `@Api(tags = "...")` | **`@Tag(name = "...")`** | `@Tag(name = "员工管理接口")` |
| **接口说明(方法上)** | `@ApiOperation("...")` | **`@Operation(summary = "...")`** | `@Operation(summary = "员工登录")` |
| **忽略参数/接口** | `@ApiIgnore` | **`@Hidden`** | `@Hidden` |
| **参数描述** | `@ApiImplicitParam` | `@Parameter` | `@Parameter(name = "id", description = "主键")` |

---

## 3. 配置类改造 (WebMvcConfiguration)

在配置类中，**必须删除**所有 `Docket` 相关的旧代码，替换为新的 `OpenAPI` 配置。

**旧代码 (删除):**
```java
// ❌ 删除 springfox 包的引用
// ❌ 删除 Docket 方法
public Docket docket() { ... }
````

**新代码 (添加):**
```Java
import io.swagger.v3.oas.models.OpenAPI;
import io.swagger.v3.oas.models.info.Info;

@Bean
public OpenAPI publicApi() {
    return new OpenAPI()
            .info(new Info()
                .title("苍穹外卖项目接口文档")
                .version("2.0")
                .description("基于 Spring Boot 3 + Spring AI 重构"));
}
```

---

## 4. 快速重构技巧 (IDEA 正则替换)

在 IDEA 中使用 `Ctrl + Shift + R` 进行全局替换，记得勾选 **Regex** (正则表达式模式)。

**场景 1：实体类描述 (@ApiModel -> @Schema)**
- **搜索**: `@ApiModel\((?:value\s*=\s*)?"(.*?)"\)`
- **替换**: `@Schema(description = "$1")`

**场景 2：字段描述 (@ApiModelProperty -> @Schema)**
- **搜索**: `@ApiModelProperty\((?:value\s*=\s*)?"(.*?)"\)`
- **替换**: `@Schema(description = "$1")`

**场景 3：包名替换**
- **搜索**: `import io.swagger.annotations.*;`
- **替换**: `import io.swagger.v3.oas.annotations.media.Schema;` (实体类)
- **替换**: `import io.swagger.v3.oas.annotations.tags.Tag;` (Controller类)
- **替换**: `import io.swagger.v3.oas.annotations.Operation;` (Controller方法)

**场景 4：分包名生成文档**
- **操作**： 引入`GroupedOpenApi`
- 修改代码：
```Java
/**  
 * 注册自定义拦截器  
 * @param registry  
 */  
public void addInterceptors(InterceptorRegistry registry) {  
    log.info("Start registering custom interceptors...");  
    registry.addInterceptor(jwtTokenAdminInterceptor)  
            .addPathPatterns("/admin/**")  
            .excludePathPatterns("/admin/employee/login")  
            // 必须添加以下放行路径，否则文档无法加载  
            .excludePathPatterns(  
                    "/doc.html",  
                    "/webjars/**",  
                    "/v3/api-docs/**",  
                    "/swagger-ui/**",  
                    "/swagger-ui.html"  
            );  
}

/**
 * 1. 配置全局信息 (标题、版本、描述)
 * 这些信息会显示在所有分组的文档顶部
 */
@Bean
public OpenAPI customOpenAPI() {
    return new OpenAPI()
            .info(new Info()
                    .title("苍穹外卖项目接口文档")
                    .version("2.0")
                    .description("苍穹外卖项目接口文档"));
}

/**
 * 2. 配置管理端接口分组
 * 扫描 package: com.sky.controller.admin
 */
@Bean
public GroupedOpenApi adminApi() {
    return GroupedOpenApi.builder()
	        .group("管理端接口") // 分组名称，会在UI右上角下拉框显示
	        .packagesToScan("com.sky.controller.admin") // 核心：只扫描 admin 包
	        .build();
}

/**
 * 3. 配置用户端接口分组
 * 扫描 package: com.sky.controller.user
 */
@Bean
public GroupedOpenApi userApi() {
    return GroupedOpenApi.builder()
            .group("用户端接口") // 分组名称
            .packagesToScan("com.sky.controller.user") // 核心：只扫描 user 包
            .build();
}

/**  
 * 扩展消息转换器  
 * @param converters  
 */  
public void extendMessageConverters(List<HttpMessageConverter<?>> converters) {  
    log.info("Extended message converter...");  
    MappingJackson2HttpMessageConverter converter = new MappingJackson2HttpMessageConverter();  
    converter.setObjectMapper(new JacksonObjectMapper());  
    converters.add(0, converter);  
  
    // 为了文档能够正常生成  
    converters.add(0, new org.springframework.http.converter.ByteArrayHttpMessageConverter());  
}
```

---

## 5. 常见报错自查
- **报错**: `Cannot resolve symbol 'Docket'`
    - **原因**: 依赖包已升级，旧包不存在。
    - **解法**: 删掉 `Docket` 方法，换成 `OpenAPI` Bean。

- **报错**: 启动时 `DocumentationPluginsBootstrapper` 空指针
    - **原因**: 这是 Spring Boot 2.6+ 与旧版 Swagger 的经典兼容性问题。
    - **解法**: 既然升级到了 Spring Boot 3，直接确保 `springfox` 依赖已完全移除，只保留 `knife4j-openapi3-jakarta-spring-boot-starter`。

---

### 💡 额外建议
在后续课程中，如果看到老师用了其他你在表格里找不到的注解，通常只需要去 `io.swagger.v3.oas.annotations` 包下找找看名字类似的即可，OpenAPI 3 的命名通常比 Swagger 2 更直观（例如 `ApiIgnore` 变成了 `Hidden`，意为“隐藏”）。

---

### 完整配置类代码
```java
// WebMvcConfiguration.java
// com.[工组].config -- 包下

import com.fasterxml.jackson.databind.ObjectMapper;  
import com.sky.interceptor.JwtTokenAdminInterceptor;  
import io.swagger.v3.oas.models.OpenAPI;  
import io.swagger.v3.oas.models.info.Info;  
import lombok.extern.slf4j.Slf4j;  
import org.springframework.beans.factory.annotation.Autowired;  
import org.springframework.context.annotation.Bean;  
import org.springframework.context.annotation.Configuration;  
import org.springframework.web.servlet.config.annotation.InterceptorRegistry;  
import org.springframework.web.servlet.config.annotation.ResourceHandlerRegistry;  
import org.springframework.web.servlet.config.annotation.WebMvcConfigurationSupport;  
  
/**  
 * 配置类，注册web层相关组件  
 */  
@Configuration  
@Slf4j  
public class WebMvcConfiguration extends WebMvcConfigurationSupport {  
  
    @Autowired  
    private JwtTokenAdminInterceptor jwtTokenAdminInterceptor;  
  
    private static final ObjectMapper objectMapper = new ObjectMapper();  
  
    /**  
     * 注册自定义拦截器  
     * @param registry  
     */  
    protected void addInterceptors(InterceptorRegistry registry) {  
        log.info("Start registering custom interceptors...");  
        registry.addInterceptor(jwtTokenAdminInterceptor)  
                .addPathPatterns("/admin/**")  
                .excludePathPatterns("/admin/employee/login");  
    }  
  
    /**  
     * 创建API信息  
     */  
    @Bean  
    public OpenAPI publicApi() {  
        log.info("Start creating API information...");  
        return new OpenAPI()  
                .info(new Info()  
                        .title("苍穹外卖 (AI智能版)")  
                        .version("1.0")  
                        .description("基于 Spring Boot 3 + Spring AI 重构的实验性项目"));  
    }  
  
    /**  
     * 设置静态资源映射  
     * @param registry  
     */  
    protected void addResourceHandlers(ResourceHandlerRegistry registry) {  
        log.info("Start setting up static resource mapping...");  
        registry.addResourceHandler("/doc.html").addResourceLocations("classpath:/META-INF/resources/");  
        registry.addResourceHandler("/webjars/**").addResourceLocations("classpath:/META-INF/resources/webjars/");  
    }  
}
```