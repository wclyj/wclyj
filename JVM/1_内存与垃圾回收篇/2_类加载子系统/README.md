# 类加载子系统

## 概述

如果只保留能让一个 Java 程序跑起来的两块骨架，一块是把字节码搬进虚拟机的类加载器，另一块是真正执行字节码的执行引擎。加载器不决定方法能不能跑通，它只负责把类型的二进制定义变成方法区里的运行时数据；能不能执行、执行得对不对，要等执行引擎接手。

磁盘上的 `.class` 像一份图纸，进了虚拟机才成为方法区里的类型元数据，随后的实例都按它布局。搬运图纸的就是类加载器。方法区还放着运行时常量池。字符串池从 JDK 7 起位于堆，不再跟永久代绑在一起。

## 加载器负责什么

加载器从文件系统或网络取得字节流。流的开头必须带有约定的标识，HotSpot 认的是 `0xCAFEBABE`。它做三件事：

- 按类型的全限定名找到定义该类型的字节流。
- 把这份静态结构翻译成方法区中的运行时结构。
- 在内存里生成一个 `java.lang.Class` 对象，作为访问该方法区数据的入口。JDK 8 的方法区实现是使用本地内存的元空间。

## 从字节码到可执行类型

下面用一个极小的入口类观察阶段，而不是把流程画成图。

```java
public class LoadPing {
    public static void main(String[] args) {
        System.out.println("loader-ok");
    }
}
```

`LoadPing` 被指定为启动类之后，虚拟机不会按源文件从上到下“执行”。它先完成加载和链接，再执行该类的初始化，然后才从 `main` 的第一条字节码开始。完整顺序是：加载，链接（验证、准备、解析），初始化。解析不一定全部发生在初始化之前，很多符号引用会拖到第一次使用时再完成。

### 加载

加载阶段只解决“字节从哪来、进内存后放哪”。它不检查方法体里的跳转是否合法，那是验证的工作；也不跑静态代码块，那是初始化的工作。

字节流的来源在规范里是开放的，常见有这些：

- 从本地磁盘直接读取，例如应用目录或模块路径上的类文件。
- 经网络取得。早期的 Applet 属于这一类。
- 从压缩包读取。`jar` 和 `war` 都建立在这一方式上。
- 在运行时生成。动态代理会在内存里拼出实现类的字节码，再交给加载器定义。
- 从别的文本生成。JSP 容器先把页面编译成类，再加载。
- 从加密或变换后的字节中还原。这是防止类文件被直接反编译的一种手段，加载前要先解密。
- 从数据库或其他存储把类文件捞出来。规范并不禁止。

### 链接：验证

验证用来确认字节流符合当前虚拟机的要求，避免有缺陷或被篡改的类损害虚拟机。HotSpot 大致做四类检查。

- 文件格式。魔数、版本号、常量池标签是否能识别。用二进制查看工具看文件头，合法类文件以 `CA FE BA BE` 开头。版本号高于当前虚拟机能支持的范围，会在这里失败。
- 元数据。父类是否存在、是否继承了被标成 final 的类、字段和方法描述符是否合法。
- 字节码。对方法体做数据流分析，检查类型是否匹配、跳转目标是否落在指令边界、局部变量在使用前是否已赋值。
- 符号引用。在把常量池里的名字变成直接指针时，确认目标类、字段、方法存在，并且当前类有权访问。

格式不对的文件到不了初始化。

### 链接：准备

准备阶段为类变量分配内存，并写入该类型的零值，而不是源码里看到的初值。

```java
public class ZeroThenAssign {
    private static int level = 6;

    public static void main(String[] args) {
        System.out.println(level);
    }
}
```

`level` 在准备阶段是 `0`。源码里的 `6` 要等初始化阶段执行 `<clinit>` 才写进去。`main` 打印出来的已经是 `6`，因为打印发生在初始化完成之后。

有一类例外：被 `static final` 修饰、而且在编译期就能确定的基本类型或字符串常量，会写入常量池的 `ConstantValue` 属性，准备阶段就按这个显式值放置，不再先放零值。例如 `static final int CAP = 32;`，准备结束后就是 `32`。

准备阶段不为实例字段分配空间。实例字段随对象在堆上分配，并在那里做零值填充。类变量在 JDK 8 及更早的逻辑模型里属于方法区一侧的静态存储。

