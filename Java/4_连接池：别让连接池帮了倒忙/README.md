# 连接池帮倒忙的三种姿势

线程池管的是工人，连接池管的是已经握过手的通道。业务里最常见的是数据库连接池、Redis 连接池和 HTTP 连接池。三者的坑很像：先看不清客户端是不是池，再把池当成一次性对象，最后在参数上拍一个“越大越稳”的数。

## 池在外面长什么样

调用方看见的通常是两个动作：借一条连接、用完还回去。能调的参数里，最小空闲和最大连接数最常被改。池内部还要做五件不一定写在业务代码里的事：建连、心跳、按借还计数、回收闲置连接、借出前做一次可用性探测。可以把结构想成左右两块：左边是你的业务代码，右边是池；池再对着真正的服务端维持若干条 TCP。

## 先判断 SDK 属于哪一种

TCP 面向连接，三次握手有成本；它又是字节流，协议本身不知道哪几个字节是一条完整命令，也不知道有几条线程在共用同一个套接字。SDK 如果只给你一条裸连接，每次都握手会慢，多线程共用还会把字节流写乱。

对外 API 大体三种，用之前要先对上号。命名只能当线索，有的库会反着起名字，所以还是要看文档、看“thread safe / singleton”的讨论，或者一直翻到 `Socket` 字段为止。

- 池和连接分开。存在一个 `XxxPool` 负责借还，借出来的 `XxxConnection` 才拿去发请求。池一般可以多线程一起借，单条连接不行。你的代码就是示意图左边那一块，用完必须自己还。
- 客户端内部带着池。对外一个 `XxxClient`，直接发请求。获取和归还发生在 SDK 里面，客户端对象通常可以多线程复用。整个蓝框都是 SDK。
- 没有池。类型名往往是 `XxxConnection`，一条连接用完就关。它多半不是线程安全的，短连接的吞吐也上不去。要在业务侧自己套一层池，或者换官方提供的池化客户端。

对应的用法：

- 分离型：池做成单例；每次用 try-with-resources 或 finally 借还，还连接是调用方的责任。
- 内置型：整个进程复用那一个 Client，不要在请求路径上反复 new。
- 无池型：要么接受短连接的开销，要么自己做池。多数数据库和中间件的官方客户端都会提供池，先找那一个。

### 用 Jedis 把分离型走一遍

Jedis 是分离型的典型。先写入两条不会变的数据，键用库存编号，避免和别的示例撞车：

```java
@PostConstruct
void seed() {
    try (Jedis jedis = new Jedis("localhost", 6379)) {
        assert "OK".equals(jedis.set("sku:red", "A"));
        assert "OK".equals(jedis.set("sku:blue", "B"));
    }
}
```

错误示范是让两条线程共用同一个 `Jedis`，各读 500 次，并断言红键是 A、蓝键是 B：

```java
Jedis shared = new Jedis("localhost", 6379);
Thread reader = new Thread(() -> {
    for (int i = 0; i < 500; i++) {
        String got = shared.get("sku:red");
        if (!"A".equals(got)) {
            log.warn("sku:red expected A but was {}", got);
            return;
        }
    }
}, "read-red");
reader.start();
Thread blue = new Thread(() -> {
    for (int i = 0; i < 500; i++) {
        String got = shared.get("sku:blue");
        if (!"B".equals(got)) {
            log.warn("sku:blue expected B but was {}", got);
            return;
        }
    }
}, "read-blue");
blue.start();
```

多跑几次会出现三类现象，不一定每次都一样：蓝键读回了 A；抛出流意外结束；或者套接字已经关掉。这些都不是 Redis 把值写错了。

沿着继承链能看见原因。`Jedis` 继承 `BinaryJedis`，后者拿着一个 `Client`；`Client` 再继承到 `Connection`。`Connection` 里是一个 `Socket`，外加一对输出流、输入流。所以一个 `Jedis` 对象就是一条 TCP，不是一组连接。

