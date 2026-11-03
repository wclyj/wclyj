# Spring源码

## Spring源码方面的知识

面试里和源码相关的问题，通常围着下面几件转。

- 一个 Bean 从定义到可用，中间经过哪些阶段
- 工厂、容器、上下文三者各管什么
- 后置处理器能在什么时机插手
- 持久层、Web 这些外围框架怎样嵌进容器
- `BeanFactory` 和 `FactoryBean` 名字相近，职责完全不同

## 谈谈你对Spring的理解

控制反转和面向切面只是框架里最常被提到的两块。容器同时还管事件、资源加载、国际化文案、校验、数据绑定、类型转换，以及用表达式在运行时读取对象图。这些能力都挂在同一套上下文上，不是各自为政的库。

可以把上下文理解成组装现场：组件先被登记，再按依赖关系装配，最后才对外提供。只背两个缩写，回答会停在概念层。

## Spring上下文

从类型上看，上下文就是 `ApplicationContext` 这一支。它比单纯的 Bean 工厂多了事件发布、资源解析和环境抽象。

从启动过程看，上下文刚建好时，一批内部组件要先各就各位：环境、后置处理器、类型转换服务都准备好，容器才算进入能正常产生产品的状态。这个能协同工作的状态，就是人们口头说的 Spring 环境。

## Spring初始化

只观察启动时，引入上下文这个依赖就够了，不必把 Web 启动器一起拉进来。版本号按项目统一管理，下面只是示意。

```xml
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-context</artifactId>
    <version>5.3.31</version>
</dependency>
```

准备三个类型，才能看见“配置 → 扫描 → 取出”。

配置类标出要扫的包：

```java
import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Configuration;

@Configuration
@ComponentScan("demo.catalog")
public class CatalogConfig {
}
```

被扫到的组件：

```java
import org.springframework.stereotype.Component;

@Component
public class PriceTag {
}
```

启动类用注解上下文，按类型取出组件并打印。默认是单例，两次取出应当是同一个实例。

```java
import org.springframework.context.annotation.AnnotationConfigApplicationContext;

public class BootLab {
    public static void main(String[] args) {
        try (AnnotationConfigApplicationContext ctx =
                     new AnnotationConfigApplicationContext(CatalogConfig.class)) {
            System.out.println(ctx.getBean(PriceTag.class));
        }
    }
}
```

开发工具里点进去的类，有时是反编译结果，和仓库里的源码在局部变量、合成方法上会有差别。要单步调试生命周期，应让构建工具下载对应版本的源码包，而不是改反编译出来的文件。

## SpringBean的生命周期

容器里的对象不是业务代码执行 `new` 就结束了。至少还有两段：先发现类并形成定义，再按定义实例化、装配、初始化。

启动入口常见两种。现在更多用注解上下文；XML 上下文仍然能看见同一套刷新流程。

```java
AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext(CatalogConfig.class);
```

这个构造会先走到无参构造。子类构造执行前，父类 `GenericApplicationContext` 的构造已经创建一个默认的 `BeanFactory`。工厂的本职是按名字或类型把对象产出来，它还不负责事件和资源。

构造的末尾调用 `refresh`。这是容器启动最集中的一段，里面按固定顺序做了十多步。和“类如何被扫进来”关系最大的，是执行 Bean 工厂后置处理器的那一步。

```text
invokeBeanFactoryPostProcessors
    处理配置类上的扫描与导入
    例如组件扫描、映射扫描，以及把外部定义登记进工厂
```

刷新的后半段才会真正把单例预实例化。所以“定义已经在表里”和“对象已经能注入”不是同一时刻。

### 普通类的实例化

普通类先由编译器变成类文件。执行到 `new` 时，虚拟机把类加载进来，在堆上分配对象，类的结构信息放在元空间，局部变量放在栈帧。没有容器参与，也不会自动注入协作者。

### Spring Bean实例化过程

容器把这件事拆开。

- 类加载器先把相关类装进虚拟机，此时还没有业务对象
- 扫描或解析配置之后，为每个组件建一份定义。定义描述名字、类型、作用域、是否延迟，而不是对象本身

```java
RootBeanDefinition definition = new RootBeanDefinition();
definition.setBeanClass(PriceTag.class);
definition.setScope("prototype");
definition.setLazyInit(true);
```

- 定义放进一张以名字为键的表。扫到几个组件，表里就有几条。这张表通常叫 Bean 定义表
- 单例在刷新后段按定义创建，放进单例池。原型不会进池，每次获取都重新创建。上面如果把作用域写成原型，池里就不应缓存它

加载过程对外留了扩展点。定义还没变成对象时，工厂后置处理器可以改定义，例如补上属性或注册更多名字。对象已经 `new` 出来、依赖也注入之后，Bean 后置处理器可以包一层代理再放回容器。很多外围框架并不改核心源码，只实现这些处理器。映射框架在扫描阶段登记接口，就是走工厂后置处理器这条缝。

```java
import org.springframework.beans.BeansException;
import org.springframework.beans.factory.config.BeanFactoryPostProcessor;
import org.springframework.beans.factory.config.ConfigurableListableBeanFactory;

public class CatalogPostProcessor implements BeanFactoryPostProcessor {
    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory)
            throws BeansException {
        if (beanFactory.containsBeanDefinition("priceTag")) {
            beanFactory.getBeanDefinition("priceTag").setLazyInit(false);
        }
    }
}
```

处理器自己也要能被容器发现，一般同样标成组件，或在配置类上手动声明。它执行的时候目标对象通常还不存在，所以这里只能改定义，不能调用那个对象的业务方法。

最后把两个容易混的名字分开：`BeanFactory` 是生产对象的工厂；`FactoryBean` 是一种特殊产品，容器向它要东西时，拿到的往往是它 `getObject` 的返回值，而不是它自己。要工厂本身，名字前要加 `&`。
