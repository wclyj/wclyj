# 注解留下的元数据，反射在运行时把它读回来

注解从 JDK 5 进入语言。它不改变方法本来的控制流，作用是给编译器或别的程序一份可机器读取的说明。注释只有人看，注解则能被编译器、注解处理器和运行中的反射代码看见。

书写形式是 `@名字`，括号里可以带参数，例如 `@SuppressWarnings(value = "unchecked")`。它可以贴在包、类、方法、字段、参数上，相当于给这些元素挂了一张卡片。卡片上的字要在运行时生效，就得保留到 `RUNTIME`，再靠反射去读。只留在源码里的注解，class 文件里根本没有。

## 编译器已经认识的三个注解

- `@Override` 定义在 `java.lang.Override`，只能标在方法上。编译器据此检查：子类是不是真的覆盖了父类或接口里的方法。签名写错时直接编译失败，而不是等到运行才发现没覆盖上。
- `@Deprecated` 定义在 `java.lang.Deprecated`，可以标在类、方法、字段上，表示不建议再使用。常见原因是有缺陷，或已经有替换物。调用处会出编译警告，程序仍能跑。
- `@SuppressWarnings` 定义在 `java.lang.SuppressWarnings`，用来压掉指定的编译警告。它和前两个不同，必须带参数，参数是约定好的警告名，不能随便写。
  - `@SuppressWarnings("all")` 把能压的都压掉，范围过大，审查时最好收窄。
  - `@SuppressWarnings("unchecked")` 只压未经检查的泛型转换。
  - `@SuppressWarnings(value = {"unchecked", "deprecation"})` 同时压两类。

压警告不是把问题修掉。能改签名、能消掉原始类型时，不要靠这个注解掩盖。

## 元注解：给注解自己贴的说明

元注解的标注对象是别的注解。标准的四个都在 `java.lang.annotation` 里：`@Target`、`@Retention`、`@Documented`、`@Inherited`。

- `@Target` 限制这张卡片能贴在哪。`ElementType.TYPE` 是类和接口，`METHOD` 是方法，`FIELD` 是字段。不写的话，很多位置都能贴，后面读的人难以判断设计意图。
- `@Retention` 决定卡片活到哪一步。三级从短到长是 `SOURCE`、`CLASS`、`RUNTIME`。`SOURCE` 编译后就丢；`CLASS` 写进 class 文件，但默认不被虚拟机装进反射可见的运行时结构；`RUNTIME` 才能在运行时用 `getAnnotation` 读到。做框架映射时几乎都要 `RUNTIME`。
- `@Documented` 表示生成文档时把这个注解也列出来。
- `@Inherited` 表示标在类上的注解，子类可以继承。它只对类这一级有效，方法上的注解不会因为写了 `@Inherited` 就跟着方法覆盖走。

定义一张只作标记、没有参数的注解，四项元注解可以一次写全：

```java
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
public @interface CargoMark {
}
```

`@CargoMark` 可以贴在类或方法上，运行时还能读到，子类会带上父类的这张类级标记。

## 自定义注解的参数其实是方法

用 `@interface` 声明注解时，它会自动实现 `java.lang.annotation.Annotation`。大括号里的每一个方法都是一个元素：方法名是参数名，返回类型是参数类型。返回类型只允许基本类型、`String`、`Class`、枚举、注解，以及它们的一维数组。用 `default` 给默认值。只有一个元素时，惯例把名字定成 `value`，这样使用处可以省略名字。没有默认值的元素，使用处必须赋值。默认值常用空串或 0；若业务上要区分“没填”和“填了 0”，可以约定 `-1` 表示未提供。

```java
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
public @interface CargoMeta {
    String title() default "";
    int weightLimit() default 0;
    int code() default -1;
    String[] hubs();
}
```

使用时，没有默认值的 `hubs` 必须写出：

```java
public class CargoDemo {
    @CargoMeta(hubs = {"east-hub"})
    public void receive() {
    }
}
```

`title` 和 `weightLimit` 可以省略。`code` 保持 `-1` 表示调用方没指定编码。注解元素在声明之后就是常量语义，不要在里面写会变的逻辑；注解类型里也不能放普通字段。

## 反射补的是“运行时还能改看法”

### 动态语言和静态语言差在哪

