# Java两种动态代理：JDK动态代理和CGLIB动态代理

## 代理模式

代理是二十三种常见设计模式之一。对象 A 握着对象 B，对外表现得和 B 一样。为了让调用方只依赖协议，B 实现某个接口，A 也实现它。真正干活的是 B，A 在调用 B 的前后插入额外步骤，例如计时、鉴权、记日志。面向切面的织入，运行时多半就是这样包出来的。

代理还用来挡住不稳定的外部协议。对方改了字段，只改代理这一处，业务类仍调用自己稳定的方法。凡是走出进程的调用，都可以先过这样一层，避免外部模型渗进领域代码。

如果 A 在源码里写死握着哪一个 B，那是静态代理。B 要到运行时才知道是谁，就要在运行时生成 A，这是动态代理。常见两条路：标准库自带的基于接口的代理，以及靠生成子类的字节码代理。

## JDK动态代理

这条路只依赖运行时，不必引第三方。先定义教练能做的两件事。

```java
public interface Coach {
    String train(String drill);
    String rest(int minutes);
}
```

再给一个具体的人。

```java
public class HanCoach implements Coach {
    @Override
    public String train(String drill) {
        System.out.println("开始训练:" + drill);
        return "完成-" + drill;
    }

    @Override
    public String rest(int minutes) {
        System.out.println("休息分钟:" + minutes);
        return "恢复-" + minutes;
    }
}
```

增强逻辑写在调用处理器里，它不是代理类本身，因为它没有实现 `Coach`，调用方不能拿它当教练用。它是包裹层：真正的代理对象由标准库生成，生成时把这个处理器装进去。

```java
public class CoachHandler implements java.lang.reflect.InvocationHandler {
    private Object target;

    public void setTarget(Object target) {
        this.target = target;
    }

    @Override
    public Object invoke(Object proxy, java.lang.reflect.Method method, Object[] args) throws Throwable {
        System.out.println("入场登记 " + method.getName());
        Object result = method.invoke(target, args);
        System.out.println("离场核对");
        return result;
    }

    public Object newProxy() {
        return java.lang.reflect.Proxy.newProxyInstance(
                target.getClass().getClassLoader(),
                target.getClass().getInterfaces(),
                this);
    }
}
```

`newProxyInstance` 要三样东西：和目标相同的类加载器、目标实现了哪些接口、出问题时把调用交给谁。返回值才是代理，类型是运行时生成的 `$Proxy` 加编号，它实现了那些接口，并继承 `Proxy`。

使用顺序：

- 先创建被代理的教练
- 再创建处理器，把教练塞进去
- 生成代理后，按接口类型使用，不要按具体类强转

```java
public class ProxyLab {
    public static void main(String[] args) {
        Coach coach = new HanCoach();
        CoachHandler handler = new CoachHandler();
        handler.setTarget(coach);
        Coach proxy = (Coach) handler.newProxy();
        System.out.println(proxy.train("折返跑"));
        System.out.println(proxy.getClass().getName());
    }
}
```

生成过程可以这样理解。标准库读到接口上有哪些方法，就声明一个新类把这些方法都实现出来。方法体不能直接去调教练，否则前后的登记无处安放，也等于绕了一圈又调用原来的对象。所以方法体统一转给处理器的 `invoke`。处理器里再反射调用真正的教练，登记代码写在反射前后。

于是有两层：生成类代理处理器，处理器再代理业务对象。人们仍把它叫作对象的代理，因为代理的是某一个已经存在的实例，而不是这个类的所有未来实例。

生成类里的每个业务方法都长得差不多，下面用说明性的片段表达结构，不是某一版反编译的逐字结果。

```java
public final class ProxyCoach extends Proxy implements Coach {
    private static Method trainMethod;

    public String train(String drill) {
        try {
            return (String) h.invoke(this, trainMethod, new Object[]{drill});
        } catch (RuntimeException ex) {
            throw ex;
        } catch (Throwable ex) {
            throw new UndeclaredThrowableException(ex);
        }
    }
}
```

`h` 就是创建时传进去的处理器。`train` 和 `trainMethod` 的对应关系在生成类的静态块里建好：按接口和参数类型查出 `Method`，存进静态字段。业务代码看不到这些字段。

`invoke` 的第一个参数是代理对象自己。如果在处理器里又去调用这个参数上的 `train`，会再次进入 `invoke`，立刻递归到栈溢出。要调用的是事先保存的目标，而不是代理。

静态块大致完成这些查找：

```java
static {
    trainMethod = Coach.class.getMethod("train", String.class);
    restMethod = Coach.class.getMethod("rest", int.class);
}
```

