# 堆

## 堆的核心概念

对一个虚拟机进程来说，堆是唯一的：一个进程里通常跑着一个 HotSpot 实例，这个实例里可以有很多线程，它们共享同一块堆。线程私有的是程序计数器、虚拟机栈和本地方法栈；堆不按线程拆成多份，只是在内部分出线程私有的分配缓冲。

一个虚拟机实例只有一块堆，它也是 Java 内存管理里最大的区域。堆在虚拟机启动时创建，初始容量和最大容量在启动参数里就定了，运行期间可以在这个区间里伸缩，但不会超过最大容量。

规范允许堆在物理上不连续，只要逻辑上能被看成连续的地址范围。实现可以用多段映射拼出这块逻辑空间。所有线程共享堆，同时还可以在 Eden 里为每条线程划出 TLAB，也就是 Thread Local Allocation Buffer，让日常的小对象分配不必抢全局指针。

启动时常见的两个开关是堆的下限和上限。例如把两者都设成 256MB：`-Xms256m` 与 `-Xmx256m`。前者是初始堆，后者是最大堆。

规范对堆的描述是：所有类实例和数组的内存都从堆上分配。从实现角度看，要加一个“几乎”。逃逸分析配合标量替换之后，有些对象根本不会以完整实例的形式出现在堆里。数组和普通对象的本体仍然不会直接放进栈帧；栈帧里保存的是引用，引用指向堆中的位置。

方法返回时，堆上的对象不会马上消失。栈帧弹出只是丢掉了栈上的那份引用，对象要等到变得不可达、并且收集器真的运行，才会被回收。若每次方法结束都立刻收集，用户线程会被频繁停住。堆因此是垃圾收集的重点区域。

### 堆内存细分

JDK 7 及以前，人们习惯把逻辑图画成三块：新生代、老年代、永久代。新生代再分成 Eden 和两块 Survivor。永久代用来实现方法区，它并不等同于“又一块放对象的堆”，只是 HotSpot 当时把类的元数据放在了堆的相邻区域里，并且受堆外的单独上限约束。

JDK 8 起，永久代被移除，方法区改由元空间实现，元空间使用本地内存，不属于 Java 堆。堆本身只剩下新生代和老年代。新生代仍是 Eden 加 Survivor。口头上的对应关系是：新生区、新生代、年轻代是同一块；养老区、老年区、老年代是同一块；永久区、永久代则是 JDK 7 及以前的方法区实现，不要再把它说成 JDK 8 堆的一部分。

所以 JDK 8 的图如果把元空间画在堆旁边，那是为了对照旧的永久代，不表示元空间仍由 `-Xmx` 管辖。

## 设置堆内存大小与 OOM

堆用来放 Java 对象实例，容量在启动时通过 `-Xms` 和 `-Xmx` 给出。

- `-Xms` 是初始堆，等价于 `-XX:InitialHeapSize`。
- `-Xmx` 是最大堆，等价于 `-XX:MaxHeapSize`。

一旦存活对象加上无法回收的占用超过 `-Xmx`，分配会失败，并抛出 `OutOfMemoryError`，信息通常是 Java heap space。

生产上常把 `-Xms` 和 `-Xmx` 设成同一个数。这样堆不必在收集之后重新向操作系统申请或归还页面，也避免运行中反复扩容带来的停顿和指针调整。

不写这两个参数时，HotSpot 有一套人体工学默认值：初始堆大约是物理内存的六十四分之一，最大堆大约是四分之一，并且还有绝对上限。服务器档机器上，默认值还会随 CPU 和内存再调整。真正生效的数字以进程里的 `-XX:+PrintFlagsFinal` 为准，不要把“六十四分之一”当成所有版本的硬编码。

可以用一段程序把运行时看到的堆打出来。类名用 `HeapBudgetProbe`：

```java
public class HeapBudgetProbe {
    public static void main(String[] args) {
        long initial = Runtime.getRuntime().totalMemory() / 1024 / 1024;
        long max = Runtime.getRuntime().maxMemory() / 1024 / 1024;
        System.out.println("initialHeapMB=" + initial);
        System.out.println("maxHeapMB=" + max);
    }
}
```

