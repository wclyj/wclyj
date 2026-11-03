# Nacos 如何给实例续约、推送和摘除

临时实例不会一直留在名单里。客户端大约每 5 秒上报一次，服务端用这份上报刷新「最后见到」的时间；太久不见就标成不健康，再久就删掉。名单变化时，服务端还可以用 UDP 通知正在订阅的调用方。下面按客户端、服务端、推送、健康检查四段把这条链走完。示例里的服务叫 `pay-ledger`，分组 `TRADE`，地址 `10.8.4.12:9106`。

## 一、Nacos 服务心跳

### 1.1、客户端心跳

客户端自己维护调度，不停调用服务端的续约接口，避免自己被摘掉。默认周期 5 秒，路径是命名模块下的 `/instance/beat`。

#### 相关类怎样被装进容器

`nacos-discovery` 在 `META-INF/spring.factories`（新版本则是 `AutoConfiguration.imports`）里声明自动配置。和登记有关的配置类会准备三个 Bean：

- 登记器：真正调用客户端 SDK 的注册与注销。
- 登记资料：保存服务名、IP、端口、分组、元数据。
- 自动登记：监听 Web 容器已启动的事件，等端口确定后再去登记。

条件大致是：发现功能开着，并且 `spring.cloud.service-registry.auto-registration.enabled` 不是 false。它还要排在通用自动登记配置之后，这样监听到的是「端口已经绑定」而不是配置还没绑完。

```java
@Configuration
public class LedgerRegistryConfig {

    @Bean
    public NacosServiceRegistry ledgerRegistry(NacosDiscoveryProperties props) {
        return new NacosServiceRegistry(props);
    }

    @Bean
    public NacosRegistration ledgerRegistration(
            NacosDiscoveryProperties props, ApplicationContext ctx) {
        return new NacosRegistration(Collections.emptyList(), props, ctx);
    }

    @Bean
    public NacosAutoServiceRegistration ledgerAutoReg(NacosServiceRegistry registry,
            AutoServiceRegistrationProperties autoProps, NacosRegistration reg) {
        return new NacosAutoServiceRegistration(registry, autoProps, reg);
    }
}
```

`NacosAutoServiceRegistration` 继承 Spring Cloud 的抽象自动登记，而那个抽象类监听 `WebServerInitializedEvent`。事件一到，才会调用登记器的 `register`。所以只创建了 Bean、容器还没监听端口时，控制台里看不到实例。

#### 登记器把请求交给 SDK

登记器实现 `ServiceRegistry`。服务名为空就直接返回。否则从发现属性里取分组，把 Spring 的注册资料转成 Nacos 的 `Instance`，再调用 `NamingService.registerInstance`。

```java
public void register(Registration reg) {
    if (!StringUtils.hasText(reg.getServiceId())) {
        log.warn("skip register, empty service id");
        return;
    }
    Instance inst = toInstance(reg);
    try {
        naming().registerInstance(inst.getServiceName(), props.getGroup(), inst);
    } catch (Exception ex) {
        if (props.isFailFast()) {
            throw new RuntimeException("register pay-ledger failed", ex);
        }
        log.warn("register failed but fail-fast is off: {}", inst.getPort());
    }
}
```

`fail-fast` 为真时，登记失败会让进程起不来；为假时只打警告，进程继续跑，但调用方找不到它。本地试验可以放宽，预发环境更适合让启动失败，避免「进程是绿的、名单里没有」。

#### registerInstance 同时做两件事

SDK 里的 `NacosNamingService.registerInstance` 并不只是发一次 HTTP。对临时实例（默认就是临时的）它先建立续约任务，再发注册请求。

```java
public void registerInstance(String service, String group, Instance inst) {
    String grouped = group + "@@" + service;
    if (inst.isEphemeral()) {
        BeatInfo info = beats.buildBeatInfo(grouped, inst);
        beats.addBeatInfo(grouped, info);
    }
    proxy.registerService(grouped, group, inst);
}
```

顺序值得注意：续约任务先放进调度器，注册报文随后才发出。若注册报文失败，调度器里可能已经有一个会继续打的任务。关闭进程时要走注销，让调度停掉，而不是只杀线程。

持久实例不进这段 `if`。它不靠续约活着，服务端也不会因为几十秒没心跳就删它。

#### BeatInfo 里放什么

`BeatInfo` 是续约报文的快照，不是线程本身。构建时从实例拷走这些字段：

- 带分组的服务名，例如 `TRADE@@pay-ledger`
- IP `10.8.4.12`、端口 `9106`
- 集群名，缺省 `DEFAULT`
- 权重和元数据
- 周期，取实例上的心跳间隔

间隔来自元数据键；没有这个键时用常量，即 5 秒对应的毫秒数。想改成 3 秒或 8 秒，应在注册元数据里写保留键，而不是改客户端常量后重新打包。

```java
BeatInfo info = new BeatInfo();
info.setServiceName(groupedName);
info.setIp(inst.getIp());
info.setPort(inst.getPort());
info.setCluster(inst.getClusterName());
info.setWeight(inst.getWeight());
info.setMetadata(inst.getMetadata());
info.setPeriod(inst.getInstanceHeartBeatInterval());
```

`scheduled` 初始为假，表示还没进过调度。周期字段决定下一次延迟，服务端也可以在响应里改掉它。

#### 放进调度池

`BeatReactor` 持有一个 `ScheduledThreadPoolExecutor`。线程是守护线程，名字类似 `com.alibaba.nacos.naming.beat.sender`，避免阻止 JVM 退出。

`addBeatInfo` 用「服务名 + IP + 端口」做键。若 map 里已有旧任务，先把旧对象标成停止，再放入新的，并按周期做一次延迟调度。这样重复注册不会叠出两条永不停的任务。

```java
public void addBeatInfo(String serviceName, BeatInfo info) {
    String key = serviceName + "#" + info.getIp() + "#" + info.getPort();
    BeatInfo previous = beatMap.remove(key);
    if (previous != null) {
        previous.setStopped(true);
    }
    beatMap.put(key, info);
    pool.schedule(new BeatTask(info), info.getPeriod(), TimeUnit.MILLISECONDS);
}
```

`setStopped(true)` 只是一个标志。已经在跑的那一次不会被打断，但它跑完后看到标志，就不再预约下一次。

#### BeatTask 每次做什么

`BeatTask` 是一次性任务，不是 `scheduleAtFixedRate`。跑完自己再预约下一次，间隔可以被响应改写。

一次执行：

1. 已停止则立刻返回。
2. 默认下一次仍用本地周期。
3. 调用 `sendBeat`。
4. 从 JSON 里读 `clientBeatInterval`。大于 0 就改写下一次延迟。
5. 读轻量续约开关，记到反应器上，下一次可以少带报文体。
6. 若业务码表示资源不存在，立刻用 `BeatInfo` 里的字段再注册一次。
7. 无论成功失败，`finally` 里再 `schedule` 一个新的 `BeatTask`。

