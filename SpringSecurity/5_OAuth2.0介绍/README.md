# OAuth 2.0

示例对准 Spring Security OAuth 在 Boot 2.1 时代的授权服务器与资源服务器。这套 `spring-security-oauth2` 项目后来停止演进，新工程应迁移到 Spring Authorization Server。角色划分和四种授权方式仍然适用，类名和端点细节要以所选用的库为准。

## 概念

OAuth 是一套授权框架，名字来自 Open Authorization。它让用户批准某个第三方在有限范围、有限时间内访问自己存放在另一处的资源，而第三方不需要拿到用户的口令。

OAuth 2.0 是后续版本，不兼容 1.0。下面说的授权码、简化模式等，都指 2.0。

## 它和单点登录不是同一件事

假设印坊要打印用户放在相册服务里的照片。传统做法是用户把相册的口令交给印坊，印坊用这套口令登录相册。口令一旦交出，印坊就能做口令能做的一切，用户也很难只收回打印这一项权利。

OAuth 改成：用户在相册的认证端点上批准“印坊可以读取照片”。相册发给印坊一枚访问令牌。令牌的范围只有读取，而且会过期。印坊拿不到相册口令。

单点登录解决的是“同一个人少输几次口令，去访问互相信任的多个系统”。OAuth 解决的是“一个系统在用户同意后，访问另一个系统里的资源”。Spring Security 的 OAuth 组件两种都能搭：用它做服务间受控共享，也可以把认证端点当成统一登录入口。不要因为都出现了令牌，就把两种需求写成同一个流程。

参与方有四个角色：

- 资源拥有者：通常是用户
- 客户端：想访问资源的应用，例如印坊
- 授权服务器：认证用户、征求同意、颁发令牌
- 资源服务器：存放照片或别的 API，检验令牌后才返回数据

## 四种授权方式

### 授权码模式

客户端是有后端的 Web 应用。令牌不经过浏览器地址栏。

1. 客户端把用户浏览器重定向到授权服务器，带上 `client_id`、`redirect_uri`、`response_type=code`，以及自己生成的 `state`。
2. 用户在授权服务器上登录，并同意客户端所请求的范围。用户必须本来就有资源服务器的账号。
3. 授权服务器把浏览器重定向回 `redirect_uri`，查询参数里带上一次性的授权码。授权码不能拿来调用资源 API。
4. 客户端的后端用授权码、客户端身份和同一个 `redirect_uri`，向令牌端点换取访问令牌。
5. 授权服务器返回访问令牌，以及可选的刷新令牌。

这是四种里面最完整的一种，浏览器上的第三方登录（社交账号登录到自己的站点）通常走它。`state` 由客户端生成并在回调时核对，用来把这次回调和自己发出的那次重定向对上。

### 简化模式

没有独立后端，整个过程在浏览器里完成，也不再发授权码。授权服务器直接把访问令牌放在回调 URI 的片段（`#` 后面）里交给前端。片段不会被浏览器发到服务器日志里，但令牌仍然出现在用户代理中，容易被同页脚本读到。

步骤是：前端把用户带到授权服务器，`response_type=token`，并带上 `redirect_uri` 和 `state`；用户登录并同意；授权服务器重定向回来，令牌在片段中。

它适合没有后端的纯前端插件。现在更推荐授权码加上 PKCE，而不是把令牌直接暴露给浏览器。理解简化模式，是为了看懂旧系统和文档，不是为了在新系统里优先选用。

### 密码模式

资源拥有者把自己在授权服务器上的用户名和口令告诉客户端，客户端拿这两项再加上自己的客户端身份，向令牌端点换令牌。

流程短，但用户的口令会经过客户端。只有客户端和授权服务器同属一个高度信任的整体（例如同一家的官方第一方应用）才考虑。第三方应用不应该使用这种模式。示例里会演示端点参数，不表示它适合作为对外接口。

### 客户端模式

