# Semaphore：信号量

## 概念

信号量同时承担两件事：

- 把一份共享资源收成互斥访问（许可数设为 1 时，效果接近互斥锁）
- 限制同一时刻能进入临界区的线程数量（许可数大于 1）

## 代码

用 4 台打印机服务 8 份报告。许可数是 4，表示同时只能有 4 份报告占用机器。第二个参数 `false` 表示非公平：后来的线程也可能先领到刚释放的许可。

```java
/**
 * 4 个许可，非公平发放，用来表示 4 台打印机
 */
Semaphore printers = new Semaphore(4, false);
```

每份报告先申请一个许可，领到才开始打印。`acquire` 在许可为 0 时会把当前线程挂起：

```java
// 领到一台空闲打印机
printers.acquire();
```

打印耗时用睡眠模拟，两秒后归还许可：

```java
try {
    TimeUnit.SECONDS.sleep(2);
} catch (InterruptedException ex) {
    Thread.currentThread().interrupt();
}
```

无论打印是否抛异常，都要在 `finally` 里归还，否则许可泄漏，后面的报告永远进不来：

```java
printers.release();
```

完整代码：

```java
import java.util.concurrent.Semaphore;
import java.util.concurrent.TimeUnit;

/**
 * 8 份报告抢 4 台打印机。
 */
public class PrinterPermitDemo {

    public static void main(String[] args) {
        Semaphore printers = new Semaphore(4, false);

        for (int report = 1; report <= 8; report++) {
            new Thread(() -> {
                try {
                    printers.acquire();
                    System.out.println(Thread.currentThread().getName() + " 开始打印");
                    try {
                        TimeUnit.SECONDS.sleep(2);
                    } catch (InterruptedException ex) {
                        Thread.currentThread().interrupt();
                    }
                    System.out.println(Thread.currentThread().getName() + " 打印结束");
                } catch (InterruptedException ex) {
                    Thread.currentThread().interrupt();
                } finally {
                    printers.release();
                }
            }, "report-" + report).start();
        }
    }
}
```

一种可能的输出（前四台先开工，两秒后归还许可，后四台再进入）：

```text
report-1 开始打印
report-3 开始打印
report-2 开始打印
report-4 开始打印
report-1 打印结束
report-2 打印结束
report-5 开始打印
report-4 打印结束
report-6 开始打印
report-3 打印结束
report-7 开始打印
report-8 开始打印
report-5 打印结束
report-6 打印结束
report-7 打印结束
report-8 打印结束
```

同时处于“开始打印”且还没“打印结束”的报告不会超过 4 份。这就是许可数对并发度的硬限制。
