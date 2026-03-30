### Part A
1. 答：
    - 名称是`ABA问题`；
    - 在T1视角下，它在一开始拿到的值就是100，然后做了减法之后，再去读取值，发现还是100，说明没有问题，可以交换；
    - 在“业务”视角下，按照日志顺序的正常来说，应该是100扣款成功，然后50扣款失败，充值50成功这个顺序；
    - 引起的原因是T1并不知道100被改了，它以为还是原来的100，虽然在这个例子下我们可以认为并不是一个很严重的问题，三个操作账户总余额：`100 - 50 + 50 - 100 = 0`，但是如果说这个并不是普通的加减；
    - 假设这是一个栈结构（里面初始为`CBA`），那我线程T1要弹出`A`让下一次的栈顶变成`B`，但是在弹出的时候，刚好被另外一个线程做了如下修改：弹出`A`和弹出`B`，然后将`D`和`A`压入栈，然后变成了`CDA`，然后线程T1回来一看，发现栈顶还是`A`，没问题，将`A`弹出；这就会导致产生很严重的问题了；

2. 答：
    - 在底层是使用了CPU自带锁的指令`CMPXCHG`，这个指令CPU会自己来在“对比”+“交换”操作前后上`Lock`
    - 为什么不会涉及到内核态的切换呢？因为这个指令是CPU自己来加锁，并不是要从用户态切换到操作系统来加互斥锁这种重量级操作；

3. 答：
- 基于`stamp`实现
```java
import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.util.concurrent.atomic.AtomicReference;

@Data
@NoArgsConstructor
@AllArgsConstructor
class AtomicIntegerWithStamp {
    private int value;
    private long stamp;
}

class PointsAccount {
    // 用户当前积分，初始 100
    private AtomicReference<AtomicIntegerWithStamp> points = 
            new AtomicReference<>(new AtomicIntegerWithStamp(100, 0));

    /**
     * 尝试扣除指定积分
     * @return 扣除成功返回 true
     */
    public boolean deduct(int amount) {
        while (true) {
            AtomicIntegerWithStamp old = points.get();
            if (old.getValue() < amount) return false;
            AtomicIntegerWithStamp newPoints = 
                    new AtomicIntegerWithStamp(old.getValue() - amount, old.getStamp() + 1);
            if (points.compareAndSet(old, newPoints)) {
                return true;
            }
        }
    }

    public int getPoints() {
        return points.get().getValue();
    }
}
```
> 适用于高并发但是低竞争的场景

- 基于`ReentrantLock`实现
```java
import java.util.concurrent.atomic.AtomicInteger;
import java.util.concurrent.locks.ReentrantLock;

public class PointsAccount {
    // 用户当前积分，初始 100
    private AtomicInteger points = new AtomicInteger(100);
    private final ReentrantLock lock = new ReentrantLock();

    /**
     * 尝试扣除指定积分
     * @return 扣除成功返回 true
     */
    public boolean deduct(int amount) {
        lock.lock();
        try {
            if (points.get() >= amount) {
                points.addAndGet(-amount);
                return true;
            } else {
                return false;
            }
        } finally {
            lock.unlock();
        }
    }

    public int getPoints() {
        return points.get();
    }
}
```
> 这个更合适是高竞争的场景

---

### Part 2

> 首先我使用了JDK21的版本，为了模拟200个用户发起请求，采用了jdk21的虚拟线程技术

