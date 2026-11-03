# 等等与equals的区别

## 前言

“双等号比地址，equals 比值”这句话只说对了一半，拿去面试会被追问倒。

- 双等号：两边是基本类型时比数值；两边是引用时比“是不是同一个对象”。两个分别 `new` 出来的对象，双等号几乎总是 false
- `equals`：定义在 `Object` 上，默认实现就是双等号。子类可以改写。`String` 改写之后比的是字符序列，不再比是不是同一块内存

和 `equals` 一起改写的通常还有 `hashCode`。只改其中一个，放进散列集合时会出现“逻辑上相等、集合里却是两条”的坑。

## 代码一

```java
import java.util.HashSet;
import java.util.Set;

public class IdentityLab {
    static class Member {
        private final String nick;

        Member(String nick) {
            this.nick = nick;
        }
    }

    public static void main(String[] args) {
        String a = new String("kiwi");
        String b = new String("kiwi");
        System.out.println(a == b);
        System.out.println(a.equals(b));
        Set<String> boxed = new HashSet<>();
        boxed.add(a);
        boxed.add(b);
        System.out.println(boxed.size());

        System.out.println("----");

        String c = "pear";
        String d = "pear";
        System.out.println(c == d);
        System.out.println(c.equals(d));
        Set<String> literals = new HashSet<>();
        literals.add(c);
        literals.add(d);
        System.out.println(literals.size());

        System.out.println("----");

        Member m1 = new Member("kiwi");
        Member m2 = new Member("kiwi");
        System.out.println(m1 == m2);
        System.out.println(m1.equals(m2));
        Set<Member> people = new HashSet<>();
        people.add(m1);
        people.add(m2);
        System.out.println(people.size());
    }
}
```

结果为 false true 1，true true 1，以及 false false 2。两次 `new` 引用不同，`String` 按字符比较所以集合只留一条。相同字面量复用常量池。`Member` 未改写相等性，集合留下两条。只改相等不改哈希，会走错桶。

## 代码二

常量折叠和 `intern` 是第二组容易混的点。编译期就能算出的字符串会直接进常量池；运行期拼接出的新对象默认不在池里，除非手动实习。

```java
public class PoolLab {
    public static void main(String[] args) {
        String base = "kiwi";
        String fresh = new String("kiwi");
        String again = "kiwi";
        String tail = "pie";
        String folded = "kiwi" + "pie";
        String mixed = again + tail;

        System.out.println("base == fresh : " + (base == fresh));
        System.out.println("base.equals(fresh) : " + base.equals(fresh));
        System.out.println("base == again : " + (base == again));
        System.out.println("folded == \"kiwipie\" : " + (folded == "kiwipie"));
        System.out.println("base == mixed : " + (base == mixed));
        System.out.println("folded == mixed : " + (folded == mixed));
        System.out.println("folded.equals(mixed) : " + folded.equals(mixed));
        System.out.println("base == mixed.intern() : " + (base == mixed.intern()));
        System.out.println("folded == mixed.intern() : " + (folded == mixed.intern()));
        System.out.println("base == fresh.intern() : " + (base == fresh.intern()));
    }
}
```

结果：

```text
base == fresh : false
base.equals(fresh) : true
base == again : true
folded == "kiwipie" : true
base == mixed : false
folded == mixed : false
folded.equals(mixed) : true
base == mixed.intern() : false
folded == mixed.intern() : true
base == fresh.intern() : true
```

`intern` 会到字符串池里找一份字符相同的实例：已经有就返回池中那个，没有就把当前引用放进去。因此只有 `equals` 为真时，两边的 `intern` 结果才是同一引用。字面量和编译期能算完的拼接会自动进池，所以 `folded` 与 `"kiwipie"` 相同。`mixed` 含变量，运行期才拼出来，双等号失败，实习之后才能和 `folded` 对上，却仍然不等于 `base`。不要拿它当业务去重。
