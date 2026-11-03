# Volatile 的应用

## 单例模式 DCL 代码

先看只有一个线程时的懒汉写法。

```java
/**
 * 单线程下够用的懒汉单例。多线程下会创建出多份。
 */
public class ConfigHub {

    private static ConfigHub hub = null;

    private ConfigHub() {
        System.out.println(Thread.currentThread().getName() + " 执行了构造");
    }

    public static ConfigHub getHub() {
        if (hub == null) {
            hub = new ConfigHub();
        }
        return hub;
    }

    public static void main(String[] args) {
        // 比较的是引用是否指向同一块内存
        System.out.println(ConfigHub.getHub() == ConfigHub.getHub());
        System.out.println(ConfigHub.getHub() == ConfigHub.getHub());
    }
}
```

单线程反复调用，构造只打印一次，引用比较都是 true。

换成 8 个线程同时第一次调用，构造可能打印多次，说明堆上出现了多份对象：

```java
public static void main(String[] args) {
    for (int i = 0; i < 8; i++) {
        new Thread(() -> {
            ConfigHub.getHub();
        }, "loader-" + i).start();
    }
}
```

可能看到：

```text
loader-1 执行了构造
loader-4 执行了构造
loader-6 执行了构造
```

多个线程都通过了 `hub == null`，又都执行了 `new`。单例在并发下不成立。

### 解决方法 1

把整个获取方法标成 `synchronized`。

```java
public static synchronized ConfigHub getHub() {
    if (hub == null) {
        hub = new ConfigHub();
    }
    return hub;
}
```

构造只会打印一次。代价是初始化完成之后，每一次读取仍然要串行进入这个方法。锁太粗，并发读被拖成了互斥读。

### 解决方法 2

改成双重检查（double-checked locking）。外层先无锁判断，只有第一次才进同步块；进了同步块再判断一次，避免两个线程都看到 null 后排队各自 new 一次。

```java
public static ConfigHub getHub() {
    if (hub == null) {
        synchronized (ConfigHub.class) {
            if (hub == null) {
                hub = new ConfigHub();
            }
        }
    }
    return hub;
}
```

构造次数变成一次，看起来单例成立。这里仍有一个洞：对象引用的发布可能早于对象初始化完成。

`hub = new ConfigHub()` 在字节码和 CPU 上大致是三步：

- `memory = allocate();`  // 1. 分配一块未初始化的内存
- `init(memory);`         // 2. 执行构造，把字段填好
- `hub = memory;`         // 3. 把地址写到静态字段，此时 hub 不再是 null

第 2 步和第 3 步没有数据依赖。单线程下先发布引用再填字段，最终结果仍然正确，所以这种重排是允许的：

- `memory = allocate();`  // 1. 分配
- `hub = memory;`         // 3. 引用已经不是 null，构造还没做完
- `init(memory);`         // 2. 稍后才初始化

另一个线程走到外层 `if (hub == null)`，看见不是 null，就直接返回。它拿到的对象可能还没有跑完构造，字段仍是默认值。

重排只承诺单线程语义不变，不承诺别的线程能看到一致的中间状态。

所以 DCL 必须把引用声明成 `volatile`。volatile 写不允许把前面的普通写（构造里的字段赋值）挪到发布引用之后。别的线程一旦看见 `hub != null`，就一定看见构造已经完成。

```java
private static volatile ConfigHub hub = null;
```

### 最终代码

```java
/**
 * 双重检查 + volatile，避免未初始化对象被发布出去。
 */
public class ConfigHub {

    private static volatile ConfigHub hub = null;

    private ConfigHub() {
        System.out.println(Thread.currentThread().getName() + " 执行了构造");
    }

    public static ConfigHub getHub() {
        if (hub == null) {
            // 外层看到 null 才抢锁；抢到锁时别人可能已经构造完
            synchronized (ConfigHub.class) {
                // 没有 volatile 时，这里发布的引用可能先于构造完成
                if (hub == null) {
                    hub = new ConfigHub();
                }
            }
        }
        return hub;
    }

    public static void main(String[] args) {
        for (int i = 0; i < 8; i++) {
            new Thread(() -> {
                ConfigHub.getHub();
            }, "loader-" + i).start();
        }
    }
}
```

8 个线程并发调用时，构造日志只出现一行。之后的调用走外层判断，不再进锁。
