# Hystrix断路器

Hystrix 已停止演进。官方更倾向用 Resilience4j 接替，国内实践里也常用 Sentinel 同时做熔断和限流。下面仍把 Hystrix 的模型写清楚，因为后来的组件沿用了同一套降级、熔断、限流词汇。

## 概述

### 分布式面临的问题

调用链一长，每个下游都可能超时或拒绝。票务服务要问库存、座位锁和支付，其中任何一跳卡住，调用方的线程就停在那里。

### 服务雪崩

服务 A 同时调用 B 和 C，B、C 再向外扇出。这就是扇出。扇出链上某一环变慢，A 的线程池会被慢慢占满，最后连不依赖那一环的接口也没有线程可用。高流量下，一个后端依赖就能在几秒内把所有实例的线程打满。比“直接失败”更麻烦的是延迟被放大：队列堆积、线程耗尽，故障沿调用方向往回传。

所以要隔离。某个实例已经不健康，就不该继续把流量送进去，更不该让它拖住别的模块。

### HyStrix的诞生

Hystrix 用来限制延迟和失败的扩散。依赖超时或抛错时，它给调用方一个事先准备好的备用结果，而不是让线程一直挂着。断路器像电路里的保险丝：故障计数达到阈值就断开，后续请求直接走备用逻辑，直到后面的探测认为下游恢复。

### Hystrix作用

- 服务降级
- 服务熔断
- 近实时的指标面板

## Hystrix重要概念

### 服务降级

下游不可用时，至少返回一个调用方能处理的结果，例如“票务繁忙，请稍后再试”，而不是一直转圈。

会走到降级的情况：

- 业务方法抛出运行时异常
- 调用超过设定的时间上限
- 断路器已经打开
- 隔离用的线程池或信号量被占满

### 服务熔断

熔断是降级的一种触发条件。访问量或失败率越过门槛后，直接拒绝继续打到下游，并改走降级方法。常见顺序是：先有降级兜底，失败累积后熔断，隔一段时间再试探恢复。

### 服务限流

限流约束的是进来的速度。秒杀场景里不能让所有请求同时打进核心交易，而是按每秒固定个数放行。Hystrix 本身更强调隔离和熔断，细粒度的 QPS 限流放到后面的 Sentinel 笔记里写。

## Hystrix案例

### 构建

#### 引入依赖

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
</dependency>
```

#### 启动类添加Hystrix注解

`@EnableCircuitBreaker` 打开断路器。下面的 Servlet 注册只为了把指标流暴露出来，和降级逻辑无关。Boot 升级后默认不会映射 `/hystrix.stream`，需要自己登记。

```java
@SpringBootApplication
@EnableDiscoveryClient
@EnableCircuitBreaker
public class TicketHystrixApp8102 {

    public static void main(String[] args) {
        SpringApplication.run(TicketHystrixApp8102.class, args);
    }

    @Bean
    public ServletRegistrationBean<HystrixMetricsStreamServlet> ticketMetrics() {
        HystrixMetricsStreamServlet servlet = new HystrixMetricsStreamServlet();
        ServletRegistrationBean<HystrixMetricsStreamServlet> bean =
                new ServletRegistrationBean<>(servlet, "/hystrix.stream");
        bean.setName("ticketMetricsStream");
        bean.setLoadOnStartup(1);
        return bean;
    }
}
```

服务监听 8102。

#### 业务类

```java
@Service
public class TicketService {

    public String ticketOk(Integer id) {
        return Thread.currentThread().getName() + " ticketOk id=" + id;
    }

    @HystrixCommand(fallbackMethod = "ticketSlowFallback", commandProperties = {
            @HystrixProperty(name = "execution.isolation.thread.timeoutInMilliseconds", value = "4000")
    })
    public String ticketSlow(Integer id) {
        try {
            TimeUnit.SECONDS.sleep(2);
        } catch (InterruptedException ex) {
            Thread.currentThread().interrupt();
        }
        return Thread.currentThread().getName() + " ticketSlow id=" + id;
    }

