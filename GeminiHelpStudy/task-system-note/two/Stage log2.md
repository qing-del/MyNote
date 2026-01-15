# 第二阶段日志（写笔记的时候用一下）

### 问题集
- [x] 需要重新学习和了解 MySQL 中的 `添加列`与`外键约束`
- [x] 不了解什么是 Hash 加密
- [x] 以为 Hash 加密是改写 getter/setter
- [x] JwtUnit的Java类如果是看的话，能看懂，但是很难单独写出来（如果以后要自己写 不知道咋搞）
- [x] Jwt的Filter也是同上，但是相对来说，看懂程度低一点
	- 为何请求头不是 ”Bearer “开头就是非法的
	- 如何知道 Token 中请求头中substring从7开始就是jwt

- [x] 为什么 Service 写在 config 包中，而不是写在 service 包中
- [x] 登录成功之后，在前端使用带Token获取玩家状态的时候，会出现程序跑到JwtAuthenticationFilter.java中的final String authHeader = request.getHeader("Authorization");时，authHeader会得到null，我不知道怎么解决
- [ ] 

---
### 需求

1. 感觉需要 JwtUnit、JwtAuthenticationFilter、SecurityConfig、UserDetailsService 的模板（最好是带注释），然后一些常用的改写
2. 遇到报错null报错，最后在JwtUnit中修改了SECRET_KEY的赋值方式
3. 多用户数据隔离比较简单，一步过了，记一下大致流程即可，但是其中的SecurityUtil.java这个工具包可以给我个模板，或者大致给我讲解一下即可
4. 需要一份基础的后端工程师可以看懂的Vue使用指南，因为我基本上全部忘记了
5. 连 Vue 项目的起步都不会了，需要对Vue内容笔记做一份详细的讲解了
6. 感觉我不喜欢写HTML，好像大部分时候都是直接然AI生成，所以我学个调试和基本修改？这样的话我构建Vue项目之后，需要AI生成前端工程页面，需要给它提示什么的话做一份笔记
7. 

---

### 个人日志

1. 在初步写完Login.vue以及一些其他配置之后，出现了后端正常，首先是前端输入错误的账号密码点击登录会出现登录已过期，修好之后又出现了点击登录没有任何反应，检查后端控制台没有问题，打开前端控制发现爆红，再处理一次之后出现了`response.data is not a function`的错误提示，最后在request.js中发现了是因为❌`return response.data()`❌这个是错误的
2. 出现了登录后获取生成的任务会之后，点击`完成任务`，会弹出`登录已过期`＋`结算失败`的错误信息，然后询问”导师“之后，我先将'自毁'Token的前端代码注释掉了，然后前端的控制台中出现了`taskId=null`这个东西，然后我打开控制查看了历史请求内容，然后发现了后端传来的任务json格式中id为null，所以我去后端中进行了检查了Controller，发现了插入任务之后没有回填`newTask`的`id`值，所以我去`TaskMapper`中的`insertTask`接口上加入了`@Options(useGeneratedKeys = true, keyProperty = "id")`，然后再次尝试已成功
3. 发现了`Game.vue`中的前端界面中永远的都是固定展示`Player`字样，然后打开`Login.vue`中发现了其中处理登陆的函数在成功部分写了`localStorage.setItem('token', res.token)`，知道了将其存入到本地的”保险箱“内了，然后我去查资料学习如何解析后端传来的token内容，在`Game.vue`中加入了如下内容
	```javascript
	import jwtDecode from 'jwt-decode'
	//...(其他内容)
	const token = localStorage.getItem('token')
	const username = jwtDecode(token).sub
	//...
	const player = reactive({
	  username: username, // 从 Token 解析用户名
	  level: 1,
	  spirit: 0,
	  body: 0,
	  currentExp: 0,
	  totalExp: 10
	})
	```
4. 我在`Game.vue`中对经验部分的展示进行了修改，我给`player`玩家数据这个结构体变量加入了`totalExp`这个整数类型成员变量，并对`getNextLevelExp`这个函数修改，修改如下
	```javascript
	// 辅助计算函数
	const getNextLevelExp = () => player.totalExp
	const calculateExpPercent = () => {
	  const max = getNextLevelExp()
	  if (max === 0) return player.currentExp * 10
	  return max;
	}
	```