在一台未改参数的开发机上，输出可能是初始几百 MB、最大几 GB。换一台机器或加上 `-Xms512m -Xmx512m` 之后，两个数会靠近 512。注意 `totalMemory` 是当前已向系统提交的堆，不等于 `-Xms` 的字面值在所有收集器上都能逐字节对上，因为某些收集器会把一块 Survivor 算进容量却不计入可用。

查看分配细节还有两条常用路径。先用 `jps` 找到进程号，再执行 `jstat -gc <pid>`，可以看到 Eden、两个 Survivor、老年代、元空间各自的容量和已用。若要在进程自己的输出里看收集过程，加上 `-XX:+PrintGCDetails`。JDK 9 之后更推荐统一日志，例如 `-Xlog:gc*`，旧参数在不少版本里仍然可用。

### OutOfMemory 举例

把堆锁死在一个很小的上限，再无界地往列表里加对象，就能稳定复现堆溢出。这里用 32MB，元素改成 `Long`，避免和常见的“不断加入同一个整数”示例完全重合。

```java
import java.util.ArrayList;
import java.util.List;

public class IntBoxFlood {
    public static void main(String[] args) {
        List<Long> held = new ArrayList<>();
        while (true) {
            held.add(17L);
        }
    }
}
```

启动参数：

```text
-Xms32m -Xmx32m
```

运行之后会得到 `java.lang.OutOfMemoryError: Java heap space`。列表本身和不断膨胀的元素数组都在堆上，装箱后的 `Long` 也在堆上。用 VisualVM 或 jcmd 看堆直方图，占用会集中在 `Object[]` 和 `Long` 上。这个例子证明的是上限太小加上引用没断，不是元空间或直接内存的问题。

## 年轻代与老年代

堆里的对象按寿命可以粗分成两类。一类是瞬时对象，创建和死亡都很快，适合放在收集频繁的区域。另一类寿命很长，极端情况下和虚拟机进程一样长，不该在每次年轻代收集时都被复制来复制去。

因此堆再分成年轻代 YoungGen 和老年代 OldGen。年轻代内部是 Eden、Survivor0、Survivor1，也常叫 S0/S1 或 From/To。

两块区域的默认比例由 `-XX:NewRatio` 控制。默认值 2 表示老年代 : 年轻代 = 2 : 1，年轻代大约占整堆的三分之一。若改成 `-XX:NewRatio=4`，则老年代是年轻代的四倍，年轻代只占五分之一。长寿对象明显偏多时，可以把老年代占比调高；短命对象极多、晋升很快把老年代撑满时，则要反过来检查是不是年轻代太小，或者有对象不该那么早晋升。

Eden 和单个 Survivor 的比例由 `-XX:SurvivorRatio` 控制。默认 8 的含义是 Eden 是每一块 Survivor 的八倍，于是 Eden : S0 : S1 = 8 : 1 : 1。年轻代一共十份，Eden 占八份。

几乎所有普通对象都在 Eden 里被 `new` 出来，绝大多数也死在年轻代。Eden 放不下的超大对象会走另一条路，直接在老年代尝试分配。有研究常被引用：新生代里大约八成对象活不过下一次收集。`-Xmn` 可以直接指定年轻代的容量，它会覆盖由 NewRatio 推出来的大小。没有明确证据时，先保持默认，让自适应策略工作，比随手把比例改得很极端更安全。

## 图解对象分配过程

### 概念

给新对象找一块内存，要同时考虑分配和回收。分配算法若和收集算法不匹配，收集之后会留下碎片，下一次就无法用指针碰撞。HotSpot 年轻代的路径可以按下面的顺序理解。

新对象先进入 Eden。Eden 有上限。Eden 满了而程序还要分配时，触发年轻代收集，也就是 Minor GC，把 Eden 里已经没有引用的对象清掉，再尝试放下新对象。仍存活的对象被复制到 Survivor 的其中一块。

下一次 Minor GC，会同时扫描 Eden 和当前的 From。还活着的对象被复制到 To，年龄加一。然后 From 和 To 交换身份：刚被复制进去的那块变成下一轮的 From，空的那块继续当 To。如此往复。

对象年龄到达阈值后晋升老年代。默认阈值是 15，因为对象头里的年龄只有四位。参数是 `-XX:MaxTenuringThreshold`。老年代相对安静，空间不足时才做老年代收集或整堆收集。若整堆收集之后仍然放不下，就抛出堆溢出。

