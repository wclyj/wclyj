# StringTable

## 字符串的基本特性

`String` 表示一串字符，写在一对双引号里。两种常见来源要分开看：

- `String city = "ningbo";` 这种字面量，由类加载解析常量池时进入字符串池。
- `String city = new String("hangzhou");` 会在堆上再得到一个 `String` 对象，同时字面量 `"hangzhou"` 仍然在池中。

`String` 被声明为 `final`，不能派生子类。它实现了 `Serializable`，因此可以序列化；也实现了 `Comparable`，因此可以按码点顺序比较。

JDK 8 及更早的实现用 `final char[]` 保存字符，每个字符固定两字节。JDK 9 起改成 `final byte[]`，另加一个编码标记。`StringBuilder` 和 `StringBuffer` 的内部存储也跟着改了，否则拼接时还得在两套表示之间来回拷贝。

### 为什么改成字节数组

大量业务里的字符串只含拉丁字母、数字和基本符号。这些字符用 Latin-1 一个字节就够，继续用 UTF-16 的 `char` 会浪费一半空间。而堆的存活数据里，字符串经常占到四分之一上下。JDK 9 的紧凑字符串按内容选择编码：能用 ISO-8859-1/Latin-1 就每字符一字节，否则退回 UTF-16，每字符两字节。编码标记记录当前用的是哪一种。这样多数英文和数字串大约省下一半字符区，含中文的串仍然是两字节，不会被截断。

### 不可变指什么

不可变不是“变量不能重新指向别的对象”，而是对象里的字符序列不能被改写。重新赋值、拼接、`replace` 都是新建一个 `String`，原来的数组继续为旧对象服务。

```java
public class TextFrozen {
    static void rebind() {
        String left = "sun";
        String right = "sun";
        System.out.println(left == right);
        left = "moon";
        System.out.println(left == right);
        System.out.println(left);
        System.out.println(right);
    }

    static void append() {
        String left = "sun";
        String right = "sun";
        right += "ray";
        System.out.println(left);
        System.out.println(right);
    }

    static void swapChar() {
        String left = "sun";
        String right = left.replace('s', 'p');
        System.out.println(left);
        System.out.println(right);
    }

    public static void main(String[] args) {
        rebind();
        append();
        swapChar();
    }
}
```

`rebind` 先打印 `true`，因为两个字面量是池中同一对象；改指 `moon` 之后变成 `false`，`right` 仍是 `sun`。`append` 打印 `sun` 和 `sunray`，旧对象没有被改掉。`swapChar` 打印 `sun` 和 `pun`。用字面量得到的串位于字符串池；`new` 得到的对象在堆上，池里另有一份相同内容的字面量。

### 传参时改的是谁

```java
public class RefSwap {
    String word = new String("east");
    char[] raw = {'p', 'a', 'r', 'k'};

    void twist(String word, char[] raw) {
        word = "west";
        raw[0] = 'm';
    }

    public static void main(String[] args) {
        RefSwap box = new RefSwap();
        box.twist(box.word, box.raw);
        System.out.println(box.word);
        System.out.println(box.raw);
    }
}
```

输出是 `east` 和 `mark`。方法参数是引用的副本。`word = "west"` 只改了副本的指向，字段仍指着原来的 `String`，而 `String` 本身也不能被改内容。`raw` 的副本和字段指向同一个数组，改下标 0 会反映到字段上。

### 池中不保存两份相同内容

字符串池不会为同样的字符序列放两个入口。它在 HotSpot 里是一张哈希表。JDK 6 这张表的桶数固定为 1009，放进大量不重复的串之后冲突链会很长，`intern` 明显变慢，而且当时基本不能靠参数把表加长。JDK 7 起默认桶数改为 60013。JDK 8 仍默认 60013，同时规定 `-XX:StringTableSize` 不得小于 1009，设得更小会启动失败。桶数使用质数更能打散哈希。这个参数必须在启动时给定，不能在运行中把表扩容成另一张。

## 字符串放在哪块内存

八种基本类型有各自的常量缓存，`String` 也有池，但池的位置变过。

直接用双引号写出的对象进池。不是双引号创建的，可以调用 `intern`，按下面的规则决定是否入池。

