# Level 1：并发基石与内存模型 (JMM) — 引导式学习问题册

> 新人你好，我是你的 JUC 导师。听闻你 MySQL 底层学得很扎实，但在真正的 Java 高并发业务里，内存和 CPU 可是会教你做人的。三个月后想拿大厂实习 Offer？那就接招吧。
>
> 本 Level 聚焦：**线程状态流转**、**`volatile` 的内存语义**、**`synchronized` 的锁升级**、**Java 对象头结构**。

---

## 【大题一】诡异的死循环 — `volatile` 与可见性

### Part A · 基础理论：推导 Bug 根因

下面这段代码在某台 4 核服务器上跑起来后，子线程**永远不会停下来**，`while` 循环变成了死循环。但在单核 CPU 的机器上却能正常退出。请分析：

```java
public class VisibilityBug {

    private static boolean running = true;  // 注意：没有 volatile

    public static void main(String[] args) throws InterruptedException {
        Thread worker = new Thread(() -> {
            int count = 0;
            while (running) {
                count++;
            }
            System.out.println("Worker 停止, count = " + count);
        });
        worker.start();

        Thread.sleep(1000);
        running = false;  // 主线程修改了 flag
        System.out.println("main 已将 running 设为 false");
    }
}
```

**请回答以下问题：**

1. 从 **JMM（Java Memory Model）** 的角度，解释为什么子线程看不到 `running = false`。请画出（或文字描述）**主内存 ↔ 工作内存** 的交互过程。
2. 为什么在**单核 CPU** 上可能正常退出？（提示：想想 CPU 缓存一致性协议在单核 vs 多核场景下的区别）
3. 如果在 `while` 循环体里加一行 `System.out.println(count)`，程序又能正常退出了，为什么？（提示：`println` 的底层实现里藏着什么同步机制？）
4. 如果给 `running` 加上 `volatile`，JMM 层面会插入哪些**内存屏障（Memory Barrier）**？它们分别阻止了什么类型的指令重排？

---

### Part B · 场景实战：秒杀系统的开关控制

**业务背景**：你负责一个秒杀系统，需要实现一个"秒杀活动开关"：

- 运营后台通过 HTTP 接口设置 `saleActive = true/false`
- 有 **200 个请求处理线程** 要实时读取这个开关
- 一旦开关关闭，所有线程必须在 **最短时间内停止接单**

**请给出你的方案，并回答：**

1. 使用 `volatile boolean` 能满足需求吗？请说明理由。
2. 如果开关的判断逻辑变为 `if (saleActive && stock > 0)` —— 涉及两个变量的**复合操作**，`volatile` 还够用吗？为什么？
3. 对比：如果用 `synchronized` 来保护这个开关读写，性能上会有什么代价？请从 **锁对象的对象头 Mark Word** 的状态变化角度来分析。

---

## 【大题二】锁升级的翻车现场 — `synchronized` 底层原理

### Part A · 基础理论：推导对象头状态

某同事写了一段"优化"代码，他声称：*"我只用了一个线程，`synchronized` 几乎没有开销，因为偏向锁嘛！"*

```java
public class BiasedLockDemo {

    private static final Object lock = new Object();

    public static void main(String[] args) throws InterruptedException {
        // Thread.sleep(5000);  // 同事注释掉了这行

        synchronized (lock) {
            System.out.println("第一次加锁");
            System.out.println(ClassLayout.parseInstance(lock).toPrintable());
        }

        synchronized (lock) {
            System.out.println("第二次加锁");
            System.out.println(ClassLayout.parseInstance(lock).toPrintable());
        }
    }
}
```

**请回答以下问题：**

1. 他注释掉了 `Thread.sleep(5000)`，结果 JOL（Java Object Layout）打印出来的锁标志位并不是偏向锁（`101`），而是轻量级锁（`00`）。为什么？
   > 提示：JVM 有一个叫 **偏向锁延迟（BiasedLockingStartupDelay）** 的参数，默认 4 秒。