动态语言在运行过程中还能改结构：加函数、换对象布局、甚至引入新代码。JavaScript、Python、PHP 属于这一类。C、C++ 以及通常所说的 Java，运行时类型布局是固定的，算静态语言。

Java 不能在运行时随意给一个已加载的类添加字段，所以它不是动态语言。它又提供了反射：已知一个名字，就能问出这个类有哪些方法、构造一个实例、调用私有成员。灵活度介于完全静态和完全动态之间，常被称作“有一点动态能力”。

### 反射在问什么

`Class.forName("java.lang.String")` 这类调用，会在类加载完成后拿到方法区里那份唯一的 `Class` 对象。这份对象描述了字段、方法、父类和注解，像一面对着类结构的镜子，所以叫反射。私有成员也能看见，只是默认过不了访问检查。

运行时常见的用途：

- 判断一个对象到底是哪个类的实例。
- 在只有名字的情况下构造实例。
- 列出字段和方法，包括参数类型。
- 读方法签名上的泛型参数，弥补擦除之后的信息。
- 调用任意对象的方法、改字段。
- 读取 `RUNTIME` 注解，驱动映射或校验。
- 生成动态代理，把调用转到统一的 `InvocationHandler`。

好处是框架不必在编译期写死具体类，插件、序列化、对象关系映射都靠这一点。代价是慢：反射每次都要做查找和访问检查，属于解释执行那条路径，比直接 `new` 再调方法多好几层。能在启动时把 `Method` 查找出来并缓存，就不要放在循环里现查。

相关类型集中在 `java.lang.reflect`：`Class` 代表类型，`Method` 代表方法，`Field` 代表字段，`Constructor` 代表构造器。

## Class 对象：一个类一份

下面用包裹单来演示。三次 `forName` 同一个名字，打印出来的哈希一致，说明加载之后只有一份 `Class`。

```java
public class ClassIdentity {
    public static void main(String[] args) throws Exception {
        Class<?> a = Class.forName("lab.reflect.Parcel");
        Class<?> b = Class.forName("lab.reflect.Parcel");
        Class<?> c = Class.forName("lab.reflect.Parcel");
        System.out.println(a == b && b == c);
    }
}

class Parcel {
    private String trackingNo;
    private int weightGram;
    private int zone;

    public Parcel() {}

    public Parcel(String trackingNo, int weightGram, int zone) {
        this.trackingNo = trackingNo;
        this.weightGram = weightGram;
        this.zone = zone;
    }

    public String getTrackingNo() { return trackingNo; }
    public void setTrackingNo(String trackingNo) { this.trackingNo = trackingNo; }
    public int getWeightGram() { return weightGram; }
    public void setWeightGram(int weightGram) { this.weightGram = weightGram; }
    public int getZone() { return zone; }
    public void setZone(int zone) { this.zone = zone; }

    @Override
    @Override public String toString() {
        return trackingNo + "/" + weightGram + "g/zone" + zone;
    }
}
```

`Object` 里有 `public final Class<?> getClass()`，任何实例都能反问“我是从哪份 `Class` 造出来的”。这是从对象回到类型，和 `new` 的方向相反。

记住这几条：

- `Class` 本身也是一个类，但它的实例由虚拟机创建，业务代码不去 `new Class`。
- 一个加载器对一个全限定名只保留一份 `Class`。
- 这份对象对应一份已加载的字节码，里面能看到字段、方法、构造器、接口和注解。
- 每个普通实例都知道自己的 `Class`。要动态加载并运行某个类，第一步永远是先拿到它。

常用方法：`forName` 按名字加载并返回 `Class`；`getDeclaredConstructor().newInstance()` 调用构造器（无参的快捷形式在新版本里不再建议直接用已废弃的 `Class.newInstance`）；`getName` 返回描述这个 `Class` 的名字；`getSuperclass` 返回父类；`getInterfaces` 返回直接实现的接口；`getClassLoader` 返回定义它的加载器；`getDeclaredFields` 返回本类声明的字段，含私有；`getMethod` 按名字和参数类型找 public 方法，含继承下来的。

### 五种取得 Class 的方式