```java
public void run() {
    if (info.isStopped()) {
        return;
    }
    long delay = info.getPeriod();
    try {
        JsonNode rsp = proxy.sendBeat(info, lightBeat);
        if (rsp.has("clientBeatInterval") && rsp.get("clientBeatInterval").asLong() > 0) {
            delay = rsp.get("clientBeatInterval").asLong();
        }
        lightBeat = rsp.path("lightBeatEnabled").asBoolean(false);
        if (rsp.path("code").asInt() == NOT_FOUND) {
            proxy.registerService(info.getServiceName(), groupOf(info), fromBeat(info));
        }
    } catch (Exception ex) {
        log.warn("beat failed, port={}, msg={}", info.getPort(), ex.getMessage());
    } finally {
        if (!info.isStopped()) {
            pool.schedule(new BeatTask(info), delay, TimeUnit.MILLISECONDS);
        }
    }
}
```

资源不存在通常说明服务端重启过，或健康检查已经把实例删了。客户端靠下一次续约发现这件事并补登记，不需要人去点控制台。

`finally` 保证网络抖动时任务不会丢。若忘记再调度，实例会在服务端超时后消失，而客户端仍以为自己在线。

#### sendBeat 的报文

方法是 PUT。轻量模式下不放 `beat` 字段，只留查询参数：命名空间、服务名、集群、IP、端口。完整模式才把整份 `BeatInfo` 序列化进报文体。

```java
public JsonNode sendBeat(BeatInfo info, boolean light) {
    Map<String, String> query = new HashMap<>();
    query.put("namespaceId", namespaceId);
    query.put("serviceName", info.getServiceName());
    query.put("clusterName", info.getCluster());
    query.put("ip", info.getIp());
    query.put("port", String.valueOf(info.getPort()));
    Map<String, String> body = new HashMap<>();
    if (!light) {
        body.put("beat", toJson(info));
    }
    String raw = reqApi("/instance/beat", query, body, "PUT");
    return parse(raw);
}
```

第一次往往是完整报文，因为服务端可能还没有这份实例，需要从 body 里还原。服务端确认之后会告诉客户端「下次不必带 beat」。于是稳态流量只剩几项查询参数。

注册报文是另一条 POST，字段包括 IP、端口、权重、是否可用、是否健康、是否临时、元数据 JSON。续约和注册不要混成一个接口来记。

客户端这段可以收成三句：调度器按周期发 UDP 之外的 HTTP 续约；服务端用响应改周期、改轻重；若被告知实例不在，就重新注册。服务端也会按收到的续约更新自己的时间戳。

### 1.2、服务端心跳

服务端入口在实例控制器的 PUT `/beat`。有无报文体，决定这是一次「带档案的续约」还是一次「只点名的续约」。

#### 分支

处理顺序可以分成这些判断：

1. 响应里先放上服务端当前的续约间隔，后面还可以按实例元数据覆盖。
2. 请求参数 `beat` 若非空，反序列化成 `RsInfo`。空则说明客户端已经在发轻量续约。
3. IP、端口、集群优先取报文体，其次取查询参数。
4. 按命名空间、服务名、集群、IP、端口找内存里的实例。
5. 找不到，并且也没有报文体：返回「资源不存在」。客户端见到这个码会自己补注册。
6. 找不到，但报文体在：用报文体组装 `Instance` 再登记。这是健康检查误删或服务端丢过内存之后的补偿。
7. 服务对象本身都不存在：这是服务端错误，直接抛出，而不是假装续约成功。
8. 报文体为空时，用查询参数拼一个最小的 `RsInfo`，保证后面的处理器有 IP 和端口。
9. 交给 `service.processClientBeat`。
10. 响应码改成成功，并带回是否允许轻量续约。若实例元数据里写了自己的间隔，用它覆盖第 1 步的默认值。

```java
@PutMapping("/beat")
public ObjectNode beat(HttpServletRequest req) throws Exception {
    ObjectNode out = emptyNode();
    out.put("clientBeatInterval", switches.clientBeatInterval());
    String raw = optional(req, "beat");
    RsInfo rs = blank(raw) ? null : fromJson(raw, RsInfo.class);
    String ip = rs != null ? rs.getIp() : optional(req, "ip");
    int port = rs != null ? rs.getPort() : intParam(req, "port");
    String cluster = rs != null && rs.getCluster() != null
            ? rs.getCluster() : optional(req, "clusterName");
    Instance found = services.find(ns(req), service(req), cluster, ip, port);
    if (found == null && rs == null) {
        out.put("code", NOT_FOUND);
        return out;
    }
    if (found == null) {
        services.register(ns(req), service(req), toInstance(rs, cluster));
    }
    Service svc = services.lookup(ns(req), service(req));
    if (rs == null) {
        rs = new RsInfo(ip, port, cluster);
    }
    svc.processClientBeat(rs);
    out.put("code", OK);
    out.put("lightBeatEnabled", switches.lightBeat());
    return out;
}
```

补偿路径要打警告日志。它说明名单和客户端认知不一致，偶尔出现正常，持续出现就要查是不是有节点在反复删实例。

#### processClientBeat 只负责丢进线程池

服务对象不在 HTTP 线程里扫描全部实例。它把 `ClientBeatProcessor` 交给命名模块的健康检查调度，延迟设为 0，也就是尽快执行。

```java
public void processClientBeat(RsInfo rs) {
    ClientBeatProcessor job = new ClientBeatProcessor();
    job.setService(this);
    job.setRsInfo(rs);
    HealthCheckReactor.scheduleNow(job);
}
```

HTTP 线程因此可以马上把间隔和轻量开关返回给客户端。真正改「最后续约时间」的是后面的任务。

#### 处理器更新时间戳并在复活时发事件

任务按集群取出实例列表，找到 IP 和端口都相同的那一台：

- 把 `lastBeat` 写成当前毫秒。
- 若该实例之前被标成不健康，并且没有被人工摘除标记挡住，就改回健康。
- 健康状态从假变成真时，调用推送服务的 `serviceChanged`，让订阅方尽快知道它回来了。

```java
for (Instance item : cluster.allIPs(true)) {
    if (!item.getIp().equals(rs.getIp()) || item.getPort() != rs.getPort()) {
        continue;
    }
    item.setLastBeat(System.currentTimeMillis());
    if (!item.isMarked() && !item.isHealthy()) {
        item.setHealthy(true);
        pushService.serviceChanged(service);
    }
}
```

只更新时间、不改变健康位时，不必推送。调用方本地缓存还认为它健康，多推一次只会放大 UDP。

`serviceChanged` 发布 `ServiceChangeEvent`。若这个服务已经有一个待执行的推送任务，就不要再排第二个，避免同一秒内变更被推很多遍。

```java
public void serviceChanged(Service changed) {
    String key = changed.getNamespaceId() + "@@" + changed.getName();
    if (futureMap.containsKey(key)) {
        return;
    }
    context.publishEvent(new ServiceChangeEvent(this, changed));
}
```

### PushService 类结构

`PushService` 实现 `ApplicationListener<ServiceChangeEvent>`。它是事件的消费者：名单变了，就按服务维度给订阅者发 UDP。

#### 一次推送的主体