2. 请完整画出一个对象从 **创建 → 偏向锁 → 轻量级锁 → 重量级锁** 的 Mark Word 位模式变化过程（64 位虚拟机）。每个阶段分别占用了 Mark Word 的哪些 bit？
3. **类比 MySQL**：MySQL InnoDB 中的锁也有"升级"概念（行锁 → 间隙锁 → 表锁意向锁）。请对比 Java `synchronized` 锁升级和 MySQL 的锁膨胀机制，说说它们的**设计动机**有何相似与不同？
4. `synchronized` 锁升级是**不可逆**的吗？在 JDK 15 之后发生了什么变化？

---

### Part B · 场景实战：用户积分扣减

**业务背景**：电商系统中，用户下单后需要扣减积分。代码如下：

```java
public class PointService {

    private int points = 1000;

    public void deduct(int amount) {
        if (points >= amount) {
            // 模拟一些业务逻辑
            try { Thread.sleep(10); } catch (InterruptedException e) {}
            points -= amount;
            System.out.println(Thread.currentThread().getName() 
                + " 扣减 " + amount + " 后剩余: " + points);
        }
    }
}
```

在另一处代码中，开了 **10 个线程同时调用 `deduct(200)`**。

**请回答：**

1. 这段代码有哪些线程安全问题？请逐一列出。
2. 请用 `synchronized` 给出一个**最小粒度**的修复方案（注意：不是无脑把整个方法加 `synchronized`）。
3. 你选择锁 `this` 还是一个独立的 `private final Object lock`？说说各自的优缺点，从**锁对象逃逸**的角度分析。
4. 主管说："扣减积分这个操作，QPS 峰值才 50，用 `synchronized` 足够了。但如果未来 QPS 暴增到 5000，你有什么升级方案？" —— 请给出思路（可以提前预告 Level 2 的 CAS / LongAdder 知识，但不需要写出完整代码）。

---

## 【大题三】线程的生死轮回 — 状态流转与打断机制

### Part A · 基础理论：诊断线上故障

凌晨 3 点，监控告警：某服务的线程池线程数飙到上限，大量请求超时。你拉了一份 **Thread Dump**，发现有 300+ 线程处于 `WAITING` 状态，堆栈如下：

```
"pool-1-thread-233" #233 prio=5 os_prio=0 tid=0x00007f... nid=0x... waiting on condition [0x...]
   java.lang.Thread.State: WAITING (parking)
        at jdk.internal.misc.Unsafe.park(Native Method)
        - parking to wait for  <0x00000000d6a08e70> (a java.util.concurrent.locks.ReentrantLock$NonfairSync)
        at java.util.concurrent.locks.LockSupport.park(LockSupport.java:194)
        at java.util.concurrent.locks.AbstractQueuedSynchronizer.parkAndCheckInterrupt(...)
        at java.util.concurrent.locks.AbstractQueuedSynchronizer.acquireQueued(...)
        ...
```

**请回答以下问题：**

1. 请画出 Java 线程的**完整六态流转图**（NEW → RUNNABLE → BLOCKED → WAITING → TIMED_WAITING → TERMINATED），标注每个状态转换触发的方法调用。
2. Thread Dump 中线程状态显示 `WAITING (parking)` 而不是 `BLOCKED`。请解释 `WAITING` 和 `BLOCKED` 的本质区别，以及为什么 `ReentrantLock` 排队的线程状态是 `WAITING` 而非 `BLOCKED`。
3. `LockSupport.park()` 和 `Object.wait()` 都能让线程挂起，但底层机制完全不同。请从**操作系统**层面对比这两者的实现。
4. **类比 MySQL**：MySQL 中当一个事务在等行锁时，你在 `SHOW ENGINE INNODB STATUS` 里看到的是什么状态？和 Java 的 `WAITING` 有何概念上的相似性？

---

### Part B · 场景实战：优雅关闭后台任务

**业务背景**：你的服务有一个后台数据同步线程，每隔 5 秒从上游拉一次数据：

```java
public class DataSyncer implements Runnable {

    private volatile boolean stopped = false;

    @Override
    public void run() {
        while (!stopped) {
            // 1. 拉取数据
            fetchFromUpstream();
            // 2. 休眠 5 秒
            try {
                Thread.sleep(5000);
            } catch (InterruptedException e) {
                // 同事写了个空 catch
            }
        }
        System.out.println("同步线程已退出");
    }

    public void shutdown() {
        stopped = true;
    }
}
```