### 图解过程

可以把它想成三块白板。Eden 不断被画上新对象。第一次年轻代收集：没有引用的对象擦掉，还在的对象拷到 S0，年龄记为 1，Eden 清空。随后 Eden 再次放满，第二次收集同时看 Eden 和 S0，存活对象拷到 S1，年龄变成 2，Eden 和 S0 清空。再下一轮，S1 变成 From，存活对象拷回 S0。年龄涨到 15，对象被拷进老年代，不再参与 Survivor 之间的来回复制。

To 区在每一轮开始时应当是空的。复制结束后交换角色，空的那块重新成为 To。这个约定让收集器永远有一块干净的目的地，不必在碎片里找空档。

### 思考：幸存区满了以后

Minor GC 的触发条件是 Eden 不足，不是 Survivor 满了。Survivor 被写满，本身不会单独发起一次年轻代收集。

若某次收集时 To 区放不下所有存活对象，多出来的对象会提前晋升老年代。这和年龄无关，是空间不够时的保底。可以把它理解成：正常路径按年龄一级级走，空间不够时允许插队，避免 To 区溢出把收集做失败。

### 对象分配的特殊情况

除了“Eden 不够就 Minor GC，年龄够了再晋升”，还有几条例外：

- 对象大到 Eden 剩余空间放不下，且年轻代收集也腾不出足够连续空间时，直接在老年代分配。
- `-XX:PretenureSizeThreshold` 大于 0 时，超过该阈值的对象直接进入老年代。默认 0 表示不按这个阈值拦截，大对象仍走普通分配路径。该参数主要对 Serial 和 ParNew 这一路生效，换收集器时要再确认。
- Survivor 中相同年龄对象的总体积超过 Survivor 的一半时，年龄大于等于该年龄的对象可以提前晋升，不必等满 MaxTenuringThreshold。这叫动态年龄判定。
- 分配时若 TLAB 和 Eden 都失败，还会进入慢路径，在锁保护下从 Eden 或老年代分配。

### 代码演示对象分配过程

下面的循环不断创建带随机载荷的对象，并让列表抓住它们，以便在可视化工具里看见 Eden、Survivor 和老年代轮流上涨。堆设成 384MB，单次载荷上限 256KB，节奏用 20 毫秒隔开，方便观察而不是瞬间打满。

```java
import java.util.ArrayList;
import java.util.Random;

public class BurstNode {
    byte[] payload = new byte[new Random().nextInt(256 * 1024)];

    public static void main(String[] args) throws InterruptedException {
        ArrayList<BurstNode> held = new ArrayList<>();
        while (true) {
            held.add(new BurstNode());
            Thread.sleep(20);
        }
    }
}
```

```text
-Xms384m -Xmx384m
```

用 `jvisualvm` 或 JDK 自带的 VisualGC 插件连接这个进程，可以看到 Eden 周期性下降，对应 Minor GC；年龄够了或 Survivor 放不下时，老年代阶梯式上升。列表一直持有引用，对象最终都算活着，年轻代和老年代都满之后，主线程抛出：

```text
Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
```

若把列表改成局部变量且不保存，曲线会完全不同：Minor GC 能把绝大多数对象收掉，老年代长期接近空。对照这两者，就能分清“分配速率高”和“泄漏”。

### 常用的调优工具

- JDK 自带命令：`jps`、`jstat`、`jmap`、`jcmd`、`jinfo`。
- Eclipse Memory Analyzer，用来离线看 hprof。
- JConsole，看概览和 MBean。
- VisualVM，把抽样、堆转储和 VisualGC 放在一起。
- JProfiler，看分配热点和引用链。
- Java Flight Recorder 配合 Mission Control，适合低开销的现场记录。
- GCViewer、GCEasy，用来把 GC 日志画成时间线。

工具只负责把容量、停顿和支配树呈现出来。是否要改堆、改比例或改收集器，仍然要先判断对象是短命、长寿，还是被不该存在的引用拽住。

### 总结

- S0 和 S1 在每次复制之后交换身份，空的那块是 To。
- 收集频率：年轻代高，老年代低，永久代或元空间更低。
- 年轻代用复制，是为了用较小的存活集合换一块连续空间，减少外部碎片。

