# 阻塞队列

## 概念

### 队列

队列是一头进、一头出的结构。先放进去的元素先被取走，和窗口排队同一个顺序。

### 阻塞队列

`BlockingQueue` 在普通队列上加了两种等待：

```mermaid
flowchart LR
    Producer[生产者线程] -->|放入| Q[阻塞队列]
    Q -->|取出| Consumer[消费者线程]
```

- 队列是空的，取元素的线程会停住，直到有人放进新元素
  - 柜子空了，就不能再从柜子里取
- 队列是满的，放元素的线程会停住，直到有人取走元素
  - 柜子满了，就不能再往里放

取的线程不会空转着反复判断，放的线程也不会在满队列上忙等。条件满足后，它们会被队列叫醒。

## 为什么要用？

餐厅满座时，后来的客人在候位区等。候位不是故障，而是把暂时处理不了的需求留在系统里。多线程里的阻塞也是这样：条件不成立就把线程挂起，条件成立再唤醒。

### 为什么需要 BlockingQueue

调用方不用自己写“满了就 wait、空了就 wait、改完再 notify”。这些都在队列实现里。`java.util.concurrent` 出现之前，每个写生产者和消费者的人都要自己把等待、唤醒和锁配好，还得避免虚假唤醒，代码容易错，也容易写得很慢。

## 架构

它和 `List` 的关系类似：`List` 是接口，`ArrayList` 是一种实现；`BlockingQueue` 是接口，下面有七个常用实现。

- `ArrayBlockingQueue`：数组实现，容量在构造时固定
- `LinkedBlockingQueue`：链表实现，容量可以指定；不指定时是 `Integer.MAX_VALUE`，大到可以当成无界，堆积起来会撑满堆
- `PriorityBlockingQueue`：无界，按优先级出队
- `DelayQueue`：元素到期才能被取走，内部用优先级队列
- `SynchronousQueue`：不保存元素。一次放入必须等一次取出配对，容量可以看成 0
- `LinkedTransferQueue`：链表，无界，还带 `transfer`，可以等消费者接手
- `LinkedBlockingDeque`：双向，头尾都能放、都能取

面试里先把前两个和 `SynchronousQueue` 说清楚。线程池的有界队列常用 `ArrayBlockingQueue`，工厂方法里的固定池用了近乎无界的 `LinkedBlockingQueue`，缓存池用的是 `SynchronousQueue`。

## BlockingQueue 核心方法

四组方法对“满”和“空”的态度不同：

| 组别 | 放入 | 取出 | 满或空时 |
| --- | --- | --- | --- |
| 抛异常 | add | remove | 满则 IllegalStateException，空则 NoSuchElementException |
| 返回布尔或 null | offer | poll | 满返回 false，空返回 null |
| 一直阻塞 | put | take | 满则放的线程等待，空则取的线程等待，直到可以继续或被中断 |
| 超时 | offer(e, time, unit) | poll(time, unit) | 等到超时仍不行，放入返回 false，取出返回 null |

### 抛出异常组

容量为 3 的数组队列，第四次 `add` 会失败：

```java
BlockingQueue<String> queue = new ArrayBlockingQueue<>(3);
System.out.println(queue.add("red"));
System.out.println(queue.add("green"));
System.out.println(queue.add("blue"));
System.out.println(queue.add("gold"));
```

```text
true
true
true
Exception in thread "main" java.lang.IllegalStateException: Queue full
	at java.base/java.util.AbstractQueue.add(AbstractQueue.java:98)
	at java.base/java.util.concurrent.ArrayBlockingQueue.add(ArrayBlockingQueue.java:329)
```

只放了三个再 `remove` 四次，第四次抛 `NoSuchElementException`：

```java
BlockingQueue<String> queue = new ArrayBlockingQueue<>(3);
queue.add("red");
queue.add("green");
queue.add("blue");
System.out.println(queue.remove());
System.out.println(queue.remove());
System.out.println(queue.remove());
System.out.println(queue.remove());
```

```text
red
green
blue
Exception in thread "main" java.util.NoSuchElementException
```

`element` 和 `peek` 都只看队头不删除。前者在空队列上抛异常，后者返回 null。

### 布尔类型组

`offer` 在满的时候返回 false，不抛异常。`poll` 在空的时候返回 null。

```java
BlockingQueue<String> queue = new ArrayBlockingQueue<>(3);
System.out.println(queue.offer("red"));
System.out.println(queue.offer("green"));
System.out.println(queue.offer("blue"));
System.out.println(queue.offer("gold"));
System.out.println(queue.poll());
System.out.println(queue.poll());
System.out.println(queue.poll());
System.out.println(queue.poll());
```

```text
true
true
true
false
red
green
blue
null
```

调用方必须看返回值。不看 `offer` 的 false，任务就静静地丢了。

### 阻塞队列组

`put` 在队列满时挂起当前线程，直到有位置，或者线程被中断。`take` 在队列空时挂起，直到有元素。消息投递若要求“放不进去就不能假装成功”，用这一组，而不是用返回 false 的 `offer`。