- 首先我先尝试了用`AtomicInteger`，我发现了`QPS`居然达到了惊人的`1000/s`，但是我也发现失败率居然达到了惊人的`100%`，笑死我了；以下是我写的代码
- 后续其实我发现就是`token生成器`出了问题，修复之后开始测试
```java
import io.jsonwebtoken.Jwts;
import io.jsonwebtoken.SignatureAlgorithm;
import lombok.Getter;
import lombok.extern.slf4j.Slf4j;

import java.util.*;
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.concurrent.locks.LockSupport;

@Slf4j
public class CurrentLimiter {

    public static class JwtTokenGenerator {
        private static final Random RD = new Random();  // Random generator UserId
        private static final String SECRET = "TestMyLimiter";
        private static final long KEEP_ALIVE_TIME = 6000;   // 1 minute

        public static String generateToken() {
            HashMap<String, Object> claims = new HashMap<>();
            claims.put("userId", RD.nextInt(10000));

            String token = Jwts.builder()
                    .signWith(SignatureAlgorithm.HS256, SECRET)
                    .setClaims(claims)
                    .setExpiration(new Date(System.currentTimeMillis() + KEEP_ALIVE_TIME))
                    .compact();

            return token;
        }
    }

    static final int MAX_TRY_TIMES = 5;

    static final int CORE_SIZE = 25;    // My CPU cores 32~64
    static final int MAX_SIZE = 40;
    static final long KEEP_ALIVE_TIME = 6000;
    static final int MAX_TOKENS = 1000;         // token max size
    static final int RECOVERY_NUM = 500;        // token recovery number
    static final int MAX_TASK_SIZE = 3000;
    static final ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(2);
    static final ThreadPoolExecutor threadPool = new ThreadPoolExecutor(
            CORE_SIZE,
            MAX_SIZE,
            KEEP_ALIVE_TIME,
            TimeUnit.MILLISECONDS,
            new LinkedBlockingQueue<Runnable>(MAX_TASK_SIZE),
            (Runnable r, ThreadPoolExecutor executor) -> {
                try {
                    // 尝试在 1 秒内将任务放入队列
                    boolean success = executor.getQueue().offer(r, 1, TimeUnit.SECONDS);

                    if (!success) {
                        // 如果 1 秒后依然失败，抛出拒绝执行异常
                        throw new RejectedExecutionException("Task rejected after waiting 1 second for queue space.");
                    }
                } catch (InterruptedException e) {
                    // 如果等待过程中线程被中断，恢复中断状态并抛出异常
                    Thread.currentThread().interrupt();
                    throw new RejectedExecutionException("Task interrupted while waiting for queue space.", e);
                }
            }
    );

    static final LinkedBlockingQueue<Thread> parkedThreads = new LinkedBlockingQueue<>();

    // initial value is 0
    static final AtomicInteger tokenPool = new AtomicInteger(MAX_TOKENS);

    @Getter
    static long successRecoveryCount = 0;

    static {
        scheduler.scheduleAtFixedRate(() -> {
            int pre;
            int next;
            do {
                pre = tokenPool.get();
                next = Math.min(pre + RECOVERY_NUM, MAX_TOKENS);
            } while (tokenPool.compareAndSet(pre, next) && pre != MAX_TOKENS);
            successRecoveryCount += next - pre;

            // 唤醒所有阻塞线程
            ArrayList<Thread> threads = new ArrayList<>(parkedThreads);
            parkedThreads.removeAll(threads);
            threads.forEach(LockSupport::unpark);
        }, 0, 1, TimeUnit.SECONDS);
    }

    public static String tryGetToken() {
        Future<?> result = threadPool.submit(() -> {
            Thread cur = Thread.currentThread();
            while (tokenPool.get() <= 0) {
                parkedThreads.add(cur);
                LockSupport.park();
            }

            int time = 0;
            for (; time < MAX_TRY_TIMES; time++) {
                int pre = tokenPool.get();
                if (pre <= 0) continue;
                int next = pre - 1;
                if (tokenPool.compareAndSet(pre, next)) {
                    return JwtTokenGenerator.generateToken();
                }
            }
            return null;
        });

        try {
            String token = (String) result.get();
            return token;
        } catch (InterruptedException | ExecutionException e) {
            return null;
        }
    }
}
```