## Minor GC、Major GC、Full GC

三个词经常被混用，先把范围说死。

- Minor GC：只收集年轻代。也叫 Young GC。
- Major GC：只收集老年代。口语里有时被用来指整堆收集，读日志时要以区域为准。
- Full GC：整堆收集，通常包括年轻代、老年代，并可能伴随方法区的类卸载。

调优时要减少长时间的停顿。Major GC 和 Full GC 的停顿往往比 Minor GC 高一个数量级，因为它们扫描的活对象更多，还可能整理老年代。

HotSpot 按收集范围分成两类。部分收集不碰完整的堆：只收年轻代；只收老年代；或者年轻代加上一部分老年代。整堆收集则把 Java 堆和方法区里需要收的部分一起处理。

部分收集里：

- 年轻代收集就是 Minor GC。Eden 放不下时触发，Survivor 满了本身不触发。
- 老年代单独收集在传统收集器里主要是 CMS。其他收集器很少有“只扫老年代、完全不动年轻代”的模式。日志里的 Major GC 是否等于 Full GC，要看它动了哪些代。
- 混合收集 Mixed GC 是 G1 的行为：收集全部年轻代区域，再加一部分垃圾更多的老年代区域。

整堆收集会处理整个 Java 堆，并可能卸载不再使用的类。

### Minor GC

年轻代空间不足，具体说是 Eden 不足以完成本次分配时，触发 Minor GC。每次都会清理年轻代：Eden 和 From 里的垃圾被丢弃，存活对象进入 To 或晋升。

因为多数对象死得快，Minor GC 很频繁，单次也通常比较短。它仍然是 STW，用户线程要停住，收集结束才恢复。停顿短不等于没有停顿。

### Major GC

Major GC 发生在老年代。对象从老年代消失，说明做了老年代回收或整堆回收。很多情况下，老年代空间紧张时会先尝试一次 Minor GC，把还留在年轻代里的垃圾清掉，并处理本该晋升的对象；若老年代仍然不够，再进入 Major GC 或 Full GC。这不是绝对顺序。Parallel Scavenge 的策略里就存在直接判断要不要做整堆收集的路径。

Major GC 一般比 Minor GC 慢一个数量级以上，STW 更长。若这次之后内存仍不够，就抛出溢出。

### Full GC

下面这些情况会促使整堆收集。有的是建议，有的是分配失败后的必经之路。

- 调用 `System.gc()` 时，虚拟机被建议做 Full GC，但不保证执行。
- 老年代空间不足，且年轻代收集无法解决。
- 方法区空间不足。永久代时代是 PermGen，元空间时代是 Metaspace 触到高水位或达到上限。
- Minor GC 之后，统计上要晋升的平均体积大于老年代剩余空间，虚拟机可能提前选择整堆收集，以免晋升失败。
- 从 Eden、From 向 To 复制时，对象放不进 To，需要晋升，而老年代也放不下。

整堆收集是调优里要尽量避开的事件。它一出现，延迟尖刺往往已经能被调用方感知。

### GC 举例

下面的程序不断把字符串拼长。拼接产生的新 `String` 和底层字节数组都在堆上，不是在元空间。元空间涨的是类元数据；这个例子涨的是堆，最后的错误也是 Java heap space。

```java
import java.util.ArrayList;
import java.util.List;

public class RopeGrowth {
    public static void main(String[] args) {
        int n = 0;
        try {
            List<String> kept = new ArrayList<>();
            String text = "north-pier";
            while (true) {
                kept.add(text);
                text = text + text;
                n++;
            }
        } catch (OutOfMemoryError err) {
            System.out.println("steps=" + n);
        }
    }
}
```

```text
-Xms16m -Xmx16m -XX:+PrintGCDetails
```

日志的形状会类似下面这样，具体 KB 数随 JDK 和平台变化：

