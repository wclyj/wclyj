# 线程池：三个会在生产里爆掉的用法

创建线程要分配栈、登记到虚拟机，成本不低。池化的思路是事先准备一批工人，任务来了领一个，做完还回去，再按闲置情况增减人数。短任务应交给池，而不是来一次请求就 `new Thread`。

线程池却也是业务代码里最容易埋雷的组件。下面按三起事故的顺序看：参数没露出来、池没有复用、不同性质的任务挤在同一个池里。

## 不要用快捷工厂把参数藏起来

`Executors` 准备了几家工厂，一行就能拿到池。《阿里巴巴 Java 开发手册》要求直接 `new ThreadPoolExecutor`，把核心线程数、最大线程数、存活时间、队列、拒绝策略写在调用点。这条不是风格偏好，`newFixedThreadPool` 和 `newCachedThreadPool` 都曾经把进程打成 OOM。

### 固定大小的池，队列却是无界的

下面的接口拿一个单线程的固定池，循环提交大量任务。每个任务先拼出一段很长的字符串，再睡 30 分钟，用来模拟“提交飞快、执行极慢”。

```java
@GetMapping("pool/fixed-oom")
public void fixedOom() throws InterruptedException {
    ThreadPoolExecutor pool = (ThreadPoolExecutor) Executors.newFixedThreadPool(1);
    watch(pool);
    for (int n = 0; n < 5_000_000; n++) {
        pool.execute(() -> {
            String payload = "x".repeat(200_000) + UUID.randomUUID();
            try {
                TimeUnit.MINUTES.sleep(30);
            } catch (InterruptedException ex) {
                Thread.currentThread().interrupt();
            }
            log.info("len={}", payload.length());
        });
    }
    pool.shutdown();
    pool.awaitTermination(30, TimeUnit.MINUTES);
}
```

跑不了多久，常见的失败是 `java.lang.OutOfMemoryError: GC overhead limit exceeded`。工人只有一个，后面的任务全部堆在队列里，每个任务还抓着一块大字符串，垃圾回收一直在救，堆还是被占满。

看工厂实现就明白了：工作队列是无参的 `LinkedBlockingQueue`，容量默认 `Integer.MAX_VALUE`，可以当成没有上界。

等价展开是：核心线程数和最大线程数都等于传入的工人数，存活时间写 0 毫秒，队列用无界的 `LinkedBlockingQueue`。把这四个槽位摊开看，线程数虽然有顶，任务队列没有顶。

线程数被钉死，只解决了“工人太多”。任务来得快、做得慢时，排队的 `Runnable` 会先把内存吃光。

### 可缓存的池，线程数没有上界

把工厂换成 `newCachedThreadPool`，不久会看到另一种 OOM：`unable to create new native thread`。这次不是堆里的对象太多，而是操作系统不肯再给新线程分配栈。

等价展开是：核心线程数为 0，最大线程数用 `Integer.MAX_VALUE`，空闲 60 秒回收，队列是不存元素的 `SynchronousQueue`。没有空闲工人时，新任务只能再拉一条线程。

最大线程数同样是 `Integer.MAX_VALUE`。配套的 `SynchronousQueue` 不存任务，一次 `offer` 必须立刻有工人来 `take`。没有空闲工人就再拉起一条线程。上面的任务要睡 30 分钟，新请求又不断进来，线程数会跟着请求数涨。一条线程栈即便按 1MB 估，几千条就已经是数 GB 的本地内存。

多数人不是不知道这两家工厂的脾气，而是觉得“我这点轻任务不要紧”。一次注册链路把这种侥幸打掉过：注册成功后要调邮件网关，平时 80 毫秒返回，每秒大约 80 次注册，可缓存池稳定在十几条线程。某天网关卡住，客户端读超时却配成 45 秒。45 秒内大约涌进 3600 封待发邮件，池就尝试拉起 3600 条线程，紧接着原生线程创建失败，进程退出。

因此快捷工厂不适合作为业务池：

- 核心线程数、最大线程数、空闲回收、队列种类、拒绝策略都要按峰值评估。队列要有界，线程数要有顶。
- 线程名字要能看懂。抓栈时如果全是 `pool-3-thread-9`，无法判断是发信、对账还是图片压缩。用线程工厂把名字设成 `mail-sender-%d` 这种格式。
- 池很能忍。除了拒绝策略抛异常，队列再长、线程再多也往往不报错。要有周期日志或指标，看见排队长度、活跃线程往上冲时就处理，而不是等 OOM。

