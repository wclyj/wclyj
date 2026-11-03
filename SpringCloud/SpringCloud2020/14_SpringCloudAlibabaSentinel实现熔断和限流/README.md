# SpringCloudAlibabaSentinel实现熔断和限流

## Sentinel

### 官网

Sentinel 是阿里开源的流量防护组件，定位和 Hystrix 接近，但控制方式不同。Hystrix 的规则大多写在注解属性里，监控面板还要自己搭。Sentinel 把规则挪到独立控制台：限流、熔断、热点、系统保护都可以在页面上改，改完对已接入的实例生效。

仍建议优先用注解和配置表达规则，少在业务里手写判断。约定能覆盖的不要硬编码。

### 是什么

服务拆开之后，稳定性取决于“流量怎么进来、失败怎么停下来”。Sentinel 从流量切入，提供限流、熔断降级和系统负载保护。

几个常被提到的特点，换成自己的话：

- 场景覆盖突发流量，例如秒杀时把瞬时 QPS 按住、把消息消费速度拉平、按集群维度限流，以及下游已经不可用时尽快熔断
- 控制台能看单机的秒级曲线；集群规模不大时也可以做汇总
- 适配模块覆盖 Spring Cloud、Dubbo、gRPC，引入依赖再配地址就能接入
- 规则存储、数据源都留了 SPI，可以换成自己的配置中心

### 主要特征

从控制台看，能力分成几块：流量控制、熔断降级、热点参数、系统自适应保护、授权规则。实时监控和规则管理是另外两条线，不写在业务代码里。

### 生态圈

常见搭配是：应用进程里放核心库，控制台单独部署，规则数据源接 Nacos、Apollo 或 ZooKeeper。网关侧也有适配，可以把 Sentinel 挂在 Gateway 或 Zuul 上。下面的例子走 Spring Cloud 应用加 Nacos。

### 下载

控制台是一个可执行的 Spring Boot 包，从发行页取与核心库匹配的版本。网络不好时可以拉源码本地打包，产物名里带 dashboard。不要用和客户端差主版本的控制台，否则心跳或规则协议对不上。

## 安装Sentinel控制台

整套分成两部分：

- 核心库跑在业务进程里，不绑定 Web 容器，Dubbo 和 Spring Cloud 都有适配
- 控制台是独立的 Spring Boot 程序，`java -jar` 就能起，不必再套 Tomcat

默认端口 8080。本实验改到 18080，避免和别的管理端冲突：

```text
java -jar sentinel-dashboard.jar --server.port=18080
```

登录后是空的。实例列表要等客户端第一次上报才出现。

## 初始化演示工程

先确认 Nacos 已在 8858 端口就绪。

### 引入依赖

```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
</dependency>
<dependency>
    <groupId>com.alibaba.csp</groupId>
    <artifactId>sentinel-datasource-nacos</artifactId>
</dependency>
```

第二个依赖这次先加上，持久化一节才会用到。

### 修改YML

业务端口用 8471。`transport.port` 是客户端和自己通信的端口，不是控制台端口，默认从 8719 起找空闲值。这里固定成 8727。

```yaml
server:
  port: 8471
spring:
  application:
    name: guard-demo
  cloud:
    nacos:
      discovery:
        server-addr: 127.0.0.1:8858
    sentinel:
      transport:
        dashboard: 127.0.0.1:18080
        port: 8727
```

### 增加业务类

```java
@RestController
@Slf4j
public class ProbeController {

    @GetMapping("/probe/alpha")
    public String alpha() {
        return "alpha-ok";
    }

    @GetMapping("/probe/beta")
    public String beta() {
        log.info("{} beta", Thread.currentThread().getName());
        return "beta-ok";
    }
}
```

启动 8471 后刷新控制台，簇点链路仍是空的。Sentinel 对 Web 资源默认懒加载，先各访问一次 `/probe/alpha` 和 `/probe/beta`，资源名才会出现。资源名默认就是请求路径。

## 流控规则

### 基本介绍

新增流控规则时这几个字段要分清：

