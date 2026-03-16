# Level 1：并发基石与内存模型 (JMM) —— 引导式学习问题册

> 新人你好，我是你的 JUC 导师。听闻你 MySQL 底层学得很扎实，但在真正的 Java 高并发业务里，内存和 CPU 可是会教你做人的。三个月后想拿大厂实习 Offer？那就接招吧。

---

## 📋 本 Level 考核范围

| 知识板块 | 核心考点 | 对应黑马章节 |
|---|---|---|
| 线程状态流转 | 五种/六种状态、状态转换触发条件 | 第三章 03.034 ~ 03.036 |
| `volatile` 内存语义 | 可见性、指令重排禁止、happens-before | 第五章 05.002 ~ 05.019 |
| `synchronized` 底层原理 | Monitor、锁升级（无锁→偏向→轻量→重量） | 第四章 04.026 ~ 04.038 |
| Java 对象头 (Mark Word) | 对象头结构、锁标志位含义 | 第四章 04.026 |
| wait/notify 机制 | 工作原理、正确使用姿势 | 第四章 04.039 ~ 04.047 |

---

## 🔥 大题一：线程状态流转 × 线上诡异 Bug

### Part A · 基础理论 —— "为什么线程卡死了？"

你是大厂夜间值班 SRE。凌晨 3 点你被告警叫醒——生产环境某服务接口超时飙升，`jstack` 导出的线程快照中发现了以下关键信息：

```
"order-process-thread-3" #42 daemon prio=5 os_prio=0 tid=0x00007f... nid=0x1a03 waiting on condition [0x00007f...]
   java.lang.Thread.State: WAITING (parking)
        at sun.misc.Unsafe.park(Native Method)
        - parking to wait for  <0x00000000d6f08e10> (a java.util.concurrent.locks.ReentrantLock$NonfairSync)
        at java.util.concurrent.locks.LockSupport.park(LockSupport.java:175)
        at java.util.concurrent.locks.AbstractQueuedSynchronizer.parkAndCheckInterrupt(AbstractQueuedSynchronizer.java:836)
        ...

"order-process-thread-1" #40 daemon prio=5 os_prio=0 tid=0x00007f... nid=0x19ff waiting for monitor entry [0x00007f...]
   java.lang.Thread.State: BLOCKED (on object monitor)
        at com.example.OrderService.processOrder(OrderService.java:87)
        - waiting to lock <0x00000000d6e01f28> (a com.example.OrderService)
        at ...

"order-process-thread-0" #39 daemon prio=5 os_prio=0 tid=0x00007f... nid=0x19fe runnable [0x00007f...]
   java.lang.Thread.State: RUNNABLE
        at com.example.OrderService.calculateDiscount(OrderService.java:142)
        at com.example.OrderService.processOrder(OrderService.java:95)
        - locked <0x00000000d6e01f28> (a com.example.OrderService)
        ...
```

**请回答以下问题：**

1. 上述三个线程各自处于 Java 线程六种状态中的哪一种？它们各自在等什么？
2. `WAITING (parking)` 和 `BLOCKED (on object monitor)` 的底层区别是什么？从 JVM 对象头、Monitor、AQS 三个角度分别说明。
3. 如果这时候你在生产上对 thread-0 执行了 `thread.interrupt()`，三个线程各自会发生什么？请分开讨论持有 `synchronized` 内置锁和持有 `ReentrantLock` 两种场景。

### Part B · 场景实战 —— "线程池里的线程泄漏"

**业务需求**：你负责一个订单服务，使用了一个固定大小为 8 的线程池处理订单。有人反馈"最近的订单处理越来越慢，好像线程池里的线程都'卡住'了"。

你用 `jstack` 发现其中 6 个线程都处于 `WAITING` 状态，堆栈指向 `Object.wait()`。经排查，发现业务代码中存在一段经典错误：

```java
public class OrderProcessor {
    private final Object lock = new Object();
    private volatile boolean dataReady = false;

    public void waitForData() {
        synchronized (lock) {
            if (!dataReady) {         // ← 注意这里
                lock.wait();           // 有异常处理省略
            }
            // 处理数据...
        }
    }

    public void notifyDataReady() {
        synchronized (lock) {
            dataReady = true;
            lock.notify();             // ← 注意这里
        }
    }
}
```

**请回答：**

1. 指出这段代码中至少存在的 **两个致命并发 Bug**，并解释在什么时序下会触发。
2. 给出你修复后的完整代码，并解释每一处修改的理由。
3. 类比 MySQL：在 MySQL InnoDB 中，也有"等待-通知"机制吗？如果有，其底层原理与 Java 的 `wait/notify` 有什么可以类比的地方？

---

> ⏳ **请回答以上【大题一】的全部内容后，我再出下一题。**
>
> 如果某个子问题你不确定，可以直接说"这部分我不太会，请给个提示方向"，我会告诉你应该去看黑马哪一节或者《Java并发编程的艺术》的哪个章节。

---

### 【极客挑战】（选做，可留给未来）

以下题目涉及更深层的源码和边缘 Case，你可以尝试作答，也可以直接说"留给未来"：

**极客挑战 1 · 锁升级的"不归路"**

> 在 JDK 15 及以后，偏向锁被彻底废弃了（JEP 374）。请问：
> - 偏向锁从 JDK 6 引入到 JDK 15 废弃，中间经历了什么样的"信仰崩塌"？其废弃的根本原因是什么？
> - 在偏向锁被废弃后，HotSpot 的锁升级路径变成了什么样？这对高并发场景是利是弊？

**极客挑战 2 · C++ 源码级 —— Monitor 的真实面貌**

> HotSpot 原生的 `ObjectMonitor` 结构体（C++ 层面）中，有 `_EntryList`、`_WaitSet`、`_cxq` 三个队列。
> - 为什么需要同时存在 `_EntryList` 和 `_cxq` 两个入口队列？为什么不能合并成一个？
> - 当一个线程从 `_WaitSet` 被唤醒后，它是被放到 `_EntryList` 还是 `_cxq`？不同的 JVM 策略（`Knob_MoveNotifyee`）会有什么不同行为？

**极客挑战 3 · 手撕 happens-before**

> 请给出以下代码的所有可能输出，用 JMM 的 happens-before 规则严格证明你的结论：
> ```java
> // 初始状态: x = 0, y = 0, a = 0, b = 0
> // 线程 1:
> a = 1;
> x = b;
>
> // 线程 2:
> b = 1;
> y = a;
> ```
> **能否出现 `x == 0 && y == 0` 的结果？如果能，请画出指令重排后的执行时序图。**

---

> 📌 **提示**：本 Level 对应黑马课程 **第三章（线程基础）、第四章（共享模型之管程）、第五章（共享模型之 JMM）**。如果你在某个知识点上感到吃力，以下是精准的学习路径：
>
> | 薄弱点 | 推荐视频 | 推荐书籍 |
> |---|---|---|
> | 线程六种状态 | 03.034 ~ 03.036 | 《Java并发编程的艺术》第4章 §4.1 |
> | synchronized 锁升级 | 04.029 ~ 04.038 | 《Java并发编程的艺术》第2章 §2.2 |
> | wait/notify 原理 | 04.039 ~ 04.047 | 《Java并发编程的艺术》第4章 §4.3 |
> | volatile 可见性与重排 | 05.002 ~ 05.018 | 《Java并发编程的艺术》第3章 §3.4 |
> | happens-before | 05.019 | 《Java并发编程的艺术》第3章 §3.7 |
