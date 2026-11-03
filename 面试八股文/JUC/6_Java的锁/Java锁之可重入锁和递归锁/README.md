# 可重入锁和递归锁 ReentrantLock

## 概念

可重入锁也叫递归锁。

同一个线程在外层已经拿到锁之后，内层方法再次请求这把锁，应该直接成功，而不是把自己堵在门外。锁内部用持有线程加一个重入次数来识别“是不是自己人”。

也就是：线程可以进入任何一段已经被自己持有的同步代码。

`synchronized` 和 `ReentrantLock` 都是可重入的。

## 代码

外层方法和内层方法如果锁的是同一个对象，外层进来之后，内层不必再向别的线程申请：

```java
public synchronized void outer() {
    inner();
}

public synchronized void inner() {
}
```

线程一旦用这台设备的监视器进入 `outer`，再调用 `inner` 时监视器的重入计数从 1 变成 2，不会死锁。两个方法都返回后，计数回到 0，别的线程才能进。

## 作用

最大的用处是避免“自己等自己”。对象方法互相调用、或者递归往下走时，如果锁不可重入，线程会在内层永远等外层把锁放开，而外层又在等内层返回，形成自死锁。

## 可重入锁验证

### 证明 Synchronized

```java
/**
 * 同一对象上的两道同步方法。外层调用内层，不应把自己卡住。
 */
class Radio {
    public synchronized void sendVoice() {
        System.out.println(Thread.currentThread().getName() + " 进入语音");
        sendText();
    }

    public synchronized void sendText() {
        System.out.println(Thread.currentThread().getName() + " 进入文字");
    }
}

public class ReentrySyncDemo {
    public static void main(String[] args) {
        Radio radio = new Radio();
        new Thread(() -> radio.sendVoice(), "crew-1").start();
        new Thread(() -> radio.sendVoice(), "crew-2").start();
    }
}
```

输出顺序是成对的：

```text
crew-1 进入语音
crew-1 进入文字
crew-2 进入语音
crew-2 进入文字
```

`crew-1` 在外层拿到 `radio` 的监视器后，内层 `sendText` 直接成功。`crew-2` 在整个一对调用结束前进不来。若锁不可重入，`crew-1` 会停在 `sendText`，`crew-2` 停在 `sendVoice`，谁都打印不完。

```text
crew-1 进入语音    外层第一次获得监视器
crew-1 进入文字    内层重入，计数加一
crew-2 进入语音    前一个线程完全退出后才获得
crew-2 进入文字    同样重入
```

### 证明 ReentrantLock

```java
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

/**
 * 显式锁的重入：外层 lock 之后再 lock 一次，必须对应同样次数的 unlock。
 */
class Gate implements Runnable {
    private final Lock lock = new ReentrantLock();

    public void enter() {
        lock.lock();
        try {
            System.out.println(Thread.currentThread().getName() + " 外层持锁");
            passInner();
        } finally {
            lock.unlock();
        }
    }

    public void passInner() {
        lock.lock();
        try {
            System.out.println(Thread.currentThread().getName() + " 内层持锁");
        } finally {
            lock.unlock();
        }
    }

    @Override
    public void run() {
        enter();
    }
}

public class ReentryLockDemo {
    public static void main(String[] args) {
        Gate gate = new Gate();
        new Thread(gate, "crew-3").start();
        new Thread(gate, "crew-4").start();
    }
}
```

`enter` 先 `lock`，再调用同样加锁的 `passInner`。持有者是当前线程，所以第二次 `lock` 只把计数加一。

输出：

```text
crew-3 外层持锁
crew-3 内层持锁
crew-4 外层持锁
crew-4 内层持锁
```

和 `synchronized` 一样，外层成功之后内层不会被自己挡住。`crew-3` 和 `crew-4` 谁先开始由调度决定，但每个人的两行不会被对方拆开。

**外层连续 lock 两次，会怎样？**

```java
public void enter() {
    lock.lock();
    lock.lock();
    try {
        System.out.println(Thread.currentThread().getName() + " 外层持锁");
        passInner();
    } finally {
        lock.unlock();
        lock.unlock();
    }
}
```

结果仍然成对打印。两次 `lock` 只是让计数变成 2，再加内层那一次是 3；`finally` 里两次 `unlock` 加上内层的一次，计数回到 0。钥匙还是这一把，多转几圈只改计数。

**连续 lock 两次，但只 unlock 一次，会怎样？**

若 `finally` 里少一次 `unlock`，当前线程离开 `enter` 时计数仍大于 0。另一个线程的 `lock` 永远等不到 0。程序停在第一条线程打完自己的两行之后，第二条线程没有下文。申请几次就要释放几次，次数必须成对。

**只 lock 一次，却 unlock 两次，会怎样？**

```java
public void enter() {
    lock.lock();
    try {
        System.out.println(Thread.currentThread().getName() + " 外层持锁");
        passInner();
    } finally {
        lock.unlock();
        lock.unlock();
    }
}
```

内层自己会做一对 lock/unlock。外层多出来的第二次 `unlock` 发现当前线程已经不是持有者（计数已经是 0），于是抛出：

```text
crew-3 外层持锁
crew-3 内层持锁
crew-4 外层持锁
crew-4 内层持锁
Exception in thread "crew-3" java.lang.IllegalMonitorStateException
    at java.base/java.util.concurrent.locks.ReentrantLock$Sync.tryRelease(ReentrantLock.java:175)
    at java.base/java.util.concurrent.locks.AbstractQueuedSynchronizer.release(AbstractQueuedSynchronizer.java:1007)
    at java.base/java.util.concurrent.locks.ReentrantLock.unlock(ReentrantLock.java:494)
    at Gate.enter(ReentryLockDemo.java:18)
```

`IllegalMonitorStateException` 的含义是：你在释放一把当前并不由你持有的锁。多解锁和少解锁都是错，只是一个立刻抛异常，一个把别的线程永久堵住。