```text
[GC (Allocation Failure) [PSYoungGen: 3072K->512K(4096K)] 3072K->1600K(14336K), 0.0041 secs]
[GC (Allocation Failure) [PSYoungGen: 3584K->480K(4096K)] 4672K->2890K(14336K), 0.0022 secs]
[Full GC (Ergonomics) [PSYoungGen: 480K->0K(4096K)] [ParOldGen: 2410K->6120K(10240K)] 2890K->6120K(14336K), [Metaspace: 4520K->4520K(1056768K)], 0.0084 secs]
[Full GC (Allocation Failure) [PSYoungGen: 0K->0K(4096K)] [ParOldGen: 6120K->6104K(10240K)] 6120K->6104K(14336K), [Metaspace: 4520K->4520K(1056768K)], 0.0071 secs]
```

元空间的占用在这几轮里基本不动，老年代在 Full GC 之后仍然很高，说明活着的是堆上那些被列表抓住的字符串。触发堆溢出之前，通常已经做过至少一次整堆收集：收集器要先确认“收完仍然不够”。例外是单次申请本身就大于 `-Xmx`，那种情况可以不收集直接失败。本例是逐步涨上去的，所以会先看到 Full GC，再看到 Java heap space。

## 堆空间分代思想

为什么要分代？不分代，虚拟机也能完成标记和清除。分代的理由只有一个：让收集的成本和对象的死亡率匹配，从而优化 GC 的性能。

不同对象的寿命不同，大约七成到九成九的对象是临时的。新生代由 Eden 和两块同样大小的 Survivor 构成，To 在收集开始时是空的。老年代存放在年轻代里熬过多次收集、或者体积大到不适合复制的对象。

假如所有对象都挤在同一块空间，每次收集都要扫描整堆，哪怕其中大部分是刚分配、马上就死的临时对象。把新对象集中到一块较小的区域，先只扫这块，就能用一次较短的复制收集换回大片空闲。长寿对象留在老年代，避免在每次 Minor GC 里被反复拷贝。分代不是语言规范的要求，是 HotSpot 这一类实现的工程选择。

## 内存分配策略

对象在 Eden 出生，经过第一次 Minor GC 仍然存活，并且 Survivor 放得下，就被移进 Survivor，年龄设为 1。此后每熬过一次 Minor GC，年龄加一。加到阈值就晋升老年代。默认阈值 15，不同收集器可能有自己的默认，都能用 `-XX:MaxTenuringThreshold` 调整，但年龄字段只有四位，超过 15 没有意义。

按年龄和体积，分配习惯可以分成几条：

- 优先在 Eden 分配。普通小对象走这里，也优先走 TLAB。
- 大对象尽量避免。很长的数组或拼接结果可能直接进入老年代。老年代的收集次数少，这些本可以很快死亡的大对象若待在老年代，会把真正的长寿对象的空间挤掉，回收也不及时。
- 长期存活的对象进入老年代。这是年龄阈值的正常路径。
- 动态年龄：Survivor 里某一年龄的对象总体积超过 Survivor 空间的一半，则大于等于该年龄的对象可以一起晋升，不必逐岁等到阈值。

还有空间分配担保，历史参数是 `-XX:HandlePromotionFailure`。Minor GC 之后若所有对象都活着，Survivor 很小，放不下的部分必须有老年代接住。担保要回答的是：老年代还剩的空间，够不够这次可能晋升的体积。

## 用 TLAB 完成分配

### 共享的是对象，不是分配指针

堆的数据对所有线程可见，这是共享。分配路径却不一定共享。TLAB 就是在共享的 Eden 里，为每条线程再划出一块只有它能往里分配的小缓冲。其他线程可以读到这块缓冲里已经发布的对象，但不能并发地从同一块 TLAB 往前推进指针。

### 为什么有 TLAB？

对象分配极其频繁。若每条线程分配时都去抢 Eden 的全局分配指针，就要靠 CAS 甚至锁，失败了还得重试。线程一多，分配本身会变成热点。给每条线程一块私有缓冲之后，常见路径只改自己的指针，不必原子地更新全局指针。

### 什么是 TLAB

从内存布局而不是从收集算法看，TLAB 是 Eden 内部的继续划分。每条线程一块，生命周期跟线程相关，空间仍算年轻代。多线程同时分配时，TLAB 既避开了大部分线程安全问题，又提高了分配吞吐，所以叫快速分配路径。OpenJDK 衍生的虚拟机普遍有这个设计。

