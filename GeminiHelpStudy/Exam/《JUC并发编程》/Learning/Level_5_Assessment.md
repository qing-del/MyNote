# Level 5：线程池工程实战 —— 引导式学习问题册

> 前四关你把 AQS、MESI、volatile 都啃下来了。但有一件事你做了四次都没做对：在令牌桶里反复用 `threadPool.submit() + result.get()` 这个"假异步"反模式，导致压测在 4000 并发下 70%+ 的崩溃率。
>
> 这一关正面拆解线程池——不是让你背参数，而是要你理解参数联动，能在出了问题的时候看懂 jstack、看懂日志、知道该调哪个旋钮。你在前几关写的线程池代码会被当作案例直接出现在题目里。

---

## 📋 本 Level 考核范围

| 知识板块 | 核心考点 | 对应黑马章节 |
|---|---|---|
| `ThreadPoolExecutor` 七大参数 | 参数联动推演、任务提交流程 | 第八章 08.011 ~ 08.013 |
| 阻塞队列与 OOM | 无界队列陷阱、`LinkedBlockingQueue` vs `ArrayBlockingQueue` | 第八章 08.014 ~ 08.016 |
| 拒绝策略 | 四种内置策略适用场景、自定义策略 | 第八章 08.009 ~ 08.010 |
| 线程池停止 | `shutdown` vs `shutdownNow`，优雅停机 | 第八章 08.020 ~ 08.021 |
| Tomcat 线程池 | 与 JDK 线程池的核心差异 | 第八章 08.032 ~ 08.033 |
| 多任务协调 | `CountDownLatch`、`CompletableFuture` 实战 | 第八章 08.068 ~ 08.072 |

---

## 🔥 大题一：ThreadPoolExecutor 参数联动 —— 你的令牌桶为什么崩了

### Part A · 尸检报告

你在 Level 2 ~ Level 4 的令牌桶里，始终用着这个线程池配置：

```java
static final ThreadPoolExecutor threadPool = new ThreadPoolExecutor(
    25,                              // corePoolSize
    40,                              // maximumPoolSize
    6000,                            // keepAliveTime（ms）
    TimeUnit.MILLISECONDS,
    new LinkedBlockingQueue<>(3000), // workQueue
    (r, executor) -> {               // 自定义拒绝策略
        boolean success = executor.getQueue().offer(r, 500, TimeUnit.MILLISECONDS);
        if (!success) throw new RejectedExecutionException(...);
    }
);
```

4000 并发线程下压测结果：
- 总请求 47097，成功 13040，**异常崩溃 34057（72.31%）**

**请回答以下问题：**

1. `ThreadPoolExecutor` 接收一个新任务时，内部的**完整路由逻辑**是什么？按以下四个判断节点逐一说明（不能只说结论，要说触发条件）：
   - 节点一：核心线程数判断
   - 节点二：队列入队判断
   - 节点三：最大线程数判断
   - 节点四：拒绝策略触发

2. 针对上面的线程池配置，在 4000 并发下，**定量**推演任务被拒绝的全过程：
   - 最多有多少个任务被线程处理？（核心线程 + 非核心线程）
   - 队列最多缓冲多少个任务？
   - 超过容量后，自定义拒绝策略等待 500ms 还是满，那这 500ms 里线程池在做什么？
   - 为什么还是会抛 `RejectedExecutionException`，最终导致 34057 个异常？

3. 这个线程池的 `keepAliveTime = 6000ms = 6秒` 对 `corePoolSize` 线程有效吗？如果要让核心线程也能超时被回收，需要做什么？

4. 如果用 **`Executors.newFixedThreadPool(40)`** 替代上面的配置，在相同 4000 并发下，会不会也出现大量异常？为什么？这个替代方案有什么隐患？

### Part B · 场景实战 —— 电商大促的聚合接口

**业务背景**：电商大促首页需要聚合展示以下 5 个服务的数据：
- 商品基础信息（`ProductService`，耗时约 50ms）
- 实时库存（`InventoryService`，耗时约 80ms）
- 用户优惠券（`CouponService`，耗时约 120ms）
- 推荐商品列表（`RecommendService`，耗时约 200ms）
- 直播状态（`LiveService`，耗时约 30ms）

