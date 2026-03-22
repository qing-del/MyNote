
## 目录
- [[#代码正文]]
- [[#可改进之处]]

---

### 代码正文
> [!info] 背景
> 用户会尝试向邮件公司申请一个验证码，然后开始等待邮件派送（随机生成用户耐心值 -- **模拟超时**）
> 邮件公司收到请求之后就将任务丢到消息队列里等快递员去取件和派送（==**消费者生产者模式**==）
> 快递员派件之后会通知所有正在等待中的用户，”有新的邮件到邮件箱了“
> 用户就会尝试去邮箱里查看是不是自己的邮件送到了（**==暂停性保护 -- 一一对应关系==**）
> 
> > 是一个关于暂停性保护和消费者生产者模式的实践`Demo`

```java
import lombok.Getter;  
import lombok.Setter;  
import lombok.extern.slf4j.Slf4j;  
  
import java.util.*;  
  
@Slf4j(topic = "d.mailCompany")  
public class MailCompany {  
    // 自增id  
    private static int id = 1;  
    // 申请获取验证码且需要被邮寄给用户的id列表  
    private static final LinkedList<Integer> idsList = new LinkedList<>();  
  
    // 随机数生成器  
    protected final static Random rd = new Random();  
  
    private synchronized static int generateId() {  
        return id++;  
    }  
  
    // 处理请求列表逻辑  
    private static final Runnable requestHandler = () -> {  
        while (true) {  
	        // 锁加在 while 外面可以防止线程安全
            synchronized (idsList) {
                while (idsList.isEmpty()) {  
                    try {idsList.wait();} catch (InterruptedException e) {e.printStackTrace();}  
                }  
            }  
  
            int id;  
            synchronized (idsList) {  
                id = idsList.removeFirst(); // 从队列中获取 id            }  
            Mail mail = new Mail(id);  
            String response = "id为" + id + "的验证码：" + String.format("%06d" ,(int)(rd.nextDouble() * 1000000));  
            mail.setResponse(response);  
            Thread postman = new Postman(id, mail);  
            postman.setName("postman_" + id);  
            postman.start();  
            log.info("已处理id为 {} 的请求", id);  
        }  
    };  
  
    /**  
     * 申请获取验证码  
     * @return 申请的id  
     */    public static int apply() {  
        int id = generateId();  
        synchronized (idsList) {  
            idsList.addLast(id);  
            idsList.notifyAll();  
        }  
        return id;  
    }  
  
    public static void main(String[] args)  
    throws InterruptedException {  
        log.info("开始启动邮件公司");  
        new Thread(requestHandler, "请求处理线程").start();  
        log.info("已开启请求处理程序");  
  
        // 以下是模拟申请的用户数量  
        int userNumber = rd.nextInt(90) + 10;  
        log.info("已生成 {} 个用户", userNumber);  
        ArrayList<Thread> threads = new ArrayList<>();  
        for (int i = 0; i < userNumber; i++) {  
            threads.add(new People("p" + i));  
        }  
        threads.forEach(Thread::start);  
        threads.forEach(t -> {  
            try {t.join();} catch (InterruptedException e) {throw new RuntimeException(e);}  
        });  
  
  
        System.exit(0);  
    }  
}  
  
@Slf4j(topic = "d.people")  
class People extends Thread {  
    public People(String name) {  
        super(name);  
    }  
  
    @Override  
    public void run() {  
        Mail mail;  
        while(true) {  
            // 申请获取验证码  
            int id = MailCompany.apply();  
            log.info("已申请id为 {} 的验证码", id);  
            int patience = MailCompany.rd.nextInt(5000) + 2000;  
            mail = MailBox.get(id, patience);  
            if (mail != null) {  
                if (id != mail.getId()) {  
                    log.error("用户表示收到不是自己的验证码！");  
                    continue;  
                }  
  
                log.info("已收到id为 {} 的验证码：{}", id, mail.get());  
                break;  
            } else {  
                log.info("用户表示并未获取到 id 为 {} 的验证码！", id);  
            }  
  
            log.warn("用户再次尝试申请验证码！");  
        }  
    }  
}  
  
@Slf4j(topic = "d.postman")  
class Postman extends Thread {  
    private int id;  
    private Mail mail;  
  
    public Postman(int id, Mail mail) {  
        this.id = id;  
        this.mail = mail;  
    }  
  
    @Override  
    public void run() {  
        log.info("已开始处理 id 为 {} 的邮件", id);  
        MailBox.put(mail);  
        synchronized (MailBox.class) {  
            MailBox.class.notifyAll();  // 快递员会通知所有在等待邮件的用户，他放了一个邮件进入邮箱  
        }  
    }  
}  
  
@Slf4j(topic = "d.mailBox")  
final class MailBox {  
    // 信箱 -- （因为锁了static，同一时间内只能有同一个线程进行放入或者是删除操作）  
    public final static HashMap<Integer, Mail> mailBox = new HashMap<>();  
  
    public synchronized static Mail get(int id, int timeout) {  
        if (id < 0 || timeout < 0) {  
            log.error("用户尝试使用错误 id 来获取邮件，已返回 null！");  
            return null;  
        }  
  
        long begin = System.currentTimeMillis();  
        long passTime = 0;  
        while (mailBox.get(id)==null) {  
            long waitTime = timeout - passTime;  
            if (waitTime <= 0) break;  
            try { MailBox.class.wait(waitTime); } catch (InterruptedException e) { e.fillInStackTrace(); }  
            passTime = System.currentTimeMillis() - begin;  
        }  
  
        if (mailBox.get(id) != null) {  
            return mailBox.remove(id);  
        } else {  
            return null;  
        }  
    }  
  
    public synchronized static void put(Mail mail) {  
        mailBox.put(mail.getId(), mail);  
        MailBox.class.notifyAll();  
    }  
}  
  
final class Mail {  
    @Getter  
    private final int id;  
  
    @Setter  
    private Object response;  
  
    public Mail (int id) {  
        this.id = id;  
    }  
  
    /**  
     * 获取结果  
     * 默认等待 3s  
     * @return 响应结果  
     */  
    public Object get() {  
        return get(3000);  
    }  
  
    /**  
     * 获取结果  
     * @param timeout 等待时长  
     * @return 响应结果  
     */  
    public synchronized Object get(long timeout) {  
        if (timeout <= 0 || response != null) {  
            return response;  
        }  
  
        long begin = System.currentTimeMillis();  
        long passTime = 0;  
        while (response == null) {  
            long waitTime = timeout - passTime;  
  
            if (waitTime <= 0) break;   // 超时退出等待  
            try {this.wait(waitTime);} catch (InterruptedException e) {e.fillInStackTrace();}  
            passTime = System.currentTimeMillis() - begin;  
        }  
        return response;  
    }  
}
```

---

### 可改进之处

### 🚨 代码中的潜在危险与错误

#### 1. 致命的“锁拆分”导致原子性失效 (在 `requestHandler` 中)
在你的消费者线程 `requestHandler` 中，你将等待逻辑和获取逻辑拆分成了两个 `synchronized` 块：
```java
synchronized (idsList) {
    while (idsList.isEmpty()) {
        try {idsList.wait();} catch (InterruptedException e) {e.printStackTrace();}
    }
}
// ⚠️ 危险地带：在这里锁被释放了！
int id;
synchronized (idsList) {
    id = idsList.removeFirst(); // 从队列中获取 id
}
```
**危险所在**：虽然你当前只有一个 `requestHandler` 线程，这段代码能正常运行。但如果未来你引入了线程池，有**多个**处理线程同时运行，当线程 A 退出第一个 `synchronized` 块时，线程 B 可能会抢占 CPU 并且取走 `idsList` 中的最后一个元素。接着线程 A 进入第二个 `synchronized` 块执行 `removeFirst()`，就会直接抛出 `NoSuchElementException`。
**修复方案**：必须保证 `wait()` 唤醒后的条件判断和消费操作在同一个锁内部。
```java
int id;
synchronized (idsList) {
    while (idsList.isEmpty()) {
        try {idsList.wait();} catch (InterruptedException e) {e.printStackTrace();}
    }
    id = idsList.removeFirst(); 
}
```

> [!warning] 警告
> `synchronized`要加在`while`外面并将其包住，不然以后多线程的时候
> 如果队列里面有值，线程A刚跑到`while`之后，线程B拿走了CPU使用权，然后将队列里面最后一个值拿走了
> 这个时候就会出现线程安全问题了

#### 2. “孤立”的保护性暂停 (在 `Mail` 类中)
你在 `Mail` 类中写了一个非常标准的带有超时的 `get(long timeout)` 方法，里面包含了 `this.wait(waitTime)`。但是，纵观全局代码，**没有任何地方调用过 `mail.notifyAll()` 或 `mail.notify()`**。
**为什么现在没报错？** 因为在 `requestHandler` 中，你是在生成验证码后，直接 `mail.setResponse(response)`，**然后**才交给 `Postman` 送进邮箱。用户从邮箱拿到 `Mail` 时，`response` 已经不是 `null` 了，所以直接 return，根本没有进入 `wait()` 循环。
**设计瑕疵**：如果 `Mail` 对象真的充当一个类似 `Future` 的凭据（比如你先给用户一个空凭据，让用户自己去 `wait`，后台慢慢生成内容），那么 `setResponse` 方法内部必须加上 `synchronized(this)` 和 `this.notifyAll()`。

#### 3. 冗余的通知逻辑 (在 `Postman` 中)
在快递员线程中：
```java
MailBox.put(mail);
synchronized (MailBox.class) {
    MailBox.class.notifyAll();  // 快递员会通知所有在等待邮件的用户
}
```
你的 `MailBox.put()` 已经是一个 `synchronized` 方法，并且里面已经调用了 `MailBox.class.notifyAll();`。这里在外面再锁一次并 notify 纯属脱裤子放屁（性能损耗），直接删掉 `synchronized` 块即可。

---