- JDK 6 及更早：池在永久代。永久代默认很小，回收也不频繁，大量 `intern` 容易把永久代撑满。
- JDK 7：Oracle 把池挪到 Java 堆，和普通对象一起参与堆的回收。调堆的大小就会同时影响到池。
- JDK 8：类元数据改到元空间，字符串池仍然留在堆里，没有跟着元空间走。

永久代又小又少回收，是搬家的直接原因。搬家之后，驻留大量字符串的应用会看到堆占用上升、永久代占用下降，堆的 `-Xmx` 往往要重新估。`intern` 的收益和风险也都变成堆上的事情：重复内容能折叠成一份，但互不相同的内容会把池和堆一起撑大。

## 字面量必须是同一实例

语言规范要求：完全相同的字符串字面量，码点序列一致，并且指向同一个 `String` 实例。这是 `==` 对两个字面量经常为真的依据。`new` 出来的对象不在这条规则里，即使内容和字面量相同，引用也不相同，除非你把 `intern` 的返回值接回来。

## 拼接会发生什么

- 常量与常量相加，结果在编译期折进池里。
- 池里不会出现两份内容相同的常量。
- 任意一侧是变量，结果就是堆上的新对象。变量拼接在 JDK 5 之前用 `StringBuffer`，JDK 5 起改用 `StringBuilder`。
- 拼接结果再调用 `intern`，若池中还没有这份内容，就把对应对象放进池并返回池中的引用。

```java
static void fold() {
    String sum = "k" + "i" + "t";
    String lit = "kit";
    System.out.println(sum == lit);
}

static void mix() {
    String head = "neo";
    String tail = "flux";
    String all = "neoflux";
    String folded = "neo" + "flux";
    String rightVar = head + "flux";
    String leftVar = "neo" + tail;
    String both = head + tail;
    System.out.println(all == folded);
    System.out.println(all == rightVar);
    System.out.println(all == leftVar);
    System.out.println(all == both);
    System.out.println(rightVar == leftVar);
    System.out.println(both.intern() == all);
}
```

`fold` 为 `true`，编译器已经把 `"k"+"i"+"t"` 收成池中的 `"kit"`。`mix` 里只有 `all == folded` 和最后的 `intern` 比较为 `true`。只要加号两侧出现变量，就相当于在堆上得到一个新的 `String`，内容是拼出来的字符序列。`intern` 再去池里找 `neoflux`：已经有字面量就返回池中那份，于是和 `all` 相等。

### 变量拼接的实际步骤

`head + tail` 在字节码里大致是：新建 `StringBuilder`，依次 `append` 两段，再 `toString`。`toString` 会 `new` 一个内容为拼接结果的 `String`，但不会自动把这个结果放进池。JDK 9 之后的编译策略可能换成 `invokedynamic` 引导的拼接策略，减少中间对象，但“变量拼接的结果不等于池中字面量”这一点仍然成立，除非结果再 `intern`，或者整个表达式在编译期就是常量。

`String` 每次修改都产生新对象。`StringBuffer` 可变且方法带同步，适合多线程共享同一缓冲区。`StringBuilder` 可变且不同步，单线程下更快。能确定只有一个线程使用时，用 `StringBuilder`。

两侧若是 `final` 的编译期常量，加号仍然可以折叠，不会走构建器。

```java
static void finals() {
    final String a = "q";
    final String b = "z";
    String expect = "qz";
    String got = a + b;
    System.out.println(expect == got);
}
```

结果为 `true`。`final` 在这里的作用是让变量变成编译期常量，不是“所有 final 字段拼起来都在池里”。若 `final` 的值要到运行期才知道，例如 `final String a = strParam;`，折叠不会发生。

### 循环里拼接和 append

```java
static void repeatPlus(int rounds) {
    String acc = "";
    for (int i = 0; i < rounds; i++) {
        acc += "z";
    }
}

static void repeatAppend(int rounds) {
    StringBuilder acc = new StringBuilder(rounds);
    for (int i = 0; i < rounds; i++) {
        acc.append("z");
    }
}
```