已知类型时，类字面量最稳，也最快，例如 `Parcel.class`。已经有实例时，调用 `getClass()`。只有一个字符串名字、并且它在 classpath 上时，用 `Class.forName`，名字写错会抛 `ClassNotFoundException`。基本类型的包装类提供 `TYPE`，例如 `Integer.TYPE` 就是 `int.class`。需要父类时，对已有的 `Class` 调 `getSuperclass`。另外也可以从某个 `ClassLoader` 出发去 `loadClass`，那是加载器视角，和 `forName` 的区别主要在初始化时机和用哪个加载器。

```java
class Staff {
    public String displayName;
    public Staff() {}
    public Staff(String displayName) { this.displayName = displayName; }
}

class Clerk extends Staff {
    public Clerk() { this.displayName = "clerk"; }
}

public class ObtainClass {
    public static void main(String[] args) throws Exception {
        Staff staff = new Clerk();
        Class<?> byInstance = staff.getClass();
        Class<?> byName = Class.forName("lab.reflect.Clerk");
        Class<?> byLiteral = Clerk.class;
        Class<?> primitive = Integer.TYPE;
        Class<?> parent = byInstance.getSuperclass();
        System.out.println(byInstance == byLiteral);
        System.out.println(primitive.getName());
        System.out.println(parent.getName());
    }
}
```

`byInstance` 实际是 `Clerk`，因为它的运行时类型是子类。`byName` 与 `byLiteral` 在同一个加载器下指向同一份 `Class`。

### 不只有普通类才有 Class

接口、数组、枚举、注解、基本类型、`void`，甚至 `Class` 自己，都有 `Class` 对象。数组的名字带维度前缀，例如一维 `String` 数组打印出来像 `[Ljava.lang.String;`，二维 `int` 数组像 `[[I`。

```java
Class<?> c1 = Object.class;
Class<?> c2 = Comparable.class;
Class<?> c3 = String[].class;
Class<?> c4 = int[][].class;
Class<?> c5 = Override.class;
Class<?> c6 = ElementType.class;
Class<?> c7 = Integer.class;
Class<?> c8 = void.class;
Class<?> c9 = Class.class;
```

维度和元素类型都相同的数组，共享同一份 `Class`。两个 `int[10]` 实例，`getClass()` 的结果相等，长度不算在类型里。长度不同不会产生新的 `Class`。

## 和反射有关的三块内存

- 堆：`new` 出来的对象和数组。线程共享。对象里的字段如果是引用，引用指向的还是堆上的另一个对象。
- 虚拟机栈：每个方法一帧。帧里放局部变量，基本类型直接存值，对象只存指向堆的引用。
- 方法区：线程共享，放已加载类的结构和静态变量。`Class` 对象是进入这片结构的入口。静态字段在准备阶段先被写成零值，初始化阶段才执行赋值。

反射并没有第四块内存。它只是允许你通过方法区里的 `Class`，去操作堆上的实例。

## 类加载的三步，和静态字段的最终值

程序第一次主动使用某个类，而这个类还不在内存里时，虚拟机会做三步：

- 加载：把 class 字节读进来，变成方法区里的运行时结构，并生成 `Class` 对象。
- 链接：再分成验证、准备、解析。验证确认字节符合规范；准备给静态字段分配空间并写零值；解析把常量池里的符号引用换成能直接定位的地址。
- 初始化：执行 `<clinit>`。编译器把静态字段赋值和静态块按出现顺序收进这个方法。它初始化的是类，不是某个实例。父类还没初始化时，会先初始化父类。虚拟机保证同一个类的 `<clinit>` 在并发下只跑一次。

用货架和货位看顺序。父类静态块、子类静态块、父类构造、子类构造，最后静态字段以文本顺序中后出现的那次赋值为准。

```java
class Shelf {
    static { System.out.println("shelf-static"); }
    Shelf() { System.out.println("shelf-ctor"); }
}

class Bin extends Shelf {
    static {
        System.out.println("bin-static");
        quota = 80;
    }
    static int quota = 40;
    Bin() { System.out.println("bin-ctor"); }
}
```

`new Bin()` 再打印 `quota`，顺序是：

```text
shelf-static
bin-static
shelf-ctor
bin-ctor
40
```