命令最终会写成 RESP：先写 `*` 和参数个数，再对命令名和每个参数写 `$`、长度和内容。这段写入直接打在那条输出流上，中间没有锁。两条线程同时 `get` 时，字节会互相插进去，对端收到的就不再是合法命令，于是把连接关掉，客户端看到流结束或套接字关闭。即便两次写入碰巧没被拆开，读的顺序也不受保证：线程甲发出读红键、线程乙发出读蓝键，服务端依次回 A 和 B，但乙可能先把 A 从输入流里读走。这就是“蓝键变成 A”。

修复是改用线程安全的 `JedisPool`，池本身静态共享，每条线程只在自己的 try-with-resources 里借用 `Jedis`：

```java
private static final JedisPool POOL = new JedisPool("localhost", 6379);

Thread reader = new Thread(() -> {
    try (Jedis jedis = POOL.getResource()) {
        for (int i = 0; i < 500; i++) {
            if (!"A".equals(jedis.get("sku:red"))) {
                log.warn("sku:red mismatched");
                return;
            }
        }
    }
}, "redis-reader");
reader.start();
```

蓝键那条线程同样从 `POOL` 借自己的 `Jedis`。进程退出前把池关掉，避免后台的空闲检测线程残留：

```java
@PostConstruct
void hook() {
    Thread shutdown = new Thread(POOL::close, "close-redis-pool");
    Runtime.getRuntime().addShutdownHook(shutdown);
}
```

`Jedis.close` 会看自己是不是从池里借的。借来的实例被设置了 `dataSource`，`close` 并不拆 TCP，而是调用池的归还；如果这条连接已经标记损坏，走归还损坏资源的分支，池会把它丢掉。不是从池借的，`close` 才落到 `Connection.disconnect`，刷新输出流并关闭套接字。

`JedisPool.getResource` 在取出对象后把池设回 `dataSource`，这样后面的 `close` 才知道该还。池的父类握着 Apache Commons Pool 的 `GenericObjectPool`，借还、最大总数、最大等待都由这个通用池执行。记法：`JedisPool` 可以共享，`Jedis` 不可以跨线程共享。

## 池本身也要复用

建池往往一次性拉起最小空闲那么多条连接，只为了后面随借随用。若每次业务都新做一个池，你可能只用到 1 条，却付了 N 条握手的成本。

池还有管理机构。多数实现会起一条后台线程，按空闲时间把多余连接回收到最小空闲。有的还要单独做保活。于是“new 一个池”常常等于“多一条甚至多条线程”。池如果不再关闭，这些线程就泄漏了。

用 Apache HttpClient 可以把这件事看清楚。每次请求都新建带连接池的客户端，并打开空闲驱逐，空闲上限 60 秒，然后访问一个只返回固定正文的本地接口：

```java
@GetMapping("http/fresh-client")
public String freshClient() {
    PoolingHttpClientConnectionManager manager = new PoolingHttpClientConnectionManager();
    HttpClientBuilder builder = HttpClients.custom();
    CloseableHttpClient client = builder
            .setConnectionManager(manager)
            .evictIdleConnections(60, TimeUnit.SECONDS)
            .build();
    HttpGet probe = new HttpGet("http://127.0.0.1:8080/probe/ping");
    try (CloseableHttpResponse response = client.execute(probe)) {
        return EntityUtils.toString(response.getEntity(), StandardCharsets.UTF_8);
    } catch (IOException ioError) {
        log.error("probe failed", ioError);
        return null;
    }
}
```

连点几次，线程列表里会出现许多 `Connection evictor`，而且不会自己消失。再用很小的并发去压这个接口，到本服务端口的 TCP 会涨到成百上千，其中只有一条是压测工具连到容器的，其余都是 HttpClient 连回来的。60 秒后这些连接进入 `CLOSE_WAIT`，最后才消失。这说明 `CloseableHttpClient` 属于第二种：里面有池，正确姿势是复用，而不是用完就丢。

