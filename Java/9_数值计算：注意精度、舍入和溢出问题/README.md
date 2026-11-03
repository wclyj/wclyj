# 金额和计数不要交给会悄悄出错的类型

人眼把 `0.1` 当成和 `1` 一样精确的数，把多写几位的整数当成“只是更长”。计算器和图灵机都不这么看。浮点数在二进制里经常存的是近似值；整数则被塞进固定宽度的格子，超过格子就会绕回另一端，而且默认不抛错。

手机计算器把百分数加百分数得出意外结果，往往还叠了一层按键语义：有的实现把 `a + b%` 理解成 `a * (1 + b%)`。那是输入法的问题。下面只谈语言本身：二进制小数、舍入、以及格子太小。

## 二进制里的小数

先看四则运算和一次判等。下面的输出是在当前常用的双精度实现上得到的：

```java
System.out.println(0.1 + 0.7);
System.out.println(1.0 - 0.9);
System.out.println(0.3 * 3);
System.out.println(19.99 / 10);

double price = 10.15;
double paid = 8.10;
if (price - paid == 2.05) {
    System.out.println("matched");
}
```

实际打印类似：

```text
0.7999999999999999
0.09999999999999998
0.8999999999999999
1.9989999999999999
```

判等不会进入分支，因为 `10.15 - 8.10` 的结果是 `2.0500000000000007`，和字面量 `2.05` 不是同一个双精度值。

`Java` 的 `float` 与 `double` 遵循 IEEE 754。很多十进制小数无法用有限的二进制小数写完。`0.1` 展开后是循环的 `0011`，再转回十进制已经不是精确的 `0.1`。单次误差在小数点后很远的位置，账本如果每次交易都用双精度去加，误差会沿流水累积。一天一百万笔、每笔偏一分，一个月就是一笔可观的金额。

精确的小数运算用 `BigDecimal`。第一个坑是：用 `double` 去构造它，只是把已经变脏的二进制值原样收进来，小数位变得更长，并没有变正确。

```java
System.out.println(new BigDecimal(0.1).add(new BigDecimal(0.7)));
System.out.println(new BigDecimal(1.0).subtract(new BigDecimal(0.9)));
System.out.println(new BigDecimal(0.3).multiply(new BigDecimal(3)));
System.out.println(new BigDecimal(19.99).divide(new BigDecimal(10)));
```

加法会得到一长串 `0.7999...`，而不是 `0.8`。改成字符串构造，十进制才按字面进入：

```java
System.out.println(new BigDecimal("0.1").add(new BigDecimal("0.7")));
System.out.println(new BigDecimal("1.0").subtract(new BigDecimal("0.9")));
System.out.println(new BigDecimal("0.3").multiply(new BigDecimal("3")));
System.out.println(new BigDecimal("19.99").divide(new BigDecimal("10")));
```

输出是 `0.8`、`0.1`、`0.9`、`1.999`。手头只有一个 `double` 时，不要把这个 `double` 直接送进构造器。`BigDecimal.valueOf(double)` 会先走 `Double.toString`，对“已经是双精度能精确表示的整数”比较安全；对 `0.1` 这种值，字符串本身就是近似值的十进制写法，仍然不等于数学上的 `0.1`。金额从界面或数据库来，就应该从十进制字符串或整数分进入。

更稳的做法是内部全程用“分”这个整数，只在展示时换成小数。下面的加法不会碰到二进制小数：

```java
final class Cents {
    private final long value;

    private Cents(long value) {
        this.value = value;
    }

    static Cents parse(String yuan) {
        return new Cents(new BigDecimal(yuan).movePointRight(2).setScale(0, RoundingMode.HALF_UP).longValueExact());
    }

    Cents plus(Cents other) {
        return new Cents(Math.addExact(this.value, other.value));
    }

    @Override
    public String toString() {
        return BigDecimal.valueOf(value, 2).toPlainString();
    }
}
```

`parse("10.15").plus(parse("0.10"))` 得到 `10.25`。`movePointRight(2)` 把元变成整数分，`longValueExact` 在放不进 long 时失败，而不是截断。`Math.addExact` 让两笔金额相加在溢出时抛错。展示再用标度 2 转回元。这条路径上没有 `double`。

