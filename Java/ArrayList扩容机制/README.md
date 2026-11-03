# 从源码看动态数组怎么长大

## 它到底是什么

`ArrayList` 是把一块对象数组包成列表。相对自己管理 `Object[]`，它多了按下标访问、在尾部追加、按位置删除，并且实现了 `List`。容量不够时它会换一块更大的数组，调用方通常不用先算好最终长度。

JDK 8 及以后常见三种构造路径：

- 不传参数：先拿一块共享的空数组，等第一次真正写入再分配，避免空列表也占着 10 个槽。
- 传入初始容量：正数就立刻分配；0 则使用另一块“明确为空”的共享数组，之后不会自动抬到默认的 10；负数直接拒绝。
- 传入另一个集合：先把对方转成数组。长度不是 0 时，如果运行时数组类型不是 `Object[]`（某些实现的 `toArray` 会返回子类数组），就再复制成 `Object[]`。对方是空集合则换成共享空数组。对方为 `null` 会在 `toArray` 处抛出空指针。

```java
public class MiniArrayList<E> {
    static final int DEFAULT_CAPACITY = 10;
    static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;
    static final Object[] EMPTY = {};
    static final Object[] LAZY_EMPTY = {};

    Object[] buffer;
    int size;

    public MiniArrayList() {
        buffer = LAZY_EMPTY;
    }

    public MiniArrayList(int requested) {
        if (requested > 0) {
            buffer = new Object[requested];
        } else if (requested == 0) {
            buffer = EMPTY;
        } else {
            throw new IllegalArgumentException("capacity must be >= 0, got " + requested);
        }
    }

    public MiniArrayList(Collection<? extends E> source) {
        buffer = source.toArray();
        size = buffer.length;
        if (size == 0) {
            buffer = EMPTY;
        } else if (buffer.getClass() != Object[].class) {
            buffer = Arrays.copyOf(buffer, size, Object[].class);
        }
    }
}
```

两块空数组引用不同，后面才能区分“还没写过、应该按默认 10 来分配”和“调用方明确要 0 容量”。

## 使用时要权衡的点

### 长处和短处

长处来自连续内存：

- 按下标读取是一次数组访问，它还实现了 `RandomAccess`，算法库会据此选择按下标遍历，而不是迭代器。
- 只在尾部追加、且还没触到容量时，就是一次赋值，摊还成本很低。
- 空间不够时自动换更大的数组，默认按大约 1.5 倍增长。

短处也来自连续内存：

- 在头部或中间插入、删除，要把后面的元素成片挪动。
- 一旦触发扩容，就要分配新数组并复制旧内容，这一次追加会明显变贵。
- 数据量已经不小、又要频繁在中间改，复制会成为主成本，这时链表或其它结构更合适。反过来，建好之后如果几乎只做随机读，动态数组仍然合适。

### 和 Vector 怎么分

`List` 的经典实现里，数组路线有 `ArrayList` 和 `Vector`，链式路线是 `LinkedList`。

`Vector` 同样用数组存元素，但每个修改和读取方法都套了同步，同一时刻只允许一个线程进入，避免并发写坏内部数组。锁的代价是没有争用时也要走同步路径，所以单线程下它通常比 `ArrayList` 慢。需要并发时，更常见的做法是自己划定锁的范围，或换并发队列，而不是整表用 `Vector`。

增长策略也不同。`ArrayList` 新容量大致是旧容量加旧容量右移一位，也就是 1.5 倍。`Vector` 如果构造时指定了正的 `capacityIncrement`，就按这个增量加；增量用默认的 0 时，新容量是旧容量的两倍。

```java
static int nextVectorCapacity(int oldCapacity, int capacityIncrement) {
    return capacityIncrement > 0 ? oldCapacity + capacityIncrement : oldCapacity * 2;
}
```

## 容量是怎么涨上去的

### 追加时先保证空间

尾部追加的实质是：先确保 `size + 1` 个槽放得下，再把元素写到 `size` 下标，然后把个数加一。结构被修改的计数也会在这一步增加，迭代器据此发现并发修改。

