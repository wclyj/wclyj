# Java 8 里散列容器改了什么

## 主要特性

- 可以把一小段行为当作参数传来传去
- 只声明一个抽象方法的接口
- 用已有方法或构造器去实现这种接口
- 面向集合的流水线运算
- 接口上允许写默认实现和静态工具
- 重新设计的日期与时间
- 用一层包装表达“值可能缺失”，调用链更不容易突然碰到空引用
- 还有类型推断、重复注解等零碎改进，这篇先不展开

## HashMap1.7

7 到 8 之间，这条映射的桶结构被重新做过一轮。

7 在哈希撞车时，把新结点插到链表最前面，也就是头插。

当时的想法是：刚放进去的键，马上又被读到的机会更大，离数组近一点似乎更划算。这个理由经不起推敲。扩容搬迁时，如果一整条链仍然落到同一个新下标，头插会把结点顺序整个翻过来，所谓热点顺序早就没了。头插没保住局部性，并发扩容时还会把链表拧成环，所以 8 改成从尾部追加。

文字示意：旧链是 `e1 → e2 → e3`，再头插 `e4`，变成 `e4 → e1 → e2 → e3`。

## HashMap1.7存在死链问题

多个线程同时给 7 的表扩容时，一条普通链表可能变成环。之后顺着 `next` 查找，会永远走不出去。8 改成尾插，并按哈希高位把一条链拆成两条，避开了这种环。

下面用精简代码看单线程时结点怎么进桶。命中已有键就覆盖，否则头插，超过门槛就扩容。

```java
public class Jdk7BucketSketch {
    static final class Node {
        final int hash;
        final String key;
        String val;
        Node next;

        Node(int hash, String key, String val, Node next) {
            this.hash = hash;
            this.key = key;
            this.val = val;
            this.next = next;
        }
    }

    private Node[] slots = new Node[16];
    private int count;
    private int border = 12;

    public String put(String key, String val) {
        int h = spread(key);
        int idx = h & (slots.length - 1);
        for (Node cur = slots[idx]; cur != null; cur = cur.next) {
            if (cur.hash == h && key.equals(cur.key)) {
                String prev = cur.val;
                cur.val = val;
                return prev;
            }
        }
        count++;
        pushFront(h, key, val, idx);
        return null;
    }

    private void pushFront(int h, String key, String val, int idx) {
        slots[idx] = new Node(h, key, val, slots[idx]);
        if (count >= border) {
            grow(slots.length << 1);
        }
    }

    private void grow(int freshCap) {
        Node[] older = slots;
        Node[] fresh = new Node[freshCap];
        for (int i = 0; i < older.length; i++) {
            Node cur = older[i];
            while (cur != null) {
                Node follow = cur.next;
                int pos = cur.hash & (freshCap - 1);
                cur.next = fresh[pos];
                fresh[pos] = cur;
                cur = follow;
            }
        }
        slots = fresh;
        border = (int) (freshCap * 0.75f);
    }

    private static int spread(String key) {
        int h = key.hashCode();
        return h ^ (h >>> 16);
    }
}
```

默认负载系数 0.75，默认长度 16，第一次扩容线是 12。过线后每个键都按新长度重新取下标。新表长度仍是旧表的两倍。

环出现在两条线程交错的时候。假设旧表长度是 2，键 9、13、21 都落在下标 1，头插之后链是 `21 → 13 → 9`。线程甲刚执行完“先记下后继”，`cur` 是 21，`nxt` 是 13，然后被换出。线程乙独自把表扩到长度 4，头插迁移把链翻成 `9 → 13 → 21`。甲恢复后仍握着过期的指针：它把 21 挂上新桶，再顺着已经被乙改过的 `next` 走到 13，最后让 13 的后继指回 21。两边互指，环就成了。这时在该桶查找一个不存在的键，会停不下来。

单线程不会出环，因为后继在改指针之前就存好了，没有人同时改同一条链。这条映射从一开始就不承诺并发安全，环只是后果之一，覆盖丢失同样会发生。

下面这段程序用两轮头插模拟上面的交错，并用快慢指针确认成环。

