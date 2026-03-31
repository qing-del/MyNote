# Level 5 · 评分报告

> **考核模块**：线程池工程实战 —— ThreadPoolExecutor 参数联动 · CompletableFuture 聚合
> **考核日期**：2026-03-31
> **评分规则**：百分制，Part A（50分）+ Part B（50分）；极客挑战额外计算

---

## 🏆 总分

| 模块 | 满分 | 实得 |
|---|---|---|
| Part A · Q1 任务路由四步逻辑 | 10 | 9 |
| Part A · Q2 定量推演拒绝过程 | 15 | 8 |
| Part A · Q3 keepAliveTime 核心线程 | 5 | 5 |
| Part A · Q4 newFixedThreadPool OOM | 10 | 9 |
| Part A · 改进版令牌桶（额外加分） | 10 | 6 |
| Part B · Q1 CompletableFuture 并行聚合 | 25 | 16 |
| Part B · Q2 默认线程池风险 | 10 | 6 |
| Part B · Q3 超时降级 | 15 | 10 |
| **合计** | **100** | **69** |

> ⚠️ **综合判定**：69 分。线程池参数联动的宏观理解到位，OOM 陷阱和 keepAliveTime 边界都答得干净利落。但两个核心扣分点非常明显：① `CompletableFuture` 被当成 `static final` 变量只执行了一次而非每次请求并行；② 定量推演停留在"概述"层面，没有按题目要求精确算出每一个数字。改进版令牌桶做了调参但「假异步」反模式仍在，第五次出现了。**勉强及格**，允许进入 Level 6 但附加条件。

---

## 📋 逐题点评

---

### Part A · Q1：任务路由四步逻辑（9 / 10）

✅ **流程图完全正确**：

```
execute() → workerCount < corePoolSize → 创建核心线程
         → queue.offer() 成功 → 入队
         → workerCount < maximumPoolSize → 创建急救线程
         → 拒绝策略
```

图示清晰，四步判断节点齐全，箭头条件标注正确。

❌ **扣分点（-1）**：

流程图只画了路由判断，但题目要求"不能只说结论，要说**触发条件**"。你没有补充文字说明每个节点的触发条件细节，比如：
- 节点一中 `addWorker(command, true)` 的 `true` 表示以 `corePoolSize` 为上限
- 节点二中是 `workQueue.offer(command)` 非阻塞尝试，满了直接返回 false
- 节点三中 `addWorker(command, false)` 的 `false` 表示以 `maximumPoolSize` 为上限

这些细节面试官会追问。

---

### Part A · Q2：定量推演（8 / 15）

✅ **部分正确**：
- 队列缓冲 3000 个任务——正确。
- "500ms 内线程池里执行中的任务还在阻塞等结果"——方向正确，触及了假异步问题的核心。
- 后续分析"每秒完成 500 个任务"的吞吐量推算有一定道理。

❌ **扣分点（-7）**：

**题目要求"定量推演"，你的回答是定性概述，缺少关键数字（-5）**

标准定量推演：

```
时间 t=0, 4000 个虚拟线程同时调用 tryGetToken():
  ┌─ 前 25 个任务 → 创建 25 个核心线程执行
  ├─ 接下来 3000 个任务 → 进入 LinkedBlockingQueue
  ├─ 接下来 15 个任务 → 创建 15 个非核心线程（25+15=40=maxPoolSize）
  └─ 剩余 4000-25-3000-15 = 960 个任务 → 触发拒绝策略

拒绝策略: offer(r, 500ms)
  - 这 960 个任务各自等待 500ms，期间线程池在干什么？
  - 40 个线程全部在执行 while(!tokenPool.tryAcquire(1))
    忙等（Level 4 已指出），消耗 CPU 但基本不释放队列位置
  - 每秒只能完成 ~500 个任务（受令牌桶限速）
  - 500ms 后 → 约 250 个任务完成 → 但 960 个都在等 → 大量仍 offer 失败
  - → 抛 RejectedExecutionException

这个过程循环发生（20秒内持续涌入新请求），所以最终累计 34057 个异常。
```

你的回答没有算出 `4000 - 25 - 3000 - 15 = 960` 这个关键数字，没有解释 500ms 内线程池处理速度和等待任务数的对比关系。

**"令牌初始计算"偏题了（-2）**

> "初始 1000 个令牌，后续每秒补充 500 个，总共会有 11000 个"

题目问的是**任务为什么被拒绝**（线程池层面），不是**令牌有多少**。令牌桶的容量是限流层面的问题，而 `RejectedExecutionException` 是线程池容量饱和的问题——两者是不同层级的瓶颈。你把两个问题混在一起了。