链接刚结束时 `quota` 还是 0。初始化里的 `<clinit>` 先执行静态块，把 `quota` 写成 80，紧接着执行字段声明上的 `quota = 40`，所以观察者看见 40。不要以为“静态块在字段后面写了就一定最后执行”，顺序以源码先后为准，声明写在块后面就会覆盖块里的赋值。

### 什么时候一定会初始化

- 启动时，含 `main` 的那个类先初始化。
- `new` 一个实例。
- 读写某个类声明的静态字段，或调用它的静态方法。`static final` 且值是编译期常量时除外。
- 用 `java.lang.reflect` 做一次会触发初始化的反射调用，例如 `Class.forName(name)` 默认会初始化。
- 初始化子类时，父类若还没有初始化，先初始化父类。

### 什么时候不会

- 通过子类名字去读父类的静态字段，只会初始化真正声明这个字段的父类。
- 创建某个类型的数组，例如 `new Bin[4]`，不会初始化 `Bin`。数组类是虚拟机另生成的。
- 读取编译期常量。常量在编译时已经抄进使用方的常量池，运行时不再触发被引用类的初始化。

所以 `Bin.MAX` 若在父类上是 `static final int MAX = 32`，打印它可能完全看不到 `shelf-static`。把 `final` 去掉，或改成运行时才算出来的值，初始化就会发生。

## 加载器分层，以及各管一段路径

加载器的工作是把字节变成方法区数据，并在堆上放一个 `Class` 作为入口。标准的应用加载器还会缓存：类一旦由它定义，就会留一段时间，直到加载器本身可以被回收。不要指望同一加载器对同一名字反复读磁盘。

JDK 8 上自下而上看，当前类的加载器一般是应用加载器，它的父级是扩展加载器，再往上是引导加载器，Java 里打印为 `null`。`Object.class.getClassLoader()` 也是 `null`，因为核心类由引导加载器定义。

```java
ClassLoader app = ObtainClass.class.getClassLoader();
ClassLoader system = ClassLoader.getSystemClassLoader();
ClassLoader ext = system.getParent();
ClassLoader boot = ext.getParent();
ClassLoader objectLoader = Object.class.getClassLoader();
```

`System.getProperty("java.class.path")` 打印的是应用加载器会搜索的那些 jar 和目录，不是整台机器上所有 `.class`。不在任何加载器路径里的文件，不会被默认加载进来，除非你自己写加载器去读。

## 把运行时结构列出来

拿到 `Class` 之后可以问：实现了哪些接口、父类是谁、有哪些构造器、方法和字段、贴了哪些注解。

```java
Class<?> type = Class.forName("lab.reflect.Parcel");
System.out.println(type.getName());
System.out.println(type.getSimpleName());

Field[] publics = type.getFields();          // 含父类 public
Field[] declared = type.getDeclaredFields(); // 仅本类，含私有
Field tracking = type.getDeclaredField("trackingNo");

Method[] allPublic = type.getMethods();
Method getter = type.getDeclaredMethod("getTrackingNo");
Method setter = type.getDeclaredMethod("setTrackingNo", String.class);
```

查找方法时把参数类型带上。只给名字会在重载时不知道要哪一个，`getDeclaredMethod` 也会直接抛找不到。`getMethods` 返回本类和父类的 public 方法；`getDeclaredMethods` 返回本类声明的全部方法，不含父类。字段的 `getFields` / `getDeclaredFields` 是同一套差别。上面注释若和你本地文档的直觉不一致，以“Declared 等于本类声明的全部，不带 Declared 的等于可见的 public 继承链”为准。

## 委派让自定义的 java.lang 类无效

想自己放一个 `java.lang.String` 时，应用加载器不会马上定义它，而是交给扩展加载器，再交给引导加载器。引导加载器在核心库里找到了同名类，就用那一份返回。下级没有机会覆盖。因此即使用不同加载器去要 `Object`，得到的也是同一份核心类型。业务类打破这个顺序的做法要非常克制，否则沙箱和类型身份一起失效。细节上，覆盖 `loadClass` 而不是只覆盖 `findClass`，才谈得上改变委派。

## 拿到 Class 之后能做的三件事

没有无参构造时，不能指望无参的实例化入口。步骤是：用 `getDeclaredConstructor` 按参数类型找到构造器，准备参数，调用 `newInstance`。构造器若是私有的，要先 `setAccessible(true)`。

