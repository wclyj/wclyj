# 双亲委派：先问上级，以及容器为什么不这么做

## 委派在解决什么

某个加载器收到“按这个全限定名把类读进来”的请求时，默认不会立刻自己读字节，而是先交给父加载器。父加载器同样再交给它的父加载器，直到引导加载器。只有上级明确说自己的路径里没有这个类，下级才自己找。

这样做有一个直接的结果：`java.lang.Object` 无论从业务代码、容器代码还是框架代码发起加载，最后都由引导加载器读核心库里的那一份。不同加载器不会各自造出一个 `Object`，否则类型系统对不上，连 `equals` 的身份都会乱。

## 三条工作步骤

- 收到请求，先交给父加载器，自己不抢先读。
- 父加载器如果还有上级，就继续上交，直到引导加载器。
- 上级能完成就返回那个 `Class`；上级的路径里根本没有这个名字，下级才尝试自己的路径。

所以默认方向是“往上请求、往下执行”。找到一份就停，不会再让更下级覆盖。

## 四层加载器

- 引导加载器：读 Java 核心类。它不是 `java.lang.ClassLoader` 的子类，由虚拟机用本地代码实现，在 Java 里拿父加载器会得到 `null`。
- 扩展加载器：JDK 8 的路径在 `jre/lib/ext`。对它调用 `getParent()` 也是 `null`，因为父级是引导加载器，Java 层看不见。JDK 9 以后这一层改名为平台加载器，委派方向没有变。
- 系统（应用）加载器：读 `-classpath` 或 `CLASSPATH` 指定的目录和 jar，业务代码默认落在这里。
- 用户自定义加载器：继承 `ClassLoader`，自己决定字节从哪来。

## JDBC 为什么会反向要一次

数据库驱动的 jar 实现的是一套 SPI 接口，接口本身在核心库或标准扩展里，实现类在业务 classpath。加载接口时仍按委派走到引导加载器，保证全进程只有一份接口类型。接着要找实现类时，引导加载器看不见业务 jar，于是通过线程上下文加载器把请求交回应用加载器，去读 `jdbc` 驱动那一份实现。这不是把委派废掉，而是在“接口在上、实现在下”时补了一次自上而下的查找。

## 沙箱：自己写一个 java.lang.String 也不会被用上

如果在工程里放一个 `java.lang.String`，启动主类时引导加载器会先命中核心库里的 `String`。你写的那个类要么根本不被选中，要么在定义 `java.lang` 开头的类时直接被拒绝。报错有时看起来像“找不到 main”，是因为真正跑起来的是核心库里的 `String`，它并没有你准备的入口。核心类因此不能被应用随意换掉，这就是沙箱要保的边界。

## 委派带来的两个好处

- 同一个名字的核心类只加载一次，避免各加载器各持一份、类型却互不兼容。
- 核心 API 不能被同名类替换。自定义 `java.lang.String` 不会顶掉引导加载器的那份；有的实现还会直接禁止定义 `java.lang` 前缀的类。

## 容器为什么要打破它

打破委派的场景包括 JDBC 的实现查找、JNDI，以及 Web 容器。下面用 Tomcat 说明，因为它要同时满足几件互相打架的事。

一个容器里往往部署多个应用：

- 两个应用可能依赖同一套第三方库的不同版本，类必须按应用隔开。
- 相同版本的公共库又希望只加载一份，不要每个应用复制一份进元空间。
- 容器自己的类不能和应用程序混在一个加载器里，否则应用能摸到服务器内部类型，服务器也会被应用里的同名类干扰。
- JSP 改完要能热替换。JSP 会编译成 class，类名常常不变，默认加载器看见名字已存在就不会再读新字节。

### 只用默认的应用加载器会怎样

应用加载器只认全限定名，一份名字只保留一份 `Class`。两个 war 里的 `org.springframework.core.SpringVersion` 若版本不同，后加载的那个要么失败，要么把先加载的顶掉，做不到隔离。容器类和应用类若走同一个加载器，也隔离不了。

JSP 同样卡住：文件改了，类名没改，方法区里已经有这个 `Class`，默认不会重读。做法是让每个 JSP 独占一个加载器，文件变化时丢掉整个加载器，再新建一个去读新的字节。加载器可以被回收，里面的类才跟着失效。

### 容器侧的几级加载器

- Common：最底层的公共加载器，它读到的类，容器本身和各个 Web 应用都能用。
- Catalina：容器私有。应用看不见这些类，避免应用依赖服务器内部实现。
- Shared：只给各个 Web 应用共享，容器自己看不见，用来放大家一致的业务公共库。
- Webapp：每个应用一个实例，只看见本应用的 `/WEB-INF`。
- Jasper：每个 JSP 一个实例，范围小到“这一份编译结果”，目的就是随时被换掉。

关系可以记成：

- Common 能读到的类，Catalina 也能用，公共库只存一份。
- Catalina 和 Shared 各自读到的类互相不可见。
- Webapp 可以使用 Shared 已加载的类，但不同 Webapp 实例之间隔离，它们是平级，不是父子。
- Jasper 只负责那一个 JSP 的 class。容器发现页面变了，就丢掉当前 Jasper 实例，新建一个，完成热替换。