---

### Part A · Q3：keepAliveTime 核心线程（5 / 5）

✅ **满分**。

- "对核心线程无效，只对非核心（急救）线程有效"——正确。
- `allowCoreThreadTimeOut(true)` 让核心线程也受超时控制——正确。

---

### Part A · Q4：newFixedThreadPool OOM（9 / 10）

✅ **核心正确**：
- 不会抛 `RejectedExecutionException`，因为内部使用**无界** `LinkedBlockingQueue`。
- 任务会被无限排队，不会报错但可能 OOM——完全正确。
- "OOM 反而还不如抛出大量异常"——这是一个很成熟的工程判断，加分。

❌ **扣分点（-1）**：

轻微不精确：`newFixedThreadPool(40)` 创建的队列是 `new LinkedBlockingQueue<Runnable>()`，容量为 `Integer.MAX_VALUE`（约 21 亿），严格来说是"有界但极端大"。在实际场景中等同于无界，但面试时可以更精确。

---

### Part A · 改进版令牌桶（6 / 10）

✅ **参数调优有效果**：
- 缩小队列（3000 → 500），减少无效排队。
- 缩短等待时间（2000ms → 400ms），更快超时。
- 令牌守恒性验证通过：10000 并发下 `成功 5500 = 恢复 5500`，完美守恒。

❌ **核心反模式第五次出现（-4）**：

```java
Future<String> result = threadPool.submit(() -> { ... });
return result.get();  // ← 第五次
```

Level 2 首次指出，Level 4 评语明确警告"Level 5 再出现会加倍扣分"。你在本关的改进集中在参数调优，但根本的架构问题始终没动。

正确做法（重复第四次了）：`Semaphore.tryAcquire(1, timeout, unit)` 是线程安全的，在调用方线程直接操作即可，**不需要线程池包装**。

```java
public static String tryGetToken() {
    try {
        if (tokenPool.tryAcquire(1, MAX_WAIT_TIME, TimeUnit.MILLISECONDS)) {
            return JwtTokenGenerator.generateToken();
        }
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
    }
    return null;
}
```

这三行代码替代你 20 行的 `threadPool + submit + while + get` 结构，而且：
- 没有线程池瓶颈
- 没有 `RejectedExecutionException`
- 没有忙等自旋浪费 CPU
- AQS 内部帮你 park/unpark，超时自动返回 false

---

### Part B · Q1：CompletableFuture 并行聚合（16 / 25）

✅ **答对的部分**：
- 5 个服务的 `CompletableFuture.supplyAsync()` 并行启动——方向正确。
- 每个 `get()` 都设置了 `MAX_WAIT_TIME = 300ms` 超时——有超时控制意识。
- 代码能跑，输出结果 212ms ≤ 300ms，满足性能要求。
- 服务模拟代码完整。

❌ **严重设计错误（-9）**：

**① `CompletableFuture` 被声明为 `static final`（-6）**

```java
private static final CompletableFuture<String> productBaseInfoFuture =
        CompletableFuture.supplyAsync(productService::getProductBaseInfo);
```

这意味着这 5 个 `CompletableFuture` 在**类加载时就执行了，且只执行一次**。后续每次调用 `getWebInfo()` 时，拿到的都是同一个已经完成的 `Future`，不会再次发起调用。

在真实业务中，每次用户请求都应该创建新的 `CompletableFuture`：

```java
public static WebInfo getWebInfo() {
    CompletableFuture<String> productFuture = 
        CompletableFuture.supplyAsync(productService::getProductBaseInfo, executor);
    CompletableFuture<String> inventoryFuture = 
        CompletableFuture.supplyAsync(inventoryService::getInventoryNumber, executor);
    // ... 其他服务
    
    CompletableFuture.allOf(productFuture, inventoryFuture, ...).join();
    // 聚合结果
}
```

你的代码之所以输出 212ms 是对的，是因为第一次调用时确实是并行的。但如果第二次调用 `getWebInfo()`，它会瞬间返回（因为 Future 已经完成），这在业务上是错误的——返回的是第一次的缓存数据。

**② 没有使用 `allOf()` 或 `thenCombine()` 聚合（-2）**

你的实现是对 5 个 Future 逐个 `get()` 串行收集结果。虽然 Future 本身是并行执行的，但如果第一个 `get()` 阻塞了 300ms，后面的 `get()` 都会额外等。

`CompletableFuture` 的正确聚合姿势是 `allOf()` 配合 `orTimeout()`：

