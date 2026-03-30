# Level 3 · 评分报告

> **考核模块**：JMM 内存模型与无锁并发 —— volatile / DCL · LongAdder 计数器
> **考核日期**：2026-03-30
> **评分规则**：百分制，Part A（40分）+ Part B（60分）；极客挑战额外计算

---

## 🏆 总分

| 模块 | 满分 | 实得 |
|---|---|---|
| Part A · Q1 DCL 半初始化 + 屏障方向 | 40 | 30 |
| Part B · Q1 long counter 线程不安全 | 10 | 9 |
| Part B · Q2 AtomicLong 总线风暴（MESI） | 15 | 14 |
| Part B · Q3 LongAdder 原理四问 | 25 | 20 |
| Part B · Q4 AtomicLong vs LongAdder 实践 | 10 | 9 |
| **合计** | **100** | **82** |

> ✅ **综合判定**：82 分。JMM 内存屏障的方向理解已经到位，MESI 总线风暴讲得很清楚，LongAdder 的分治思路和 @Contended 的原因也能说明白。扣分集中在 DCL 的字节码拆分不够精确（混淆了"NPE 的来源"）和屏障类型的对应关系。允许进入 Level 4。

---

## 📋 逐题点评

---

### Part A · Q1：DCL 半初始化 + 屏障（30 / 40）

✅ **答对的部分**：
- 正确识别根本原因：`instance` 被赋值了指针地址，但对象内部字段还未初始化——这是"半初始化对象"的核心。
- 字节码拆解方向正确：列出了 `new / dup / invokespecial / putstatic` 的顺序，并且识别出问题在 `21（invokespecial）` 和 `24（putstatic）` 发生重排序。
- `volatile` 加入后内存屏障的插入位置（`putstatic` 前加 `StoreStore`，之后加 `StoreLoad`）是正确的。
- 用 mermaid 流程图辅助说明，思路清晰。

❌ **扣分点（-10）**：

**① "config 为 null" 的根因描述有偏差（-4）**

> "config 对象可能已经被赋值了指针地址，但是那个地址的对象还没被初始化，所以直接返回了一个 null 没被初始化的对象"

这个描述指向的是 `ConfigCenter` 对象本身还未初始化——但题目代码的 `getInstance()` 返回的是 `ConfigCenter` 实例（**不为 null**），NPE 发生在 `config.get(key)`，即 **`config` 这个 `Map` 字段为 null**。

正确的因果链是：

```
线程1执行 instance = new ConfigCenter() 发生指令重排：
  步骤1: new ConfigCenter（分配内存，所有字段默认为 null）
  步骤2: putstatic instance ← 此时 config 字段还是 null（构造函数未执行）
  步骤3: invokespecial <init>（执行构造函数，this.config = loadFromRemote()）

线程2在步骤2完成后、步骤3之前执行第一次检查：
  if (instance == null) → 不为 null！拿到了半初始化的 ConfigCenter
  return instance → 拿到的对象中 config 字段是 null
  getConfig(key) → NullPointerException
```

关键是：半初始化的是 **`ConfigCenter`** 对象内的 **`config` 字段**，不是 `config` 对象的地址被赋了但对象没创建——这两个描述有本质区别。

**② 字节码拆解的步骤粒度不够精确（-3）**

你列出的字节码中 `dup` 的说明是"purpose to init the object"，这不准确。`dup` 的作用是**复制操作数栈顶的引用**——因为 `invokespecial <init>` 会消耗一个引用（`this`），而后续 `putstatic` 还需要一个引用，所以要 dup 出两份。

标准的三步拆解应该是：
```
步骤1：new         → 分配内存，对象字段全部默认值（config = null）
步骤2：invokespecial <init> → 执行构造函数（config = loadFromRemote()）
步骤3：putstatic instance  → 将引用写入静态字段
```
指令重排导致步骤 3 跑到步骤 2 前面：先写 `instance`，再执行构造函数。

**③ 屏障类型名称写对了但对应关系说反了（-3）**

你在答案里写的是：
> "其实是发生了'写后读'，机器指令会变成如下：[在 getstatic 后加 LoadLoad+LoadStore，在 putstatic 前加 StoreStore，在 putstatic 后加 StoreLoad]"