### 链接：解析

解析把常量池中的符号引用换成直接引用。符号引用是一组名字和描述符，按规范写在类文件里；直接引用是指向目标的指针、相对偏移，或可以间接定位目标的句柄。

解析主要面向类或接口、字段、类方法、接口方法、方法类型等，对应常量池里的 `CONSTANT_Class`、`CONSTANT_Fieldref`、`CONSTANT_Methodref` 等项。

很多解析会推迟。虚拟机可以在初始化完成之后、第一次执行到相关指令时再解析。因此“链接在初始化之前全部做完”只对验证和准备成立，对解析不成立。

### 初始化

初始化就是执行类构造器 `<clinit>`。这个方法不是程序员写的，编译器把静态字段的赋值语句和静态代码块按源码出现顺序合并而成。类里如果没有任何静态字段赋值和静态代码块，可以没有 `<clinit>`。

`<clinit>` 不是实例构造器。实例构造器在虚拟机里叫 `<init>`，每个类在编译后至少有一个。若存在父类，虚拟机保证子类的 `<clinit>` 执行前，父类的 `<clinit>` 已经结束。

```java
public class InitTrace {
    private static int left = 3;

    static {
        left = 8;
        right = 50;
    }

    private static int right = 12;

    public static void main(String[] args) {
        System.out.println(left);
        System.out.println(right);
    }
}
```

打印结果是 `8` 和 `12`。`left` 先被赋成 `3`，静态块里再改成 `8`。`right` 在静态块中先被写成 `50`，声明处的 `12` 位于静态块之后，会把 `50` 覆盖。若在静态块里、`right` 的声明之前读取 `right`，编译器会拒绝，这叫非法的前向引用。写可以，读不行。

父子类的顺序可以用下面的类型观察。

```java
public class ParentFirst {
    static class Base {
        static int seed = 4;
        static { seed = 11; }
    }

    static class Child extends Base {
        static int copied = seed;
    }

    public static void main(String[] args) {
        System.out.println(Child.copied);
    }
}
```

输出是 `11`。触发 `Child` 初始化时，虚拟机先完成 `Base` 的 `<clinit>`：`seed` 先写成 `4`，再被静态块改成 `11`，然后 `Child` 才读取 `seed`。反编译 `Base` 的 `<clinit>`，能看到先把常量 `4` 用 `putstatic` 写入，再用 `bipush` 写入 `11`，最后 `return`。

多条线程同时首次使用同一个类时，`<clinit>` 会被加锁，只执行一次。其他线程阻塞，直到这次初始化结束或失败。

```java
public class OnceInit {
    public static void main(String[] args) {
        new Thread(() -> touch(), "alpha").start();
        new Thread(() -> touch(), "beta").start();
    }

    static void touch() {
        System.out.println(Thread.currentThread().getName() + " enter");
        new Gate();
    }
}

class Gate {
    static {
        System.out.println(Thread.currentThread().getName() + " runs clinit");
        if (true) {
            while (true) { }
        }
    }
}
```

两条线程都会打印 `enter`，但 `runs clinit` 只会出现一次。另一条线程停在类初始化锁上，不会再次进入静态块。若 `<clinit>` 抛出异常，这个类型会被标记为初始化失败，以后的使用直接抛出 `ExceptionInInitializerError` 一类的错误，而不会悄悄重试成功。

## 加载器如何划分

虚拟机规范把加载器分成引导类加载器，以及所有由 Java 代码实现、派生自 `ClassLoader` 的加载器。后者在规范用语里都算“自定义加载器”，范围比日常说的“自己写的加载器”大，扩展加载器和应用加载器也包括在内。

实际排错时最常见的仍是三层，外加开发者自己写的加载器。它们是委派关系，不是继承关系，也不是“下层包含上层”的内存包含关系。

```java
public class LoaderProbe {
    public static void main(String[] args) {
        ClassLoader app = ClassLoader.getSystemClassLoader();
        ClassLoader ext = app.getParent();
        ClassLoader boot = ext.getParent();
        System.out.println(app);
        System.out.println(ext);
        System.out.println(boot);
        System.out.println(LoaderProbe.class.getClassLoader());
        System.out.println(String.class.getClassLoader());
    }
}
```

