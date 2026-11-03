# 一次远程调用要单独预算时间、重试和连接数

本地方法要么很快返回，要么在本进程里抛错。走 HTTP 则是一次网络往返：建连可能失败，发出去之后对方可能迟迟不写回，中间的代理还可能替你再发一次。三件事要提前定下来。

框架默认的等待是不是业务能接受的；超时或失败之后客户端会不会自动再试，服务端的接口承不承担得起再试；并发上来之后，客户端会不会像早期浏览器那样把同一主机的连接数卡死。

在仍使用 Netflix 组件的那一代 Spring Cloud 里，声明式客户端是 OpenFeign，负载均衡是 Ribbon。不走这套、只在 Spring Boot 里发请求时，常见的是 Apache HttpClient。下面的超时数字和重试开关都针对这一代组合。较新的 Spring Cloud 去掉了 Ribbon，改由 LoadBalancer 配合客户端自己的超时，属性名要以当前版本文档为准，但“建连”和“等响应”必须分开配，这个区分没有变。

## 建连等待和等响应不是同一个钟

HTTP 跑在 TCP 上。TCP 先握手再传数据，所以客户端几乎都有两个参数：

- 建连超时：三次握手最多等多少。
- 读超时：套接字上一次读操作最多阻塞多久。对一次调用来说，它覆盖的是把请求写出去之后，等到响应数据的时间，其中大部分通常是对方执行业务的时间。

两边要对这两个数有一致的预期，否则客户端先放弃、服务端还在算，或者客户端等得太久，把自己的线程池拖死。

建连时间设成几十秒没有意义。握手正常是毫秒到一两秒。几秒都连不上，多半是路由、防火墙或进程没听端口，再等一分钟也不会自己好。内网调用把建连压到一秒以内，下游机器下线时可以快速失败。公网可以放宽到几秒，不必到几十秒。

排查时先问清握手的对端是谁。客户端自己做负载均衡、直连实例，建连失败多半在实例或到实例的网络。前面还有一层反向代理时，客户端握的是代理，超时应先看代理是否接住连接，而不是先怀疑后面的业务进程。

读超时更容易想错，常见有三种。

第一种：认为客户端放弃等待之后，服务端也会停。用一个最小对照就能否定。客户端读超时 2 秒，服务端故意睡 5 秒：

```java
String callSlow(String path) throws IOException {
    return Request.Get("http://127.0.0.1:45678" + path)
        .connectTimeout(1_000)
        .socketTimeout(2_000)
        .execute()
        .returnContent().asString(java.nio.charset.StandardCharsets.UTF_8);
}

@GetMapping("/caller")
public String caller() throws IOException {
    return callSlow("/worker?pauseMillis=5000");
}

@GetMapping("/worker")
public String worker(@RequestParam int pauseMillis) throws InterruptedException {
    log.info("worker start");
    TimeUnit.MILLISECONDS.sleep(pauseMillis);
    log.info("worker finish");
    return "done";
}
```

客户端大约 2 秒后收到读取超时，服务端日志仍会在 5 秒时打印完成。容器已经把请求交给工作线程，套接字断开不会取消这条线程。因此超时之后不能假设“对方没改数据”。要靠业务上的状态、幂等键或查询来决定是重试、补偿还是告诉用户未知。

第二种：把读超时理解成纯网络传输时间，于是设成 100 毫秒。套接字层分不清是对方还没写完，还是包在路上。连接已经建立时，读超时几乎总是“对方处理加上回包”的预算。它应该按接口的耗时来设，而不是按理想的网络延迟。

第三种：以为等得越久成功率越高，把读超时放到数分钟。同步调用会占着一条容器线程。下游集体变慢时，线程堆在等待上，本服务先被拖垮。面向用户的同步接口，读超时很少有理由超过二三十秒。定时任务可以更长。不同下游的服务水平不同，超时就按接口分开设，而不是全局一个数。设得比对方正常耗时还短，调用会稳定失败，这是预算问题，不是框架故障。

连接池还会引出第三个容易漏掉的时间：从池里借连接的等待。建连超时管的是握手，读超时管的是已经拿到连接之后的读取。池里的连接都被占着时，调用线程堵在借连接上，既不会触发建连超时，也不会触发读超时。Apache HttpClient 里这是连接请求超时，要单独设短一点，否则下游变慢时，借连接的队列会把调用方的线程全部粘住。三个时间建议一起出现在客户端构造里：借连接、握手、读取。面向用户的接口，三者都按秒计，而不是按分钟。

