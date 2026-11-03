# Lambda表达式

## 为什么要用Lambda表达式

Lambda 是一段没有名字的函数。它让行为可以像数据一样被传来传去，调用方只关心“要做什么”，不必先声明一个只被用一次的类。写法更短，读的人更容易看见真正的条件，而不是看见一层模板代码。

## 实例代码

它最先替掉的，就是只为了实现一个方法而写的匿名类。下面用比较器给整数排顺序，先看匿名类要写多少样板。

```java
import java.util.Comparator;
import java.util.TreeSet;

public class CompareBefore {
    public static void fill() {
        Comparator<Integer> ordering = new Comparator<Integer>() {
            @Override
            public int compare(Integer left, Integer right) {
                return Integer.compare(left, right);
            }
        };
        TreeSet<Integer> ranks = new TreeSet<>(ordering);
        ranks.add(42);
        ranks.add(7);
    }
}
```

改成箭头之后，参数和比较逻辑留在同一行，集合只负责接收这个比较器。

```java
import java.util.Comparator;
import java.util.TreeSet;

public class CompareAfter {
    public static void fill() {
        Comparator<Integer> ordering = (left, right) -> Integer.compare(left, right);
        TreeSet<Integer> ranks = new TreeSet<>(ordering);
        ranks.add(42);
        ranks.add(7);
    }
}
```

## 策略设计模式

需求是从快递员里挑出驾龄超过 8 年的人。把“怎么判定”抽成接口，筛选过程就不用跟着条件一起改。

```java
public class Courier {
    private final String name;
    private final int years;
    private final double bonus;

    public Courier(String name, int years, double bonus) {
        this.name = name;
        this.years = years;
        this.bonus = bonus;
    }

    public String getName() { return name; }
    public int getYears() { return years; }
    public double getBonus() { return bonus; }
}
```

判定接口只留一个方法：

```java
public interface Gate<T> {
    boolean allow(T item);
}
```

按驾龄实现一次：

```java
public class YearsGate implements Gate<Courier> {
    @Override
    public boolean allow(Courier courier) {
        return courier.getYears() > 8;
    }
}
```

调用处准备名单，再把判定对象交进去。注意要遍历原始名单，结果放到另一个列表，否则会在遍历时改到正在迭代的集合。

```java
public static List<Courier> roster() {
    return Arrays.asList(
            new Courier("周宁", 4, 3200),
            new Courier("韩雪", 11, 8600),
            new Courier("马林", 9, 5400),
            new Courier("邱岩", 6, 9100),
            new Courier("苏晚", 2, 2800)
    );
}

public static List<Courier> pick(List<Courier> all, Gate<Courier> gate) {
    List<Courier> chosen = new ArrayList<>();
    for (Courier one : all) {
        if (gate.allow(one)) {
            chosen.add(one);
        }
    }
    return chosen;
}

public static void byYears() {
    List<Courier> chosen = pick(roster(), new YearsGate());
    chosen.forEach(c -> System.out.println(c.getName()));
}
```

第二天条件改成奖金高于 8000。筛选循环一行都不用动，只要再写一个实现。

```java
public class BonusGate implements Gate<Courier> {
    @Override
    public boolean allow(Courier courier) {
        return courier.getBonus() > 8000;
    }
}
```

使用时把 `new YearsGate()` 换成 `new BonusGate()` 即可。算法骨架固定、可变步骤交给子类或实现类，这就是策略模式。

## 匿名内部类

条件一多，实现类也会变多，包里堆满只被用一次的类型。可以把判定直接写在调用点，少一个源文件。

```java
public static void byAnonymous() {
    pick(roster(), new Gate<Courier>() {
        @Override
        public boolean allow(Courier courier) {
            return courier.getBonus() <= 4000;
        }
    });
}
```

行为还是那一个抽象方法，只是类没有名字。样板仍在：要写类型、要写方法签名、要写注解。

## Lambda表达式

函数式接口只有一个抽象方法时，匿名类可以再收成箭头。参数对应抽象方法的参数，箭头右边就是方法体。

