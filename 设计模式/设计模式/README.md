# 设计模式

## 前言

后面的例子都围绕三条原则展开：

- 调用方依赖抽象，而不是依赖某个具体类。抽象稳定之后，替换实现不会把调用方一起改掉。
- 单一职责。一个类只封装一件事，并且把这件事做完整。工厂只负责装配对象，不负责把业务规则也塞进来。
- 对修改封闭，对扩展开放。已经交付的行为不要为了新需求去改核心分支；新能力通过新增类或新增实现接进来。

创建型模式解决“对象从哪里来”。结构型模式调整类和对象的组合方式，让变化不要互相缠住。行为型模式把算法步骤或对象之间的职责分配固定下来。创建型入门快，结构型和行为型更能看出设计的差别。

代理、模板这类带继承或包装的模式，建议把示例自己写一遍，只看类图不容易发现调用顺序。

## 创建型模式

创建对象最直接的写法是 `new` 之后再逐个设属性。类一旦要交给别的开发者用，就不应把构造细节暴露出去。工厂的本质，是用一个方法调用代替调用方手写的 `new`。

工厂这一族通常分成三档：简单工厂、工厂方法、抽象工厂。创建时不向调用方暴露分支逻辑，调用方只拿一个共同的父类型。

### 简单工厂模式

下面用快递面单做例子。`Parcel` 是共同父类，`SameCityParcel` 和 `AirParcel` 是两种具体单据。

```java
public class ParcelWorkshop {
    public static Parcel build(String kind) {
        if ("same-city".equals(kind)) {
            Parcel local = new SameCityParcel();
            local.setLane("市内骑手");
            return local;
        }
        if ("air".equals(kind)) {
            Parcel flight = new AirParcel();
            flight.setLane("航空干线");
            return flight;
        }
        throw new IllegalArgumentException("未知面单类型: " + kind);
    }
}
```

调用方只跟车间打交道：

```java
public class DispatchDesk {
    public static void main(String[] args) {
        Parcel one = ParcelWorkshop.build("air");
        ParcelWorkshop.build("same-city");
    }
}
```

简单工厂通常就是这样一个 `XxxWorkshop`：一个静态方法根据参数返回同一父类的不同子类。车间的唯一职责是生产 `Parcel`，调度台不再自己 `new`。构造细节集中在一处，调用代码更短。

它的弱点也很明显。每增加一种面单，都要回去改 `build` 里的分支，违反开闭原则。种类少、变化慢时，这个代价可以接受。

### 工厂模式

种类一多，或者产品线本身就要分开维护，就该升级到工厂方法：先选工厂，再由工厂生产。工厂是接口，每条产品线一个实现。

```java
public interface ParcelWorkshop {
    Parcel build(String speed);
}

public class InlandWorkshop implements ParcelWorkshop {
    @Override
    public Parcel build(String speed) {
        if ("express".equals(speed)) {
            return new InlandExpress();
        }
        if ("economy".equals(speed)) {
            return new InlandEconomy();
        }
        throw new IllegalArgumentException(speed);
    }
}

public class OverseasWorkshop implements ParcelWorkshop {
    @Override
    public Parcel build(String speed) {
        if ("express".equals(speed)) {
            return new OverseasExpress();
        }
        if ("economy".equals(speed)) {
            return new OverseasEconomy();
        }
        throw new IllegalArgumentException(speed);
    }
}
```

`InlandExpress`、`InlandEconomy`、`OverseasExpress`、`OverseasEconomy` 都是 `Parcel` 的子类。

```java
public class DispatchApp {
    public static void main(String[] args) {
        ParcelWorkshop workshop = new InlandWorkshop();
        Parcel parcel = workshop.build("express");
    }
}
```

两次都调用 `build("express")`，国内车间和海外车间交出来的对象完全不同。关键在第一步：调用方必须先决定实例化哪一个车间。

日志也是同一结构。定义 `AuditSinkFactory`，`FileAuditSinkFactory` 把记录写入本地文件，`QueueAuditSinkFactory` 把记录写入消息队列。客户端先选定工厂，后面的写入路径就被这条选择锁死。

### 抽象工厂模式

工厂方法一次只解决一个产品等级。一旦要同时生产一组必须配套的对象，就进入抽象工厂，也就是产品族。

