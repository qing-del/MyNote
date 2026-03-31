# Level 4 · 评分报告

> **考核模块**：AQS 框架与显式锁推演 —— ReentrantLock 源码流程 · Semaphore 令牌桶
> **考核日期**：2026-03-31
> **评分规则**：百分制，Part A（60分）+ Part B（40分）

---

## 🏆 总分

| 模块 | 满分 | 实得 |
|---|---|---|
| Part A · Q1 lock() 完整调用链 | 20 | 14 |
| Part A · Q2 公平 vs 非公平锁两处差异 | 20 | 16 |
| Part A · Q3 release() 路径 + waitStatus 语义 | 20 | 14 |
| Part B · Q1 Semaphore 原理 | 10 | 8 |
| Part B · Q2 Semaphore 令牌桶实现 | 20 | 14 |
| Part B · Q3 release() AQS 传播 | 10 | 8 |
| **合计** | **100** | **74** |

> ✅ **综合判定**：74 分。核心流程的主干脉络你已经掌握，公平/非公平锁的差异点定位精准，能用 Semaphore 写出可运行的令牌桶并识别"惊群效应"问题。扣分集中在：`lock()` 调用链的细节层次不够（漏掉 `addWaiter` 和 `acquireQueued` 的关键逻辑）、`waitStatus` 的值方向理解有误、Semaphore 方案仍残留 `threadPool + result.get()` 反模式。允许进入 Level 5。

---

## 📋 逐题点评

---

### Part A · Q1：lock() 完整调用链（14 / 20）

✅ **答对的部分**：
- 流程图的主干正确：`lock()` → CAS 失败 → `acquire(1)` → `tryAcquire()` → 重入判断 → 入队 park → 唤醒重试。
- 重入锁的 `state + 1` 逻辑正确（`Owner == 当前线程` 时直接累加）。
- 非公平锁在 `lock()` 入口处直接 CAS 抢锁（不走 `acquire`）这一关键特征画出来了。

❌ **扣分点（-6）**：

**① `addWaiter` 的入队细节完全缺失（-3）**

你的流程图里，"加入 CLH 队列" 这一步只有一个框，没有展开。实际上 `addWaiter(Node.EXCLUSIVE)` 有两个重要的分支：

```
加入队列时:
  - 如果 tail 不为 null（队列已存在）: CAS 将新节点设为 tail（快速路径）
  - 如果 tail 为 null（队列不存在）: 进入 enq() → 自旋 CAS 初始化 head（哨兵节点）再设 tail
```

**哨兵节点（dummy head）的存在是 AQS 的核心设计**——head 始终是一个空节点，不代表任何线程，真正的等待线程从 `head.next` 开始。你的图里没有体现这个。

**② `acquireQueued` 的自旋逻辑缺失（-3）**

入队后不是直接 `park()`，而是在 `acquireQueued()` 里先做一次判断：

```java
// acquireQueued 核心逻辑（简化）
for (;;) {
    Node p = node.predecessor();
    if (p == head && tryAcquire(arg)) {  // ← 如果前驱是 head，再尝试一次
        setHead(node);
        p.next = null;  // 帮助 GC
        return interrupted;
    }
    if (shouldParkAfterFailedAcquire(p, node))  // ← 判断是否应该 park
        parkAndCheckInterrupt();  // ← 才真正 park
}
```

关键逻辑：**只有当前驱节点的 `waitStatus` 被设置为 `SIGNAL(-1)` 后，当前节点才最终 park**。`shouldParkAfterFailedAcquire` 负责把前驱节点的 status 改为 SIGNAL，这是唤醒机制的前提。你的图跳过了这一步。

**标准调用链补充**：
```
lock()
  └─ NonfairSync.lock()
       ├─ 直接 CAS(0→1) 成功 → setExclusiveOwnerThread → 结束
       └─ 失败 → acquire(1)
            └─ tryAcquire(1)（非公平版：不检查队列）
                 ├─ state==0 → CAS成功 → 返回true
                 ├─ 当前线程是Owner → state++ → 返回true
                 └─ 失败 → addWaiter(EXCLUSIVE)
                           → acquireQueued(node, 1)
                                → shouldParkAfterFailedAcquire（设前驱SIGNAL）
                                → parkAndCheckInterrupt（park挂起）
                                → 被unpark后重新进入自旋 tryAcquire
```

---

### Part A · Q2：公平 vs 非公平锁两处差异（16 / 20）

✅ **主要答对**：
- 精确指出了 `tryAcquire()` 中公平锁多了 `!hasQueuedPredecessors()` 判断——这是第二处差异，答对了。
- 贴出了两版 `tryAcquire()` 源码对比，直观清晰。
- 饥饿原因解释正确：新来线程不断"插队"，队列中的线程每次醒来都发现锁被抢走，反复 park。

❌ **扣分点（-4）**：

**① 漏掉了第一处差异（-4）**

公平锁 vs 非公平锁的差异有**两处**：

