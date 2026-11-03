# 代码块

普通代码块就是方法或语句中间的一对花括号，用来收紧局部变量的作用域。

静态代码块在类被装进内存时执行，整个类加载过程里只跑一次，和后来 `new` 了几次无关。

普通代码块和它旁边的普通语句谁先谁后，只看书写顺序，写在上面的先执行。

和构造方法容易混的是实例初始化块：它写在类里、方法外，没有 `static`。每创建一个对象都要跑一遍，而且跑在该构造方法的其余语句之前。

## 代码一

```java
public class LocalBlockLab {
    public static void main(String[] args) {
        {
            int width = 21;
            System.out.println("第一段宽度=" + width);
        }
        {
            int height = 34;
            System.out.println("第二段高度=" + height);
        }
        int width = 55;
        System.out.println("方法里的宽度=" + width);
    }
}
```

花括号一结束，里面的 `width` 就消失了，所以下面可以再声明同名变量。输出顺序就是书写顺序。

```text
第一段宽度=21
第二段高度=34
方法里的宽度=55
```

如果把 `int width = 55` 挪到第一段花括号之前，第一段里的同名声明会编译失败。普通代码块不改变“先写先执行”，只限制变量能活到哪里。

## 代码二

实例初始化块和构造方法放在一起时，块并不按“离构造方法近”来排序，而是按在类体里出现的先后，并且全部先于构造方法体。

```java
public class InitBlockLab {
    {
        System.out.println("靠前的实例块");
    }

    public InitBlockLab() {
        System.out.println("构造方法体");
    }

    {
        System.out.println("靠后的实例块");
    }

    public static void main(String[] args) {
        new InitBlockLab();
        System.out.println("----");
        new InitBlockLab();
    }
}
```

```text
靠前的实例块
靠后的实例块
构造方法体
----
靠前的实例块
靠后的实例块
构造方法体
```

`new` 最终会进入构造方法，但编译器会把实例块挪到构造方法体的最前面，并且放在对父构造的调用之后。所以每次创建对象，块都会再跑，静态块则不会出现在这次输出里。优先级可以记成：实例块先于本类构造方法体，但晚于父类构造。

## 代码三

静态块、实例块、构造方法分属三个时机。再把另一个类的加载插进来，就能看见“谁先被用到，谁才加载”。

```java
class Helper {
    public Helper() {
        System.out.println("Helper 构造");
    }

    {
        System.out.println("Helper 实例块");
    }

    static {
        System.out.println("Helper 静态块");
    }
}

public class LoadOrderLab {
    {
        System.out.println("LoadOrder 实例块");
    }

    static {
        System.out.println("LoadOrder 静态块");
    }

    public LoadOrderLab() {
        System.out.println("LoadOrder 构造");
    }

    public static void main(String[] args) {
        System.out.println("进入主方法");
        new Helper();
        System.out.println("----");
        new Helper();
        System.out.println("----");
        new LoadOrderLab();
    }
}
```

```text
LoadOrder 静态块
进入主方法
Helper 静态块
Helper 实例块
Helper 构造
----
Helper 实例块
Helper 构造
----
LoadOrder 实例块
LoadOrder 构造
```

要执行 `main`，虚拟机必须先加载 `LoadOrderLab`，于是它的静态块最先出现，而且只出现一次。打印“进入主方法”时，`Helper` 还没被碰到，所以它的静态块还没跑。第一次 `new Helper()` 触发加载，静态块先于实例块，实例块又先于构造方法。第二次再创建时类已在内存里，静态块跳过。最后创建主类的实例时，主类早就加载过了，只剩实例块和构造方法。

静态块里如果调用了别的类的静态成员，会把那个类的加载提前插进来。排查“日志顺序和想象不一样”时，先找是谁在静态初始化里碰了别的类。

## 代码四

有继承时，加载和创建都要先父后子。子类构造方法即使不写，开头也有一次对父构造的调用。

```java
class Elder {
    {
        System.out.println("父类实例块");
    }

    public Elder() {
        System.out.println("父类构造");
    }

    static {
        System.out.println("父类静态块");
    }
}

class Junior extends Elder {
    public Junior() {
        System.out.println("子类构造");
    }

    {
        System.out.println("子类实例块");
    }

    static {
        System.out.println("子类静态块");
    }
}

public class FamilyBlockLab {
    public static void main(String[] args) {
        System.out.println("主方法开始");
        new Junior();
        System.out.println("----");
        new Junior();
        System.out.println("----");
        new Elder();
    }
}
```

```text
主方法开始
父类静态块
子类静态块
父类实例块
父类构造
子类实例块
子类构造
----
父类实例块
父类构造
子类实例块
子类构造
----
父类实例块
父类构造
```

第一次 `new Junior()` 时，子类要先加载，加载子类又必须先加载父类，所以两段静态块按父到子各跑一次。接着进入子类构造，隐式的父构造先执行；父构造体之前先跑父类实例块。父对象就绪后，才轮到子类实例块和子类构造体。

第二次创建子类，两个类都已加载，静态块不再出现，只把“父实例块 → 父构造 → 子实例块 → 子构造”再走一遍。

单独 `new Elder()` 不会去加载子类，更不会跑子类的块。父类静态块在第一次创建子类时已经跑过，这里也不会再打印。

把顺序收成三句：

- 类加载走父到子，每个类的静态块只在加载时执行一次
- 创建对象走父到子，每一层都是实例块先于该层构造方法体
- 子类构造的第一件事是父类构造，父类没完成，子类实例块不会开始
