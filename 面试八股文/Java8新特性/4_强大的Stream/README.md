# Stream API

## 了解Stream

Java 8 在集合处理上有两处明显变化。

- 箭头函数让行为可以当参数传
- `java.util.stream` 把遍历本身收成一条流水线

流描述的是“对这些元素依次做哪些计算”，过滤、变换、汇总都可以串在一起。读起来有点像给集合写查询，也可以声明成并行执行。它提供的是一种少写循环、又便于改条件的写法，而不是另一套容器。

## 区别

这里的流和输入输出流不是一回事。数据从源头走向结果时，中间会叠上切片、排序、筛选等步骤，最后得到一个新的结果。源头里的元素不会被这些步骤改掉。

可以把它想成传送带：货物还在原来的箱子里，传送带上只是依次做检查、贴标和装箱。没有人按下“出结果”之前，检查甚至不会真正开始。

## 什么是Stream

流是一条计算通道，元素来自集合、数组或生成器。集合负责保存数据，流负责描述计算。

注意三点：

- 流自己不保存元素，元素仍在源头
- 流不改源头，每步返回的是装着结果的新流
- 中间步骤是延迟的，只有遇到终止操作才真正计算

因此在终止之前多次叠 `filter`，不会把集合扫描多遍。整个链条会被合成一次遍历。

## Stream操作的三部曲

### 创建流

从数据源拿到流，本身还不会遍历。

- 集合用 `stream()` 拿顺序流，用 `parallelStream()` 拿并行流

```java
List<String> codes = new ArrayList<>();
Stream<String> fromList = codes.stream();
```

- 数组用 `Arrays.stream`

```java
Parcel[] row = new Parcel[4];
Stream<Parcel> fromArray = Arrays.stream(row);
```

- 几个离散元素用 `Stream.of`

```java
Stream<String> fixed = Stream.of("北坞", "东闸", "西坡");
```

- 没有自然终点的序列用 `iterate` 或 `generate`，使用时必须自己截断

```java
Stream<Integer> evens = Stream.iterate(3, n -> n + 2).limit(4);
```

上面会得到 3、5、7、9。忘掉 `limit`，后面的终止操作就会一直要下去。

### 中间操作

中间操作连接成链，只记录“以后要做什么”，立即返回另一个流。过滤、映射、排序、截断都属于这一段。

### 终止操作

终止操作触发整条链，产出结果或副作用。一个流只能终止一次，终止之后再对它做中间操作会抛异常。常见的终止有遍历、收集、归约、匹配和计数。

整条链可以记成：源头 → 若干中间操作 → 一次终止。

## Stream流的操作

中间操作无论写了多少个，都要等终止操作出现才执行。这叫惰性求值。好处是没有人要结果时，昂贵的映射根本不会跑；也可以把过滤放在映射前面，少做无用功。

### 筛选与切片

- `filter`：按断言留下元素
- `distinct`：用 `hashCode` 和 `equals` 去掉重复
- `limit`：最多保留给定个数
- `skip`：丢掉前面若干个；不够丢就得到空流

```java
import java.util.Arrays;
import java.util.List;

record Parcel(String name, int years, double bonus) {}

public class ParcelStreamLab {
    static List<Parcel> parcels() {
        return Arrays.asList(
                new Parcel("周宁", 4, 3200),
                new Parcel("韩雪", 11, 8600),
                new Parcel("马林", 9, 5400),
                new Parcel("邱岩", 6, 9100),
                new Parcel("苏晚", 2, 2800)
        );
    }

    public static void main(String[] args) {
        parcels().stream()
                .filter(p -> p.years() > 8)
                .limit(1)
                .forEach(p -> System.out.println(p.name()));
    }
}
```

驾龄大于 8 的有韩雪和马林，`limit(1)` 只留下先出现的韩雪。`limit` 在顺序流里还能让后面的元素不再被计算，这是短路。

`Parcel` 需要自己实现 `equals` 和 `hashCode`，`distinct` 才按业务字段去重，而不是按对象身份。

### 映射

`map` 把每个元素变成另一个值，一对一。`flatMap` 把每个元素变成一条流，再把这些流摊平成一条，适合“一句拆成多个词”这种一对多。