```java
public class HeadInsertCycleLab {
    static final class Cell {
        final int id;
        Cell next;

        Cell(int id) {
            this.id = id;
        }
    }

    static Cell push(Cell head, Cell node) {
        node.next = head;
        return node;
    }

    public static void main(String[] args) {
        Cell nine = new Cell(9);
        Cell thirteen = new Cell(13);
        Cell chain = push(null, nine);
        chain = push(chain, thirteen);

        Cell held = chain;
        Cell remembered = held.next;

        Cell rebuilt = null;
        for (Cell walk = chain; walk != null; ) {
            Cell follow = walk.next;
            rebuilt = push(rebuilt, walk);
            walk = follow;
        }

        held.next = rebuilt;
        boolean loop = false;
        Cell slow = held;
        Cell fast = held;
        while (fast != null && fast.next != null) {
            slow = slow.next;
            fast = fast.next.next;
            if (slow == fast) {
                loop = true;
                break;
            }
        }
        System.out.println("paused-on=" + held.id + ", remembered=" + remembered.id + ", cycle=" + loop);
    }
}
```

## HashMap每次扩容为什么是2倍

下标不是用除法取余，而是 `(长度 - 1) & hash`。长度保持 2 的幂时，`长度 - 1` 的二进制全是 1，与运算只留下哈希的低几位，效果和取模一样，但只是位运算。

长度 16 时掩码是 `01111`，低位不同的哈希会进不同的槽。若把长度设成 10，掩码变成 `01001`，中间被掩掉的位不再参与定位，不同哈希更容易挤进同一槽。

```java
public class PowerOfTwoIndex {
    static int slot(int hash, int cap) {
        return (cap - 1) & hash;
    }

    public static void main(String[] args) {
        int[] samples = {0b0001, 0b0011, 0b0101, 0b0111};
        System.out.println("len=16 mask=01111");
        for (int h : samples) {
            System.out.printf("%8s -> %d%n", Integer.toBinaryString(h), slot(h, 16));
        }
        System.out.println("len=10 mask=01001");
        for (int h : samples) {
            System.out.printf("%8s -> %d%n", Integer.toBinaryString(h), slot(h, 10));
        }
    }
}
```

因此每次把长度翻倍：掩码始终是一串 1，扩容后每个键只可能留在原下标，或原下标再加上旧长度。8 的搬迁就靠这一位把链拆开，不必做昂贵的除法。

## JDK1.8结构变化

7 的桶是数组配链表。8 仍以这两者为主，单桶过长时再换成红黑树。

树化要同时满足两点：这条链上的结点数超过 8，并且数组长度已经不小于 64。只是链长过 8、表还很小，会先扩容把键摊开，而不是马上建树。结点后来少到 6，树会退回链表，避免来回变换。

收益主要在查找和删除。链表平均要扫过 O(n) 个结点，红黑树降到 O(log n)。纯插入若只是接到尾部，链表是 O(1)，树还要旋转，所以短链继续用链表更合适。

## 为什么HashMap使用红黑树而不是AVL树

把过长的链换成树，是为了避免查找退化成线性扫描。平衡树不止一种，AVL 比红黑树更严，这里为什么不用它？

并发容器对单个桶加互斥，写冲突时其它写线程要等。AVL 为了把高度差压到 1，插入时旋转更多，临界区更长。红黑树放宽平衡，一次插入最多转两次，持锁时间更短。普通 HashMap 虽然不保证并发，但同一套树结点还要给并发映射复用，旋转成本要按持锁时间来估。

两种树的查找、删除、更新都是 O(log n)，差别在常数和旋转次数。

- AVL 左右高度差不超过 1，查找路径更短，适合几乎只读
- 红黑树平衡更松，插入和删除更省旋转，适合写得频繁
- AVL 的旋转情形多，实现和排查都更重

### 总结

- 两者都把高度维持在对数级，真正的差别是每次修改要转几次
- 查找同为 O(log n)。AVL 平均路径更短，改结构更慢
- AVL 上任意叶子的高度差最多为 1。红黑树里，最长路径可以达到最短路径的大约两倍
- 红黑树恢复颜色最多旋转两次，改指针本身是常数时间。决定在哪个祖先转，仍可能沿树走上对数步