    public String ticketSlowFallback(Integer id) {
        return Thread.currentThread().getName() + " ticket desk busy, id=" + id;
    }

    @HystrixCommand(fallbackMethod = "ticketFuseFallback", commandProperties = {
            @HystrixProperty(name = "circuitBreaker.requestVolumeThreshold", value = "8"),
            @HystrixProperty(name = "circuitBreaker.sleepWindowInMilliseconds", value = "8000"),
            @HystrixProperty(name = "circuitBreaker.errorThresholdPercentage", value = "50"),
            @HystrixProperty(name = "circuitBreaker.enabled", value = "true")
    })
    public String ticketFuse(Integer id) {
        if (id < 0) {
            throw new IllegalArgumentException("id must be positive");
        }
        return Thread.currentThread().getName() + " fuse-ok " + UUID.randomUUID();
    }

    public String ticketFuseFallback(Integer id) {
        return "rejected non-positive id " + id;
    }
}
```

正常方法两秒返回，上限四秒，所以不会降级。把睡眠改成五秒，或在方法里制造算术异常，就会进 `ticketSlowFallback`。

### 高并发测试

用压测工具开两万个线程打慢接口。慢接口把 Tomcat 工作线程占满之后，本来很快的 `ticketOk` 也会排队甚至超时。问题不在业务计算，而在线程被慢调用占用，没有余量处理别的请求。

可以粗算一笔账。容器工作线程假如是 200，慢接口每次占用 2 秒，那么这 200 条线程每秒大约只能消化 100 个慢请求。压测把线程全部借走之后，健康接口连一条空闲线程都拿不到，表现为“没报错的接口也超时”。这就是雪崩在单机上的样子：故障点不一定是被压测的那个方法本身，而是它占用了共享的线程池。

Hystrix 的隔离就是为了切开这个共享池。两种方式：

- 线程池隔离：每个命令有自己的一小池线程。慢命令把自己的 10 条线程占满后，多出来的调用直接降级，Tomcat 的线程可以返回去处理别的接口。代价是多一次线程切换。
- 信号量隔离：不另开线程，只限制同时进入方法的个数。适合本身很快、主要怕并发打满的本地逻辑，不适合会阻塞很久的远程调用。

远程调用默认应使用线程池隔离。池大小按“下游可承受的并发”来设，而不是按入口 QPS 来设。池太小会误降级，池和 Tomcat 一样大就等于没有隔离。

和隔离配套的几个属性，名字长，但只记住用途即可：

- `execution.isolation.strategy`：`THREAD` 或 `SEMAPHORE`
- `execution.isolation.thread.timeoutInMilliseconds`：跑业务方法的时间上限，超时进入兜底
- `coreSize`：该命令专用线程池的大小
- `maxQueueSize`：池外还能排多少，-1 表示用同步队列，排不下就拒绝
- `circuitBreaker.sleepWindowInMilliseconds`：打开后多久允许一次探测

这些属性都挂在 `@HystrixCommand` 的 `commandProperties` 或 `threadPoolProperties` 上。同一个下游的多个方法如果共用一个线程池键（`threadPoolKey`），隔离才是按“下游”而不是按“方法”生效。不写键时，默认用命令分组，常常是每个类一个池，粒度偏粗。票务里可以把 `ticketSlow` 和 `ticketFuse` 放进同一个 `ticket-desk` 池，让慢查询和熔断探测一起受这组线程限制，健康的 `ticketOk` 留在别的池，避免被拖死。

拒绝发生时走的仍是 fallback，和超时、异常是同一条降级出口。日志里要区分原因：超时、拒绝、短路、业务异常，否则会把“池太小”误判成“下游挂了”。

### 服务消费者加入

只在 8102 做降级还不够。调用方 7088 如果同步等下游，自己的线程一样会被拖死。消费端打开 Hystrix，并给 Feign 配一个兜底实现。

```java
@SpringBootApplication
@EnableFeignClients
@EnableHystrix
public class TicketCallerApp7088 {
    public static void main(String[] args) {
        SpringApplication.run(TicketCallerApp7088.class, args);
    }
}
```

```yaml
server:
  port: 7088