超时之后的处理要写成状态机，而不是“捕获异常再调一次”。调用方本地生成一个幂等键，和订单号一起交给下游。下游用这把键做唯一约束：第一次插入业务行，重复请求只返回上一次的结果。这样即使客户端、网关各重试了一次，账上也只有一条。没有这把键时，读超时只能当成未知，先查询再决定要不要补单，不能直接再 POST 一次。

## 声明式客户端和负载均衡各有一套超时

Feign 自己能配超时，Ribbon 也能配。不看源码的话，很容易只改了一处，运行时仍走默认的 1 秒。

准备一个什么都不做、睡很久的下游：

```java
@PostMapping("/billing/hold")
public void hold() throws InterruptedException {
    TimeUnit.SECONDS.sleep(8 * 60);
}
```

声明式接口和调用端：

```java
@FeignClient(name = "billing")
public interface BillingClient {
    @PostMapping("/billing/hold")
    void hold();
}
```

```java
@GetMapping("/pay")
public String pay() {
    long begin = System.nanoTime();
    try {
        billingClient.hold();
        return "ok";
    } catch (RuntimeException ex) {
        long millis = TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - begin);
        log.warn("failed after {} ms: {}", millis, ex.getMessage());
        return "timeout";
    }
}
```

只配实例列表、不配超时时：

```properties
billing.ribbon.listOfServers=127.0.0.1:45678
```

调用大约 1 秒失败，错误是读取超时。Ribbon 的默认客户端配置里，建连和读取都被设成 1000 毫秒。这是第一个坑：默认值偏短，慢接口在没有任何业务异常时就会失败。

想改 Feign 的全局默认值，要同时写两个属性：

```properties
feign.client.config.default.connectTimeout=2500
feign.client.config.default.readTimeout=2500
```

只写读取那一行不会生效。工厂方法里是“两个值都不是 null 才构造新的 `Request.Options`”。缺一个，就继续用 Ribbon 已经放进配置的 1 秒。这是第二个坑。

某个服务要覆盖全局值时，把 `default` 换成 `@FeignClient` 的名字，两个值仍然都要写：

```properties
feign.client.config.billing.connectTimeout=2000
feign.client.config.billing.readTimeout=2000
```

调用会改成大约 2 秒失败。单独客户端覆盖全局，这是符合直觉的，不算坑。

也可以改 Ribbon 一侧。属性名的首字母是大写，和 Feign 的驼峰不同：

```properties
ribbon.ConnectTimeout=4500
ribbon.ReadTimeout=4500
```

两边同时配置时，以 Feign 为准。例如 Feign 写成 2.5 秒、Ribbon 写成 4.5 秒，失败发生在大约 2.5 秒。原因在负载均衡包装的客户端里：请求上的 `Options` 若不是默认对象，就用一份从 Feign 选项造出来的配置替换 Ribbon 的默认配置。容易误以为更底层的 Ribbon 一定赢，实际相反。

还有一种假象：只配了 Feign 的读取超时，又配了 Ribbon 的 4.5 秒，结果按 4.5 秒失败。这不是 Ribbon 覆盖了 Feign，而是第二个坑仍在：Feign 的读取超时单独出现时根本没变成 `Options`，于是继续用 Ribbon。

排错时看失败耗时最直接。耗时贴近哪一组数字，生效的就是哪一组。两个值要成对出现，Ribbon 的键注意大小写。

可以把覆盖关系收成一张对照，数字换成你自己的配置后再看耗时：

| 实际写上的配置 | 大约多久失败 | 原因 |
| --- | --- | --- |
| 只给了实例列表 | 1 秒 | Ribbon 默认建连和读取都是 1000 毫秒 |
| 只写 Feign 的读取时间 | 仍是 1 秒 | 两个字段缺一，选项对象不会被换掉 |
| Feign 的建连和读取都写了 | 贴近 Feign 的数字 | 请求上带了非默认选项 |
| 只写 Ribbon，且首字母大写 | 贴近 Ribbon 的数字 | Feign 没有覆盖选项 |
| 两边都成对写了 | 贴近 Feign | 非默认选项会替换负载均衡器里的客户端配置 |
| Feign 只写了读取，Ribbon 写了 4500 | 贴近 4500 | 看起来像被底层覆盖，其实 Feign 那一行没生效 |

命名空间也容易写错。`billing.ribbon.ReadTimeout` 只作用于名字为 billing 的客户端；`ribbon.ReadTimeout` 是全局。Feign 侧同理，`feign.client.config.billing` 覆盖 `feign.client.config.default`。改完配置要确认进程真的重新加载了，旧进程仍用启动时算好的选项。