不是每个对象都能放进 TLAB。数组太大、缓冲剩余不足时会失败。尽管如此，虚拟机仍把 TLAB 当作首选。`-XX:+UseTLAB` 默认开启，可以用 `-XX:-UseTLAB` 关掉做对照。

单个 TLAB 相对整个 Eden 很小。`-XX:TLABWasteTargetPercent` 的默认值是 1，表示希望因 TLAB 退回而浪费的空间大约占 Eden 的百分之一，它不是“每条线程固定分到 Eden 的 1%”。TLAB 的实际大小会按线程的分配速率动态调整。

TLAB 放不下时，虚拟机退回慢路径：在 Eden 的共享部分用 CAS 或加锁保证原子性，再不行就触发年轻代收集或向老年代分配。

### TLAB 分配过程

对象到来时，先看当前线程的 TLAB 还剩多少。放得下，指针碰撞，返回地址。放不下，按浪费策略决定是丢弃这块 TLAB 的剩余部分并换一块新的，还是直接在 TLAB 外分配。TLAB 外的分配进入 Eden 的共享区域；Eden 也不够，就 Minor GC。收集之后仍然不够，再考虑老年代。

## 和堆有关的参数

- `-XX:+PrintFlagsInitial`：查看参数的默认初值。
- `-XX:+PrintFlagsFinal`：查看最终生效值，命令行和人体工学覆盖都会反映在这里。
- `-Xms`：初始堆，默认约物理内存的六十四分之一。
- `-Xmx`：最大堆，默认约物理内存的四分之一。
- `-Xmn`：年轻代容量，同时定住初始和最大。
- `-XX:NewRatio`：老年代相对年轻代的倍数。
- `-XX:SurvivorRatio`：Eden 相对单块 Survivor 的倍数。
- `-XX:MaxTenuringThreshold`：晋升年龄上限。
- `-XX:+PrintGCDetails`：打印详细 GC 日志。简要日志可以用 `-XX:+PrintGC` 或 `-verbose:gc`。JDK 9 起还可以用 `-Xlog:gc*`。
- `-XX:HandlePromotionFailure`：早期用来控制晋升担保失败时是否冒险做 Minor GC。

发生 Minor GC 之前，虚拟机要估计老年代能不能接住可能晋升的对象。历史上的规则是：

- 若老年代最大可用连续空间大于年轻代所有对象的总体积，这次 Minor GC 是安全的。
- 若小于，就看 `HandlePromotionFailure`。为 true 时，再比较老年代剩余空间和历次晋升的平均体积。大于平均值就冒险做 Minor GC，这次仍可能失败；小于平均值就改做 Full GC。为 false 时，直接 Full GC。

JDK 6 Update 24 之后，这个参数不再真正参与决策。源码里还能看到名字，逻辑已经不用它。新规则是：只要老年代剩余的连续空闲仍大于年轻代当前的对象总量，或大于以往晋升体积的均值，就继续做 Minor GC，否则改为 Full GC。

## 对象是不是只能放在堆上

### 逃逸分析

随着即时编译和逃逸分析成熟，栈上分配、标量替换会让“所有对象都在堆上”变得不那么绝对。规范的概念模型仍是：对象在堆上分配。特殊情况是，若分析证明一个对象不会逃出当前方法或当前线程，编译器就可以不按完整对象的方式在堆上物化它，从而省掉分配和随后的收集。

逃逸分析是一种跨方法的数据流分析，用来判断新对象的引用会传到哪里。HotSpot 的编译器据此决定能不能做标量替换和锁消除。基本判别只有两条：

- 对象在方法里创建，并且只在这个方法内部使用，没有发生逃逸。
- 对象被写进字段、放进静态结构，或者作为返回值、参数传到外部，就发生了逃逸。

基于 OpenJDK 的一些定制虚拟机还尝试过把长寿对象挪到 GC 不管的区域，例如淘宝虚拟机里的 GCIH。那是堆外的另一条路，普通 HotSpot 没有这个区域。

#### 逃逸分析举例

没有逃逸的对象，才有资格被标量替换。方法结束时，这些标量随栈帧一起消失，不必等收集器。

```java
public void keepLocal() {
    SessionView view = new SessionView();
    int n = view.size();
    if (n < 0) {
        view = null;
    }
}
```