- 资源名：唯一标识，HTTP 接口默认用路径
- 针对来源：可以只限制某个调用方的服务名，留空或 default 表示不区分来源
- 阈值类型：QPS 是每秒通过数；线程数是同时占用的并发
- 是否集群：单机实验保持关闭
- 流控模式：直接、关联、链路
- 流控效果：快速失败、预热、排队等待

### 流控模式

#### 直接（默认）

给 `/probe/alpha` 配一条直接规则，阈值类型 QPS，单机阈值 1，效果选快速失败。每秒点超过一次，多出来的请求立刻失败，响应体是框架默认的限流提示。

线程数模式的含义不同：阈值 1 表示同一时刻只允许一个线程停在这个资源里。给接口加一秒睡眠后再用浏览器连点，第二个请求不会排队，而是直接被拒绝。

```java
@GetMapping("/probe/delta")
public String delta() throws InterruptedException {
    TimeUnit.SECONDS.sleep(1);
    return "delta-ok";
}
```

直接模式的默认文案对前端不友好。后面用 `@SentinelResource` 的 `blockHandler` 换成自己的返回。

#### 关联

关联模式限制的是“自己”，触发条件看的是“别人”。当关联资源的指标越过阈值，就拒绝当前资源。典型场景：支付接口已经很忙时，暂时关掉下单入口，避免订单堆在支付前面。

实验：资源名填 `/probe/alpha`，模式选关联，关联资源填 `/probe/beta`，QPS 阈值 1。用压测工具对 beta 维持每秒多次调用，再访问 alpha，alpha 会被拒绝，即使 alpha 自己几乎没有流量。停止压 beta 之后，alpha 恢复。

#### 链路

链路模式只统计从某个入口进来的流量，而不是把一个资源的全部调用混在一起算。

例如 `quote()` 同时被“下单入口”和“询价入口”调用。只想限制下单这条链时，把入口资源设成下单的那个上下文，询价入口即使很热也不会把下单误伤。使用前要确认 Web 适配把链路上下文分开了：同一 URL 下如果上下文被合并，链路规则会看起来像没生效。Spring Cloud 适配里通常还要打开 `web-context-unify: false`，否则不同入口会被收成同一个上下文。

```yaml
spring:
  cloud:
    sentinel:
      web-context-unify: false
```

### 流控效果

#### 直接

快速失败是默认效果。超出阈值立刻抛 `FlowException`，调用方看到的默认句子类似 `Blocked by Sentinel (flow limiting)`。适合必须马上拒绝、不能多占线程的接口。

#### 预热

系统最怕从接近空闲一下子拉到峰值。预热让通过量从“阈值 / 冷启动因子”慢慢爬到阈值。冷启动因子默认是 3。

阈值设 9、预热时长 5 秒时，起始通过量大约是 9 / 3 = 3，五秒内再升到 9。秒杀开场就可以用这个效果，避免冷缓存和冷连接被第一秒打穿。阈值类型必须是 QPS，线程数没有“慢慢放量”的含义。

#### 排队等待

排队对应漏桶：请求按固定间隔通过，多出来的在超时时间内等待，超时仍未轮到就拒绝。阈值类型同样必须是 QPS。

例如 `/probe/alpha` 每秒放行 1 个，超时 20000 毫秒。突发的十个请求不会在第一秒全部失败，而是大约每秒处理一个，二十秒内还没排到的才丢弃。适合消息拉取、报表导出这类可以匀速消化的入口，不适合用户在线等待的短接口。

## 降级规则

### 名词介绍

三种策略：

- 平均响应时间（RT）：秒级统计。窗口内通过的请求数达到最小样本（常见是 5），且平均 RT 超过阈值，就打开熔断。时间窗口结束后再关闭。统计时 RT 有上限，默认超过 4900 毫秒按 4900 计。要放宽这个上限，用启动参数 `-Dcsp.sentinel.statistic.max.rt` 指定更大的毫秒数
- 异常比例：秒级。QPS 达到最小样本且异常占比超过阈值（0 到 1）时熔断，窗口结束再关闭
- 异常数：按分钟累计。一分钟内异常次数超过阈值就熔断。因为统计周期是一分钟，时间窗口如果小于 60 秒，窗口刚结束可能马上又被剩余异常数再次打开

