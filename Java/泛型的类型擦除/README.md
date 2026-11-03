# 编译期的类型参数，运行期还在吗

## 为什么两个看起来不同的方法编不过

泛型让集合、返回值在编译期就带上元素类型，少写很多强转。需要记住的限制是：这些参数主要存在于编译器。生成字节码时，类型参数会被擦掉，虚拟机看到的是擦除之后的原始类型。

因此下面两个方法不能同时存在。擦除之后它们都变成 `listMethod(List)`，签名冲突：

```java
public final class Basket {
    public void listMethod(List<String> texts) {
    }

    public void listMethod(List<Integer> numbers) {
    }
}
```

编译器会提示两个方法擦除后的签名相同。用反编译器看，方法只剩下一个原始的 `List` 参数。

常见擦除规则可以记成：

- `List<String>`、`List<T>` 在字节码里都是 `List`。
- 它们的数组形式擦成 `List[]`。
- 带 `? extends` 或 `? super` 的写法，运行时同样只剩原始的 `List`；边界留给编译器插检查和强转。
- 类型变量 `T extends Serializable & Cloneable` 擦成最左边的边界 `Serializable`。多个边界时，第一个用来充当擦除后的类型。

这样设计主要是为了两件事。一是让已经发出去的、没有泛型的旧字节码继续能跑，虚拟机不必为了泛型重做对象布局。二是保留原始类型，旧代码可以把 `List` 和 `List<String>` 放在一起编译。代价是运行期分不出“字符串列表”和“整数列表”。

## 不同参数的列表，Class 却是同一个

```java
List<String> texts = new ArrayList<String>();
List<Integer> numbers = new ArrayList<Integer>();
System.out.println(texts.getClass() == numbers.getClass());
```

打印 `true`。擦除之后两者都是 `ArrayList`，不存在 `List<String>.class` 这种独立的类对象，只有 `List.class`。

## 不能 new 一个带具体参数的数组

```java
List<String>[] blocked = new List<String>[8];
```

这一行无法编译。可以声明这种数组类型，但不能创建。擦除之后它和 `List<Object>[]` 无法区分，而数组在运行期是要检查存放类型的。若允许创建，后面就能利用数组的协变把别的列表塞进去，绕过泛型检查。需要数组时只能写成 `new List<?>[8]`，并接受未检查转换的警告。更直接的做法是改用列表的列表。

## instanceof 也不能带着参数

```java
List<String> texts = new ArrayList<String>();
System.out.println(texts instanceof List<String>);
```

编译失败。运行期已经没有 `List<String>` 这个信息，检查无从做起。应当写成 `texts instanceof List<?>`，或者先判断 `instanceof List` 再逐个检查元素。

## 擦除具体做了什么

使用泛型时写上的参数，编译器在生成字节码前会删掉，这个过程叫类型擦除。`List<String>` 与 `List<Object>` 到了虚拟机里都只是 `List`。编译器会尽量在编译期拦住不安全的写法，但原始类型和泛型混用时，仍可能把错误推迟到运行期的强制转换。

和 C++ 的模板不同：模板会按每个参数组合生成一份代码；Java 的泛型在运行期只有一份擦除后的代码。

几条“怪规则”都从这里来：

- 泛型类没有按参数分开的 `Class`。静态字段也只有一份。`Box<String>` 和 `Box<Integer>` 读写的是同一个 `Box.counter`。
- 类型参数不能出现在 `catch` 里。异常匹配发生在运行期，虚拟机分不清 `Failure<String>` 和 `Failure<Integer>`，两者都只是 `Failure`。
- 不能 `new T()`，也不能把类型参数当成静态字段的类型。擦除之后编译器不知道要调用哪一个构造器。

擦除的步骤并不复杂：没有边界的类型变量换成 `Object`；写了上界就换成最左边的上界；再删掉尖括号。例如 `T get()` 变成 `Object get()`，`List<String>` 变成 `List`。如果擦完之后接口方法对不上，编译器会补一个桥方法。

```java
final class Word implements Comparable<String> {
    @Override
    public int compareTo(String other) {
        return 0;
    }
}
```

擦除后接口方法是 `compareTo(Object)`。编译器会生成一个桥方法，接收 `Object`，检查类型后转调 `compareTo(String)`。这就是为什么反射时有时能看到一个你没写过的、带 `synthetic` 或 `bridge` 标志的方法。