`view` 没有被保存到外面，属于未逃逸。下面这种把缓冲区返回给调用者的写法，对象一定逃逸：

```java
public static StringBuilder join(String left, String right) {
    StringBuilder sb = new StringBuilder();
    sb.append(left);
    sb.append(right);
    return sb;
}
```

若调用者只需要拼接后的文本，改成返回 `String`，缓冲区本身就可以留在方法内部，不再逃逸：

```java
public static String joinText(String left, String right) {
    StringBuilder sb = new StringBuilder();
    sb.append(left);
    sb.append(right);
    return sb.toString();
}
```

再用一组方法把几种情况并排放好。类名用 `EscapeLab`。

```java
public class EscapeLab {
    public EscapeLab ref;

    public EscapeLab publish() {
        return ref == null ? new EscapeLab() : ref;
    }

    public void store() {
        this.ref = new EscapeLab();
    }

    public void localOnly() {
        EscapeLab lab = new EscapeLab();
        int unused = lab.hashCode();
    }

    public void relay() {
        EscapeLab lab = publish();
        lab.store();
    }
}
```

`publish` 把新对象返回给调用者，逃逸。`store` 把新对象写进字段，逃逸。`localOnly` 的对象只在方法内使用，不逃逸。`relay` 通过已经逃逸的实例继续调用，相关对象也逃逸。判断口诀很简单：`new` 出来的引用会不会在方法结束之后仍被别处拿到。

#### 参数设置

JDK 7 之后，HotSpot 默认开启逃逸分析。更早的版本可以用 `-XX:+DoEscapeAnalysis` 显式打开。`-XX:+PrintEscapeAnalysis` 用来查看筛选结果，排查“为什么这个分配还在”时才需要。

#### 结论

能用局部变量解决的数据，不要写成会被外部长期引用的字段。在逃逸分析成立的前提下，编译器可以做三类优化：

- 标量替换：不把对象当成一整块堆内存，字段拆开之后放进寄存器或栈槽。教材常把这一步叫成栈上分配。
- 同步消除：对象只被一条线程碰到时，去掉它上面的锁。
- 若对象创建之后完全没有被读取，编译器甚至可以删掉这次分配。做实验时要让字段参与结果，否则测到的是死代码消除，不是标量替换。

### 栈上分配

JIT 若证明对象没有逃出方法，就可以避免在堆上物化完整对象。调用结束后栈帧弹出，拆开的字段一并消失，年轻代里不会多出一个需要扫描的实例。

常见的逃逸途径已经在上面列过：写进字段、作为返回值、传到未知的外部方法。这三种都会阻止栈上化或标量替换。

#### 举例

用一个只在方法内创建的小对象做对照。为了避免分配被当成死代码删掉，把 `id` 累加进一个静态结果。循环次数取六千万，堆固定 768MB。

```java
public class EscapeBench {
    static volatile long sink;

    static class AccountDraft {
        int id;
        int level;
        int flags;
        int region;
    }

    public static void main(String[] args) {
        long start = System.currentTimeMillis();
        for (int i = 0; i < 60_000_000; i++) {
            AccountDraft draft = new AccountDraft();
            draft.id = i;
            sink += draft.id;
        }
        long cost = System.currentTimeMillis() - start;
        System.out.println("elapsedMs=" + cost);
    }
}
```

关闭逃逸分析：

```text
-Xmx768m -Xms768m -XX:-DoEscapeAnalysis -XX:+PrintGCDetails
```

此时每次循环都在堆上留下实例，年轻代会周期性收集，耗时通常是开启优化时的数倍到数十倍。打开逃逸分析：

```text
-Xmx768m -Xms768m -XX:+DoEscapeAnalysis -XX:+PrintGCDetails
```

耗时会明显下降，GC 日志里年轻代的分配量也小得多。堆直方图里看不到六千万个 `AccountDraft`。需要说清的是：HotSpot 并没有在栈帧里放一个带对象头的完整实例，而是把 `id` 这类字段变成了标量。效果像“没在堆上分配”，机制是标量替换。

### 同步省略

进入和退出监视器的成本不低，会限制吞吐。JIT 编译同步块时，可以借助逃逸分析判断锁对象会不会被其他线程看到。若锁对象只在当前方法里创建，没有发布出去，别的线程不可能靠它做互斥，这段同步就可以删掉。这个过程叫同步省略，也叫锁消除。

