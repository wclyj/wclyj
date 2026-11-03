# CountDownLatch

## 概念

它让一个或多个线程先停住，直到另外一批线程把约定的动作做完，再一起被叫醒。

核心只有两个动作。调用 `await` 的线程会停在闩上；其他线程每完成一步就调用 `countDown`，计数减一，调用方自己并不阻塞。计数落到 0 时，所有停在 `await` 上的线程恢复执行。

## 场景

流水线末尾有一道质检。前面 5 个工位各自做完自己的零件，质检员必须等 5 份都交齐才能盖章放行。工位之间谁先做完无所谓，质检这一步必须排在最后。

## 解决方案

准备一个初始值为 5 的闩，和工位数量对齐：

```java
// 还差 5 个工位交件
CountDownLatch gate = new CountDownLatch(5);
```

每个工位线程结束后把计数减一：

```java
for (int station = 1; station <= 5; station++) {
    new Thread(() -> {
        System.out.println(Thread.currentThread().getName() + " 零件已交出");
        gate.countDown();
    }, "station-" + station).start();
}
```

质检线程用 `await` 当屏障。计数没到 0 之前，这行后面的代码不会跑：

```java
gate.await();
System.out.println(Thread.currentThread().getName() + " 五件齐了，开始盖章");
```

若主线程不等闩，盖章语句会插在工位打印中间，因为 `start` 只是把线程丢进调度器，主线程会继续往下走。

一种可能的提前打印：

```text
station-2 零件已交出
main 五件齐了，开始盖章
station-1 零件已交出
station-4 零件已交出
station-5 零件已交出
station-3 零件已交出
```

加上 `await` 之后，盖章一定出现在五条交件日志之后，五条日志彼此的先后仍由调度决定：

```text
station-3 零件已交出
station-1 零件已交出
station-5 零件已交出
station-2 零件已交出
station-4 零件已交出
main 五件齐了，开始盖章
```

## 完整代码

```java
import java.util.concurrent.CountDownLatch;

/**
 * 五个工位交件，主线程等齐再盖章。
 * 计数必须和真正会调用 countDown 的次数一致，否则 await 永不返回。
 */
public class LatchWorkshop {
    public static void main(String[] args) throws InterruptedException {
        CountDownLatch gate = new CountDownLatch(5);

        for (int station = 1; station <= 5; station++) {
            new Thread(() -> {
                System.out.println(Thread.currentThread().getName() + " 零件已交出");
                gate.countDown();
            }, "station-" + station).start();
        }

        gate.await();
        System.out.println(Thread.currentThread().getName() + " 五件齐了，开始盖章");
    }
}
```
