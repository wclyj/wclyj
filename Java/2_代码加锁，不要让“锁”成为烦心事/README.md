# 加锁之前，先写清要保护的那一段

锁是处理线程安全的常用手段，用错的方式却很固定：锁了不该锁的方法、锁对象和字段不在一个层级、临界区过大，或者几把锁的获取顺序互相缠住。下面用四组可运行的例子把这些坑走一遍。

## 两个计数器看起来一起变，比较结果却自相矛盾

有人拿下面的类去问“是不是虚拟机算错了”。类里有两个 `volatile int`：`arrived` 表示到站件数，`departed` 表示离站件数。`record` 循环 8000 次，每次让两个字段都加一。`audit` 同样循环 8000 次，一旦看到到站数小于离站数，就把两个值和“到站是否大于离站”再打出来。

```java
public class GateCounter {
    volatile int arrived = 1;
    volatile int departed = 1;

    public void record() {
        for (int i = 0; i < 8000; i++) {
            arrived++;
            departed++;
        }
    }

    public void audit() {
        for (int i = 0; i < 8000; i++) {
            if (arrived < departed) {
                System.out.printf("arrived=%d departed=%d arrivedGt=%s%n",
                        arrived, departed, arrived > departed);
            }
        }
    }
}
```

两条线程各跑一个方法：

```java
GateCounter gate = new GateCounter();
new Thread(gate::record).start();
new Thread(gate::audit).start();
```

直觉是：两边同步加一，两个数应当永远相等，`arrived < departed` 不该成立，更不该在成立的同时又打印出 `arrived > departed` 为真。实际跑起来，不但会打日志，而且同一行里两种比较有时同时为真。

有人马上给 `record` 加上方法锁：

```java
public synchronized void record() { /* 循环体不变 */ }
```

日志还在。原因不是锁失效，而是这把锁从头到尾只有 `record` 那条线程会去拿，`audit` 根本不竞争它。锁只能让“抢同一把锁的线程”互斥，不能自动保护没进入临界区的读。

两个方法的语句还会交错，而且语句本身也不是原子的。`arrived++` 与 `departed++` 之间，审计线程可以插进来读。更隐蔽的是，`arrived < departed` 在字节码里至少是“读到站、读离站、比较”三步；条件成立后再次计算 `arrived > departed`，又是一轮读取。写线程完全可能在这两轮读取之间把两个字段改完，于是第一轮看到旧的到站数配上新的离站数，第二轮又看到反过来的组合。

正确做法是让写和读抢同一把锁，写的时候读进不来：

```java
public synchronized void record() { /* 两个字段一起加 */ }

public synchronized void audit() { /* 在锁里完成整段比较和打印 */ }
```

实例方法上的 `synchronized` 锁的是 `this`。两条线程调用同一个 `GateCounter` 时，写和读才会真正互斥。下结论之前先列一张表：共享的是哪几个字段、哪些线程会写、哪些线程会读、准备加的锁对象是谁。对不上，这把锁就是摆设。

## 锁对象要和字段落在同一层

静态字段属于类，所有实例看见的是同一份；非静态方法上的同步锁属于某一个实例。两层套在一起，保护会漏。

```java
class TicketMeter {
    private static int issued = 0;

    public static void reset() {
        issued = 0;
    }

    public static int issued() {
        return issued;
    }

    public synchronized void punch() {
        issued++;
    }
}
```

压测时每次都 new 一个新对象再调用：

```java
@GetMapping("meter/coarse")
public int coarse(@RequestParam(defaultValue = "200000") int times) {
    TicketMeter.reset();
    IntStream.rangeClosed(1, times).parallel().forEach(i -> new TicketMeter().punch());
    return TicketMeter.issued();
}
```

`times` 取 20 万时，返回值往往明显小于 20 万。`punch` 的锁是各不相同的 `this`，线程之间几乎不互斥，而 `issued` 又是共享的，自增互相覆盖。

修法是引入一把所有实例都能看见的锁，只包住对 `issued` 的修改。不必为了加锁把实例方法改成静态方法，那样会把调用方式也改掉。

```java
class TicketMeter {
    private static int issued = 0;
    private static final Object issueLock = new Object();

    public void punch() {
        synchronized (issueLock) {
            issued++;
        }
    }
}
```

