# 一行判等，会把对象身份和业务身份混在一起

判等写错不一定每次都爆。包装类型用 `==` 时，小数字碰巧指向同一个缓存实例，测试是绿的；数值一过缓存上界就变成另一张单。自定义类型只重写了相等、没重写哈希，放进集合之后“明明一样却查不到”。排序用的比较和相等各看各的字段，二分查找会指向另一个人。这些问题可以藏很久。

下面按操作符、相等契约、哈希、比较器和代码生成的顺序看。

## 操作符比的是槽里的那份值

基本类型的值就是数值本身，只能用 `==`。引用类型槽里放的是地址。`==` 问的是不是同一个对象，`equals` 才由类型自己定义“内容算不算同一份”。

因此：比 `int`、`long` 用 `==`；比 `Integer`、`Long`、`String` 以及自己的类型，要用 `equals`，除非你确实在问是不是同一个实例。

包装类型偶尔用 `==` 也能得到 true，是缓存和拆箱在帮忙，不是规则变了。下面五组把这几条路分开。数值选在缓存内的 100 和缓存外的 200，默认缓存是闭区间 [-128, 127]。

```java
Integer a = Integer.valueOf(100);
Integer b = Integer.valueOf(100);
System.out.println(a == b);

Integer c = Integer.valueOf(200);
Integer d = Integer.valueOf(200);
System.out.println(c == d);

Integer e = Integer.valueOf(100);
Integer f = Integer.valueOf(100).intValue() == 100 ? new Integer(100) : null;
System.out.println(e == f);

Integer g = new Integer(100);
Integer h = new Integer(100);
System.out.println(g == h);

Integer boxed = Integer.valueOf(200);
int raw = 200;
System.out.println(boxed == raw);
```

`new Integer` 已不推荐，这里只为表明“新实例不进缓存”。五次结果是 true、false、false、false、true。

`valueOf` 在范围内返回缓存数组里的同一个实例，所以 100 和 100 是同一地址。200 超出默认上界，每次都新建，地址不同。`new` 出来的对象始终不走缓存。最后一组一边是包装类型、一边是基本类型，包装类型先拆成 int 再比数值，所以为 true。

缓存上界可以用 `-XX:AutoBoxCacheMax` 抬高，下界仍是 -128。抬高之后 200 也可能变成 true。这更说明不能依赖它：换一个启动参数，判等结果就变了。

业务上只关心编号是否相同，就写 `Objects.equals(left, right)`。基本类型没有这个方法，继续用 `==`。

缓存边界可以用一段循环看清，不必背源码。126 和 127 两次自动装箱会得到 true，128 起变成 false：

```java
for (int n = 126; n <= 129; n++) {
    Integer left = n;
    Integer right = n;
    System.out.println(n + " same instance? " + (left == right));
}
```

`valueOf` 的逻辑可以收成几行自己的伪代码，方便记住，而不是依赖某次碰巧的结果：

```java
static Integer boxed(int n, Integer[] cache, int low, int high) {
    if (n >= low && n <= high) {
        return cache[n - low];
    }
    return new Integer(n);
}
```

`low` 固定是 -128。`high` 默认 127，启动参数只能把它抬高，不能改低到 127 以下。数组在类加载时填好，所以范围内的每次装箱都返回同一格。范围外的 `new` 不进这张表。比较两个包装类型时若写成 `==`，就等于在问“这次有没有都命中缓存”，而不是在问数值。

`Objects.equals` 会先处理 null：两边都空则相等，只有一边空则不相等，否则才调用 `equals`。手写 `left.equals(right)` 时，`left` 自己为空会抛空指针。状态码这种外部传入的字段，用 `Objects.equals` 更稳。

一个真实事故是枚举里的状态码声明成了 `Integer`，外部查询对象里的状态也是 `Integer`，比较时用了 `==`。状态从 120 往上长，早期都落在缓存里，线上是好的；某天新状态用了 200，同一状态被判断成不同。看枚举常量 `PAID(120, "已付款")` 时，很像在传基本类型，字段声明才是包装类型。包装类型之间不要用 `==`。静态检查能标出这种写法，值得打开。

