## 前言

Boot 和 Cloud 一起抬到 2.x 之后，注册中心先挂在两处配置上。

旧的安全开关写在顶层 `security` 下，Boot 2 不再识别：

```yaml
security:
  basic:
    enabled: true
  user:
    name: registry
    password: change-me
```

账号改挂到 `spring.security.user`：

```yaml
spring:
  security:
    user:
      name: registry
      password: change-me
```

服务端依赖的构件名也换了。坐标改为 `org.springframework.cloud:spring-cloud-starter-netflix-eureka-server`，中间多了 `netflix`。旧的 `spring-cloud-starter-eureka-server` 在后续 Release Train 里已经退役，留着旧坐标时构建会直接找不到构件。

只改名字还不够。Boot 2 拿掉了“默认打开 HTTP Basic”的那段 YAML，`security.basic.enabled` 不再有人读。不写安全配置类时，注册请求会被拦下来，客户端却按 Eureka 的实例 JSON 去解析响应体。报错看起来像 Jackson 的字段对不上：根上出现了 `timestamp`，解析器等待的是 `instance`。那是未授权页面的错误 JSON，不是实例数据。

用配置类把 Basic 认证加回来，并关掉 CSRF。注册是服务间调用，没有浏览器表单，CSRF 令牌只会让客户端注册失败。会话策略用 `NEVER`，避免为每条续约创建会话：

```java
@Configuration
@EnableWebSecurity
public class EurekaSecurityConfig extends WebSecurityConfigurerAdapter {

    @Override
    protected void configure(HttpSecurity http)
            throws Exception {
        http.csrf()
                .disable();
        http.sessionManagement()
                .sessionCreationPolicy(SessionCreationPolicy.NEVER);
        http.authorizeRequests()
                .anyRequest()
                .authenticated()
                .and()
                .httpBasic();
    }
}
```

必须是 `httpBasic`。如果换成表单登录，客户端配置里的 `http://用户:密码@主机:端口/eureka/` 不会带上凭证。

客户端的 `defaultZone` 要写上同一套账号，否则仍会被 Basic 拦住：

```yaml
eureka:
  client:
    service-url:
      defaultZone: http://registry:change-me@127.0.0.1:8761/eureka/
```

密码若含 `@` 或 `:`，先做 URL 编码，否则主机名会被截断。那种失败发生在建立连接之前，响应体里不会出现 `timestamp` 与 `instance` 对不上的解析错误。注册中心自身如果也是 Eureka 客户端，记得让它 `register-with-eureka` 和 `fetch-registry` 按需关闭，避免自己用错误的地址去登记自己。