如果业务本来就是“全类一份计数”，也可以把锁加在 `TicketMeter.class` 上，效果等价于静态同步方法。块级 `synchronized` 与方法修饰符的差别在字节码上也能看到：方法修饰符由方法访问标志 `ACC_SYNCHRONIZED` 表达，进入方法时由虚拟机自动加解锁；块级则是编译器插入成对的 `monitorenter` / `monitorexit`，锁对象由你指定，因此可以锁类、锁字段、锁一个专门的哨兵对象。

## 临界区只包住真正共享的那几行

把 `synchronized` 写在方法上很省事，于是有的服务给几乎每个入口都加了。这会带来两件事。

一是多数 Web 入口没有需要保护的内存字段。请求从控制层到服务层再到仓储层，状态落在数据库里，方法本身是无状态的。给它们加锁，并没有一份共享数据被保护。

二是 Spring 默认把这些层做成单例。方法锁等价于锁住整个 Bean，所有请求串行通过，吞吐直接塌掉。

即便真有共享集合，也只锁碰它的那几行。下面的 `auditRemote` 只是在模拟一次 8 毫秒的外部核对，不读不写 `shipmentIds`。粗粒度写法把核对和入列放进同一把锁；细粒度写法让核对留在锁外，只在 `add` 时锁住列表本身。

```java
private final List<Integer> shipmentIds = new ArrayList<>();

private void auditRemote() {
    try {
        TimeUnit.MILLISECONDS.sleep(8);
    } catch (InterruptedException ex) {
        Thread.currentThread().interrupt();
    }
}

@GetMapping("ship/coarse")
public int coarse() {
    long begin = System.nanoTime();
    IntStream.rangeClosed(1, 800).parallel().forEach(id -> {
        synchronized (shipmentIds) {
            auditRemote();
            shipmentIds.add(id);
        }
    });
    log.info("coarse ms={}", (System.nanoTime() - begin) / 1_000_000);
    return shipmentIds.size();
}

@GetMapping("ship/fine")
public int fine() {
    long begin = System.nanoTime();
    IntStream.rangeClosed(1, 800).parallel().forEach(id -> {
        auditRemote();
        synchronized (shipmentIds) {
            shipmentIds.add(id);
        }
    });
    log.info("fine ms={}", (System.nanoTime() - begin) / 1_000_000);
    return shipmentIds.size();
}
```

粗粒度版本里，800 次睡眠被同一把锁串起来，墙钟时间接近 800×8 毫秒。细粒度版本里，睡眠可以重叠，列表上的临界区只剩一次 `add`。两边最后的列表长度都应当是 800，差的是等待时间。

锁范围收到最小之后如果还不够快，再按读写比例往下拆，而不是继续加更多的互斥。

- 读远多于写时，可以用 `ReentrantReadWriteLock`：多个读者共享，写者独占。
- JDK 8 及以上、并且读与读几乎不冲突时，可以看 `StampedLock` 的乐观读：先拿一个戳记去读，读完再校验戳记有没有被写操作作废，没有作废就不用升级成悲观读锁。
- `ReentrantLock` 和读写锁都有公平模式。临界区很短时打开公平，线程会严格排队，上下文切换变多，吞吐可能差一个数量级以上。没有“必须防止饥饿”的需求时保持非公平。

## 一次业务拿多把锁，先约定全局顺序

粒度拆细之后，一个用例可能要连续拿几把锁。拿锁顺序不一致，就会循环等待。

一个真实形态的例子是：下单要扣掉购物车里每种商品的库存，必须先拿到这些商品各自的锁，全部拿到才扣减，扣完再一起放开。上线后失败率很高，用户被迫重试。线程转储显示不是库存不够，而是互相占着对方下一步需要的那把锁。

用场地预约来复现。每种场地有自己的余量和一把锁，初始余量 200，一共 8 间。

```java
@Data
static class Room {
    final String code;
    int seatsLeft = 200;
    @ToString.Exclude
    final ReentrantLock lock = new ReentrantLock();
}
```

选座时从目录里随机抽 3 间，不考虑同一间被抽中两次：

```java
private List<Room> pickRooms() {
    return IntStream.range(0, 3)
            .mapToObj(i -> "R" + ThreadLocalRandom.current().nextInt(rooms.size()))
            .map(rooms::get)
            .collect(Collectors.toCollection(ArrayList::new));
}
```

预约逻辑：按列表顺序 `tryLock`，最多等 3 秒。中途有一间没拿到，就把已经拿到的全部放开，返回失败。全部拿到之后才减余量。