接口上的方法，以及 `equals`、`hashCode`、`toString`，都会被这样绑好。代理如果被放进散列集合，相等性默认走处理器；没有特殊处理时，两个代理即使包着同一个目标，也不一定相等。

这条路的限制也很明确：目标至少要实现一个接口。只想代理一个没有接口的具体类，标准库生成不出来。另外，对类本身的 `final` 方法没有意义，因为调用方如果拿具体类型而不是接口，根本不会走到代理。

## CGLIB动态代理

同一个行为也可以不靠接口。子类继承父类后，可以重写公开方法，并在重写里加上增强。里氏替换允许在需要父类的地方放入子类，所以调用方甚至可以按具体类型接收代理。

这就是字节码库生成子类的思路。它不要求目标实现接口。

```java
public class SubclassHook implements net.sf.cglib.proxy.MethodInterceptor {
    public Object spawn(Class<?> type) {
        net.sf.cglib.proxy.Enhancer maker = new net.sf.cglib.proxy.Enhancer();
        maker.setSuperclass(type);
        maker.setCallback(this);
        Object created = maker.create();
        return created;
    }

    @Override
    public Object intercept(Object self, java.lang.reflect.Method method, Object[] args,
                            net.sf.cglib.proxy.MethodProxy fast)
            throws Throwable {
        System.out.println("入场登记 " + method.getName());
        Object result = fast.invokeSuper(self, args);
        System.out.println("离场核对");
        return result;
    }
}
```

`spawn` 只需要一个类对象。生成出来的是目标的子类，类名里通常带 `EnhancerByCGLIB`。没有接口也可以代理，但目标类不能是 `final`，要拦截的方法也不能是 `final` 或 `private`，因为子类重写不了它们。

生成类里大致有三组方法。一组是重写后的公开方法，外部调用会进这里。一组是给父类方法留的快速入口，方法名带特殊标记，方法体只是 `super` 调用。还有拦截器上的统一入口。重写方法不直接写业务，而是把自身、原方法、参数和快速入口一起交给 `intercept`。

这里有个容易写错的选择：四个参数里，为什么用快速入口，而不用反射方法？

```java
return fast.invokeSuper(self, args);
```

`method.invoke(self, args)` 会再次命中子类已经重写的方法，于是又进 `intercept`，同样是死循环。父类方法被子类挡住了，反射看不见“请调用 super”这个语义。`invokeSuper` 走的是生成类里那个直接调用父实现的入口。

这个入口按编号找到方法，而不是每次反射查找，所以热路径上更快。编号表就是常说的 fast class：为每个可拦截方法分配下标，调用时用下标跳进去。

和标准库那条路对比：两边都有一个第三者来分发。标准库把调用分给另一个对象；子类代理最后分给自己的父类实现。标准库代理的是对象，子类代理更像在代理这个类，因为连接口都不要，新的实例从子类构造出来，原对象甚至可以不先存在。

## JDK动态代理和CGLIB的区别

标准库只能代理接口。类没有实现接口时，它没有可生成的方法表。

子类方案对着类生成子类，重写方法来插入逻辑。`final` 类和 `final` 方法都无法继承或覆盖，构造方法也不会被拦截。

子类方案底层用 ASM 写字节码。在较早的运行时上，它比反射调用快。后来标准库的代理也被持续优化：调用次数不多时，两者差距很小；调用非常密时才值得用基准重新测。不要背“某一个永远更快”，版本和调用形态都会改结论。

容器在二者之间的默认选择：

- 目标实现了接口，默认用标准库代理
- 目标没有接口，改用子类代理
- 即使有接口，也可以通过配置强制走子类代理，这样按具体类注入时也能命中切面

只能走子类的另一个原因是单继承。标准库生成的类已经继承了 `Proxy`，就不能再继承业务类。业务类如果自己还有父类，标准库代理并不妨碍它，因为代理的是接口，不是去继承业务类。一旦调用方要的类型是具体类而不是接口，就只能生成业务类的子类。

## 总结

- 目标有接口时，默认按接口生成代理来做切面
- 有接口也可以强制生成子类，让具体类类型同样被增强
- 没有接口时只能生成子类，容器会自己切换

标准库这条路的条件：

- 实现调用处理器
- 用 `Proxy.newProxyInstance` 得到代理
- 被代理的类型至少有一个接口

子类这条路依赖字节码库。它不要求接口，而是指定一个类，生成其非 final 子类。代理对象是子类的新实例，增强写在拦截器里，父类逻辑通过 `invokeSuper` 执行。

选型时先看“调用方拿的是接口还是具体类”，再看目标方法是不是 final。两条路都不要在拦截器里回调代理自身的同一方法。需要把代理放进集合时，明确写出相等性，不要假设它等于目标对象。