### 概念

某个资源持续超时或持续抛错时，熔断会在后续时间窗口内直接拒绝对它的调用，避免线程堵在这个资源上再去拖垮别的接口。默认行为是抛 `DegradeException`。

早期 Sentinel 的断路器没有半开：窗口一到就关闭，再靠新的统计决定要不要再次打开。Hystrix 则是半开时先放一个探测请求。读规则时不要把两边的状态机当成同一个。较新的 Sentinel 版本补了慢调用比例等策略，并引入探测恢复，但本笔记的实验仍按“窗口结束即关闭”来理解控制台上的三个旧策略。

### 降级策略实战

#### RT

平均响应时间策略：1 秒内持续进来的请求数达到 N，且这些请求的平均耗时超过阈值（毫秒），接下来的 `timeWindow` 秒内调用直接熔断。

```java
@GetMapping("/probe/slow")
public String slow() throws InterruptedException {
    TimeUnit.MILLISECONDS.sleep(800);
    log.info("slow returned");
    return "slow-ok";
}
```

规则：RT 阈值 200 毫秒，时间窗口 1 秒，最小请求数按默认 5。压测工具每秒打 10 次。方法要睡 800 毫秒，平均 RT 明显高于 200，样本也超过 5，于是下一秒内再访问会得到熔断提示，而不是慢慢返回 `slow-ok`。停掉压测、等窗口过去，接口恢复。

#### 异常比例

每秒请求量达到最小样本，且异常次数占通过量的比例超过阈值，就进入降级。阈值 0.5 表示一半。

让接口在一半情况下抛错，例如按查询参数决定是否除零。单独点一次只会看到业务异常。用压测每秒 10 次打满，比例超过 0.5 之后，后续请求不再进方法体，而是直接降级。时间窗口设 3 秒时，大约 3 秒后才会重新允许真实调用。

#### 异常数

近一分钟异常次数超过阈值即熔断。窗口建议大于等于 60 秒，否则刚恢复又会因为这一分钟的计数还未清掉而再次熔断。

```java
@GetMapping("/probe/boom")
public String boom() {
    int x = 10 / 0;
    return "boom-" + x;
}
```

阈值设 5、时间窗口 60 秒。前几次访问能看到除零异常；累计到第 5 次之后，再访问得到的是熔断，而不是新的异常栈。这能把“已经确定会失败的方法”从调用链上摘掉。

## Sentinel热点规则

### 什么是热点数据

热点就是被频繁带上的参数值。希望限制的不是整个接口，而是其中最热的那几个参数，例如：

- 以商品编号为参数，只限制一段时间内被下单最多的那几个编号
- 以用户编号为参数，只限制狂刷接口的那几个用户

统计用 LRU 保留最近最常出现的参数，再用令牌桶按参数限流。可以看成一种只对“带了热点参数的调用”生效的流控，并支持集群维度。参数必须是基本类型或字符串，自定义对象不会参与统计。

### 兜底的方法

不写处理方法时，热点限流和普通限流一样返回框架默认英文提示。要换成自己的文案，用 `@SentinelResource` 指定 `blockHandler`。这和 Hystrix 的 `fallbackMethod` 类似，但职责更窄：`blockHandler` 只接规则拦截，不接业务异常。

### 配置

`value` 是资源名，热点规则要配在这个名字上，而不是配在 URL 上。方法被注解标记后，URL 路径和资源名可以不同。

```java
@GetMapping("/probe/hot")
@SentinelResource(value = "coverHot", blockHandler = "onHotBlocked")
public String coverHot(@RequestParam(value = "sku", required = false) String sku,
                       @RequestParam(value = "wh", required = false) String wh) {
    return "hot-" + sku;
}

public String onHotBlocked(String sku, String wh, BlockException ex) {
    return "hot-key-blocked";
}
```