5. 做完3和4的修改之后，我发现不管怎么登录都会出现`登录已过期`的提示，然后我在后端打上断点，发现并没有跑到AuthController的函数里面就已经报错了，我通过查看后端报错信息加查资料，发现是之前注释掉了`localStorage.removeItem('token')`导致的，所以我对`request.js`做了修改，如下面
	```javascript
	import { useRouter } from 'vue-router'
	//...
	const router = useRouter()
	//...
	request.interceptors.response.use(
	    response => {
	        return response.data  // 只返回后端的部分数据，丢掉 axios 的壳
	    },
	    error => {
	        // 登录或则注册的报错给登录界面自己处理
	        if(error.config.url.includes('/login') ||
	         error.config.url.includes('/register')) {
	            return Promise.reject(error)
	        }
	        // 如果后端返回 403/401， 说明 Token 过期或者错误
	        if(error.response && (error.response.status === 403 ||
	         error.response.status === 401)) {
	            ElMessage.error('登录已过期，请重新登录！')
	            localStorage.removeItem('token')
	            // 这里可以加入跳转登录页的逻辑
	            router.push('/')
	        } else {
	            ElMessage.error(error.message || '网络异常')
	        }
	        return Promise.reject(error)
	    }
	)
	```
6. 做完上述修改就开始给我弹`登录失败！请检查账号和密码`，通过查询后端发现依旧是同样的报错，然后发现老是发送已过期的`token`，所以我在`utils`中加入了`auth.js`，然后再改进`Login.vue`
	```javascript
	// auth.js
	export const clearAuthToken = () => {
		localStorage.removeItem('token')
		console.log("Auth token cleared.")
	}
	```
	```javascript
	// Login.vue
	//...
	onMounted(() => {
		const token = localStorage.getItem('token')
		if(token) {
			// 如果有 token 检测一下有没有效(有效直接跳转至游戏界面)（后续加入）
			
			// 否则直接清除
			localStorage.removeItem('token')
		}
	})
	//...
	```
7. 修改完上述之后，点击登录会弹出`登录成功！正在进入游戏界面...`但是依旧是停留在`Login.vue`，并且在vue工程的终端中会有如下错误信息`[vite] Internal server error: Failed to resolve import "jwt-decode" from "src/views/Game.vue". Does the file exist?`，我通过查资料发现，似乎是因为没有安装`jwt-decode`这个依赖，所以我在控制台执行了一些指令来进行安装并在`node_modules`文件夹中发现`jwt-decode`之后重启vue工程开始测试，然后依旧是有些错误，但是通过查看前端控制台结合查资料发现了是语法错误，我使用了v2的旧语法`import jwtDecode from 'jwt-decode'`，应该是`import { jwtDecode } from 'jwt-decode'`，改进之后继续开始测试，发现页面的右上角从固定的`Player`变成了从后端传过来的用户名，以及经验也对应上了后端传过来的数值，但是`currentExp`依旧是显示为0，但是进度条却是显示`50%`
	```bash
	npm install jwt-decode
	```
8. 我又开始对`currentExp`问题的调整，我先打开前端控制台抓包查看，发现数据传输没有问题，后端控制台没有报错，所以我去查看原代码，改成了`percentage="calculateExpPercent(player.currentExp)"`即可，然后发现还是没变，结果发现是我个人日志4的位置改错了，所以又将其改了回去，就变成正常显示了
9. 根据下一步指导，在编写`GlobalExceptionHandler.java`时，遇到了不懂`@ExceptionHandler({AuthenticationException.class, AccessDeniedException.class})`要怎么用
10. 后续开始调整`TaskMapper`中的指令，开始思考需不需要学习XML配置文件来映射SQL查询语句，突然想到了通过`用户id`来进行查询任务列表的时候需要返回一个任务列表，所以我去回顾了XML配置怎么写，学习完毕之后自己改进了一点东西，新增分页查询
	```java
	//TaskMapper.java
	/**  
	 * 通过用户id查询任务（分页查询）  
	 * 如果传入的 limit 为 0 则返回所有任务  
	 * @param userId 用户id  
	 * @return 用户的任务列表  
	 */
	List<Task> selectByUserId(long userId);
	```
	```xml
	<mapper namespace="com.jacolp.task_system.mapper.TaskMapper">  
	    <insert id="insert" resultType="com.jacolp.task_system.entity.Task">  
	        insert into task (title, description, exp_reward, reward_type, reward, user_id, status, create_time)  
	        value (#{title}, #{description}, #{expReward}, #{rewardType}, #{reward}, #{userId}, #{status}, now())    </insert>  
  
	    <insert id="insertTask" resultType="com.jacolp.task_system.entity.Task">  
	        insert into task (title, description, exp_reward, reward_type, reward, user_id, status, create_time)  
            value (#{title}, #{description}, #{expReward}, #{rewardType}, #{reward}, #{userId}, #{status}, now())    </insert>  
  
	    <select id="selectByUserId" resultType="com.jacolp.task_system.entity.Task">  
	        select * from task where user_id = #{userId}  
	        order by create_time desc    </select>  
	</mapper>
	```
	```java
	// TaskServiceImpl.java
	/**  
	 * 获取玩家任务列表  
	 * @param playerId 玩家id  
	 * @param pageNum 页码  
	 * @param pageSize 每页数量  
	 * @return 玩家任务列表  
	 */  
	@Override  
	public PageResult<Task> getTaskList(Long playerId, int pageNum,
	 int pageSize) {  
	    // 1. 设置分页查询参数  
	    PageHelper.startPage(pageNum, pageSize);  
  
	    // 2. 调用 Mapper 接口方法  
	    List<Task> list = taskMapper.selectByUserId(playerId);  
  
	    // 3. 解析并封装结果  
	    Page<Task> p = (Page<Task>) list;  
	    return new PageResult<Task>(p.getTotal(), p.getResult());  
	}
	```
	```java
	// TaskController.java
	/**  
	 * 获取玩家任务列表  
	 * @param pageNum 页码  
	 * @param pageSize 每页数量  
	 * @return 玩家任务列表  
	 */  
	@GetMapping("/taskList")  
	public ResponseEntity<PageResult<Task>> getTaskList(@RequestParam int pageNum, @RequestParam int pageSize) {  
	    PlayerStatus status = getCurrentPlayerStatus();  
	    PageResult<Task> result = taskService.getTaskList(status.getId(), pageNum, pageSize);  
	    return ResponseEntity.ok(result);  
	}
	```