复用时把客户端做成静态字段，限制每个路由和总连接数，退出时关闭。也可以注册成 Bean，在销毁回调里 `close`。

```java
private static final CloseableHttpClient HTTP = build();

private static CloseableHttpClient build() {
    HttpClientBuilder builder = HttpClients.custom();
    CloseableHttpClient client = builder
            .setMaxConnPerRoute(1)
            .setMaxConnTotal(1)
            .evictIdleConnections(60, TimeUnit.SECONDS)
            .build();
    Thread hook = new Thread(() -> {
        try {
            client.close();
        } catch (IOException closeError) {
            log.debug("skip close error on shutdown");
        }
    }, "close-http-pool");
    Runtime.getRuntime().addShutdownHook(hook);
    return client;
}

@GetMapping("http/shared-client")
public String sharedClient() {
    HttpGet probe = new HttpGet("http://127.0.0.1:8080/probe/ping");
    try (CloseableHttpResponse response = HTTP.execute(probe)) {
        return EntityUtils.toString(response.getEntity(), StandardCharsets.UTF_8);
    } catch (IOException ioError) {
        log.error("probe failed", ioError);
        return null;
    }
}
```

如果暂时做不到进程级复用，至少在每次用完后把客户端关掉，这样驱逐线程不会越积越多。关掉的代价是连接也不能留给下一次：

```java
@GetMapping("http/close-each-time")
public String closeEachTime() {
    PoolingHttpClientConnectionManager manager = new PoolingHttpClientConnectionManager();
    HttpGet probe = new HttpGet("http://127.0.0.1:8080/probe/ping");
    HttpClientBuilder builder = HttpClients.custom();
    try (CloseableHttpClient client = builder
            .setConnectionManager(manager)
            .evictIdleConnections(60, TimeUnit.SECONDS)
            .build();
         CloseableHttpResponse response = client.execute(probe)) {
        return EntityUtils.toString(response.getEntity(), StandardCharsets.UTF_8);
    } catch (IOException ioError) {
        log.error("probe failed", ioError);
        return null;
    }
}
```

同样压 60 秒，每次新建池的吞吐会明显低于复用池。差距主要来自 TCP，而不是业务计算。上面把最大连接数设成 1，是为了让对比更干净：复用路径上应当一直用同一条连接；新建路径上每次都是新的四元组。

抓包可以验证。走“用完即关”的接口时，每次请求的客户端临时端口都不同，能看见完整的三次握手和四次挥手。走复用接口时，下一次请求的源端口和上一次相同，抓包工具会把它们标成同一个会话里的后续请求。只有空闲超过 60 秒，驱逐线程才会拆掉这条连接，下次再重新握手。

## 最大连接数要按观测来，而不是按感觉

池会暴露最小空闲、最大连接、空闲存活、连接最长寿命等参数。其中最大连接数决定上限，到顶之后新的借用者只能等别人归还。

设得过大并不更安全。客户端要维护更多套接字；服务端如果是一条连接一个线程，几千条连接就是几千个线程，切换开销会先于内存把 CPU 打满。设得过小，借用等待变长，吞吐下降，最后在超时里失败。

下面用数据库池被打满来看怎么确认。开通账户的方法开着事务，中间睡 400 毫秒。事务没结束，底层连接就不会还回池。

```java
@Transactional
public Account openAccount() {
    Account account = new Account();
    account.setName("acct-" + System.currentTimeMillis());
    accountRepository.save(account);
    try {
        TimeUnit.MILLISECONDS.sleep(400);
    } catch (InterruptedException ex) {
        Thread.currentThread().interrupt();
    }
    return account;
}
```

打开 Hikari 的 JMX 注册，方便用 JConsole 看活跃数和等待线程：

```properties
spring.datasource.hikari.register-mbeans = true
```