`onApplicationEvent` 里排一个只跑一遍的延迟任务，大约 1 秒后执行，并用 `futureMap` 记下这个服务正在推。任务结束时把键删掉，后面的变更才能再次进入。

任务内部：

1. 从 `clientMap` 取出这个服务的 UDP 客户端表。表是空的就返回。
2. 跳过僵尸客户端，并把它们从地图删掉。
3. 为每个客户端准备数据包。缓存窗口很大时可以复用已压缩的字节；默认窗口较小，通常每次重新组装。
4. 调用 `udpPush` 把数据报发出去。

```java
public void onApplicationEvent(ServiceChangeEvent change) {
    Service svc = change.getService();
    String key = fullName(svc);
    Future<?> job = udpPool.schedule(() -> {
        try {
            Map<String, PushClient> clients = clientMap.get(key);
            if (clients == null || clients.isEmpty()) {
                return;
            }
            for (PushClient client : new ArrayList<>(clients.values())) {
                if (client.zombie()) {
                    clients.remove(client.mapKey());
                    continue;
                }
                AckEntry entry = prepare(client, hostsOf(client));
                udpPush(entry);
            }
        } finally {
            futureMap.remove(key);
        }
    }, 1, TimeUnit.SECONDS);
    futureMap.put(key, job);
}
```

延迟 1 秒是为了把紧挨着的多次变更合成一次推送。它不是每秒轮询，只在事件到达时预约一次。

#### udpPush 与重试

发送前看重试次数。默认最多再试 1 次，超过就把记录从待确认表删掉，计一次失败，不再发。

未超过时：

- 将数据包放进 `ackMap`，表示已发出、还没收到确认。
- 记下发送时刻。
- 用 `DatagramSocket` 发送。
- 再预约一个大约 10 秒后的重传任务。

```java
AckEntry udpPush(AckEntry entry) {
    if (entry == null) {
        return null;
    }
    if (entry.retryTimes() > 1) {
        ackMap.remove(entry.key());
        sendAt.remove(entry.key());
        return entry;
    }
    ackMap.put(entry.key(), entry);
    sendAt.put(entry.key(), System.currentTimeMillis());
    socket.send(entry.packet());
    entry.bumpRetry();
    retryPool.schedule(new Retransmit(entry), 10, TimeUnit.SECONDS);
    return entry;
}
```

重传任务本身很短：10 秒后若 `ackMap` 仍有这只包，说明确认没到，再调用一次 `udpPush`。确认若已经把键删了，重传任务什么都不做。

所以失败或超时的代价是多一次 UDP，不是无限重试。调用方即使没收到推送，仍可在下次拉清单时对账，推送只是加速。

#### 静态块里启动的两件后台工作

类被加载时会做三件事：创建数据报套接字、拉起接收线程、每 20 秒扫一次僵尸客户端。

```java
static {
    socket = new DatagramSocket();
    Thread reader = new Thread(new AckReader(), "naming-push-ack");
    reader.setDaemon(true);
    reader.start();
    retryPool.scheduleWithFixedDelay(PushService::dropZombies, 0, 20, TimeUnit.SECONDS);
}
```

接收线程和清理任务都挂在这个类上，不随某一次 HTTP 请求创建。套接字创建失败时，推送整体不可用，但续约仍走 HTTP，注册中心还能用，只是订阅方更新变慢。

#### 接收线程只维护三张表

循环读取 UDP。缓冲区给到几十 KB，够一份实例列表。读到的 JSON 转成确认包，用对端 IP、端口和上次引用时间拼出键。

- 若确认来得比 10 秒阈值还晚，打一条警告，便于发现单程延迟。
- 从 `ackMap` 移除该键。移除不掉说明包已经因超时被丢掉，或键算法不一致。
- 用发送时刻计算耗时，放进耗时表，再从发送时刻表删掉。

```java
public void run() {
    byte[] buf = new byte[64 * 1024];
    while (!Thread.currentThread().isInterrupted()) {
        DatagramPacket packet = new DatagramPacket(buf, buf.length);
        socket.receive(packet);
        AckPacket ack = parse(packet);
        String key = ackKey(packet, ack);
        AckEntry waiting = ackMap.remove(key);
        if (waiting == null) {
            continue;
        }
        long cost = System.currentTimeMillis() - sendAt.getOrDefault(key, 0L);
        costMap.put(key, cost);
        sendAt.remove(key);
    }
}
```

三张表的分工：

- `ackMap`：发出去但还没确认的包。
- `sendAt`：每个包开始发送的时间。
- `costMap`：确认到达后的耗时，用来观察推送是否越来越慢。

接收线程不做业务更新。实例名单的修改已经在续约和注册路径上完成，这里只结束「在途」状态。

#### 僵尸客户端

每 20 秒遍历 `clientMap`。某个 `PushClient` 若太久没有刷新引用时间，就从内层 map 去掉，以后的变更不再给它发 UDP。

```java
boolean zombie() {
    long idle = System.currentTimeMillis() - lastRefTime;
    return idle > switches.pushCacheMillis(serviceName);
}
```

`lastRefTime` 在客户端对象创建时设为当前时间，之后每次成功的交互再刷新。超过推送缓存时间（常见是 10 秒量级，具体看开关）仍无刷新，就视为对方不再订阅。判断用的是服务端时钟上的空闲时长，不是 UDP 是否丢过一次包。

#### UDP 客户端从哪注册进来

`clientMap` 不是启动时扫描出来的。订阅方来拉实例列表时，如果带了大于 0 的 UDP 端口，并且该客户端类型允许推送，服务端才调用 `addClient`。

组装推送体时会问数据源要一份当前名单。数据源内部走和查询接口相同的组装逻辑。有的调用把 UDP 端口传成 0，表示这次只是查询，不要开启推送。端口大于 0 才进入 `addClient`。

```java
boolean pushable = udpPort > 0 && pushService.allowPush(agent);
if (pushable) {
    InetSocketAddress sink = new InetSocketAddress(clientIp, udpPort);
    pushService.addClient(namespaceId, serviceName, clusters, agent, sink, dataSource);
}
```

因此「续约正常但调用方很久才知道下线」，有时不是 UDP 坏了，而是订阅请求根本没带端口，服务端从未把这个调用方放进 `clientMap`。调用方仍能靠周期性的拉清单接口更新，只是慢一个缓存周期。

#### addClient 如何写入

外层键是「命名空间 + 服务名」，因为变更事件按服务触发。内层键是客户端自己的地址描述。已存在则刷新引用时间；不存在则放入。

```java
public void addClient(PushClient incoming) {
    String serviceKey = incoming.getNamespaceId() + "@@" + incoming.getServiceName();
    ConcurrentMap<String, PushClient> table =
            clientMap.computeIfAbsent(serviceKey, k -> new ConcurrentHashMap<>());
    PushClient old = table.get(incoming.mapKey());
    if (old != null) {
        old.refresh();
    } else {
        table.put(incoming.mapKey(), incoming);
    }
}
```

并发下用 `putIfAbsent` 或 `computeIfAbsent`，避免两个查询同时新建内层表时丢掉其中一个客户端。

