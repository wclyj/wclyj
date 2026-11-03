# 服务网关

## 前言

Zuul 1 进入维护后，Zuul 2 的重写在社区内部出现分歧，核心作者离开，版本一再推迟。等待期间 Spring Cloud 自己做了 Gateway，现在新项目的网关默认就选它。

## 概念

### Zuul

Zuul 是 Netflix 给设备和站点请求准备的边缘入口，负责动态路由、观测、弹性和安全，也能把流量分到多个自动伸缩组。Spring Cloud 早期集成的是它的 1.x。

### Gateway

1.x 时代全家桶的网关位是 Zuul。2.x 因为 Zuul 2 迟迟不能作为稳定依赖，Spring 团队用 Gateway 接替 Zuul 1。它基于 Spring 5、Spring Boot 2 和 Project Reactor，路由规则写起来直接，过滤器可以做熔断、限流和重试。

Gateway 跑在 WebFlux 上，底层通信用 Netty，而不是给每个请求占一个 Servlet 线程。Spring Cloud 2 没有把 Zuul 2 的非阻塞实现收进来，仍停留在 Zuul 1 的阻塞模型上，所以性能差距主要来自线程模型，不是来自“路由”这个功能本身。

它的目标是：用统一的路由表把请求送到下游，再用过滤器链补上安全、指标和限流。

### 能做啥

- 反向代理
- 鉴权
- 流量控制
- 熔断
- 访问日志

### 使用场景

浏览器和开放平台只认识一个入口域名。网关根据路径、请求头或参数把调用分到订单、库存、账户。下游实例的 IP 不暴露给外部。

### 为什么选用Gateway

Zuul 1 只维护不演进。Gateway 由 Spring Cloud 团队维护，和注册发现、负载均衡的集成是现成的。Zuul 1 里不少高级能力在 Spring Cloud 集成层并没有真正用起来。Zuul 2 虽然发布了，Spring Cloud 没有排进整合计划，因为 Netflix 组件整体进入维护。综合维护状态和异步模型，Gateway 是更合适的默认网关。

### Gateway特性

构建在 Spring Framework 5、Project Reactor 和 Spring Boot 2 之上。

- 路由可以匹配路径、头、参数、时间等请求属性
- 每条路由能挂断言和过滤器
- 能接 Hystrix 或替代的断路器
- 能从注册中心按服务名发现实例
- 断言和过滤器都可以自己写
- 自带请求限流
- 支持改写路径

### Spring Cloud Gateway  和 Zuul的区别

Finchley 正式发布之前，官方推荐的网关仍是 Zuul。

- Zuul 1 是阻塞 IO 的 API 网关
- 它基于 Servlet 2.5 的阻塞模型，不支持长连接。每次 IO 都要从工作线程里取一条，请求线程一直等到工作完成。Nginx 也是多工作进程，但实现语言和运行时不同；Zuul 跑在 JVM 上，冷启动和线程成本更高
- Zuul 2 想改成基于 Netty 的非阻塞并支持长连接，Spring Cloud 没有集成。官方基准里 Gateway 的每秒请求数大约是 Zuul 1 的 1.6 倍
- Gateway 建立在 Spring 5 和 Boot 2 上，使用非阻塞 API
- Gateway 支持 WebSocket，和 Spring 的配置方式一致，过滤器可以用响应式类型编写

Spring Cloud 带的 Zuul 跑在 Tomcat 里，走的是传统 Servlet。容器启动时创建 Servlet 并调用 `init`；每个请求从线程池拿一条线程调用 `service`；容器关闭时调用 `destroy`。

并发不高时这很合适。并发一上去，线程数跟着涨，上下文切换和栈内存都会变成延迟来源。有些网关逻辑只是改个头、转发一下，并不需要“一请求一线程”。Zuul 1 的入口本质上仍是那个阻塞的 DispatcherServlet，所以绕不开这个模型。

### WebFlux框架

Struts2、Spring MVC 都站在 Servlet API 上。Servlet 3.1 之后容器可以异步，但编程模型仍围绕 Servlet。WebFlux 则是另一套栈：核心是 Reactor，可以跑在 Netty、Undertow，或支持 Servlet 3.1 的容器里。它不依赖 Servlet API，用响应式流把背压传下去。使用它需要 Java 8 的函数式写法，这是 Spring 5 的基线。

网关选择 WebFlux，是因为转发本身是 IO 密集，用少量事件循环线程就能扛住大量连接。

## 三大核心概念

### Route 路由

路由是网关的一条规则，包含唯一 id、目标 URI、一组断言和一组过滤器。断言全部通过，请求才走这条路由。

### Predicate 断言

对应 Java 8 的 `Predicate`。它检查请求的路径、方法、头、Cookie、参数、时间。匹配成功才进入该路由。

### Filter 过滤

`GatewayFilter` 的实例。可以在转发前改请求，也可以在下游返回后改响应。

## Gateway工作流程

请求先被断言匹配到一条路由，再沿过滤器链前进，最后转发到目标 URI。断言决定“去哪”，过滤器决定“转发前后改什么”。

客户端请求进入 Gateway 后，由 Handler Mapping 选出路由，交给 Web Handler。Handler 按顺序执行过滤器链，再把请求代理到真实服务，响应沿同一条链回来。

过滤器分成 pre 和 post。图里常用虚线把两段隔开：虚线之前还没发出代理请求，虚线之后下游已经返回。