较新的栈若已经没有 Ribbon，声明式客户端的超时往往直接落在底层 HTTP 实现上，例如 `spring.cloud.openfeign.client.config` 这一组。迁移时把“两个值都要出现”这件事重新验证一次，不要假设键名没变、默认的 1 秒也还在。验证方法不变：下游睡得足够久，看客户端在第几秒抛出读取超时。

## 负载均衡会替 GET 再打下一台

有的客户端觉得网络丢包是短故障，失败后换一台再试一次，出发点是好的。接口若不是幂等的，这一次“帮忙”就是重复副作用。

一个支付通知接口用 GET，参数放在查询串里，服务端睡 2 秒模拟慢处理，本身没有去重：

```java
@GetMapping("/notify/paid")
public void notifyPaid(@RequestParam String orderNo, HttpServletRequest request)
        throws InterruptedException {
    log.info("hit {} order={}", request.getRequestURL(), orderNo);
    TimeUnit.MILLISECONDS.sleep(2_000);
}
```

```java
@FeignClient(name = "notifier")
public interface NotifierClient {
    @GetMapping("/notify/paid")
    void notifyPaid(@RequestParam("orderNo") String orderNo);
}
```

```properties
notifier.ribbon.listOfServers=127.0.0.1:45679,127.0.0.1:45678
```

调用端只打一次。日志里却能看到两台机器各收到一条，订单号相同。调用端大约 2 秒后才报读取超时：第一次在 1 秒时读超时，Ribbon 换到下一台又等了 1 秒。客户端代码路径只有一条，副作用发生了两次。

Ribbon 里 `MaxAutoRetries` 默认 0，同一台不重试；`MaxAutoRetriesNextServer` 默认 1，失败后可以换一台。重试策略还要求方法是 GET，或者显式允许所有方法重试。所以这个“自动再试一次”只砸在 GET 上，更隐蔽。

处理有两条：

- 把通知改成 POST。有副作用的接口不该用 GET。选择方法看的是语义：查询用 GET，提交变更用 POST 或 PUT。不是“参数短就用 GET，参数长就用 POST”。GET 可以被缓存、被预取、被代理重试，这是协议给它的待遇。
- 在确认调用方必须保留 GET 时，把换机重试关掉：`ribbon.MaxAutoRetriesNextServer=0`。同一台的 `MaxAutoRetries` 保持 0，除非下游明确承诺幂等。

责任在两边。提供方应让通知可以按订单号去重，重复投递只生效一次。调用方则要知道负载均衡器的默认重试，否则会在自己的代码里反复找一个不存在的 for 循环。

把时间轴摊开更直观。假设读取预算是 1 秒，下游固定睡 2 秒，列表里有 45679 和 45678 两台：

- 0 毫秒，调用端进入，选中第一台，把 GET 发出去。
- 大约 1000 毫秒，第一台还在睡，调用端读超时，记一次失败。
- 紧接着换第二台，再发同一个查询串。此时第一台的睡眠还没结束，副作用已经发生。
- 大约 2000 毫秒，第二台也还没返回，调用端再次读超时，这次才把异常抛给业务。
- 第一台在自己的 2000 毫秒处打印完成，第二台也一样。业务日志只有一条调用，下游有两条命中。

所以“客户端 2 秒后才报错”并不等于读取超时配成了 2 秒，而是 1 秒预算被用了两次。HEAD 和 GET 一样会被当成可重试。如果健康检查误用了这种会改状态的 GET，探测流量本身就会把通知打出去。

关闭换机重试之后，失败应停在第一台，总耗时回到大约 1 秒，另一台的日志保持安静。这是回归时要看的两条证据，缺一不可。只看调用端不再抛两次异常是不够的，因为异常本来就只抛一次，重复发生在下游。

## 同一主机的默认并发只有两条

爬虫或网关自己也是 HTTP 客户端。线程池已经扩得很大，总耗时仍几乎线性增长，瓶颈常常在连接池，不在线程。

下游每个请求睡 1 秒后返回 1。爬虫用无界的 `newCachedThreadPool` 把任务扔进去。这里只为了证明“线程不是上限”；生产里不要用没有上限的池，任务暴涨时线程会先把机器打满。