到这里，僵尸规则就完整了：客户端进表时记下时间；确认或后续查询会刷新；刷新停太久就被 20 秒一次的扫描清掉，下一次变更不再推给它。

### 心跳与推送对照

**推送**

- 名单变化发布服务变更事件。
- `PushService` 按服务找出 `clientMap` 里的订阅者，用 UDP 通知。
- 10 秒内没有确认，就从 `ackMap` 里再试一次。
- 默认只多试一次。仍无确认就放弃这只包。

**确认**

- 静态块里的守护线程一直读 UDP。
- 读到确认就从 `ackMap` 和发送时刻表移除，并记下耗时。

**订阅者集合**

- 同一静态块每 20 秒看一次 `clientMap`。
- 空闲超过推送缓存时间的客户端被移除。
- 新订阅者来自「带 UDP 端口的实例查询」，不是来自续约接口。

续约回答「这台机器还在不在」。推送回答「订阅方要不要马上更新本地列表」。两条线通过「健康位发生变化时发布事件」接在一起，但传输一个走 HTTP，一个走 UDP。

## Nacos 服务的健康检查

只有续约、没有检查的话，崩溃的进程会永远留在名单里。服务端另外有定时任务：超过约 15 秒没见到续约，把健康位置为假；超过约 30 秒，把实例删掉。实例恢复后续约到达，又会按上一节的补偿逻辑登记回来。

### 任务何时挂上

新服务放进管理器时会 `init`。初始化负责把 `ClientBeatCheckTask` 放进调度池，并为这份实例列表注册一致性监听。

```java
private void remember(Service service) {
    put(service);
    service.init();
    consistency.listen(instanceListKey(service, true), service);
    consistency.listen(instanceListKey(service, false), service);
}
```

临时实例和持久实例的监听键不同。健康检查主要约束临时实例。持久实例没有「30 秒不续约就删除」这条规则。

### 检查任务的两个阈值

任务先问自己是不是这台机器负责该服务。集群用 Distro 分片时，不负责的节点直接返回，避免三台一起删。总开关关掉健康检查时也返回。

然后取该服务全部临时实例：

- 现在减去 `lastBeat`，大于心跳超时（默认 15 秒）：若当前仍标记健康，则改为不健康，推送变更，并发布超时事件。
- 若配置允许过期删除，且空闲超过删除超时（默认 30 秒）：调用删除。

```java
public void run() {
    if (!distro.responsible(service.getName()) || !switches.healthCheck()) {
        return;
    }
    for (Instance item : service.allIPs(true)) {
        long idle = System.currentTimeMillis() - item.getLastBeat();
        if (idle > item.getInstanceHeartBeatTimeOut() && item.isHealthy() && !item.isMarked()) {
            item.setHealthy(false);
            push.serviceChanged(service);
        }
    }
    if (!config.expireInstance()) {
        return;
    }
    for (Instance item : service.allIPs(true)) {
        long idle = System.currentTimeMillis() - item.getLastBeat();
        if (!item.isMarked() && idle > item.getIpDeleteTimeout()) {
            deleteIp(item);
        }
    }
}
```

默认值：

```java
long beatTimeoutMs = TimeUnit.SECONDS.toMillis(15);
long deleteTimeoutMs = TimeUnit.SECONDS.toMillis(30);
```

15 秒这一档，调用方若尊重健康位，就不会再选这台机器，但记录还在，方便看「刚失去联系」。30 秒这一档才从名单移除。两个数都可以通过实例元数据覆盖，客户端构建 `BeatInfo` 时用的也是同一组保留键。把删除时间改得比超时还短没有意义，删除判断会先发生，不健康状态几乎看不到。

人工标记过的实例会被跳过，避免巡检和运维操作对着干。

### deleteIp 不直接改内存

删除走本机的 HTTP 注销接口，而不是在巡检线程里直接改集合。这样删除和外部调用的注销共用一条路径，一致性插件也能接到同一次写入。

请求是异步的。URL 指向本机端口上的命名上下文加 `/instance`，参数带 IP、端口、集群、服务名、命名空间，以及 `ephemeral=true`。

```java
void deleteIp(Instance item) {
    String url = "http://" + localIp() + ":" + port
            + contextPath + "/instance?ip=" + item.getIp()
            + "&port=" + item.getPort()
            + "&ephemeral=true&clusterName=" + item.getClusterName()
            + "&serviceName=" + service.getName()
            + "&namespaceId=" + service.getNamespaceId();
    http.asyncDelete(url, result -> {
        if (!result.ok()) {
            log.error("auto delete failed, port={}", item.getPort());
        }
    });
}
```

异步是为了不让巡检线程堵在 HTTP 上。失败只记日志，下一轮巡检仍会再试，因为实例还在且时间戳依旧过期。

### 注销接口

DELETE `/instance` 读出 IP 和端口，找到服务。服务已不存在就当作成功，避免调用方重试。存在则调用管理器的 `removeInstance`。

```java
@DeleteMapping
public String deregister(HttpServletRequest req) {
    Instance inst = readInstance(req);
    Service svc = services.lookup(ns(req), service(req));
    if (svc == null) {
        return "ok";
    }
    services.removeInstance(ns(req), service(req), inst.isEphemeral(), inst);
    return "ok";
}
```

客户端主动下线走的也是这个入口。巡检超时和进程正常退出在服务端汇合。

### removeInstance 写回一致性层

管理器对同一个服务对象加锁，算出「去掉目标之后还剩哪些实例」，再 `put` 到一致性服务。键里带有是否临时，所以不会把持久实例一起删掉。

```java
public void removeInstance(String ns, String serviceName, boolean ephemeral, Instance... gone) {
    Service svc = mustGet(ns, serviceName);
    synchronized (svc) {
        String key = instanceListKey(ns, serviceName, ephemeral);
        List<Instance> left = subtract(svc, ephemeral, gone);
        Instances wrapper = new Instances();
        wrapper.setInstanceList(left);
        consistency.put(key, wrapper);
    }
}
```

锁的范围是单个服务，不是整个注册表。不同服务的摘除可以并行。

### 从当前表里扣掉地址

`subtract` 最终进到更新地址的方法，动作是 `REMOVE`。步骤：

1. 读一致性层里当前的实例 datum。没有就用空表。
2. 用服务内存里的列表建立「IP:端口 → 实例」索引，并收集已有实例 ID。
3. 对要删除的每一台，从 map 里按 datum 键 `remove`。
4. 返回剩余列表，交给上一层写回。

动作若是新增而不是删除，则保留旧的实例 ID，没有旧记录时再生成新 ID。删除分支不会生成 ID。

```java
if ("REMOVE".equals(action)) {
    current.remove(inst.datumKey());
} else {
    Instance old = current.get(inst.datumKey());
    inst.setInstanceId(old == null ? newId(usedIds) : old.getInstanceId());
    current.put(inst.datumKey(), inst);
}
```

空列表在「新增」动作下会被拒绝，避免写出一个没有地址却声称添加成功的服务。删除到零台是允许的，表示这个临时服务此刻没有活着的进程。