```java
BlockingQueue<String> queue = new ArrayBlockingQueue<>(3);
queue.put("red");
queue.put("green");
queue.put("blue");
System.out.println("三格已满，再 put 会停住");
queue.take();
queue.take();
queue.take();
// 再 take 一次，当前线程会停在这里，直到别的线程 put
```

不要在单线程演示里对满队列 `put` 或对空队列 `take`，那条线程会一直停着，后面的代码没有机会放或取。

### 不见不散组

带超时的 `offer` 和 `poll` 会等到指定时间。时间到了还放不进，返回 false；还取不到，返回 null。

```java
BlockingQueue<String> queue = new ArrayBlockingQueue<>(3);
System.out.println(queue.offer("red", 1L, TimeUnit.SECONDS));
System.out.println(queue.offer("green", 1L, TimeUnit.SECONDS));
System.out.println(queue.offer("blue", 1L, TimeUnit.SECONDS));
System.out.println(queue.offer("gold", 1L, TimeUnit.SECONDS));
System.out.println(queue.poll(1L, TimeUnit.SECONDS));
System.out.println(queue.poll(1L, TimeUnit.SECONDS));
System.out.println(queue.poll(1L, TimeUnit.SECONDS));
System.out.println(queue.poll(1L, TimeUnit.SECONDS));
```

第四次放入会卡大约 1 秒再返回 false。第四次取出会卡大约 1 秒再返回 null。超时让线程能够回到自己的循环里做别的事，而不是无限等下去。

## SynchronousQueue

`SynchronousQueue` 没有存储格。一次 `put` 必须等到另一次 `take` 把元素接走，否则放入方停住。它适合“交接”而不是“缓冲”。`Executors.newCachedThreadPool` 用它，是为了有空闲线程就直接交给它，没有就新建线程，而不是把任务堆在队列里。

下面生产者依次交出 A、B、C，消费者每次取走之前先等 1 秒。生产者的下一次 `put` 会卡在上一次还没被取走的时候。

```java
import java.util.concurrent.BlockingQueue;
import java.util.concurrent.SynchronousQueue;
import java.util.concurrent.TimeUnit;

/**
 * 不存储。put 和 take 必须配对。
 */
public class HandoffDemo {
    public static void main(String[] args) {
        BlockingQueue<String> handoff = new SynchronousQueue<>();

        new Thread(() -> {
            try {
                System.out.println(Thread.currentThread().getName() + " 交出 A");
                handoff.put("A");
                System.out.println(Thread.currentThread().getName() + " 交出 B");
                handoff.put("B");
                System.out.println(Thread.currentThread().getName() + " 交出 C");
                handoff.put("C");
            } catch (InterruptedException ex) {
                Thread.currentThread().interrupt();
            }
        }, "producer").start();

        new Thread(() -> {
            try {
                for (String token : new String[]{"A", "B", "C"}) {
                    TimeUnit.SECONDS.sleep(1);
                    handoff.take();
                    System.out.println(Thread.currentThread().getName() + " 接走 " + token);
                }
            } catch (InterruptedException ex) {
                Thread.currentThread().interrupt();
            }
        }, "consumer").start();
    }
}
```

输出会一对一对出现：

```text
producer 交出 A
consumer 接走 A
producer 交出 B
consumer 接走 B
producer 交出 C
consumer 接走 C
```

“交出 B”不会出现在“接走 A”之前。生产者打印了“交出 B”之后，要等消费者 `take` 返回，才能继续打印“交出 C”。这就是零容量交接。

## 阻塞队列的用处

### 生产者消费者模式

题目：一个从 0 开始的整数，两条线程交替加一和减一，做 4 轮。加的线程只允许在值为 0 时加，减的线程只允许在值为 1 时减。

写这种代码先记三条：

- 线程操作资源类，不要把锁散落在业务线程里
- 先判断，再修改，再通知
- 判断用 `while`，防止虚假唤醒

资源类自己拿着锁和条件：

```java
class Counter {
    private int value = 0;
    private final Lock lock = new ReentrantLock();
    private final Condition changed = lock.newCondition();

    public void increment() throws InterruptedException {
        lock.lock();
        try {
            while (value != 0) {
                changed.await();
            }
            value++;
            System.out.println(Thread.currentThread().getName() + " -> " + value);
            changed.signalAll();
        } finally {
            lock.unlock();
        }
    }
}
```

`await` 会放开锁并进入条件队列。被唤醒后要重新抢锁，然后再次判断 `value`。若这里写成 `if`，线程被唤醒后不再检查，可能在值已经是 1 时又加一次。虚假唤醒和 `signalAll` 叫醒了不该行动的那一方，都会让 `if` 出错。`while` 把这两种情况都挡回去。

减法和加法对称，只是等待条件换成 `value == 0`：

```java
public void decrement() throws InterruptedException {
    lock.lock();
    try {
        while (value == 0) {
            changed.await();
        }
        value--;
        System.out.println(Thread.currentThread().getName() + " -> " + value);
        changed.signalAll();
    } finally {
        lock.unlock();
    }
}
```