```java
CompletableFuture<Void> allDone = CompletableFuture.allOf(
    productFuture, inventoryFuture, couponFuture, recommendFuture, liveFuture
).orTimeout(300, TimeUnit.MILLISECONDS);

try {
    allDone.join();
} catch (CompletionException e) {
    // 超时或异常的降级处理
}

// 此时所有已完成的 Future 都可以 getNow(降级值) 获取结果
webInfo.productBaseInfo = productFuture.getNow("默认商品信息");
```

**③ 超时不是单独控制每个 Future，而是控制整体（-1）**

你给每个 Future 单独设了 300ms 超时，但如果 `productFuture.get(300ms)` 等了 299ms 才返回，后面的 `inventoryFuture.get(300ms)` 还能再等 300ms——整体可能超过 300ms。应该用一个全局 deadline 或 `allOf().orTimeout()`。

---

### Part B · Q2：默认线程池（6 / 10）

❌ **线程池类型答错了（-4）**

> "默认实现线程池为 `newCachedThreadPool`"

`CompletableFuture.supplyAsync()` 不指定线程池时，默认使用的是 **`ForkJoinPool.commonPool()`**，不是 `newCachedThreadPool`。

| | `ForkJoinPool.commonPool()` | `Executors.newCachedThreadPool()` |
|---|---|---|
| 线程数量 | 默认 = `Runtime.availableProcessors() - 1` | 无上限（0 ~ Integer.MAX_VALUE） |
| 队列类型 | 工作窃取队列（双端 Deque） | `SynchronousQueue`（无缓冲） |
| 风险 | 线程数太少，阻塞任务多时线程耗尽 | 线程数太多，OOM |

✅ **OOM 风险判断正确**：不应该用默认线程池，应该自定义 `ThreadPoolExecutor`——这个结论对。

---

### Part B · Q3：超时降级（10 / 15）

✅ **答对的部分**：
- 每个 Future 的 `get()` 都包了 `TimeoutException` catch——有超时意识。
- 提到了前端 Vue 也有超时控制——这是全栈视角，说明你有工程经验。

❌ **扣分点（-5）**：

**① 没有给出降级值实现（-3）**

题目明确要求 `LiveService` 超时时返回降级值 `"直播已结束"`，你的代码在 catch 里只打了 `System.err.println("获取推荐列表超时")`（而且这条日志文案还写错了——LiveService 超时打的是"获取推荐列表超时"），没有赋降级值。

正确实现：
```java
try {
    webInfo.liveStatus = liveStatusFuture.get(remainingMs, TimeUnit.MILLISECONDS);
} catch (TimeoutException e) {
    webInfo.liveStatus = "直播已结束";  // ← 降级值
}
```

或者更优雅的 `CompletableFuture` 方式：
```java
CompletableFuture<String> liveStatusFuture = CompletableFuture
    .supplyAsync(liveService::getLiveStatus, executor)
    .orTimeout(300, TimeUnit.MILLISECONDS)
    .exceptionally(ex -> "直播已结束");  // ← 超时自动降级
```

**② 架构图偏题了（-2）**

你画了一个 Vue + Tomcat 的完整 MVC 架构图，虽然思路有意义（前端也有超时），但题目问的是**后端 CompletableFuture 层面的降级策略**，不是讨论前后端分工。这部分时间花在写降级代码上会更有价值。

---

## 🎮 极客挑战 · 补充评分

### 极客挑战 1 · 线程池 `ctl` 字段（13 / 15）

✅ **答对的核心**：
- 五种状态转换路径流程图正确：`RUNNING → SHUTDOWN → TIDYING → TERMINATED` / `RUNNING → STOP → TIDYING → TERMINATED`。
- `shutdown()` 触发 `RUNNING → SHUTDOWN`，`shutdownNow()` 触发 `RUNNING → STOP`——正确。
- 合并的原因是减少一次 CAS 操作——核心理由正确。

❌ **扣分点（-2）**：

**① 漏掉了 `SHUTDOWN → STOP` 的转换（-1）**

`shutdown()` 执行后如果再调用 `shutdownNow()`，状态会从 `SHUTDOWN → STOP`。你的流程图里漏了这条边。

**② 对"减少总线风暴"的描述稍过了（-1）**

> "大大减少了'总线风暴'的可能性"

主要原因不是"减少总线风暴"（`ctl` 的操作频率没有高到触发总线风暴），而是：**将状态和线程数捆绑在一个原子变量里，保证了"读状态 + 改线程数"或"读线程数 + 改状态"是一个原子快照**。如果分两个字段，判断"当前是 RUNNING 状态"和"线程数 +1"之间可能有其他线程插入 `shutdown()`，导致在 SHUTDOWN 后还新增了工作线程。

