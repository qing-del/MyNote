# Level 2 · 评分报告

> **考核模块**：底层框架与显式锁 (AQS & ReentrantLock) —— CAS 与 ABA · 令牌桶限流器
> **考核日期**：2026-03-27
> **评分规则**：百分制，Part A（50分）+ Part B（50分）

---

## 🏆 总分

| 模块 | 满分 | 实得 |
|---|---|---|
| Part A · Q1 ABA 问题定义与分析 | 20 | 16 |
| Part A · Q2 CAS CPU 指令原子性 | 15 | 10 |
| Part A · Q3 修复方案（Stamp + Lock） | 15 | 12 |
| Part B · 令牌桶设计与实现 | 50 | 36 |
| **合计** | **100** | **74** |

> ✅ **综合判定**：74 分。有真实的代码产出、压测数据说话，实践能力明显强于上一关。但是 CAS 的 CPU 层面原理描述不够完整，Stamp 方案实现存在根本性思路错误，Part B 的架构选型逻辑虽有亮点但隐患不少，自我发现的 Bug 只有部分被修复。允许进入 Level 3。

---

## 📋 逐题点评

---

### Part A · Q1：ABA 问题定义与分析（16 / 20）

✅ **答对的部分**：
- 精准命名"ABA 问题"，基础无误。
- T1 视角的 CAS 逻辑解释清晰（读到 100 → CAS(100, 0) 成功）。
- 用**栈结构弹出**的例子说明 ABA 在引用类型下的危险性，非常加分——这个例子比数字例子难多了，说明你不只是背了定义。

❌ **扣分点（-4）**：

**① 业务视角的分析方向有偏差（-2）**

> "按照日志顺序的正常来说，应该是 100 扣款成功，然后 50 扣款失败..."

你对"正确行为"的判断本身有模糊。ABA 问题的核心在于：**T1 的 `compareAndSet(100, 0)` 从 CAS 机制角度确实是"合法"的**，因为它读到了 100，也看到了 100，修改成功。问题在于**这个 100 不是同一个语义上的 100**——中间已经经历了"扣 50 → 充 50"的语义变化，代表的是"被二次充值之后的 100"，但 T1 的 CAS 完全感知不到这个中间过程。

你的回答侧重在"日志顺序"，稍微跑偏了正确的发力点：应该强调的是**"值相同不代表状态相同，CAS 只比较值，不比较历史"**。

**② 业务例子中的计算（-2）**

> "三个操作账户总余额：100 - 50 + 50 - 100 = 0"

你说"并不是一个很严重的问题"，但这个算法是**错的**。实际上账户经历了：初始 100 → T2 扣 50 → 变 50 → 充值线程充 50 → 变 100 → T1 扣 100 → 变 0。账面最终是 0，但 **T1 本来应该在 T2 扣款之后失败**（因为 T2 抢先修改了），或在充值后成功扣款。整个过程两次扣款都"成功"了，但时序语义上 T1 的扣款是对"错误时间点的 100"操作，积分类系统里这可能导致超扣。

---

### Part A · Q2：CAS CPU 指令原子性（10 / 15）

✅ **答对的部分**：
- 正确点出了 `CMPXCHG` 指令（x86 架构下的 CAS 实现）。
- 知道这是 CPU 级别的操作，不涉及用户态→内核态切换，解释了和 `synchronized` 重量级锁的本质区别。

❌ **扣分点（-5）**：

**① 单 CPU 场景和多 CPU 场景的"原子性"保证机制完全漏掉（-5）**

`CMPXCHG` 在**单核 CPU** 下本身就是原子的（CPU 顺序执行指令，不会被中断到用户态，当然就是原子的）。
但在**多核 / 多 CPU** 场景下，问题来了：两个核心可以同时访问同一内存地址。这时候 `CMPXCHG` **单靠自己不够**，需要加 **`LOCK` 前缀**：

```asm
LOCK CMPXCHG [mem], reg
```

`LOCK` 前缀的作用是：在执行 `CMPXCHG` 期间，**锁住该内存地址对应的缓存行（Cache Line Lock）或总线（Bus Lock）**，确保其他核心在这段时间内无法读写这块内存，从而在多核下实现原子性。

Java 的 `Unsafe.compareAndSwapInt()` 在 JVM 层面就是生成了 `LOCK CMPXCHG` 指令。你只答了 `CMPXCHG`，漏掉了 `LOCK` 前缀，在多核场景下这个答案是**不完整的**。

**标准答案补充**：

| 场景 | 原子性保证手段 |
|---|---|
| 单核 CPU | `CMPXCHG` 本身是原子的（单核顺序执行） |
| 多核 CPU | `LOCK CMPXCHG`：LOCK 前缀锁住缓存行/总线，禁止其他核心并发访问 |
| JVM 映射 | `Unsafe.compareAndSwapInt()` → HotSpot → `LOCK CMPXCHG`（x86） |