`repeatPlus` 每一轮都创建构建器，`append` 一次，再 `toString` 出新的 `String`，下一轮把这个新串再拷进去。`repeatAppend` 从头到尾只有一个构建器。轮次到几万时，两者的耗时常常差出两个数量级，精确毫秒数取决于机器，不值得背。构建器的无参构造默认容量是 16，超出后按实现策略扩容并拷贝。能估计上界时，用 `new StringBuilder(rounds)` 这种带容量的构造，减少扩容。

## `intern` 做什么

`intern` 是本地方法。池最初为空，由 `String` 私有地维护。调用时若池里已经有与当前对象 `equals` 的串，就返回池中的那个引用，当前对象不会进池。若没有，JDK 6 会把字符拷贝一份新对象放进永久代的池，返回这份拷贝；JDK 7 起不再拷贝字符，而是把堆上这个对象的引用登记进池，返回的就是原引用。

因此，对任意串调用 `intern` 之后，返回值必须和同一内容的字面量是同一个实例。`("k" + "i" + "t").intern() == "kit"` 在规范下为真。大量重复的城市名、状态码如果先 `intern` 再保存，堆上可以只留一份字符数据。互不重复的标记则不该 `intern`，否则池只增不减，直到这些引用都消失并且发生回收。

下面的程序用一小撮重复数字制造大量内容相同的串，便于观察 `intern` 对保留大小的影响。正式对比时应固定堆并做两次运行：一次调用 `intern`，一次不调用。

```java
public class InternFoldDemo {
    static final int ROUNDS = 200000;
    static final String[] SLOT = new String[ROUNDS];

    public static void main(String[] args) {
        int[] seed = {1, 2, 3, 4, 5, 6, 7, 8};
        for (int i = 0; i < ROUNDS; i++) {
            SLOT[i] = new String(String.valueOf(seed[i % seed.length])).intern();
        }
        System.out.println(SLOT[0] == SLOT[8]);
    }
}
```

`SLOT[0]` 和 `SLOT[8]` 内容都来自 `"1"`，`intern` 之后应是同一个引用，打印 `true`。若去掉 `intern`，数组里会留下二十万个各自独立的 `String`。社交资料里反复出现的省市区名称，就属于这种重复度很高、适合折叠的数据。

## 创建了几个对象

`String str = new String("xy");` 在 `xy` 尚未入池时，涉及两个 `String`：`ldc` 让池中出现 `"xy"`，`new` 在堆上再构造一个内容相同的对象。字节码顺序是 `new`、`dup`、`ldc`、`invokespecial`。若 `"xy"` 已经在池中，`ldc` 不再新建，这次表达式只多一个堆对象。底层的 `char[]` 或 `byte[]` 是否再单独算一个，随 JDK 版本变化，面试里说的“两个”指的是两个 `String`。

`new String("x") + new String("y")` 从字节码上可以数出六个对象：一个 `StringBuilder`；堆上的 `new String("x")` 和池中的 `"x"`；堆上的 `new String("y")` 和池中的 `"y"`；`toString` 里新的 `String`，内容是 `xy`。最后这个 `xy` 不会自动进入池。把结果赋给变量，变量指向的是堆对象，不是池对象。

### JDK 6 和 JDK 7 的差别

```java
static void afterIntern() {
    String heap = new String("m") + new String("n");
    heap.intern();
    String lit = "mn";
    System.out.println(heap == lit);
}
```

JDK 6 打印 `false`。`intern` 在永久代复制出一个新的 `"mn"`，随后的字面量指向这份拷贝，`heap` 仍是堆上那个对象。JDK 7 及以后打印 `true`。池里没有 `"mn"` 时，`intern` 把 `heap` 的引用登记进去，字面量再解析到同一引用。

若先写字面量，两个版本都变成 `false`：

```java
static void literalFirst() {
    String heap = new String("m") + new String("n");
    String lit = "mn";
    heap.intern();
    System.out.println(heap == lit);
}
```

字面量已经在池中占了位置，`intern` 发现已有相等内容，直接返回池中的对象，但返回值没有赋回 `heap`，所以 `heap` 和 `lit` 仍然不是一个引用。

`new String("m")` 再 `intern`，和字面量 `"m"` 比较，两个版本都是 `false`，因为 `"m"` 在执行 `new String("m")` 的 `ldc` 时就已经入池，`intern` 不会把堆对象替换进池。只有把 `intern` 的返回值赋回去，引用才会指到池中那份。

