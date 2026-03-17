---
title: 共享模型之无锁_JMM
tags:
 - JUC
 - JMM
 - volatile
 - 可见性
 - 有序性
 - happens-before
 - 设计模式
create_time: 2026-03-18
---

## 目录
- [[#可见性问题]]
	- [[#问题引入]]
	- [[#可见性解决 — volatile]]
	- [[#可见性 vs 原子性]]
- [[#volatile 的应用]]
	- [[#设计模式 — 两阶段终止（volatile 版）]]
	- [[#设计模式 — 犹豫模式（Balking）]]
- [[#有序性问题]]
	- [[#指令重排序]]
	- [[#指令重排的底层原理 — 指令级并行]]
	- [[#指令重排引发的问题]]
	- [[#禁止重排 — volatile 写屏障与读屏障]]
- [[#volatile 原理深入]]
	- [[#内存屏障（Memory Barrier）]]
	- [[#volatile 如何保证可见性]]
	- [[#volatile 如何保证有序性]]
	- [[#DCL 单例与 volatile]]
- [[#happens-before 规则]]
- [[#习题与实战]]
	- [[#线程安全单例模式]]

---

## 可见性问题

### 问题引入

```java
static boolean run = true;  // 共享变量

Thread t = new Thread(() -> {
    while (run) {
        // 循环体内没有对 run 的写操作
    }
    System.out.println("线程 t 结束");
});
t.start();

Thread.sleep(1000);
run = false;  // 主线程修改 run，期望 t 退出循环
System.out.println("主线程设置 run = false");
// 实际现象：t 线程可能永远不会退出！
```

> [!failure] 为什么 t 线程看不到 run 的修改？
> 这涉及 **JMM（Java 内存模型）** 和 CPU 缓存架构：
> 1. 主线程和 t 线程各自的工作内存（**CPU 高速缓存 / 寄存器**）中都缓存了 `run` 的副本
> 2. 主线程修改的是主内存中的 `run`，但 t 线程一直读的是自己工作内存中的缓存副本
> 3. JIT 编译器可能将 `while(run)` 优化为 `while(true)`（因为循环体内没有对 run 的写入，编译器认为 run 不会变）

> [!info] JVM/OS 知识 —— CPU 缓存体系
> 现代 CPU 为了弥补内存速度与 CPU 速度的巨大差距，采用了多级缓存：
> ```
> CPU Core 0           CPU Core 1
> ┌──────────┐         ┌──────────┐
> │ Register │         │ Register │
> │  L1 Cache│         │  L1 Cache│
> │  L2 Cache│         │  L2 Cache│
> └────┬─────┘         └────┬─────┘
>      │                    │
>      └────── L3 Cache ────┘
>              │
>         Main Memory（主内存）
> ```
> - 每个 CPU 核心有自己的 L1/L2 缓存
> - 线程读写共享变量时，实际操作的是 CPU 缓存中的副本
> - **缓存一致性（Cache Coherence）** 协议（如 MESI）会将修改同步回主内存，但时机不确定
> - 这就是可见性问题的硬件根源
>
> **JMM 是 Java 对底层硬件差异的抽象**：它定义了线程与主内存之间的交互规则，屏蔽了不同 CPU 架构的差异

---

### 可见性解决 — volatile

```java
static volatile boolean run = true;  // 加 volatile

Thread t = new Thread(() -> {
    while (run) { }  // 每次读取都从主内存获取最新值
    System.out.println("线程 t 结束");
});
t.start();

Thread.sleep(1000);
run = false;  // 主线程写入后立即刷新到主内存
// t 线程下次读取 run 时，会从主内存读取到 false → 退出循环
```

`volatile` 关键字的两大作用：
1. **保证可见性**：对 volatile 变量的写操作会**立即刷新到主内存**，对其读操作会**从主内存重新读取**
2. **保证有序性**：禁止编译器和 CPU 对 volatile 变量相关的操作进行重排序

> [!warning] volatile 不保证原子性！
> `volatile` 只能保证读/写单个变量的可见性，不能保证"读-改-写"的原子性
> ```java
> static volatile int counter = 0;
> // counter++ 依然不是线程安全的！
> // 因为 counter++ = read + add + write，三步之间可以被打断
> ```
> 要保证原子性，需要使用 `synchronized` 或 `Atomic` 原子类

---

### 可见性 vs 原子性

| 特性 | 可见性（Visibility） | 原子性（Atomicity） |
|------|---------------------|---------------------|
| 含义 | 一个线程的修改能被其他线程看到 | 一组操作要么全部执行完，要么全都不执行 |
| 保证手段 | `volatile`、`synchronized`、`Lock` | `synchronized`、`Lock`、`Atomic` 类、CAS |
| 典型问题 | 主线程改 flag，子线程看不到 | `counter++` 丢失更新 |
| `volatile` | ✅ 能保证 | ❌ 不能保证 |
| `synchronized` | ✅ 能保证（释放锁时刷新缓存） | ✅ 能保证 |

> [!tip] 关联思考
> `synchronized` 的**可见性保证**来源于 JMM 规范：
> - 线程**获取锁时**：清空工作内存，从主内存重新加载共享变量
> - 线程**释放锁时**：将工作内存中的修改刷新到主内存
> 所以 `synchronized` 既保证了原子性，也保证了可见性

---

## volatile 的应用

### 设计模式 — 两阶段终止（volatile 版）

在第三章中我们用 `interrupt` 实现了两阶段终止，这里使用 `volatile` 标志位来实现：

```java
class MonitorService {
    private Thread monitor;
    private volatile boolean stop = false;  // 必须 volatile，保证可见性

    public void start() {
        monitor = new Thread(() -> {
            while (true) {
                if (stop) {
                    System.out.println("收尾工作...");
                    break;
                }
                try {
                    Thread.sleep(1000);  // 模拟监控
                    System.out.println("监控中...");
                } catch (InterruptedException e) {
                    // 如果 sleep 被打断也结束
                }
            }
        }, "monitor");
        monitor.start();
    }

    public void stop() {
        stop = true;
        monitor.interrupt();  // 如果正在 sleep，打断它使其及时退出
    }
}
```

> [!question] interrupt 版 vs volatile 版对比
> | 方案 | 优点 | 缺点 |
> |------|------|------|
> | interrupt | 能打断阻塞状态（sleep/wait/join） | 需处理 InterruptedException，需重设标记 |
> | volatile | 代码更直观，逻辑更简单 | 无法打断阻塞状态，需配合 interrupt 使用 |
>
> **最佳实践**：两者结合，`volatile` 标志位 + `interrupt()` 兜底

---

### 设计模式 — 犹豫模式（Balking）

**核心思想**：如果某个动作已经被执行或正在被执行，就不再重复执行——"犹豫一下"然后放弃。

> 类比：你准备做饭，走到厨房发现室友已经在做了，你"犹豫"了一下然后回房间了
> CS 术语：**Balking Pattern** — 通过前置条件检查避免重复执行。本质上是一种**幂等性保护**机制

```java
class MonitorService {
    private volatile boolean started = false;

    public void start() {
        // Balking：如果已经启动了，就不再重复启动
        if (started) {
            return;  // "犹豫"后放弃
        }
        started = true;
        // 执行启动逻辑...
    }
}
```

> [!warning] 多线程下的 Balking 需要同步
> 上面的 `if (started) return` + `started = true` 是一个"先检查后执行"（Check-Then-Act）操作
> 在多线程场景下不是原子的，需要加锁：
> ```java
> public synchronized void start() {
>     if (started) return;
>     started = true;
>     // ...
> }
> ```

#### 应用场景
- **单例模式的懒加载**：只有第一次调用时才创建实例
- **配置文件只加载一次**：多处调用 `loadConfig()` 但只执行一次
- **任务调度防重复提交**：一个任务只允许提交一次执行

---

## 有序性问题

### 指令重排序

**定义**：编译器和 CPU 为了优化性能，可能改变代码的执行顺序（只保证**单线程**下结果一致性）。

```java
// 源代码顺序
int a = 1;    // 语句1
int b = 2;    // 语句2
int c = a + b;// 语句3

// 实际执行可能为：语句2 → 语句1 → 语句3（语句3 依赖 a 和 b，不能提前）
// 单线程下结果相同，但多线程下可能出问题
```

> [!info] 发生重排序的三个层面
> 1. **编译器重排**（JIT Compiler）：编译器在不改变**单线程语义**的前提下重排指令
> 2. **CPU 指令重排**（Out-of-Order Execution / 乱序执行）：CPU 利用流水线技术，将不相关的指令并行执行
> 3. **内存系统重排**（Store Buffer / Invalidate Queue）：CPU 写缓冲区和读失效队列的存在导致读写顺序对其他核心不可见

### 指令重排的底层原理 — 指令级并行

> [!info] OS/CPU 知识 —— 指令流水线（Instruction Pipeline）
> 现代 CPU 将一条指令拆分为多个阶段（如 **取指 → 译码 → 执行 → 访存 → 写回**），不同指令的不同阶段可以**重叠执行**：
>
> ```
> 时钟周期:    1    2    3    4    5    6    7
> 指令A:      取指  译码  执行  访存  写回
> 指令B:           取指  译码  执行  访存  写回
> 指令C:                取指  译码  执行  访存  写回
> ```
>
> 当两条指令之间**没有数据依赖**时，CPU 可以自由调整它们的执行顺序来最大化流水线利用率。这就是**指令级并行（Instruction-Level Parallelism，ILP）**
>
> **问题所在**：CPU 只检查当前核心内部的数据依赖（**单线程语义**），不关心不同核心上线程之间的依赖关系。所以指令重排在多线程场景下可能产生语义错误

### 指令重排引发的问题

经典案例：双重检查锁（DCL）问题（详见下文 [[#DCL 单例与 volatile]]）

另一个简单示例：
```java
// 线程1
int num = 2;          // 语句A
boolean ready = true; // 语句B（可能与A重排）

// 线程2
if (ready) {           // 可能先看到 ready=true
    result = num * 2;  // 但 num 可能还是 0！（因为语句A被重排到了B后面）
}
```

> [!failure] 核心问题
> 指令重排在单线程中总是安全的（JVM 保证了 **as-if-serial** 语义）
> 但在多线程中，**线程1的重排顺序对线程2可见**，可能导致线程2看到不一致的中间状态

### 禁止重排 — volatile 写屏障与读屏障

对 `volatile` 变量的读/写会插入**内存屏障（Memory Barrier / Memory Fence）**：

```java
// 线程1
int num = 2;                    // 普通写
volatile boolean ready = true;  // volatile 写（插入写屏障）
// 写屏障保证：num=2 一定在 ready=true 之前对其他线程可见

// 线程2
if (ready) {                    // volatile 读（插入读屏障）
    // 读屏障保证：读到 ready=true 后，一定能读到 num=2
    result = num * 2;           // result 一定等于 4
}
```

---

## volatile 原理深入

### 内存屏障（Memory Barrier）

内存屏障是 CPU 指令，用于控制特定条件下的重排序和内存可见性。JMM 定义了四种屏障：

| 屏障类型 | 指令示例 | 效果 |
|---------|---------|------|
| LoadLoad | — | 确保前面的**读**操作完成后才执行后面的**读**操作 |
| StoreStore | — | 确保前面的**写**操作刷新到主内存后才执行后面的**写**操作 |
| LoadStore | — | 确保前面的**读**操作完成后才执行后面的**写**操作 |
| StoreLoad | `mfence` / `lock addl` | 确保前面的**写**刷新到主内存后才执行后面的**读**（最重的屏障） |

> [!info] CPU 层面的屏障实现
> 在 x86 架构下，CPU 本身已经保证了较强的内存序（TSO — Total Store Ordering）
> 只有 StoreLoad 屏障需要显式指令（如 `mfence` 或 `lock` 前缀的指令）
> 在 ARM/RISC-V 等弱内存序的 CPU 上，几乎所有屏障类型都需要显式指令
> 这也是为什么 Java 并发程序在不同 CPU 架构上可能表现不同 —— JMM 就是为了屏蔽这些差异而存在的

### volatile 如何保证可见性

```
volatile 写:
┌──────────────────────────────────────────────────────┐
│ 普通写操作（对所有共享变量的修改）                        │
│  ↓                                                    │
│ 【StoreStore 屏障】 — 确保前面的写都刷新到主内存          │
│  ↓                                                    │
│ volatile 写操作  → 值立即刷新到主内存                     │
│  ↓                                                    │
│ 【StoreLoad 屏障】 — 防止 volatile 写与后续操作重排     │
└──────────────────────────────────────────────────────┘

volatile 读:
┌──────────────────────────────────────────────────────┐
│ volatile 读操作  → 从主内存读取最新值                     │
│  ↓                                                    │
│ 【LoadLoad 屏障】 — 确保 volatile 读完成后才做后续读     │
│ 【LoadStore 屏障】 — 确保 volatile 读完成后才做后续写    │
│  ↓                                                    │
│ 普通读写操作                                            │
└──────────────────────────────────────────────────────┘
```

> [!tip] 简记
> - **volatile 写**：在写操作**之前**插入 StoreStore 屏障，**之后**插入 StoreLoad 屏障
> - **volatile 读**：在读操作**之后**插入 LoadLoad + LoadStore 屏障
> - 效果：volatile 写"推送"最新值到主内存；volatile 读"拉取"主内存最新值
>
> 类比：volatile 写像是**发公告**（写完后贴在公告板上，所有人都能看到）；volatile 读像是**查公告板**（每次都去看最新内容，不看自己的旧笔记）
> CS 术语：volatile 写触发 **store buffer flush**，volatile 读使 **CPU cache line invalidate**

### volatile 如何保证有序性

- volatile 写之前的所有操作**不会被重排到** volatile 写之后
- volatile 读之后的所有操作**不会被重排到** volatile 读之前

**总结一下**：
```
volatile 写是一道"屏风" — 前面的操作不能翻越它往后跑
volatile 读是一道"屏风" — 后面的操作不能翻越它往前跑
```

---

### DCL 单例与 volatile

**DCL（Double-Checked Locking）** 是实现懒加载单例的经典方案：

#### 有问题的 DCL（不加 volatile）
```java
public class Singleton {
    private static Singleton INSTANCE;  // ⚠️ 没有 volatile

    public static Singleton getInstance() {
        if (INSTANCE == null) {                   // 第一次检查（无锁）
            synchronized (Singleton.class) {
                if (INSTANCE == null) {            // 第二次检查（有锁）
                    INSTANCE = new Singleton();    // ⚠️ 此处有问题！
                }
            }
        }
        return INSTANCE;
    }
}
```

> [!failure] 问题分析 —— `new Singleton()` 的字节码分解
> `INSTANCE = new Singleton()` 实际分为三步：
> 1. `allocate` — 在堆上分配内存空间
> 2. `init` — 调用构造方法初始化对象
> 3. `assign` — 将引用赋值给 INSTANCE 变量
>
> 由于 **指令重排**，步骤可能变为 1→3→2：
> - 线程A执行到 1→3（INSTANCE 已经不是 null，但对象尚未初始化）
> - 线程B此时进入 `getInstance()`，第一次检查发现 `INSTANCE != null`
> - 线程B直接 return，拿到了一个**未初始化的对象**（空指针、错误状态等）
>
> 类比：你在网上买了一台电脑（分配空间），快递还在路上（尚未初始化），但物流系统已经显示"已送达"（引用已赋值）。别人看到"已送达"就去拿，拿到的是一个空箱子。
> CS 术语：这是**部分构造（Partially Constructed Object）** 的典型问题，由 **StoreStore 重排** 导致

#### 正确的 DCL（加 volatile）
```java
public class Singleton {
    private static volatile Singleton INSTANCE;  // ✅ volatile 禁止重排

    public static Singleton getInstance() {
        if (INSTANCE == null) {
            synchronized (Singleton.class) {
                if (INSTANCE == null) {
                    INSTANCE = new Singleton();
                    // volatile 写屏障确保：init 一定在 assign 之前完成
                }
            }
        }
        return INSTANCE;
    }
}
```

> [!tip] 更简洁的单例方案
> 利用类加载机制的线程安全性，避免 DCL 的复杂性：
> ```java
> // 静态内部类方案（推荐）
> public class Singleton {
>     private Singleton() {}
>
>     private static class Holder {
>         static final Singleton INSTANCE = new Singleton();
>     }
>
>     public static Singleton getInstance() {
>         return Holder.INSTANCE;
>     }
> }
> ```
> JVM 保证类的静态字段初始化是线程安全的（`<clinit>` 方法由 JVM 通过锁保证只执行一次）

---

## happens-before 规则

**happens-before** 是 JMM 中最核心的规则，它定义了什么情况下一个操作的结果**一定对另一个操作可见**。

> [!info] 理解 happens-before
> "A happens-before B" 并不是说 A 在时间上先于 B 执行，而是说 **A 的操作结果一定对 B 可见**
> 类比：这像是法律条文——不管你实际上什么时候办了事，法律规定你必须保证结果对另一方可见
> CS 术语：happens-before 是一种**偏序关系（Partial Order）**，定义了操作之间的可见性保证

### JMM 规定的 happens-before 规则

| 规则 | 含义 | 示例 |
|------|------|------|
| **程序顺序规则** | 同一线程中，前面的操作 hb 后面的操作 | `int a = 1; int b = a;` — a的写 hb b的读 |
| **volatile 规则** | volatile 写 hb 后续对该变量的 volatile 读 | 线程A写 `volatile x=1` hb 线程B读 `volatile x` |
| **锁规则** | 解锁（unlock）hb 后续对同一个锁的加锁（lock） | `synchronized` 块内的修改对后续获取同一把锁的线程可见 |
| **线程启动规则** | `t.start()` hb t 线程中的任何操作 | start 之前的变量修改，t 线程一定能看到 |
| **线程终止规则** | t 线程中的操作 hb 其他线程检测到 t 终止（`join()/isAlive()`） | t 内修改的变量，join 之后一定能看到 |
| **中断规则** | `t.interrupt()` hb 被中断线程检测到中断（`isInterrupted()`/catch） | — |
| **传递性** | 如果 A hb B，B hb C，则 A hb C | 串联上述规则 |
| **对象终结规则** | 对象构造方法结束 hb `finalize()` 方法开始 | — |

> [!warning] 核心价值
> happens-before 是你分析多线程程序**是否正确**的理论工具：
> - 如果两个操作之间**存在** happens-before 关系 → 前者的结果对后者**一定可见**
> - 如果两个操作之间**不存在** happens-before 关系 → 可能出现可见性问题
>
> 当你写并发代码时，应该确保关键操作之间有 happens-before 关系（通过 volatile、synchronized、Lock 等建立）

### volatile 的传递性应用

```java
int x = 0;
volatile boolean flag = false;

// 线程A
x = 42;                  // 普通写
flag = true;             // volatile 写（插入屏障）

// 线程B
if (flag) {              // volatile 读（有 happens-before）
    System.out.println(x); // 此时一定能读到 42
}
```

**推导链**：
1. `x = 42` hb `flag = true`（程序顺序规则）
2. `flag = true`（volatile 写）hb `if (flag)`（volatile 读）（volatile 规则）
3. 由传递性：`x = 42` hb `if (flag)` 后的读 x → **x 一定为 42**

> [!tip] 实际意义
> volatile 不仅影响自身变量的可见性，还**间接保证**了它周围普通变量的可见性（通过屏障和 happens-before 传递性）
> 这就是为什么 DCL 单例中 `volatile` 能保证对象完全初始化后才对其他线程可见

---

## 习题与实战

### 线程安全单例模式

#### 方案1：饿汉式
```java
public class Singleton {
    private static final Singleton INSTANCE = new Singleton();
    private Singleton() {}
    public static Singleton getInstance() { return INSTANCE; }
}
```
- **线程安全**：类加载时初始化，JVM 保证 `<clinit>` 的线程安全性
- **缺点**：不管用不用都会创建实例

#### 方案2：DCL + volatile（已分析）
- 见 [[#DCL 单例与 volatile]]

#### 方案3：静态内部类（推荐）
```java
public class Singleton {
    private Singleton() {}

    private static class LazyHolder {
        static final Singleton INSTANCE = new Singleton();
    }

    public static Singleton getInstance() {
        return LazyHolder.INSTANCE;
    }
}
```
- **懒加载**：只有调用 `getInstance()` 时才触发 `LazyHolder` 类的加载
- **线程安全**：JVM 类加载机制保证

> [!info] JVM 知识 —— 类加载的线程安全
> JVM 规范要求：同一个类加载器下，一个类的 `<clinit>()` 方法（静态初始化块）只会被一个线程执行，其他线程会被阻塞等待
> 这等价于一个天然的 `synchronized` 保护，不需要程序员手动加锁

#### 方案4：枚举单例（最安全）
```java
public enum Singleton {
    INSTANCE;

    public void doSomething() { ... }
}
```
- **防反射攻击**：枚举类不能通过反射调用 `newInstance()`
- **防序列化攻击**：枚举的序列化由 JVM 特殊处理，不会创建新实例
- **线程安全**：枚举本质上是静态 final 常量，类加载时初始化

> [!tip] 实际选型
> | 方案 | 懒加载 | 线程安全 | 防反射 | 防序列化 | 推荐场景 |
> |------|--------|---------|--------|---------|---------|
> | 饿汉式 | ❌ | ✅ | ❌ | ❌ | 简单场景，确定一定会使用 |
> | DCL | ✅ | ✅ | ❌ | ❌ | 面试重点，理解原理用 |
> | 静态内部类 | ✅ | ✅ | ❌ | ❌ | 一般推荐方案 |
> | 枚举 | ❌ | ✅ | ✅ | ✅ | 安全性要求最高时 |

---

## 第五章小结

| 概念 | 核心要点 |
|------|---------|
| 可见性 | 工作内存缓存导致线程间不可见，volatile 解决 |
| 原子性 | volatile **不保证**原子性，需要锁或 Atomic 类 |
| 有序性 | 编译器/CPU 指令重排，volatile 通过内存屏障禁止 |
| JMM | Java 对底层硬件的抽象，定义了线程与主内存的交互规则 |
| volatile 原理 | 写屏障（StoreStore + StoreLoad）+ 读屏障（LoadLoad + LoadStore） |
| happens-before | 判断多线程可见性的理论基础，7 条核心规则 + 传递性 |
| DCL | 必须加 volatile 防止部分构造对象 |
| 设计模式 | 两阶段终止（volatile 版）、Balking 犹豫模式 |

> [!warning] 最重要的结论
> - **可见性**用 `volatile` 或 `synchronized`
> - **原子性**用 `synchronized` 或 `Atomic` 类（CAS）
> - **有序性**用 `volatile`
> - 当你对并发正确性有疑问时，回来用 **happens-before** 规则推导

---
