# Bean 的生命周期

## 前言

脱离容器时，对象多半来自 `new`。另外三条路也常见：反射（`Class.getDeclaredConstructor().newInstance()` 或 `Constructor.newInstance`）、`clone()`，以及 `ObjectInputStream` 把字节流还原成对象。没有引用之后，这些实例就交给垃圾回收器处理。

交给 Spring 的 IoC 容器之后，创建和销毁之间会插入一串回调。下面按发生顺序把这条链写清楚，不依赖示意图。

## Bean 的生命周期

一次典型的单例创建，容器大致做这些事：

1. 启动时找出要托管的类，调用构造器，得到原始实例。
2. 按定义把依赖和普通属性填进去，setter 或字段注入发生在这一步。
3. 若实现了 `BeanNameAware`，把注册名传给 `setBeanName`。
4. 若实现了 `BeanFactoryAware`，把当前 `BeanFactory` 传给 `setBeanFactory`。
5. 若实现了 `ApplicationContextAware`，再把所在的 `ApplicationContext` 交给 `setApplicationContext`。Aware 系列只负责“告诉 Bean 自己活在哪个容器里”，不要在这里做重业务。
6. 容器中每个 `BeanPostProcessor` 都会收到 `postProcessBeforeInitialization`。注意：被创建的业务 Bean 自己通常不实现这个接口，接口是给加工器实现的。
7. 若实现了 `InitializingBean`，调用 `afterPropertiesSet`。`@Bean(initMethod = "...")` 或 XML 的 `init-method` 指向的方法紧随其后。
8. 同一个加工器再走 `postProcessAfterInitialization`。AOP 代理常常在这里把原始对象换成代理。
9. 到这里 Bean 可以对外提供服务，并一直留在上下文里，直到上下文关闭。
10. 关闭时，若实现了 `DisposableBean`，调用 `destroy`；声明过 `destroyMethod` 的话，对应方法也会执行。

## 接口方法的分类

整条链上的回调可以收成四组。

### Bean 自身的方法

构造器、普通 setter，以及你在定义里点名的初始化方法和销毁方法，都属于 Bean 自己的代码。容器只是按约定去调用，不要求额外接口。

### Bean 级生命周期的方法

`BeanNameAware`、`BeanFactoryAware`、`ApplicationContextAware`、`InitializingBean`、`DisposableBean` 写在业务类上。每个实例各自实现，容器按类型逐个回调。

### 容器级生命周期的方法

`BeanPostProcessor` 和它的子接口 `InstantiationAwareBeanPostProcessor` 注册一次，就能碰容器里的许多 Bean。习惯把这类实现叫做后处理器。它们看得到创建前、填充属性、初始化前后几个缝隙。

### 工程后处理器接口方法

`BeanFactoryPostProcessor` 更早：Bean 还没实例化，只改 Bean 定义。`ConfigurationClassPostProcessor`、`CustomAutowireConfigurer` 以及和 AspectJ 织入相关的工厂后处理器都属于这一层。上下文装载完定义后会立刻调用它们，所以它们影响的是“接下来怎么造对象”，而不是某一个已经造好的对象。

## DEMO 举例

下面用一个货运单 Bean 把顺序跑出来。类上同时实现四个 Bean 级接口，并用 `@Bean` 的 `initMethod` / `destroyMethod` 代替 XML 属性。

```java
public class Cargo implements BeanNameAware, BeanFactoryAware,
        DisposableBean, InitializingBean {

    private String sku;
    private String city;
    private int weight;
    private String registeredName;
    private BeanFactory factory;

    public Cargo() {
        System.out.println("1 构造 Cargo");
    }

    public void setSku(String sku) {
        System.out.println("2 写入 sku");
        this.sku = sku;
    }

    public void setCity(String city) {
        System.out.println("2 写入 city");
        this.city = city;
    }

    public void setWeight(int weight) {
        System.out.println("2 写入 weight");
        this.weight = weight;
    }

    @Override
    public void setBeanName(String name) {
        System.out.println("3 BeanNameAware 收到名字 " + name);
        this.registeredName = name;
    }

    @Override
    public void setBeanFactory(BeanFactory factory)
            throws BeansException
    {
        System.out.println("4 BeanFactoryAware 收到工厂");
        this.factory = factory;
    }

    @Override
    public void afterPropertiesSet() {
        System.out.println("7 InitializingBean.afterPropertiesSet");
    }

    public void onReady() {
        System.out.println("7 initMethod onReady");
    }

    @Override
    public void destroy() {
        System.out.println("10 DisposableBean.destroy");
    }

    public void onShutdown() {
        System.out.println("10 destroyMethod onShutdown");
    }
}
```

加工器是单独的组件，两个方法的入参都是待加工对象和 Bean 名，返回值决定后续还用不用原对象。返回 `null` 会让容器丢掉这个实例，演示时请把原对象交回去。

```java
@Component
public class TracePostProcessor implements BeanPostProcessor {

    @Override
    public Object postProcessBeforeInitialization(Object target, String name) {
        if (target instanceof Cargo) {
            System.out.println("6 beforeInitialization " + name);
        }
        return target;
    }

    @Override
    public Object postProcessAfterInitialization(Object target, String name) {
        if (target instanceof Cargo) {
            System.out.println("8 afterInitialization " + name);
        }
        return target;
    }
}
```

`InstantiationAwareBeanPostProcessor` 能再往前插一脚，发生在构造器之前，也能插手属性填充。继承适配器只覆盖关心的方法；`postProcessBeforeInstantiation` 返回非空对象时，容器会跳过真正的构造。

```java
@Component
public class TraceInstantiationProcessor
        extends InstantiationAwareBeanPostProcessorAdapter
{

    @Override
    public Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) {
        if (Cargo.class.isAssignableFrom(beanClass)) {
            System.out.println("0 尚未构造 " + beanName);
        }
        return null;
    }

    @Override
    public PropertyValues postProcessProperties(PropertyValues pvs, Object bean, String beanName) {
        if (bean instanceof Cargo) {
            System.out.println("2 属性即将注入 " + beanName);
        }
        return pvs;
    }
}
```

注册时把初始化、销毁方法名写在 `@Bean` 上：

```java
@Configuration
public class CargoConfig {
    @Bean(initMethod = "onReady", destroyMethod = "onShutdown")
    public Cargo cargo() {
        Cargo cargo = new Cargo();
        cargo.setSku("SKU-18");
        cargo.setCity("杭州");
        cargo.setWeight(3);
        return cargo;
    }
}
```

用 `AnnotationConfigApplicationContext` 启动再 `close()`，控制台会按上面编号依次出现。若 `beforeInstantiation` 返回了替身对象，后面的构造和注入都不会发生，这是最容易把演示跑丢的一点。

## 参考

对照 Spring Framework 文档里关于 Bean 生命周期、`BeanPostProcessor` 与 `BeanFactoryPostProcessor` 的章节即可，不必依赖某篇转载。
