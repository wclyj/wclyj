# CyclicBarrier

## 概念

它和 CountDownLatch 的计数方向相反。Latch 从 N 减到 0；Barrier 从 0 累加，凑满指定人数才放行，并且用完可以回到初始状态再凑下一轮，所以叫“可循环的屏障”。

一组线程各自跑到同步点后调用 `await` 停住。最后一个到达的线程会触发可选的回调，然后所有被拦住的线程一起继续。少一个人，屏障就不开。

## 案例

四名队员要在山口会合，人齐之后先由屏障回调打印“地图已展开”，然后每人继续下山。先到的人必须在 `await` 上等待。

```java
/**
 * 参数 1：需要到达的线程数
 * 参数 2：最后一名到达时先执行的动作
 */
CyclicBarrier ridge = new CyclicBarrier(4, () -> {
    System.out.println("地图已展开");
});
```

四名队员各跑一条线程，到达后调用 `await`：

```java
for (int member = 1; member <= 4; member++) {
    final int badge = member;
    new Thread(() -> {
        System.out.println(Thread.currentThread().getName() + " 抵达山口，编号 " + badge);
        try {
            // 未满员时停在这里，满员后全体返回
            ridge.await();
        } catch (InterruptedException ex) {
            Thread.currentThread().interrupt();
        } catch (BrokenBarrierException ex) {
            ex.printStackTrace();
        }
        System.out.println(Thread.currentThread().getName() + " 开始下山");
    }, "member-" + member).start();
}
```

完整代码：

```java
import java.util.concurrent.BrokenBarrierException;
import java.util.concurrent.CyclicBarrier;

/**
 * 四人在山口会合，齐了再一起离开。
 */
public class RidgeMeetDemo {

    public static void main(String[] args) {
        CyclicBarrier ridge = new CyclicBarrier(4, () -> {
            System.out.println("地图已展开");
        });

        for (int member = 1; member <= 4; member++) {
            final int badge = member;
            new Thread(() -> {
                System.out.println(Thread.currentThread().getName() + " 抵达山口，编号 " + badge);
                try {
                    ridge.await();
                } catch (InterruptedException ex) {
                    Thread.currentThread().interrupt();
                } catch (BrokenBarrierException ex) {
                    ex.printStackTrace();
                }
                System.out.println(Thread.currentThread().getName() + " 开始下山");
            }, "member-" + member).start();
        }
    }
}
```

若某个线程在等待中被中断，或者超时，屏障会进入 broken 状态，其余正在 `await` 的线程收到 `BrokenBarrierException`。下一轮要重新用，可以调用 `reset`，但已经在等待的线程会被打断，更稳妥的做法是新建一个屏障。