默认最大连接数是 10。用压测工具打这个接口，活跃连接会很快顶到 10，同时有线程堵在借连接上。大约 30 秒后（Hikari 默认的 `connectionTimeout`）出现：

```text
HikariPool-1 抛出 SQLTransientConnectionException：30 秒内没有可用连接。
```

处理办法是把上限调到观测到的并发之上，并留出余量，而不是一次改成几百：

```properties
spring.datasource.hikari.maximum-pool-size=40
```

调完再看同一组指标：活跃数稳定在压测并发附近，等待线程回到 0，才算这个值合适。真实扩容时可以先临时放到数据库能承受的较大值，记下稳态并发，再按“稳态再留一半余量”收成最终配置。更早的做法是给使用率设报警，例如活跃数超过上限的一半就预警，而不是等超时异常才改配置。演示可以用 JConsole；生产上应把活跃数、空闲数、等待线程、获取耗时接到监控系统。

改完配置一定要确认它真的作用在正在使用的那个池上。有过一次大促扩容：把 Druid 的 `maxActive` 从 40 调到 80，当天仍因连接不够被打爆。事后发现框架升级时连接池已经换成 Hikari，Druid 那组键不再被读取，Hikari 仍跑在默认的 10。调参要在监控里看见上限变了、等待掉了，才算生效。

## 把一次写乱摊到字节上

红键的命令是读 `sku:red`，蓝键是读 `sku:blue`。RESP 里一条命令先写星号和参数个数，再写两段带长度的 bulk：命令名、键名。线程甲可以已经把星号和 `GET` 写进输出流，键还没写；线程乙此时把整条读蓝键的命令插进来；甲再补上 `sku:red`。服务端拼出来的不再是任何一条合法命令，于是关掉连接。客户端这边就表现为流突然结束，或者下一次写的时候套接字已经关闭。

另一种交错没有把单条命令拆开，只是两条完整命令挨在一起写。服务端按顺序回了 A 和 B，它们排在同一条输入流里。谁先读，谁拿走队头那个 bulk。蓝键线程先读到 A，日志就是“期望 B，实际是 A”。这不是值被写错，是响应被别的线程领走了。

每条线程从池里借到自己的 `Jedis` 之后，输出流和输入流都只属于这一次借用，写和读不再交叉。借用期间不要把这个对象塞进别的线程，也不要放进异步回调。

归还同样关键。忘了 `close`，对象不会回到 `GenericObjectPool`，活跃计数一直涨，很快所有调用都堵在最大等待时间上，现象像“Redis 卡死”，其实池被借空了。try-with-resources 保证断言失败、运行时异常也会归还。归还前客户端会清掉自己的会话状态，例如没结束的事务标记；清不掉就按损坏连接处理，避免下一位借用者读到上一位留下的半条协议。

## 最大连接数可以先算，再拿监控核对

开通账户的例子里，事务把连接占了大约 400 毫秒。如果峰值是每秒 50 次，同时被占住的连接大约是 50×0.4＝20。上限取这个数再留一半余量，落在 30 上下。示例里写成 40，是为了让等待线程归零之后仍看得到空闲。这个乘法只在“慢操作包在事务里面”时成立。睡眠若移到事务提交之后，连接在 `save` 返回时就还回去了，池可以小得多。先确认慢调用在不在事务边界内，再决定要不要把池调大。

池很大却仍然超时，常见是旁边这几项在拖后腿：

- 最小空闲如果等于最大连接，进程一启动就把连接建满，数据库上会躺着一排闲置会话。流量起伏大时，最小空闲保持较小值，让池按需增长。
- 空闲回收太慢，数据库或中间的负载均衡已经按自己的空闲上限把会话踢了，池还以为能用。借出前做一次轻量探测，或者打开保活，才能避开“拿到一条已经死掉的连接”。
- 连接的最长寿命应短于服务端和负载均衡的切断时间，由池主动换新，而不是等下一次查询才暴露断开。
- HTTP 池要同时看总连接和每路由连接。总数放得很大、每路由仍是很小的默认值时，打向同一台主机的调用照样排队。前面把两者都设成 1，是为了观察复用，不是推荐的生产值。
- Jedis 对应的是 `maxTotal`、`maxIdle`、`minIdle` 和借用等待。只放大总数、不设置等待上限，池耗尽时业务线程会一直堵着，接口看起来是挂起而不是失败。等待时间应略短于调用方的整体超时，把错误留在本服务里做降级。