字符串有另一套“看起来是同一个对象”的机制，目的是省内存。类文件里的字面量会进常量池，相同内容的字面量编译后指向同一实例。`new String("k")` 一定在堆上新建。`intern` 会再查常量池，有则返回池里的引用，没有则放进去。

```java
String p = "k";
String q = "k";
System.out.println(p == q);

String r = new String("m");
String s = new String("m");
System.out.println(r == s);
System.out.println(r.intern() == s.intern());
System.out.println(r.equals(s));
```

四次结果是 true、false、true、true。前三次都是在比地址，只有最后一次在比内容。业务代码比字符串用 `equals`。

`intern` 不是免费的规范化。常量池是固定桶数的表。桶太少、不同字符串又特别多时，每个桶后面拖着长链，查找从近似常数变成沿链扫描。JDK 8 的默认桶数大约六万；当前环境的 JDK 21 启动后表长是 65536。把上千万个互不相同的数字字符串全部驻留，平均链长会到一百以上。`-XX:StringTableSize` 可以把桶加大，`-XX:+PrintStringTableStatistics` 在退出时打印桶数、条目数和平均桶长。需要驻留时先估算不同字符串的规模，让平均桶长接近 1，而不是在请求路径上对用户输入调用 `intern`。

## 自己写相等要先挡住空和类型

`Object.equals` 就是 `this == obj`。`Integer` 和 `String` 能比内容，是因为它们重写了方法：先看是不是同一引用，再看类型，然后逐个字段比。

自定义类型不重写的话，内容一样也是 false。一个格子只关心行和列：

```java
final class Cell {
    private final int row;
    private final int col;
    private final String label;

    Cell(int row, int col, String label) {
        this.row = row;
        this.col = col;
        this.label = label;
    }
}
```

`new Cell(1, 2, "a")` 与 `new Cell(1, 2, "a")` 用默认相等仍是 false，比的是两块不同的堆内存。

若业务规定行和列相同就是同一个格子，有人会写成直接强转：

```java
@Override
public boolean equals(Object other) {
    Cell that = (Cell) other;
    return row == that.row && col == that.col;
}
```

三个用例会拆穿它。传入 `null` 时，访问 `that.row` 抛出空指针。传入普通 `Object` 时，强转抛出类型转换异常。只有传入另一个行和列相同的 `Cell` 才返回 true。

可用的写法是：同一引用直接返回；对方为 null 或运行时类不同则返回 false；然后再强转并比较字段。描述文字不参与相等。

```java
@Override
public boolean equals(Object other) {
    if (this == other) {
        return true;
    }
    if (other == null || getClass() != other.getClass()) {
        return false;
    }
    Cell that = (Cell) other;
    return row == that.row && col == that.col;
}
```

先比引用是省掉后面的字段比较。先排除 null 和不同类型，强转才安全。字段要和业务身份一致，不要把展示用的标签算进去，除非标签也是身份的一部分。

把失败的三种入参各自接住，日志会是空指针、类型转换失败，以及行和列相同时的 true。修过之后，空和普通对象都安静地返回 false，不再把异常当成“不相等”的信号。用异常来表达不相等会把真正的故障淹没，调用方也无法区分“类型写错了”和“格子不同”。

相等还要满足对称和传递。只重写一边、或者子类用 `instanceof` 再比一个父类没有的字段，就会出现 `parent.equals(child)` 为 true 而反过来为 false。下面的子类把标签也算进身份，父类却只用 `instanceof` 比行列，对称性立刻坏掉：

```java
class LooseCell {
    final int row;
    final int col;

    LooseCell(int row, int col) {
        this.row = row;
        this.col = col;
    }

    @Override
    public boolean equals(Object other) {
        if (!(other instanceof LooseCell)) {
            return false;
        }
        LooseCell that = (LooseCell) other;
        return row == that.row && col == that.col;
    }
}

final class TaggedCell extends LooseCell {
    final String label;

    TaggedCell(int row, int col, String label) {
        super(row, col);
        this.label = label;
    }

    @Override
    public boolean equals(Object other) {
        if (!(other instanceof TaggedCell)) {
            return false;
        }
        return super.equals(other) && label.equals(((TaggedCell) other).label);
    }
}
```