客户端用自己的 `client_id` 和 `client_secret` 换令牌，流程里没有用户。令牌代表的是客户端自己，不是某个资源拥有者。

适合服务之间读取与具体用户无关的数据，例如一个批处理任务拉取公共配置。它已经离开了“用户授权给第三方”的原始场景，但协议仍把它列为一种 grant。

## 授权服务器使用的表

演示可以把客户端写死在内存里。真正对外的授权服务器要让第三方自行注册，注册信息落在库里，而不是等资源方改代码。下面的表来自 Spring Security OAuth 的 JDBC 实现，名字是框架约定的。

官方建表脚本在该项目的 `schema.sql` 中。实验库只建框架真正读写的那些列即可，有些文章里的归档标记、信任标记是后来扩展的，不在默认脚本里。

### oauth_client_details

一行代表一个客户端，是注册信息的核心。

| 列 | 作用 |
| --- | --- |
| client_id | 主键。公开的客户端标识，注册时指定 |
| resource_ids | 允许访问的资源服务器编号，逗号分隔，要和资源服务声明的 id 一致 |
| client_secret | 客户端密钥。应存编码器产生的摘要，而不是明文 |
| scope | 允许申请的范围，例如 `read,write`。资源服务器用它做方法级限制 |
| authorized_grant_types | 允许的授权方式，逗号分隔。不在列表里的 grant 会被拒绝 |
| web_server_redirect_uri | 回调地址。授权码和简化模式会核对，防止把码或令牌送到预先登记之外的地址 |
| authorities | 客户端自己的权限，可选 |
| access_token_validity | 访问令牌秒数，空则用框架默认 |
| refresh_token_validity | 刷新令牌秒数，空则用框架默认 |
| additional_information | 可选的附加 JSON，协议本身不用 |
| autoapprove | 为 `true` 或某个 scope 时，跳过用户确认页。只应对第一方、高度信任的客户端打开 |

回调地址在授权码模式的两步里都要一致：换授权码时带上的 `redirect_uri`，和用授权码换令牌时再带的 `redirect_uri`，都必须落在这列登记的值里。简化模式则把令牌放在该地址的片段中返回。

### oauth_client_token

给“作为 OAuth 客户端的那个应用”存它从别处拿回的令牌。授权服务器自己的 JDBC 实现通常不写这张表，客户端库才会用到。

### oauth_access_token

保存已经颁发的访问令牌。默认实现把令牌值和认证信息序列化后入库，资源服务器用同一张表校验。主要访问类是 `JdbcTokenStore`。进程重启后，库里的令牌仍然有效；若改用内存存储，重启即失效。

默认脚本里的列可以按用途记：

- `token_id`：对令牌值做提取后的标识，作为查找键
- `token`：序列化后的访问令牌本身
- `authentication_id`：把客户端、用户名和范围合在一起算出的标识，用来判断是不是同一次授权
- `user_name`、`client_id`：便于按人或按客户端清理
- `authentication`：序列化后的认证对象，资源服务器靠它还原权限
- `refresh_token`：指向配套的刷新令牌标识，没有刷新令牌时为空

这些列由框架写入，应用代码不要直接 UPDATE 令牌字节。作废一台客户端的全部访问令牌时，按 `client_id` 删除行即可，同时删掉对应的刷新令牌行。只清空 `token` 列会让查询抛出反序列化错误，表现得像随机 500。

### oauth_refresh_token

保存刷新令牌。同样由 `JdbcTokenStore` 读写。客户端的 grant 列表里没有 `refresh_token` 时，这张表不会有新行。刷新令牌的有效期应长于访问令牌，但它本身也要能作废。

列比访问令牌少：`token_id`、序列化后的 `token`，以及序列化后的 `authentication`。没有单独的用户名列时，要清理某人的刷新令牌，得从访问令牌表的 `refresh_token` 列反查，或者在反序列化认证对象里看用户名。所以作废时先删访问令牌行，再按标识删刷新令牌，避免留下还能换新访问令牌的记录。