屏障的**插入位置**是正确的，但命名"写后读"不准确。DCL 这里需要禁止的是 **"写后写"被重排（即 `invokespecial` 写字段 → `putstatic` 写引用，不能被重排）**，用的是 `StoreStore` 屏障——禁止 store 重排到 store 后面。

`StoreLoad` 屏障（`putstatic` 之后）是为了保证线程2能看到线程1写入的最新值（可见性），这是"写后读"屏障，但它的目的是可见性而不是防止半初始化。你把两个屏障的职责混在一起了。

| 屏障 | 位置 | 目的 |
|---|---|---|
| `StoreStore` | `putstatic` 之前 | 禁止构造函数内的写（步骤2）重排到 instance 赋值（步骤3）之后 |
| `StoreLoad` | `putstatic` 之后 | 保证其他线程读到 instance 时，构造函数的写操作也对它可见 |

---

### Part B · Q1：long counter 线程不安全（9 / 10）

✅ **序列图画得很清楚**，CPU1 读到 0、counter++、写回 1，CPU2 同时做了一样的事，最终丢了一次计数——JMM 工作内存/主内存的数据竞争过程图示正确。

❌ **扣分点（-1）**：

`counter++` 的非原子性可以更精确地从三步拆解来说明：**read → modify → write** 三步之间可以插入其他线程的操作。你的时序图展示了这个，但文字说明部分没有明确提到"++ 不是原子操作，是由 3 条字节码指令组成"（`getfield` / `iadd` / `putfield`），这是面试中被追问时必须能说出来的。

---

### Part B · Q2：AtomicLong 总线风暴（14 / 15）

✅ **这是本关最亮眼的作答之一**。序列图准确地展示了 MESI 协议下多核争抢同一缓存行的过程：E（独占）→ M（修改）→ 导致其他核心 I（失效）→ 再次 E 读取 → 循环。图示清晰且符合 MESI 协议的状态转换规则。

❌ **扣分点（-1）**：

序列图中写"读取含有 count 变量的行到缓存区（E）"，但实际上初次读取的状态应该是 **S（Shared）**（多个核心共享读），只有当某个核心申请修改时才升级到 M，同时其他核心变成 I。你直接从 E 开始，跳过了多核场景下初始的 S 状态，对 MESI 四态转换的完整路径有一点点省略。

---

### Part B · Q3：LongAdder 原理四问（20 / 25）

✅ **答对的核心**：
- `base` + `Cell[]` 的职责划分正确。
- 低并发用 `base`，高并发进 `longAccumulate` 触发 `Cell[]` 扩容——正确。
- `@Contended` 防止伪共享，缓存行 64 字节，多个 Cell 共享一行会互相失效——原因讲清楚了。
- `sum()` 不精确，适合软限制场景（流量统计）不适合硬限制——结论正确。

❌ **扣分点（-5）**：

**① `@Contended` 的实现机制没有说清楚（-2）**

你说"用了才能防止行级伪共享"，但没有说**怎么防**。`@Contended` 的实现是在对象前后填充 **128 字节的 padding 空间**，使每个 `Cell` 独占一整条缓存行，物理上隔离不同核心的写操作。不说填充机制，这个知识点只答了一半。

**② `sum()` 偏差方向有误（-2）**（和极客挑战 1 同一个错误）

> "高并发下会比精确值略低"

`sum()` 是 `base + Σ(cells[i])`，求和期间没有锁，并发写入可能被读到（偏高）也可能漏掉（偏低），是**不确定方向的近似值**，不是固定偏低。

**③ Cell 初始化的时机（-1）**

你说"初始化长度为 2，用到才会进行初始化"，这是正确的，但可以更精确：Cell 数组扩容时新的槽位被分配内存（`new Cell[]`），但数组中的每个 Cell 对象是**懒初始化**的——第一次有线程 hash 到某个槽位时，才给该槽创建 `Cell` 对象。这个细节在你的极客挑战 2 的图里画出来了，在正文里可以补一句。

---

### Part B · Q4：AtomicLong vs LongAdder 实践（9 / 10）

✅ 结论正确（肯定更慢），理由正确（总线风暴）。提到了 `pause` 指令作为缓冲手段，说明你对 CAS 自旋的底层优化有了解。

❌ **扣分点（-1）**：

> "特别是高压测试的 2000 个虚拟线程"

你的压测代码里最高并发是 4000 个虚拟线程（`runTest(4000, 20)`），说成 2000 有点记混了，这是小细节扣分。

---