造一套骑手终端可以说明这一点。终端包含扫码模组、打印模组和定位模组，三者必须来自同一供应商，否则协议对不上。与其分别做“扫码工厂”“打印工厂”，不如做一个终端工厂，由它一次交出一整族配件。

同一工厂产出的、分属不同等级的产品构成一个产品族。例如北厂同时生产冷链箱和温控仪，南厂也生产这两种设备，但接口互不兼容。北厂的冷链箱只和北厂的温控仪一起用。

抽象工厂的定义是：为调用方提供一个接口，用来创建一组相关或互相依赖的对象，调用方不需要点名具体类，就能拿到同一族里不同等级的产品。

它是工厂方法的升级。工厂方法只覆盖一个等级；抽象工厂覆盖多个等级。适合它的系统一般满足：

- 存在多个产品族，每个具体工厂负责一族里的各个等级
- 一次只消费其中一族，同族产品配套使用

相对工厂方法，它还有两点好处：

- 一族里的多个等级可以在同一个工厂里一起管理，不必为每个等级再引入一套平行的管理类
- 新增一个产品族时，只要增加一个具体工厂，不必改已有工厂，符合开闭原则

缺点是：族内要增加一个新等级（例如再加一块电池模组）时，抽象工厂和每一个具体工厂都要加方法。这正好撞上“不要为新需求去改已有类”。所以产品等级稳定、产品族会增加时，抽象工厂很合适；等级本身还在膨胀时，它会变成负担。

### 单例模式

有些对象在整个进程里只应该出现一份，例如配置中心客户端、序号生成器。构造器设为私有，外面不能 `new`。对外只留一个静态方法返回那份引用。多次调用拿到的是同一个对象，不是一份拷贝。

三个结构要点：

- 构造器私有
- 类内部持有自己类型的静态引用
- 用静态方法把这份引用交出去

**饿汉式**

类加载时就把实例建好。

```java
public final class AppClock {
    private static final AppClock INSTANCE = new AppClock();

    private AppClock() {}

    public static AppClock get() {
        return INSTANCE;
    }
}
```

写法简单，线程安全，因为类加载由虚拟机保证只执行一次。缺点是：即使业务还没用到时钟，实例也已经占了内存。初始化很重、而且未必会用到时，这个代价不划算。

**懒汉式与双重检查**

需要时再创建。只在方法上加 `synchronized` 也能做对，但每次获取都进锁，热点路径开销大。双重检查把锁缩小到第一次创建：

```java
public final class AppClock {
    private AppClock() {}

    private static volatile AppClock instance;

    public static AppClock get() {
        if (instance == null) {
            synchronized (AppClock.class) {
                if (instance == null) {
                    instance = new AppClock();
                }
            }
        }
        return instance;
    }
}
```

两次判空都要留。外层避免已经建好之后还去抢锁；内层避免两个线程同时穿过外层后各建一个对象。`volatile` 不能省。没有它，对象的赋值可能在构造完成之前就被别的线程看见，读到半初始化的实例。

**嵌套类**

更干净的写法是把实例放进一个嵌套类，利用类加载的懒惰性：

```java
public final class AppClock {
    private AppClock() {}

    private static class Holder {
        private static final AppClock INSTANCE = new AppClock();
    }

    public static AppClock get() {
        return Holder.INSTANCE;
    }
}
```

外部类加载时不会去加载 `Holder`。第一次调用 `get()` 才触发 `Holder` 的初始化，虚拟机保证这份静态初始化是线程安全的。嵌套类和内部类不是一回事：静态嵌套类不持有外部实例，只能碰到外部类的静态成员；真正的内部类隐含着一个外部对象引用。

**枚举**

枚举常量在类加载时全部初始化，虚拟机保证不会再被反射或序列化额外造出一份。因此单元素枚举天生就是单例，序列化、反射都比较难把它破坏。需要懒加载、或者实例化可能抛受检异常时，枚举就不如嵌套类灵活。

### 建造者模式

对象字段很多、而且有的组合不合法时，一长串构造参数很难读。建造者把赋值拆成链式步骤，最后用 `build()` 一次产出不可变对象。它仍是创建型：调用方决定有哪些零件，导演类（或建造者自己）决定装配顺序。适合配置对象、报文头这种“步骤固定、取值可选”的场景。