## 默认的接纳顺序

监控可以先做成每秒打一行。注意这个调度器自己也是池，应当随进程关闭，不要在每个请求里新开一个。

```java
private final ScheduledExecutorService watcher =
        Executors.newSingleThreadScheduledExecutor(r -> {
            Thread t = new Thread(r, "pool-watch");
            t.setDaemon(true);
            return t;
        });

private void watch(ThreadPoolExecutor pool) {
    watcher.scheduleAtFixedRate(() -> log.info(
            "size={} active={} done={} queued={}",
            pool.getPoolSize(),
            pool.getActiveCount(),
            pool.getCompletedTaskCount(),
            pool.getQueue().size()),
            0, 1, TimeUnit.SECONDS);
}
```

用一组故意偏小的参数观察接纳顺序：核心 3、最大 6、空闲 4 秒回收、容量 8 的 `ArrayBlockingQueue`、默认 `AbortPolicy`。线程名通过工厂加上 `demo-worker-%d`。测试每隔 1 秒提交一个任务，共 16 个，每个任务睡 6 秒。

```java
@GetMapping("pool/policy")
public int policy() throws InterruptedException {
    AtomicInteger seq = new AtomicInteger();
    ThreadPoolExecutor pool = new ThreadPoolExecutor(
            3, 6,
            4, TimeUnit.SECONDS,
            new ArrayBlockingQueue<>(8),
            new ThreadFactoryBuilder().setNameFormat("demo-worker-%d").get(),
            new ThreadPoolExecutor.AbortPolicy() /* 满则抛出拒绝异常 */);
    watch(pool);
    for (int n = 0; n < 16; n++) {
        TimeUnit.SECONDS.sleep(1L);
        int id = seq.incrementAndGet();
        try {
            pool.submit(() -> {
                log.info("job {} begin", id);
                try {
                    TimeUnit.SECONDS.sleep(6);
                } catch (InterruptedException ex) {
                    Thread.currentThread().interrupt();
                }
                log.info("job {} end", id);
            });
        } catch (RejectedExecutionException ex) {
            log.error("job {} rejected", id);
            seq.decrementAndGet();
        }
    }
    TimeUnit.SECONDS.sleep(40);
    return seq.get();
}
```

返回值会小于 16，日志里出现 `RejectedExecutionException`。把每秒的 size、active、queued 画出来，能对上下面这条默认顺序。早期任务还没做完时，拒绝发生在“队列已满并且线程数已经到最大”的那个瞬间；已经开始执行的任务到期后会从队列里拉新工作，所以不能用“提交次数减去容量”这种静态减法去猜拒绝时刻，要以监控为准。

默认行为可以记成五条：

- 创建池时不会立刻拉起 `corePoolSize` 条线程，有任务才建。
- 核心线程满了，优先把任务放进队列，而不是马上扩容。
- 队列也满了，才把线程数往 `maximumPoolSize` 加。
- 线程数到顶且队列仍满，走拒绝策略。
- 线程数高于核心、并且空闲超过 `keepAliveTime`，多余线程退出。

容量规划就是在给这五条填数字。还有两个开关经常被忽略：

- `prestartAllCoreThreads()` 可以在接流量之前把核心线程建好，避免冷启动抖动。
- `allowCoreThreadTimeOut(true)` 让核心线程在空闲时也能退出，适合流量忽高忽低、平时希望把栈内存还回去的服务。

还有一个和直觉相反的地方：池是先排队、再扩容。队列如果设成几千甚至上万，最大线程数几乎碰不到，等队列终于满了再加人，慢任务已经堆积太久。任务本身要跑数秒时，更合理的是先把工人加到上限，队列只做短缓冲。

改法的思路是骗过入队，而不是改 `ThreadPoolExecutor` 的状态机。执行器在 `offer` 失败时才会尝试 `addWorker`。于是可以包装一个有界队列：当前线程数还没到最大时，`offer` 直接返回 `false`，迫使执行器去加人；人已经加满，再真正入队。加人失败后会进入拒绝策略，此时自定义的拒绝处理器把任务放回队列（必要时阻塞），这样队列才是后备，而不是第一道闸门。

