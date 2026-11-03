# 方法引用与构造器引用

## 方法引用

### 概念

箭头右边若只是调用一个现成方法，就可以改写成方法引用。它仍然是函数式接口的实例，编译器按目标抽象方法去匹配参数和返回值。

### 格式

- 某个对象::实例方法
- 某个类::静态方法
- 某个类::实例方法

### 注意

- 被引用方法的参数、返回值要和抽象方法一致
- 第一个参数是接收者、后面的参数才是入参时，写成 `类名::实例方法`

### 对象::实例方法名

```java
import java.util.function.Consumer;
import java.util.function.Supplier;

public class BoundMethodRef {
    public static void main(String[] args) {
        Consumer<String> bound = System.out::println;
        bound.accept("栈桥已关闭");
        Shift slot = new Shift("夜航", 3);
        Supplier<String> read = slot::label;
        System.out.println(read.get());
    }

    static final class Shift {
        private final String label;
        private final int lane;
        Shift(String label, int lane) {
            this.label = label;
            this.lane = lane;
        }
        String label() { return label + "#" + lane; }
    }
}
```

`println` 收一个字符串且无返回值，才能接到 `Consumer`。`label` 无参并返回字符串，对应 `Supplier`。

### 类::静态方法名

```java
import java.util.Comparator;

public class StaticMethodRef {
    public static void main(String[] args) {
        Comparator<Integer> linked = Integer::compare;
        System.out.println(linked.compare(18, 24));
    }
}
```

`Integer.compare` 是静态方法，两个整数进、一个整数出，和比较器同形。

### 类::实例方法名

```java
import java.util.function.BiPredicate;

public class UnboundMethodRef {
    public static void main(String[] args) {
        BiPredicate<String, String> linked = String::equals;
        System.out.println(linked.test("north", "south"));
    }
}
```

调用时第一个参数当接收者，第二个当实参。

## 构造器引用

### 格式

- 类名::new

### 注意

选哪个构造器，看目标接口的参数列表。无参方法对无参构造，单参数对单参数构造。重载同时存在时，要看赋值给了哪一种接口。

### 代码

```java
import java.util.function.Function;
import java.util.function.Supplier;

public class CtorRefLab {
    public static void main(String[] args) {
        Supplier<Badge> fresh = Badge::new;
        Function<Integer, Badge> numbered = Badge::new;
        System.out.println(fresh.get().code());
        System.out.println(numbered.apply(27).code());
    }
    static final class Badge {
        private final int code;
        Badge() { this.code = -1; }
        Badge(int code) { this.code = code; }
        int code() { return code; }
    }
}
```

同一个 `Badge::new`，赋给 `Supplier` 走无参构造，赋给 `Function` 走带整数的构造。

## 数组引用

### 格式

- 数组类型::new

数组没有普通构造器，但分配仍然可以引用。接口要接收长度、返回数组。

```java
import java.util.function.Function;

public class ArrayRefLab {
    public static void main(String[] args) {
        Function<Integer, String[]> linked = String[]::new;
        String[] row = linked.apply(6);
        row[0] = "泊位";
        System.out.println(row.length + row[0]);
    }
}
```

`apply(6)` 得到长度为 6 的空数组。`int[]::new` 同理，目标类型改为 `Function<Integer, int[]>`。