`new LooseCell(1, 2).equals(new TaggedCell(1, 2, "a"))` 走父类，只看行列，得到 true。反过来子类发现对方不是 `TaggedCell`，得到 false。哈希表无法同时满足这种关系。父类改成比较 `getClass()` 之后，两边都是 false，对称性回来了，代价是子类和父类永远不会相等。没有明确的“子类可以替代父类”设计时，选后一种。

## 哈希必须和相等看同一组字段

相等改完之后，下面这个集合用例仍然失败：

```java
Cell left = new Cell(1, 2, "a");
Cell right = new Cell(1, 2, "b");
Set<Cell> cells = new HashSet<>();
cells.add(left);
System.out.println(cells.contains(right));
```

按上面的相等，两者应当算同一个格子，`contains` 却是 false。哈希表先用 `hashCode` 找桶，默认实现大体上随身份而变，两个实例几乎不会进同一个桶，后面的 `equals` 根本不会被调用。

契约是：相等为 true 的两个对象，哈希必须相同。反过来哈希相同不要求相等。用同一组字段生成哈希：

```java
    @Override
    public int hashCode() { return 31 * row + col; }
```

补上之后，`equals(null)` 为 false，和普通对象比为 false，行和列相同则为 true，集合也能 `contains`。

字段如果会变，放进集合之后再改，对象会掉进别的桶，既查不到也删不掉，相当于把条目泄漏在表里。参与哈希的字段保持为 final。

可以自己走一遍查找。假设哈希对 16 取模后落在 5 号桶。`contains` 先算探针的哈希，进 5 号桶，再沿链表或红黑树用相等确认。哈希若按默认的身份来，探针和已经放入的格子不在 5 号桶，相等根本不会被调用，于是返回 false。把哈希改成只由行和列计算之后，两者进入同一桶，相等返回 true，`contains` 才为 true。

这也是不要只写相等的原因。集合、映射的键、以及一些缓存框架，都是先哈希再相等。单元测试若只调用 `equals`，这条路径是绿的，一放进 `HashSet` 就坏。测试里至少断言一次 `contains`。

```java
Set<Cell> cells = new HashSet<>();
cells.add(new Cell(1, 2, "a"));
System.out.println(cells.contains(new Cell(1, 2, "b")));
```

标签不同但行列相同，修复后应打印 true。若打印 false，先看哈希是不是把标签也算进去了，或者相等仍在比标签。两边字段集合不一致时，相等为 true 的对象可能哈希不同，契约被破坏，集合行为没有定义。

手写容易漏步骤。开发环境可以按字段生成这两个方法。生成之后仍要看一眼：是不是把不该参与身份的字段也算进去了。

## 排序用的比较要和相等一致

有人把线性的 `indexOf` 换成先排序再二分，想把复杂度从按长度扫描降到对数。查找标准若和相等不一致，会“找到”另一个元素。

```java
final class Book implements Comparable<Book> {
    private final int isbn;
    private final String title;

    Book(int isbn, String title) {
        this.isbn = isbn;
        this.title = title;
    }

    @Override
    public int compareTo(Book other) {
        return Integer.compare(this.isbn, other.isbn);
    }
}
```

列表里有 `(1, "zhang")` 和 `(2, "wang")`，查找 `(2, "li")`。`indexOf` 走 `equals`。在没有重写相等时，`equals` 比的是实例，结果是 -1，这是对的：没有这本书。`Collections.binarySearch` 走 `compareTo`，只看编号，会返回编号为 2 的那一本的下标。

若相等已经按编号和书名一起算，比较器却只看编号，二分查找的契约就被破坏了：比较结果为 0 的两者，相等必须为 true。修正是让比较器看同一组字段，次序要固定：

```java
@Override
public int compareTo(Book other) {
    int byTitle = this.title.compareTo(other.title);
    if (byTitle != 0) {
        return byTitle;
    }
    return Integer.compare(this.isbn, other.isbn);
}
```