现在收到需求：服务下线时，这个同步线程必须在 **3 秒内优雅退出**（不能等到 sleep 5 秒结束才退出）。

**请回答：**

1. 当前代码的 `shutdown()` 方法有什么问题？为什么最慢需要等 5 秒才能退出？
2. 请使用 `Thread.interrupt()` 机制重写 `shutdown()` 和 `run()` 方法，实现"立即中断 sleep 并退出"。注意正确处理 `InterruptedException` 和中断标志位。
3. 如果 `fetchFromUpstream()` 内部是一个**阻塞 I/O 操作**（如 Socket 读取），`interrupt()` 还能打断它吗？如果不能，你有什么替代方案？
4. Spring Boot 项目中，`@PreDestroy` 注解的方法被调用时，你如何保证这个同步线程已经完全停止并释放了所有资源？请给出代码骨架。

---

## 【大题四】`volatile` 的边界 — 它不是万能的

### Part A · 基础理论：拆穿 volatile 的假象

看下面这段"看似线程安全"的单例模式：

```java
public class Singleton {

    private static Singleton INSTANCE;

    private int value;

    private Singleton() {
        this.value = 42;
    }

    public static Singleton getInstance() {
        if (INSTANCE == null) {                    // ① 第一次检查
            synchronized (Singleton.class) {
                if (INSTANCE == null) {            // ② 第二次检查
                    INSTANCE = new Singleton();    // ③ 创建实例
                }
            }
        }
        return INSTANCE;
    }
}
```

**请回答以下问题：**

1. 这段 DCL（Double-Checked Locking）为什么是**有 Bug** 的？请从字节码层面（`new → dup → invokespecial → astore`）分析第 ③ 步可能发生的**指令重排序**，说明另一个线程在第 ① 步可能拿到什么样的对象。
2. 加上 `volatile` 修饰 `INSTANCE` 之后，JMM 在第 ③ 步的赋值操作前后分别插入了什么内存屏障？这些屏障如何阻止了重排序？
3. 有人说："JDK 9 之后用 `VarHandle` 可以更精细地控制内存顺序。" 请简单说明 `VarHandle` 的 `setRelease()` / `getAcquire()` 和 `volatile` 的语义差异。
4. **灵魂拷问**：如果面试官问你"volatile 和 synchronized 的区别"，你会怎么回答？请从**原子性、可见性、有序性**三个维度给出一个30秒的精炼回答。

---

### Part B · 场景实战：配置中心热更新

**业务背景**：你需要实现一个本地配置缓存，每隔 30 秒从配置中心拉取最新配置，其他业务线程随时读取：

```java
public class ConfigCache {

    private Map<String, String> configMap = new HashMap<>();

    // 定时任务线程调用
    public void refresh() {
        Map<String, String> newConfig = fetchFromConfigCenter();
        configMap = newConfig;  // 直接引用替换
    }

    // N 个业务线程调用
    public String getConfig(String key) {
        return configMap.get(key);
    }
}
```

**请回答：**

1. 当前代码在多线程环境下有哪些问题？（提示：不仅仅是可见性问题）
2. 有人建议把 `configMap` 改成 `volatile Map<String, String>`，这足够吗？为什么？
3. 另一位同事建议用 `ConcurrentHashMap` 替换 `HashMap`，你同意吗？请从**读写模式**的角度分析这里用 `ConcurrentHashMap` 是不是"杀鸡用牛刀"。
4. 请给出你认为**最优雅**的方案，并解释为什么。（提示：思考不可变对象 + 引用替换的模式）

---

## 答题须知

- 每道大题独立作答，先完成 Part A 再完成 Part B
- 回答时鼓励用**代码 + 文字**结合的方式
- 如果涉及锁的状态变化，建议画**状态图**
- 可以类比 MySQL 的 MVCC / 锁机制来辅助理解，导师会给予专业点评
- 回答"我不会"没有关系，导师会给你提示，但你需要自己最终推导出答案

> 💡 **完成本 Level 后**，导师将对你的回答进行逐题 Code Review 式点评，并决定你是否具备进入 **Level 2：AQS & ReentrantLock** 的资格。