pre 适合做参数校验、权限、流量计数、访问日志、协议转换。post 适合改响应头、改响应体、记录耗时。网关的核心就两句话：按路由转发，并执行过滤器链。

## 入门配置

### 引入依赖

不要同时引入 `spring-boot-starter-web`，它会把应用拉回 Servlet 栈，和 Gateway 冲突。

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-gateway</artifactId>
</dependency>
```

### 修改YML

网关进程叫 `edge-door`，监听 7600。两条路由都指向注册中心里的 `BILLING-CORE`，用 `lb://` 表示按服务名负载。

```yaml
spring:
  application:
    name: edge-door
  cloud:
    gateway:
      discovery:
        locator:
          enabled: true
      routes:
        - id: bill_query
          uri: lb://BILLING-CORE
          predicates:
            - Path=/bill/query/**
        - id: bill_lb
          uri: lb://BILLING-CORE
          predicates:
            - Path=/bill/lb/**
            - After=2024-06-01T09:30:00.000+08:00[Asia/Shanghai]
```

### 访问

没有网关时，直接打下游：

```text
本机 8101 端口 /bill/query/31
```

加上网关后，外部只访问：

```text
本机 7600 端口 /bill/query/31
```

下游真实端口可以更换，调用方不用跟着改。第二条路由还要求当前时间晚于断言里的时刻，适合做“到点才开放”的入口。

### 路由匹配

请求路径先和 `Path` 断言比。`/bill/query/**` 能吃掉多级子路径。若两条路由都能匹配，优先级看配置顺序和断言的具体程度，调试时给每条路由打上不同的响应头，确认实际命中了哪一条。

### 路由配置的两种方式

- 写在 yml 里，适合大部分静态规则
- 注册 `RouteLocator` Bean，适合规则要在代码里拼接的场景

```java
@Configuration
public class EdgeRouteConfig {

    @Bean
    public RouteLocator digestRoute(RouteLocatorBuilder builder) {
        return builder.routes()
                .route("digest-brief",
                        r -> r.path("/brief").uri("lb://DIGEST-HUB"))
                .build();
    }
}
```

访问网关的 `/brief` 会被转到摘要服务。目标用服务名，而不是把外部站点地址写进路由。

## 按注册名做动态转发

打开发现定位器之后，Gateway 会为注册中心里的每个服务生成一条路由，路径前缀默认是服务名。

```yaml
spring:
  cloud:
    gateway:
      discovery:
        locator:
          enabled: true
          lower-case-service-id: true
```

手写路由时把目标写成：

```yaml
uri: lb://BILLING-CORE
```

`lb` 表示从注册中心选实例。服务扩容或缩容不需要改网关配置。

## Predicate的使用

### 概念

断言决定路径或其他条件是否命中。Gateway 把路由匹配接在 WebFlux 的 HandlerMapping 上。内置一批 Route Predicate 工厂，每个工厂对应 HTTP 请求的一种属性。多个断言写在同一条路由下时按“与”组合，必须全部成立。

创建路由时，框架用 `RoutePredicateFactory` 产生 `Predicate`，再塞进 Route 对象。

### 常用的Predicate

- After：该时间点之后才匹配
- Before：该时间点之前才匹配
- Between：落在两个时间点之间才匹配
- Cookie：必须带上指定 Cookie。调试可以用 curl 的 cookie 参数，例如给 7600 端口的 `/bill/lb` 带上 `caller=ada`
- Header：必须带某个请求头，并可用正则约束值
- Host：按主机名匹配，适合一个网关挂多个域名
- Method：限定 GET、POST 等
- Path：按路径匹配，例如 `- Path=/open-api/**`
- Query：必须带某个查询参数

时间类断言使用带时区的 ISO 时间。Cookie、Header、Query 适合做灰度：只有带特定标记的请求才进入新路由。

## Filter的使用

### 概念

路由过滤器改的是进入的请求和返回的响应，并且只作用于声明了它的那条路由。内置过滤器都由对应的工厂创建，在 yml 里按名字引用即可，例如 `StripPrefix`、`AddRequestHeader`。

### Spring Cloud Gateway Filter

生命周期只有两段：pre 与 post。

类型也只有两类：

- GatewayFilter：绑在单条路由上
- GlobalFilter：对所有路由生效

### 自定义过滤器

常见用途：

- 统一打访问日志
- 在网关做一层粗粒度鉴权

实现 `GlobalFilter` 和 `Ordered`。`getOrder` 越小越先执行。

```java
@Slf4j
@Component
public class AccessStampFilter implements GlobalFilter, Ordered {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange,
                             GatewayFilterChain chain) {
        log.info("edge enter {}", Instant.now());
        String caller = exchange.getRequest().getQueryParams().getFirst("caller");
        if (caller == null || caller.isBlank()) {
            log.info("缺少 caller，拒绝转发");
            ServerHttpResponse denied = exchange.getResponse();
            denied.setStatusCode(HttpStatus.UNAUTHORIZED);
            return denied.setComplete();
        }
        Mono<Void> downstream = chain.filter(exchange);
        return downstream;
    }

    @Override
    public int getOrder()
    {
        return -10;
    }
}
```

查询参数里没有 `caller` 时，过滤器直接完成响应，不再调用 `chain.filter`，下游不会收到请求。校验通过才把交换对象交给下一个过滤器。生产环境应改成校验令牌头，而不是一个明文查询参数；这里只用来说明“不放行就短路”的写法。