写回一致性层之后，监听器会更新内存服务，并再次走到变更事件，UDP 推送把「少了一台」告诉订阅方。所以摘除不是静默的：健康位变化推一次，记录被删再推一次。

### 文字时序

以 `pay-ledger` 的 `10.8.4.12:9106` 为例。

```text
启动
  客户端登记 POST /instance
  同时每 5 秒 PUT /instance/beat
服务端
  beat 更新 lastBeat
  若从故障恢复，发布变更，UDP 通知订阅方
  订阅方回 ACK；10 秒无 ACK 则再发一次

进程崩溃
  15 秒无续约 -> healthy=false，推送一次
  30 秒无续约 -> DELETE /instance，名单去掉 9106，再推送
进程恢复
  下一次 beat 发现资源不在 -> 客户端重新 POST /instance
```

正常退出则不必等 30 秒：进程在关闭钩子里主动 DELETE，订阅方更快看到下线。

## 踩坑

- 只看客户端日志以为续约成功，但服务端负责该服务的分片不在你查看的那台机器上。先确认 Distro 责任节点，再看它的巡检日志。
- 把临时实例的删除超时改到 5 秒，而续约仍是 5 秒，网络一抖就会被删。超时应明显大于两到三个续约周期。
- 订阅方没有开 UDP 端口时，`clientMap` 为空，推送任务直接返回。调用方只能等本地缓存过期后再拉，故障切换会变慢。
- 轻量续约开始后，服务端若丢了内存，又收不到报文体，会返回资源不存在。客户端必须能处理这个码并重新注册。自己写的续约客户端若忽略该码，实例就再也回不来。
- 健康检查关闭时，15 秒和 30 秒的逻辑都不会跑。实例会一直保持注册时的健康位，直到有人手工删除。
- 守护线程不会挡住进程退出，但退出前仍应注销。否则要等满删除超时，调用方才会把流量移走。


## 订阅方收到 UDP 之后

服务端把名单推出去，只完成一半。调用方进程里还有一个接收口，负责把数据报变成本地缓存。没有这一段，服务端日志里即使写着 send udp，业务流量也不会立刻躲开坏实例。

### 接收口何时打开

调用方在向服务端拉实例列表时，可以附带自己的 UDP 端口。端口来自客户端 SDK 预先绑定的套接字，不是业务的 HTTP 端口 `9106`。服务端把这个地址放进 `clientMap` 之后，后续变更才会往这里推。

若配置或旧客户端把端口留成 0，服务端按「只查询、不推送」处理。调用方只能等本地缓存的有效期结束，再主动拉一次。故障切换的速度就等于缓存时间，而不是 UDP 的 1 秒级延迟。

### 收到包以后改哪块内存

接收线程解出服务名、实例列表和引用时间。本地按「分组 + 服务名」找到缓存项，整表替换，而不是按 IP 打补丁。替换之后，下一次负载均衡读到的就是新名单。

确认包要尽快回给服务端。内容里带上服务端随推送下来的引用时间，好让对端用它拼 `ackMap` 的键。迟迟不回，服务端会在大约 10 秒后重发同一份名单；连续两次都没回，这一轮推送就结束，本地仍保持旧名单，直到下次主动查询。

```java
void onPacket(DatagramPacket packet) {
    PushBody body = decode(packet);
    String key = body.getGroup() + "@@" + body.getService();
    cache.replaceAll(key, body.getHosts(), body.getRefTime());
    socket.send(ackOf(body.getRefTime(), packet.getSocketAddress()));
}
```

替换要原子。若一边在遍历旧列表选实例，一边在清空，可能读到半截名单。实现里通常整包替换引用，读的一方拿到的要么是旧表要么是新表。

### 和主动拉取怎样配合

推送是加速，拉取是兜底。缓存项上有一个有效期，过期后即使没收到 UDP，后台刷新任务也会再 GET 一次实例列表。所以 UDP 丢了不会永久用脏数据，只是多等一个周期。

周期不要调得比续约还短很多，否则每个调用方都会变成高频轮询，把服务端的访问日志再次写满。推送正常时，拉取只是保险。

### 调用方自己也要续约

订阅方如果同时是一个被别人调用的服务（例如 `checkout-api` 既调用 `pay-ledger`，自己也被网关调用），它身上有两条独立的客户端逻辑：

- 作为提供者：每 5 秒 PUT 续约，维持自己在名单里。
- 作为消费者：收 UDP 或拉列表，更新要访问的下游。

关掉其中一条，另一条仍在。只看「我还能调通下游」不能证明自己的续约正常。

## 用元数据改三个时间

5 秒、15 秒、30 秒是默认值，不是硬编码死在业务里。实例元数据里的保留键可以覆盖。Spring Cloud 侧通常写成发现配置，启动时写进注册资料：

```yaml
spring:
  cloud:
    nacos:
      discovery:
        heart-beat-interval: 4000
        heart-beat-timeout: 12000
        ip-delete-timeout: 24000
        server-addr: 10.8.4.10:8848
```

三个数的单位是毫秒。它们会变成实例上的保留元数据，服务端巡检读的是实例上的值，不是自己进程里的常量。所以同一集群里，`pay-ledger` 可以 4 秒续一次，另一个批处理服务可以 10 秒续一次。

约束关系：

- 续约间隔应明显小于不健康阈值，否则一次迟到就变不健康。
- 不健康阈值应小于删除阈值，中间留一段「还在名单里但不要被选中」的时间。
- 删除阈值至少覆盖三次续约加上网络抖动。上面 4 秒 / 12 秒 / 24 秒是一组能用的例子；默认 5 / 15 / 30 同理。

改了客户端却没重启，旧进程仍按旧元数据续约。控制台里点开实例，能看到实际生效的间隔。看不到新值就说明新配置没进这次注册。

服务端开关还可以整体关掉巡检。关掉之后，元数据里的超时不会再生效，实例会一直留着。这适合短暂排障，不适合当常态。

## 和 Eureka 租约对照着记

两边都是「客户端定期报平安，服务端过期就摘」。数字和名词不同，容易配错。

| 动作 | Eureka 常见默认 | Nacos 临时实例默认 |
| --- | --- | --- |
| 报平安间隔 | 30 秒 | 5 秒 |
| 多久没收到就摘除 | 90 秒 | 先 15 秒标不健康，30 秒删除 |
| 保护 | 短时间丢太多客户端时暂停摘除 | 不靠同一套自我保护；分片节点只检查自己负责的服务 |
| 调用方更新 | 约 30 秒拉一次注册表 | 缓存过期再拉，另加 UDP 推送 |

因此把 Eureka 的 30 秒心跳习惯直接套到 Nacos 上，会让实例看起来「很久才报一次」，巡检却按 15 秒判不健康，名单会抖。迁移时单独核对这三个毫秒值。

Eureka 的自我保护会在网络分区时留下可能已死的实例。Nacos 的临时实例更倾向于按超时删掉。分区恢复后，客户端的下一次续约若收到「资源不存在」，会重新注册，把实例补回来。这段空窗里调用方可能短暂看不到它，属于模型差异，不是登记丢了。