### 原型模式

当创建一个对象的成本高于复制，或者新对象只是某个样板的小改动，可以用原型。Java 里常见的做法是实现拷贝方法，返回一份字段独立的新对象。要注意浅拷贝和深拷贝：嵌套的可变对象如果只复制引用，修改副本会污染原型。

## 结构型模式

创建型解决对象的诞生。结构型通过改变组合关系来解耦，让以后替换某一块时不用翻整棵调用链。

### 代理模式

代理用一个和真实对象相同的接口挡住实现细节，并可以在真实调用前后插入额外逻辑。客户端只认代理，代理自己不完成业务，只是一层外壳，但对外必须表现得像那个真实对象。

接口：

```java
public interface LabelPrinter {
    Label print(String waybillNo);
}
```

真实实现只做打印：

```java
public class LaserLabelPrinter implements LabelPrinter {
    @Override
    public Label print(String waybillNo) {
        Label label = new Label();
        label.setWaybillNo(waybillNo);
        label.setBody("面单已排版");
        return label;
    }
}
```

代理持有真实打印机，在调用前后记账、补盖章：

```java
public class AuditedPrinter implements LabelPrinter {
    private final LabelPrinter delegate = new LaserLabelPrinter();

    @Override
    public Label print(String waybillNo) {
        audit("start " + waybillNo);
        Label label = delegate.print(waybillNo);
        label.stamp("已复核");
        audit("done " + waybillNo);
        return label;
    }

    private void audit(String line) {
        System.out.println(line);
    }
}
```

客户端必须通过代理拿到接口，否则附加逻辑不会执行：

```java
LabelPrinter printer = new AuditedPrinter();
printer.print("WB-2048");
```

这种手写的、编译期就存在的代理叫静态代理。面向切面编程里的动态代理是同一思想：容器在运行时生成一个实现了目标接口的类，把标在方法上的前置、后置、环绕逻辑织进去。业务类不用自己写 `AuditedPrinter`。Spring 对 `@Before`、`@After`、`@Around` 的处理就是这条路径。代理相对真实对象多出来的工作，应该是横切关注点，不要把核心业务搬进代理，否则真实类就空了。

## 行为型模式

### 模板模式

父类规定步骤和顺序，把会变化的步骤留给子类。子类不改算法骨架，只重写允许变化的那几步。

好处是重复流程只写一次。缺点是每一种差异都要一个子类，类的数量会上去。

要点可以收成两条：

1. 用抽象类充当模板。里面放基本方法、模板方法和钩子。模板方法负责把它们按顺序串起来，并标成 `final`，避免子类改顺序。留给子类的方法用 `protected`。
2. 子类只覆盖抽象步骤，必要时覆盖钩子，把细节补全。

适合的情况：

- 多个子类有同一段逻辑，只是其中几步不同，可以把相同部分上收到父类
- 主流程固定，细节随场景变化，例如导出报表都是“取数、排版、写出”，写出格式不同

#### 架构方法介绍

模板方法允许子类在不改算法结构的前提下重定义某些步骤。基本方法又分成三类，它们和模板方法一起出现在抽象类里：

1. 抽象方法：父类只声明，子类必须实现。
2. 具体方法：父类已经给出实现，子类通常不覆盖。这是大家都能用的公共步骤。
3. 钩子方法：父类给出一个默认实现，常常是空方法。子类可以覆盖，也可以不覆盖。它和具体方法在语法上一样，差别在意图：钩子是预留的扩展点，相当于一份默认配置。它和抽象方法的差别是，抽象方法不能不写，钩子可以保持沉默。
4. 模板方法：一个方法里写出整个流程的骨架，按顺序调用上面三类方法。

用“生成日结单”做骨架：

```java
public abstract class DailySheet {
    public final void render() {
        open();
        fillBody();
        close();
    }

    protected void open() {
        System.out.println("写入表头");
    }

    protected abstract void fillBody();

    protected void close() {
    }
}
```

`render` 是模板方法，顺序固定。`open` 是具体方法，子类一般沿用。`fillBody` 是抽象方法，文本版和表格版各自实现。`close` 是钩子，默认什么都不做；需要在末尾追加签名的子类再覆盖它。