```java
public static void upper() {
    List<String> tags = Arrays.asList("dock", "pier", "yard");
    tags.stream().map(String::toUpperCase).forEach(item -> System.out.println(item));
}

public static void words() {
    List<String> lines = Arrays.asList("dock pier", "yard gate");
    lines.stream()
            .flatMap(line -> Arrays.stream(line.split(" ")))
            .forEach(item -> System.out.println(item));
}
```

第一段输出 `DOCK`、`PIER`、`YARD`。第二段输出四个独立单词。如果误用 `map` 去做拆分，得到的是 `Stream<String[]>`，而不是单词流。

### 排序

- `sorted()`：按元素自己的自然顺序
- `sorted(Comparator)`：按给定比较器

```java
public static void order() {
    List<String> tags = Arrays.asList("yard", "dock", "pier", "gate");
    tags.stream().sorted().forEach(item -> System.out.println(item));

    parcels().stream().sorted((a, b) -> {
        if (a.years() == b.years()) {
            return a.name().compareTo(b.name());
        }
        return Integer.compare(a.years(), b.years());
    }).forEach(item -> System.out.println(item));
}
```

标签按字典序是 `dock`、`gate`、`pier`、`yard`。包裹按驾龄升序，驾龄相同再按姓名，顺序为苏晚、周宁、邱岩、马林、韩雪。

`sorted` 必须看到全部元素才能决定第一个是谁，所以它不是短路操作，放在 `limit` 前面会把截断的好处抵消掉。需要“最小的三个”时，应先按比较器排序再 `limit`，或者直接用后续版本里的部分排序思路，至少不要在排序前就把数据滤掉条件写反。

### 终止操作

一旦执行下面这些操作，流就结束了，不能再复用。

查找与匹配：

- `allMatch`：是否每一个都满足
- `anyMatch`：是否至少有一个满足
- `noneMatch`：是否一个都不满足
- `findFirst`：顺序流里的第一个
- `findAny`：任意一个，并行时不保证是哪一个
- `count`：个数
- `max` / `min`：按比较器取极值
- `forEach`：内部迭代，由流来遍历

```java
public static void match() {
    boolean allAdult = parcels().stream().allMatch(p -> p.years() > 3);
    boolean anySenior = parcels().stream().anyMatch(p -> p.years() > 10);
    System.out.println("全部驾龄大于3:" + allAdult);
    System.out.println("至少一人驾龄大于10:" + anySenior);
}
```

按前面的名单，苏晚只有 2 年，所以第一行是 false；韩雪 11 年，第二行是 true。`allMatch` 遇到第一个不满足就可以停，不必把剩下的人都看完。

`findFirst` 和 `findAny` 返回的是 `Optional`。源头可能一个都不剩，调用方要用 `orElse` 或 `ifPresent` 处理空。

### 规约

`reduce` 把一串元素收成一个值。带初值的形式是 `reduce(identity, accumulator)`，不带初值时返回 `Optional`，因为空流没有结果。

它和大数据里“先映射再汇总”是同一类思路：`map` 先把元素变形成可合并的值，`reduce` 再两两合并。

```java
List<Integer> tons = Arrays.asList(4, 6, 1, 9, 2);
int total = tons.stream().reduce(0, (acc, item) -> acc + item);
System.out.println(total);
```

初值 0 充当第一次的 `acc`，依次加上 4、6、1、9、2，结果是 22。初值必须是合并运算的单位元，求和用 0，求积用 1。并行归约时，这个运算还要满足结合律，否则分段相加的结果会和从左到右不一样。

### 收集

`collect` 把流收成另一种结构。真正怎么收，由 `Collector` 决定。`Collectors` 里准备了收成列表、集合、映射，以及分组、连接字符串等现成收集器。

```java
public static void names() {
    List<String> who = parcels().stream()
            .map(Parcel::name)
            .collect(Collectors.toList());
    who.forEach(item -> System.out.println(item));
}
```

这里得到五个人名，顺序与源头一致。若要按驾龄分组，把收集器换成 `Collectors.groupingBy(Parcel::years)`，得到的是 `Map<Integer, List<Parcel>>`。

收集会把元素拷进新容器，源头的列表仍然保持原样。这和“流不修改数据源”是同一条约定。需要顺带去重时用 `toSet()`，但集合不保证顺序。