## 🎮 极客挑战 · 补充评分

### 极客挑战 2 · `LongAdder` 的 `Cell[]` 扩容时机（22 / 25）

> 极客挑战 1 已跳过（标注 null）。

✅ **这道题你做的非常认真**：
- 扩容触发条件（CAS 竞争失败）和最大上限（CPU 核心数）——正确。
- `add()` 方法的流程图：`cells 未创建 → CAS base → 失败 → longAccumulate` 与 `cells 已创建 → hash 找 Cell → CAS Cell → 失败 → longAccumulate`——完全正确。
- `longAccumulate` 三路分支（cells 已创建且有大小 / cells 未创建 / 兜底用 base）的主干流程——基本正确，能看出你认真读过源码。

❌ **扣分点（-3）**：

**① `COLLIDE` 和 `wasUncontended` 的区分没有点明（-2）**

题目问的核心是这两个标志位的语义和配合方式，你的流程图里用的是中文描述（"是否计算标识符错误" / "冲突标识符"），没有直接对应源码变量名：

| 源码变量 | 语义 | 作用 |
|---|---|---|
| `wasUncontended` | 上一次 CAS Cell 是否无竞争 | 首次进入 longAccumulate 时，如果 Cell 存在但 CAS 失败，置为 false，下一轮跳过 CAS 直接换位置或扩容 |
| `collide` | 上一轮是否已经发生了碰撞 | true 时触发扩容；false 时先尝试换 hash 位置（重试一次），再标记 collide=true |

两者配合的逻辑是：先换位置（重试），再标记碰撞（collide），再扩容（cells × 2）——三步递进，避免频繁扩容。你的图里这个递进关系不够清晰。

**② 流程图中部分箭头条件表述模糊（-1）**

比如 `B1 → C1` 标注的是"存在该线程使用的 cell，但是是否计算标识符错误"，这个"是否"让人不知道是"是"还是"否"触发这个路径。建议用"计算标识符为错误（wasUncontended=false）"或直接写布尔值。

> **极客挑战 2 小计：22 / 25**

---

## 📊 能力雷达图（文字版）

| 能力维度 | 评价 |
|---|---|
| volatile / JMM 屏障 | ⭐⭐⭐⭐ 良好，屏障位置正确，类型含义有混淆但方向对 |
| DCL 半初始化分析 | ⭐⭐⭐ 中等，根因指向正确但字节码粒度和描述有偏差 |
| MESI 协议理解 | ⭐⭐⭐⭐⭐ 优秀，总线风暴时序图是本关最佳表现 |
| LongAdder 分治原理 | ⭐⭐⭐⭐ 良好，@Contended 机制细节稍缺，sum() 方向有误 |
| 源码阅读能力 | ⭐⭐⭐⭐ 良好，极客挑战 2 的 longAccumulate 流程图说明认真读过源码 |
| 概念精确性 | ⭐⭐⭐ 中等，MESI 协议名、sum() 偏差方向等多处细节需订正 |

---

## 🎯 进入 Level 4 前的补课清单

Level 4 的战场是 **AQS 骨架 + ReentrantLock 源码推演**，这是你在 Level 1 就暴露过弱点的区域，现在带着 JMM 和 CAS 的更扎实的理解再去看，会完全不同：

1. **必看**：黑马 `08.037 ~ 08.039`（AQS 概述 + 自定义锁）——Level 1 混淆 AQS 和 synchronized 的旧账，在这里系统清理
2. **必看**：黑马 `08.040 ~ 08.046`（ReentrantLock 加锁/解锁全流程 + 公平锁）——源码推演是 Level 4 主战场
3. **带着问题去看**：`ReentrantLock.lock()` 的非公平锁第一步为什么是直接 CAS（不入队）？公平锁为什么必须先判断队列是否有前驱节点？

---

## ✅ 导师判定

> **82 分，允许进入 Level 4。**
>
> 这一关最惊喜的是 MESI 总线风暴的时序图——你不只是说了"会有竞争"，而是画出了 E → M → I → E 的状态转换链，这说明你对 CPU 缓存一致性有真实的认知，不是背答案。
>
> Level 4（AQS + ReentrantLock）是 Level 1 的"还债关"，你当时把 AQS 和 synchronized 搅成了一锅粥。现在带着 CAS、MESI、LockSupport 的扎实理解去看 AQS 源码，应该会豁然开朗。