控制台新增热点规则：资源名 `coverHot`，参数索引 0，单机阈值 1，统计窗口 1 秒。索引 0 就是 `sku`。

连续请求 `sku=red` 会在超过每秒 1 次后返回 `hot-key-blocked`。只带 `wh=east`、不带 `sku`，不受这条规则约束。限流看的是被选中的参数，不是整个 URL。

### 参数例外项

普通情况：参数 0 的 QPS 超过 1 就拦截。

例外：当 `sku` 等于某个特定值时，单独给它更高的阈值。例如平时阈值为 1，但 `sku=vip` 的阈值是 200。控制台的例外项里参数值填 `vip`，例外阈值填 200。

一直刷新 `sku=vip` 不会进 `onHotBlocked`；换成别的值立刻又受 1 QPS 的约束。适合给白名单商品或内部账号放宽。

### 结语

`@SentinelResource` 配合 `blockHandler`，处理的是控制台规则触发的 `BlockException`。方法里写 `int n = 10 / 0` 抛出的是运行时异常，不会进 `blockHandler`。

业务异常要另外指定 `fallback`，它接收 `Throwable`：

```java
@SentinelResource(value = "coverHot",
        blockHandler = "onHotBlocked",
        fallback = "onHotFailed")
public String coverHot(@RequestParam(required = false) String sku,
                       @RequestParam(required = false) String wh) {
    return "hot-" + sku;
}
```

规则拦截优先走 `blockHandler`；方法自己抛错才走 `fallback`。两个都配时，被限流不会误进 fallback。

## Sentinel系统配置

系统规则不针对单个资源，而针对这台机器的入口流量（进入应用的入站调用，例如 Web 请求或 Dubbo 提供方）。它参考负载、CPU、平均 RT、入口 QPS 和并发线程，让入口流量和机器承受力保持平衡。

五种模式：

- Load：仅类 Unix 系统有效。以 load1 为启发值，并且当前并发已经超过按 `maxQps * minRt` 估算的容量时才触发。参考阈值常取 CPU 核数乘 2.5
- CPU 使用率：1.5.0 之后可用，超过 0 到 1 之间的比例就保护，反应比 Load 更快
- 平均 RT：全部入口流量的平均耗时达到毫秒阈值就保护
- 并发线程数：入口上同时占用的线程达到阈值就保护
- 入口 QPS：入口每秒请求数达到阈值就保护

把它理解成整站的总闸。单个接口的 QPS 规则仍然要单独配，系统规则只保证机器整体不被打满。

## @SentinelResource注解

两种限流粒度：

- 按注解里的资源名
- 按 URL（适配器自动埋点，不一定需要注解）

### 问题

只用默认提示，文案不符合业务。把兜底方法写在每个控制器旁边，又和业务耦在一起，方法数量一多就膨胀。另外，控制台里手点的规则存在内存中，8471 一重启就没了，看不到持久化。

### 客户自定义限流处理逻辑

把处理方法抽到独立类，并写成静态方法。`blockHandlerClass` 指向这个类。

```java
public final class GuardReplies {
    private GuardReplies() {
    }

    public static ApiEnvelope<Void> first(BlockException ex) {
        return ApiEnvelope.fail(4601, "guard-reply-1");
    }

    public static ApiEnvelope<Void> second(BlockException ex) {
        return ApiEnvelope.fail(4602, "guard-reply-2");
    }
}
```

```java
@GetMapping("/quota/custom")
@SentinelResource(value = "quotaCustom",
        blockHandlerClass = GuardReplies.class,
        blockHandler = "second")
public ApiEnvelope<LedgerRow> quotaCustom() {
    return ApiEnvelope.ok(new LedgerRow(88L, "row-88"));
}
```

资源名配 `quotaCustom`。触发后返回 `guard-reply-2`，控制器里不必再写一份相似的方法。

### 更多注解属性说明

核心编程 API 有三个：

- `SphU`：定义资源，进入和退出成对出现
- `Tracer`：把业务异常记入统计，否则熔断只看框架捕获到的异常
- `ContextUtil`：打开或切换上下文，链路限流依赖它区分入口