---

### 极客挑战 2 · `execute()` 的二次检查（13 / 15）

✅ **核心逻辑答对了**：
- 防止入队后线程池被 shutdown，导致任务永远不被执行——一针见血。
- 时序图中准确展示了：任务入队 → 另一个线程调 shutdown → 状态变化 → 通过 recheck 发现——正确。
- 提到了 `workerCountOf(recheck) == 0` 时会 `addWorker(null, false)` 作为兜底——这是很容易漏掉的细节。

❌ **扣分点（-2）**：

**① "尝试拒绝；无法拒绝将开急救线程急救" 描述不够精确（-2）**

实际逻辑是两个分支，不是"尝试拒绝后开急救"：

```java
if (!isRunning(recheck) && remove(command))
    reject(command);                      // 分支A：不在RUNNING且移除成功→拒绝
else if (workerCountOf(recheck) == 0)
    addWorker(null, false);               // 分支B：仍在RUNNING，但0工作线程→兜底
```

- 分支 A：线程池不再 RUNNING 且能从队列移除任务 → 直接拒绝
- 分支 B：线程池仍在 RUNNING 但工作线程数为 0（所有核心线程在 recheck 前刚好全部超时死亡）→ 加一个空任务的 worker 来消费队列

这两个分支是 `if-else` 关系，不是"先尝试再兜底"的顺序关系。

---

## 📊 能力雷达图（文字版）

| 能力维度 | 评价 |
|---|---|
| 线程池参数联动理解 | ⭐⭐⭐⭐ 良好，路由逻辑清晰，OOM 陷阱一眼看穿 |
| 定量分析能力 | ⭐⭐ 较弱，宏观方向对但精确数字算不出来 |
| CompletableFuture 使用 | ⭐⭐ 较弱，核心 API 用错（static final / 无 allOf / 无降级值） |
| 线程池源码理解 | ⭐⭐⭐⭐ 良好，ctl 合并字段、execute 二次检查都能说清楚 |
| 工程架构意识 | ⭐⭐⭐ 中等，有全栈视角但"假异步"反模式始终未修复 |
| 压测迭代能力 | ⭐⭐⭐⭐ 良好，持续调参 + 数据验证 + 万级并发测试 |

---

## 🎯 进入 Level 6 前的附加条件

Level 6 是最后一关（ConcurrentHashMap），但在进入之前，你需要**认知修正**以下两点：

### 1. `threadPool.submit() + result.get()` 反模式（第五次出现）

这个写法从 Level 2 到 Level 5 共出现了 5 次，每次评分都指出了。根源在于你把"限流器"和"线程池"的职责混在一起了：

```
正确分层:
┌─────────────────────────────┐
│  调用方线程（虚拟线程 / Tomcat 线程）  │
│  ↓ 直接调用                     │
│  Semaphore.tryAcquire(timeout) │ ← 限流层（AQS 内部 park/unpark）
│  ↓ 获取到令牌                   │
│  业务逻辑（生成 JWT 等）          │ ← 业务层
└─────────────────────────────┘

你的分层:
┌─────┐     ┌──────────┐     ┌──────────┐
│调用方│ →→→ │线程池排队  │ →→→ │Semaphore│ 
│get()│ ←← │execute   │ ←← │tryAcquire│
└─────┘     └──────────┘     └──────────┘
  ↑ 阻塞         ↑ 瓶颈           ↑ 限流
```

中间那个"线程池"是多余的，它只带来了瓶颈和异常。

### 2. `CompletableFuture` 核心 API

- `allOf()` / `anyOf()` ——并行等待所有/任一完成
- `orTimeout(duration, unit)` ——超时自动 CompletionException
- `exceptionally(ex -> 降级值)` ——异常降级
- `getNow(默认值)` ——非阻塞获取当前值
- **不要**把 `CompletableFuture` 声明为 `static final`

---

## ✅ 导师判定

> **69 分，勉强及格。允许进入 Level 6，但附加条件：Level 6 的代码中不允许再出现 `threadPool.submit() + result.get()` 模式，否则直接额外扣 10 分。**
>
> 这一关你的亮点在参数联动理解和极客挑战的源码分析，但 CompletableFuture 的 static final 错误暴露了你对这个 API 的使用经验不足。建议在做 Level 6 之前，花 30 分钟动手写几个 CompletableFuture 的小 Demo（`allOf` + `orTimeout` + `exceptionally`），这个工具在面试中出现频率极高。
