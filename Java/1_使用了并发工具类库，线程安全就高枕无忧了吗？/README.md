# 并发容器只担保它自己声明过的操作

评审里常听到两种省事结论：把普通哈希表换成并发哈希表，竞争就消失了；或者换成写时复制列表，因为听起来不加锁，吞吐一定更好。两种说法都把工具的承诺范围夸大了。

语言提供同步器，也提供能同时读写的容器，是为了少让业务自己用底层原子指令拼结构。容器只保证文档里写明的那些调用。没弄清场景、复合步骤和线程从哪来，轻则白付复制和加锁的成本，重则多线程下的业务结果是错的。业务里容器比同步器更常见，下面四类坑也以容器为主。

## 工作线程被复用后，槽里会留下上一单的身份

线上出现过这种错乱：偶发读到的操作者属于另一个人。实现用线程局部变量缓存本次请求查到的身份。这个工具的契约是，同一条线程上的各个方法共享这份数据，别的线程看不见。身份查询若很贵，放进槽里避免重复读库是合理的。串号的原因不是槽没有隔离，而是这条线程下一回还会派给谁。

用一个接口把现象固定下来。槽里放长整型编号，一开始没有值。进入方法先读一次，再写入参数，然后再读一次，连同线程名返回。

```java
private static final ThreadLocal<Long> operatorSlot = new ThreadLocal<>();

@GetMapping("/audit/raw")
public Map<String, String> raw(@RequestParam("operatorId") Long operatorId) {
    String earlier = Thread.currentThread().getName() + " -> " + operatorSlot.get();
    operatorSlot.set(operatorId);
    String later = Thread.currentThread().getName() + " -> " + operatorSlot.get();
    Map<String, String> body = new LinkedHashMap<>();
    body.put("earlier", earlier);
    body.put("later", later);
    return body;
}
```

若每次都是新线程，写入前应当一直是空。Servlet 容器不是这么调度的。请求跑在工作线程上，工作线程来自池：处理完回到池里，等待下一个请求。线程没销毁，槽里的引用也不会跟着响应一起消失。下一个请求只要在写入前先读，拿到的就是上一单留下的编号。

把池收成一条线程，交错就稳定了。Spring Boot 可以限制：

```properties
server.tomcat.threads.max=1
```

先用编号 11 调用，两次读数是空和 11。再用编号 22 调用，线程名仍是同一个 `http-nio-*-exec-*`，两次读数变成 11 和 22。第二次请求在写入前看见了第一次的残留。

写业务时据此做两个判断：

- 方法里没有自己 `new Thread`，不代表它跑在单线程环境。容器为了同时接住大量请求，本来就把每个请求丢进池。接口能并发，是因为池里有多条线程，不是因为业务代码显式开了线程。
- 创建线程昂贵，所以服务器和各种客户端都用池。凡是把状态绑在线程上，都要在这次任务的终点把槽清掉。自己 `Executors` 出来的池一样，线程回到池中时槽还在。只 `set(null)` 不如 `remove`：前者留下键已经空掉的条目，值暂时断了，条目还占着表。

修正是放进 `finally`。即便中间返回，清理仍会执行：

```java
@GetMapping("/audit/clean")
public Map<String, String> clean(@RequestParam("operatorId") Long operatorId) {
    String earlier = Thread.currentThread().getName() + " -> " + operatorSlot.get();
    operatorSlot.set(operatorId);
    try {
        String later = Thread.currentThread().getName() + " -> " + operatorSlot.get();
        Map<String, String> body = new LinkedHashMap<>();
        body.put("earlier", earlier);
        body.put("later", later);
        return body;
    } finally {
        operatorSlot.remove();
    }
}
```

真实工程里，清理点通常放在过滤器或拦截器的 `finally`，而不是散落在每个业务方法里。放进异步线程或响应式切换之后，槽不会自动跟着跳过去，那是另一条线程。

过滤器可以把清理收成一处。业务方法只管写入，出口无论成功、业务异常还是容器错误，都会走到移除：

```java
@Component
public class OperatorSlotFilter extends OncePerRequestFilter {
    static final ThreadLocal<Long> operatorSlot = new ThreadLocal<>();

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain chain)
            throws ServletException, IOException {
        String raw = request.getHeader("X-Operator");
        if (raw != null && !raw.isEmpty()) {
            operatorSlot.set(Long.valueOf(raw));
        }
        try {
            chain.doFilter(request, response);
        } finally {
            operatorSlot.remove();
        }
    }
}
```