| | 非公平锁 `NonfairSync` | 公平锁 `FairSync` |
|---|---|---|
| **差异1：`lock()` 入口** | 直接 `compareAndSetState(0, 1)` 抢锁，失败才 `acquire(1)` | 直接调用 `acquire(1)`，**不抢锁** |
| **差异2：`tryAcquire()` 内** | `state==0` 时直接 CAS | `state==0` 时先 `!hasQueuedPredecessors()` 再 CAS |

你的答案只说了 `tryAcquire` 内的差异（差异2），完全漏掉了 `lock()` 方法本身的差异（差异1）。非公平锁在 `lock()` 入口就做了一次"插队尝试"，这是两者吞吐量差距最大的来源——大量线程在进入 AQS 的完整路径之前，就直接 CAS 成功拿锁了。

---

### Part A · Q3：release() 路径 + waitStatus（14 / 20）

✅ **答对的部分**：
- `tryRelease()` 的流程基本正确：判断 Owner → 计算新 state → 完全释放才清 Owner → 触发唤醒。
- 重入锁的递减（"递归调用"释放，虽然描述方式有点奇怪，但含义对）。
- 唤醒时跳过无效节点（CANCELLED 节点），寻找下一个有效节点——这个细节答出来了。

❌ **扣分点（-6）**：

**① `waitStatus` 的"有效"判断方向写反了（-4）**

> "当 `status != 0` 的时候才可以唤醒，因为 0 一般是刚入队，处于醒的状态，不需要唤醒"

这是**错误的**。`unparkSuccessor` 的实际判断逻辑是：

```java
// unparkSuccessor 核心逻辑
Node s = node.next;
if (s == null || s.waitStatus > 0) {  // ← waitStatus > 0 表示 CANCELLED，跳过
    s = null;
    // 从 tail 向前找最近的 waitStatus <= 0 的节点
    for (Node t = tail; t != null && t != node; t = t.prev)
        if (t.waitStatus <= 0)
            s = t;
}
if (s != null)
    LockSupport.unpark(s.thread);
```

正确逻辑是：**`waitStatus <= 0` 才是有效节点**（包含 SIGNAL=-1、CONDITION=-2、PROPAGATE=-3）；**`waitStatus > 0`（即 CANCELLED=1）才是无效节点，需要跳过**。

能被唤醒的前提是节点的 `waitStatus` 为 **`SIGNAL(-1)`**——这是在 `shouldParkAfterFailedAcquire` 里由前面的节点设置的，表示"我的后继需要被我唤醒"。等于 0 的节点是刚入队还没被设置 SIGNAL 的，它的前驱还没到唤醒它的时机，**不是"处于醒的状态"**。

**② `unparkSuccessor` 从尾向前遍历没有提及（-2）**

你的流程图写的是"剔除无效节点并获取下一个节点"，但没有说明这里有个关键设计：**从 tail 向前遍历**（`t = t.prev`）而不是从 `head.next` 向后遍历。原因是 `addWaiter` 中 `prev` 指针先于 `next` 指针建立，新节点的 `prev` 是可靠的，而 `next` 可能刚被 CAS 成功但还没 `pred.next = node` 就被抢占，造成 `next` 指针短暂断链。

---

### Part B · Q1：Semaphore 原理（8 / 10）

✅ **核心描述正确**：`acquire()` 消耗 `state` 中的许可证数，不足时进 CLH 队列 park；`release()` 增加 `state` 许可证数并唤醒等待节点——方向完全正确。

❌ **扣分点（-2）**：

**Semaphore 是共享锁，唤醒逻辑与独占锁不同（-2）**

你描述的是独占锁的 AQS 框架，对 `Semaphore` 的共享锁特性没有展开。`Semaphore.acquire()` 走的是 `acquireSharedInterruptibly(1)` → `tryAcquireShared()` → 返回负数时才入队。

共享锁和独占锁最大的区别是**唤醒传播**：独占锁 `unpark` 后只唤醒一个节点；共享锁 `release()` 后，如果 `state` 还有剩余许可证，会继续唤醒后面的等待节点（`doReleaseShared` → `PROPAGATE` 状态的来源）。这个传播机制在你 Level 2 的自制方案里是靠"惊群效应"实现的，`Semaphore` 的做法是有序传播。

---

### Part B · Q2：Semaphore 令牌桶实现（14 / 20）

✅ **亮点**：
- 令牌补充逻辑正确：`Math.min(MAX_TOKENS - availablePermits(), RECOVERY_NUM)` 防止超出上限。
- `tryAcquire(1)` 非阻塞消费 + 超时自旋的思路合理，避免了无限阻塞。
- 正确识别了 Level 2 自制方案的"惊群效应"问题，并说出了 `Semaphore` 的有序唤醒优势。
- 压测令牌守恒：4000 并发下 `13500 - 13040 = 460`，令牌没有超发，核心正确性通过。

❌ **扣分点（-6）**：

**① `threadPool.submit() + result.get()` 反模式依然存在（-4）**

```java
public static String tryGetToken() {
    Future<String> result = threadPool.submit(() -> { ... });
    return result.get();  // ← 调用方被阻塞
}
```

