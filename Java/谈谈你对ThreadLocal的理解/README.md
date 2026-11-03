# 每个线程一份变量，用完要自己拆掉

## 它解决什么

`ThreadLocal` 提供线程内部的变量。同一条线程上的 `get` 和 `set` 看见的是这份副本，别的线程看不见。实例通常声明成私有静态字段，用来把“当前上下文”挂在线程上，而不是挂在某个业务对象上。

可以把它的用途收成三点：

- 并发时不必把这份数据做成共享可变状态。
- 同一次调用链上的多个方法都能拿到它，不用每个方法再加一个参数。
- 线程之间互不覆盖。

它不负责把数据传给线程池里的下一次任务，也不负责在线程结束前自动清空。那是使用方的事。

## 四个入口

| 方法 | 作用 |
| --- | --- |
| 构造 | 造一个槽，此时还没有任何线程写入 |
| `set` | 给当前线程放值 |
| `get` | 取当前线程的值；没有则走初始值 |
| `remove` | 把当前线程这份条目删掉 |

另外还有可覆盖的 `initialValue`，以及用工厂的 `withInitial`。

## 共享字段为什么会串

五个线程共用一个对象，各自把自己的名字写进字段再读出来。字段只有一份，写和读之间会被别人插进来。

```java
final class SharedBoard {
    private String note;

    public static void main(String[] args) throws InterruptedException {
        SharedBoard board = new SharedBoard();
        int workers = 5;
        for (int n = 0; n < workers; n++) {
            Thread worker = new Thread(() -> {
                board.note = Thread.currentThread().getName() + "#data";
                System.out.println(Thread.currentThread().getName() + " -> " + board.note);
            }, "t" + n);
            worker.start();
        }
    }
}
```

输出里会出现 `t1 -> t4#data` 这种张冠李戴。把字段换成槽之后，每个线程只看自己写入的那份：

```java
final class IsolatedBoard {
    private static final ThreadLocal<String> content = new ThreadLocal<>();

    public static void main(String[] args) throws InterruptedException {
        List<Thread> workers = new ArrayList<>();
        int total = 5;
        for (int n = 0; n < total; n++) {
            Thread worker = new Thread(() -> {
                content.set(Thread.currentThread().getName() + "#data");
                System.out.println(Thread.currentThread().getName() + " -> " + content.get());
            }, "t" + n);
            workers.add(worker);
            worker.start();
        }
        for (Thread worker : workers) {
            worker.join();
        }
    }
}
```

此时每一行的名字和内容前缀一致。线程结束前最好再 `remove`。这个例子里线程会退出，条目随线程一起变成垃圾；放进池里时就不会退出。

## 加锁也能让打印不串，但把并发拿掉了

同一段逻辑可以收进同步块：进入之后别人不能改这个字段，打印出来的名字和内容是匹配的。代价是五个线程排队，本来可以同时做的事变成串行。

同步解决的是“多个人共用一份，不能同时改”。线程局部变量解决的是“每个人本来就该有自己的一份”。这个例子要的是隔离，不是互斥，所以槽更合适，线程之间不用排队。

|  | 同步 | 线程局部变量 |
| --- | --- | --- |
| 思路 | 一份数据，用时间换互斥 | 每条线程一份，用空间换同时进行 |
| 关注点 | 共享资源的访问顺序 | 线程之间的数据不要串 |

## 把连接绑到当前事务

### 转账要的是同一个连接

账户表里 Alice 给 Blake 转一笔。扣款和入账是两条更新，中间如果失败，两边都必须回到原样。演示可以用 JDBC 加一个连接池，不必引入整套框架。

服务层若写成先扣、再制造一个除零、再入账，异常被吃掉之后，扣款已经提交，入账没有发生。

```java
public boolean transfer(String fromUser, String toUser, int cents) {
    AccountDao dao = new AccountDao();
    try {
        dao.debit(fromUser, cents);
        int broken = 1 / 0;
        dao.credit(toUser, broken + cents);
        return true;
    } catch (RuntimeException ex) {
        ex.printStackTrace();
        return false;
    }
}
```

