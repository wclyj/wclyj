# CAS 底层原理

## 概念

CAS 是 Compare-And-Swap，一条 CPU 级并发原语。

它做的事很窄：看某个内存位置是不是仍然等于预期值，是就把新值写进去。比较和写入合成一步，中间不会被别的线程插进来。

Java 里这组原语出现在 `jdk.internal.misc.Unsafe`（早期 JDK 是 `sun.misc.Unsafe`）的本地方法上。JVM 把它们编译成平台的 CAS 指令，例如 x86 的 `lock cmpxchg`。原语在操作系统里指一小段必须连续做完、不允许中途打断的操作。CAS 就是这样一条原子指令，不会出现“比较成功后、写入前被别人改掉”的撕裂，因此单变量的 CAS 本身是线程安全的。

## 代码使用

先建一个初值为 8 的原子整数：

```java
AtomicInteger counter = new AtomicInteger(8);
```

第一次 CAS 期望内存仍是 8，想把它改成 30：

```java
counter.compareAndSet(8, 30)
```

紧接着再试一次，还是期望 8，想改成 40。此时内存已经是 30，这次会失败：

```java
counter.compareAndSet(8, 40)
```

完整代码：

```java
import java.util.concurrent.atomic.AtomicInteger;

/**
 * compareAndSet：预期值等于当前值才替换。
 */
public class CompareSwapDemo {
    public static void main(String[] args) {
        AtomicInteger counter = new AtomicInteger(8);

        // 预期 8，更新为 30。当前确实是 8，返回 true
        System.out.println(counter.compareAndSet(8, 30) + "\t now: " + counter.get());

        // 内存已是 30，预期仍写 8，返回 false，值不变
        System.out.println(counter.compareAndSet(8, 40) + "\t now: " + counter.get());
    }
}
```

输出：

```text
true	 now: 30
false	 now: 30
```

第一次预期和内存一致，写入成功。第二次内存已经是 30，和预期 8 不符，方法返回 false，30 被保留。

这和版本库的提交类似：工作区基线没被别人推进，就可以直接提交；基线变了，就必须先重新拉取再合并，不能拿旧基线覆盖新提交。

## CAS 底层原理

`AtomicInteger.getAndIncrement()` 并不在 Java 层加锁，它把自增委托给 Unsafe 的 `getAndAddInt`。

### 1、unsafe 类

Java 不能直接发 CPU 指令，需要本地方法这条通道。Unsafe 就是这条通道的入口，可以按地址读写内存，能力接近 C 的指针。原子类的 CAS 全部落在它的本地方法上。

这些方法都带 `native`，真正的比较交换在 JVM 的 C++ 实现里完成，而不是在 Java 字节码里用锁模拟。

原子包装类能保证单个变量的原子更新，靠的就是 Unsafe 加上 CPU 的原子指令，不是靠 `synchronized`。

### 2、变量 valueOffset

`valueOffset` 是字段 `value` 在对象头之后的字节偏移。Unsafe 不走字段访问指令，而是用“对象地址 + 偏移”定位到那个 int。

自增时先按这个偏移把旧值读出来，再算出旧值加 1，最后对同一偏移做 CAS。

### 3、变量 value 用 volatile 修饰

`value` 声明为 `volatile`，读写都会走缓存一致性协议，线程不会一直拿着自己核心上的过期副本。

`getAndAddInt` 的循环可以分成几步理解（不同 JDK 的局部变量名不同，角色不变）：

- 对象本身
- `value` 的偏移
- 要加上的增量，自增时是 1
- 按对象和偏移读到的当前内存值

每次循环都重新读内存里的最新值，再调用比较并交换：

- 内存值仍等于刚读到的那个数，就写成“该数 + 增量”，返回成功
- 不相等，说明刚被别的线程写过，回到循环开头再读

这里没有 `synchronized`。每个线程在自己的循环里读最新值、尝试写入，失败就重试。单变量上既有原子性，又避免了重量级锁把所有线程串行化。

假设甲、乙同时对初值为 6 的 `AtomicInteger` 做自增，跑在两颗核上：

1. 主内存里的 value 是 6。按 JMM，两个人各自读到 6。
2. 甲读到 6 后被换下 CPU。
3. 乙也读到 6，并且 CAS 成功，内存变成 7。
4. 甲恢复后拿手里的 6 去比，内存已经是 7，本次 CAS 失败，回到循环。
5. `value` 是 volatile 的，甲重新读到 7，再尝试把 7 换成 8，直到成功。

Unsafe 加上这段循环，就是自旋：失败不阻塞，只是再转一圈。

## 底层汇编

`compareAndSet` 最终进入 HotSpot 的 unsafe 实现，核心步骤是：

- 由对象和偏移算出 `value` 的内存地址
- 调用平台相关的 `cmpxchg`：参数里一个是期望的旧值，一个是准备写入的新值
- 前面带锁前缀，保证这条读改写在总线上原子完成

## CAS 缺点

CAS 不加互斥锁，单次比较交换是原子的，但失败后要反复重试。

- 竞争激烈时循环停不下来。最坏情况是某个线程每次读到的值都被别人改掉，空转占用 CPU。
- 一次 CAS 只覆盖一个内存位置。一个共享变量可以靠循环 CAS 做原子更新；两个共享变量要一起成功或一起失败时，单次 CAS 保证不了，需要锁，或者改成把两个字段收进一个不可变对象再整体替换。
- 只比较头尾，会漏掉中间被改回去的情况，也就是 ABA。

## ABA 问题

内存从 A 变成 B 再变回 A，CAS 只看见头尾都是 A，会认为没人动过。若这中间的 B 代表一次出队或一次状态切换，逻辑就已经错了。解决办法是给值再配一个版本号，每次修改都让版本加一，比较时值和版本一起看。

## 总结

### CAS

比较工作内存里准备使用的旧值和主内存当前值。相同就写入新值；不同就再读再比，直到某一次旧值和主内存一致。

### CAS 应用

三个操作数：内存位置 V，预期旧值 A，新值 B。仅当 V 当前等于 A 时，把 V 改成 B，否则什么也不写，并告诉调用方失败。