```java
public boolean add(E item) {
    ensureRoom(size + 1);
    buffer[size++] = item;
    return true;
}
```

### 第一次写入时抬到默认容量

空列表第一次 `add`，传入的最小需求是 1。若当前缓冲还是那块“懒分配”的空数组，就把需求抬到 `max(10, 1)`，也就是 10。明确用 0 容量构造的列表不会走这一支，最小需求保持为 1，于是第一次追加只分配 1 个槽，之后仍按 1.5 倍长。

```java
    private void ensureRoom(int least) {
        int required = least;
        if (buffer == LAZY_EMPTY) {
            required = Math.max(DEFAULT_CAPACITY, least);
        }
        ensureExplicit(required);
    }
```

### 判断要不要真的换数组

这里把修改计数加一，再用“最小需求减当前长度”判断。写成减法而不是直接比较，是为了在容量接近整型上限时少踩溢出。只有需求大于现有长度才进入增长。

```java
    private void ensureExplicit(int least) {
        modCount++;
        if (least - buffer.length > 0) {
            grow(least);
        }
    }
```

### 按 1.5 倍计算新长度并复制

增长的核心是：

```text
newCapacity = oldCapacity + (oldCapacity >> 1)
```

右移一位相当于除以 2，加上原长就是 1.5 倍。若这样仍然小于本次需求（比如一次 `addAll` 进来很多元素），就直接用本次需求。若新长度超过 `Integer.MAX_VALUE - 8`，再交给超大容量处理：需求本身已经是负的，说明整型溢出，抛出内存错误；需求比上限还大，就试到 `Integer.MAX_VALUE`；否则停在上限减 8。留出 8，是因为一些虚拟机在数组对象头上还要占一点长度，直接要到最大整型可能分配失败。

复制使用 `Arrays.copyOf`。它内部会新建目标数组并调用 `System.arraycopy`。两者的差别是：`arraycopy` 要求你自己准备好目标数组，还能指定源起点、目标起点和长度；`copyOf` 把“建新数组再拷贝”合成一次调用，并返回新数组。

```java
    private void grow(int least) {
        int oldCapacity = buffer.length;
        int half = oldCapacity >> 1;
        int expanded = oldCapacity + half;
        if (expanded - least < 0) {
            expanded = least;
        }
        if (expanded - MAX_ARRAY_SIZE > 0) {
            expanded = huge(least);
        }
        buffer = Arrays.copyOf(buffer, expanded);
    }

    private static int huge(int least) {
        if (least < 0) {
            throw new OutOfMemoryError("capacity overflow");
        }
        return least > MAX_ARRAY_SIZE ? Integer.MAX_VALUE : MAX_ARRAY_SIZE;
    }
```

可以用一段小程序把容量变化打出来。无参构造后容量看起来是 0；第一次 `add` 变成 10；第 2 到第 10 个元素不再增长；第 11 个元素触发 `10 + 5 = 15`。

```java
public static void main(String[] args) {
    MiniArrayList<Integer> values = new MiniArrayList<>();
    for (int n = 1; n <= 11; n++) {
        values.add(n);
        System.out.println("size=" + values.size() + " capacity=" + values.capacity());
    }
}
```

预期形状是：第一行 `size=1 capacity=10`，一直到第十个元素容量仍是 10，第十一个元素容量变成 15。

### 把几次判断串起来

- 第一次追加时长度还是 0，最小需求被抬到 10，于是进入增长，直接分配 10（因为 0 的 1.5 倍仍是 0，小于 10，会改用本次需求）。
- 第二次追加时最小需求是 2，缓冲长度已经是 10，不再增长。
- 继续放到第 11 个元素，最小需求 11 大于 10，新容量算成 15。

再往后每次触顶都按当时长度的 1.5 倍向上取，直到靠近整型上限才走超大容量那一支。预先知道大致规模时，构造时传入容量或调用 `ensureCapacity`，可以避开中间这几次复制。

## 延伸

对照时直接打开所用 JDK 里 `ArrayList` 的 `grow`，以及 `Vector` 里按增量或翻倍计算的那段。转载文章里的行号很容易跟当前主版本对不上，以本机 `javap` 或源码压缩包为准。
