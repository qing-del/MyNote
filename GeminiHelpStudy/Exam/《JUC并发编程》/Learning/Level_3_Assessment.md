# Level 3：JMM 内存模型与无锁并发 —— 引导式学习问题册

> 上一关你能写出真实压测代码、多次 Debug 迭代，工程直觉是对的。但 CAS 的 CPU 底层你只答了一半，`AtomicStampedReference` 你借道而行却不知道走的路是错的。
>
> Level 3 的战场是 JMM 的内存屏障丛林，volatile 和 DCL 的生死线，以及 `LongAdder` 的 Cell 分段艺术。你在压测代码里用了 `LongAdder` 来统计 QPS，却没能解释"为什么"——这一关会让你把这个债还清。

---

## 📋 本 Level 考核范围

| 知识板块 | 核心考点 | 对应黑马章节 |
|---|---|---|
| `volatile` 内存语义 | 可见性实现原理、写屏障 / 读屏障、指令重排禁止 | 第五章 05.002 ~ 05.014 |
| happens-before 规则 | 传递性推导、volatile 变量规则 | 第五章 05.019 |
| DCL 单例 | 为什么双重检查需要 `volatile`，半初始化对象的危险 | 第五章 05.015 ~ 05.018 |
| `AtomicStampedReference` | 正确解决 ABA 问题的标准工具 | 第六章 06.013 ~ 06.014 |
| `LongAdder` 内部原理 | `base` + `Cell[]` 分治、缓存行伪共享、`@Contended` | 第六章 06.019 ~ 06.026 |

---

## 🔥 大题一：volatile 与 DCL —— 一个崩溃在凌晨的配置中心

### Part A · 基础理论 —— "神奇的 null 指针"

你所在的团队维护一套配置中心客户端，使用 DCL（双重检查锁）单例加载远程配置。某天线上突然出现大量 `NullPointerException`，堆栈均指向 `getInstance().getConfig(key)` 的 `.getConfig()` 调用。诡异的是：`getInstance()` 的返回值**不是 null**——单例对象确实存在，但对象内部的 `config` 字段是 null。

代码如下：

```java
public class ConfigCenter {
    private static ConfigCenter instance;   // ← 注意这里

    private Map<String, String> config;

    private ConfigCenter() {
        // 从远程加载配置，耗时操作
        this.config = loadFromRemote();
    }

    public static ConfigCenter getInstance() {
        if (instance == null) {                     // 第一次检查
            synchronized (ConfigCenter.class) {
                if (instance == null) {             // 第二次检查
                    instance = new ConfigCenter();  // ← 注意这里
                }
            }
        }
        return instance;
    }

    public String getConfig(String key) {
        return config.get(key);  // ← NPE 发生在这里
    }
}
```

**请回答以下问题：**

1. 这段代码为什么会出现 `config` 字段为 null 的情况？请从 **JVM 对象初始化的字节码指令顺序** 和 **指令重排** 两个角度，精确解释 NPE 的触发路径（画出时序图更佳）。

2. `instance = new ConfigCenter()` 这一行，在 JVM 层面可以拆解为哪几个步骤？哪两个步骤的重排才是导致 NPE 的根本原因？

3. 给出正确的修复方案，并解释 `volatile` 在这里具体禁止了哪个方向的指令重排（是"写后读"还是"写后写"？`StoreStore` 屏障还是 `StoreLoad` 屏障？）。

### Part B · 场景实战 —— "计数器失真的报表系统"

你们的报表服务负责统计每分钟的 API 调用次数，用于生成实时监控图表。技术选型时有人提议以下三种方案，请你做技术评审：

**方案一**：
```java
private long counter = 0;

public void increment() {
    counter++;
}
```

**方案二**：
```java
private AtomicLong counter = new AtomicLong(0);

public void increment() {
    counter.incrementAndGet();
}
```

**方案三**：
```java
private LongAdder counter = new LongAdder();

public void increment() {
    counter.increment();
}
```