eureka:
  client:
    register-with-eureka: true   # 调用方也登记，便于面板按服务名找到它
    fetch-registry: true         # 需要拉到 ticket-desk 的实例列表
    # 指向 registry-a 的 7101，值里要写协议和 /eureka 路径
feign:
  hystrix:
    enabled: true
```

```java
@FeignClient(value = "ticket-desk", fallback = TicketClientFallback.class)
public interface TicketClient {
    @GetMapping("/ticket/ok/{id}")
    String ticketOk(@PathVariable("id") Integer id);

    @GetMapping("/ticket/slow/{id}")
    String ticketSlow(@PathVariable("id") Integer id);
}
```

```java
@Component
public class TicketClientFallback implements TicketClient {
    @Override
    public String ticketOk(Integer id) {
        return "caller fallback ok, id=" + id;
    }

    @Override
    public String ticketSlow(Integer id) {
        return "caller fallback slow, id=" + id;
    }
}
```

`feign.hystrix.enabled` 不打开时，Feign 的 fallback 不会生效。

## 解决方案

### 原因

超时就要停止等待。出错或宕机就要有兜底。三种典型情况：

- 8102 处理太慢，7088 不能一直阻塞
- 8102 进程没了，7088 不能一直阻塞
- 8102 正常，但 7088 自己的时限更短，由调用方主动降级

### 服务降级

注解用 `@HystrixCommand`。提供方启动类使用 `@EnableCircuitBreaker`。在时限内走原方法，超过时限或抛错就进 fallback。fallback 的参数列表要和原方法一致。

上面的 `ticketSlow` 把时限设成 4000 毫秒。睡眠只有 2 秒时返回正常文案；睡眠超过 4 秒，或方法里出现除零，都会进兜底。降级可以写在提供方，也可以写在调用方。更常见的是写在调用方，因为提供方已经忙不过来时，再在它里面做降级，线程可能已经不够用了。

调用方示例：

```java
@GetMapping("/caller/ticket/slow/{id}")
@HystrixCommand(fallbackMethod = "slowFallback", commandProperties = {
        @HystrixProperty(name = "execution.isolation.thread.timeoutInMilliseconds", value = "1800")
})
public String ticketSlow(@PathVariable("id") Integer id) {
    return ticketClient.ticketSlow(id);
}

public String slowFallback(Integer id) {
    return "caller timed out, id=" + id;
}
```

调用方只愿等 1.8 秒，哪怕提供方 4 秒内能返回，调用方也会先降级。热部署对 `@HystrixCommand` 里的属性经常不生效，改完时限建议整进程重启。

两端都配了兜底时，谁的时限更短谁先返回。7088 在 1.8 秒降级之后，8102 上的调用仍会继续跑完，除非调用方使用的是可取消的隔离线程。所以“调用方已经返回兜底”不等于“提供方没有执行”。有写库副作用的接口要自己保证幂等，不能假设超时就等于没发生。

消费端 yml 同样需要：

```yaml
feign:
  hystrix:
    enabled: true
```

**目前问题**

每个接口一个兜底方法，会和业务方法缠在一起，控制器很快膨胀。

**方法1：**

核心接口保留自己的 fallback，其余接口走类上的默认兜底。`@HystrixCommand` 不写 `fallbackMethod` 时，使用 `@DefaultProperties` 指定的方法。

```java
@RestController
@DefaultProperties(defaultFallback = "globalFallback")
public class TicketCallerController {

    @Resource
    private TicketClient ticketClient;

    @GetMapping("/caller/ticket/slow/{id}")
    @HystrixCommand
    public String ticketSlow(@PathVariable("id") Integer id) {
        if (id == 0) {
            throw new IllegalStateException("local guard");
        }
        return ticketClient.ticketSlow(id);
    }