## ConcurrentHashMap变化

### 为何JDK8要放弃分段锁？

7 把整张表切成若干段，每段继承可重入锁，段内仍是数组加链表，没有树。段的个数在创建时写死，以后只扩段里面的数组。读大多不加锁，但锁的粒度是整段：段划得太大，无关的写也会互斥；段划得太小，空段白占内存。段数还无法随数据增长，并发度有上限。

8 去掉段，直接在数组、链表和红黑树上做安全更新。空桶用比较并交换放入第一个结点；桶里已有数据时，只锁头结点。数组引用，以及结点里的值和后继，都标成易变，读路径才能看见最新指针。表一扩容，可同时更新的桶变多，并发度跟着涨。

### 弃用的原因

- 每一段都带着同步队列，数据少的时候这些对象是纯开销
- 线上两次写撞进同一段的概率往往不高，段锁却让整段更新排队
- 锁对象变少，垃圾回收要扫描的结构也更轻

### 新的同步方案

旧的段类型还留在源码里，主要为了读老版本序列化数据，正常的写入不再走它。

桶是空的，就直接比较并交换，成功就返回：

```java
if (table == null || table.length == 0) {
    table = initSlots();
} else if ((head = elementAt(table, index)) == null) {
    if (compareAndSet(table, index, null, new Node<>(hash, key, value))) {
        break;
    }
}
```

桶里已经有结点，就锁住这个头。哈希不小于 0 表示还是普通链表：相等就覆盖，走到尾就追加。头结点若是树容器，改走树插入。若碰到的是搬迁占位结点，说明别的线程正在扩容，当前线程应去帮忙搬运，而不是在占位结点上嵌套更新。

```java
synchronized (head) {
    if (elementAt(table, index) == head) {
        if (head.hash >= 0) {
            for (Node<K, V> cur = head; ; ) {
                if (cur.hash == hash && key.equals(cur.key)) {
                    cur.val = value;
                    break;
                }
                if (cur.next == null) {
                    cur.next = new Node<>(hash, key, value);
                    break;
                }
                cur = cur.next;
            }
        } else if (head instanceof TreeBin) {
            ((TreeBin<K, V>) head).putTreeVal(hash, key, value);
        }
    }
}
```

记法可以收成一句：空桶做比较并交换，非空桶锁头结点，语言提供的互斥配合无锁空位。

## ConcurrentHashMap为什么要使用synchronized而不是如ReentranLock这样的可重入锁？

先看锁有没有变粗。

- 粒度反而更细。每扩容一次，独立的桶头变多，能同时进行的更新变多
- 定位更直接。7 要先找段再找桶，做两次散列。8 用链表或树一次定位，把这段差距补上
- 搬迁可以分担。其它线程看见转发结点后，可以认领一段区间一起搬，不必整张表停写

再看为什么不给结点配可重入锁。

- 省内存。可重入锁要带同步队列，真正需要互斥的只有桶头或树根。给每个结点配一把锁，大部分锁对象永远用不上
- 能跟着虚拟机变快。可重入锁是类库，后面能优化的余地小。语言级互斥由虚拟机实现，运行时可以做锁消除、锁粗化和自适应自旋，升级运行时就可能变快，业务代码不用改

## 内存结构优化

8 去掉了永久代，类的元数据改放到本地内存中的元空间。方法区里主要是类的结构模板，不再挤在一块划死的堆内存里，因此“类信息把永久代撑满”这种错误少了很多。调节参数换成：

- MetaspaceSize
- MaxMetaspaceSize

元空间默认只受机器内存限制。类加载器如果泄漏，仍会把本机内存吃光，所以生产上要给它设上限。

## 参考

对照三件事就够：头插在两次交错搬迁里怎样互指、掩码为什么必须是一串 1、桶头上的互斥加上树化之后临界区为什么更短。把这三步在纸上画出来，比背一串默认参数有用。