手写时用 try / catch / finally，在 finally 里 `exit`，否则统计会漏。注解方式已经把这套调用包起来了，只有要在非 Web 入口埋点时才直接用 API。

`exceptionsToIgnore` 用来声明哪些异常不计入熔断、也不进 fallback，例如参数校验失败不应把断路器打开。

## 服务熔断

这一节把 Sentinel 和远程调用放在一起：提供方是 9303、9304，调用方是 7084，调用方式可以是 `RestTemplate`，也可以换成 OpenFeign。

### 不设置任何参数

只写资源名、不写 fallback 和 blockHandler 时，下游异常会变成错误页或原始异常，调用方体验很差。

```java
@RequestMapping("/caller/ledger/{id}")
@SentinelResource(value = "ledgerFallback")
public ApiEnvelope<LedgerRow> ledger(@PathVariable Long id) {
    ApiEnvelope<LedgerRow> body = restTemplate.getForObject(
            SERVICE + "/ledger/sql/" + id, ApiEnvelope.class);
    if (id == 4) {
        throw new IllegalArgumentException("bad id");
    }
    if (body == null || body.getData() == null) {
        throw new NullPointerException("no row");
    }
    return body;
}
```

`SERVICE` 指向注册中心里的 `wallet-api`。

### 设置fallback

`fallback` 只接业务异常，例如上面的非法参数和空数据。限流抛出的 `BlockException` 仍走默认提示。

```java
@SentinelResource(value = "ledgerFallback", fallback = "whenLedgerFails")
public ApiEnvelope<LedgerRow> ledger(@PathVariable Long id) {
    // 调用与分支与上一例相同
    return restTemplate.getForObject(SERVICE + "/ledger/sql/" + id, ApiEnvelope.class);
}

public ApiEnvelope<LedgerRow> whenLedgerFails(Long id, Throwable ex) {
    return ApiEnvelope.fail(444, "ledger fallback: " + ex.getMessage(),
            new LedgerRow(id, "none"));
}
```

fallback 方法最后多一个 `Throwable`。

### 设置blockHandler

`blockHandler` 只接控制台规则触发的拦截，业务异常仍会抛出去，除非同时配了 fallback。

```java
public ApiEnvelope<LedgerRow> whenLedgerBlocked(Long id, BlockException ex) {
    return ApiEnvelope.fail(445, "ledger blocked", new LedgerRow(id, "none"));
}
```

注解上 `blockHandler = "whenLedgerBlocked"`。

### blockHandler和fallback一起配置

两个都写上时，规则拦截只进 `blockHandler`，业务抛错才进 `fallback`。不要指望限流被 fallback 接住。

### 异常忽略

有些异常是正常的业务分支，例如“余额不足”。把它放进 `exceptionsToIgnore`，既不触发 fallback，也不计入异常比例，避免一次正常的业务拒绝把熔断打开。

## Feign系列

#### 引入依赖

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```

#### 修改YML

调用方监听 7084，仍把控制台指到 18080。关键是打开 Feign 的 Sentinel 适配。

```yaml
server:
  port: 7084
spring:
  application:
    name: wallet-caller
  cloud:
    nacos:
      discovery:
        server-addr: 127.0.0.1:8858
    sentinel:
      transport:
        dashboard: 127.0.0.1:18080
        port: 8727
feign:
  sentinel:
    enabled: true
remote:
  wallet-api: wallet-api
