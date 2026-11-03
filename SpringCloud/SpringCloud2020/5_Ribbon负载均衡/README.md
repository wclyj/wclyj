# Ribbon 实现负载均衡

Ribbon 已进入维护，新项目会看到 Spring 自带的 `LoadBalancer`。现有工程里客户端选实例仍大量走 Ribbon，下面按这套机制说明它如何选地址。

## 概念

Ribbon 是放在调用方进程里的负载均衡库，不是独立网关。它维护一份可用服务器列表，并按规则挑一台，再把 HTTP 请求发过去。超时、重试等也在调用方配置。

规则可以替换。轮询、随机、按耗时加权都是同一套接口的不同实现。

## 负载均衡在解决什么

把请求摊到多台提供者上，避免单台打满。常见做法分两类：

- 集中式：调用方和提供者之间放 F5、LVS 或 Nginx，由这台设备转发。
- 进程内：调用方自己从注册中心拿到清单，在进程里挑一台。Ribbon 属于这一类，它只是一个库。

进程内方案少一跳，选路逻辑和业务进程同生共死；集中式则所有语言的客户端都能共用同一套规则。

## 和 Nginx 的差别

Nginx 收的是全部外部请求，再由它转发，选路发生在服务端。Ribbon 在发远程调用之前，先读本地缓存的实例表，选好 IP 再直连。注册中心不参与这次转发。

可以记成：Ribbon 负责挑地址，`RestTemplate` 负责把请求发出去。两者靠 `@LoadBalanced` 串起来。没有这个注解时，`http://billing-api/...` 里的名字不会被换成实例地址。

## 工作原理

Ribbon 不绑定死某一种注册中心，和 Eureka 配合只是常见组合。一次选择分两步：

1. 先在同一区域里挑负载较轻的注册中心读取清单（多注册中心时）。
2. 再按当前规则，从该服务的实例表里挑一台。

内置规则包括轮询、随机，以及按响应时间加权。清单来自上一次从注册中心拉到的缓存，不是每次请求都同步查询。

## 引入依赖

使用较新的 Eureka 客户端时，Ribbon 已在传递依赖里，不必再单独声明一份，否则容易出现两套版本。

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId><artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

若依赖树里没有 `ribbon-loadbalancer`，再按发行版补 `spring-cloud-starter-netflix-ribbon`。不要手写一个和父 POM 冲突的版本号。

## RestTemplate 的两种读法

`getForObject` / `postForObject` 直接给反序列化后的体。需要状态码时改用 `getForEntity`。

```java
@GetMapping("/buyer/invoice")
public InvoiceView create(InvoiceDraft draft) {
    return restTemplate.postForObject(
            BILLING + "/invoice", draft, InvoiceView.class);
}

@GetMapping("/buyer/invoice/{id}")
public InvoiceView find(@PathVariable Long id) {
    return restTemplate.getForObject(
            BILLING + "/invoice/" + id, InvoiceView.class);
}

@GetMapping("/buyer/invoice/raw/{id}")
public InvoiceView findRaw(@PathVariable Long id) {
    ResponseEntity<InvoiceView> entity = restTemplate.getForEntity(
            BILLING + "/invoice/" + id, InvoiceView.class);
    if (entity.getStatusCode().is2xxSuccessful()) {
        return entity.getBody();
    }
    throw new ResponseStatusException(HttpStatus.BAD_GATEWAY, "billing unavailable");
}
```

`BILLING` 形如 `http://billing-api`。主机名是服务名，路径才是对方控制器上的路径。

## 核心接口 IRule

默认是轮询。`IRule` 从负载均衡器给出的清单里选一台 `Server`：

```java
public interface IRule {
    Server choose(Object key);
    void setLoadBalancer(ILoadBalancer balancer);
    ILoadBalancer getLoadBalancer(); // 取当前清单
}
```

真正的清单在 `ILoadBalancer` 里，规则对象只负责挑选。更换规则时替换这个 Bean，调用代码不用改。

### 七种常用规则

- `RoundRobinRule`：按顺序逐台轮转。
- `RandomRule`：在存活列表里随机。
- `RetryRule`：先按轮询取，取到的实例失败则在时限内再试下一台。
- `WeightedResponseTimeRule`：近期响应越快，被选中的权重越高。
- `BestAvailableRule`：跳过已熔断的实例，在剩余里选并发最小的。
- `AvailabilityFilteringRule`：先去掉故障实例，再避开并发过高的。
- `ZoneAvoidanceRule`：默认规则，同时看所在区域的健康度和实例是否可用。

