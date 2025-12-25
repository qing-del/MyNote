# 项目上下文存档：LifeGame (Java Spring Boot)

## 1. 用户身份
我是 Java 基础复健的大二学生，正在开发一个“个人属性升级系统 (Life Game)”。

## 2. 当前进度 (MVP 已完成)
- **技术栈**：JDK 17, Spring Boot 3.3.4, MyBatis 3.0.3, MySQL 8.0, DeepSeek-V3 API。
- **已实现功能**：
  - 玩家状态 (PlayerStatus) 的查询与升级逻辑。
  - 任务 (Task) 的 CRUD 与结算（事务管理）。
  - AI 自动生成任务：调用 DeepSeek API，根据玩家当前的 Spirit/Body 属性生成 JSON 格式任务。
  - 前端：一个单页 index.html，原生 JS + Fetch 实现交互。

## 3. 关键代码结构
- **Entity**: `PlayerStatus` (id, spirit, body, level, exp...), `Task` (id, title, reward...)
- **Mapper**: 使用 Mybatis 注解模式 (`@Select`, `@Insert`)，已开启驼峰映射。
- **Service**: 
  - `TaskServiceImpl`: 包含 `completeTask` 逻辑，处理经验结算和升级 (`@Transactional`)。
  - `AiTaskService`: 使用 Java 11 `HttpClient` 调用 AI 接口。
- **Config**: `application.yml` 中配置了 `ai.deepseek.api-key` 和 `base-url`。

## 4. 当前环境状态
- **数据库**：表 `player_stats` 和 `task` 已建好并有测试数据。
- **Bug修复记录**：已解决 Spring Boot 版本冲突、JUnit 依赖缺失、MyBatis 驼峰映射失效、YAML 缩进错误。

## 5. 下一步目标
我们要开始 **第二阶段开发**。
请作为我的 Senior Java 导师，指导我完成以下任务（请询问我想先做哪个）：
1. **用户认证 (Auth)**：实现简单的注册/登录，区分多用户。
    - **目标**：引入 Spring Security + JWT，实现用户登录注册。
    - **重点**：将硬编码的 PLAYER_ID = 1 改为动态从 Token 获取，实现真正的多用户隔离。
2. **UI 重构**：优化前端界面。
    - **目标**：抛弃原生单页 HTML，引入 Vue 3 (Vite) + Element Plus。
    - **重点**：构建可交互的“属性面板”和“任务卡片列表”，增加经验条增长动画。
3. **任务逻辑进化**：
    - **目标**：在调用 AI 时，不仅发送当前属性，还发送“历史任务日志”。
    - **重点**：防止 AI 重复生成任务，并根据目标，让 AI 自动生成任务内容。

请基于以上上下文，直接进入指导模式。