- 第一次改进
```java
private static final int MAX_TRY_TIMES = 3;
// ...
static {
        scheduler.scheduleAtFixedRate(() -> {
            if (tokenPool.get() < MAX_TOKENS) {
                int pre, next;
                do {
                    pre = tokenPool.get();
                    next = Math.min(pre + RECOVERY_NUM, MAX_TOKENS);
                    if (pre >= MAX_TOKENS) break;
                } while (!tokenPool.compareAndSet(pre, next));
                successRecoveryCount += next - pre;

                // 唤醒所有阻塞线程
                ArrayList<Thread> threads = new ArrayList<>(parkedThreads);
                parkedThreads.removeAll(threads);
                threads.forEach(LockSupport::unpark);
            }
        }, 0, 1, TimeUnit.SECONDS);
    }

    public static String tryGetToken() {
        Future<String> result = threadPool.submit(() -> {
            Thread cur = Thread.currentThread();
            isNeedToPark(cur);

            int time = 0;
            for (; time < MAX_TRY_TIMES; time++) {
                int pre = tokenPool.get();
                if (pre <= 0) {
                    boolean flag = isNeedToPark(cur);
                    if (flag) time = 0;
                }
                int next = pre - 1;
                if (tokenPool.compareAndSet(pre, next)) {
                    return JwtTokenGenerator.generateToken();
                }
            }
            return null;
        });

        try {
            return result.get();
        } catch (InterruptedException |  ExecutionException e) {
            return null;
        }
    }

    private static boolean isNeedToPark(Thread cur) {
        if (tokenPool.get() <= 0) {
            while (tokenPool.get() <= 0) {
                parkedThreads.add(cur);
                LockSupport.park();
            }
            return true;
        }
        return false;
    }
```

- 测试结果：
```txt
========== 开始热身 (Warm-up) ==========
所有线程就绪，开始压测...
测试持续时间: 3.07 秒
并发线程数: 100
总请求数: 2100
----------------------------------------
成功获取 Token 数: 2100
未获取到 (返回 null) 数: 0
异常崩溃数: 0
----------------------------------------
总 QPS (吞吐量): 683.37 req/s
成功 QPS: 683.37 req/s
错误率: 0.00%
期间内后台令牌桶恢复成功的量：1500

========== 开始第一轮压力测试 (Stress Test) ==========
所有线程就绪，开始压测...
测试持续时间: 12.00 秒
并发线程数: 1000
总请求数: 6400
----------------------------------------
成功获取 Token 数: 6400
未获取到 (返回 null) 数: 0
异常崩溃数: 0
----------------------------------------
总 QPS (吞吐量): 533.24 req/s
成功 QPS: 533.24 req/s
错误率: 0.00%
期间内后台令牌桶恢复成功的量：6000

========== 开始高压力测试 (Stress Test) ==========
所有线程就绪，开始压测...
测试持续时间: 26.98 秒
并发线程数: 4000
总请求数: 30025
----------------------------------------
成功获取 Token 数: 13296
未获取到 (返回 null) 数: 0
异常崩溃数: 16729
----------------------------------------
总 QPS (吞吐量): 1112.86 req/s
成功 QPS: 492.81 req/s
错误率: 55.72%
期间内后台令牌桶恢复成功的量：13500
```
> 结果计算：
> 1000 + 1500 - 2100 = 400
> 400 + 6000 - 6500 = 0
> 13500 - 13296 = 204
> 没有出现负数，没有超出