JDBC 里关掉自动提交，成功则 `commit`，失败则 `rollback`。前提有两个：服务层和数据访问层必须拿同一个 `Connection`；池里每条工作线程只能碰自己的那条连接，不能串到别人的事务。

### 参数传来传去，或者整段加锁

常规做法是从服务层把连接一路传到每个数据访问方法，或者在转账外面加一把全局锁。前者让每个方法签名都多一个与业务无关的参数，中间再套一层第三方库就传不进去。后者让所有转账排队，连接池的并发没有意义。

### 槽里放当前连接

从池里取连接的入口改成：当前线程已经绑过就直接返回，没有则取一条并绑上。事务结束时提交或回滚，然后归还连接并 `remove`，避免下一次任务沿用这条已经还回池里的连接。

```java
final class Connections {
    private static final ThreadLocal<Connection> current = new ThreadLocal<>();

    static Connection open(DataSource source) throws SQLException {
        Connection existing = current.get();
        if (existing != null) {
            return existing;
        }
        Connection created = source.getConnection();
        current.set(created);
        return created;
    }

    static void close() throws SQLException {
        Connection existing = current.get();
        current.remove();
        if (existing != null) {
            existing.close();
        }
    }
}
```

服务层在开头取出连接、关掉自动提交，数据访问层再调 `open` 时拿到的是同一条。异常路径回滚，正常路径提交，`finally` 里归还。

```java
public boolean transfer(String fromUser, String toUser, int cents) {
    try {
        Connection conn = Connections.open(source);
        conn.setAutoCommit(false);
        AccountDao dao = new AccountDao();
        dao.debit(fromUser, cents);
        dao.credit(toUser, cents);
        conn.commit();
        return true;
    } catch (Exception ex) {
        rollbackQuietly();
        return false;
    } finally {
        closeQuietly();
    }
}
```

数据访问方法内部再次调用 `Connections.open`，拿到的是刚才那条，而不是池里的另一条。因此扣款和入账要么一起提交，要么一起回滚。若 `debit` 自己去池里 `getConnection`，它会在另一条连接上自动提交，外层回滚也追不回这笔扣款。槽解决的是“怎么找到同一条”，不是“事务语法本身”。自动提交必须关掉，否则每条语句结束就已经落库。

池化线程上还有一个顺序问题：`close` 若先把连接还给池、后 `remove`，中间一旦抛错，槽里仍指着一条已经不属于这次任务的连接。下一次 `open` 看见非空就直接返回，于是用了别人正在用的连接。所以先从槽里摘下来，再关闭或归还。归还失败也要保证槽已经空了。

这样做的好处就是前面说的两点：调用链不用传递连接参数；不同线程的事务同时进行，又不会共用一条连接。

## 副本放在线程里面，不放在槽对象里面

### 早期容易猜成的结构

不看实现时，会以为每个 `ThreadLocal` 自己持有一张表，键是线程，值是副本。早期版本确实接近这种设计。线程很多、槽很少时，每张表都按线程数长大，线程结束后还要设法从表里删掉自己。

### 现在是线程持有表

当前的实现反过来：

- 每条 `Thread` 里有一张 `ThreadLocalMap`。
- 表的键是 `ThreadLocal` 实例，值才是业务放进去的对象。
- `get` 和 `set` 先找到当前线程，再操作这张表。
- 别的线程的表根本走不到，所以自然隔离。

线程数量通常远大于槽的数量，于是每张表只装这条线程真正用过的那几个槽，条目更少。线程结束时，表作为线程的字段一起不可达，不需要到全局结构里按线程清理。

## 四个方法实际在动这张表

`set`：取当前线程；表已存在就把“本槽 → 值”放进去；表还不存在就建表并放入第一对。

`get`：取当前线程的表；按本槽找到条目且键还在，就返回值。表不存在、条目不存在、或键已经被清掉，则调用 `initialValue`（默认 null），再把这个初始值放进新的或现有的表。所以先 `get` 后 `set` 时，会先触发一次初始值。

