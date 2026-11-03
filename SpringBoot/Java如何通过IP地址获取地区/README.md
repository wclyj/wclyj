# 按 IP 地址判断地区

## 思路

公网地址被登记成一段段区间，每段对应国家、省份、城市和运营商。要给访问日志补上地区，不必每次去请求第三方接口：把离线库（常见的是 ip2region 的 xdb）跟应用一起发布，进程内按区间检索即可，延迟稳定，也不把访客地址送出机房。

检索前先决定“哪一个 IP 代表用户”。直连时用 `request.getRemoteAddr()`；前面有反向代理时，真实地址通常写在 `X-Forwarded-For` 的第一段，但这个头可以被调用方伪造。

## 示例

启动时把库读进内存，查询方法只做两件事：解析客户端地址，再查库。

```java
@Component
public class RegionLookup {
    private final Searcher searcher;

    public RegionLookup() throws IOException {
        byte[] blob = StreamUtils.copyToByteArray(
                new ClassPathResource("ip2region.xdb").getInputStream());
        this.searcher = Searcher.newWithBuffer(blob);
    }

    public String locate(HttpServletRequest request) {
        String ip = clientIp(request);
        if (isPrivate(ip)) {
            return "局域网";
        }
        try {
            // 形如 中国|0|浙江|杭州|电信
            return searcher.search(ip);
        } catch (Exception ex) {
            return "未知";
        }
    }

    private String clientIp(HttpServletRequest request) {
        String remote = request.getRemoteAddr();
        if (!trustedProxy(remote)) {
            return remote;
        }
        String forwarded = request.getHeader("X-Forwarded-For");
        if (forwarded == null || forwarded.isBlank()) {
            return remote;
        }
        return forwarded.split(",")[0].trim();
    }
}
```

`trustedProxy` 只放行本机网关或负载均衡的地址；`isPrivate` 覆盖 `10/8`、`172.16/12`、`192.168/16` 和回环。控制器或日志切面里调用 `locate`，把返回串按 `|` 拆开再入库。

## 坑

内网、回环和 IPv6 映射地址（`::ffff:192.168.1.8`）用公网库往往查不到，应当先归一化并标成局域网，不要因为空结果再去打外部 HTTP 接口，否则一次内网调试就会把线程堵在超时上。库文件几个月不更新时，新划分的地址段会被标到旧城市，查询失败要吞掉并记成“未知”，避免地区服务拖垮下单或登录主流程。