持久实例更接近「人工登记的地址」，不参与这套 30 秒删除。把一个会宕机重启、IP 不变的进程标成持久，崩溃后调用方仍会打到它，直到有人删记录。临时才是普通微服务该用的。

## 一次崩溃的时间表

下面用 `pay-ledger` 的 `10.8.4.12:9106` 走一遍，假设间隔都是默认值，订阅方 `checkout-api` 已带 UDP 端口。

```text
10:00:00  进程完成注册，lastBeat = 10:00:00，healthy = true
10:00:05  续约成功，lastBeat 拨到 10:00:05
10:00:07  进程被 kill -9，没有注销请求
10:00:10  本该发出的续约没有到达
10:00:20  巡检发现 idle > 15 秒，healthy 改为 false，UDP 通知 checkout-api
10:00:20  checkout-api 回 ACK，本地名单里这台不再被选中
10:00:37  idle > 30 秒，巡检对本机发 DELETE
10:00:37  名单去掉 9106，再次 UDP
10:01:05  进程被拉起，注册成功，又一次 UDP，流量回来
```

若 10:00:20 的 UDP 丢了，且 `checkout-api` 的缓存还要 8 秒才过期，则最坏还要再等这次拉取，期间可能继续把请求打到已经不健康的地址。连接失败应当触发调用方自己的重试或换实例，不能只依赖推送。

若进程是正常停机，关闭钩子在 10:00:07 就发出 DELETE，则不必等到 10:00:37。容器编排若给的退出时间短于注销超时，钩子没跑完就被强杀，表现会退回到上面的 kill -9 时间表。

## 集群里谁有权删

三台 Nacos 共用一个 MySQL 时，配置数据可以一起看见，但临时实例的巡检不是三台各做一遍。Distro 按服务名把责任分到一台。不负责的节点在任务开头就返回。

因此：

- 在非责任节点上找不到「auto delete」日志是正常的。
- 责任节点宕机后，分片会转移，另一台开始巡检。转移完成前，摘除会暂停一会儿。
- 客户端的 `server-addr` 若只写某一台，这台一停，续约全部失败，即使另外两台还在。前面集群笔记里把入口放在 Nginx，就是为了避免这个单点。

责任节点处理续约更新的是它内存里的 `lastBeat`。一致性层负责把实例列表同步给其他节点。所以「控制台是从另一台打开的，列表仍能看到实例」并不证明续约打到了你眼前这台。查续约失败要看客户端连上的那个节点，以及 Distro 责任节点，两份日志一起看。

## 日志里哪些词有用

客户端：

- 续约发送失败：调度线程名带 `beat.sender`，后面是服务名和端口。偶尔一条是抖动；连续超过删除阈值就要当故障。
- 收到资源不存在后重新注册：说明服务端曾经把实例删掉。若每分钟都发生，多半是超时设得太紧，或续约打到了不负责、也不转发的节点。

服务端：

- 补偿注册：续约带了报文体，但内存里没有实例。偶发于服务端重启；持续出现说明有人在删，或有两个集群写同一批客户端。
- 推送失败或重试：看 `ackMap` 是否长期不下降。长期不下降表示订阅方的 UDP 口没开、被安全组挡住，或确认键对不上。
- 自动删除：带 IP 和端口。对照客户端是否真的停了。客户端还在却被删，就是时间参数或网络问题，不是业务代码抛异常。

访问日志里的 `/instance/beat` 行数，大约等于临时实例数乘以每分钟续约次数。100 个实例、5 秒一次，就是每分钟约 1200 次，再加拉取列表。它很大不一定是故障。真正要报警的是磁盘，而不是行数本身。行数突然掉到接近 0，反而是续约停了。

## 自测时按这个顺序

1. 只启动一个 `pay-ledger`，控制台出现 `10.8.4.12:9106`，健康为真。
2. 抓 10 秒客户端日志，应看到约两次续约，而不是一次注册后再无下文。
3. `kill -9` 之后先等 20 秒，健康应变假，记录还在；再等过 30 秒，记录消失。
4. 进程重新拉起，不手工点控制台，实例应自己回来。回不来就查续约响应里的业务码有没有被客户端忽略。
5. 让 `checkout-api` 在循环里调用。第 3 步健康变假之后，新请求不应再打到 9106。若仍打到，先确认订阅是否带了 UDP 端口，再看本地缓存还有多久过期。
6. 把续约间隔改成 4 秒并重启提供者，控制台元数据里应出现新的间隔。巡检仍用 15 秒的话，容错余量变大，这是预期，不是没生效。

六步里任何一步要靠重启 Nacos 才能恢复，都说明客户端没有处理「资源不存在」或注销路径没走通。先修客户端，再调超时数字。


## 客户端怎样选一台 Nacos 来续约

`server-addr` 可以写多台，用逗号隔开。SDK 不会把每一次续约都广播给全部节点，而是挑一台发 PUT。这一台失败，再换列表里的下一台。所以三台里宕一台，续约仍能成功，前提是列表里不只有那一台。

```java
String beatOnce(BeatInfo info) {
    List<String> nodes = serverList(); // 例如 10.8.4.10:8848,10.8.4.11:8848
    RuntimeException last = null;
    for (String node : nodes) {
        try {
            return put(node + "/nacos/v1/ns/instance/beat", info);
        } catch (RuntimeException ex) {
            last = ex;
        }
    }
    throw last;
}
```

挑选通常带一点随机，避免所有客户端都打在列表的第一台。Nginx 已经做了转发时，`server-addr` 只写 Nginx 那一个入口也行，换节点的责任在 Nginx。两种不要叠成「客户端列表里每台又各写一个只会打到单机的地址」，否则你以为有三台，实际三次都进了同一台。

注册 POST 和续约 PUT 走同一套选点。可能出现「注册打到 A、续约打到 B」。临时实例的内存以责任节点为准，其他节点靠同步看到它。刚注册的一两秒内在另一台控制台上看不见，不一定是失败。超过几个续约周期仍只有一台能看见，才是同步或 `cluster.conf` 没配对。

## 停机时要停掉调度，不能只断流量

进程收到退出信号时，正确顺序是：

1. 把 `BeatInfo` 标成停止，让 `finally` 里不要再预约下一次。
2. 向命名服务发注销，而不是等 30 秒被巡检删掉。
3. 再关 HTTP 端口，避免注销尚未完成时又有新流量进来。

只关端口、不标停止，调度线程仍会 PUT。服务端会认为实例还活着，甚至在你已经注销之后又因为带报文体的续约把实例补偿注册回来。表现是「容器已经没了，控制台里的 IP 还在，而且健康」。

```java
public void shutdown(BeatInfo info) {
    info.setStopped(true);
    proxy.deregister(info.getServiceName(), info.getIp(), info.getPort());
}
```

Spring 容器的销毁回调会走到登记器的 `deregister`。自己用 `Runtime.halt` 或编排器直接 SIGKILL，回调不会执行，只能靠超时删除。给容器的 `terminationGracePeriod` 要大于注销请求的超时，否则和第 3 步对不上。

