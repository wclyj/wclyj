# 队列同步器 AQS 到底在协调什么

## 一个对不上的计数

AQS，也就是 `AbstractQueuedSynchronizer`，是并发包里多数同步工具的骨架。可重入互斥锁、信号量、倒计时门闩、读写锁，都是在它维护的状态字和等待队列上长出来的。

先看一段没有同步的累加。80 条线程各自把共享整数加 50 次，如果每次加法都完整落袋，结果应当是 4000。

```java
public class CounterRace {
    static int total = 0;

    public static void main(String[] args) throws Exception {
        Thread[] workers = new Thread[80];
        for (int i = 0; i < workers.length; i++) {
            workers[i] = new Thread(() -> {
                for (int step = 0; step < 50; step++) {
                    total++;
                }
            });
            workers[i].start();
        }
        for (Thread worker : workers) {
            worker.join();
        }
        System.out.println(total);
    }
}
```

多跑几次，打印值会在 4000 附近晃，有时少几十，偶尔也能刚好打满。`total++` 不是一条不可分割的指令：先把字段读到线程私有的工作区，加上一，再写回去。两条线程读到同一个旧值时，后写回的那次会把前一次的加一盖掉。

可以把一次交错写成四拍：线程甲读到 10；线程乙也读到 10；甲写回 11；乙仍按自己手里的 10 写回 11。两次加法只留下一次效果。线程越多、临界区越短，这种覆盖越容易发生，所以结果时对时错，看起来像虚拟机抽风，其实是共享变量缺少互斥。

## 办法一：用同一把对象监视器包住加法

让所有工人在改 `total` 之前先进入 `CounterSync.class` 的监视器。同一时刻只有一个线程能待在里面。

```java
public class CounterSync {
    static int total = 0;

    public static void main(String[] args) throws Exception {
        Thread[] workers = new Thread[80];
        for (int i = 0; i < workers.length; i++) {
            workers[i] = new Thread(() -> {
                synchronized (CounterSync.class) {
                    for (int step = 0; step < 50; step++) {
                        total++;
                    }
                }
            });
            workers[i].start();
        }
        for (Thread worker : workers) {
            worker.join();
        }
        System.out.println(total);
    }
}
```

无论重复多少次，输出都是 4000。编译器会在进入块之前放上 `monitorenter`，在正常出口和异常出口放上 `monitorexit`。没拿到监视器的线程不能改 `total`。

这把锁默认不排队公平。正在执行的持有者不会被别人从临界区里拽出来；一旦它离开，阻塞队列里的线程和刚好到来的新线程都要再抢一次，后来者可能先成功。所以“会阻塞”不等于“先来的一定先执行”。

### 监视器锁会随竞争升级

刚开始并不会直接做一次内核级加锁。

- 只有一条线程反复进入时，对象头记下这条线程，形成偏向。同一线程再来，几乎只比对一下线程号。
- 第二条线程也来抢时，偏向撤销，变成轻量级锁。失败者用 CAS 自旋，期望持有者很快退出。
- 自旋仍然长时间失败，就膨胀成重量级锁。后来的线程不再空转，而是挂起，等持有者退出时被唤醒。

旧版本可以手写自旋上限。后来改成自适应：刚刚抢成功过的线程允许多转几圈，一直失败的线程更快放弃 CPU。升级路径要记住，否则会误以为 `synchronized` 每次都要进内核。

## 办法二：改成可重入的显式锁

同一段累加也可以交给 `ReentrantLock`。已经持有的线程可以再次进入，状态计数加一；其它线程必须等它成对释放。

```java
public class CounterLock {
    static int total = 0;

    public static void main(String[] args) throws Exception {
        ReentrantLock mutex = new ReentrantLock();
        Thread[] workers = new Thread[80];
        for (int i = 0; i < workers.length; i++) {
            workers[i] = new Thread(() -> {
                mutex.lock();
                try {
                    for (int step = 0; step < 50; step++) {
                        total++;
                    }
                } finally {
                    mutex.unlock();
                }
            });
            workers[i].start();
        }
        for (Thread worker : workers) {
            worker.join();
        }
        System.out.println(total);
    }
}
```

无参构造得到的是非公平实现。`unlock` 放在 `finally`，避免加法中途抛错后锁永不归还。

### 关键字锁已经够用，为什么还要 API 锁

重量级的 `synchronized` 在阻塞和唤醒时要请操作系统帮忙，线程从用户态进入内核态，切换本身就要时间。短临界区如果频繁发生这种切换，吞吐会掉下去。

`ReentrantLock` 的状态和等待队列在语言层用 CAS 维护，抢锁失败时先在队列里登记，真的需要挂起才 `park`。除此之外它还能表达关键字不好直接写的语义：`lockInterruptibly` 可在等待中响应中断，`tryLock(timeout)` 可以放弃，一把锁还能挂出多个 `Condition`，按条件精确唤醒，而不是只有一个等待集合。

## 办法三：按 Lock 接口写一把教学锁

接口把“获取”和“释放”拆开，于是可以自己做一把很朴素的锁：用 `volatile` 标记是否被占用，占用时 `wait`，释放时 `notifyAll`。它内部仍然靠对象监视器，只是把占用协议摊开给调用方看。

```java
public class FlagLock implements Lock {
    private final Object monitor = new Object();
    private volatile int taken = 0;

    @Override
    public void lock()
    {
        synchronized (monitor) {
            while (taken != 0) {
                try {
                    monitor.wait();
                } catch (InterruptedException ex) {
                    Thread.currentThread().interrupt();
                }
            }
            taken = 1;
        }
    }

    @Override
    public void unlock()
    {
        synchronized (monitor) {
            taken = 0;
            monitor.notifyAll();
        }
    }

    @Override
    public void lockInterruptibly() throws InterruptedException { }

    @Override
    public boolean tryLock() { return false; }

    @Override
    public boolean tryLock(long time, TimeUnit unit) { return false; }

    @Override
    public Condition newCondition() { return null; }
}
```

