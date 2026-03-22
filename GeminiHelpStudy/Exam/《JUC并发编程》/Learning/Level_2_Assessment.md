# Level 2：底层框架与显式锁 (AQS & ReentrantLock) —— 引导式学习问题册

> 第一关你过了——虽然 AQS 和 synchronized 的边界被你搅成了一锅粥，但我给你机会在这里赎回来。
> Level 2 的战场是 AQS 的钢铁丛林，CAS 的刀锋，以及 ReentrantLock 在真实高并发下的生死挣扎。

---

## 📋 本 Level 考核范围

| 知识板块 | 核心考点 | 对应黑马章节 |
|---|---|---|
| CAS 机制 | 底层原理、ABA 问题与解法 | 第六章 06.003 ~ 06.015 |
| AQS 框架 | CLH 队列结构、Node 状态流转、`state` 语义 | 第八章 08.037 ~ 08.039 |
| ReentrantLock 加锁流程 | 公平锁 vs 非公平锁差异（源码级）| 第八章 08.040 ~ 08.046 |
| LockSupport | park/unpark 的 permit 机制 | 第四章 04.059 ~ 04.060 |
| interrupt 与 lockInterruptibly | 可打断锁的 AQS 实现 | 第八章 08.045 |

---

## 🔥 大题一：CAS 与 ABA —— 库存系统的幽灵扣减

### Part A · 基础理论 —— "一个对账对不平的 Bug"

你们公司的积分系统用了如下代码做无锁积分扣减，在压测阶段发现了**对账对不平**的问题——账面显示扣了 100 分，但日志里只能找到 2 次成功扣减记录：

```java
public class PointsAccount {
    // 用户当前积分，初始 100
    private AtomicInteger points = new AtomicInteger(100);

    /**
     * 尝试扣除指定积分
     * @return 扣除成功返回 true
     */
    public boolean deduct(int amount) {
        while (true) {
            int current = points.get();
            if (current < amount) return false;
            int next = current - amount;
            if (points.compareAndSet(current, next)) {
                return true;
            }
        }
    }

    public int getPoints() {
        return points.get();
    }
}
```

日志记录如下（时间轴从上到下）：

```
[T1] 读取积分 = 100，准备扣减 100
[T2] 读取积分 = 100，扣减 50 成功，积分变为 50   ← T2 成功
[T3] 读取积分 = 50，充值 50 成功，积分变为 100   ← 充值线程
[T1] CAS(100 → 0) 成功！                          ← ???
[T1] 返回 true，扣减成功
```

**请回答以下问题：**

1. 上述 Bug 的名称是什么？请用一句话精准定义它，并解释为什么 `compareAndSet(100, 0)` 在 T1 视角下是"正确的"，而在业务视角下是"错误的"。

2. 这段代码底层依赖 `Unsafe.compareAndSwapInt()`。请从 CPU 指令层面解释：CAS 是如何保证"比较+交换"这两个操作的**原子性**的？（提示：x86 架构下是什么指令？多 CPU 环境下怎么保证？）

3. 给出两种修复方案，分析各自的适用场景和性能取舍：
   - 方案 A：基于版本号（Stamp）
   - 方案 B：改用互斥锁 `ReentrantLock`

### Part B · 场景实战 —— "大促限流器"

**业务需求**：双十一大促，你需要为某个核心接口实现一个**全局令牌桶限流器**，要求：
- 令牌桶容量 1000，每秒补充 500 个令牌
- 多线程并发消费令牌（每次请求消耗 1 个）
- **不能使用任何 `synchronized` 关键字**，必须基于 CAS 或 AQS 相关工具实现
- 需要考虑令牌耗尽时的处理策略（直接拒绝 or 等待）

请给出你的完整设计与核心代码（可以用 `AtomicInteger` / `Semaphore` / 自定义 AQS 均可，但必须说明选型理由）。

---

> ⏳ **请回答【大题一】全部内容后，我再出大题二（ReentrantLock 源码级 vs synchronized）。**
>
> 如果某个子问题不确定，直接说"这部分不太会，给个方向"，我告诉你去看黑马哪一节。

---

### 【极客挑战】（选做，可留给未来）

**极客挑战 1 · `LongAdder` 为什么比 `AtomicLong` 快？**
> `LongAdder` 的源码里有一个 `Cell[] cells` 数组和一个 `base` 字段。
> - 在低并发下，`LongAdder` 和 `AtomicLong` 行为有什么不同？
> - 在高并发下，`Cell[]` 数组的大小是怎么确定的？为什么用 `@Contended` 注解修饰 `Cell`？（提示：CPU 缓存行是 64 字节，画个图）
> - `LongAdder.sum()` 返回的值在并发场景下是精确值吗？为什么？适合什么场景？

**极客挑战 2 · CAS 的"自旋风暴"**
> 在 32 核机器上，1000 个线程同时对同一个 `AtomicInteger` 执行 CAS 操作：
> - 最坏情况下，每一轮只有 1 个线程 CAS 成功，其余 999 个自旋重试。这会对 CPU 总线产生什么影响？（提示：MESI 缓存一致性协议）
> - HotSpot 对这种"总线风暴"有什么缓解手段？（提示：pause 指令）
> - 如果把 `AtomicInteger` 改成 `LongAdder`，上述问题会消失吗？为什么？

**极客挑战 3 · 设计一个不用 CAS 的无锁栈**
> - 单生产者单消费者（SPSC）场景下，是否能设计一个既不用 `synchronized`，也不用 CAS，却线程安全的有界队列？需要哪些 JMM 层面的保证？
> - 这与 `Disruptor` 框架的 RingBuffer 有什么关联？

---

> 📌 **推荐学习路径（针对 Level 1 的弱点）**：
>
> | 薄弱点 | 推荐视频 | 推荐书籍 |
> |---|---|---|
> | CAS 原理与 ABA | 06.003 ~ 06.014 | 《Java并发编程的艺术》§5.2.5 |
> | AQS 框架骨架 | 08.037 ~ 08.039 | 《Java并发编程的艺术》§5.2 |
> | park/unpark permit 机制 | 04.059 ~ 04.060 | — |
> | ReentrantLock 加锁源码流程 | 08.040 ~ 08.046 | 《Java并发编程的艺术》§5.3 |
> | synchronized vs AQS 边界 | 对比 04.026 和 08.037 | 《Java并发编程的艺术》§2.2 vs §5.2 |