调用方法：`getDeclaredMethod` 拿到 `Method`，`invoke(目标, 参数...)`。返回值就是原方法的返回值；原方法返回 `void` 时，`invoke` 得到 `null`。静态方法的目标对象可以传 `null`。没有参数时，参数数组省略或传空。私有方法要在 `invoke` 之前把可访问打开。

`setAccessible(true)` 对 `Method`、`Field`、`Constructor` 都适用。它关掉的是 Java 语言层的访问检查，不是操作系统权限。打开之后，私有成员能读写，循环里的检查开销也会下降。传 `false` 则恢复检查。这不是安全边界的替代品：安全策略如果明确拒绝，`setAccessible` 仍会失败。

```java
Class<?> type = Class.forName("lab.reflect.Parcel");
Constructor<?> ctor = type.getDeclaredConstructor(String.class, int.class, int.class);
Parcel created = (Parcel) ctor.newInstance("PX-18", 1200, 3);

Parcel blank = (Parcel) type.getDeclaredConstructor().newInstance();
Method setTracking = type.getDeclaredMethod("setTrackingNo", String.class);
setTracking.invoke(blank, "PX-19");

Field zone = type.getDeclaredField("zone");
zone.setAccessible(true);
zone.setInt(blank, 7);
```

构造路径打印出来的单号是 `PX-18`，重量 1200，分区 3。第二段先走无参构造，再反射调用 setter，单号变成 `PX-19`，分区仍是 0。关掉检查后直接写 `zone`，分区才变成 7。setter 写不进私有字段这件事并不成立——setter 是类自己的 public 方法，它内部可以写私有字段；需要 `setAccessible` 的是反射代码绕过 setter、直接改字段的那条路。

## 直接调用和反射差在检查

把 `getTrackingNo` 在循环里调用足够多次，能看出三条路径的差距：直接调用最快；每次 `invoke` 都做访问检查时最慢；对同一个 `Method` 先 `setAccessible(true)` 再循环，耗时会明显下降，往往能少掉将近一半，但仍然远慢于直接调用。查找 `Method` 的成本应放在循环外。比较时不要把 `forName` 和 `getDeclaredMethod` 算进循环，否则测到的是查找而不是调用。

JIT 还可能把没有副作用的直接调用优化掉，所以这种微基准只能用来看数量级，不能当成生产环境的精确毫秒数。生产里反射慢，通常是因为在热路径上反复查找、反复装箱，而不是因为某一次 `invoke` 本身不可接受。

## 擦除之后，签名仍留在反射里

Java 的泛型在编译后会擦除。`List<Parcel>` 在字节码的操作数里大多只剩 `List`，编译器借泛型做类型检查，并省掉一部分强转。擦除不等于“什么信息都没留下”：方法签名上的泛型会以 Signature 属性的形式留在 class 文件里，反射用专门的类型来读。

- `ParameterizedType`：参数化类型，例如 `Map<String, Parcel>`。
- `GenericArrayType`：元素是参数化类型或类型变量的数组。
- `TypeVariable`：类型变量，例如 `T extends Parcel`。
- `WildcardType`：带 `?` 的通配，含上界和下界。

```java
public class SignatureProbe {
    public void inbound(Map<String, Parcel> byTracking, List<Parcel> batch) {}

    public Map<String, Parcel> index() { return null; }

    public static void main(String[] args) throws Exception {
        Method inbound = SignatureProbe.class.getMethod(
                "inbound", Map.class, List.class);
        for (Type type : inbound.getGenericParameterTypes()) {
            if (type instanceof ParameterizedType) {
                for (Type arg : ((ParameterizedType) type).getActualTypeArguments()) {
                    System.out.println(arg);
                }
            }
        }
        Type ret = SignatureProbe.class.getMethod("index").getGenericReturnType();
        if (ret instanceof ParameterizedType) {
            for (Type arg : ((ParameterizedType) ret).getActualTypeArguments()) {
                System.out.println(arg);
            }
        }
    }
}
```

`getParameterTypes` 只能看见擦除后的 `Map` 和 `List`。要看见 `String` 和 `Parcel`，必须走 `getGenericParameterTypes`，再判断是不是 `ParameterizedType`。返回值同理，用 `getGenericReturnType`。这一套是序列化框架和依赖注入容器找回“字段到底是哪种列表”的办法。