    public String globalFallback() {
        return "ticket caller is protecting itself";
    }
}
```

默认兜底方法不能带参数。方法体内在调用下游之前就抛错，一样会进它。

**方法2：**

把兜底从控制器挪到 Feign 的实现类，业务代码只保留接口。格式是 `@FeignClient(fallback = TicketClientFallback.class)`。

调用方要处理的失败主要是三类：

- 运行期异常
- 超时
- 下游宕机

`TicketClientFallback` 实现 `TicketClient` 的每个方法。提供方进程关掉之后，7088 仍能返回兜底文案，不会把 Tomcat 线程挂死。注意 fallback 类本身要是 Spring Bean。

### 服务熔断

熔断可以看成降级在“失败已经成趋势”时的特例。

#### 熔断概念

扇出链上某个节点持续不可用或极慢时，继续调用只会放大故障。熔断把这个节点暂时摘掉，直接返回降级结果。探测到恢复后再把流量放回去。

在这套框架里由 Hystrix 统计调用结果。默认经验值常被说成“5 秒内 20 次失败就断开”，实际阈值都在注解里，下面的例子改成了更小的窗口，方便手工打出来。注解仍然是 `@HystrixCommand`。

马丁·福勒描述过断路器：打开期间不再调用被保护的动作；和楼里的空气开关不同，软件断路器可以自己试探，成功后再合上。

三种状态：

- 关闭：请求正常进入原方法，失败只被统计
- 打开：请求不再进入原方法，直接降级
- 半开：打开一段时间后放进少量请求。成功则关闭，失败则回到打开

在 8102 上给熔断单独留一个方法。四个属性：

四个开关分开记，不要写成一整行注解：

- `circuitBreaker.enabled` 设为开启
- `circuitBreaker.requestVolumeThreshold` 设为 8
- `circuitBreaker.sleepWindowInMilliseconds` 设为 8000
- `circuitBreaker.errorThresholdPercentage` 设为 50

含义是：统计窗口里至少 8 次调用，失败达到一半，断路器打开；打开后过 8 秒进入半开。`ticketFuse` 在 id 为负时抛错。连续用负数打够次数后，再传正数也会直接进 `ticketFuseFallback`，直到半开探测成功。

### 总结

#### 熔断类型

- 打开：不再调用当前下游。内部时钟大约按平均恢复时间来设，到期转入半开
- 关闭：不拦截，只统计
- 半开：按规则放行少量请求。符合恢复条件就关闭，否则继续打开

#### 断路器启动条件

三个关键数：快照时间窗、请求总数阈值、错误百分比阈值。

- 快照时间窗：统计最近一段调用，默认大约 10 秒
- 请求总数阈值：窗口内调用次数不够就不够格熔断。默认 20。10 秒里只来了 5 次且全部失败，断路器仍保持关闭
- 错误百分比阈值：次数已经超过总数阈值之后，失败占比再和这个百分比比。例如 30 次里 15 次超时，在默认 50% 下就会打开

合在一起的默认行为：

- 窗口内请求数超过门槛（常说 10 秒 20 次）
- 失败率超过门槛（常说 50%）
- 两个条件同时满足才打开
- 打开期间请求都不再转发到原逻辑
- 休眠窗口结束（默认约 5 秒）进入半开，放行一次。成功就关闭，失败就重新打开并重新计时

打开之后的恢复方式：Hystrix 不会永远走降级。休眠窗口内降级逻辑临时充当主路径；窗口结束做一次真实调用。这次成功，主路径回来；这次仍失败，继续打开，休眠重新计算。

用数字走一遍半开。窗口 10 秒、最少 8 次、失败率 50%、休眠 8 秒。第 0 到 10 秒来了 10 次，其中 6 次失败，失败率 60%，断路器打开。第 10 到 18 秒内即使用户传了合法 id，也不会进 `ticketFuse`。第 18 秒放行一次：若返回正常，计数清空，状态回到关闭；若这次仍抛错，打开状态延续，再等 8 秒。手工验证时把休眠设短一点，否则会误以为“正数 id 也永远进兜底”。

### 服务限流

Hystrix 能用线程池和信号量限制并发，但按 QPS、按热点参数限流不是它的强项。那部分放到 Sentinel 的笔记里。

## Hystrix工作流程

面板上的示意通常用一种颜色画调用方向，另一种颜色画返回方向。按步骤走一遍：

1. 把一次调用包装成 Hystrix 命令
2. 选择隔离方式：线程池或信号量
3. 若结果缓存命中，直接返回
4. 断路器若已打开，跳到降级
5. 断路器关闭时，还要看隔离池里有没有空位，没有也降级
6. 执行构造和 run，也就是真正的业务调用
7. 把成功、超时、失败上报给断路器，用于后续判断
8. 失败路径调用 `getFallback`
9. 成功路径把原结果返回给调用方

缓存、断路器、隔离池是三道提前返回的门，任何一道拦住都不会打到下游。

## 服务监控HystrixDashboard

### 概述

Hystrix 会记录经过它的每次调用：每秒多少请求、成功多少、失败多少、超时多少。`hystrix-metrics-event-stream` 把这些数以流的形式吐出。Dashboard 再把流画成图，用来看单个命令的健康和流量，而不是替代全链路追踪。

### 搭建

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-hystrix-dashboard</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

面板单独起一个进程，监听 9310：

```yaml
server:
  port: 9310