在 JDK 8 上，前两行分别是 `AppClassLoader` 和 `ExtClassLoader` 的实例，第三行是 `null`。当前类由应用加载器定义，`String` 的 `getClassLoader()` 也是 `null`。`null` 在这里表示引导类加载器，不是“没有加载”。JDK 9 之后扩展加载器被平台类加载器取代，类名不再是 `sun.misc.Launcher$ExtClassLoader`，但引导类加载器仍然不能用一个 Java 对象表示，`getParent()` 和核心类的 `getClassLoader()` 依然可能是 `null`。

### 引导类加载器

引导类加载器由 C/C++ 实现，嵌在虚拟机内部，不继承 `java.lang.ClassLoader`，也没有 Java 层面的父加载器。它装载虚拟机自己运行所需要的核心库。JDK 8 上主要是 `JAVA_HOME/jre/lib` 下的 `rt.jar`、`resources.jar` 以及 `sun.boot.class.path` 指出的其他路径。出于安全考虑，它只处理 `java`、`javax`、`sun` 等受信任包名开头的类。它还会创建扩展加载器和应用加载器，并成为它们委派链的顶端。

### 扩展类加载器

JDK 8 的扩展加载器用 Java 写成，实现类是 `sun.misc.Launcher$ExtClassLoader`，父加载器是引导类加载器。它从系统属性 `java.ext.dirs` 指定的目录，或 JDK 安装目录下的 `jre/lib/ext` 读取库。用户如果把 jar 丢进这个目录，也会由它加载。JDK 9 取消了扩展机制，对应角色变成平台类加载器，扩展目录不再是受支持的扩展点。

### 应用类加载器

应用类加载器也叫系统类加载器。JDK 8 的实现是 `sun.misc.Launcher$AppClassLoader`，父加载器是扩展类加载器。它负责环境变量 `CLASSPATH` 或系统属性 `java.class.path` 上的类，是普通应用的默认加载器。`ClassLoader.getSystemClassLoader()` 返回的就是它。

### 自己实现的加载器

日常三类加载器配合就够用。需要隔离、更换来源或保护字节码时，再写加载器。典型动机有：同一份类文件在不同加载器里各加载一份，从而隔离框架和业务；从数据库、网络或加密字节中取流；避免核心包被应用目录里的同名类替换。

做法是继承 `java.lang.ClassLoader`。JDK 1.2 之后不建议覆盖 `loadClass`，否则容易把委派顺序写坏。自定义的查找逻辑应放在 `findClass` 里。需求只是“从若干目录或 URL 读类”时，可以直接继承 `URLClassLoader`，少写读字节流的代码。

### 引导加载器看得见的路径

JDK 8 可以用下面的程序列出引导路径，而不是依赖某个写死的安装盘符。

```java
public class BootPathDump {
    public static void main(String[] args) {
        String path = System.getProperty("sun.boot.class.path");
        for (String item : path.split(java.io.File.pathSeparator)) {
            System.out.println(item);
        }
        System.out.println(javax.crypto.Cipher.class.getClassLoader());
    }
}
```

输出里通常能看到 `resources.jar`、`rt.jar`、`sunrsasign.jar`、`jsse.jar`、`jce.jar`、`charsets.jar`、`jfr.jar`，有的发行版还有一个 `classes` 目录。`Cipher` 位于这些 jar 中，它的加载器打印为 `null`，说明由引导加载器定义。JDK 9 引入模块之后，`sun.boot.class.path` 不再可用，核心类改由模块路径提供。

### 取得 ClassLoader 的几种入口

- 某个类型的定义加载器：`clazz.getClassLoader()`。
- 当前线程的上下文加载器：`Thread.currentThread().getContextClassLoader()`。
- 系统加载器：`ClassLoader.getSystemClassLoader()`。
- 调用方加载器。应使用 `StackWalker` 一类公开 API，不要依赖已移走的内部方法。

`sun.misc.Launcher` 只存在于 JDK 8 的启动过程。JDK 9 之后不要再假设它一定在。

## 双亲委派

虚拟机按需加载：第一次使用某类型时才去找它的类文件。查找时采用委派，而不是每个加载器各扫一遍自己的路径。

### 工作步骤

- 加载器收到请求后，不先自己读盘，而是交给父加载器。
- 父加载器继续向上，直到引导类加载器。
- 父加载器能定义这个类，就把结果返回。父加载器确认自己的路径里没有，子加载器才在自己的路径里查找。