```java
int harvest(int count, Supplier<CloseableHttpClient> client) throws InterruptedException {
    AtomicInteger sum = new AtomicInteger();
    ExecutorService pool = Executors.newCachedThreadPool();
    long begin = System.nanoTime();
    for (int i = 0; i < count; i++) {
        pool.execute(() -> {
            HttpGet get = new HttpGet("http://127.0.0.1:45678/meter/once");
            try (CloseableHttpResponse response = client.get().execute(get)) {
                sum.addAndGet(Integer.parseInt(EntityUtils.toString(response.getEntity())));
            } catch (IOException ex) {
                System.err.println("harvest failed: " + ex);
            }
        });
    }
    pool.shutdown();
    pool.awaitTermination(1, TimeUnit.HOURS);
    long millis = TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - begin);
    log.info("completed {} in {} ms", sum.get(), millis);
    return sum.get();
}
```

默认的 `PoolingHttpClientConnectionManager` 在这一代实现里，同一路由最多 2 条连接，所有路由加起来最多 20 条。10 个各睡 1 秒的请求因此大约要 5 秒：两条两条地过闸。20 这个总数在“只打一个主机、并发 10”时还不是瓶颈。若同一个客户端要打 10 个主机、每台 10 路并发，每路由上限设 10 的同时，总数至少要 100，否则总数会先满。

默认值小，是因为 HTTP/1.1 那个年代建议单用户客户端对同一服务器不要维持超过两条连接，代理再按活跃用户数放大。目的是避免少数客户端把服务器挤满。今天的浏览器早就不守这个上限了。爬虫、网关、高并发的聚合服务，必须自己把每路由上限和总上限抬到和下游容量匹配的水平，无论用的是哪一家客户端。

HttpClient 4 的写法：

```java
CloseableHttpClient tuned = HttpClients.custom()
        .setMaxConnPerRoute(50)
        .setMaxConnTotal(100)
        .build();
```

10 个各 1 秒的请求这时可以在大约 1 秒内跑完。抬高连接数之后，下游和本机文件描述符变成新的上限，需要一起看。

判断是不是被每路由上限卡住，有一个简单对照：把并发从 2 提到 10，若耗时几乎不降，再看客户端配置而不是先加机器。耗时接近 `ceil(请求数 / 每路由上限) * 单次耗时` 时，闸门就是连接池。例如 10 个 1 秒的请求、每路由 2 条，理论值约 5 秒，和默认池的表现一致。

租约还回也要放在关闭响应的同一处。上面的例子用了 try-with-resources 关闭响应，连接才能回到池里。若只读了流、忘记关闭，池里的 50 条会慢慢全部处于租出状态，后面的请求又开始在借连接处排队，现象变回“超时”，根因却是泄漏。压测前后看已租出连接数，应该回到接近 0。

不同主机不要共用一个很小的总数。一个客户端同时抓十个站点，每站希望 10 路，总数至少是 100。只把每路由抬到 10、总数仍是 20，前两个站点就会把总额占满，其余站点全部排队。网关进程特别容易踩中：它用一个静态客户端访问所有下游，默认总额 20 会变成整个网关的出口上限。

把三个时间放进同一个客户端，避免只改了读超时、借连接仍无限等：

```java
RequestConfig budget = RequestConfig.custom()
        .setConnectionRequestTimeout(500)
        .setConnectTimeout(1000)
        .setSocketTimeout(2000)
        .build();

CloseableHttpClient client = HttpClients.custom()
        .setDefaultRequestConfig(budget)
        .setMaxConnPerRoute(50)
        .setMaxConnTotal(100)
        .evictIdleConnections(30, TimeUnit.SECONDS)
        .build();
```

`connectionRequestTimeout` 是从池里拿到连接的上限。池被占满时，500 毫秒后失败，而不是一直堵到容器线程耗尽。`evictIdleConnections` 丢掉闲置过久的连接，减少“池里这条已经被对端关掉、下一次读才发现”的情况。空闲时间要小于防火墙切断空闲 TCP 的时间，常见是几十秒，而不是默认的一直留着。

调用点仍然要关闭响应。配置了池并不等于每次 `execute` 都会新建连接；不关闭就把租约挂死，上面的 50 会慢慢变成 0 条可用。可以用一次压测验证：并发 20、每个下游睡 1 秒，打开驱逐和三个超时之后，耗时应接近 1 秒，而不是接近 10 秒。若仍接近 10 秒，先打印连接池的已租出数量，再怀疑超时数字。

声明式客户端没有这么直观的三个旋钮时，至少确认两件事：读超时已经不是默认的 1 秒，以及失败后的 GET 不会被负载均衡器再发到下一台。这两条用日志的时间戳和下游的命中次数就能证明，不必先上链路追踪。