目录上，老版本把这几级对应到 `/common`、`/server`、`/shared` 和每个应用的 `WEB-INF`。`conf/catalina.properties` 里如果没有配置 `server.loader` 和 `shared.loader`，就不会真正创建 Catalina 和 Shared，两者的职责合并进 Common。Tomcat 6 之后把 common、shared、server 收成一个 `lib` 目录，所以新装的服务器上看不到那三个旧目录。

### 应用类的查找顺序

Webapp 加载器打破委派的地方，在于它不总是先问父亲。简化后的顺序是：

1. 先保证引导加载器负责的核心类不会被应用替换。
2. 再查本应用的 `/WEB-INF/classes`。
3. 再查本应用 `/WEB-INF/lib` 里的 jar。
4. 然后才是系统加载器路径。
5. 最后才是 Common 那一层的公共目录。

落实到一次 `loadClass`，步骤通常是：

- 先查本地缓存（实现里常叫资源项表）。命中就返回，避免重复定义。
- 再让系统加载器尝试那些必须由上级装载的基础类，防止 Web 应用用同名类把核心类型盖掉。命中就返回。
- 前两步都没有，就由当前 Webapp 加载器读自己的 classes 和 lib。
- 自己也没有，才委托父级 Common。

因为第 3 步发生在委托父级之前，两个 war 可以各自带着不同版本的 Spring，全限定名相同也互不覆盖。公共库若希望共享，应放进 Common 或 Shared，而不是打进每个 war。

## 哪一层其实没打破

容器自己的 Common、Catalina、Shared 仍然按委派工作。它们在整个进程里只有一套，重复的名字以这一套为准，没有“两个容器版本并存”的需求。

打破发生在每个 war 自动得到的那个 Webapp 加载器上。假如下级仍先问父亲，父亲一旦从公共目录或系统路径把 `org.springframework` 装进来，war 里的那一份就永远没机会定义。Spring 4 的应用和 Spring 5 的应用就无法放在同一台容器里。Webapp 加载器因此先读 war 内部的 class，只有 war 里没有的类型（例如还没装过的公共类、JDK 类）才继续交给上级。

## 在自定义加载器里把委派改掉

设磁盘上有一份 `lab.runtime.badge.Badge` 的字节码，应用的编译输出里也有同名类。按默认委派，请求会先到应用加载器，应用加载器能找到，自定义加载器的 `findClass` 根本不会被调用。

委派写在 `ClassLoader.loadClass` 里。不想用这套模型，就覆盖 `loadClass`，而不是只覆盖 `findClass`。只覆盖 `findClass` 时，父级已经成功的话子级代码不会走到。

下面的加载器约定：名字以 `lab.runtime.badge` 开头的，自己读字节并 `defineClass`；其余名字仍交给父加载器，这样 `java.lang.String`、集合类不会被我们从随便一个目录定义出来。

```java
public class BadgeLoader extends ClassLoader {
    private final Path root;

    public BadgeLoader(Path root) {
        this.root = root;
    }

    @Override
    protected Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException {
        Object gate = getClassLoadingLock(name);
        synchronized (gate) {
            Class<?> loaded = findLoadedClass(name);
            if (loaded == null) {
                if (name.startsWith("lab.runtime.badge.")) {
                    loaded = findClass(name);
                } else {
                    loaded = getParent().loadClass(name);
                }
            }
            if (resolve) {
                resolveClass(loaded);
            }
            return loaded;
        }
    }

    @Override
    protected Class<?> findClass(String binaryName) throws ClassNotFoundException {
        Path file = root.resolve(binaryName.replace('.', '/') + ".class");
        try {
            byte[] bytes = Files.readAllBytes(file);
            return defineClass(binaryName, bytes, 0, bytes.length);
        } catch (IOException ex) {
            throw new ClassNotFoundException(binaryName, ex);
        }
    }
}
```

调用时传入字节码所在目录，加载 `lab.runtime.badge.Badge`，再反射执行它的 `show` 方法。打印 `getClassLoader()` 的类名，应当是 `BadgeLoader`，而不是应用加载器。

```java
    public static void main(String[] args) throws ReflectiveOperationException {
    BadgeLoader loader = new BadgeLoader(Path.of("/opt/badge-classes"));
    Class<?> type = loader.loadClass("lab.runtime.badge.Badge");
    Object badge = type.getDeclaredConstructor().newInstance();
    type.getDeclaredMethod("show").invoke(badge);
    System.out.println(type.getClassLoader().getClass().getName());
}
```

注意两件事。一是前缀判断要严，系统类仍走父加载器，否则会把 JDK 类从错误的目录定义出来，沙箱等于被自己拆掉。二是同一个加载器对同一个名字只能 `defineClass` 一次，所以 `findLoadedClass` 要放在最前；热替换的做法是丢掉整个加载器，而不是对旧加载器再次定义。
