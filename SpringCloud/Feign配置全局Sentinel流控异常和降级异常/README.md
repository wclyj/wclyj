# Feign 配置全局 Sentinel 流控异常和降级异常

## 前言

给声明式客户端接上 Sentinel 之后，限流或熔断一旦命中，框架默认只会甩出一行英文提示，类似：

```text
Blocked by Sentinel (flow limiting)
```

这行字既没有业务错误码，也进不了统一响应包装。前端只能当成未知失败。更合适的做法是在 Feign 这一层把两类问题分开接住：服务不可用时走降级兜底，规则命中时走流控兜底，两边都返回项目自己的 JSON。

## Feign全局兜底异常处理

规则也可以下发到 Nacos，但那只解决“规则丢了要重配”的问题，并不负责把异常翻译成业务报文。单个接口可以用 `@SentinelResource` 指定处理方法，例如封面查询：

```java
@GetMapping("/cover/hot")
@SentinelResource(value = "coverHot", blockHandler = "onCoverBlocked")
public String coverHot(String sku, String region) {
    return coverService.locate(sku, region);
}

public String onCoverBlocked(String sku, String region, BlockException ex) {
    return ReplyKit.fail("封面查询被限流");
}
```

接口一多，每个方法都配一个兜底，改动面会散落在控制器里。更干净的位置是 Feign 接口本身的 `fallback`。

已有远程接口 `CoverFeignClient`，在注解上挂上兜底类：

```java
/**
 * 封面中心远程接口。
 */
@FeignClient(name = "atlas-cover", fallback = CoverFeignFallback.class)
public interface CoverFeignClient {

    @GetMapping("/asset/meta")
    String fetchMeta(@RequestParam("assetIds") String assetIds,
                     @RequestParam("token") String token);

    @PostMapping("/asset/importByLink")
    String importByLink(CoverBundle bundle);
}
```

兜底类实现同一套方法。封面中心进程挂掉、超时或抛出业务异常时，Feign 会把调用转到这里，而不是把异常原样抛给调用方：

```java
@Slf4j
@Component
public class CoverFeignFallback implements CoverFeignClient {

    @Override
    public String fetchMeta(String assetIds, String token) {
        String path = CtxHolder.current().getRequestURI();
        log.error("封面中心不可用，元数据查询已降级, path={}", path);
        return ReplyKit.fail("封面元数据暂时不可用");
    }

    @Override
    public String importByLink(CoverBundle bundle) {
        String path = CtxHolder.current().getRequestURI();
        log.error("封面中心不可用，链接导入已降级, path={}", path);
        return ReplyKit.fail("封面导入暂时不可用");
    }
}
```

`ReplyKit.fail` 负责拼统一错误体，这样降级结果和普通业务失败长得一样。

还要在配置里打开 Sentinel 对 Feign 的适配，否则 `fallback` 不会被触发：

```yaml
feign:
  sentinel:
    enabled: true
```

## Feign处理全局流控异常

上一节处理的是“对端不可用或业务抛错”。另一类是控制台或规则文件里配出来的保护规则，常见有五种：

- 接口被 QPS 或线程数限流
- 接口被熔断降级
- 热点参数被单独限流
- 授权规则拒绝了来源
- 系统保护规则拦住了入口流量

它们抛出的都是 `BlockException` 的子类，默认文案仍是 `Blocked by Sentinel (...)`。要按类型翻译成中文提示，可以实现 `UrlBlockHandler`，让适配层在写响应之前把异常接走。

```java
@Slf4j
@Component
public class FeignGuard implements UrlBlockHandler {

    @Override
    public void blocked(HttpServletRequest req, HttpServletResponse resp, BlockException ex)
            throws IOException {
        String tip;
        if (ex instanceof FlowException) {
            tip = "接口触发限流";
        } else if (ex instanceof DegradeException) {
            tip = "接口触发熔断";
        } else if (ex instanceof ParamFlowException) {
            tip = "热点参数触发限流";
        } else if (ex instanceof AuthorityException) {
            tip = "来源未通过授权规则";
        } else if (ex instanceof SystemBlockException) {
            tip = "系统保护规则已生效";
        } else {
            tip = "请求被流量防护拦截";
        }
        log.error("{}, uri={}", tip, req.getRequestURI());
        resp.setCharacterEncoding("UTF-8");
        resp.setContentType("application/json");
        resp.getWriter().write(ReplyKit.fail(tip));
    }
}
```

这个类必须落在启动类能扫到的包里。封面服务的启动类把 Feign 接口包和防护包一起纳入扫描：

```java
@SpringBootApplication
@EnableDiscoveryClient
@EnableFeignClients("shop.atlas.shared.feign")
@ComponentScan({
        "shop.atlas.shared.feign",
        "shop.atlas.shared.guard"
})
public class CoverBoot {
    public static void main(String[] args) {
        SpringApplication.run(CoverBoot.class, args);
    }
}
```

这样，降级走 Feign 的 `fallback`，限流、熔断、热点、授权和系统规则走 `UrlBlockHandler`。调用方拿到的都是统一错误体，不必再解析框架默认的英文句子。
