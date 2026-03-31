> [!warning] 使用`LongAdder`作为限流器
> 除非有办法限制在高并发那一秒钟不会进入过大的超量
> 否则无法使用这个作为限流器

```java
import io.jsonwebtoken.Jwts;  
import io.jsonwebtoken.SignatureAlgorithm;  
import lombok.Getter;  
import lombok.extern.slf4j.Slf4j;  
  
import java.util.*;  
import java.util.concurrent.*;  
import java.util.concurrent.atomic.LongAdder;  
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
  
    private static final int CORE_SIZE = 25;    // My CPU cores 32~64  
    private static final int MAX_SIZE = 40;  
    private static final long KEEP_ALIVE_TIME = 6000;  
    private static final long RECOVERY_NUM = 500;        // token recovery number  
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
  
    private static final long MAX_TOKEN = 1000;  
    private static final LongAdder tokenPool = new LongAdder();  
    private static final LinkedBlockingQueue<Thread> parkedThreads = new LinkedBlockingQueue<>();  
  
    @Getter  
    private static long successRecoveryCount = 0;  
  
    static {  
        tokenPool.reset();  
        tokenPool.add(MAX_TOKEN);  
        scheduler.scheduleAtFixedRate(() -> {  
            long recoveryNum = Math.min(MAX_TOKEN - tokenPool.sum(), RECOVERY_NUM);  
            tokenPool.add(recoveryNum);  
            successRecoveryCount += recoveryNum;  
  
            ArrayList<Thread> threads = new ArrayList<>(parkedThreads);  
            threads.forEach(LockSupport::unpark);  
            parkedThreads.removeAll(threads);  
        }, 0, 1, TimeUnit.SECONDS);  
    }  
  
  
    public static String tryGetToken() {  
        Future<String> result = threadPool.submit(() -> {  
            Thread cur = Thread.currentThread();  
            isNeedToPark(cur);  
  
            do {  
                if (tokenPool.sum() > 0) {  
                    tokenPool.decrement();  
                    return JwtTokenGenerator.generateToken();  
                }  
                isNeedToPark(cur);  
            } while (true);  
        });  
  
        try {  
            return result.get();  
        } catch (InterruptedException |  ExecutionException e) {  
            return null;  
        }  
    }  
  
    private static void isNeedToPark(Thread cur) {  
        if (tokenPool.sum() <= 0) {  
            parkedThreads.add(cur);  
            LockSupport.park();  
        }  
    }  
}
```