字符串路径还会带上标度。标度是小数点右边的位数，精度是有效数字的个数。乘法的结果标度是两边标度之和，所以同样是“一百乘 1.015”，写法不同，尾随零的个数就不同：

```java
static void dump(BigDecimal value) {
    BigDecimal product = value.multiply(new BigDecimal("1.015"));
    System.out.println("scale=" + value.scale()
            + " precision=" + value.precision()
            + " product=" + product);
}

dump(new BigDecimal("100"));                 // scale 0 precision 3 product 101.500
dump(new BigDecimal(Double.toString(100d))); // scale 1 precision 4 product 101.5000
dump(BigDecimal.valueOf(100d));              // scale 1 precision 4 product 101.5000
```

`toString` 会把标度反映出来。要给用户看固定小数位，不要依赖这个默认字符串，显式规定位数和舍入。

## 格式化会把近似值四舍五入到意想不到的一侧

`String.format` 的 `%.1f` 使用四舍五入（一半向上）。`double` 和 `float` 对同一个十进制字面量的近似方向可能相反，于是格式化结果差一档。

```java
double wide = 2.35;
float narrow = 2.35f;
System.out.println(String.format("%.1f", wide));
System.out.println(String.format("%.1f", narrow));
```

得到 `2.4` 和 `2.3`。把两个值展开成精确的十进制：

```text
2.350000000000000088817841970012523233890533447265625
2.349999904632568359375
```

宽类型略大于 `2.35`，保留一位时进到 `2.4`；窄类型略小于 `2.35`，保留一位时落到 `2.3`。格式化类里走的是 `RoundingMode.HALF_UP`。

换成 `DecimalFormat` 并把模式设为向零截断，保留两位时，宽类型仍显示 `2.35`，窄类型变成 `2.34`，因为后者实际是 `2.34999...`。所以“换了格式化工具并指定舍入”仍然救不了已经存脏的二进制值。

可靠的做法是全程停留在 `BigDecimal` 上，用 `setScale` 同时指定位数和模式：

```java
BigDecimal amount = new BigDecimal("2.35");
System.out.println(amount.setScale(1, RoundingMode.DOWN));
System.out.println(amount.setScale(1, RoundingMode.HALF_UP));
```

输出 `2.3` 和 `2.4`。旧常量 `BigDecimal.ROUND_DOWN` 已经不推荐，直接用 `RoundingMode`。

## 值相等，标度不同，equals 仍然会失败

包装类型比较内容时通常用 `equals`。`BigDecimal` 是例外：它的相等同时要求数值和标度一致。`"1.0"` 的标度是 1，`"1"` 的标度是 0，所以：

```java
System.out.println(new BigDecimal("1.0").equals(new BigDecimal("1")));
```

结果是 `false`。只想知道数值是否一样，用 `compareTo` 是否返回 0：

```java
System.out.println(new BigDecimal("1.0").compareTo(new BigDecimal("1")) == 0);
```

`hashCode` 同样混入了标度。放进 `HashSet` 之后，用另一个标度去查会落空：

```java
Set<BigDecimal> byHash = new HashSet<>();
byHash.add(new BigDecimal("2.50"));
System.out.println(byHash.contains(new BigDecimal("2.5")));
```

打印 `false`。两条出路：

- 换成 `TreeSet`。它不用哈希，也不用 `equals`，而是用 `compareTo`。数值相同就视为同一个元素，上面的查找会得到 `true`。
- 放入和查找前都调用 `stripTrailingZeros()`，把尾随的零去掉，让相同数值落到同一标度。`2.50` 和 `2.500` 处理之后都能对上。注意零本身的标度会被收成科学计数法形态，展示前再 `toPlainString`。

## 整型格子满了会绕回去

`int` 和 `long` 都有上限。对 `Long` 的最大值加一：

```java
long edge = Long.MAX_VALUE;
System.out.println(edge + 1);
System.out.println(edge + 1 == Long.MIN_VALUE);
```

输出是 `-9223372036854775808` 和 `true`。溢出是静默的，没有异常，最容易在计数器、过期时间戳相加时漏掉。

一种改法是用 `Math.addExact`、`subtractExact`、`multiplyExact`。溢出时抛出 `ArithmeticException`（运行时异常）：

```java
try {
    System.out.println(Math.addExact(Long.MAX_VALUE, 1));
} catch (ArithmeticException ex) {
    System.out.println(ex.toString());
}
```

