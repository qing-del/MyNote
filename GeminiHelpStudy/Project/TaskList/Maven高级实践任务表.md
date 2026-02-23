### 第二部分：实战任务 —— 架构重构 (Refactoring)

你现在的 `task-system` 是一个单体项目。为了巩固 Maven 高级知识，你的任务是将它拆分为一个标准的**多模块项目**。

#### 🎯 任务目标
创建一个父工程 `jacolp-parent`，并拆分出以下子模块：

1.  **`task-common` (公共模块)**
    * **内容**: 存放 `Result` (InfoResponse), `JwtUtil`, `AliyunOSSOperator` (你可以把那个 starter 的代码整合进来或者依赖它), 全局异常类 `GlobalExceptionHandler`。
    * **依赖**: `lombok`, `jjwt`, `aliyun-oss`。
2.  **`task-pojo` (实体模块)**
    * **内容**: 存放 `User`, `Task`, `Item`, `PlayerStatus` 以及所有的 DTO (`UserRequest` 等)。
    * **依赖**: `lombok`。
3.  **`task-web` (业务模块 - 也就是原来的 task-system)**
    * **内容**: `Controller`, `Service`, `Mapper`, `Application启动类`。
    * **依赖**: 依赖 `task-common` 和 `task-pojo`，以及 `mysql`, `mybatis`, `spring-web` 等。

#### 🛠️ 操作步骤指南

**Step 1: 创建父工程 `jacolp-parent`**
* 新建一个 Maven 项目（不要选 Spring Boot Initializr，选 Maven）。
* 删除 `src` 目录（父工程不需要代码）。
* 修改 `pom.xml`:
    * `<packaging>pom</packaging>`
    * `<modules>` 暂时留空。
    * `<dependencyManagement>`: 把你原来 `task-system` 里所有的版本号提取到这里（Spring Boot, Mybatis, JWT 等）。
    * `<properties>`: 定义版本变量。

**Step 2: 创建 `task-pojo`**
* 在父工程下右键 -> New Module。
* 把 `entity` 包和 `dto` 包里的代码剪切过来。
* pom 中只需要 `lombok`。

**Step 3: 创建 `task-common`**
* 在父工程下右键 -> New Module。
* 把 `utils` 包, `exception` 包, `dto` 中的通用返回对象 (如 `PageResult`, `InfoResponse`) 剪切过来。
* pom 中需要引入通用依赖（如 fastjson, jwt 等）。

**Step 4: 改造 `task-web` (原 task-system)**
* 你可以保留原来的项目，将其变成子模块。
* **删除** 已经移走的 entity, dto, utils 代码。
* **引入依赖**:
    ```xml
    <dependency>
        <groupId>com.jacolp</groupId>
        <artifactId>task-pojo</artifactId>
        <version>1.0-SNAPSHOT</version>
    </dependency>
    <dependency>
        <groupId>com.jacolp</groupId>
        <artifactId>task-common</artifactId>
        <version>1.0-SNAPSHOT</version>
    </dependency>
    ```
* **去除版本号**: 所有在父工程锁定的依赖，这里都去掉 `<version>` 标签。

**Step 5: 测试**
* 在父工程目录下执行 `mvn clean install`。
* 如果提示 "BUILD SUCCESS"，说明聚合构建成功。
* 启动 `task-web`，验证游戏能否正常运行。


**🎓 导师寄语**：
这个任务完成后，你的项目结构将变得非常专业。以后如果还要开一个 `admin-system` (管理后台)，你只需要依赖 `task-pojo` 和 `task-common`，就能直接复用代码，这就是分模块开发的威力！

准备好了吗？开始动手拆分吧！遇到依赖报错（最常见的问题）随时把报错信息发给我。