```java
static final class ExpandFirstQueue<E> extends ArrayBlockingQueue<E> {
    private ThreadPoolExecutor owner;

    ExpandFirstQueue(int capacity) {
        super(capacity);
    }

    void bind(ThreadPoolExecutor owner) {
        this.owner = owner;
    }

    @Override
    public boolean offer(E task) {
        if (owner.getPoolSize() < owner.getMaximumPoolSize()) {
            return false;
        }
        return super.offer(task);
    }
}
```

```java
static final class EnqueueWhenFull implements RejectedExecutionHandler {
    @Override
    public void rejectedExecution(Runnable task, ThreadPoolExecutor ex) {
        if (ex.isShutdown()) {
            throw new RejectedExecutionException("pool closed");
        }
        try {
            ex.getQueue().put(task);
        } catch (InterruptedException ex1) {
            Thread.currentThread().interrupt();
            throw new RejectedExecutionException("enqueue interrupted", ex1);
        }
    }
}
```

这里有竞态：`offer` 返回 `false` 的瞬间，别的线程可能刚好把工人建满，于是走到拒绝逻辑再入队。教学代码能表达“先扩容、后排队”，生产上要参考 Tomcat 那套线程池对 `TaskQueue` 的处理，把执行器是否还在接受任务、当前工人数是否仍可增加判断完整。不要在每个请求里 new 这种池。

## 确认拿到的是同一份池

另一起报警是线程数偶尔冲过 2000，几分钟后又掉下来，访问量并没有同比例变化。线程数高的时候抓栈，发现一千多个名字各异的池，而不是固定的那几份。业务侧只有类似下面的调用，看不出每请求一个池：

```java
@GetMapping("pool/helper")
public String viaHelper() {
    ThreadPoolExecutor pool = WorkerPools.borrow();
    for (int n = 0; n < 10; n++) {
        pool.execute(() -> {
            try {
                TimeUnit.SECONDS.sleep(1L);
            } catch (InterruptedException ex) {
                Thread.currentThread().interrupt();
            }
        });
    }
    return "OK";
}
```

问题在工具类里：每次 `borrow` 都 `Executors.newCachedThreadPool()`。

```java
final class WorkerPools {
    static ThreadPoolExecutor borrow() {
        ExecutorService fresh = Executors.newCachedThreadPool();
        return (ThreadPoolExecutor) fresh;
    }
}
```

一次请求提交 10 个慢任务，池就会拉起大约 10 条线程。并发一高，几千条线程并不奇怪。线程数后来能降下来，是因为这家工厂的核心线程数是 0，空闲 60 秒就回收，所以它“爆了又缩回去”，看起来像抖动，而不是立刻把机器打死。

修复是把池放进静态字段，并且按前面的要求手写参数、写上名字。调用方只取这一份。

```java
final class WorkerPools {
    private static final ThreadPoolExecutor SHARED = new ThreadPoolExecutor(
            8, 32,
            3, TimeUnit.SECONDS,
            new ArrayBlockingQueue<>(500),
            new ThreadFactoryBuilder().setNameFormat("mail-worker-%d").get(),
            new ThreadPoolExecutor.AbortPolicy() /* 满则抛出拒绝异常 */);

    static ThreadPoolExecutor borrow() {
        return SHARED;
    }
}
```

如果池不是自己 new 的，而是某个库“借给你”的，一定要打开实现看：是返回单例，还是每次制造一个 `newCachedThreadPool`。

## 复用不等于所有任务挤一个池

池的意义是复用工人，不是全进程只留一个池。参数要跟着任务的轻重来：

- 慢、量不大的 IO（外呼、写盘、等邮件网关）需要更多线程，队列宜短，否则调用方以为已经异步，其实在长队列里干等。
- 吃 CPU 的计算，线程数接近核数或两倍核数就够了。多出来的线程只是增加切换，吞吐不会上升。这种任务可以用较长的队列做缓冲。

曾经有一段内存里的汇总，理论上不碰磁盘，监控里却要数秒才返回，CPU 也不忙。追下去发现，它和后台的文件批处理共用一个池。那个池只有 2 条线程，队列容量 80，拒绝策略是 `CallerRunsPolicy`：