```java
public static void byLambda() {
    List<Courier> chosen = pick(roster(), c -> c.getBonus() <= 4000);
    chosen.forEach(System.out::println);
}
```

连自定义接口都可以省掉，标准库的流已经把过滤、截断和遍历串好了。

```java
public static void byStream() {
    roster().stream()
            .filter(c -> c.getBonus() >= 5000)
            .limit(2)
            .map(Courier::getName)
            .forEach(System.out::println);
}
```

这里先留下奖金不低于 5000 的人，最多取两名，再只打印姓名。

## 学习Lambda

Java 8 增加了 `->`。它把一段 Lambda 分成左右两边。

- 左边是参数，对应函数式接口里抽象方法的参数
- 右边是要执行的逻辑，也就是这个抽象方法的实现

### 语法格式

#### 1、无参，无返回值

没有参数就写一对空括号。方法体只有一条语句时，花括号可以不写。

```java
Runnable job = () -> System.out.println("早班开始");
```

和匿名类对照：

```java
public static void contrast() {
    Runnable classic = new Runnable() {
        @Override
        public void run() {
            System.out.println("早班开始");
        }
    };
    Runnable compact = () -> System.out.println("早班改由箭头执行");
    classic.run();
    compact.run();
}
```

箭头里用到外面的局部变量时，这个变量必须是事实上的最终变量：后面不能再赋值。从 8 开始不必亲手写成 `final`，编译器会检查。

```java
int lane = 4;
Runnable show = () -> System.out.println("当前通道" + lane);
```

如果在 `show` 创建之后又写 `lane = 5`，上面的箭头就无法编译。

#### 2、有一个参数，有返回值

一个参数时，圆括号可以省略。没有返回值、方法体又只有一条，花括号和分号按语句规则取舍。

```java
Consumer<String> print = msg -> System.out.println(msg);
print.accept("夜航日志");
```

带括号的 `(msg) -> ...` 同样合法。参数类型可以不写，见后面的类型推断。

#### 3、有多个参数，一个返回值

两个以上参数必须加括号。方法体如果有多条语句，要用花括号，并且把结果 `return` 出去。

```java
public static void multiLine() {
    Comparator<Integer> ordering = (left, right) -> {
        System.out.println("正在比较 " + left + " 与 " + right);
        return Integer.compare(left, right);
    };
    System.out.println(ordering.compare(15, 9));
}
```

#### 4、有多个参数，只有一条语句

右边只剩一条表达式时，花括号和 `return` 一起省掉，表达式的值就是返回值。

```java
public static void oneLine() {
    Comparator<Integer> ordering = (left, right) -> Integer.compare(left, right);
    System.out.println(ordering.compare(3, 11));
}
```

一旦加上花括号，就必须显式返回，不能再把 `return` 省略。

### 类型推断

参数类型通常可以不写，编译器根据赋值左侧的目标类型推断。下面两种写法等价。

```java
Comparator<Integer> explicit = (Integer left, Integer right) -> Integer.compare(left, right);
Comparator<Integer> inferred = (left, right) -> Integer.compare(left, right);
```

推断不等于不检查。左边如果不是函数式接口，或者参数个数对不上，编译照样失败。类型检查只是挪到了编译器，没有消失。

## 函数式接口

箭头能赋值成功，前提是目标类型是函数式接口：接口里只能有一个抽象方法。默认方法和静态方法不算在这个唯一抽象方法里。

```java
@FunctionalInterface
public interface Gate<T> {
    boolean allow(T item);
}
```

`@FunctionalInterface` 不是运行时必须的，但它会让编译器帮忙盯着。以后如果有人往接口里再加一个抽象方法，实现类还没改，这里就会先报错。`Runnable`、`Comparator`、`Consumer` 都是标准库里的这类接口。

## 场景

把“对一个整数做什么运算”也做成参数。接口只约定输入和输出，具体是平方还是取负，由调用方现场决定。

```java
@FunctionalInterface
public interface IntOp {
    int apply(int raw);
}

public static int compute(int raw, IntOp op) {
    return op.apply(raw);
}

public static void demo() {
    int squared = compute(12, n -> n * n);
    int negated = compute(12, n -> -n);
    System.out.println(squared + "," + negated);
}
```