完整程序：

```java
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

/**
 * 传统版交替加减。判断必须是 while。
 */
class Counter {
    private int value = 0;
    private final Lock lock = new ReentrantLock();
    private final Condition changed = lock.newCondition();

    public void increment() throws InterruptedException {
        lock.lock();
        try {
            while (value != 0) {
                changed.await();
            }
            value++;
            System.out.println(Thread.currentThread().getName() + " -> " + value);
            changed.signalAll();
        } finally {
            lock.unlock();
        }
    }

    public void decrement() throws InterruptedException {
        lock.lock();
        try {
            while (value == 0) {
                changed.await();
            }
            value--;
            System.out.println(Thread.currentThread().getName() + " -> " + value);
            changed.signalAll();
        } finally {
            lock.unlock();
        }
    }
}

public class AlternateDemo {
    public static void main(String[] args) {
        Counter counter = new Counter();
        new Thread(() -> {
            for (int i = 0; i < 4; i++) {
                try {
                    counter.increment();
                } catch (InterruptedException ex) {
                    Thread.currentThread().interrupt();
                }
            }
        }, "plus").start();
        new Thread(() -> {
            for (int i = 0; i < 4; i++) {
                try {
                    counter.decrement();
                } catch (InterruptedException ex) {
                    Thread.currentThread().interrupt();
                }
            }
        }, "minus").start();
    }
}
```

输出是严格交替的：

```text
plus -> 1
minus -> 0
plus -> 1
minus -> 0
plus -> 1
minus -> 0
plus -> 1
minus -> 0
```

锁和条件都在资源类里面，两个线程只负责调用。这就是把同步细节收拢，而不是让每条业务线程自己 `synchronized`。

## 生产者和消费者：阻塞队列版

并发包把等待和唤醒收进队列之后，资源类只需要 `offer` 和 `poll`。下面用一个可关闭的开关、一个原子序号，以及调用方注入的队列。队列的具体类型由外面传入，资源类不依赖某一种实现。

`volatile` 用在开关上，是为了让生产者和消费者立刻看见“可以停了”。序号用 `AtomicInteger`，避免 `number++` 在多生产者时丢数。放入和取出都带超时，这样停止之后线程不会永远卡在队列上。

```java
import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.BlockingQueue;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicInteger;

/**
 * 队列由构造器注入。停止时只改开关，线程靠超时从 offer / poll 里出来。
 */
class Pipeline {
    private volatile boolean running = true;
    private final AtomicInteger sequence = new AtomicInteger();
    private final BlockingQueue<String> queue;

    public Pipeline(BlockingQueue<String> queue) {
        this.queue = queue;
        System.out.println(queue.getClass().getName());
    }

    public void produce() throws InterruptedException {
        while (running) {
            String data = String.valueOf(sequence.incrementAndGet());
            boolean accepted = queue.offer(data, 1L, TimeUnit.SECONDS);
            System.out.println(Thread.currentThread().getName()
                    + (accepted ? " 放入 " + data : " 放入超时 " + data));
            TimeUnit.MILLISECONDS.sleep(400);
        }
        System.out.println(Thread.currentThread().getName() + " 停止生产");
    }

    public void consume() throws InterruptedException {
        while (running) {
            String data = queue.poll(1L, TimeUnit.SECONDS);
            if (data == null) {
                if (!running) {
                    System.out.println(Thread.currentThread().getName() + " 队列已空，退出");
                    return;
                }
                continue;
            }
            System.out.println(Thread.currentThread().getName() + " 取到 " + data);
        }
    }

    public void stop() {
        running = false;
    }
}

public class PipelineDemo {
    public static void main(String[] args) throws InterruptedException {
        Pipeline pipeline = new Pipeline(new ArrayBlockingQueue<>(8));
        new Thread(() -> {
            try {
                pipeline.produce();
            } catch (InterruptedException ex) {
                Thread.currentThread().interrupt();
            }
        }, "producer").start();
        new Thread(() -> {
            try {
                pipeline.consume();
            } catch (InterruptedException ex) {
                Thread.currentThread().interrupt();
            }
        }, "consumer").start();

        TimeUnit.SECONDS.sleep(3);
        System.out.println("3 秒后关闭");
        pipeline.stop();
    }
}
```

一种可能的输出：

```text
java.util.concurrent.ArrayBlockingQueue
producer 放入 1
consumer 取到 1
producer 放入 2
consumer 取到 2
producer 放入 3
consumer 取到 3
3 秒后关闭
producer 停止生产
consumer 队列已空，退出
```

开关被写成 false 之后，生产者离开 `while`。消费者若正堵在 `poll` 上，最迟 1 秒返回 null，再看到开关已关就退出。不要在 `running == false` 时还调用不带超时的 `take`，那会在队列刚好空的时候永久等待。

循环条件用 `while (running)`，不要用 `if`。`stop` 可能发生在检查和等待之间，醒来之后需要再看一次开关。队列版本把“满了怎么办、空了怎么办”交给 `BlockingQueue`，业务代码只处理序号和停止信号。