### oauth_code

授权码表只有 `code` 和序列化的 `authentication`。授权码模式把用户已经同意的范围暂存在这里，兑换成功后删除该行。表一直增长，说明兑换没有发生，多半是回调地址拿到码之后没有走服务器端的换令牌请求，码就留在库里直到过期清理。简化模式、密码模式和客户端模式不写这张表。

### oauth_code

保存尚未兑换的授权码。读写类是 `JdbcAuthorizationCodeServices`。只有授权码模式会写它。授权码应短命，并且兑换一次就失效，避免重放。

## 实战：资源服务与授权服务

父工程使用 Boot 2.1 和 Greenwich 一代的 Spring Cloud 依赖管理，子模块是 `print-resource` 与 `album-auth`。Cloud 的依赖管理只为了对齐 `spring-cloud-starter-oauth2` 的版本，示例并不使用注册中心。

资源模块的关键依赖是 Web、Security、`spring-cloud-starter-oauth2` 2.1、MySQL 和 MyBatis。

### 资源服务

端口 9102，数据源指向同一实验库，因为访问令牌存在库里，资源服务要能读到 `oauth_access_token`。

```yaml
server:
  port: 9102
spring:
  datasource:
    url: jdbc:mysql://127.0.0.1:3306/desk_access?useSSL=false
    username: desk
    password: ${DESK_DB_PASSWORD}
  main:
    allow-bean-definition-overriding: true
```

`allow-bean-definition-overriding` 用来允许用自己声明的 `TokenStore` 覆盖自动配置放进容器的同名对象。

只读接口先放一个占位：

```java
@RestController
@RequestMapping("/photos")
public class PhotoApi {
    @GetMapping
    public String list() {
        return "photo-list";
    }
}
```

只引入 Security 时，这个地址会被默认登录页挡住。下一步不是再做一遍会话登录，而是把服务标成资源服务器，改由访问令牌决定是否放行。用户和角色类仍然需要，因为令牌里的认证信息会还原成 `UserDetails` 上的权限；可以从单机篇复制，包名改到资源模块。

资源配置：

```java
@Configuration
@EnableResourceServer
public class PhotoResourceConfig extends ResourceServerConfigurerAdapter {
    @Autowired
    private DataSource dataSource;

    @Bean
    public TokenStore tokenStore() {
        return new JdbcTokenStore(dataSource);
    }

    @Override
    public void configure(ResourceServerSecurityConfigurer resources) {
        resources.resourceId("photo-api").tokenStore(tokenStore());
    }

    @Override
    public void configure(HttpSecurity http) throws Exception {
        http.authorizeRequests()
                .antMatchers(HttpMethod.GET, "/**").access("#oauth2.hasScope('read')")
                .antMatchers(HttpMethod.POST, "/**").access("#oauth2.hasScope('write')")
                .antMatchers(HttpMethod.PUT, "/**").access("#oauth2.hasScope('write')")
                .antMatchers(HttpMethod.DELETE, "/**").access("#oauth2.hasScope('write')")
            .and()
                .cors();
    }
}
```

`resourceId` 必须出现在客户端注册信息的 `resource_ids` 里，否则令牌即使签名正确也不会被这个资源接受。`#oauth2.hasScope` 检查的是令牌上的 scope，不是用户角色。查询要求 `read`，改变数据的方法要求 `write`。角色检查仍可以叠加 `@Secured`，两者同时满足才放行。

跨域若由独立网关统一处理，这里不要再写允许任意来源。示例把 CORS 交给 Spring 的 `cors()` 配置，具体允许哪些源写在 `CorsConfiguration` 里，而不是对所有来源反射请求头。任意来源加上携带凭证，会把资源暴露得过宽。