**请回答：**

1. 方案一的 `counter++` 在多线程下为什么不安全？从 JMM 的角度（工作内存 / 主内存模型），解释计数丢失的完整过程。

2. 方案二在高并发下的性能瓶颈是什么？请从 **CPU 缓存一致性协议（MESI）** 的角度解释，为什么大量线程同时对同一个 `AtomicLong` 做 CAS 会产生"总线风暴"（Bus Contention）。

3. 方案三的 `LongAdder` 是如何解决方案二的瓶颈的？请解释：
   - `base` 字段和 `Cell[]` 数组各自的职责是什么？
   - 低并发和高并发下，`LongAdder` 的行为有什么不同？
   - 为什么 `Cell` 类要用 `@Contended` 注解修饰？如果不加会怎样？（提示：画出 64 字节缓存行的示意图）
   - `LongAdder.sum()` 在并发场景下返回的是精确值吗？这个不精确有没有关系，适合什么场景？

4. 结合你在 Level 2 压测代码里用 `LongAdder` 统计 QPS 的实践：如果当时换成 `AtomicLong`，在 4000 并发下的统计结果会更慢还是更快？请说明理由。

---

> ⏳ **请回答【大题一】全部内容后，我再出大题二（`AtomicStampedReference` + CAS 进阶）。**
>
> 如果某个子问题不确定，直接说"这部分不太会，给个方向"，我会告诉你去看黑马哪一节。

---

### 【极客挑战】（选做，可留给未来）

**极客挑战 1 · `volatile` 的"半个锁"代价**

> JMM 规范要求，对 `volatile` 变量的写操作之后，必须插入 `StoreLoad` 屏障（这是四种屏障中代价最高的一种）。
> - 在 x86 架构下，`StoreLoad` 屏障是通过什么指令实现的？（提示：`mfence` 或 `lock addl`）
> - 为什么 x86 的 TSO（Total Store Order）内存模型下，`StoreStore` 和 `LoadLoad` 屏障是"免费"的，但 `StoreLoad` 屏障不是？
> - 这对 volatile 读 vs volatile 写的性能有什么影响？（谁更快？）

**极客挑战 2 · `LongAdder` 的 `Cell[]` 扩容时机**

> `LongAdder` 内部的 `Cell[]` 并非一开始就分配好的，它会在高并发竞争下动态扩容。
> - 扩容的触发条件是什么？最大 size 由什么决定？（提示：和 CPU 核心数有关）
> - 在 `longAccumulate` 方法里，`COLLIDE` 标志位和 `wasUncontended` 标志位各自代表什么状态？这两个标志是如何合作来决定是"自旋重试"还是"扩容"的？

**极客挑战 3 · 不用 `volatile` 的 DCL 替代品**

> 除了给 `instance` 加 `volatile`，还有哪些方式可以实现线程安全的懒加载单例？
> - **静态内部类（Initialization-on-demand Holder）** 模式是如何利用 JVM 的类加载机制来保证线程安全的？它依赖的 happens-before 规则是哪一条？
> - 对比 `volatile DCL` 和 `Holder` 模式，哪个更好？为什么大多数现代代码更推荐 `Holder` 模式？

---

> 📌 **精准学习路径（针对 Level 2 的遗留弱点）**：
>
> | 薄弱点 | 推荐视频 | 推荐书籍 |
> |---|---|---|
> | volatile 写/读屏障 | 05.013 ~ 05.014 | 《Java并发编程的艺术》第3章 §3.4 |
> | happens-before 推导 | 05.019 | 《Java并发编程的艺术》第3章 §3.7 |
> | DCL 半初始化对象 | 05.015 ~ 05.018 | 《Java并发编程的艺术》第3章 §3.8 |
> | AtomicStampedReference | 06.013 ~ 06.014 | — |
> | LongAdder Cell 原理 | 06.019 ~ 06.026 | 《Java并发编程的艺术》§5.2 |