容易写错有两个原因。一是相等和哈希常常由生成器按全部字段产生，人只盯着自己写的 `compareTo`。二是 `compareTo` 要返回一个数，于是随手挑了唯一的整数段。自定义类型只要实现 `Comparable`，相等、哈希、比较三者看同一组身份字段。

把两种查找放进同一段程序，差别就固定下来。相等同时看编号和书名，比较却只看编号：

```java
final class Book implements Comparable<Book> {
    final int isbn;
    final String title;

    Book(int isbn, String title) {
        this.isbn = isbn;
        this.title = title;
    }

    @Override
    public boolean equals(Object other) {
        if (this == other) {
            return true;
        }
        if (other == null || getClass() != other.getClass()) {
            return false;
        }
        Book that = (Book) other;
        return isbn == that.isbn && title.equals(that.title);
    }

    @Override
    public int hashCode() { return 31 * isbn + title.hashCode(); }

    @Override
    public int compareTo(Book other) {
        return Integer.compare(isbn, other.isbn);
    }
}

List<Book> shelf = new ArrayList<>();
shelf.add(new Book(1, "zhang"));
shelf.add(new Book(2, "wang"));
Book probe = new Book(2, "li");
int linear = shelf.indexOf(probe);
Collections.sort(shelf);
int binary = Collections.binarySearch(shelf, probe);
```

`linear` 是 -1，列表里没有书名为 li 的那本。`binary` 是 1，指向书名为 wang 的那本，因为比较结果为 0 就被当成命中。二分查找的契约写的是：比较为 0 时，相等也为 true。这里破坏了契约，所以两种 API 给出不同答案。若业务身份只是编号，相等和哈希也只留编号，两种查找才会同时命中。若身份包含书名，比较器就要先比书名再比编号，并且列表在二分之前按这个完整比较器排序。只改查找算法、不改比较字段，复杂度看起来好了，结果是错的。

`TreeSet` 同样只看比较。把这两本书放进树，再加入探针，比较为 0 的探针会被当成重复元素丢弃，集合大小仍是 2。放进 `HashSet` 则因为相等为 false，大小变成 3。同一个“是否已存在”的问题，两种集合答案不同。选树的时候，比较器就是身份；选哈希的时候，相等和哈希才是身份。不要假设它们会互相调用。

## 生成器默认不算父类，也会把所有字段都算上

`@Data` 会带上相等和哈希，默认使用所有非静态、非瞬态字段。两个证件号相同、姓名不同的人会被判成不同的人。若业务身份只是证件号，在姓名上标排除，生成结果里就只剩证件号。

有继承时默认更危险。子类员工多一个公司字段：

```java
class PersonRecord {
    private final String name;
    private final String documentNo;

    PersonRecord(String givenName, String documentNo) {
        name = givenName;
        this.documentNo = documentNo;
    }
}

class StaffMember extends PersonRecord {
    private final String employer;

    StaffMember(String name, String documentNo, String employer) {
        super(name, documentNo);
        this.employer = employer;
    }
}
```

不打开 `callSuper` 时，生成的相等只看子类自己的字段。两个公司同为 `"acme"`、证件号分别是 `"001"` 和 `"002"` 的员工会被判成相等。生成器不会自动把父类字段算进来。

显式写成 `@EqualsAndHashCode(callSuper = true)` 之后，子类先比自己的公司，再调用父类的相等。父类若只按证件号算，这两个员工就不再相等。打开开关的前提是父类的相等已经有意义；父类仍用 `Object` 的引用比较时，子类调用上去之后，内容相同的父类部分也会被判成不同。

用一段不依赖生成器的代码把两种行为固定下来，避免只在编译产物里才看得见。类名换成证件持有人和职员，避免和业务里已有的类型撞车：

```java
class Holder {
    private final String name;
    private final String documentNo;

    Holder(String givenName, String documentNo) {
        name = givenName;
        this.documentNo = documentNo;
    }

    boolean sameDocument(Holder other) {
        return documentNo.equals(other.documentNo);
    }
}

final class Staff extends Holder {
    private final String employer;

    Staff(String name, String documentNo, String employer) {
        super(name, documentNo);
        this.employer = employer;
    }

    boolean same(Staff other, boolean includeParent) {
        if (other == null || !employer.equals(other.employer)) {
            return false;
        }
        return !includeParent || sameDocument(other);
    }
}
```