注销请求本身是 DELETE，参数和第 2 节巡检拼出来的一样：IP、端口、集群、服务名、是否临时。分组已经包含在服务名的 `TRADE@@pay-ledger` 这种形式里时，不要再额外拼一次分组，否则服务端会找不到服务，返回成功的空操作，实例却还在。

## 轻量续约的两轮报文

第一轮客户端还不知道服务端是否允许省略报文体，于是带上完整 JSON。服务端内存里若已有 `10.8.4.12:9106`，就只更新 `lastBeat`，响应里 `lightBeatEnabled=true`，并给出下一次间隔。

```text
PUT /nacos/v1/ns/instance/beat
    ?serviceName=TRADE@@pay-ledger
    &ip=10.8.4.12&port=9106&clusterName=DEFAULT
body: beat={...ip,port,weight,metadata...}

200
{ "clientBeatInterval": 5000, "code": 10200, "lightBeatEnabled": true }
```

第二轮起，反应器记下开关，报文体不再带 `beat`。服务端靠查询参数找到同一台实例。找不到时不能做补偿，因为没有档案可还原，于是返回资源不存在。客户端收到后立刻走注册 POST，下一轮再恢复成完整报文或继续轻量，取决于新的响应。

把轻量开关缓存在进程里是对的，但服务端重启后内存空了，而客户端仍按轻量发送，就会连续几次「资源不存在 → 重新注册」。这是预期中的自愈，日志里允许出现一小段，不应把它当成死循环去把续约线程杀掉。

业务码不要写死在各处。用 SDK 常量判断即可。不同小版本的数字若有调整，自己解析 JSON 时只认 `code` 字段的名字，并和常量比较。

## 开关：巡检、过期删除、推送

服务端有一组运行时开关，不必改代码：

- 健康检查总开关。关掉后，15 秒与 30 秒两段都不会改实例。适合对照「是不是巡检误删」。
- 是否允许过期删除。关掉时仍会把健康位置假，但记录留着。调用方若只过滤不健康实例，效果和删除接近；若有人忽略健康位，流量仍会打到坏地址。
- 是否允许轻量续约。关掉后客户端即使想省略报文体，响应也会要求下次继续带上。排障时打开完整报文更容易看到权重和元数据。
- 推送缓存时间。它既影响服务端判定僵尸客户端的空闲上限，也写进推送给调用方的 `cacheMillis`，告诉对方本地名单可以信多久。

```java
class PushSwitches {
    boolean healthCheck = true;
    boolean expireInstance = true;
    boolean lightBeat = true;
    long pushCacheMs = 10_000L;
}
```

`cacheMillis` 不是续约间隔。调用方过了这段时间还没收到新的 UDP，就应主动拉列表。把它调成 60 秒，故障切换会变慢；调成 1 秒，拉取会非常密。10 秒量级是推送和拉取之间的折中。

这些开关改完对已经在跑的任务生效，不必重启全部业务进程。但客户端已经缓存的轻量开关要等下一次响应才更新。

## 健康位、标记位、权重

三个字段常被看成一个「能不能调」：

- `healthy`：巡检和续约在改。假表示最近没有按时报平安。
- `marked`：人工或外部流程打的标。为真时，巡检即使发现超时也不要自动改健康位、也不要自动删，避免和变更单对着做。
- `weight`：负载均衡用的比重。0 表示还在名单里，但新请求不要再分给它。适合发布前先摘流量，进程仍然续约，记录不会在 30 秒后消失。

发布一台新版本时，可以先把旧实例权重改成 0，等在途请求结束，再停进程。若直接停进程，要忍受最多 15 秒的不健康窗口，期间是否还被选中取决于调用方听不听健康位。权重为 0 更可控。

标记位不要拿来做日常发布。它会让自动删除失效，实例会变成永远留在控制台里的脏数据，直到有人记得取消标记。

## 推送体里调用方真正用到的字段

服务端组包时放的是「当前这份名单」，不是增量。调用方收到后整体替换。包里除了 IP、端口、健康、权重，还有：

- 服务名，用来找到本地缓存槽。
- 引用时间，用来做确认和判断这包是不是过期的重传。
- `cacheMillis`，本地在此之后若没有新包，就自己拉。

重传的是同一份快照。若这 10 秒内名单又变了，新的事件会再排一次推送，不依赖重传把新数据带上。所以重传解决的是丢包，不是把多次变更合并。合并靠的是 `futureMap`：同一服务 1 秒内只保留一个待执行的推送任务，任务执行时再读最新名单。

```text
09:00:00.000  实例 A 下线，预约 1 秒后推送
09:00:00.400  实例 B 上线，发现已有预约，不再另起任务
09:00:01.000  任务执行，包里同时没有 A、有 B
```

若两次变更间隔超过这个合并窗口，就会推两次。调用方按引用时间丢掉更旧的包，避免后到的重传把新名单覆盖掉。比较的是包内时间，不是本机收到的时刻。

## 最小观测

不必上完整监控也能判断这条链是否健康。每台业务机看三个数，每台 Nacos 看三个数。

业务机：

- 最近一次续约的时间。超过两个间隔还没有成功，先查网络和 `server-addr`。
- 续约失败计数。偶发可以忽略，斜率突然变陡就要告警。
- 本地下游缓存的年龄。长期接近 `cacheMillis` 上限，说明 UDP 基本没在更新。

Nacos：

- 某服务的健康实例数。发布期间允许短暂下降，平时不应抖动。
- 待确认推送包的数量。持续上涨说明确认回不来。
- 自动删除次数。白天业务没有发布却频繁删除，就是续约或阈值问题。

把这三个数和访问日志的体积分开看。日志大，可能只是实例多；删除次数高，才是可用性事件。

## 和配置中心的长轮询不是同一条日志

Nacos 客户端还有配置长轮询，默认也会周期性打服务端，日志器在 `config` 而不是 `naming`。它不更新 `lastBeat`，也不受 15 秒健康检查约束。配置拉不到时，服务仍可能是健康的；续约失败时，配置仍可能是新的。

排障时先看线程名和记录器：

- 带 `beat.sender` 或 naming：实例是否还在名单里。
- 带 config 的 long-polling：远程配置有没有刷新。

把配置客户端的级别调高，不会让续约变安静；反过来也一样。磁盘被访问日志占满时，两条 HTTP 都会出现在 `access_log` 里，但处理手段是滚动和保留天数，不是把业务续约停掉。

## 读源码时建议的断点

不想一次跟完全部异步任务，可以只在这些点暂停，每次只看一个方向：

1. 客户端 `registerInstance`：确认临时实例先入调度、再发注册。
2. `BeatTask.run` 的 `finally`：确认失败也会再次预约。
3. 服务端 `beat`：分清「无实例且无报文体」与「无实例但有报文体」。
4. `ClientBeatProcessor`：只有健康位从假变真才推送。
5. `ClientBeatCheckTask`：先改健康，再按更长的阈值删除。
6. `udpPush`：重试计数超过 1 就放弃。

六处对上之后，中间的线程池只是搬运。跳来跳去感到乱，是因为注册、续约、推送、巡检各有自己的调度器，不是因为名单有两份互相矛盾的事实来源。内存里的实例表是事实，UDP 和查询接口都是它的投影。


