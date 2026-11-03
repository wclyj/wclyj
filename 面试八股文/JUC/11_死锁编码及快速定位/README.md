# 死锁编码及定位分析

## 概念

两个或更多线程（或进程）各自握着对方下一步要用的资源，又都不放手，于是谁也走不下去。没有外力介入时，这种互相等待会一直保持。资源很充裕、申请总能马上满足时，死锁很少出现；资源有限又按不利顺序申请时，就容易卡死。

```mermaid
flowchart LR
    T1[线程甲 持有印台] -->|等待| Stamp[线程乙手里的名章]
    T2[线程乙 持有名章] -->|等待| Pad[线程甲手里的印台]
```

## 产生死锁的原因

- 资源总量不够，后面的申请必须等待
- 线程推进顺序正好形成环
- 分配策略允许“先拿一部分，再等另一部分”

## 死锁产生的四个必要条件

- 互斥
  - 缓解思路：能改成可同时读的资源就不要做成独占；必须独占的资源无法从这一条上拆掉
- 占有且等待
  - 缓解思路：申请前先释放已持有的资源，或者一次申明全部需求。后者简单，但会降低资源利用率
- 不可抢占
  - 缓解思路：申请失败时主动丢掉已经拿到的资源，稍后再整体重试
- 循环等待
  - 缓解思路：给资源编号，所有线程都按同一方向申请，环就无法形成

四个条件同时成立才会死锁，破坏任意一条就能从结构上避开。

## 死锁代码

两个线程各自先拿自己的字符串锁，睡一会儿，再去拿对方的锁。睡眠是为了让两边都先进入外层 `synchronized`，从而稳定复现环。

```java
import java.util.concurrent.TimeUnit;

/**
 * 两把锁交叉获取，用来稳定复现死锁。
 */
class CrossGrab implements Runnable {

    private final String first;
    private final String second;

    public CrossGrab(String first, String second) {
        this.first = first;
        this.second = second;
    }

    @Override
    public void run() {
        synchronized (first) {
            System.out.println(Thread.currentThread().getName()
                    + " 已拿 " + first + "，准备再拿 " + second);
            try {
                TimeUnit.SECONDS.sleep(2);
            } catch (InterruptedException ex) {
                Thread.currentThread().interrupt();
            }
            synchronized (second) {
                System.out.println(Thread.currentThread().getName()
                        + " 两把都拿到了");
            }
        }
    }
}

public class CrossLockDemo {
    public static void main(String[] args) {
        String ink = "ink-pad";
        String seal = "name-seal";
        new Thread(new CrossGrab(ink, seal), "clerk-a").start();
        new Thread(new CrossGrab(seal, ink), "clerk-b").start();
    }
}
```

运行后主线程不会正常结束，控制台通常停在：

```text
clerk-a 已拿 ink-pad，准备再拿 name-seal
clerk-b 已拿 name-seal，准备再拿 ink-pad
```

内层那句“两把都拿到了”不会出现。

## 如何排查死锁

先用 `jps` 找到仍在运行的 Java 进程：

```text
jps -l
```

输出里能看到 `CrossLockDemo` 的进程号，例如 `18432`。

再用 `jstack` 打出该进程的线程栈：

```text
jstack 18432
```

关键段落类似：

```text
Found one Java-level deadlock:
=============================
"clerk-b":
  waiting to lock monitor 0x00007f88ac10 (object 0x00000000d7a01080, a java.lang.String),
  which is held by "clerk-a"
"clerk-a":
  waiting to lock monitor 0x00007f88ac40 (object 0x00000000d7a010b8, a java.lang.String),
  which is held by "clerk-b"

Java stack information for the threads listed above:
===================================================
"clerk-b":
        at CrossGrab.run(CrossLockDemo.java:24)
        - waiting to lock <0x00000000d7a01080> (a java.lang.String)
        - locked <0x00000000d7a010b8> (a java.lang.String)
        at java.lang.Thread.run(Thread.java:840)
"clerk-a":
        at CrossGrab.run(CrossLockDemo.java:24)
        - waiting to lock <0x00000000d7a010b8> (a java.lang.String)
        - locked <0x00000000d7a01080> (a java.lang.String)
        at java.lang.Thread.run(Thread.java:840)

Found 1 deadlock.
```

末尾的 `Found 1 deadlock` 就是结论。往上看 `- waiting to lock` 和 `- locked`，能对上“甲等乙持有的对象，乙等甲持有的对象”。JDK 自带的 `jcmd <pid> Thread.print` 也能打出同样的死锁段落。