只在正常返回时清理是不够的。中途抛错若跳过清理，这条工作线程回到池里时仍带着编号，下一单在写入前读到的还是上一单。`set(null)` 也弱于 `remove`：值虽然没了，表里仍留着键已经空掉的条目，线程长期复用时这些空条目要靠后续探测才能收掉。

子线程不会自动看见父线程的槽。需要把任务提交时的上下文拷过去，要么在提交处显式 `get` 再在子线程 `set`，要么使用可继承的那种子类，并确认池里的工作线程是在提交之后新创建的。池里已经存在的线程不会重新继承。复制出去的那份同样要在子任务的 `finally` 里移除，否则池化的子线程也会串。

线程局部变量靠“每条线程一份”避开共享。数据必须被多条线程同时看见时，才轮到并发容器。容器安全，仍不等于你拼出来的一段业务安全。

## 单次调用安全，盖不住先观察再修改

`ConcurrentHashMap` 保证自己的单次 `get`、`put` 不会把内部桶写坏，一部分方法还把“没有就创建、有就改”收成一次原子更新。它不保证你在外面连续做的两步之间，表保持静止。

误用长这样：表里已有 800 条，目标 1000 条，8 个任务一起补。每个任务先 `size()` 算缺口，再循环放入那么多新键。作者认为换了并发表就可以不加锁。

```java
static final int WORKERS = 8;
static final int TARGET = 1000;

@GetMapping("/catalog/race")
public int race() throws Exception {
    ConcurrentHashMap<String, Long> catalog = new ConcurrentHashMap<>();
    for (int i = 0; i < TARGET - 200; i++) {
        catalog.put("seed-" + i, 1L);
    }
    ExecutorService pool = Executors.newFixedThreadPool(WORKERS);
    List<Future<?>> tasks = new ArrayList<>();
    for (int n = 0; n < WORKERS; n++) {
        tasks.add(pool.submit(() -> {
            int missing = TARGET - catalog.size();
            log.info("{} missing={}", Thread.currentThread().getName(), missing);
            for (int i = 0; i < missing; i++) {
                catalog.put(UUID.randomUUID().toString(), 1L);
            }
        }));
    }
    for (Future<?> task : tasks) {
        task.get();
    }
    pool.shutdown();
    log.info("final={}", catalog.size());
    return catalog.size();
}
```

一次实际交错里，8 个任务都打印 `missing=200`，结束大小是 2400。八个人在任何人插入之前读到了同一张快照。若先把缺口那么多的条目组装好再 `putAll`，插入过程更慢，中途再读 `size()` 还会看到零头，甚至看到已经超过目标之后的负数缺口。

可以想成货架上已有 800 件，目标 1000，8 个理货员各自看一眼差额再搬。货架允许同时放入，内部链表不会被写花；但它不会在甲正在看差额时把数量冻结给乙。“搬入 200 件”又是一个循环，外人能看见 800 和 1000 之间的每一个中间值。

边界就这三条：

- 多次调用之间不存在“这段时间没有别人改过”。整段要一致，就自己加锁，或改成容器提供的单个原子方法。
- `size`、`isEmpty`、`containsValue` 在并发下允许反映中间状态，只能当近似，不能用来决定还要不要继续补。
- `putAll` 也不是原子的，填到一半时别人能读到一部分新键。

把观察和补货放进同一把锁，只有一个任务看到缺口 200，其余看到 0，最终是 1000：

```java
tasks.add(pool.submit(() -> {
    synchronized (catalog) {
        int missing = TARGET - catalog.size();
        log.info("{} missing={}", Thread.currentThread().getName(), missing);
        for (int i = 0; i < missing; i++) {
            catalog.put(UUID.randomUUID().toString(), 1L);
        }
    }
}));
```

全程锁住并发表，确实和“先锁再改一张普通表”差不多，分段并发的好处没用上。这个容器的优势在另一类 API：一次调用完成复合更新，内部按槽位做比较并交换，而不是让所有线程抢同一把监视器。

锁的粒度也要看谁在读。上面的 `synchronized (catalog)` 只拦住同样来抢这把锁的补货任务。别的线程如果直接 `get`，并不会进入这把锁，因此在循环放到一半时仍能看见 900、1100 这种中间数量。若业务要求“要么还是 800，要么已经是 1000”，观察者和补货者必须共用同一把锁，或者改成先在线程私有的映射里组装好、再一次交换出去。`size` 返回的是一个近似计数，高并发下为了不去锁整张表，它允许短暂偏差；拿它做流程判断，等于把近似值当成了事务读。