```java
private boolean reserve(List<Room> chosen) {
    List<ReentrantLock> held = new ArrayList<>();
    for (Room room : chosen) {
        try {
            if (room.lock.tryLock(3, TimeUnit.SECONDS)) {
                held.add(room.lock);
            } else {
                held.forEach(ReentrantLock::unlock);
                return false;
            }
        } catch (InterruptedException ex) {
            held.forEach(ReentrantLock::unlock);
            Thread.currentThread().interrupt();
            return false;
        }
    }
    try {
        chosen.forEach(room -> room.seatsLeft--);
        return true;
    } finally {
        held.forEach(ReentrantLock::unlock);
    }
}
```

并发跑 60 次“选座 + 预约”，统计成功次数和剩余座位。未排序时，成功次数经常小于 60：一张单子先拿到 R1 再等 R3，另一张先拿到 R3 再等 R1，两边都在 `tryLock` 里干等到超时。因为等待有上限，进程不会永远卡死，但每次冲突都要付出数秒，用户侧就是预约失败。

用 VisualVM 或 `jstack` 在冲突窗口抓线程，能看到两条工作线程各自持有一把 `ReentrantLock`，又阻塞在对方持有的那把上。这就是典型的循环等待。

避免的办法不是把超时再加长，而是让所有线程按同一套顺序拿锁。这里按场地编码排序后再预约：

```java
@GetMapping("reserve/ordered")
public long ordered() {
    return IntStream.rangeClosed(1, 60).parallel()
            .mapToObj(i -> pickRooms().stream()
                    .sorted(Comparator.comparing(Room::getCode))
                    .collect(Collectors.toCollection(ArrayList::new)))
            .map(this::reserve)
            .filter(ok -> ok)
            .count();
}
```

排序之后，任何线程需要 R1 和 R3 时都是先 R1 后 R3。不会再出现“我握着你的下一步、你握着我的下一步”。60 次应全部成功，耗时接近扣减本身，而不是成片的 3 秒超时。余量减少量等于成功次数乘以每次 3 个座位，可以用来核对有没有重复扣或漏扣。

这个例子里超时避免了永久死锁，排序消除了循环等待。两件事都要留：只排序但 `lock()` 无限等，一旦顺序约定被新代码打破，系统仍会僵住；只超时不排序，失败和重试会把体验打穿。

## 回顾

- 先写清共享状态是类一级还是实例一级，以及哪些线程会碰它。`synchronized` 关联的是哪个对象、哪段代码，要能指出来。只锁写方法、不锁读方法，保护是断的。
- 临界区等于一段不能并行的代码。Web 应用天然多线程，给单例方法大范围加锁会把并发度打成 1。能锁集合就不要锁整个服务对象。读多写少再考虑读写锁或带戳记的乐观读；公平锁留给确实会饥饿的场景。
- 一个用例拿多把锁时，同时避开无限等待和循环等待：获取要有超时，顺序要全局一致。
- 加锁和释放必须成对。复杂分支里要检查每条路径，包括异常路径，是不是都释放了，有没有对同一把锁解锁两次。分布式锁还会自动到期，业务如果还在跑，另一台机器可能再次拿到同名锁，把同一笔操作做两遍。锁的租约要盖过业务耗时，或者用校验令牌防止过期后的旧持有者继续提交。

演示代码为了方便，在接口里临时开线程。真正的 Web 请求本来就在不同线程上执行，共享字段更容易被忽略，误加的方法锁也更容易把整站吞吐拉低。锁逻辑一旦分支变多，应把外部依赖换成替身，再压一轮，确认没有超时风暴，也没有线程转储里的环形等待。

## 还值得追的两点

开篇的两个计数器为什么要 `volatile`？它保证审计线程不会一直拿着寄存器里的旧值，每次比较都能看见写线程已经发布的结果。它并不把“读-改-写”变成原子操作，所以单靠它消不掉交错，只是让交错变得可见。

另一个常见坑是：后台线程用 `while (running)` 空转执行任务，主线程稍后把 `running` 设成 `false`。这个布尔如果不是 `volatile`，也没有别的同步边，JIT 可能把它当成循环不变量，子线程永远看不到新值，循环停不下来。改成 `volatile`，或者用锁发布这次写入，子线程才能退出。

发现“加锁没配对”可以在联调时打开 `-XX:NativeMemoryTracking` 并无太大帮助，更直接的是代码审查加静态检查：锁的获取放在 `try` 之外、释放放在 `finally`，或者使用能自动关闭的包装。发现“锁到期后逻辑重入”则要在提交结果前核对持有者令牌，令牌不符就放弃写入，而不是假设自己仍然独占。