令牌的存储还可以是内存或 Redis。多实例资源服务必须共享同一份存储，否则一台上颁发的令牌在另一台上查不到。JDBC 是示例选择，因为它和授权服务器共用一个库，部署简单。

### 授权服务

端口 9101，依赖与资源服务相同。把账号查询、`UserDetailsService` 和口令编码器接上，用户登录仍然走原来的账号表。客户端信息不在账号表里。

Spring Security 这一侧只负责“人如何登录”：

```java
@Configuration
@EnableWebSecurity
public class AlbumUserSecurity extends WebSecurityConfigurerAdapter {
    @Autowired
    private AccountService accountService;

    @Bean
    public BCryptPasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }

    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth.userDetailsService(accountService).passwordEncoder(passwordEncoder());
    }

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.authorizeRequests().anyRequest().authenticated()
            .and().formLogin().permitAll()
            .and().csrf().disable();
    }

    @Bean
    @Override
    public AuthenticationManager authenticationManagerBean() throws Exception {
        return super.authenticationManagerBean();
    }
}
```

把 `AuthenticationManager` 暴露成 Bean，是因为密码模式要它来校验资源拥有者的口令。授权码模式的登录页同样走这套表单登录。

OAuth 的端点由另一个配置类打开：

```java
@Configuration
@EnableAuthorizationServer
public class AlbumAuthConfig extends AuthorizationServerConfigurerAdapter {
    @Autowired private DataSource dataSource;
    @Autowired private AuthenticationManager authenticationManager;

    @Bean
    public JdbcClientDetailsService clientDetails() {
        return new JdbcClientDetailsService(dataSource);
    }

    @Bean
    public TokenStore tokenStore() {
        return new JdbcTokenStore(dataSource);
    }

    @Bean
    public ApprovalStore approvalStore() {
        return new JdbcApprovalStore(dataSource);
    }

    @Bean
    public AuthorizationCodeServices codeServices() {
        return new JdbcAuthorizationCodeServices(dataSource);
    }

    @Override
    public void configure(ClientDetailsServiceConfigurer clients) throws Exception {
        clients.withClientDetails(clientDetails());
    }

    @Override
    public void configure(AuthorizationServerSecurityConfigurer security) {
        security.allowFormAuthenticationForClients()
                .checkTokenAccess("isAuthenticated()");
    }

    @Override
    public void configure(AuthorizationServerEndpointsConfigurer endpoints) {
        endpoints.approvalStore(approvalStore())
                .authenticationManager(authenticationManager)
                .authorizationCodeServices(codeServices())
                .tokenStore(tokenStore());
    }
}
```

`allowFormAuthenticationForClients` 允许客户端把 `client_id` 和 `client_secret` 放在表单里，而不只是 HTTP Basic。`checkTokenAccess` 限制谁可以调用检查令牌的端点，示例要求调用者已经认证。授权码、同意记录和令牌都走 JDBC，重启后仍在。

### 注册一个客户端

向 `oauth_client_details` 插入印坊这个客户端。`client_secret` 用与授权服务相同的 BCrypt 编码器生成摘要后再写入。`resource_ids` 写 `photo-api`。`authorized_grant_types` 先把五种都打开，便于逐个对照；收紧时删掉不需要的项。回调只登记自己的地址，例如 `http://127.0.0.1:8080/oauth/callback`，不要登记一个与本应用无关的站点。

`autoapprove` 保持 `false`，这样授权码模式会停在确认页，能看见用户正在批准哪些 scope。

## 按模式走一遍

下列地址都是框架的标准端点。实验时用浏览器或接口工具访问本机，不要把客户端密钥写进前端脚本。

### 授权码

浏览器打开：

```text
http://127.0.0.1:9101/oauth/authorize?response_type=code&client_id=print-shop&redirect_uri=http://127.0.0.1:8080/oauth/callback&scope=read&state=desk-state-1
```