## 一次发布里这些机制怎样叠在一起

把 `pay-ledger` 从旧进程切到新进程，续约、健康位和推送会叠在同一分钟里。按下面的顺序做，调用方几乎不用等到 30 秒删除。

1. 新进程 `10.8.4.13:9106` 先启动并注册，权重先保持 1。此时名单里新旧两台都健康，流量开始分到新进程。
2. 看新进程的续约是否连续成功两次。没有续约就不要动旧进程。
3. 把旧进程权重改成 0。它仍每 5 秒续约，所以不会被判死，只是新请求不再选它。
4. 等在途请求结束，再发正常停机，让注销 DELETE 马上删掉旧记录，并触发一次 UDP。
5. 订阅方本地名单应只剩 `10.8.4.13`。若还看到旧 IP，先等一个 `cacheMillis`，仍然看到再查推送确认有没有回来。

权重改成 0 之后旧进程如果被误杀，15 秒内会变成不健康，30 秒后记录消失。因为权重已经是 0，这段窗口里本来就不会接新流量。这是双保险，不是必须靠超时来摘流。

回滚时把旧包重新拉起即可。只要客户端仍会在「资源不存在」时重新注册，就不必手工在控制台补实例。回滚后的第一笔流量若打到已经删掉的地址，说明订阅方缓存还没换，应查 UDP 端口和安全组，而不是再把删除阈值调短。

## 不要和配置刷新共用同一套超时想象

配置长轮询的超时常常是「卡住大约 30 秒再返回」，数字上和实例删除超时一样，含义完全不同。长轮询 30 秒是一次 HTTP 的等待上限，到点返回「没有新配置」。实例删除 30 秒是「从最后一次续约起算的生存期」。

所以看到日志里两处都出现 30000，不要合成一个开关。改配置轮询时间不会让崩溃的实例更快消失；改 `ip-delete-timeout` 也不会让配置刷新变勤。两个客户端甚至可以连不同的 Nacos 集群：发现写 `discovery.server-addr`，配置写 `config.server-addr`。续约只会打发现地址。

## 安全组和连接被重置时的表象

UDP 确认被防火墙丢掉时，HTTP 续约仍然 200。控制台里实例一直健康，但调用方迟迟不更新。这时：

- 服务端待确认包变多，重试一次后放弃。
- 调用方缓存年龄顶在 `cacheMillis`。
- 主动拉取仍能成功，所以最终会一致，只是慢。

TCP 续约被重置时相反：UDP 可能还通，但 `lastBeat` 不再前进，15 秒后健康变假。调用方若只靠推送，会很快停止选择这台；若推送也断了，就要等本地缓存过期后的那次拉取。两条路同时断，才会把流量继续打向坏地址直到缓存失效。监控上应同时看续约成功率和推送确认，而不是只看进程还在不在。

端口记得分开：8848（或 Nginx 上的 8088）是 HTTP；推送用的 UDP 端口在调用方机器上，由 SDK 临时绑定，不是 8848。安全组若只放行 8848，续约正常、推送全灭，就是这个原因。

## 和本文前半的对应关系

前半按调用方向写了登记、续约、服务端更新、UDP、巡检删除。这一段只是把同一套状态机放进发布和故障里看。记住四件事实就够排障：

- 临时实例靠周期 PUT 活着，默认约 5 秒一次。
- 大约 15 秒无续约改为不健康，大约 30 秒删除；都可以用元数据改。
- 删除和健康变化通过 UDP 尽快通知订阅方，确认失败只重试有限次。
- 订阅方还有按 `cacheMillis` 的主动拉取，推送不是唯一的一致途径。

持久实例、配置轮询、访问日志滚动都不改变这四句。它们只是容易被同一台 Nacos 的日志卷在一起，查的时候要拆开。


## 本地 beat 表的键不要和注册名搞混

调度器里的 map 用「带分组的服务名 + IP + 端口」做键，例如 `TRADE@@pay-ledger#10.8.4.12#9106`。同一进程如果因为配置刷新又注册了一次，键相同，旧的 `BeatInfo` 会被标停止，新的接上。键若少了分组，会和另一个分组里碰巧同名的服务共用一条任务，续约打到错误的服务上，控制台里一边多实例、一边空着。

端口以注册当时绑定的端口为准。后面用 `server.port` 改了配置但没有重启，任务仍向旧端口续约，服务端更新的是旧记录，新端口上的进程没有心跳，看起来像起了两套。改端口就要走完注销再注册，不要只改配置文件。

IP 同样参与键。容器重建后 IP 变了，旧键不会自动消失，要靠旧进程的注销或服务端超时把旧 IP 删掉。新 IP 是一条新键。滚动发布时短时间内两台 IP 并存是正常的，不正常的是旧 IP 在进程已经没了之后还继续健康超过删除阈值。

## 服务端列表会变，任务不会因此停

Nacos 集群扩容后，客户端手里的服务器列表可能被刷新。已经在跑的 `BeatTask` 不需要重建，下一次 `sendBeat` 会读最新列表。若刷新失败，继续用旧列表，直到旧地址全部连不上才连续报错。因此扩容不必重启全部业务进程；缩容时要把下线节点从列表或 Nginx 上游拿掉，否则客户端仍会把一部分续约打到已关闭的端口，失败日志会周期性出现，但只要还剩一台可写，实例就不会被删。

列表为空时无法续约，也没有「本地假装成功」。这种情况要让就绪探针失败，避免无注册中心的实例继续接流量。探针可以看最近一次续约是否在两个间隔之内，而不是只看进程端口开着。

## 读代码时容易看反的两处

第一处：`schedule` 的延迟是上一次响应里的间隔，不是固定 5 秒的 `scheduleAtFixedRate`。服务端把间隔改成 8 秒后，客户端下一次才变，并且任务对象是新的，旧对象不会被线程池周期触发。在调试器里对旧对象下断点，会以为续约停了。

第二处：巡检删除走的是对本机的 HTTP DELETE，不是在巡检线程里直接 `map.remove`。断点若只打在内存集合的 remove 上，会看到调用栈来自控制器，而不是来自 `ClientBeatCheckTask`。任务只负责发现超时并发起请求。这样写是为了让手动下线和自动下线共用一致性写入。跟着 HTTP 进 `removeInstance`，才能看到剩余列表被放回一致性层。

两处都对上之后，再去看 UDP 重试就不会把「没进断点」当成「没执行」。异步任务的断点要打在新对象的 `run` 上，或者打在永远存在的接收循环里。

再补一条和数字有关的核对：续约间隔、不健康阈值、删除阈值必须递增。一旦删除阈值小于续约间隔，健康的进程也会被周期性删掉又注册回来，控制台里实例数来回跳，UDP 也会跟着刷。先在实例详情里看这三个毫秒值，再决定是网络问题还是参数写反。集群入口应指向全部节点或它们前面的转发层，单机地址只适合本机试验。试验结束后把阈值改回大于两个续约周期，避免把调试参数留在预发环境里影响其他服务的注册与摘除。确认控制台里的间隔已回到常规值后，再结束这次排查。
