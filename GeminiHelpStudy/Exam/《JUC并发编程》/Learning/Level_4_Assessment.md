# Level 4：AQS 框架与显式锁推演 —— 引导式学习问题册

> Level 1 你把 AQS 和 synchronized 搅成了一锅粥；Level 2 你把 ReadWriteLock 的读写语义用错了；Level 3 你把 MESI 总线风暴讲得漂亮，CAS 的 `LOCK` 前缀也补上了。
>
> 现在带着这些积累，正式踏入 AQS 的钢铁丛林。这一关的对手是 ReentrantLock 的源码骨架、公平锁与非公平锁的实际差异、读写锁的升级/降级规则，以及锁超时的取消机制。你在 Level 2 里写的令牌桶，真正的生产级做法应该用什么——这里会给你答案。

---

## 📋 本 Level 考核范围

| 知识板块 | 核心考点 | 对应黑马章节 |
|---|---|---|
| AQS 骨架 | CLH 双向队列、`state` 语义、`Node` 状态流转 | 第八章 08.037 ~ 08.039 |
| ReentrantLock 非公平锁 | `lock()` 加锁全流程（加锁成功 / 加锁失败入队 / 解锁竞争）| 第八章 08.040 ~ 08.043 |
| ReentrantLock 公平锁 | 与非公平锁的源码差异、`hasQueuedPredecessors()` | 第八章 08.046 |
| 可打断锁 | `lockInterruptibly()` 在 AQS 层面的实现与普通 `lock()` 的区别 | 第八章 08.045 |
| ReentrantReadWriteLock | 读写状态编码（高16位/低16位）、锁降级规则、不支持锁升级的原因 | 第八章 08.049 ~ 08.060 |

---

## 🔥 大题一：AQS 骨架与 ReentrantLock —— 订单服务的"锁风暴"诊断

### Part A · 基础理论 —— "排队的艺术"

你的团队在生产环境中遇到了一个性能问题：大促期间，订单服务使用了 `ReentrantLock` 的**非公平锁**，发现有少量线程长时间拿不到锁（饥饿），但整体吞吐量比切换成**公平锁**高不少。

```java
// 非公平锁（默认）
ReentrantLock lock = new ReentrantLock();

// 公平锁
ReentrantLock fairLock = new ReentrantLock(true);
```

**请回答以下问题：**

1. `ReentrantLock.lock()` 在 AQS 层面完整的调用链是什么？请从 `lock()` 开始，逐步追踪到线程最终被 `park()` 挂起的过程（关键方法名：`acquire` → `tryAcquire` → `addWaiter` → `acquireQueued` → `parkAndCheckInterrupt`）。

2. 非公平锁和公平锁在 `lock()` 源码上的差异只有**两处**，请精确指出这两处差异，并解释为什么非公平锁比公平锁吞吐量更高，但可能造成饥饿。

3. AQS 内部的 CLH 队列中，`Node` 节点有以下几种 `waitStatus`：

   | waitStatus | 值 | 含义 |
   |---|---|---|
   | `CANCELLED` | 1 | 线程已被取消（超时/中断） |
   | `SIGNAL` | -1 | 后继节点需要被唤醒 |
   | `CONDITION` | -2 | 在 Condition 队列中等待 |
   | `PROPAGATE` | -3 | 共享锁传播唤醒 |

   当 `thread-A` 持有锁，`thread-B` 和 `thread-C` 依次在 CLH 队列中排队，此时：
   - `thread-A` 调用 `unlock()` 释放锁，AQS 如何选择唤醒谁？完整描述 `release()` → `tryRelease()` → `unparkSuccessor()` 的执行路径。
   - `thread-B` 的 `waitStatus` 应该是什么值，才能被 `unparkSuccessor()` 唤醒？为什么不是 0？

### Part B · 场景实战 —— "令牌桶的正确打开方式"

在 Level 2 中，你用 `AtomicInteger` + 自定义 `park/unpark` 实现了令牌桶。导师当时指出你的方案有"假异步"和竞态窗口的问题。