调用方自己再包一层重试时，间隔要拉开，并且只对幂等方法或带幂等键的 POST 进行。立即重试会在下游已经超时的时候再叠一批请求，把恢复中的服务再次打满。指数后退加上少量抖动，比固定间隔更不容易让所有实例在同一毫秒重试。重试次数写成 1 或 2 就够看网络抖动；更多次数通常是在掩盖超时预算设得太短。

下游如果提供查询接口，读超时之后的补偿应先查再决定是否重放。例如支付通知超时后，用订单号问一次“是否已入账”，已入账就结束，未入账才带同一个幂等键再 POST。没有查询接口时，宁可把这次调用标成未知并告警，也不要无键重放。网关层的 `proxy_next_upstream` 若同时开着，应用层的这一次重试会再乘上网关的换机次数，预算要按乘积来估。

日志里至少留下四样东西：选中的下游地址、幂等键、从发出到异常的毫秒数、异常类型是建连失败还是读取超时。只有异常类型时，无法区分“1 秒预算用了两次”和“本来就配了 2 秒”。把这四样打在同一行，前面的通知重复、抓取变慢才有办法在一次事故里对上。

再单独看一眼连接复用。开启长连接之后，同一条 TCP 会连续发多个请求，读超时仍是按每一次读取计算，不会因为连接已经建好就取消。对方若在两次响应之间把连接关掉，下一次写入可能先成功，读的时候才得到连接重置。这种失败和“从未握手成功”不同：建连超时不会出现，堆栈落在读取或重置上。重试策略若把重置一律当成可重试的 GET，又会回到重复副作用。把异常类型分成握手失败、读取超时、连接重置三类，再决定要不要重放。

池的等待时间也要出现在这行日志里。借连接超过预算时，异常往往不是 `SocketTimeoutException`，而是连接池自己的超时类型。若监控只按读取超时告警，池被占满会完全看不见，线程却已经在借连接的队列里堆积。告警维度至少分开：借连接、握手、读取。三者同时变长，才是下游处理变慢；只有借连接变长，是自己的池太小或响应没关闭。

本地把这三类失败做出来，比背默认数字有用。下游进程不启动，短建连超时应在一秒内失败，读取超时不会出现。下游启动但睡眠长于读取预算，异常是读取超时，下游日志仍会在睡眠结束后打印完成。把池的每路由上限临时改成 1，再用两条各睡一秒的调用并行打出去，总耗时会接近两秒；改回足够大的上限后应回到一秒左右。三条都符合预期，才说明超时和并发闸门按你写的配置生效，而不是仍停在框架默认值上。

## 收束

建连超时是握手的预算，读超时是等对方把响应写回来的预算，里面包含对方的处理时间。建连失败先确认握的是实例还是代理。读超时按下游的服务水平和自己能被拖住多久来定，并确认框架里这两项真的覆盖了默认值。Feign 那一代要两个值一起写，同时配置时 Feign 压过 Ribbon。

重试方面，协议把 GET 和 HEAD 当成可重复的查询，所以客户端和代理会自作主张再发一次。接口做不到幂等，就关掉自动重试；更好的做法是让方法和语义一致，变更用 POST，并在服务端按业务键去重。

连接池和浏览器一样会限制到同一主机的并发。爬虫和网关若留着默认的每路由 2 条，线程加得再多也上不去。把每路由和总数调到与目标容量一致，再压一次看耗时是否接近单次调用。

## 可以顺着想的三个问题

为什么很少单独说写超时？一次短请求把字节写进发送缓冲区通常很快就返回，墙钟时间耗在等响应，所以读超时已经覆盖了主要等待。真正会在写阶段堵住的是大文件上传：对端接收窗口为 0 时，`write` 会停住。Apache HttpClient 的 `socketTimeout` 主要约束读；OkHttp 则把写超时单独设了 `writeTimeout`。上传接口要核对客户端有没有写方向的截止时间，不能假设读超时能管住发送缓冲区。

Nginx 也有类似的换上游重试。`proxy_next_upstream` 默认包含 `error` 和 `timeout`。请求已经发往上游之后，非幂等方法默认不会再换一台，除非把 `non_idempotent` 加进这个指令。想完全禁止，设成 `off`。网关层的重试和客户端的重试叠在一起时，一次用户点击可能打出多于两次的下游调用。

还有一类坑是空闲连接被对端或防火墙拆掉，池里仍把它当可用，下一次调用才在读的时候失败。给连接池设空闲校验或较短的存活时间，能减少这种“第一次失败、立刻重试就成功”的假故障。重试若开着，记得这种失败同样会把非幂等请求打两次。