这里的“父”是组合出来的委派对象，不是 Java 继承意义上的父类。应用加载器的父加载器是扩展加载器，扩展加载器再委派给引导加载器。

### 数据库驱动为什么看起来是反着来的

`java.sql.Driver` 这类接口由引导加载器定义，具体驱动在应用的 jar 里，引导加载器看不见。JDK 使用线程上下文类加载器把加载请求交回应用加载器，这就是 SPI 里常见的反向委派。JDBC 驱动注册走的就是这条路：核心接口仍由引导加载器定义，实现类由上下文加载器去加载。它没有废除委派，而是在父加载器无法看到实现类时，显式指定另一个加载器。

### 沙箱

如果在应用目录放一个 `java.lang.String`，并给它写上 `main`，真正被执行的仍是核心库里的 `String`。引导加载器会先定义 `rt.jar`（或模块）中的那一份，应用加载器没有机会再用同名类替换它。核心类里没有你写的 `main`，于是启动失败。另外，即使用自定义加载器，也不允许定义 `java.` 开头的类，虚拟机会直接拒绝。这就是沙箱要保住的边界：核心 API 不能被类路径上的同名类型篡改。

### 这样设计的好处

- 同一个全限定名不会被不同层级重复定义进同一条委派链。
- 核心类型的身份稳定。任意库都不能用自己的 `java.lang.String` 混进已有的堆对象体系。

## 怎样判断两个 Class 是同一个类型

同时满足两点，虚拟机才认为是同一个类：

- 全限定名一致，包名也算在内。
- 定义它们的类加载器实例是同一个。

哪怕字节完全相同、跑在同一个进程里，只要定义加载器不同，这两个 `Class` 就不相等，它们的实例也不能互相赋值。虚拟机会把用户加载器的引用存在类型信息里。解析一个类型对另一个类型的引用时，要保证双方的加载器关系符合加载约束，否则抛出链接错误。

## 主动使用和被动使用

主动使用才会触发初始化，规范里主要有七种：

- 创建实例，例如执行 `new`、反射调用构造器。
- 读取或设置某个类、接口的静态字段。编译期常量除外，见下面的易错点。
- 调用静态方法。
- 使用反射，例如 `Class.forName("demo.kit.OrderHook")`。默认会初始化；若传入的参数明确要求不初始化，则只完成加载和链接。
- 初始化某个子类时，先初始化其父类。
- 虚拟机启动时，被指定为启动类的那个类，要先初始化再进入 `main`。
- JDK 7 起，`java.lang.invoke.MethodHandle` 解析到 `getstatic`、`putstatic`、`invokestatic` 对应的方法句柄时，若目标类还没初始化，则初始化它。

除此之外的使用都是被动使用，不会导致初始化。例如：通过子类名访问父类的静态字段，初始化的是声明该字段的父类，子类可以仍未初始化；声明一个数组，只需要数组类型本身，不会初始化元素类型；访问编译期常量。

## 参数与观察手段

加载顺序没有总开关。JDK 8 用 `-verbose:class` 查看类型由谁加载、来自哪条路径；JDK 9 起改用 `-Xlog:class+load=info`。两者只增加日志，不改变委派，生产上不宜长期全开。

## 易错点

- 准备阶段的零值和初始化阶段的赋值不是同一次写入。在 `<clinit>` 跑完之前，别的线程如果能观察到对象，可能看见静态字段仍是零值。发生这种情况，通常是初始化过程中把 `this` 过早发布了出去。
- `static final` 的编译期常量不会触发类的初始化。使用方在编译时就把值抄进了自己的常量池。你改了常量所在类却没重新编译使用方，运行起来仍是旧值。
- 覆盖 `loadClass` 并且忘记先委派，会破坏沙箱，让应用目录里的类挡住核心类。查找逻辑应写在 `findClass`。
- 同一个类文件被两个加载器各定义一次，里面的静态字段就有两份，单例、缓存都会“失效”。容器里出现 `ClassCastException` 而类名又完全一样时，先比较两侧的加载器。
- JDK 8 的扩展目录和 `sun.misc.Launcher` 不能套用到 JDK 11 及以后。模块路径和平台类加载器才是新的结构，委派的方向仍然是先父后子。