## 用注解把类说成一张表

对象关系映射做的是三层对应：类对表、字段对列、实例对一行。框架启动时扫注解，而不是在业务代码里手写拼接。下面用两张运行时可见的注解模拟这个过程，名字不要和真实框架撞车。

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@interface TableBind {
    String value();
}

@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
@interface ColumnBind {
    String column();
    String sqlType();
    int length() default 0;
}

@TableBind("wh_parcel")
class ParcelRow {
    @ColumnBind(column = "parcel_id", sqlType = "int", length = 11)
    private int id;
    @ColumnBind(column = "weight_g", sqlType = "int", length = 11)
    private int weightGram;
    @ColumnBind(column = "tracking_no", sqlType = "varchar", length = 32)
    private String trackingNo;
}
```

读取时先拿类上的 `TableBind`，再逐个字段拿 `ColumnBind`。保留策略如果误写成 `CLASS` 或 `SOURCE`，`getAnnotation` 会得到 `null`，这是模拟映射时最常见的空白结果。

```java
public class MappingProbe {
    public static void main(String[] args) throws Exception {
        Class<?> type = ParcelRow.class;
        TableBind table = type.getAnnotation(TableBind.class);
        System.out.println(table.value());
        for (Field field : type.getDeclaredFields()) {
            ColumnBind column = field.getAnnotation(ColumnBind.class);
            if (column == null) {
                continue;
            }
            System.out.println(field.getName() + " -> " + column.column()
                    + " " + column.sqlType() + "(" + column.length() + ")");
        }
    }
}
```

输出的表名是 `wh_parcel`，三个字段分别对应 `parcel_id`、`weight_g`、`tracking_no`。真正的框架还会继续用前面的反射能力：用构造器创建行对象，用 `Field.set` 把一列的值写进私有字段，或根据泛型签名决定这一列是不是集合。注解负责“声明”，反射负责“在不知道具体类的编译期引用时仍能执行”。两者分开就只剩注释或只剩硬编码；合在一起才是映射、校验和依赖注入的共同底座。

读注解还有 `getAnnotations`（本类可见的全部）和 `getDeclaredAnnotations`。带 `@Inherited` 的类级注解，子类上 `getAnnotation` 可能取到父类的那一张；字段和方法不继承这张标记。写扫描器时要明确扫的是声明还是包含继承，避免把父类字段映射两次。

运行时拿到的注解实例并不是你写的那个接口的普通实现，而是虚拟机生成的代理。调用 `column()`、`sqlType()` 就是在读代理里缓存的元素值，不会再执行你想象中的方法体，因为注解类型里本来就没有方法体。数组元素每次调用通常返回一份拷贝，改返回数组不会改掉注解上的原始值，也不要把它当成可复用的缓存列表。

默认值只有在使用处没写该元素时才会出现。反射读到的是“最终值”，区分不出这一份是默认来的还是调用方显式写的，除非你再和注解类型上的 `getDefaultValue` 比较。扫描器如果把 `-1` 当成“未提供”，就要在文档里写死这个约定，不能假设框架会给 `null`。

还有一层容易漏：只在接口方法上标了注解，实现类的方法不会自动带上它。Spring 一类的容器有时会沿接口再找一次，那是容器的规则，不是 `Method.getAnnotation` 的默认行为。自己写扫描器时，要同时看实现类方法和它覆盖的接口方法，否则映射表会少列。类上的 `@Inherited` 也管不到这种“方法注解跟着接口走”的需求。

最后，反射改私有字段、读泛型签名、扫注解，都应放在启动或第一次请求的缓存里。热路径上保留已经解析好的 `ColumnBind` 和 `Field`，而不是每个对象都 `getDeclaredFields` 再 `getAnnotation`。这一步不改变上面的语义，但决定了映射层能不能用在批量导入上。

若字段类型本身又是泛型，例如 `List<Parcel>`，要同时用 `getGenericType` 而不是只看 `getType`。后者在擦除后只剩 `List`，列映射会丢成“一个不带元素类型的集合”。把这一步和前面的 `ParameterizedType` 判断接在同一个扫描循环里，表结构才能和 Java 字段对齐。扫描时跳过静态字段和合成字段，避免把编译器生成的成员也映射成列。