还有一种更慢的交错值得自己跑一次：每个任务先根据缺口造出一整张临时表，再 `putAll`。造表的时间里，别人已经把数据填进去了，于是后到的任务打印出的缺口会变成零头。缺口计算和写入之间隔得越久，过期快照越明显。把这两步收进锁里之后，造表也发生在锁内，其它补货任务要么看见补完的结果，要么还没开始看。锁内不要做远程调用，否则这把锁会把所有补货者都堵住。

## 外面套一把大锁，就用不上按槽位的原子更新

统计键出现次数很常见。对照用 8 条线程、200 个键、一共 100 万次累加。保守写法锁住整张表，再读旧值、加一、写回：

```java
static final int LOOPS = 1_000_000;
static final int THREADS = 8;
static final int KEYS = 200;

Map<String, Long> withMonitor() throws Exception {
    ConcurrentHashMap<String, Long> freqs = new ConcurrentHashMap<>(KEYS);
    ExecutorService pool = Executors.newFixedThreadPool(THREADS);
    List<Future<?>> tasks = new ArrayList<>();
    for (int t = 0; t < THREADS; t++) {
        tasks.add(pool.submit(() -> {
            ThreadLocalRandom random = ThreadLocalRandom.current();
            for (int i = 0; i < LOOPS / THREADS; i++) {
                String key = "k" + random.nextInt(KEYS);
                synchronized (freqs) {
                    Long prev = freqs.get(key);
                    freqs.put(key, prev == null ? 1L : prev + 1);
                }
            }
        }));
    }
    for (Future<?> task : tasks) {
        task.get();
    }
    pool.shutdown();
    return freqs;
}
```

结果正确，但所有键的更新挤在同一把锁上，线程数越多越像串行。改成容器自己的计算，值用可以分段累加的 `LongAdder`：

```java
Map<String, Long> withAtomicUpdate() throws Exception {
    ConcurrentHashMap<String, LongAdder> freqs = new ConcurrentHashMap<>(KEYS);
    ExecutorService pool = Executors.newFixedThreadPool(THREADS);
    List<Future<?>> tasks = new ArrayList<>();
    for (int t = 0; t < THREADS; t++) {
        tasks.add(pool.submit(() -> {
            ThreadLocalRandom random = ThreadLocalRandom.current();
            for (int i = 0; i < LOOPS / THREADS; i++) {
                String key = "k" + random.nextInt(KEYS);
                freqs.computeIfAbsent(key, ignored -> new LongAdder()).increment();
            }
        }));
    }
    for (Future<?> task : tasks) {
        task.get();
    }
    pool.shutdown();
    Map<String, Long> plain = new HashMap<>();
    freqs.forEach((key, adder) -> plain.put(key, adder.sum()));
    return plain;
}
```

`computeIfAbsent` 在键缺失时用函数生成值并放入，然后返回该值；“检查再放入”由容器完成。不同键甚至同一个键上的 `increment` 都不必抢那一把大锁。校验两条：键的个数是 200，所有值之和是 100 万。

一次 8 线程、100 万次的本地对照里，监视器版本约两百多毫秒，原子版本约几十毫秒。倍数随争用变化，方向稳定。容器往桶里发布新节点时，用的是比较并交换：槽里仍是期望的旧引用才换成新引用，失败就重读再试。这比给整张表挂重量级锁便宜，但只覆盖这一次调用。你自己写的“先 size 再 putAll”不会自动变成这种原子操作。

`merge` 也能做计数，适合值本身就是数字、争用不极端的情况：

```java
ConcurrentHashMap<String, Long> counts = new ConcurrentHashMap<>();
counts.merge(key, 1L, Long::sum);
```

一次调用完成“没有则放入 1，有则把旧值与 1 相加”。它仍可能在同一个键上重试，但重试范围是这个键所在的槽，不是整张表。高争用下 `LongAdder` 往往更合适，因为加法分散在多个格子里，读取时再求和。无论哪一种，都先读文档里该方法算不算原子复合操作，再决定要不要在外面加锁。