`remove`：表存在就按本槽删掉条目。不存在就什么也不做。

`initialValue` 是延迟的，只有 `get` 发现没有值时才调用，默认返回 null。想要非空初始值就覆盖它，或者用 `withInitial`。它是受保护的，就是为了让子类提供初始值。不要在里面做重活，因为每个线程的第一次 `get` 都会走到。

用伪代码把三条路径写开，读源码时就能对上号。这里的 `map` 是当前线程里的那张表，键是槽对象自己：

```java
void set(T value) {
    Thread self = Thread.currentThread();
    SlotMap map = self.slotMap;
    if (map != null) {
        map.put(this, value);
    } else {
        self.slotMap = new SlotMap(this, value);
    }
}

T get() {
    Thread self = Thread.currentThread();
    SlotMap map = self.slotMap;
    if (map != null) {
        Object found = map.find(this);
        if (found != SlotMap.MISSING) {
            return (T) found;
        }
    }
    T seed = initialValue();
    set(seed);
    return seed;
}

void remove() {
    SlotMap map = Thread.currentThread().slotMap;
    if (map != null) {
        map.delete(this);
    }
}
```

`get` 在没有条目时会调用 `set`，所以“只读一次”也可能把初始值写进表。初始值若是可变对象，每个线程第一次 `get` 都应拿到自己的实例，不能把同一个 `ArrayList` 从 `initialValue` 里反复返回。工厂写在 `withInitial` 里时，每次调用工厂都会新建，这一点是安全的。

`remove` 之后再次 `get`，会重新走初始值，而不是把旧值变回 null 后还留着条目。需要“有没有人设置过”的语义时，不要用 null 当业务值，否则和“从未设置”分不开。可以放一个明确的空对象，或者改用 `Optional`，并仍然在任务结束时移除。

## 表本身长什么样

`ThreadLocalMap` 是槽的内部类，没有实现 `Map` 接口，自己维护一个条目数组。

字段和哈希表类似：初始容量 16，而且保持 2 的幂；`table` 存放条目；`size` 是已用个数；`threshold` 是再散列的门槛。容量用 2 的幂，是为了用“与运算”代替取模。

条目的键是弱引用包着的 `ThreadLocal`，值是普通强引用：

```java
static final class SlotEntry extends WeakReference<ThreadLocal<?>> {
    Object value;

    SlotEntry(ThreadLocal<?> key, Object value) {
        super(key);
        this.value = value;
    }
}
```

键用弱引用，是为了让槽对象本身的生命周期不必跟着线程走。业务代码丢掉槽的引用之后，键可以先被收掉，条目上留下空键，值留待后面的探测去清。

## 弱引用没有取消泄漏，只是多了一次补救

### 两种失败不要混

内存溢出是申请时已经没有可用空间。内存泄漏是已经不用的对象仍被引用着，空间还不回来。泄漏堆久了才会表现为溢出。

强引用是普通引用，还有一条强引用在，回收器就不动这个对象。弱引用则是：一旦只剩弱引用，下一次回收就会把对象清掉，不管当前堆还空不空。

### 键若是强引用

业务方法结束，栈上的槽引用没了。若条目仍强引用着槽，槽就活着。线程还在跑（尤其是池里的线程），链路是：线程 → 表 → 条目 → 槽和值。整个条目都回不来。

### 键换成弱引用

同样丢掉槽引用之后，键会被回收，条目里的键变成 null。值这边仍有强引用：线程 → 表 → 条目 → 值。值再也不会被 `get` 到，因为键已经对不上，于是泄漏的是值，不是键。

### 真正的前提

两条路都要同时满足才会留下垃圾：没有人删除条目，而且线程还活着。调用 `remove` 能把条目整个拿掉。线程若随后结束，表也会一起不可达。池化线程不会结束，所以第一条是唯一稳妥的办法。

既然记得 `remove` 就行，为什么还要把键做成弱引用？因为 `set`、`get`、`remove` 在探测时会看见空键，并把对应的值抹掉。忘记清理时，只要这条线程以后还碰过这张表，弱引用至少让值有机会被顺手清掉。强引用连这层机会都没有。这是补救，不是许可证。过滤器的 `finally` 里仍然要 `remove`。