```

```java
@SpringBootApplication
@EnableHystrixDashboard
public class TicketDashApp9310 {
    public static void main(String[] args) {
        SpringApplication.run(TicketDashApp9310.class, args);
    }
}
```

被监控的服务自己也要有 Actuator，并且像前面一样把 `HystrixMetricsStreamServlet` 映射到 `/hystrix.stream`。只有面板没有指标流，页面上会一直转圈。

浏览器打开 9310 上的 hystrix 页面，在输入框里填被监控实例的指标流地址，例如 8102 的 `/hystrix.stream`，再填一个标题后开始监控。

### 使用监控

用 9310 去看 8102，不要让面板监控自己。先访问 8102 的正常熔断接口，例如 `/ticket/fuse/31`。图上开始出现请求计数，实心圆为绿色。再连续访问负数 id，例如 `/ticket/fuse/-3`。失败率升到很高之后，断路器标记变为打开，随后的正数请求也不会进入原方法。

图怎么读：

七种颜色对应七类结果，悬停在图例上可以核对，含义分别是：

- 成功
- 短路（断路器打开后被直接拒绝）
- 非法请求（Hystrix 认为不属于故障的坏参数）
- 超时
- 线程池或信号量拒绝
- 失败（业务抛错）
- 另一类执行异常

实心圆有两层信息。颜色从绿到黄、橙、红，表示健康变差。圆的面积随流量变大，用来在一堆实例里先看到最忙的那个。

圆上的曲线记录大约两分钟内的相对流量，上升和下降能看出突发。数字区里的线程池活跃数如果顶满，说明该先加隔离容量或给慢调用降级，而不是只看业务异常。

读图时按这个顺序，避免只盯着颜色：

1. 先看圆的颜色。绿色表示最近的桶大多成功；变成橙色或红色时，再看是超时多还是失败多。
2. 再看圆的大小。又大又红，是高流量下的故障；又小又红，可能只是低频接口偶发失败，优先级不同。
3. 看短路比例。短路占比突然升高，说明断路器已经打开，此时去查下游日志，而不是继续加调用方重试。
4. 看线程池拒绝。拒绝高而失败不高，是隔离池太小，不是下游业务错了。
5. 看两分钟曲线。曲线陡升后颜色变差，多半是突发流量打到了冷缓存或慢依赖。

面板进程 9310 只负责画图。真正的计数发生在 8102。8102 没人访问时，图是空的，这不代表监控挂了。先打一笔 `/ticket/fuse/31`，再刷新面板。指标流地址填错端口时，页面会停在 loading，优先核对 8102 是否映射了指标流，而不是先怀疑面板依赖。