---

### Part A · Q3：修复方案（12 / 15）

#### 方案 A · Stamp（基于版本号）

❌ **根本性思路错误（-2）**

你的实现方式是：
```java
private AtomicReference<AtomicIntegerWithStamp> points =
        new AtomicReference<>(new AtomicIntegerWithStamp(100, 0));
```

这是自己手动包装了一个"值+版本号"的类，然后用 `AtomicReference` 来做 CAS。

这个思路可行，但有一个严重问题：**你 CAS 比较的是对象引用（`AtomicReference`），不是值**。你用 `compareAndSet(old, newPoints)` 时，比较的是 `old` 这个对象的**引用地址**，而不是 stamp 或 value 的内容。

所以当 T2 把 `old (value=100, stamp=0)` 改成 `(value=50, stamp=1)` 之后，T3 再改成 `(value=100, stamp=2)`——此时 stamp 已经从 0 变成了 2，引用地址也变了，**T1 手里的 `old` 引用已经是失效对象**，CAS 会正确失败。

等等，这意味着你的代码逻辑上是**可以正确检测 ABA 的**——不是因为 stamp，而是因为 `AtomicReference` 的引用比较！每次 `new AtomicIntegerWithStamp(...)` 都是新对象，引用必然不同。

> 但这同时意味着 stamp 字段是**多余的**：即使 stamp 相同，引用也不等，CAS 也会失败；即使 stamp 不同，引用不等，CAS 也失败。stamp 在你的实现里没有起到任何实质作用。

JDK 提供的正确工具是 `AtomicStampedReference<V>`，它在底层将值和 stamp 打包，精确地用"值 + stamp 同时比较"来解决 ABA：

```java
AtomicStampedReference<Integer> points = new AtomicStampedReference<>(100, 0);

// 读取当前值和 stamp
int[] stampHolder = new int[1];
int current = points.get(stampHolder);
int currentStamp = stampHolder[0];

// CAS：必须值和 stamp 同时匹配才能成功
points.compareAndSet(current, current - amount, currentStamp, currentStamp + 1);
```

**结论**：你的方向对，但用错了工具；应该用 `AtomicStampedReference`，而不是自己包装。实际运行效果虽然恰好正确（因为引用不等），但背后的**理解是错的**。

#### 方案 B · ReentrantLock（实现正确）

✅ 代码结构完整，`finally` 里正确释放锁，锁粒度合理，适用场景描述（"高竞争场景"）正确。

---

### Part B · 令牌桶限流器设计（36 / 50）

这道题你做了真实的完整实现和压测，值得充分肯定。以下是客观评价：

#### ✅ 亮点

1. **有真实压测数据**，用数字说话，远好于只写伪代码。
2. 使用了 `LongAdder` 做压测统计（并且知道这比 `AtomicLong` 在高并发下更好）——Level 3 才考的东西你已经知道了，加分。
3. `CountDownLatch` 作发令枪控制所有线程"同时起跑"，是正确的压测模式。
4. 发现并修复了 `do-while` 里 CAS 返回值判断反置的 Bug（初版的 `while (tokenPool.compareAndSet(pre, next) && pre != MAX_TOKENS)` 是逆向的）。
5. 自定义拒绝策略（`CallerRunsPolicy` 进阶版：等待 1 秒再尝试入队）思路合理。

#### ❌ 问题点

**① `park/unpark` 的唤醒检查存在 TOCTOU 竞态（-5）**

```java
while (tokenPool.get() <= 0) {
    parkedThreads.add(cur);
    LockSupport.park();
}
```

这段代码存在一个经典的竞态窗口：
1. 线程 A 执行 `tokenPool.get()` → 返回 `> 0`，准备退出 while
2. 调度器切走，其他线程把 token 消耗到了 0
3. 线程 A 继续向后执行进入 `for` 循环，此时 token 已经是 0 了

这就是你在 4000 并发下出现 55% 错误率的根本原因之一——`for` 循环里 `continue` 之后直接返回 `null`，而不是重新 `park`。错误率过高说明在极端并发下，被唤醒的线程大量争抢失败后直接丢弃了请求。

你的第二版用 `isNeedToPark()` 做了修复，but 这里还有一个更深的隐患：

```java
parkedThreads.add(cur);  // ← 先加入队列
LockSupport.park();       // ← 再挂起
```

如果在 `add` 和 `park` 之间，定时器刚好执行了 `unpark(cur)`，那么 `park` 调用会立刻返回（因为 permit 已经被消费了），线程不会真正挂起——这是正确的 `LockSupport` permit 机制。但如果你的 `parkedThreads.removeAll(threads)` 在此之前就把 cur 移除了，下一秒的 unpark 就不会再唤醒它了，线程会一直挂在那里（直到下一秒的 `add` 才被再次唤醒）。这个窗口很窄，但在 4000 并发下会出现。

