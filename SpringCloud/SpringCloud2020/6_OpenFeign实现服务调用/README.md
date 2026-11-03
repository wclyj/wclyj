# OpenFeign实现服务调用

早期的 Feign 已经停止演进，Spring Cloud 侧接替它的是 OpenFeign。下面的写法都按 OpenFeign 来。

## 概述

Feign 是声明式的 HTTP 客户端。先写一个 Java 接口，再在接口和方法上加注解，运行时由动态代理去发请求。编码器、解码器都可以换。Spring Cloud 在这层外面又包了一层，让接口能直接使用 Spring MVC 的映射注解和消息转换器，并且能从注册中心拿实例列表做负载均衡。

## Feign的作用

用 Ribbon 加 `RestTemplate` 时，每个调用点都要自己拼路径、自己处理返回值。同一个下游接口往往被多个业务类调用，于是每个服务都会手写一套客户端包装。Feign 把这件事收成“接口即客户端”：以前在 Dao 上标 Mapper，现在在远程接口上标 Feign 注解，绑定、负载和编解码都由代理完成。

## Feign集成Ribbon

Ribbon 维护着下游实例列表，并按轮询等策略选一台。Feign 把这次选择藏在代理里。调用方只声明“我要访问哪个服务名的哪个路径”，不必再写 `restTemplate.getForObject`。负载策略仍然是 Ribbon 那一套，只是入口从模板方法换成了接口方法。

## Feign和OpenFeign的区别

| Feign | OpenFeign |
| --- | --- |
| Spring Cloud 里较轻的 REST 客户端，自带 Ribbon，用自己的注解描述接口，再由代理去注册中心找实例 | 在 Feign 之上识别 Spring MVC 注解，例如映射路径、路径变量。`@FeignClient` 会为接口生成代理，代理里做负载并调用下游 |
| 依赖坐标 `spring-cloud-starter-feign` | 依赖坐标 `spring-cloud-starter-openfeign` |

新项目只保留第二列。两套注解混用时，路径解析很容易和实际控制器对不上。

## OpenFeign使用步骤

### 引入依赖

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```

### 修改启动类

用 `@EnableFeignClients` 打开扫描。不写包名时，从启动类所在包往下找带 `@FeignClient` 的接口。

```java
@SpringBootApplication
@EnableFeignClients
public class CheckoutFeignApp {
    public static void main(String[] args) {
        SpringApplication.run(CheckoutFeignApp.class, args);
    }
}
```

### 添加业务逻辑接口

`value` 写注册中心上的服务名，不要写主机和端口。

```java
@FeignClient(value = "billing-core")
public interface BillingClient {

    @GetMapping("/bill/{billId}")
    ApiEnvelope<Bill> loadBill(@PathVariable("billId") Long billId);

    @GetMapping("/bill/quote/slow")
    String slowQuote();
}
```

### 具体使用

控制器注入接口，像调用本地方法一样调用。代理会按服务名解析实例。

```java
@RestController
public class CheckoutController {

    @Resource
    private BillingClient billingClient;

    @GetMapping("/checkout/bill/{billId}")
    public ApiEnvelope<Bill> loadBill(@PathVariable("billId") Long billId) {
        return billingClient.loadBill(billId);
    }

    @GetMapping("/checkout/bill/quote/slow")
    public String slowQuote() {
        // 下游故意睡 4 秒，用来观察客户端等待上限
        return billingClient.slowQuote();
    }
}
```

## OpenFeign的超时控制

下游报价要 4 秒才返回，客户端默认大约只等 1 秒，于是调用方先超时，表现为读超时而不是业务错误。把读超时和连接超时都拉长即可。下面按 Ribbon 的配置项来写，因为这一代 OpenFeign 默认仍走 Ribbon：

```yaml
ribbon:
  # 连接建立之后，等待响应体的上限
  ReadTimeout: 8000
  # 建立 TCP 连接的上限
  ConnectTimeout: 8000
```

两个值单位都是毫秒。连接超时过短会在网络抖动时误伤，读超时过长又会把调用方线程占住。按下游 P99 再留一点余量，比一律写很大更稳妥。

## OpenFeign日志打印功能

### 概念

Feign 可以按接口打出 HTTP 细节，用来确认实际路径、请求头和响应体是否和接口定义一致。它看的是代理发出去的请求，不是下游业务日志。

### 日志级别

- NONE：不输出，默认值
- BASIC：方法、URL、状态码、耗时
- HEADERS：在 BASIC 上增加请求头和响应头
- FULL：再增加正文和元数据

生产环境通常停在 BASIC。需要核对鉴权头时再用 HEADERS。FULL 会把报文体打进日志，只适合联调，示例里没有打开它。

```java
@Configuration
public class BillingFeignLog {
    @Bean
    public Logger.Level billingFeignLoggerLevel() {
        return Logger.Level.HEADERS;
    }
}
```

### 修改YML文件

日志级别要精确到接口的全限定名，并且用 debug。配到包上会把同包其它类也打出来。

```yaml
logging:
  level:
    shop.checkout.client.BillingClient: debug
```