这是 Level 2 就指出过的"假异步"问题，本关仍未修复。`Semaphore.tryAcquire()` 是线程安全的无锁操作（内部 CAS），**完全不需要额外的线程池包装**。4000 并发下 72% 错误率中的大部分 `异常崩溃数=34057` 正是来自线程池拒绝，而不是令牌耗尽。

正确做法（直接在调用方线程操作，删掉 threadPool）：
```java
public static String tryGetToken() {
    long deadline = System.currentTimeMillis() + MAX_WAIT_TIME;
    try {
        long remaining = deadline - System.currentTimeMillis();
        if (remaining > 0 && tokenPool.tryAcquire(1, remaining, TimeUnit.MILLISECONDS)) {
            return JwtTokenGenerator.generateToken();
        }
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
    }
    return null;  // 超时返回 null
}
```

> 用 `Semaphore.tryAcquire(permits, timeout, unit)` 直接支持带超时的阻塞等待，不需要自己写 `while` 自旋，这才是 AQS 封装的价值所在——你用了 `Semaphore` 但没有用到它最关键的这个方法签名。

**② 超时自旋的 `while (!tryAcquire)` 本质是忙等（-2）**

```java
while (!tokenPool.tryAcquire(1)) {
    passTime = System.currentTimeMillis() - begin;
    if (passTime > MAX_WAIT_TIME) return null;
}
```

`tryAcquire()` 失败时立刻重试，这是**忙等（busy-wait）**，会疯狂消耗 CPU。在 4000 并发下每个等待的线程都在疯狂 CAS，这比 Level 2 的 `park()` 还要差。正确姿势是 `tryAcquire(1, timeout, TimeUnit.MILLISECONDS)`，让 AQS 内部帮你 park 等待，到时间自动唤醒。

---

### Part B · Q3：release() AQS 传播（8 / 10）

✅ **核心正确**：令牌补充时调用 `semaphore.release(recoveryNum)`，这会增加 `state` 并按量唤醒 CLH 队列中的等待线程。

❌ **扣分点（-2）**：

> "将需要加入的许可证数量放入到 `status` 中"

字段名是 `state`，不是 `status`。这不只是笔误，`status` 可能和 `Node.waitStatus` 混淆。`state` 是 `AbstractQueuedSynchronizer` 的核心字段，`Semaphore` 用它存放当前剩余许可证数量；`waitStatus` 是 CLH 队列中每个 `Node` 的状态字段，两者是完全不同的东西。

---

## 📊 能力雷达图（文字版）

| 能力维度 | 评价 |
|---|---|
| AQS 主干流程（lock/unlock） | ⭐⭐⭐⭐ 良好，主干正确，addWaiter / acquireQueued 细节缺失 |
| 公平/非公平锁差异定位 | ⭐⭐⭐ 中等，差异2（tryAcquire）找到了，差异1（lock入口）漏掉 |
| waitStatus 语义 | ⭐⭐ 较弱，有效/无效方向判断反了，SIGNAL含义理解有偏差 |
| Semaphore 原理 | ⭐⭐⭐ 中等，方向对但缺少共享锁传播特性的说明 |
| 工程实践（令牌桶） | ⭐⭐⭐ 中等，用对了工具但没用对最关键的 API，busy-wait 问题仍在 |
| 自我反思能力 | ⭐⭐⭐⭐ 良好，识别出了惊群效应，说明对 Level 2 的问题有真正回顾 |

---

## 🎯 进入 Level 5 前的补课清单

Level 5 的战场是**线程池工程实战**，考察 `ThreadPoolExecutor` 核心参数联动、拒绝策略、OOM 场景，以及 `CountDownLatch`/`CompletableFuture` 多任务协调。你的令牌桶代码恰好暴露了线程池参数配置的问题，Level 5 会正面拆解这些。

1. **必看**：黑马 `08.040 ~ 08.043`（ReentrantLock 完整加锁/解锁流程）——本关扣分集中在细节层，建议重新完整看一遍，重点关注 `addWaiter` 里哨兵节点的初始化
2. **必看**：黑马 `08.011 ~ 08.013`（`ThreadPoolExecutor` 构造方法 + 池状态）——Level 5 主战场，带着"我的 threadPool 为什么在 4000 并发下爆"的问题去看
3. **订正并记住**：`waitStatus <= 0` 才是有效节点，`waitStatus > 0`（CANCELLED）才是无效节点；`SIGNAL = -1` 是"需要被前驱唤醒"的标志
4. **带着问题去看**：`Semaphore.tryAcquire(n, timeout, unit)` 的底层是怎么实现带超时的 park 的？超时后 AQS 如何取消节点（`cancelAcquire`）？

---

## ✅ 导师判定

> **74 分，允许进入 Level 5。**
>
> 这一关最大的进步是：你终于知道 AQS 和 synchronized 是两套体系，并且能在流程图里说清楚 `acquire → tryAcquire → addWaiter → park` 的主干——这是 Level 1 欠下的旧账，算是还清了一半。
>
> 没还清的那一半是 `waitStatus` 的方向和 `threadPool + result.get()` 反模式。后者在令牌桶里连续两关都出现了，Level 5 里如果还看到这个写法，扣分力度会加倍。