现在用你在 Level 4 学到的 `Semaphore`（基于 AQS 实现的共享锁）来重新设计令牌桶限流器：

**需求**（同 Level 2）：
- 令牌桶容量 1000，每秒补充 500 个令牌
- 多线程并发消费令牌（每次请求消耗 1 个）
- 令牌耗尽时：**等待策略**（不直接拒绝，最多等待 500ms）

**请完成以下任务：**

1. 说明 `Semaphore` 的内部实现原理：`acquire()` 方法如何利用 AQS 的 `state` 字段和 CLH 队列来实现"令牌消耗"与"线程阻塞"？

2. 给出基于 `Semaphore` 的完整令牌桶实现（核心代码，不需要压测框架），并解释为什么这个实现比你在 Level 2 中的自制 `park/unpark` 方案更安全。

3. （对比 Level 2）你在 Level 2 的方案里，`scheduler` 唤醒等待线程是通过手动遍历 `LinkedBlockingQueue<Thread>` + `LockSupport.unpark()`。在 `Semaphore` 方案中，令牌补充时应该调用什么方法？这个方法在 AQS 层面做了什么？

---

> ⏳ **请回答【大题一】全部内容后，我再出大题二（ReentrantReadWriteLock 读写锁降级 + 可打断锁）。**
>
> 如果某个子问题不确定，直接说"这部分不太会，给个方向"，我会告诉你去看黑马哪一节。

---

### 【极客挑战】（选做，可留给未来）

**极客挑战 1 · `unparkSuccessor` 为什么从尾部向前遍历**

> 在 `unparkSuccessor(Node node)` 中，寻找离 head 最近的有效后继节点时，代码是**从 tail 向前（`t = t.prev`）遍历**，而不是从 head 向后（`s = s.next`）。
> - 为什么不能直接用 `node.next` 找后继？在什么情况下 `node.next` 是不可靠的？（提示：`addWaiter` 方法里 `node.next` 和 `pred.next` 的赋值顺序）
> - 从尾部向前遍历能解决这个问题的根本原因是什么？（提示：`prev` 指针和 `next` 指针哪个先被建立？）

**极客挑战 2 · `ReentrantReadWriteLock` 的状态编码设计**

> `ReentrantReadWriteLock` 用一个 `int state` 同时表示读锁和写锁的状态：高 16 位表示读锁持有次数，低 16 位表示写锁重入次数。
> - 这种位运算编码方式是怎么实现的？给出读锁计数和写锁计数的提取公式（用位运算表示）。
> - 为什么不用两个独立的 `AtomicInteger` 分别记录读锁和写锁，而要挤在一个 `int` 里？这和 AQS 只有一个 `state` 字段有什么根本关系？

**极客挑战 3 · 锁升级为什么不被支持**

> `ReentrantReadWriteLock` 支持**锁降级**（写锁 → 读锁），但不支持**锁升级**（读锁 → 写锁）。
> - 用一个具体的死锁场景，证明"允许锁升级"会导致死锁。
> - Java 21 引入的 `StampedLock` 中的乐观读锁如何绕过这个问题？它和读写锁的本质区别是什么？

---

> 📌 **精准学习路径（针对你的历史弱点）**：
>
> | 薄弱点 | 推荐视频 | 推荐书籍 |
> |---|---|---|
> | AQS 骨架 + CLH 队列 | 08.037 ~ 08.039 | 《Java并发编程的艺术》§5.2 |
> | ReentrantLock 非公平锁全流程 | 08.040 ~ 08.043 | 《Java并发编程的艺术》§5.3 |
> | 公平锁 vs 非公平锁源码差异 | 08.046 | 《Java并发编程的艺术》§5.3.4 |
> | 可打断锁 lockInterruptibly | 08.045 | — |
> | ReadWriteLock 状态编码 + 锁降级 | 08.049 ~ 08.060 | 《Java并发编程的艺术》§5.4 |
> | Semaphore 原理（acquire/release） | 08.066 ~ 08.067 | — |