`includeParent` 为 false 时，两名雇主都是 `acme`、证件号不同的职员会被当成同一个人。为 true 时先比雇主，再比证件号，结果分开。生成器的 `callSuper = false` 就是前一种；打开之后才是后一种。

生成结果大致还会先比引用、再判断类型、再逐个字段处理 null。阅读编译产物时重点看字段列表里有没有父类的证件号，而不是看它用了多少行空值判断。`@Data` 同时会生成 `toString`，日志里把证件号打出来是另一件事，不要为了日志去改相等。排除注解只影响相等和哈希，不会从 `toString` 里拿走字段。需要隐藏敏感字段时，两处都要处理。

瞬态字段和静态字段默认不参与。把缓存放进 `transient` 是合适的，把业务身份放进 `transient` 则会导致反序列化前后相等结果变化。生成器不会警告这件事。

## 收束

比内容：基本类型用 `==`，包装类型和字符串用 `equals`。`==` 在缓存范围内、字面量之间、或者一边是基本类型时会碰巧成立，不能当成规则。

自己的类型要参与相等，就成对实现 `equals` 和 `hashCode`，字段集合保持一致，并且不要在进入哈希容器之后修改这些字段。还要排序或放进树时，`compareTo` 为 0 的时候相等必须为 true。

生成器默认纳入全部普通字段，并且不调用父类。用排除注解拿掉展示字段，有继承时显式打开 `callSuper`，再读一遍生成出来的方法是不是业务身份。

## 两个容易混的实现细节

相等里用 `getClass` 还是 `instanceof`？`getClass` 要求运行时类相同，对称性好守：父类对象和子类对象互不为相等。`instanceof` 允许子类通过父类的检查。子类若再增加字段，就可能出现父类认为相等、子类认为不相等，对称性被破坏，哈希容器的行为也会乱。没有“子类可以替代父类参与相等”的明确设计时，用 `getClass`。

`TreeSet.contains` 和 `HashSet.contains` 不是同一条路。哈希集合用哈希找桶，再用相等确认。树集合用比较器或 `compareTo`，比较结果为 0 就当作同一个元素，根本不调用相等。比较和相等不一致时，树里会把两个业务上不同的对象当成重复，哈希集合则能同时留下它们。这正好对应前面书号和书名的那个错误。

```java
Book onShelf = new Book(2, "wang");
Book probe = new Book(2, "li");
Set<Book> byHash = new HashSet<>();
byHash.add(onShelf);
Set<Book> byTree = new TreeSet<>();
byTree.add(onShelf);
System.out.println(byHash.contains(probe));
System.out.println(byTree.contains(probe));
```

在“相等看编号加书名、比较只看编号”的实现下，第一行是 false，第二行是 true。若这不是你要的身份，就改比较器，而不是换成另一种集合来碰运气。树还要求比较是全序，和相等一致只是其中一条；违反传递性时，树的查找会漏元素，甚至抛出非法参数。比较器里不要调用会变化的外部状态。

再看一个会悄悄破坏传递性的写法：金额用 `BigDecimal`，比较时只看 `compareTo`，相等却用 `equals`。`"1.0"` 和 `"1"` 数值相同、标度不同，比较结果是 0，相等却是 false。放进 `TreeSet` 时后者会被当成已经存在；放进 `HashSet` 时两者都在。这和书号、书名的分裂是同一类契约问题，只是字段换成了标度。数值身份用 `compareTo` 定义时，不要再把标度留在 `equals` 里，除非你根本不把它们放进树。

空集合和空字符串也要单独测。`Objects.equals(null, null)` 为 true，但实例方法 `left.equals(right)` 在 left 为空时抛出空指针。外部传入的包装类型，测试里应包含“两边都空、只有一边空、值相同、值不同”四组，而不是只测缓存范围内的两个非空小整数。只测 100 和 100，`==` 也会通过。
