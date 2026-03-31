### Part 1
1. 答：
    - 流程图
    ```mermaid
    flowchart TD
        A["lock()"] --> B{"CAS: state 0→1?"}
        B -->|成功| C["设置 Owner 为当前线程"]
        B -->|失败| D["acquire(1)"]
        D --> E{"tryAcquire: state==0?"}
        E -->|是| F["CAS 0→1, 设 Owner"]
        E -->|否| G{"Owner == 当前线程?"}
        G -->|是| H["state + 1（重入）"]
        G -->|否| I["加入 CLH 队列, park 阻塞"]
        I --> J["被 unpark 唤醒后重试 tryAcquire"]
        J --> E
    ```
    - 我就记得是在某个地方多了一个判断等待队列中是否有有效的等待线程，有的话就不能获取锁，而是要去等待队列中唤醒队列中的线程来获取锁；然后我去翻看了原代码，然后找到了，在`tryAcquire()`方法中多了个对队列的判断
    - 非公平锁
    ```java
    protected final boolean tryAcquire(int acquires) {
        if (getState() == 0 && compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(Thread.currentThread());
               return true;
        }
        return false;
    }
    ```
    - 公平锁
    ```java
    protected final boolean tryAcquire(int acquires) {
        if (getState() == 0 && !hasQueuedPredecessors() &&
        compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(Thread.currentThread());
            return true;
        }
        return false;
    }
    ```
    - 非公平锁可能造成饥饿是如果每次都是刚好一个线程结束之后，刚好就有新的线程来了，就会导致在队列中的线程每次醒来都会发现锁又被占用了，又只能去睡觉，一直下去的话就会导致队列中的锁饥饿；
    - AQS会先唤醒`thread-1`后面的一个线程；
    ```mermaid
    flowchart TD
        A["线程执行release()调用tryRelease()"]
        B["判断锁持有线程是否等于当前线程"]
        C["判断是否完全释放锁（有可能是重入）"]
        CE["发生监视器状态异常"]
        D["将当前节点状态设置为无效，尝试唤醒下一个节点"]
        E["判断下一个节点是否有效"]
        EX["剔除无效节点并获取下一个节点"]
        SUC["唤醒节点"]
        F["退出"]

        A --> |"进入tryRelease()"|B
        B --> |是|C
        B --> |否|CE
        C --> |是|D
        C --> |否，继续等待递归调用释放锁|A
        D --> E
        E --> |否|EX
        EX --> |下一个节点不为null|E
        E --> |是|SUC
        EX --> |获取到null节点|F
    ```
    - 当`status!=0`的时候才可以唤醒，因为`0`一般是刚入队，处于醒的状态，不需要唤醒；

---

### Part 2

- 跟正常的AQS异曲同工，只不过是对`acquire()`函数有略微不同，它用来判断这个AQS里面的`state`是否还有许可证，如果有就放行，否则就进入`park()`；即为进入`CLH队列`中等待`unpark()`；
- 对于实现的话就是相当于`acquire(int premit)`会消耗`premit`张许可证，而`release(int premit)`会放入`premit`张许可证，这就给予了这个场景一定的操作空间；

- 实现
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

    private static final long MAX_WAIT_TIME = 2000; // default wait 2 seconds
    private static final int CORE_SIZE = 25;    // My CPU cores 32~64
    private static final int MAX_SIZE = 40;
    private static final long KEEP_ALIVE_TIME = 6000;
    private static final int MAX_TOKENS = 1000;         // token max size
    private static final int RECOVERY_NUM = 500;        // token recovery number
    private static final int MAX_TASK_SIZE = 3000;
    private static final ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(2);
    private static final ThreadPoolExecutor threadPool = new ThreadPoolExecutor(
            CORE_SIZE,
            MAX_SIZE,
            KEEP_ALIVE_TIME,
            TimeUnit.MILLISECONDS,
            new LinkedBlockingQueue<Runnable>(MAX_TASK_SIZE),
            (Runnable r, ThreadPoolExecutor executor) -> {
                try {
                    // 尝试在 0.5 秒内将任务放入队列
                    boolean success = executor.getQueue().offer(r, 500, TimeUnit.MILLISECONDS);

                    if (!success) {
                        // 如果等待后依然失败，抛出拒绝执行异常
                        throw new RejectedExecutionException("Task rejected after waiting 1 second for queue space.");
                    }
                } catch (InterruptedException e) {
                    // 如果等待过程中线程被中断，恢复中断状态并抛出异常
                    Thread.currentThread().interrupt();
                    throw new RejectedExecutionException("Task interrupted while waiting for queue space.", e);
                }
            }
    );

    private static final Semaphore tokenPool = new Semaphore(MAX_TOKENS);

    @Getter
    private static long successRecoveryCount = 0;

    static {
        scheduler.scheduleAtFixedRate(() -> {
            int recoveryNum = Math.min(MAX_TOKENS - tokenPool.availablePermits(), RECOVERY_NUM);
            tokenPool.release(recoveryNum);
            successRecoveryCount += recoveryNum;
        }, 0, 1, TimeUnit.SECONDS);
    }


    public static String tryGetToken() {
        Future<String> result = threadPool.submit(() -> {
            long begin = System.currentTimeMillis();
            long passTime;
            while (!tokenPool.tryAcquire(1)) {
                passTime = System.currentTimeMillis() - begin;
                if (passTime > MAX_WAIT_TIME) {
                    return null;    // timeout, return null
                }
            }
            return JwtTokenGenerator.generateToken();
        });

        try {
            return result.get();
        } catch (InterruptedException |  ExecutionException e) {
            return null;
        }
    }
}
```
- 测压数据：
```txt
========== 开始热身 (Warm-up) ==========
所有线程就绪，开始压测...
测试持续时间: 3.06 秒
并发线程数: 100
总请求数: 2100
----------------------------------------
成功获取 Token 数: 2100
未获取到 (返回 null) 数: 0
异常崩溃数: 0
----------------------------------------
总 QPS (吞吐量): 685.83 req/s
成功 QPS: 685.83 req/s
错误率: 0.00%
期间内后台令牌桶恢复成功的量：1500

========== 开始第一轮压力测试 (Stress Test) ==========
所有线程就绪，开始压测...
测试持续时间: 11.99 秒
并发线程数: 1000
总请求数: 6400
----------------------------------------
成功获取 Token 数: 6400
未获取到 (返回 null) 数: 0
异常崩溃数: 0
----------------------------------------
总 QPS (吞吐量): 533.64 req/s
成功 QPS: 533.64 req/s
错误率: 0.00%
期间内后台令牌桶恢复成功的量：6000

========== 开始高压力测试 (Stress Test) ==========
所有线程就绪，开始压测...
测试持续时间: 27.00 秒
并发线程数: 4000
总请求数: 47097
----------------------------------------
成功获取 Token 数: 13040
未获取到 (返回 null) 数: 0
异常崩溃数: 34057
----------------------------------------
总 QPS (吞吐量): 1744.07 req/s
成功 QPS: 482.89 req/s
错误率: 72.31%
期间内后台令牌桶恢复成功的量：13500
```
> 在达到4000个线程之后，QPS开始骤降到`482.89 req/s`，但是整体还处于可以接收的范围内
- 更安全是因为`Semaphore`中是按量唤醒，随着队列链表逐个唤醒满足条件的线程，而不是我那种自制的“惊群效应”；

- 其实加入了类似于AQS的唤醒机制和防溢出问题防护，会将需要加入的许可证数量放入到`status`中，然后判断`status`是否溢出，没有就返回`true`，然后就可以去尝试唤醒等待队列中的节点；