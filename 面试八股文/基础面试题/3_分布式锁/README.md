# 分布式锁

## 集群环境下保证ID不重复

`synchronized` 和进程内的 `Lock` 只能管住同一台虚拟机。两台机器各有一把这样的锁时，它们互相看不见。

集群把“很多调用方”对上“很多进程”。如果每个进程自己用本地自增或随机串当编号，并发一高就可能撞号。要在进程之间互斥，锁必须放到大家都能看见的外部系统里，这才是分布式锁。编号服务本身也可以不靠锁，而靠分段发号，这篇先把锁的路讲清楚。

流量不大时，常见三种落地：

- 数据库里用版本号做乐观控制
- 用缓存的现成锁实现，例如 Redisson
- 用协调服务的临时结点，顺带还能做注册

乐观锁适合冲突少的更新：读出版本，写回时带上“版本仍是刚才那个”的条件，失败就重试。冲突剧烈时重试会打满数据库。缓存锁吞吐更高，但要处理持有者宕机后的过期。协调服务的锁依赖会话，会话一断，锁自动消失。

## Zookeeper实现分布式锁

协调服务保证同一路径只能有一个结点。抢锁就是去创建那个结点：创建成功的人持有锁，用完把结点删掉。

```text
create /ticket/lock                  持久结点，进程退出还在
create -e /ticket/lock owner-7       临时结点，会话结束就被清掉
```

单进程里继续用内置互斥即可。跨进程才值得付一次网络往返。

### 创建锁

多台机器同时创建同一个临时路径，只有一个创建请求会成功，成功者进入临界区。没创建成功的一方不要空转重试，而是在这个路径上登记监视。持有者删除结点或会话掉线后，服务端发出通知，等待者再抢一次。

直接监听同一个固定路径会有“惊群”：锁一释放，所有等待者一起醒来。更稳的做法是每人创建一个带序号的临时结点，只监视比自己小的前一个序号。前一个消失，才轮到自己，这样一次只叫醒一个人。

### 释放锁

释放有两条路。主动删除结点，后面的监视者立刻收到事件。如果进程崩溃，来不及删除，临时结点会随会话一起消失，效果和主动释放一样。所以锁的租约要和会话绑在一起，不要建成持久结点，否则宕机之后锁永远留在那里。

等待者被叫醒之后不能假设锁已经属于自己，必须回到“尝试创建或比较序号”的步骤。通知只表示“可以再试”，不表示“你已经拿到”。

## 单机环境下的锁

先看一把只在进程内会计数的发号器。它故意不加保护，用来暴露丢失更新。

```java
public class TicketSeq {
    private int current = 0;

    public String next() {
        return "票号-" + (++current);
    }
}
```

业务对象包一层：

```java
public class TicketService {
    private final TicketSeq seq = new TicketSeq();

    public String issue() {
        return seq.next();
    }
}
```

二十条线程一起取号：

```java
public class LocalClient {
    public static void main(String[] args) {
        TicketService service = new TicketService();
        for (int i = 0; i < 20; i++) {
            new Thread(() -> System.out.println(service.issue()), "t" + i).start();
        }
    }
}
```

`++current` 不是原子的，读和写之间会插入别的线程，于是同一个票号可能打印两次，末尾的号码还会小于 20。线程越多越容易看见。

进程内的修法是同一把可重入锁包住取号。锁要作为成员，不能在方法里现造，否则每个调用各持一把，等于没锁。

```java
private final Lock lock = new ReentrantLock();

public String issue() {
    lock.lock();
    try {
        return seq.next();
    } finally {
        lock.unlock();
    }
}
```

`unlock` 放在 `finally`，发号抛错时也不会把锁留在当前线程上。这把锁仍然只对这一个服务对象有效。上面的客户端如果每次 `new TicketService()`，锁又分开了。单机演示要共享同一个服务实例。

## 分布式环境下的锁

### 模板模式

#### 概念