未登录时会先进入账号表的登录页，这里用的是资源拥有者的账号，不是客户端密钥。登录后出现同意页，分别列出申请的范围，可以同意或拒绝。同意后浏览器跳到回调地址，查询参数中有 `code` 和原来的 `state`。客户端必须先核对 `state`。

后端再向令牌端点发起一次自身的请求，带上 `grant_type=authorization_code`、`code`、`redirect_uri`，并用客户端身份认证。响应 JSON 里有 `access_token`、`token_type`、`expires_in`，若注册了刷新令牌还会有 `refresh_token`。授权码用过即失效。

带着 `Authorization: Bearer <access_token>` 访问资源服务的 `GET /photos`。scope 含 `read` 且资源 id 匹配时返回正文。

### 简化模式

把 `response_type` 改成 `token`，其余查询参数类似。用户同意后，回调地址的片段里直接出现 `access_token`，没有单独的换码步骤。前端取出后用同样的头访问资源服务。观察地址栏就能理解为什么它不适合不可信环境：令牌出现在用户代理里。

### 密码模式

向 `/oauth/token` 提交 `grant_type=password`，以及资源拥有者的用户名、口令、`scope`，客户端身份放在 Basic 或表单中。授权服务器用前面暴露的 `AuthenticationManager` 校验口令，成功则发令牌。这一步会让客户端看到用户口令，只在第一方应用的实验里使用。

### 客户端模式

`grant_type=client_credentials`，只提供客户端身份和 scope，没有用户名字段。颁发的令牌没有资源拥有者。资源服务器若还要求用户角色，这种令牌会在角色检查处失败；示例的资源规则只检查 scope，因此可以读到 `GET /photos`。这不等于客户端模式适合读取某个用户的私有照片。

### 刷新令牌

访问令牌过期前，用 `grant_type=refresh_token` 加上 `refresh_token` 的值，再次调用令牌端点。客户端身份仍然要带。成功时返回新的访问令牌。注册信息里没有 `refresh_token` 这种 grant 时，端点会拒绝。客户端模式默认不发刷新令牌，因为它不代表一个需要长期续期的用户会话。

四种模式换到的访问令牌，资源服务器的校验路径是同一条：读库里的令牌记录，核对 scope 和 resource id。差别只在令牌是怎么颁发的。收紧某个客户端时，修改的是 `authorized_grant_types` 和 `scope`，而不是在资源服务器上为每种模式各写一个过滤器。

## 令牌响应里有什么

不论哪种 grant，令牌端点成功时返回的 JSON 形状相同，方便资源服务器不必关心颁发方式：

```json
{
  "access_token": "8f3c0b1e-example",
  "token_type": "bearer",
  "expires_in": 43199,
  "scope": "read",
  "refresh_token": "b71e-example"
}
```

`token_type` 为 bearer 时，资源请求使用 `Authorization: Bearer` 加上 `access_token` 的值。`expires_in` 是秒。没有刷新令牌的模式不会出现 `refresh_token` 字段。这些值来自授权服务器的颁发逻辑，客户端应原样保存，不要自己拆开令牌猜用户身份；这套 JDBC 实现里的访问令牌常常是不透明的标识，不是前面一篇里的 JWT。资源服务器靠查库还原认证信息，所以两台资源服务器必须连同一份令牌表。

客户端密钥在库里是 BCrypt 摘要时，调用令牌端点仍提交原始密钥，由 `JdbcClientDetailsService` 做比对。把摘要本身当成密钥再提交，会一直失败。生成摘要可以用授权服务里的那个 `PasswordEncoder` 写一个一次性的小程序，不要把编码器换成与登录用户不同的算法，否则客户端和用户两套口令对不齐。

## 同意页与范围

授权码模式在 `autoapprove` 为 false 时会停在同意页。页面上的每一项对应一个 scope。用户拒绝全部范围时，回调里带来的是错误参数，而不是授权码，客户端要能处理“用户取消”，不要把错误参数再拿去换令牌。