- 第二次打算使用`ReentrantLock`的读写锁，我打算把`tryGetToken`当作一次读操作，然后恢复500个桶当作是写操作；
```java
import io.jsonwebtoken.Jwts;
import io.jsonwebtoken.SignatureAlgorithm;
import lombok.Getter;
import lombok.extern.slf4j.Slf4j;

import java.util.*;
import java.util.concurrent.*;
import java.util.concurrent.locks.LockSupport;
import java.util.concurrent.locks.ReentrantReadWriteLock;

@Slf4j
public class CurrentLimiter {

    public static class JwtTokenGenerator {
        private static final Random RD = new Random();  // Random generator UserId
        private static final String SECRET = "TestMyLimiter";
        private static final long KEEP_ALIVE_TIME = 6000;   // 1 minute

        public static String generateToken() {
            HashMap<String, Object> claims = new HashMap<>();
            claims.put("userId", RD.nextInt(10000));

            String token = Jwts.builder()
                    .signWith(SignatureAlgorithm.HS256, SECRET)
                    .setClaims(claims)
                    .setExpiration(new Date(System.currentTimeMillis() + KEEP_ALIVE_TIME))
                    .compact();

            return token;
        }
    }


    private static final int CORE_SIZE = 25;    // My CPU cores 32~64
    private static final int MAX_SIZE = 40;
    private static final long KEEP_ALIVE_TIME = 6000;
    private static final int MAX_TOKENS = 1000;         // token max size
    private static final int RECOVERY_NUM = 500;        // token recovery number
    private static final int MAX_TASK_SIZE = 3000;
    private static final ThreadPoolExecutor threadPool = new ThreadPoolExecutor(
            CORE_SIZE,
            MAX_SIZE,
            KEEP_ALIVE_TIME,
            TimeUnit.MILLISECONDS,
            new LinkedBlockingQueue<Runnable>(MAX_TASK_SIZE),
            (Runnable r, ThreadPoolExecutor executor) -> {
                try {
                    // 尝试在 1 秒内将任务放入队列
                    boolean success = executor.getQueue().offer(r, 1, TimeUnit.SECONDS);

                    if (!success) {
                        // 如果 1 秒后依然失败，抛出拒绝执行异常
                        throw new RejectedExecutionException("Task rejected after waiting 1 second for queue space.");
                    }
                } catch (InterruptedException e) {
                    // 如果等待过程中线程被中断，恢复中断状态并抛出异常
                    Thread.currentThread().interrupt();
                    throw new RejectedExecutionException("Task interrupted while waiting for queue space.", e);
                }
            }
    );

    private static final ReentrantReadWriteLock lock = new ReentrantReadWriteLock();
    private static final ReentrantReadWriteLock.ReadLock readLock = lock.readLock();
    private static final ReentrantReadWriteLock.WriteLock writeLock = lock.writeLock();

    private static final LinkedBlockingQueue<Thread> parkedThreads = new LinkedBlockingQueue<>();

    // initial value is 0
    private static int tokenPool = MAX_TOKENS;

    @Getter
    private static int successRecoveryCount = 0;


    private static final ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(2);
    static {
        scheduler.scheduleAtFixedRate(() -> {
            writeLock.lock();
            try {
                successRecoveryCount -= tokenPool;
                tokenPool = Math.min(tokenPool + RECOVERY_NUM, MAX_TOKENS);
                successRecoveryCount += tokenPool;
            } finally {
                writeLock.unlock();
            }

            // 唤醒所有阻塞线程
            ArrayList<Thread> threads = new ArrayList<>(parkedThreads);
            parkedThreads.removeAll(threads);
            threads.forEach(LockSupport::unpark);
        }, 0, 1, TimeUnit.SECONDS);
    }

    public static String tryGetToken() {
        Future<?> result = threadPool.submit(() -> {
            Thread cur = Thread.currentThread();
            while (tokenPool <= 0) {
                parkedThreads.add(cur);
                LockSupport.park();
            }

            writeLock.lock();
            try {
                tokenPool--;
            } finally {
                writeLock.unlock();
            }
            return JwtTokenGenerator.generateToken();
        });

        try {
            String token = (String) result.get();
            return token;
        } catch (InterruptedException | ExecutionException e) {
            return null;
        }
    }
}
```

- 测试数据如下：
```txt
========== 开始热身 (Warm-up) ==========
所有线程就绪，开始压测...
测试持续时间: 3.07 秒
并发线程数: 100
总请求数: 2100
----------------------------------------
成功获取 Token 数: 2100
未获取到 (返回 null) 数: 0
异常崩溃数: 0
----------------------------------------
总 QPS (吞吐量): 684.26 req/s
成功 QPS: 684.26 req/s
错误率: 0.00%
期间内后台令牌桶恢复成功的量：1500

========== 开始第一轮压力测试 (Stress Test) ==========
所有线程就绪，开始压测...
测试持续时间: 12.00 秒
并发线程数: 1000
总请求数: 6400
----------------------------------------
成功获取 Token 数: 6400
未获取到 (返回 null) 数: 0
异常崩溃数: 0
----------------------------------------
总 QPS (吞吐量): 533.51 req/s
成功 QPS: 533.51 req/s
错误率: 0.00%
期间内后台令牌桶恢复成功的量：6000

========== 开始高压力测试 (Stress Test) ==========
所有线程就绪，开始压测...
测试持续时间: 26.97 秒
并发线程数: 4000
总请求数: 29831
----------------------------------------
成功获取 Token 数: 13217
未获取到 (返回 null) 数: 0
异常崩溃数: 16614
----------------------------------------
总 QPS (吞吐量): 1105.96 req/s
成功 QPS: 490.01 req/s
错误率: 55.69%
期间内后台令牌桶恢复成功的量：13500
```
> 其实我感觉第二种做法有点问题的，因为读锁大家都跑进去做`tokenPool--`了
> 理论上会出现线程安全问题，就是应该略微会超出一点的
> 但是测试的时候数据没有问题，我跑了好几次都是这样，这就很奇怪了