`compute` 不需要知道运算细节，因此加一种运算不必改这个方法。

## 训练

用 `Collections.sort` 给定制顺序：先比驾龄，驾龄相同再比姓名。比较器直接用箭头，不必另写类。

```java
public static void sortRoster() {
    List<Courier> team = new ArrayList<>(roster());
    Collections.sort(team, (a, b) -> {
        if (a.getYears() == b.getYears()) {
            return a.getName().compareTo(b.getName());
        }
        return Integer.compare(a.getYears(), b.getYears());
    });
    team.stream().map(Courier::getName).forEach(System.out::println);
}
```

驾龄升序时，输出顺序是苏晚、周宁、邱岩、马林、韩雪。若两人驾龄相同，则按姓名的字典序决定先后。

## Java内置函数接口

标准库在 `java.util.function` 里准备了几组通用接口。多数业务只要从里面挑一个，不必再声明自己的 `Gate` 或 `IntOp`。按“有没有入参、有没有返回值”来记最不容易混。

## Consumer 消费型接口

形状是 `Consumer<T>`，抽象方法 `void accept(T t)`。传进一个值，只产生副作用，不把结果交回来。

```java
public static void pay(double amount, Consumer<Double> consumer) {
    consumer.accept(amount);
}

public static void demoConsume() {
    pay(256.5, money -> System.out.println("已入账 " + money));
}
```

打印、落日志、写一条消息，都属于消费。如果还要把处理结果交回去，就不是这个接口。

## Supplier 供给型接口

形状是 `Supplier<T>`，抽象方法 `T get()`。调用时不给入参，每次取一个新值。适合延迟创建，或者每次都要一个新的随机数。

```java
public static List<Integer> draw(int times, Supplier<Integer> supplier) {
    List<Integer> bag = new ArrayList<>();
    for (int i = 0; i < times; i++) {
        bag.add(supplier.get());
    }
    return bag;
}

public static void demoSupply() {
    List<Integer> marks = draw(6, () -> (int) (Math.random() * 50));
    marks.forEach(System.out::println);
}
```

六次调用会得到六个 0 到 49 的整数，具体数字每次运行不同。供给函数里如果做了重活，只有真正调用 `get` 时才会执行。

## Function 函数型接口

形状是 `Function<T, R>`，抽象方法 `R apply(T t)`。吃进一种类型，变出另一种类型，没有强制的副作用。

```java
public static String reshape(String raw, Function<String, String> fn) {
    return fn.apply(raw);
}

public static void demoMap() {
    String code = reshape("  cargo-42 ", text -> text.trim().toUpperCase());
    System.out.println(code);
}
```

输出是 `CARGO-42`。映射可以继续套：先 `trim`，再截取，只要类型接得上。

## Predicate 断言型接口

形状是 `Predicate<T>`，抽象方法 `boolean test(T t)`。用来做判断，结果只有真和假。流里的 `filter` 收的就是它。

```java
public static List<String> keep(List<String> raw, Predicate<String> predicate) {
    List<String> hit = new ArrayList<>();
    for (String token : raw) {
        if (predicate.test(token)) {
            hit.add(token);
        }
    }
    return hit;
}

public static void demoTest() {
    List<String> ports = Arrays.asList("bay", "harbor", "dock", "wharf", "pier");
    keep(ports, token -> token.length() > 4).forEach(System.out::println);
}
```

长度大于 4 的是 `harbor` 和 `wharf`。断言还可以用 `and`、`or`、`negate` 组合，不必把所有条件挤进一个箭头。

## 扩展

上面四个接口都只有一个输入。两个输入的场景要换成对应的双参数版本：`BiConsumer` 消费一对值，`BiFunction` 把一对值映射成结果，`BinaryOperator` 是两端类型相同的 `BiFunction`，`BiPredicate` 对一对值做判断。

基本类型如果走通用接口，会被装箱。热路径上可以改用 `IntConsumer`、`LongFunction`、`DoublePredicate` 这类特化接口，避免反复把 `int` 包成 `Integer`。选接口时先看参数个数和返回值，再看要不要避开装箱。