区域规则在多机房时能避免把流量打进整区不可用的一段。单机房试验时，它的表现接近轮询。

### 替换默认规则

自定义配置类不要放在启动类所在包以及子包里。组件扫描扫到它之后，这个规则会变成全局默认，所有 Ribbon 客户端共用，无法按服务区分。

把配置类放到扫描范围之外，例如 `com.shop.checkout.lb`：

```java
@Configuration
public class PickAtRandom {
    @Bean
    public IRule randomAmongBilling() {
        return new RandomRule();
    }
}
```

只对 `billing-api` 生效：

```java
@SpringBootApplication
@EnableDiscoveryClient
@RibbonClient(name = "billing-api", configuration = PickAtRandom.class)
public class CheckoutApp {
    public static void main(String[] cli) {
        SpringApplication.run(CheckoutApp.class, cli);
    }
}
```

`name` 与注册名一致。Eureka 面板是大写时，这里仍写配置里的 `spring.application.name`，由客户端去做匹配。另一个服务要固定轮询时，再声明一份独立的配置类，不要复用同一个 `@Configuration`。

## 手写轮询

### 原理

下标 = 第几次调用 % 实例数。重启后计数从 0 或 1 重新开始都可以，只要单调增加。

两台实例时：

```text
第 1 次  1 % 2 = 1  -> 列表下标 1
第 2 次  2 % 2 = 0  -> 列表下标 0
第 3 次  3 % 2 = 1  -> 又回到下标 1
```

实例数变化后，同一余数会落到不同机器，这是预期行为。扩容瞬间允许有一两下打偏，不需要和注册表加分布式锁。

随机规则的源码并不是取余，而是在存活列表长度内取随机整数，实例为空或已死就让出 CPU 再试。手写轮询不必复制那段循环，只要保证计数在并发下不丢即可。

### 实现

先去掉 `RestTemplate` 上的 `@LoadBalanced`，否则请求会先被 Ribbon 拦截，走不到自己的算法。此时 URL 必须已经是具体的 `http://ip:port`。

```java
@Bean
public RestTemplate plainClient() {
    return new RestTemplate(); // 不经过负载均衡拦截器
}
```

选择器只做一件事：从一份实例列表里返回一台。

```java
public interface InstancePicker {
    ServiceInstance pick(List<ServiceInstance> candidates);
}
```

用原子整数做计数，CAS 失败就重读，避免加锁：

```java
@Component
public class RoundRobinPicker implements InstancePicker {

    private final AtomicInteger seq = new AtomicInteger();

    private int nextTicket() {
        int prev;
        int updated;
        do {
            prev = seq.get();
            updated = (prev == Integer.MAX_VALUE) ? 0 : prev + 1;
        } while (!seq.compareAndSet(prev, updated));
        return updated;
    }

    @Override
    public ServiceInstance pick(List<ServiceInstance> candidates) {
        int slot = Math.floorMod(nextTicket(), candidates.size());
        return candidates.get(slot);
    }
}
```

`floorMod` 在计数回到 0 时也不会得到负数下标。

调用步骤：按服务名取出全部实例，列表为空就直接返回；否则交给选择器，再用普通 `RestTemplate` 访问选中的 URI。

```java
@GetMapping("/buyer/invoice/pick")
public String pickOne() {
    List<ServiceInstance> nodes = discoveryClient.getInstances("BILLING-API");
    if (nodes == null || nodes.isEmpty()) {
        return "no billing instance";
    }
    URI target = picker.pick(nodes).getUri();
    return restTemplate.getForObject(target + "/invoice/who", String.class);
}
```

`@LoadBalanced` 和手写选择器不要同时开。打开注解后，服务名可以被解析，但上面的 `getUri()` 路径会被再处理一次，容易出现地址套地址。试验手写算法时只保留一条路径。

## 踩坑

- 规则类被组件扫描扫进默认上下文后，所有服务突然都变成随机，难以察觉。
- 实例列表为空时仍做取余，会抛除零异常。先判断再运算。
- 手写轮询拿的是调用那一刻的清单，不会自动跳过刚刚失败的连接。需要失败转移时，用现成的 `RetryRule`，或在手写逻辑里把失败实例临时排除。
- Ribbon 维护期结束后，新代码应优先看 Spring `LoadBalancer` 的 `ReactorLoadBalancer`，选择思想相同：仍是调用方进程内挑实例。