```java
public void trace() {
    Object token = new Object();
    synchronized (token) {
        sink(token.hashCode());
    }
}
```

`token` 的寿命只在 `trace` 之中，编译之后等价于直接使用这个局部对象，不再生成监视器进入和退出。字节码里仍能看到 `monitorenter`，因为 javac 不做这件事；锁消除发生在 JIT，需要看编译后的本地代码或打开编译日志才能确认。

### 把聚合量拆开

原始类型这种不能再拆的量，叫做标量。还能拆开的数据叫聚合量，普通对象属于聚合量，因为它由字段组成，字段既可以是标量，也可以是另一个对象。

JIT 若确认对象不会被方法外部访问，就可以用其中的成员变量代替整个对象，这个过程是标量替换。

```java
static int dump() {
    Coord coord = new Coord(3, 5);
    return coord.x + coord.y;
}

static class Coord {
    int x;
    int y;
    Coord(int x, int y) {
        this.x = x;
        this.y = y;
    }
}
```

替换之后，效果等价于：

```java
static int dump() {
    int x = 3;
    int y = 5;
    return x + y;
}
```

`Coord` 这个聚合量没有逃逸，两个整数直接参与计算，堆上不必出现对象头、类型指针和对齐填充。少掉的是分配和后续扫描，不是计算本身。标量替换也是人们讨论“栈上分配”时，HotSpot 真正落实的那一层。

### 代码优化之标量替换

若在循环里创建数千万个小对象，每个实例仅对象头就要十几个字节，累加起来会到 GB 级。堆若小于这个量，必然发生收集。可以用下面一组参数观察标量替换打开时的行为：

```text
-server -Xmx96m -Xms96m -XX:+DoEscapeAnalysis -XX:+PrintGC -XX:+EliminateAllocations
```

参数含义：

- `-server`：使用服务端编译策略。逃逸分析和 C2 在这个模式下才完整。
- `-XX:+DoEscapeAnalysis`：打开逃逸分析。
- `-Xmx96m` 与 `-Xms96m`：把堆限制在 96MB，分配若没有被优化掉，收集会非常明显。
- `-XX:+PrintGC`：打印收集摘要。
- `-XX:+EliminateAllocations`：打开标量替换，默认就是开的。对象的字段会被当成独立的局部量，而不是一块连续的堆内存。

对照时把 `EliminateAllocations` 或 `DoEscapeAnalysis` 关掉，同一段循环会重新出现密集的年轻代收集。

### 逃逸分析的不足

相关论文在 1999 年前后就已经发表，HotSpot 到 JDK 6 才把它做进产品，而且到现在也不是所有分配都能被证明不逃逸。分析本身要做复杂的数据流，成本不一定总能被省下的分配抵消。极端情况下，跑完分析发现每个对象都逃逸了，这段分析就是纯开销。

因此它是即时编译器里很重要的一项优化，但不是可以在源码层依赖的语义。有观点认为虚拟机会把未逃逸对象完整地分配在栈上。理论上做得到，Oracle HotSpot 的选择却是标量替换，而不是在栈上放置带对象头的实例。文档和实验都支持这个结论：对象实例若真的被物化，仍然在堆上。

还要注意版本差异。JDK 7 以前，字符串常量表和静态变量曾经放在永久代。永久代换成元空间之后，`intern` 之后的字符串实例和静态字段引用的对象仍在堆上，并没有跟着类元数据进元空间。这和“被物化的对象实例在堆上”是同一件事。

## 小结

年轻代是对象出生、熬过几次复制、然后死亡或晋升的地方。对象在这里被应用使用，多数在这里被收掉。

老年代放置长寿命对象，通常是从 Survivor 筛选、复制过来的。也有例外：普通对象优先在 TLAB 上分配；对象较大时，会改在 Eden 的共享部分分配；对象大到年轻代找不到足够的连续空间，就直接进入老年代。

只发生在年轻代的收集叫 Minor GC。发生在老年代的收集叫 Major GC，若把整堆一起收了，就是 Full GC。Minor GC 的频率远高于后者，因为短命对象集中在年轻代，而老年代要等空间真正紧张才值得付出一次长停顿。