11. 改写了一下`TaskController`中的AI生成任务的方法，配合全局异常处理器去处理错误。我写了一个新的自定义异常类放在`exception`包下，然后用全局异常处理器去捕获这个`TaskException`
	```java
	// GlobalExceptionHandler.java
	/**  
	 * 捕获任务处理相关的异常  
	 * @param e 捕获到的异常  
	 * @return 错误信息  
	 */  
	@ExceptionHandler({TaskException.class})  
	public ResponseEntity<String> handleTaskException(AuthenticationException e) {  
	    return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
	    .body("服务器任务处理错误：" + e.getMessage());  
	}
	```
12. 将`Game.vue`改写了一下，将任务列表的`id`展示替换成了展示页面顺序中的id，而不是后端传过来的id，同时利用Ai辅助，添加了退出登录跳转到登录界面的过度动画
13. 我将`TaskController`中的`getCurrentPlayerStatus`移动到了`PlayerService`的实现类中了
14. 后续在`TaskMapper`中加入了`selectByTaskCondition`方法，使用XML文件映射
	```xml 
	<select id="selectByTaskCondition" resultType="com.jacolp.task_system.entity.Task">  
	    select * from task  
	    <where>  
	        <if test="userId != null">  
	            and user_id = #{userId}  
	        </if>  
	        <if test="id != null">  
	            and id = #{id}  
	        </if>  
	        <if test="title != null">  
	            and title like CONCAT('%', #{title}, '%')  
	        </if>  
	        <if test="description != null and description != ''">  
	            and description like CONCAT('%', #{description}, '%')  
	        </if>  
	        <if test="status != null">  
	            and status = #{status}  
	        </if>  
	        <if test="rewardType != null">  
	            and reward_type = #{rewardType}  
	        </if>  
	    </where>  
	    order by create_time desc  
	</select>
	```