调用形状和前面的显式锁一样：先 `lock`，业务放在 `try`，`finally` 里 `unlock`。空着的超时和条件方法说明这把锁还不完整。真正的库实现不会在 `lock()` 里再套一层 `synchronized`，而是把“没抢到的线程”交给 AQS 的队列。

## AQS 怎样决定谁拿到资源

核心协议可以压成一段话：资源空闲时，把申请者记成当前工作线程，并把共享状态改成已占用；资源已被占用时，不能让申请者空转到天荒地老，而要有一套阻塞、入队、被唤醒后再分配的规则。AQS 用 CLH 锁的变体来做这件事。CLH 队列是逻辑双向链，并没有一个单独的队列容器存放全部节点，只有节点之间的前驱、后继关系。

每条申请线程先被包成一个节点，再按 FIFO 挂到尾部。说得再直白一点：`state` 用 `volatile` 保证可见，线程用 CAS 去改它；改成功就拥有资源，改失败就进等待队列，等前驱释放后被唤醒。唤醒不等于已经拥有，线程还会再自旋尝试一次 CAS，直到自己成功或继续挂起。所以它既是队列锁，也带着自旋。

同步器里和状态相关的入口主要是：

- `getState`
- `setState`
- `compareAndSetState`

资源的分享方式分成两类。

- 独占：同一时刻只放行一条线程，可重入互斥锁属于这种。
- 共享：多条线程可以同时通过。信号量、倒计时门闩、读写锁的读侧、循环栅栏属于这种。

自定义同步器选择哪一种，取决于 `state` 表示的是“被谁独占”还是“还剩多少份额”。

## 模板方法：子类只回答成功与否

AQS 把流程固定成模板。自己写同步器时通常做两件事。

- 继承 `AbstractQueuedSynchronizer`，重写获取和释放 `state` 的钩子。
- 把同步器作为组合对象放进对外组件里。组件的公开方法调用 `acquire`、`release` 这类模板；模板在需要判断“这次算不算抢到”时回调钩子。

下面是一把不可重入独占锁的同步器。CAS 把 0 改成 1 才算成功，并记下独占线程；释放时只允许拥有者把状态清回 0。

```java
public class ExclusiveSync extends AbstractQueuedSynchronizer {

    @Override
    protected boolean tryAcquire(int permits) {
        boolean won = compareAndSetState(0, 1);
        if (won) {
            Thread current = Thread.currentThread();
            setExclusiveOwnerThread(current);
            return true;
        }
        return false;
    }

    @Override
    protected boolean tryRelease(int permits) {
        if (getExclusiveOwnerThread() != Thread.currentThread()) {
            throw new IllegalMonitorStateException("current thread does not hold the sync");
        }
        Thread released = null;
        setExclusiveOwnerThread(released);
        setState(0);
        return true;
    }

    @Override
    protected boolean isHeldExclusively()
    {
        Thread owner = getExclusiveOwnerThread();
        return owner == Thread.currentThread();
    }
}
```

队列的入队、取消、`park` / `unpark` 都由父类完成。子类如果再自己维护一条等待链表，很容易和父类的唤醒顺序打架。

钩子的约定：

- `isHeldExclusively`：当前线程是否正独占资源。要用条件队列时才必须给出。
- `tryAcquire`：独占尝试。成功 `true`，失败 `false`。
- `tryRelease`：独占释放。成功 `true`。
- `tryAcquireShared`：共享尝试。负数失败；0 表示成功但份额用尽；正数表示成功且仍有剩余。
- `tryReleaseShared`：共享释放。释放之后如果应该叫醒后继，返回 `true`。

### 可重入独占锁的 state

`ReentrantLock` 把 `state` 从 0 开始。线程甲第一次 `lock`，`tryAcquire` 成功，`state` 变成 1。甲在持有期间再次进入，发现拥有者就是自己，于是 `state` 再加一，这就是可重入。线程乙此时调用 `tryAcquire` 会失败，随后被模板方法放进队列。

甲必须 `unlock` 同样的次数，`state` 才能减回 0。减到 0 才清除拥有者并唤醒队列头。少释放一次，锁就永远占着；多释放一次，则会因为当前线程并不是拥有者而抛出监视器状态异常。

### 倒计时门闩的 state

`CountDownLatch` 把 `state` 设成参与任务数 N。每个任务结束时 `countDown`，内部用 CAS 把 `state` 减 1。减到 0 时，把正在 `await` 上等待的主线程 `unpark`。主线程从 `await` 返回，继续做汇总。这里的数字表示“还差几次”，任何线程都可以减它，不存在单一拥有者。

## 容易写错的地方

自定义同步器多数只走独占，或只走共享，实现对应的一对钩子即可。也允许两套一起提供，读写锁就是例子：写侧独占，读侧共享。

等待时的中断语义不要混用。

- `acquire` 和 `acquireShared` 在队列中等待时忽略中断，中断标记最多留到获取成功之后。
- `acquireInterruptibly` 和 `acquireSharedInterruptibly` 在等待中发现中断会立刻撤出。

业务如果要把“取消等待”当成正常控制流，必须走可中断入口。另外，`tryRelease` 里不要在已经把 `state` 写成 0 之后再调用父类的释放逻辑，父类默认实现并不知道你的资源语义，重复释放会让状态和队列不一致。释放是否成功，应由子类自己根据这次状态变化返回。
