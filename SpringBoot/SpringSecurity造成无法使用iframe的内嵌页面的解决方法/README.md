# Spring Security 造成 Eureka 无法使用 iframe 的内嵌页面的解决方法

## 前言

后台想把注册中心嵌进来，页面只放了一个框架：

```html
<section class="panel">
  <iframe title="registry" :src="registryUrl" width="100%" height="640"></iframe>
</section>
```

```javascript
export default {
  data() {
    return { registryUrl: "http://127.0.0.1:8761/" };
  }
};
```

框架区域是空白的。开发者工具的网络面板里，文档请求其实成功了，控制台却拒绝绘制：响应头 `X-Frame-Options` 的值是 `deny`。这是 Spring Security 的默认策略，意思是任何站点都不许把本响应放进框架。

## X-Frame-Options

这个响应头用来降低点击劫持的风险。攻击者把目标站放进一个透明框架，盖在自己的按钮上，访客以为在点诱饵页面，实际点到了框架里的转账或授权。浏览器看到头之后，决定要不要渲染被嵌入的文档。响应里完全没有这个头时，只能靠更新的 `Content-Security-Policy: frame-ancestors` 来约束，否则旧浏览器会允许任意嵌套。

取值常见三种：

- `DENY`：拒绝所有框架，包括自己的域名。
- `SAMEORIGIN`：只允许同一站点的页面来嵌。
- `ALLOW-FROM uri`：只允许列出的源。多数现代浏览器已经忽略它，需要限制来源时应改用 CSP 的 `frame-ancestors`。

可以先用 curl 看注册中心现在回了什么头，再决定改哪一档：

```text
curl -sI http://127.0.0.1:8761/ | grep -i frame
```

若输出里是 `X-Frame-Options: DENY`，浏览器就会丢弃框架里的文档，地址栏却不一定报错，所以只看页面空白会误判成前端没写对。

同源的后台（同样是 `127.0.0.1`，只是端口不同）在旧规范里并不算 SAMEORIGIN，端口不同也算跨源。因此即便管理端和注册中心都在本机，默认的 DENY 或收紧后的 SAMEORIGIN 仍可能挡住 iframe。这种情况要么走反向代理把两者收成同一主机同一端口，要么在确认只有内网能访问注册中心之后，再关闭这个响应头。

## 解决方案

Security 默认等价于 `DENY`，所以 Eureka 的页面无法出现在管理端的框架里。确认管理端和注册中心的嵌套是预期行为后，在安全配置里关掉这项头。下面用新的 `SecurityFilterChain` 写法，效果与旧的 `WebSecurityConfigurerAdapter#configure` 相同：

```java
@Configuration
@EnableWebSecurity
public class RegistrySecurityConfig {

    @Bean
    public SecurityFilterChain registryChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())
            .sessionManagement(sm -> sm.sessionCreationPolicy(SessionCreationPolicy.NEVER))
            .headers(headers -> headers.frameOptions(frame -> frame.disable()))
            .authorizeHttpRequests(auth -> auth.anyRequest().authenticated())
            .httpBasic(Customizer.withDefaults());
        return http.build();
    }
}
```

`httpBasic` 要保留，这样客户端仍能用 `http://用户:密码@主机:端口/eureka/` 这种地址注册；改成表单登录后，URL 里的账号不会生效。

关掉 `X-Frame-Options` 等于允许任意网站嵌这个端口。若管理端和注册中心同源，用 `frame.sameOrigin()` 比完全关闭更合适。改完重启，框架里应能看到注册表。