15. 修改了`AiTaskService`，加入了读取历史五条已完成任务的内容
16. 对`Game.vue`进行修改，加入创建任务的按钮以及条件筛选查询人物列表的按钮，并完成逻辑实现，改完之后启动后端发现报错，查看报错发现是`PlayerServiceImpl`没有加上`@Service`的注解，后续再次尝试启动，然后启动成功，进入前后端联动测试
17. 这里发现对后端的改动之后，在登录页面输入正确的账号密码后点击登录，一直是出现`请检查账户和密码！`的错误，我去后端控制台发现又没什么问题，也没有日志显示，我直接去了前端控制台，查看是收到了`403`的错误码，但是我在想是不是预检测的问题，结果发现好像不是，因为预检测是过的，但是登录是不过的，后续我在后端中打断点，终于发现了`JwtFilter`中会拦截，然后在登录的请求中，会出现请求头为空，但是后端中我改了点东西忘记同步，所以过滤的请求原地址为`/api/auth`，但是我改成了`/auth`作为请求地址，又忘记去修改`SecurityConfig`中的过滤地址导致的，后续进行修改后进行前后端联动测试，这一次找问题找了挺久的，所以我回顾我很久以前的笔记，重新学习了一下日志技术
18. 改进了一下前端页面的`Game.vue`中的代码，在创建任务的面板中加入了`奖励类型`和`属性奖励`，第一个用于任务奖励什么类型（比如是精神属性还是肉体属性），第二个决定奖励的数值，并且加入了条件分页查询
19. 在少部分地方加入了日志打印，并且引入了`logback.xml`文件，用于后续查找bug时路过的点作为一个展示，方便后续调整代码
20. 改进`TaskController.java`，完成任务1：`Controller 瘦身计划`
21. 创建table表`shop_items`；完成`Item`实体类的创建；完成`ItemMapper`的创建；完成了`ShopService`和基本的`ShopController`；然后我的猫跑到电脑的键盘上睡觉了，不知道出了点啥问题，将IDEA报错的内容改掉了，然后出现无法启动的问题（好像是死循环，并不会终止进程，而是卡在某个地方不动了），通过控制台的日志去查资料，先检查了一下Mapper的XML配置文件格式，发现没有问题，然后去检查了一下数据库的连接，没有问题，最后是`logback.xml`中格式错误导致的问题
22. 启动成功之后又出现了无法登录的问题，输入正确的账号密码就会显示`500`服务器异常，这个时候再点击注册就会有`403`状态码，根据查资料结合AI辅助，发现是缺失了`io.jsonwebtoken`这个依赖，添加进去之后，成功登录，进行跳转商店，创建任务，完成任务等测试均无问题
23. 由于`Task`实体中没有`coinReward`属性从而无法获取硬币，先去数据库修改Task表，加入coinReward这一列，然后修改实体类，再去修改Mapper接口，后续调整前端中显示的内容（游戏界面中的创建任务和AI创建任务以及展示部分，同时在玩家属性展示面板中加入硬币），前后端测试没有问题。在后端中加入一个测试商品进行测试，在点击购买按钮之后，会给我弹出登录已过期的错误信息，发现是后端处理购买失败的时候直接使用抛出异常的方法来返回失败信息导致的，后续加入了`ShopResponse`实体类来返回购买信息之后，又出现后端处理是购买成功，但是前端却出现了捕获异常处理，后续发现是由于`request.js`中的逻辑会脱壳，所以要将`res.data.result`改为`res.result`之后，再次测试可以正常购买了
24. 现在打算加入任务编辑功能，以及后续的管理端。修改的时候发现`task`数据库表中的`coinReward`是驼峰命名法，但是sql是大小写不敏感的，故改为`coin_reward`；

---``

### 个人发现问题

- PS：如果描述的问题被修复了，会打勾

1. [x] 首先就是对于‘自毁’Token这个操作复原的前提是，后端里面处理好特定信息的返回，现在是有一些错误即使不是Token的问题，但是依旧会返回`403`这个错误码，需要去调整一下后端的代码
2. [x] 目前MySQL中任务表中的数据是没有做用户隔离的，就是后续需要将任务做一个隔离，并且可以通过用户的id来查询用户拥有的任务列表有哪几个
3. [ ] 


---

### 处理规划
> PS：从`个人日志`24点开始记录

24. 增加编辑功能 + 完善管理者接口和逻辑
	1. [x] 先在后端完善处理编辑任务的功能
		1. [x] 确保`TaskController`有编辑任务的接口
		2. [x] 确保`TaskServiceImpl`和`TaskMapper`中有逻辑和数据处理
	2. [x] 在后端中完成删除任务操作的逻辑
		1. [x] 确保`TaskController`中有删除任务的接口
		2. [x] 确保在`TaskService`和`TaskMapper`中有删除任务的逻辑和数据处理
	3. [x] 完成前端游戏界面中加入`修改`和`删除`按钮的展示和逻辑实现
	4. [ ] 由于后端中安全系统是获取用户名之后再通过用户名去获取玩家信息的
		1. [ ] 后端中在`User`表中加入`authority`作为权限级别的代表
		2. [ ] 给实体类`User`加入`authority`这一个成员变量
	5. [ ] 数据处理
		1. [ ] 给Mapper层修改一些语句
	6. [ ] 逻辑处理
		1. [ ] 管理玩家属性
		2. [ ] 管理任务
		3. [ ] 管理用户
		4. [ ] 管理商店
	7. [ ] 接口开发
		1. [ ] 登录检测
		2. [ ] 管理玩家属性
		3. [ ] 管理任务
		4. [ ] 管理用户
		5. [ ] 管理商店
	8. [ ] 前端页面开发
		1. [ ] 管理员登录面板
		2. [ ] 
	9. [ ] 完成前后端联桥测试
25. [ ] 