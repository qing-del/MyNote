# Level 1 · 评分报告

> **考核模块**：并发基石与内存模型 (JMM)
> **考核日期**：2026-03-21
> **评分规则**：百分制，Part A（60分）+ Part B（40分）

---

## 🏆 总分

| 模块 | 满分 | 实得 |
|---|---|---|
| Part A · Q1 线程状态判断 | 15 | 13 |
| Part A · Q2 WAITING vs BLOCKED 底层区别 | 25 | 15 |
| Part A · Q3 interrupt 行为分析 | 20 | 12 |
| Part B · Q1 Bug 识别 | 10 | 7 |
| Part B · Q2 代码修复 | 20 | 17 |
| Part B · Q3 MySQL 类比 | 10 | 3 |
| **合计** | **100** | **67** |

> ⚠️ **综合判定**：67 分。基础结构清晰，有自己的思考，但 **AQS 理解有混淆，interrupt 边界分析不够精确，MySQL 类比严重跑偏**。建议针对薄弱点补课后可进入 Level 2，但 Level 2 题目中会延续考察这些概念。

---

## 📋 逐题点评

---

### Part A · Q1：线程状态判断（13 / 15）

✅ **答对的部分**：
- thread-0 `RUNNABLE`、thread-1 `BLOCKED`、thread-3 `WAITING` 三者判断完全正确。
- 能从 jstack 识别出"等什么"（等锁地址 / 等 unpark）。

❌ **扣分点（-2）**：
- "thread-3 在 OS 层面处于条件等待队列"——这个说法不够精确。`LockSupport.park()` 在 OS 层面让线程挂起，但并不是进入 OS 的"条件变量等待队列"，而是通过 `Unsafe.park()` 触发 **许可证（permit）机制**，在 JVM 层面由 `_parker` 对象持有。
- 关于"为什么卡死"的推测部分，两种推测都有道理，但没有聚焦到题目要求的"状态是什么、在等什么"，有**答偏**的嫌疑。题目本身没让分析根本原因。

**标准答案补充**：
```
thread-0: RUNNABLE       — 正在执行 calculateDiscount()，持有 OrderService 实例锁
thread-1: BLOCKED        — 竞争 <0x...d6e01f28>，在 Monitor 的 EntryList 中排队
thread-3: WAITING        — 调用了 park()，挂起在 AQS 的 CLH 队列节点上，等待 unpark() 或 interrupt()
```

---

### Part A · Q2：WAITING vs BLOCKED 底层区别（15 / 25）

✅ **答对的部分**：
- 正确识别出"主动停/被动停"的本质区别。
- 知道 `WaitSet` vs `EntryList` 的 Monitor 层面区分。
- 提到了 AQS 和 CLH Queue。

❌ **严重扣分点（-10）**：

**① 对象头描述完全混乱（-5）**

> "如果使用 park() 的时候，对象头如果不是重量级锁，会被占有线程主动申请"

这是**根本性错误**。`LockSupport.park()` 和对象头**没有任何关系**。`park/unpark` 是 AQS 构建显式锁的基础原语，它操作的是线程的内部 `_parker` 对象，**不涉及任何 Java 对象的 Mark Word**。

对象头 (Mark Word) 只在 `synchronized` 场景下才与锁状态有关：
- 无锁 → 偏向锁 → 轻量级锁 → 重量级锁，这四种状态都记录在 Mark Word 的 **lock bits（低2位）** 中
- 只有膨胀为重量级锁时，Mark Word 才存的是 `ObjectMonitor*` 指针，Monitor 的 `EntryList` 才出现

**② AQS 描述有严重概念混淆（-5）**

> "`BLOCKED(on object monitor)` 是因为 state==0 而...进入 ObjectCondition 队列"

`BLOCKED (on object monitor)` 是 `synchronized` 关键字产生的，**AQS 根本不参与 synchronized**！`synchronized` 走的是 JVM 内置的 `ObjectMonitor`（C++ 实现），而 AQS 是 `java.util.concurrent` 包中 `ReentrantLock`/`Semaphore` 等显式锁的骨架。两套体系**完全独立**，不要混用。

**标准对比表**：