模板方法把算法骨架写在父类里，把会变化的步骤留成抽象方法。子类填步骤，但不能改骨架的调用顺序。

它要解决的是：抢锁、失败则等待、被叫醒后再抢、用完释放，这几步在各种存储上都一样；“怎样试一次”“怎样睡觉”则取决于用的是结点、缓存键还是数据库行。

#### 应用实例

- 一套固定流程里，每一关的具体内容不同，但关卡顺序不能打乱
- 持久层模板把打开会话、开始事务、提交和关闭写死，业务只提供真正要保存的对象

#### 优缺点

- 不变的流程只写一次，变化点仍可扩展
- 公共步骤集中，改一处即可
- 顺序由父类掌控，子类不容易把释放忘掉
- 每一种存储都要一个子类，类型会变多

#### 使用场景

- 多个子类的主流程相同，只有一两步不同
- 这几步如果写错顺序会出事故，例如忘记释放

### 代码

先定锁的两个动作。

```java
public interface DistLock {
    void acquire();
    void release();
}
```

骨架放在抽象类里。抢到就返回；没抢到就等待，然后从头再来。释放留给子类，因为不同存储的删除方式不同，但调用点固定在 `finally`。

```java
public abstract class LockSkeleton implements DistLock {
    @Override
    public final void acquire() {
        if (tryOnce()) {
            System.out.println(Thread.currentThread().getName() + " 拿到名额");
            return;
        }
        waitTurn();
        acquire();
    }

    protected abstract boolean tryOnce();

    protected abstract void waitTurn();

    @Override
    public void release() {
    }
}
```

递归再次进入 `acquire` 时要有退出条件，否则通知丢失会把栈撑满。生产实现更常见的是循环，并给等待加上超时。

协调服务的子类只填两步。示例里用布尔值代替真实客户端，方便看清控制流；真正接入时，`tryOnce` 去创建临时结点，`waitTurn` 在监视上阻塞。

```java
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.atomic.AtomicBoolean;

public class NodeLock extends LockSkeleton {
    private final AtomicBoolean held = new AtomicBoolean(false);
    private CountDownLatch gate = new CountDownLatch(1);

    @Override
    protected boolean tryOnce() {
        return held.compareAndSet(false, true);
    }

    @Override
    protected void waitTurn() {
        try {
            gate.await();
        } catch (InterruptedException ex) {
            Thread.currentThread().interrupt();
        }
    }

    @Override
    public void release() {
        held.set(false);
        CountDownLatch previous = gate;
        gate = new CountDownLatch(1);
        previous.countDown();
    }
}
```

业务侧无论发号是否抛错，都要释放。每个线程可以有自己的服务对象，但锁必须是这些对象共享的那一个，否则又退回进程内各锁各的。

```java
public class ClusterTicketService {
    private final TicketSeq seq = new TicketSeq();
    private final DistLock lock;

    public ClusterTicketService(DistLock lock) {
        this.lock = lock;
    }

    public String issue() {
        lock.acquire();
        try {
            return seq.next();
        } finally {
            lock.release();
        }
    }
}
```

客户端共享同一把锁，再开多条线程。票号应当不重复，并且从 1 覆盖到线程数。

```java
public class ClusterClient {
    public static void main(String[] args) {
        DistLock lock = new NodeLock();
        for (int i = 0; i < 20; i++) {
            new Thread(() -> {
                String no = new ClusterTicketService(lock).issue();
                System.out.println(no);
            }, "w" + i).start();
        }
    }
}
```

这把示例锁还在同一个进程里。把它换成真正的临时结点之后，多台机器才能互斥。换存储时只新增子类，`issue` 里的加锁和释放顺序保持不动，这就是模板方法要保住的部分。

使用时还有三个坑：等待被中断后要把中断标记设回去；释放必须由持有者来做，不能谁醒来谁删除；锁粒度要盖住“取号并保存”的整段，只锁取号、外面再写数据库，仍可能插入重复行。