```

`transport.port` 若和 8471 那台冲突，客户端会从配置值开始向上找空闲端口。两个进程不要写成同一个且都占用成功的端口。

#### 启动类激活Feign

```java
@SpringBootApplication
@EnableDiscoveryClient
@EnableFeignClients
public class WalletCallerApp7084 {
    public static void main(String[] args) {
        SpringApplication.run(WalletCallerApp7084.class, args);
    }
}
```

#### 引入Feign接口

```java
@FeignClient(value = "wallet-api", fallback = WalletClientFallback.class)
public interface WalletClient {
    @GetMapping("/ledger/sql/{id}")
    ApiEnvelope<LedgerRow> load(@PathVariable("id") Long id);
}
```

#### 加入fallback兜底方法实现

```java
@Component
public class WalletClientFallback implements WalletClient {
    @Override
    public ApiEnvelope<LedgerRow> load(Long id) {
        return ApiEnvelope.fail(4440, "wallet degraded", new LedgerRow(id, "degraded"));
    }
}
```

#### 测试

访问 7084 的调用入口，确认能转到 9303。然后停掉 9303。过一会儿再请求，返回体变成 `wallet degraded`，而不是连接失败的堆栈。这就是 Feign fallback 在提供方宕机时的表现。若 `feign.sentinel.enabled` 仍是 false，停掉提供方只会得到 Feign 的原始异常。

## 熔断框架对比

同一类问题，三套组件的侧重点不同：

| 能力 | Hystrix | Resilience4j | Sentinel |
| --- | --- | --- | --- |
| 隔离 | 线程池、信号量 | 信号量、舱壁、限时 | 线程数规则、系统规则 |
| 熔断状态 | 关闭、打开、半开 | 同样有半开和自动过渡 | 早期按时间窗口开关，新版本补了恢复探测 |
| 限流 | 弱，主要靠并发隔离 | 有限速器 | QPS、关联、链路、热点、排队、预热 |
| 控制台 | 需要单独的 Dashboard，规则在代码里 | 指标多，规则仍偏代码 | 规则可在控制台改，并可外置到 Nacos |
| 现状 | 已停止演进 | 常作为 Hystrix 的替代 | 国内 Spring Cloud Alibaba 体系里更常见 |

选型时不要三套叠在同一条调用上，否则一次失败会被多个兜底各处理一次，返回值无法预期。

## Sentinel规则持久化

### 是什么

在控制台点出来的规则放在客户端内存里。8471 重启后规则消失，生产上不能接受。需要把规则存到外部，启动时再拉回来。

### 怎么玩

把流控规则写成 JSON 放进 Nacos。8471 启动并访问任意一个已配置资源后，控制台能看到从 Nacos 推下来的规则。只要 Nacos 里的配置还在，重启应用规则仍会回来。注意方向：这个数据源示例是 Nacos 推给客户端。在控制台上改规则，不会自动写回 Nacos，除非再接一套反向的规则存储。

### 解决方法

#### 引入依赖

`sentinel-datasource-nacos` 已在工程依赖里。版本随 Spring Cloud Alibaba 的依赖清单走，不要单独乱升。

#### 修改yml

在 sentinel 节点下加 datasource。`rule-type: flow` 表示这份配置只承载流控规则。降级、热点要换类型并另起一个 dataId。

```yaml
spring:
  cloud:
    sentinel:
      datasource:
        flow-nacos:
          nacos:
            server-addr: 127.0.0.1:8858
            dataId: guard-demo-flow
            groupId: SENTINEL_GROUP
            data-type: json
            rule-type: flow
```

#### 添加nacos配置

在 Nacos 建配置，Data ID 为 `guard-demo-flow`，Group 为 `SENTINEL_GROUP`，内容是 JSON 数组。字段含义：

- resource：资源名，要和注解 value 或 URL 一致
- limitApp：来源应用，default 表示不区分
- grade：0 是线程数，1 是 QPS
- count：阈值
- strategy：0 直接，1 关联，2 链路
- controlBehavior：0 快速失败，1 预热，2 排队
- clusterMode：是否集群限流

示例只限制 `coverHot` 这个资源，每秒 2 次，快速失败：

```json
[
  {
    "resource": "coverHot",
    "limitApp": "default",
    "grade": 1,
    "count": 2,
    "strategy": 0,
    "controlBehavior": 0,
    "clusterMode": false
  }
]
```

发布配置后访问一次 8471 上对应的接口。控制台的流控规则页会出现这条记录。删掉 Nacos 配置再重启，规则不再回来。这样就把“控制台里看不到持久化”的问题解决到了流控这一类规则上。其它规则类型按同样的数据源结构各写一份即可。