| 维度 | `WAITING (parking)` / AQS | `BLOCKED (on object monitor)` / synchronized |
|---|---|---|
| 底层机制 | `LockSupport.park()` → Unsafe → OS park | JVM ObjectMonitor.enter() → EntryList 排队 |
| 对象头关系 | **无关** | Mark Word 低位标志锁状态，重量级时存 Monitor 指针 |
| 在哪排队 | AQS CLH 双向链表（Node） | ObjectMonitor._EntryList |
| 能被 interrupt 打断？ | ✅ 可被 interrupt 唤醒 | ❌ 不可被 interrupt 打断，只能等锁 |
| wait() 后去哪 | — | ObjectMonitor._WaitSet |

---

### Part A · Q3：interrupt 行为分析（12 / 20）

✅ **答对的部分**：
- 核心思路正确：interrupt 是"协作式"的，不是强制停止。
- 知道需要线程自己处理打断信号并释放锁。
- 知道 thread-3 需要等 unpark 才能继续。

❌ **扣分点（-8）**：

**① 题目问的是对 thread-0 执行 interrupt，你需要分两种场景：**

**场景 A：thread-0 持有 `synchronized` 内置锁**
- `interrupt()` 只是给 thread-0 设置中断标志位（`isInterrupted() = true`）
- thread-0 **不会立即停**，它会继续执行直到代码里主动检查 `Thread.interrupted()` 或调用会抛 `InterruptedException` 的方法（如 `sleep`、`wait`）
- `synchronized` 的 `BLOCKED` 状态**对 interrupt 完全免疫**——thread-1 依然在等，不会被唤醒

**场景 B：thread-0 持有 `ReentrantLock`（非 lockInterruptibly）**
- 与 synchronized 基本相同，持有锁的线程不响应 interrupt，只是标志位被置位

**场景 C（你遗漏了）：如果 thread-0 持有 `ReentrantLock` 且调用了 `lock.lockInterruptibly()`**
- 如果此时 thread-0 **还没拿到锁**，interrupt 会让它抛出 `InterruptedException` 并出队
- 但从 jstack 看它已经是 RUNNABLE（已经拿到锁了），所以这个场景在本题中不适用

**② 对 thread-1 的分析缺失（-4）**：你没有回答 interrupt 之后 thread-1 会怎样。答案是：不管 thread-0 因 interrupt 还是正常执行释放锁，thread-1 都只能等 thread-0 释放锁，`BLOCKED` 对 interrupt **不感冒**。

---

### Part B · Q1：Bug 识别（7 / 10）

✅ **答对的核心**：
- 正确识别 `if` 应改为 `while`（虚假唤醒问题）
- 正确识别 `notify()` 可能唤醒错误线程
- 场景描述逻辑清晰

❌ **扣分点（-3）**：

**遗漏了第三个 Bug**：`lock.wait()` 没有捕获 `InterruptedException`。题目注释说"有异常处理省略"，但实际上 `wait()` 声明了 `throws InterruptedException`，如果调用线程被 interrupt，会直接从 wait 中抛出异常，跳过后续的数据处理逻辑，这是一个真实的生产隐患。

另外，你的场景描述（用户A/B数据）有轻微偏差：`notify()` 的问题不在于"唤醒了处理另一个用户的线程"，而在于：当多个线程都在 `WaitSet` 等待同一个条件时，`notify()` 随机唤醒其中一个，而这个线程被唤醒后不会再次检查条件（`if` 而非 `while`），可能在条件**根本不满足**的情况下就执行了业务逻辑。

---

### Part B · Q2：代码修复（17 / 20）

✅ **答对的部分**：
- `if → while` 修复正确，注释到位。
- `notify() → notifyAll()` 修复正确。
- 额外给出了 `ReentrantLock + Condition` 精确唤醒方案，**加分项**，思路完全正确。

❌ **扣分点（-3）**：
- `ReentrantLock + Condition` 方案只说了思路，没有给出完整代码。导师期望你能**写出来**，哪怕是伪代码级别的完整结构。以下是标准示范：

