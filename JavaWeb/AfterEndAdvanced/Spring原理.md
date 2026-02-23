# Spring 原理篇
## 目录
- [配置优先级](#配置优先级)
    - [配置文件](#配置文件)
    - [[#Java系统属性 ＆ 命令行参数]]
- 
- SpringBoot 原理

---

## 配置优先级
### 配置文件
- 一般分为三种配置文件的类型：
    - application.properties
    - application.yml
    - application.yaml

```properties
server.port=8081
```

```yml
server:
  port: 8082
```

```yaml
server:
  port: 8083
```

- 配置文件优先级：properties > yml > yaml
- 在实际开发中，一般使用 yml 文件，因为 yml 文件更易读。

### Java系统属性 ＆ 命令行参数
- Java系统属性：通过 -D 参数设置 `-Dserver.port=9000`
- 命令行参数：通过 -- 参数设置 `--server.port=10010`
- 可以通过 IDEA 的设置进行配置：
    - Java系统属性：
        - Run/Debug Configurations -> VM Options -> -Dserver.port=9000
    - 命令行参数：
        - Run/Debug Configurations -> Program arguments -> --server.port=10010
    - 最新版的 IDEA 配置方式：
        - 可以通过当前的启动项进行配置，先点击`Edit Configurations`
        - 然后确认选择了是当前项目需要进行修改的启动项，然后点击`Modify options`
        - 然后勾选`Add VM options`和`Program arguments`即可

- 优先级：命令行参数 > Java系统属性 > 配置文件

---
## Bean管理
### Bean作用域
- Spring支持五种作用域，后三种在web环境生效：
	- singleton（默认）：创建单个实例
	- prototype：每次使用都会创建一个新的示例
	- request：
	- session：
	- application：