# 前后端分开部署时，浏览器为什么拦请求

## 一个真实会碰到的画面

本地界面用构建工具起在 `9528`，接口进程听在 `8601`。页面一加载，控制台就出现跨域失败：脚本所在的源和接口的源不是同一个。即便后来把两边收进同一个域名，裁剪图片时仍可能再踩一次。图片若是由反向代理从磁盘映射出来的，响应里往往没有允许跨源的头；同一张图如果改由已经带好这些头的对象存储输出，裁剪组件就能读到像素。现象是点了裁剪，画布是空的。

## 浏览器在查什么

同源指协议、主机、端口三者一致。页面从另一个组合去取资源，就是跨源请求。脚本发起的 `fetch` 或 `XMLHttpRequest` 默认被同源策略拦住；图片、样式、脚本标签能直接加载别的主机，但画布一旦读取像素，同样要检查响应头。

服务器用一组响应头声明“这个源可以读我”：

- `Access-Control-Allow-Origin`：允许哪个源。带凭证时不能是星号，必须是具体源。
- `Access-Control-Allow-Methods` / `Access-Control-Allow-Headers`：预检通过的方法与请求头。
- `Access-Control-Allow-Credentials`：是否允许携带 cookie。
- `Access-Control-Max-Age`：预检结果可以缓存多久。

所谓预检，是浏览器在真正的 `PUT`、`DELETE` 或带自定义头的请求之前，先发一个 `OPTIONS`。预检失败，后面的业务请求不会发出。

Spring 把这套规则交给 `DefaultCorsProcessor`。处理顺序可以收成下面几步：

1. 请求没有 `Origin`，或者源和当前请求相同，就不算跨源，原样继续。
2. 响应里已经写过允许源的头，说明前面的过滤器处理过了，不再改。
3. 找不到匹配的跨源配置时：如果这是预检，直接拒绝；如果是普通请求，则不写头、继续交给控制器。浏览器侧仍会因为缺头而丢掉脚本对响应的访问。
4. 有配置时，逐项核对源、方法、请求头是否落在 `CorsConfiguration` 里。全部通过才把允许头写进响应；有一项不通过就拒绝。

## 接口这一侧怎么放行

### 用 MVC 的映射注册

在启动配置里登记路径和允许的源。需要把 cookie 交给页面时，源必须写成明确地址，并打开凭证。星号和凭证不能同时出现，较新的 Spring 会直接拒绝这种组合。

```java
    @Bean
    public WebMvcConfigurer crossOriginSetup() {
        WebMvcConfigurer configurer = new WebMvcConfigurer() {
            @Override
            public void addCorsMappings(CorsRegistry mappings) {
                String anyHeader = "*";
                mappings.addMapping("/api/**")
                        .allowedOriginPatterns("http://localhost:9528")
                        .allowedMethods("GET", "POST", "PUT", "DELETE", "OPTIONS")
                        .allowedHeaders(anyHeader)
                        .allowCredentials(Boolean.TRUE)
                        .maxAge(1800);
            }
        };
        return configurer;
    }
```

### 用一个专用过滤器

另一种写法是自行组装配置并注册到路径上：

```java
CorsConfiguration buildLooseConfig() {
    CorsConfiguration config = new CorsConfiguration();
    config.addAllowedOriginPattern("http://localhost:9528");
    config.addAllowedHeader("*");
    config.addAllowedMethod("*");
    config.setAllowCredentials(true);
    return config;
}

    @Bean
    public CorsFilter crossOriginFilter() {
        UrlBasedCorsConfigurationSource rules = new UrlBasedCorsConfigurationSource();
        rules.registerCorsConfiguration("/**", buildLooseConfig());
        CorsFilter filter = new CorsFilter(rules);
        return filter;
    }
```

过滤器和 MVC 映射不要各写一套互相矛盾的规则。过滤器更早执行，登录、刷新令牌这种在过滤器里写 cookie 的逻辑，必须由过滤器上的配置来放行。

### 令牌要写进 cookie 时

接近过期就换发令牌，是常见需求。服务端在剩余时间短于窗口时生成新令牌，写到响应的 `Set-Cookie`，并顺手延长缓存里的会话。

```java
if (online != null && !tokens.expired(raw)) {
    long remain = tokens.remainSeconds(raw);
    if (remain < refreshWindowSeconds) {
        String renewed = tokens.renew(raw, ttlSeconds);
        String cookie = "session-token=" + renewed
                + "; Path=/; HttpOnly; SameSite=None; Secure; Max-Age=" + ttlSeconds;
        response.addHeader("Set-Cookie", cookie);
        sessions.expire("login:" + renewed, Duration.ofSeconds(ttlSeconds));
    }
}
filterChain.doFilter(request, response);
```

页面侧要让浏览器在跨源请求里带上 cookie，也要接受对方写回的 cookie：

```javascript
const client = axios.create({
  baseURL: "http://localhost:8601",
  withCredentials: true,
  timeout: 8000
});
```

`withCredentials` 默认是 `false`。同源时改不改影响不大；跨源时设为 `true`，请求会带上凭证，响应里的 `Set-Cookie` 也才可能被保存。设为 `false` 时，浏览器既不附带 cookie，也会忽略跨源响应里想写入的 cookie。

只做了第二种过滤器、却忘了 `setAllowCredentials(true)` 时，前端一把开关打开，登录就会失败：响应里没有 `Access-Control-Allow-Credentials: true`，或者允许源仍是星号，浏览器会丢弃整份响应。把凭证开关和明确的源一起配上之后，新令牌才能出现在站点的 cookie 里。另外，跨站 cookie 现在通常还要 `SameSite=None` 并且走 HTTPS，本地 HTTP 调试要单独放宽。

## 裁剪图片时被拦住

图片文件放在应用机器上，由反向代理做静态映射。页面和图片主机不同，裁剪库要用画布读取像素，于是触发跨源检查。代理默认不会加允许头。

在图片所在的 `location` 里补上。注意 `if` 在这里会形成嵌套位置，外层的 `add_header` 不会自动进入这个分支，预检分支里要把头再写一遍：

```nginx
location /media/ {
    if ($request_method = OPTIONS) {
        add_header Access-Control-Allow-Origin $http_origin always;
        add_header Access-Control-Allow-Methods "GET, OPTIONS" always;
        add_header Access-Control-Allow-Headers "Content-Type" always;
        add_header Access-Control-Max-Age 1800 always;
        return 204;
    }
    add_header Access-Control-Allow-Origin $http_origin always;
    add_header Access-Control-Allow-Methods "GET, OPTIONS" always;
}
```

改完后重新加载代理进程。若图片源会随环境变化，用 `$http_origin` 回写请求里的源，比写死星号更适合以后要带凭证的画布。生产上应再校验这个源是否在白名单里，避免任意网站都能读你的图片字节。