**② `ReadWriteLock` 的选型根本错误（-5）**

你在第二版说：
> "我打算把 `tryGetToken` 当作一次读操作，然后恢复 500 个桶当作是写操作"

这个类比是错的。`ReentrantReadWriteLock` 的"读"定义是**多线程可以并发读取，但不修改共享状态**。而 `tryGetToken` 里的 `tokenPool--` 是**写操作**（修改了 tokenPool），让它持有读锁是错误的。

你后来自己发现了这个问题（"去了，结果发现我写了都是写锁"），但这不是"写错了写成写锁"的笔误——而是**选型本身就应该是写锁**，因为消费 token 是修改行为。

读写锁在这个场景下并不适合：`tryGetToken`（写）和 `recovery`（写）都是写操作，根本没有"读多写少"的前提，用读写锁没有性能收益，反而增加了理解复杂度。

**③ `tryGetToken` 把任务提交给 `threadPool` 再同步 `get()` 是反模式（-4）**

```java
public static String tryGetToken() {
    Future<?> result = threadPool.submit(() -> { ... });
    return (String) result.get();  // 调用方被阻塞等待
}
```

这是一个"假异步"：你提交任务然后立刻阻塞等结果，相当于同步调用，线程池带来的唯一"好处"是多了一次任务排队。在 4000 虚拟线程并发场景下，真正的瓶颈是你 40 个核心线程的 `threadPool` 处理能力，而不是令牌桶本身，限流效果被线程池天花板干扰了。

正确做法：令牌桶的消费本身就是 `AtomicInteger.decrementAndGet()` 一行原子操作，根本不需要提交给线程池。直接在调用方线程做 CAS 即可：

```java
public static boolean tryConsume() {
    while (true) {
        int cur = tokenPool.get();
        if (cur <= 0) return false;  // 无令牌，直接拒绝
        if (tokenPool.compareAndSet(cur, cur - 1)) return true;
    }
}
```

**④ 高压场景 55% 错误率是架构问题，不只是参数问题（-0，已在③扣过）**

4000 并发下大量 `异常崩溃数` 来自 `RejectedExecutionException`（队列满了，拒绝策略等待 1 秒还是满），这是线程池配置和压测场景不匹配的问题，不是令牌桶本身的问题。从限流器的正确性角度看，**最终消费的 token 数量没有超出上限**（你的结果计算：无负数，无超出），这说明**令牌桶的核心逻辑是正确的**，这一点要肯定。

---

## 📊 能力雷达图（文字版）

| 能力维度 | 评价 |
|---|---|
| ABA 问题理解 | ⭐⭐⭐⭐ 良好，栈结构类比说明理解到位 |
| CAS CPU 原理 | ⭐⭐⭐ 中等，知道 CMPXCHG 但漏掉了多核下的 LOCK 前缀 |
| AtomicStampedReference | ⭐⭐ 较弱，方向对但用错了工具，对 JDK 标准实现不够熟悉 |
| ReentrantLock 使用 | ⭐⭐⭐⭐ 良好，代码结构正确，finally 释放锁正确 |
| 工程实践 / 压测 | ⭐⭐⭐⭐ 良好，有完整代码 + 压测数据，主动发现并修复 Bug |
| 并发架构设计 | ⭐⭐⭐ 中等，park/unpark 使用有竞态隐患，ReadWriteLock 选型错误 |
| 自我 Debug 能力 | ⭐⭐⭐⭐ 良好，能发现问题并多次迭代（从 AI 处问对了问题） |

---

## 🎯 进入 Level 3 前的补课清单

Level 3 的战场是 **JMM 内存模型 + volatile + CAS 进阶（LongAdder）**，这里有几个和你本关答案直接挂钩的点：

1. **必看**：黑马 `06.013 ~ 06.014`（`AtomicStampedReference` / `AtomicMarkableReference`）——搞清楚 JDK 提供的 ABA 解决方案，你本关用的自制版本不是正确答案
2. **必看**：黑马 `06.019 ~ 06.026`（`LongAdder` 原理 + `Cell` 缓存行伪共享）——Level 3 的核心考点之一，你在答案里用了 `LongAdder` 统计但没有说出为什么，下一关你要能说清楚
3. **带着问题去看**：`LOCK CMPXCHG` 在多核场景下锁的是总线还是缓存行？JDK 新版本里有什么优化？（这是极客挑战方向）

---

## ✅ 导师判定

> **74 分，允许进入 Level 3。**
>
> 你的最大亮点是**敢于写真实代码、敢于压测、敢于 debug**。靠这种实践节奏学习，吸收速度比只看视频快得多。
>
> 下一关（JMM + volatile + CAS 进阶）是你目前的弱点区：CAS 的底层 CPU 语义你只答了一半，ABA 的 JDK 标准解法你用错了。Level 3 会在这两个点上直接追问。