另一种是改用 `BigInteger`。它的位数随数值增长。若最后仍要收成 `long`，用 `longValueExact`，放不下时同样抛异常，而不是悄悄截断。`longValue()` 才会截断。

```java
BigInteger edge = BigInteger.valueOf(Long.MAX_VALUE);
System.out.println(edge.add(BigInteger.ONE));
try {
    edge.add(BigInteger.ONE).longValueExact();
} catch (ArithmeticException ex) {
    System.out.println(ex);
}
```

第一行得到 `9223372036854775808`，第二行提示超出 `long` 范围。

## 记几条再写代码

- 需要精确小数时用 `BigDecimal`，用十进制字符串或整数分来构造。`double` 构造器收进来的是已经近似过的值。`valueOf` 只适合“这个双精度值本身就是你要的全部信息”。
- 运算的每一步都走 `BigDecimal` 的方法。中途落回 `double` 再转回来，精度已经丢了。
- 展示用 `setScale` 写明位数和 `RoundingMode`。`String.format` 固定是一半向上，而且作用在近似值上。
- 数值比较用 `compareTo`。要放进哈希容器，先统一标度。
- 固定宽度整数的加减乘会静默绕回。用 `Math.*Exact` 把溢出变成异常，或者改用 `BigInteger`。

金额、计量、科学计数优先落在大数类型上。双精度适合可以接受相对误差的物理量，不适合分毫不能差的账。

## 留下来可以自己算一遍的问题

`RoundingMode` 一共八种。用 `1.5`、`2.5`、`-1.5`、`1.1` 收到整数，差别如下：

| 模式 | 1.5 | 2.5 | -1.5 | 1.1 |
| --- | --- | --- | --- | --- |
| UP | 2 | 3 | -2 | 2 |
| DOWN | 1 | 2 | -1 | 1 |
| CEILING | 2 | 3 | -1 | 2 |
| FLOOR | 1 | 2 | -2 | 1 |
| HALF_UP | 2 | 3 | -2 | 1 |
| HALF_DOWN | 1 | 2 | -1 | 1 |
| HALF_EVEN | 2 | 2 | -2 | 1 |
| UNNECESSARY | 只要还要舍位就抛异常 |  |  |  |

`UP` 远离零，`DOWN` 朝零，`CEILING` 朝正无穷，`FLOOR` 朝负无穷。三个 `HALF_*` 只在恰好一半时分道：向上（远离零）、向下（朝零）、或收到最近的偶数，也就是银行家舍入，用来减少连续舍入的单向偏差。

库里的小数列应定义成定点，例如 `DECIMAL(12, 2)` 存元，或干脆用整数存分。`FLOAT` / `DOUBLE` 是二进制近似，和语言里的双精度是同一类问题。查询时让数据库用定点比较，应用侧用 `BigDecimal` 接过来，不要先读成 `double`。整型按范围选 `INT` 或 `BIGINT`，并想好累加会不会超出。

另一个容易漏的点是：`BigDecimal` 的除法在除不尽且没给舍入模式时会抛 `ArithmeticException`。`new BigDecimal("1").divide(new BigDecimal("3"))` 就是这样。写商的时候永远带上标度和模式。

```java
BigDecimal unit = new BigDecimal("1").divide(new BigDecimal("3"), 2, RoundingMode.HALF_UP);
System.out.println(unit);
```

得到 `0.33`。忘记第二个和第三个参数时，异常信息是需要舍入，调用方若没接住，一笔分摊就会把整单打掉。这比静默的双精度误差更吵，但也更容易在测试里被发现，所以仍然比 `double` 适合做金额。

`stripTrailingZeros` 对零要单独看一眼。`new BigDecimal("0.00").stripTrailingZeros()` 的标度可能变成负数，`toString` 打印成科学计数法。放进集合前可以先去掉尾零，展示时再用 `toPlainString`，避免界面上出现 `0E-8` 这种形式。比较数值仍然用 `compareTo`，不要因为展示字符串不同就认为金额不同。

`float` 字面量记得加 `f`。`float narrow = 2.35;` 在较新的编译器上会因为精度丢失直接报错，或者在旧版本里先变成双精度再收窄，和 `2.35f` 不是同一条路径。格式化实验要明确写出后缀，否则你以为在比两种宽度，其实只测了双精度。