把池化线程的生命周期写成四步，泄漏出现在哪一步就清楚了：

1. 请求 A 进入，工作线程 T 从池里取出，`set` 放入用户 A。表里一条条目，键和值都可达。
2. 请求结束但没有 `remove`。T 回到池里，条目还在。
3. 业务代码不再持有槽的静态引用以外的路径。静态字段本身通常还在，键是强可达的，弱引用帮不上忙。只有槽对象也失去了所有强引用（例如槽存在于一个已被卸载的类加载器，或者用完就把静态字段置空），键才会先被回收，值留下。
4. 请求 B 再次用到 T。若步骤 3 没有发生，B 在 `set` 之前 `get` 到的仍是用户 A。这是串数据，也是值没被释放。

所以池上的首要问题是串数据，泄漏是线程长时间活着且条目不删的后果。弱引用只在槽对象自己已经不可达时，帮你在后续的 `get` 或 `set` 里把值清掉。应用里的槽几乎都是静态常量，一直可达，弱引用这条补救根本不会触发。能依赖的只有 `remove`。

值若还持有大对象，例如一份几兆的报表字节，一条忘记清理的池线程就会把这块字节留到线程被销毁。池不会为了释放它而销毁线程。监控里若看见老年代随请求稳态上涨、满负载后又不回落，可以把槽里的对象大小算进去。

## 冲突时向后探一格

`ThreadLocal.set` 最终调用表的 `set`。表不存在则走构造。

构造时分配长度为 16 的数组，用键的哈希选下标，放入第一对，并设好门槛。哈希来自一个原子计数加上固定增量 `0x61c88647`。这个增量和黄金分割有关，目的是让连续创建的槽散开在 2 的幂长度的数组上，而不是挤在相邻下标。下标计算是 `hash & (length - 1)`，在容量为 2 的幂时等价于取模，并且不会得到负数下标。

放入已有表时：算出下标，看这一格。键相同就覆盖值。键是 null，说明槽已被回收，用替换过期条目的逻辑把这一格重用，并顺手清掉附近的过期值。这一格被别的键占着，就看下一格，到数组末尾就绕回 0。直到找到空位才新建条目，个数加一。

最后会做一次局部清理。个数达到门槛时再全表扫描，把空键清掉，必要时扩容再散列。所以冲突的解决是线性探测，数组在逻辑上是环形的。例如长度 16、哈希落到 14，而 14 被别人占用，就试 15、0、1，直到空位。

用一张长度为 8 的小表走一遍。三个槽的哈希分别落到 3、3、4。第一个槽占住下标 3。第二个槽也算出 3，发现键不同，于是探测 4。若 4 空着就放在 4；若第三个槽已经占了 4，第二个槽继续看 5。查找时必须沿同一条探测链走，不能在第一格键不同时就宣布没有。中间若出现空键（槽已被回收），查找不能停，因为真正的键可能在更后面；放入时则可以重用这个空键的格子，避免数组被过期条目撑满。这就是替换过期条目要比“直接再找一个空位”多做的事：既复用格子，又保持后面的探测链仍然能找到原来的键。

扩容之后下标要按新长度重算，不能把旧下标原样拷过去。门槛一般是容量的三分之二左右，提前再散列，是为了让探测链不要长到接近整张表。槽的数量在一个应用里通常只有几十个，这条链很少成为热点；忘记 `remove` 时，过期条目才会把链拉长，每次 `get` 都要多看好几格。

## 框架和业务里它还出现在哪

事务管理器要保证一次业务方法里的多次数据库操作共用一个连接，又要按传播行为把外层事务挂起、恢复。它把资源映射、同步回调和当前事务名放在各自的槽里。业务方法不必接收连接参数，切面在进入时绑定、退出时清理。这和前面手写的 `Connections` 是同一个结构，只是槽里放的是一整张资源表。