范围字符串在客户端注册、授权请求和资源规则三处要使用同一套词。注册了 `read`，请求时却写 `photo.read`，授权服务器会拒绝。资源上的 `#oauth2.hasScope('read')` 看的是最终写进令牌的范围，不是用户在账号表里的角色名。一个常见的混淆是：用户角色决定“这个人能不能管理相册”，scope 决定“这枚令牌能不能读或写”。印坊拿到的令牌即使属于管理员账号，没有 `write` 也不能调用删除接口。

同意记录落在 `oauth_approvals`（由 `JdbcApprovalStore` 使用）。同一用户对同一客户端已经同意过的范围，下次可以不再询问，具体是否跳过还受 `autoapprove` 影响。撤销授权时要删掉对应的同意记录和仍未过期的令牌，只删客户端行不会使已经发出的访问令牌立刻失效。

## 换令牌时各模式要带的参数

令牌端点都是授权服务上的 `POST /oauth/token`。客户端身份用 HTTP Basic，用户名是 `client_id`，密码是原始密钥；若打开了表单客户端认证，也可以放在表单字段里。两种不要各写一半。

- 授权码：`grant_type=authorization_code`，再加上 `code` 和与申请授权码时相同的 `redirect_uri`
- 密码模式：`grant_type=password`，再加上资源拥有者的 `username`、`password` 和 `scope`。这是协议里的字段名，口令只应出现在第一方后端发往授权服务器的这一次请求中
- 客户端模式：`grant_type=client_credentials`，加 `scope`
- 刷新：`grant_type=refresh_token`，加 `refresh_token`

简化模式不访问这个端点，令牌已经在重定向片段里。授权码若被用第二次，端点返回无效授权，这是预期行为，客户端应重新走浏览器授权，而不是重试同一个码。

资源服务器拒绝时先分清三件事：令牌在表里不存在或过期，是认证失败；令牌在，但 scope 不含 `read`，是范围不足；`resource_ids` 里没有 `photo-api`，是这枚令牌根本不是发给当前资源的。三种都可能表现为 401 或 403，看响应里的错误码比只看 HTTP 数字更省时间。

## 和上一篇 JWT 方案怎么选

上一篇的资源服务拿公钥本地验签，不访问认证服务，也不查令牌表。这一篇的资源服务每次都按令牌值查库。本地验签延迟更低，令牌在过期前难以单独作废；查库可以立刻删掉一行让令牌失效，但资源服务依赖数据库。需要随时吊销时用这一篇的不透明令牌；需要资源服务无状态水平扩展、且可以接受短有效期时，用上一篇的签名令牌。两种不要叠在同一个过滤器链里各写一半，否则一个请求既要会话、又要 JWT、又要查 OAuth 表，失败时很难判断是哪一层拒绝的。

注册客户端之后先用客户端模式打通令牌端点和资源端点，这一步不需要浏览器，也不需要用户口令，适合确认库表、`resource_ids` 和 scope 是否一致。这条链路通了，再测授权码，才能把“回调地址配错”和“令牌表根本写不进去”分开。授权码测试失败时，先看 `oauth_code` 有没有新行：没有，说明浏览器那一步没完成；有，说明换令牌的服务器端请求没发出或被拒绝。`state` 由客户端自己生成并在回调时比对，对不上就丢弃这次 `code`，不要继续兑换。授权服务器不替客户端保存 `state` 的含义，它只是原样带回。回调地址必须与注册值逐字符一致，多一个斜杠或换了端口都会在发码之前被拒绝，此时 `oauth_code` 里不会有新行。登记多个回调时用逗号分隔，请求里带的那一个必须是其中之一，不能临时改成未登记的地址。本地和测试环境的地址都要事先写入，不要等联调时再改表，否则已经发出的授权请求仍按旧地址校验。改完表后不用重启进程，下一次授权就会读到新行。插入后用查询确认 `client_id` 只有一行，重复插入会在主键上直接失败。