另一个容易混的例子是启动时就已经在池里的串，例如某些版本里的 `"java"`。对这种内容，`new String("ja") + new String("va")` 再 `intern`，返回的是池里原有的对象，而不是拼接出来的堆对象，所以 `intern() == 拼接结果` 为 `false`。内容若是池里从来没有过的 `"mn"`，JDK 7 及以后这次比较才为 `true`。JDK 6 因为总是拷贝，两种内容都是 `false`。

## 池本身也会被回收

池在堆上之后，不再被引用的入池字符串可以在垃圾回收时消失，桶里的入口一并去掉。下面的循环制造大量互不相同的串并 `intern`，用来观察回收和表统计，而不是用来模拟业务。

```text
java -Xms18m -Xmx18m -XX:+PrintStringTableStatistics -XX:+PrintGCDetails UniqueTokenDemo
```

```java
public class UniqueTokenDemo {
    public static void main(String[] args) {
        for (int i = 0; i < 80000; i++) {
            String.valueOf(i).intern();
        }
    }
}
```

这些局部结果没有被字段保存，方法结束后只剩池中的引用。堆只有 18MB 时，后续分配会迫使收集器运行，统计信息里能看到入池数量和因此发生的回收。JDK 9 之后详细回收日志要改成 `-Xlog:gc*`，`PrintGCDetails` 不再是推荐写法。`-XX:+PrintStringTableStatistics` 仍用于在退出时打印桶的分布。

## G1 的字符串去重

这里说的重复，指堆上内容相同但不是同一个对象的 `String`，不是池里的重复。池本身就不会放两份。

有的统计里，存活堆数据约有四分之一是 `String`，其中接近一半内容彼此重复，平均长度大约几十个字符。G1 从 JDK 8u20 起可以在回收过程中把这些底层数组折叠成一份。它和 `intern` 不是一回事：去重之后两个 `String` 对象仍然是两个，`==` 仍为假，只是它们的 `value` 指向同一份数组，`equals` 为真。

做法是：收集器访问存活对象时，把达到年龄阈值的 `String` 放进队列。后台线程用一张表记录已经见过的字符数组。若堆上已有内容相同的数组，就把后来的 `String` 改指到已有数组，原数组失去引用后再被回收。若没有，就把这份数组登记进表，供以后共享。

相关参数：

- `-XX:+UseStringDeduplication`：打开去重。默认关闭，并且只对 G1 生效。
- `-XX:+PrintStringDeduplicationStatistics`：打印去重统计。
- `-XX:StringDeduplicationAgeThreshold`：候选对象要达到的年龄，默认是 3。年龄太小会把大量短命串送进队列，得不偿失。

对象若超过 Region 的一半，会进入 Humongous 区域，去重并不打算处理这种超大字符串。

## 参数

- `-XX:StringTableSize`：JDK 7 起默认 60013；JDK 8 最小 1009。只影响池的桶数，不改变堆的大小。
- `-XX:+PrintStringTableStatistics`：进程退出时打印表的使用情况。
- 上面的 G1 去重三项。不要在 Parallel 或 CMS 的进程里指望 `UseStringDeduplication` 生效。

## 易错点

- `==` 比较的是引用。内容相等要用 `equals`。字面量相等不一定能推广到 `new` 和变量拼接。
- JDK 6 的 `intern` 是拷贝，JDK 7 及以后在池中没有该内容时登记的是原引用。把旧教程的 `true/false` 背到 JDK 8 上会全部反掉。先出现字面量还是先 `intern`，结果也不同。
- 循环里用 `+` 拼接，看起来和 `append` 一样，实际每轮都有构建器和中间串。
- 对唯一标识大量 `intern`，池会变成巨型哈希表，既占堆，又让查找变慢。去重解决的是数组副本，不能代替池，也不能让两个对象变成 `==`。
- JDK 9 的 `byte[]` 不表示“所有字符串都省了一半”。中文仍按 UTF-16 存储。判断版本时看 `java -version`，不要用 JDK 8 的字段布局去读 JDK 11 的堆转储。