```java
private final ReentrantLock lock = new ReentrantLock();
// 每个 userId 对应一个 Condition
private final Map<Long, Condition> conditionMap = new ConcurrentHashMap<>();

public void waitForData(long userId) throws InterruptedException {
    Condition cond = conditionMap.computeIfAbsent(userId, k -> lock.newCondition());
    lock.lock();
    try {
        while (!isDataReady(userId)) {  // 仍然必须是 while
            cond.await();
        }
        // 处理数据...
    } finally {
        lock.unlock();  // 必须在 finally 里释放！
    }
}

public void notifyDataReady(long userId) {
    Condition cond = conditionMap.get(userId);
    if (cond != null) {
        lock.lock();
        try {
            // ... 标记数据就绪
            cond.signal();  // 精确唤醒
        } finally {
            lock.unlock();
        }
    }
}
```

> ⚠️ 注意：`lock.unlock()` **必须在 `finally` 块中**，否则发生异常会导致锁永远不释放（死锁）。你的简单版也没有写 finally。

---

### Part B · Q3：MySQL 类比（3 / 10）

❌ **严重跑偏（-7）**：

你说的 `Double Write` 机制和本题要类比的"等待-通知"**毫无关系**。Double Write 是解决**页面断裂**（partial write）的持久化问题，与线程间同步通知根本不在同一个概念层面。

**正确类比方向**：

InnoDB 中确实有"等待-通知"机制，主要体现在**行锁等待**场景：

```
场景：事务 A 正在修改某行（持有行锁），事务 B 想修改同一行

InnoDB 的实现：
1. 事务 B 发现锁冲突 → 进入 lock_wait（类似 Java 的 WaitSet）
2. lock_wait 本质是操作系统级别的 IO 等待（调用了 mutex 或 cond_wait）
3. 事务 A 提交 → InnoDB 调用 lock_release → 唤醒等待队列中的事务 B（类似 notify）
4. 事务 B 重新检查是否能拿到锁（类似 while 循环检查条件）
```

类比关系：

| Java | MySQL InnoDB |
|---|---|
| `synchronized` + `wait()` | 行锁 + `lock_wait` |
| `WaitSet` | `lock_sys.wait_array` |
| `notifyAll()` | `lock_release_all_on_table()` |
| `while (!condition)` 重新检查 | 锁等待超时后重新尝试加锁 |
| 死锁（线程互等） | 死锁检测（`SHOW ENGINE INNODB STATUS` 中的 DEADLOCK 信息） |

---

## 📊 能力雷达图（文字版）

| 能力维度 | 评价 |
|---|---|
| 线程状态感知 | ⭐⭐⭐⭐ 良好，jstack 基本读得懂 |
| synchronized/Monitor 原理 | ⭐⭐⭐ 中等，WaitSet/EntryList 清楚，但对象头掌握弱 |
| AQS / LockSupport | ⭐⭐ 较弱，和 synchronized 混淆严重 |
| interrupt 机制 | ⭐⭐⭐ 中等，知道协作式，但边界条件分析不全 |
| wait/notify 实战 | ⭐⭐⭐⭐ 良好，核心问题识别准确 |
| MySQL 横向类比 | ⭐ 薄弱，需要重新建立正确映射 |

---

## 🎯 进入 Level 2 前的补课清单

在 Level 2（AQS & ReentrantLock）中，AQS 是核心战场。你当前的弱点恰好是 Level 2 的地雷区，建议**在进入 Level 2 之前**（或同步进行）：

1. **必刷**：黑马 `08.037 aqs-概述` + `08.038 aqs-自定义锁` — 彻底搞清楚 AQS 和 synchronized 是两套体系
2. **必刷**：黑马 `04.026 Monitor-对象头` — 理解 Mark Word 4种锁状态的位标志
3. **选看**：《Java并发编程的艺术》第2章 §2.3（`synchronized` 实现原理） + 第5章 §5.2（AQS 框架）
4. **思考题（带着问题去看）**：`ReentrantLock.lock()` 在 AQS 层面做了什么？为什么它能支持 interrupt 而 synchronized 不行？

---

## ✅ 导师判定

> **67 分，允许进入 Level 2，但有附加条件。**
>
> Level 2（AQS & ReentrantLock）的题目中，我会直接考察 AQS 与 synchronized 的边界，以及 interrupt 在 AQS 层面的完整行为。你在 Level 1 中的遗漏会被当作"已知条件"直接使用，不再补充解释。
>
> 所以：**带着这份扣分项去打 Level 2，刚好可以在实战中弥补。**