**性能要求**：整体接口响应时间 ≤ 300ms（若串行调用总计 480ms，必须并行）。

**请完成以下任务：**

1. 用 `CompletableFuture` 实现上述 5 个服务的**并行调用与聚合**，给出完整代码（包括异常处理、超时控制），不需要服务的具体实现，用 `Thread.sleep(xms)` 模拟耗时即可。

2. 在你的实现中，`CompletableFuture` 默认使用什么线程池？在生产环境中为什么不应该用默认线程池，应该怎么做？

3. 如果 `LiveService` 调用超时（超过 300ms 才返回），你的代码应该如何处理？是等待它、还是用降级值（`"直播已结束"`）继续返回？给出实现。

---

> ⏳ **请回答【大题一】全部内容后，我再出大题二（Tomcat 线程池 vs JDK 线程池 + 线程池优雅停机）。**
>
> 如果某个子问题不确定，直接说"这部分不太会，给个方向"，我会告诉你去看黑马哪一节。

---

### 【极客挑战】（选做，可留给未来）

**极客挑战 1 · 线程池的 `ctl` 字段**

> `ThreadPoolExecutor` 内部用一个 `AtomicInteger ctl` 同时存储**线程池状态**和**工作线程数量**（高 3 位存状态，低 29 位存数量），这和 `ReentrantReadWriteLock` 用一个 `int state` 同时存读锁和写锁计数的思想如出一辙。
> - `RUNNING`、`SHUTDOWN`、`STOP`、`TIDYING`、`TERMINATED` 五种状态的转换路径是什么？`shutdown()` 和 `shutdownNow()` 分别触发哪个转换？
> - 为什么要将状态和线程数合并在一个原子变量里，而不是两个独立字段？

**极客挑战 2 · `execute()` 的二次检查**

> 在 `execute(Runnable command)` 中，有一段看起来多余的代码：
> ```java
> int c = ctl.get();
> if (workerCountOf(c) < corePoolSize) {
>     if (addWorker(command, true)) return;
>     c = ctl.get();  // ← 重新读取 ctl
> }
> if (isRunning(c) && workQueue.offer(command)) {
>     int recheck = ctl.get();  // ← 为什么要 recheck？
>     if (!isRunning(recheck) && remove(command)) reject(command);
>     else if (workerCountOf(recheck) == 0) addWorker(null, false);
> }
> ```
> - `recheck` 这个二次检查是为了防止什么竞态条件？如果没有它，在什么时序下会出现"任务进了队列但永远不会被执行"的 Bug？

**极客挑战 3 · 虚拟线程与线程池的未来**

> JDK 21 引入了虚拟线程（你在压测代码里用的 `Executors.newVirtualThreadPerTaskExecutor()`）。
> - 虚拟线程的 `park()` 和平台线程的 `park()` 有什么本质区别？虚拟线程 `park` 时，它的载体线程（carrier thread）会发生什么？
> - 在什么场景下，把传统平台线程池换成虚拟线程会有明显收益？什么场景下换了反而更差？（提示：CPU 密集型 vs IO 密集型）
> - 你在令牌桶里混用了虚拟线程（测试侧）+ 平台线程池（服务侧），这个组合的性能瓶颈在哪？

---

> 📌 **精准学习路径（针对你的历史弱点）**：
>
> | 薄弱点 | 推荐视频 | 推荐书籍 |
> |---|---|---|
> | 线程池 7 参数 + 任务路由 | 08.011 ~ 08.013 | — |
> | 拒绝策略实战 | 08.009 ~ 08.010 | — |
> | `newFixedThreadPool` 的 OOM 陷阱 | 08.014 ~ 08.016 | — |
> | 线程池优雅停机 | 08.020 ~ 08.021 | — |
> | Tomcat 线程池 vs JDK 线程池 | 08.032 ~ 08.033 | — |
> | `CountDownLatch` 实战 | 08.068 ~ 08.070 | — |
> | `CompletableFuture` 多任务聚合 | 08.072 | — |