日期格式化对象内部带着可修改的日历。一个线程刚 `add`，另一个线程 `clear`，解析结果就会串。给每个线程 `new` 一个格式化器能避开共享，但在池里不必为每条任务都新建：用槽保存每条线程自己的那一个，`initialValue` 里构造。用完不必 `remove` 格式化器本身，除非线程会被回收、你在意那点内存；任务级的用户数据则一定要删。

共享一个格式化器时，解析会错。`parse` 内部先清空日历再按字段相加。线程甲清空之后，线程乙把年月写进去，甲再继续加日，得到的日期就不属于任何一个输入：

```java
static final DateFormat shared = new SimpleDateFormat("yyyy-MM-dd", Locale.ROOT);

static Date read(String text) throws ParseException {
    synchronized (shared) {
        return shared.parse(text);
    }
}
```

加锁能得到正确结果，但所有解析排队。槽让每条线程保留自己的格式化器，解析不必互斥：

```java
static final ThreadLocal<DateFormat> formats = ThreadLocal.withInitial(
        () -> new SimpleDateFormat("yyyy-MM-dd", Locale.ROOT));

static Date read(String text) throws ParseException {
    return formats.get().parse(text);
}
```

池里的线程数有上限，格式化器的个数也就有上限，不必为每个任务新建。这里的值生命周期和线程一样长，可以不在每次任务结束时移除。一旦槽里放的是这次请求的用户而不是格式化器，就必须移除，两者不要混成一个习惯。`java.time` 的格式化器本身不可变，可以安全地共享一个静态常量，连槽都不需要。新代码优先用那一套，这个例子只说明“内部有可变日历的对象不能跨线程共享”。

另一种用途是上下文：用户、追踪号要穿过一长串方法，有的方法来自改不了的库，加参数不现实。进入链之前放进槽，库外面的代码自己取，`finally` 里移除。

```java
static final ThreadLocal<User> currentUser = new ThreadLocal<>();

void handle(User user) {
    currentUser.set(user);
    try {
        load();
        verify();
        store();
    } finally {
        currentUser.remove();
    }
}
```

许多把请求级 cookie 或会话状态和线程对齐的实现，也是这个模式。界面框架里，有的事件循环用槽保证一条线程只有一个循环对象：`get` 已经有值就拒绝再次准备，否则 `set` 一个新的循环。这样写循环的代码不用把循环对象传来传去，也能防止同一条线程建出两个循环。

## 延伸

对照时直接看所用 JDK 里 `ThreadLocal` 的 `set`、`get`、`remove`，以及内部表的探测和过期条目清理。不同主版本的私有字段名会调整，清理发生在探测过程中这一点是稳定的。池化线程上的数据，以 `finally` 里的 `remove` 为准，不要依赖弱引用碰巧被收掉。

给槽起一个能在转储里辨认的名字，排查“是哪一个槽没清”会快很多。可以包一层，在 `toString` 里写上业务含义：

```java
final class NamedSlot<T> extends ThreadLocal<T> {
    private final String label;

    NamedSlot(String label) {
        this.label = label;
    }

    @Override
    public String toString() {
        return "slot:" + label;
    }
}
```

线程转储或调试器里看到 `slot:operator`，就知道是操作者上下文，而不是日期格式化器。名字不能代替 `remove`。它只让泄漏发生时，你能指出是哪一类数据还挂在工作线程上。静态槽的个数应和业务上下文的种类一样少；每一种请求级数据一个槽，而不是每个方法一个槽。方法级的临时变量放在栈上即可。

提交到池里的任务若在构造时读了外层槽，读到的是提交线程的值，不是执行线程以后会写进去的值。要把上下文带进池，在提交处先 `get` 到局部变量，再在任务开头 `set`，任务的 `finally` 里 `remove`。不要在任务里直接读外层线程的槽，那一次读取发生在错误的线程上，得到的是空或者池线程上一次留下的值。可继承的子类只对 `new Thread` 之后才启动的子线程拷贝一份，对已经活在池里的线程无效。池和继承一起用时，以提交处的显式拷贝为准。