驱逐线程的数量也跟着“池的份数”走。每个 HttpClient 连接管理器一条。进程内复用一份管理器，就只有一条驱逐线程；每次请求新做一个客户端，这种线程会和 `CLOSE_WAIT` 一起涨。用完就 `close` 能把线程停掉，所以能止住泄漏，但下一次请求仍要重新握手。又要止泄漏、又要复用连接，仍然是一份静态客户端加上退出钩子，而不是在请求路径上新建。

改完任一参数，都回到同一组指标上看：上限数字变了没有、活跃数是否落在预期并发附近、等待线程是否回到零。只改配置文件、不看运行时的池实现，就会重复前面 Druid 键已经失效、Hikari 仍在默认值上的事故。

借出前的探测也要单独看。Hikari 可以按间隔做 keepalive，避免把已经被数据库踢掉的会话再借给业务；探测本身会占一次往返，间隔不要短到和业务查询一样密。Jedis 侧如果打开 testOnBorrow，每次借用都会多一次 PING，延迟更稳，吞吐会下降。高压读接口可以改成 testWhileIdle，把探测放到空闲回收线程上，而不是放在请求路径里。

借来的连接也不要跨方法长期存放。有的代码把 `Jedis` 放进线程本地变量，请求结束忘了移除，线程回到容器线程池之后，下一次请求仍拿着上一次的连接，既不归还，也可能读到过期的协议状态。线程本地只适合放“当前调用栈一定能归还”的对象，并在过滤器的 finally 里清掉。数据库连接同理：让事务管理器在方法边界借还，业务代码不要自己 `getConnection` 之后塞进成员字段。

## 回顾

- 客户端有三种形态：池与连接分离、内部自带池、单连接。Jedis 是第一种，`JedisPool` 可共享，`Jedis` 不可跨线程。Apache HttpClient 是第二种，复用 `CloseableHttpClient` 即可。
- 使用上只有两条硬要求：池在进程里复用；退出前显式关闭，把驱逐线程和套接字一起收掉。池的初始化比借一条连接贵，不复用时甚至比每次新建短连接更差。
- 参数里最要紧的是最大连接数。高并发系统常因为上限太小而超时，上限太大又会压垮服务端。数据库、HTTP、Redis 这几类池都要有使用率报警，按容量而不是按默认值来调。

## 两个超时，以及 Mongo 客户端该不该复用

借连接常常有两段等待，不要配成同一个数。

- 从池里等一条闲置连接，叫借用等待。Jedis 放在 `JedisPoolConfig.setMaxWait`；HttpClient 放在请求配置的 `connectionRequestTimeout`；Hikari 的 `connectionTimeout` 就是调用方最多花多久从池里拿到一条连接，默认 30 秒。
- 池决定新建一条 TCP 时，三次握手自己的截止时间才是连接超时。Jedis 构造池时的 `timeout` / `connectionTimeout` 传给新建连接；HttpClient 用 `connectTimeout`；数据库则多半写在 JDBC URL 的 `connectTimeout` 上，和 Hikari 的借用等待不是同一个键。读超时（socketTimeout、Hikari 不直接等同）又是第三段，别和前两段混用。

MongoDB 的 Java 驱动里，`MongoClient` 自带连接池并且设计成线程安全，官方要求整个应用只建一份、在各处复用，进程结束时再关。每次请求 new 一个客户端，会重复建立监控线程和连接，和上面 HttpClient 的反例是同一类错误。