### 用一个会污染集合的例子看编译器在拦什么

```java
public void inspect(List<Object> items) {
    for (Object item : items) {
        System.out.println(item);
    }
    items.add(1);
}

public void reject() {
    List<String> texts = new ArrayList<String>();
    inspect(texts);
}
```

`reject` 无法编译。假如允许把 `List<String>` 传给 `List<Object>`，`inspect` 里的 `add(1)` 就会把整数放进声称只装字符串的列表，以后读出来强转时抛出类型转换异常。泛型列表是不变的：参数不同，就不能互相替代。编译器能确定违规时给错误；只能怀疑时给未检查警告。不要把警告直接消掉。

### 通配符用来表示“某一种未知类型”

`List<String>` 把元素定死为字符串。`List<?>` 表示元素类型未知，可能是字符串，也可能是整数。它不是 `List<Object>`。后者确定可以按 `Object` 放入任何对象；前者因为不知道真实参数，除了 `null` 之外不能安全地放入任何值。

```java
public void wildcard(List<?> items) {
    items.add(1);
}
```

这一行会失败。读取则总可以当成 `Object`，因为无论真实类型是什么，都是 `Object` 的子类。未知也意味着不能 `new ArrayList<?>()`。

范围可以再收窄。`List<? extends Number>` 表示读出来至少是 `Number`，于是可以调用 `intValue` 一类方法，但不能放入具体的 `Integer`（真实列表也许是 `List<Double>`）。`List<? super Integer>` 表示可以放入整数，读出来只能当成 `Object`。记法是：需要读取用上界，需要写入用下界。

### 类型系统多出来的两个方向

平时熟悉的是继承：需要 `Object` 的地方可以给 `String`，反过来要强转，编译器不保证强转成功。数组沿用了这种协变，`String[]` 可以传给 `Object[]`，类型错误推迟到写入时。

泛型加了两个独立方向：

- 参数相同的时候，关系跟容器自己的继承走。`List<String>` 是 `Collection<String>` 的子类型。
- 出现通配符时，两个方向都能展开。`Collection<? extends Number>` 的子类型，既可以是 `List<? extends Number>`，也可以是 `Collection<Integer>`，于是 `ArrayList<Long>` 也算它的子类型。
- 多个类型参数时，每个参数分别套用上面的规则。

按这个规则，前面的 `inspect` 只要改成接收 `List<?>`，`List<String>` 就能传进去。代价是方法内部不能再 `add(1)`。

### 自己写一个泛型类

声明方式和普通类一样，只是类名后多了类型参数，每个参数可以写上界。参数能出现在字段、局部变量、参数和返回值上，不能用来创建实例，也不能做静态字段的类型。

```java
final class Trio<A extends Number, B, C> {
    private A first;
    // private static B shared; 不能通过编译

    public A getFirst() {
        return first;
    }

    public void wrong() {
        // C created = new C(); 擦除后不知道构造哪一个类
    }
}
```

### 参数可以一层层传下去

外层类可以把自己的类型参数交给字段或另一个泛型类，形成一条约束链。例如仓库只保存某种实体，分页器再声明自己分页的就是这种实体：

```java
final class Page<T> {
    private final List<T> rows;
    Page(List<T> rows) {
        this.rows = rows;
    }
    public T first() {
        return rows.get(0);
    }
}

final class Repo<E> {
    Page<E> firstPage(List<E> rows) {
        return new Page<>(rows);
    }
}
```

调用 `Repo<User>` 时，`E` 在这一层被确定为 `User`，`Page<E>` 也随之确定。层次可以继续套，但每一层最终都要在某个具体类型上落地，否则调用方只能拿到上界或 `Object`。

### 写业务时的几条习惯

- 不要把 `List<String>` 和原始 `List` 混在同一个流程里。混用会带来未检查警告，并把污染推迟到运行期。实在要接很老的库，把转换关在一个小方法里。
- 见到 `?` 就把它想成一组未知类型，而不是“随便什么都能放”。未知类型上，大多数写入都不合法。
- 泛型不要和数组绑在一起。能创建的是 `new List<?>[10]` 这种弱化形式，表达力差，也容易引出未检查转换。需要可增长的格子时直接用集合。
- 编译器的警告是擦除之后仅剩的安全网，逐条看清再决定是否抑制。