> 去了，结果发现我写了都是写锁，怪不得，不过性能确实是及格了


> 以下是我搞了用来测试的裸测试代码
```java
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.atomic.LongAdder;

public class TestLimiter {
    static long pre = 0;

    public static void main(String[] args) throws InterruptedException {
        System.out.println("========== 开始热身 (Warm-up) ==========");
        // 热身阶段：少量线程，短时间运行，让 JVM 完成类加载和 JIT 即时编译优化
        runTest(100, 3);

        System.out.println("\n========== 开始第一轮压力测试 (Stress Test) ==========");
        // 正式测试：模拟 1000 个并发用户，持续压测 10 秒
        runTest(1000, 10);

        System.out.println("\n========== 开始高压力测试 (Stress Test) ==========");
        // 正式测试：模拟 4000 个并发用户，持续压测 10 秒
        runTest(4000, 20);

        // 测试结束后强制退出，防止 CurrentLimiter 里的 ScheduledExecutorService 阻止 JVM 关闭
        System.exit(0);
    }

    /**
     * 模拟并发压测核心逻辑
     *
     * @param threadCount 模拟的并发线程数（虚拟用户数）
     * @param durationSec 持续时间（秒）
     */
    private static void runTest(int threadCount, int durationSec) throws InterruptedException {
        ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor();
        CountDownLatch startLatch = new CountDownLatch(1); // 保证所有线程同时起跑
        CountDownLatch endLatch = new CountDownLatch(threadCount); // 等待所有线程执行完毕

        // 统计指标：使用 LongAdder 避免高并发下 AtomicLong 造成的 CPU 缓存行竞争
        LongAdder totalRequests = new LongAdder();
        LongAdder successCount = new LongAdder();
        LongAdder nullCount = new LongAdder();
        LongAdder exceptionCount = new LongAdder();

        long endTimeMs = System.currentTimeMillis() + (durationSec * 1000L);

        for (int i = 0; i < threadCount; i++) {
            executor.submit(() -> {
                try {
                    startLatch.await(); // 阻塞，等待发令枪

                    // 在指定时间内疯狂循环请求
                    while (System.currentTimeMillis() < endTimeMs) {
                        totalRequests.increment();
                        try {
                            String token = CurrentLimiter.tryGetToken();
                            if (token != null) {
                                successCount.increment();
                            } else {
                                // 达到 MAX_TRY_TIMES 后依然没拿到，返回了 null
                                nullCount.increment();
                            }
                        } catch (Exception e) {
                            exceptionCount.increment();
                        }
                    }
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                } finally {
                    endLatch.countDown(); // 跑完汇报
                }
            });
        }

        System.out.println("所有线程就绪，开始压测...");
        long actualStartTime = System.currentTimeMillis();
        startLatch.countDown(); // 鸣枪！所有线程同时开始跑

        endLatch.await(); // 等待所有线程跑完时间
        long actualEndTime = System.currentTimeMillis();
        executor.shutdown();

        // --- 统计结果计算 ---
        double actualDurationSec = (actualEndTime - actualStartTime) / 1000.0;
        long total = totalRequests.sum();
        long success = successCount.sum();
        long nulls = nullCount.sum();
        long exceptions = exceptionCount.sum();

        double qps = total / actualDurationSec;
        double successQps = success / actualDurationSec;

        System.out.printf("测试持续时间: %.2f 秒%n", actualDurationSec);
        System.out.printf("并发线程数: %d%n", threadCount);
        System.out.printf("总请求数: %d%n", total);
        System.out.println("----------------------------------------");
        System.out.printf("成功获取 Token 数: %d%n", success);
        System.out.printf("未获取到 (返回 null) 数: %d%n", nulls);
        System.out.printf("异常崩溃数: %d%n", exceptions);
        System.out.println("----------------------------------------");
        System.out.printf("总 QPS (吞吐量): %.2f req/s%n", qps);
        System.out.printf("成功 QPS: %.2f req/s%n", successQps);
        System.out.printf("错误率: %.2f%%%n", (double)(nulls + exceptions) / total * 100);
        System.out.println("期间内后台令牌桶恢复成功的量：" + (CurrentLimiter.getSuccessRecoveryCount() - pre));
        pre = CurrentLimiter.getSuccessRecoveryCount();
    }
}
```