校验不要只看没有抛异常。把每个任务的循环次数加总，应等于所有值之和；键的个数应等于随机范围。少了通常是异常被吞，多了通常是任务被重复提交。原子方法保证的是单次更新不被写花，不保证你的循环边界写对了。

```java
static void assertTally(Map<String, Long> freqs) {
    if (freqs.size() != KEYS) {
        throw new IllegalStateException("key count " + freqs.size());
    }
    long sum = 0L;
    for (Long value : freqs.values()) {
        sum += value;
    }
    if (sum != LOOPS) {
        throw new IllegalStateException("tally " + sum);
    }
}
```

## 写时复制只适合几乎不改的读多场景

通用容器在多数负载下过得去。为窄场景准备的结构，只在它假定的读写比下更快，用反了会更慢。

一段本地缓存把大量且经常变的数据放进 `CopyOnWriteArrayList`。这个名字表示：修改时复制整份数组，再把内部引用换成新数组。读路径因此不必加锁，遍历也不会被写者撕开。它适合监听器名单、很少改的配置快照。写入很密时，每次复制的成本盖过收益。

```java
@GetMapping("/lists/write")
public Map<String, Long> compareWrites() {
    List<Integer> copying = new CopyOnWriteArrayList<>();
    List<Integer> locking = Collections.synchronizedList(new ArrayList<>());
    int rounds = 20_000;
    long copyingMs = elapsed(() ->
            IntStream.range(0, rounds).parallel().forEach(copying::add));
    long lockingMs = elapsed(() ->
            IntStream.range(0, rounds).parallel().forEach(locking::add));
    Map<String, Long> body = new LinkedHashMap<>();
    body.put("copyingMs", copyingMs);
    body.put("lockingMs", lockingMs);
    return body;
}
```

一次本地运行里，两万次并行追加，写时复制大约一两百毫秒，同步包装的普通列表大约几毫秒。读取反过来：各自先放入二十万个元素，再并行按下标取四十万次，写时复制更快，因为它的 `get` 不加锁。倍数取决于数组已经多长、核数和争用，不要背成固定比例。

写入的形状如下。第 n 次追加要复制 n 个槽，连续追加的复制总量接近平方级：

```java
public boolean add(E item) {
    synchronized (guard) {
        Object[] current = snapshot;
        Object[] next = Arrays.copyOf(current, current.length + 1);
        next[current.length] = item;
        snapshot = next;
        return true;
    }
}
```

注意它并不是完全无锁：写者之间仍用一把锁串行，只是读者不拿这把锁。选择标准是读写比。写多就用普通列表加明确的临界区，或者换为并发写入设计的队列。

读取方向可以用同一套计时看相反的结果。先各自顺序放入二十万个整数，再并行按下标取四十万次。写时复制的 `get` 只是数组访问；同步包装的列表每次读取都要进监视器。在读远多于写的监听器名单上，这个差别是正收益。一旦写者也频繁 `add` 或 `remove`，新数组的分配和复制会把年轻代打满，读得再快也补不回停顿。

```java
static long timeReads(List<Integer> values, int rounds) {
    int size = values.size();
    return elapsed(() -> IntStream.range(0, rounds).parallel().forEach(i -> values.get(i % size)));
}
```

迭代时还要记住：迭代器绑的是开始那一瞬间的数组。循环过程中别的线程追加的元素，这次迭代看不见，也不会抛出并发修改异常。需要“读到最新名单”的地方，不要假设迭代期间的写入会反映进来。配置快照正好利用这一点；账户余额这种必须立刻可见的数据则不合适。

删除同样会复制整份数组，只是少一个槽。所以“偶尔删除一个监听器”可以接受，“每个请求都删一条再加一条”不该用这个列表。换结构之前先量一下写的频率和单次复制的数组长度，而不是只看类名里有没有 lock。

## 收束

四类误用：

- 没弄清代码跑在哪条线程上。池化线程会把线程局部变量带给下一单，任务结束前要 `remove`。
- 把类换成线程安全的实现，就以为复合业务也安全。先观察再批量写入这种多步逻辑，需要锁，或者收成一次原子调用。
- 新容器仍按旧习惯用。外面一把大锁，就用不上按槽位做的原子更新。
- 专用结构和场景不匹配。写时复制用在密集写入上，每次修改都整表复制。

避开的办法：先读该容器文档里每个方法的并发说明，再用小程序把竞争跑出来。这类缺陷往往要压到一定交错才出现，单次手工点击不一定看得到。