```java
private static final ThreadPoolExecutor BATCH = new ThreadPoolExecutor(
        2, 2,
        60, TimeUnit.MINUTES,
        new ArrayBlockingQueue<>(80),
        new ThreadFactoryBuilder().setNameFormat("ledger-batch-%d").get(),
        new ThreadPoolExecutor.CallerRunsPolicy() /* 满载时退回调用方线程 */);
```

启动后一条后台线程不停往池里塞写文件任务，每次把很长的一段文本写进同一个路径：

```java
@PostConstruct
void flood() {
    watch(BATCH);
    Thread feeder = new Thread(() -> {
        String block = "a".repeat(500_000);
        while (!Thread.currentThread().isInterrupted()) {
            BATCH.execute(() -> {
                try {
                    Files.writeString(
                            Path.of("ledger-batch.txt"),
                            LocalTime.now() + ":" + block,
                            StandardOpenOption.CREATE,
                            StandardOpenOption.TRUNCATE_EXISTING);
                } catch (IOException ex) {
                    log.error("write failed", ex);
                }
            });
        }
    }, "ledger-feeder");
    feeder.setDaemon(true);
    feeder.start();
}
```

监控上会看到 2 条线程一直活跃，队列长期顶满。`CallerRunsPolicy` 的特殊之处在于：池和队列都满时，任务改在调用 `execute` 的那条线程里同步跑。喂任务的线程自己被写文件拖住，所以提交速度会降下来，任务也不会被丢掉。选择这个策略的人多半是测试时看见拒绝异常，又不想丢数据。代价是“提交”不再意味着“异步”，调用方的线程被借走了。

业务接口如果把一个只睡 15 毫秒的汇总也丢进 `BATCH`，请求线程要先等文件任务让出工人。池饱和时，`submit().get()` 甚至会在 Tomcat 的请求线程上直接把这 15 毫秒跑掉，连不相关的同步接口一起拖慢。

```java
private Callable<Integer> tally() {
    return () -> {
        TimeUnit.MILLISECONDS.sleep(15);
        return 1;
    };
}

@GetMapping("pool/mixed")
public int mixed() throws Exception {
    return BATCH.submit(tally()).get();
}
```

拆出一个只做短计算的池。睡 15 毫秒更接近等待而不是纯 CPU，线程数可以按并发目标来定，而不是按核数：

```java
private static final ThreadPoolExecutor TALLY = new ThreadPoolExecutor(
        120, 120,
        60, TimeUnit.MINUTES,
        new ArrayBlockingQueue<>(800),
        new ThreadFactoryBuilder().setNameFormat("tally-%d").get(),
        new ThreadPoolExecutor.AbortPolicy() /* 计算池满则拒绝 */);

@GetMapping("pool/isolated")
public int isolated() throws Exception {
    return TALLY.submit(tally()).get();
}
```

隔离之后，文件批处理再饱和，也不会把汇总请求赶到 Tomcat 线程上。这和用虚拟机把进程隔开是同一类想法：资源配额按工作负载分，而不是所有人抢同一台物理机。

顺带一个容易忽略的共享池：Java 8 的 `parallelStream()` 用的是整个进程一份 `ForkJoinPool.commonPool()`，默认并行度是可用核数减 1。纯 CPU 的集合运算用它合适。如果 lambda 里要查库、调远程接口，这些工人会被 IO 占住，别的并行流一起变慢。这种情况应自建 `ForkJoinPool`（或普通池），在自己的池里跑，不要碰公共池。

## 回顾

- 快捷工厂把队列上界和线程上界藏起来了。固定池可能因无界队列撑爆堆，可缓存池可能因无限建线程耗尽本地栈。自己 new 池，写上有界队列、有限线程、有意义的名字，并盯着队列长度和活跃数。
- 池必须是被复用的那一份。每次 new 一个出来，成本高于直接 new 线程：既没有复用，又可能留下回收周期长达一分钟的空闲线程。从工具类拿池时要看它返回的是不是静态实例。
- 复用的前提是任务性质相近。IO 慢任务和 CPU 短任务分开；`CallerRunsPolicy` 会把异步变回调用方同步执行，可能污染 Web 容器线程。`parallelStream` 也是一份共享池，里面不要做阻塞 IO。

线程池在进程内部，往往没有人像盯消息队列那样盯它。等到拒绝、OOM 或线程数报警，事故已经发生。把 size、active、queue、rejected 接到指标上，比事后抓栈有效。
