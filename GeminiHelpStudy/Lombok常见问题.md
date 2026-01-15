# 关于Lombok中一些问题
## 1.IDEA中Lombok调试问题
* 在Spring Boot项目中，有时候运行Application.java文件时，会报如下错误：
  > E:\JavaProject\web01\tlias-web-management\src\main\java\com\example\controller\DeptController.java:55:16 java: 找不到符号  
  > 符号:   方法 getId()  
  > 位置: 类型为com.example.pojo.Dept的变量 dept

    * 解决：
        1. 检查Lombok的依赖版本 首先有些Lombok的版本只能支持到某个版本的Java语言。
        2. 尝试运行jar包 如果jar包可以运行的情况下 Lombok的编译过程是没有问题的，若是运行jar包也是失败，那么需要去检查IDEA的设置是否正确
      > 例如：
      >> 1. 是否启用注释处理
      >> 2. 注释处理器中 lombok处理器的路径是否正确


        3. 如果可以运行jar包，那么就去检查maven的配置和IDE的编译配置是否一致
        4. 可以采用增强Lombok配置（XML示例代码）
##### XML示例代码
```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-compiler-plugin</artifactId>
    <version>3.10.1</version> <!-- 升级到最新版本 -->
    <configuration>
        <release>${java.version}</release> <!-- 使用新式配置 -->
        <annotationProcessorPaths>
            <path>
                <groupId>org.projectlombok</groupId>
                <artifactId>lombok</artifactId>
                <version>1.18.38</version> <!-- 显式声明版本 -->
            </path>
        </annotationProcessorPaths>
    </configuration>
</plugin>
```