### 从AI那里得到的信息

#### AtomicInteger解法
##### 第一轮信息
###### 1. 灵魂拷问：CAS 的返回值与 `do-while` 循环
请重点关注 `scheduler` 里负责恢复 Token 的这段核心代码：
```java
do {
    pre = tokenPool.get();
    next = Math.min(pre + RECOVERY_NUM, MAX_TOKENS);
    if (pre >= MAX_TOKENS) break;
} while (tokenPool.compareAndSet(pre, next)); 
successRecoveryCount += next - pre;
```

* **线索 A：** `compareAndSet` 如果**成功**修改了值，会返回 `true` 还是 `false`？如果它返回了 `true`，你的 `while` 循环会继续执行还是退出？这会导致你的 Token 池是一次性加满，还是按照 `RECOVERY_NUM` 逐步增加？
* **线索 B：** 假设在高并发下，`compareAndSet` 因为竞争**失败**了（返回了另一布尔值），循环退出了。紧接着下面的 `successRecoveryCount += next - pre;` 会发生什么？你是不是把“尝试失败”的额度也算进“成功恢复”的计数里了？

###### 2. 线程唤醒后的“饥饿”陷阱
来看看当定时任务发放 Token 并唤醒线程时发生了什么，以及 `tryGetToken` 里的这段逻辑：

```java
while (tokenPool.get() <= 0) {
    parkedThreads.add(cur);
    LockSupport.park();
}

int time = 0;
for (; time < MAX_TRY_TIMES; time++) {
    int pre = tokenPool.get();
    if (pre <= 0) continue;
    // ...尝试获取...
}
return null;
```

* **线索 C：** 你的定时器每秒可能会同时 `unpark` 唤醒几百个线程。它们醒来后会立刻进入 `for` 循环抢 Token。如果抢得慢的线程发现 `pre <= 0`（Token 已经被其他刚醒的线程抢光了），你的代码里用了 `continue`。这意味着它会瞬间空转 5 次，然后直接返回 `null`。
* **思考：** 一个被唤醒的线程如果没抢到 Token，它是应该直接被丢弃（返回 null），还是应该重新回到 `park()` 状态，等待下一秒的 Token 发放？这直接影响了你压测结果中“未获取到 (返回 null) 数”的比例。

###### 3. 阻塞模型的隐患（架构设计层面）
你在 `tryGetToken` 中使用了传统线程池 `threadPool.submit(...)`，然后立刻调用 `result.get()` 同步阻塞等待结果。而在 `TestLimiter` 中，你又使用了 JDK 21 的虚拟线程（Virtual Threads）来发起请求。

* **线索 D：** 虚拟线程本来是非常轻量级的，但当它调用 `result.get()` 时，它就被迫停下来等待你那 25~40 个核心线程去处理队列里多达 3000 个任务。
* **思考：** 你现在的瓶颈，到底是被 `tokenPool` 限制住了（真正的限流），还是仅仅因为你的 `threadPool` 消费不过来，导致大量请求在排队或者被 `RejectedExecutionException` 拒绝掉了？

---

### 极客挑战 1
- 答：
    - 在低并发下，LongAdder 和 AtomicLong 其实都只是 CAS 自旋操作，但是在高并发下`LongAdder`会多出一个冲突扩容的功能
    - 在高并发下，`Cell[]`数组最大的大小是跟CPU核心数相同，初始大小是2；
    - 用`@Contented`修饰是因为IEMS协议，因为一整条缓存行是可以存储不止一个Long变量，如果不加入这个注解，就是相当于两个变量在同一行里面，若是`core-1`拿到了`变量A`在数据行X上，同时`core-2`拿到了`变量B`也在数据行X上，就会导致竞争，相互使对方拿到的变量失效，导致伪共享现象；
    - 不精确，在高并发下会比精确值略低；比如做限流器这种软限制的场景，不能用在硬限制的场景（比如每日这个接口普通用户只能调用5次这样）；

---

