# 初识 Spring Security

## 权限管理概念

权限管理是按事先定好的规则，决定一个调用方能碰到哪些资源。凡是要区分访客和登录用户的系统都会碰到它。讨论时把两件事分开：

- 认证：用用户名和口令（或别的凭证）通过校验之后，系统知道当前主体是谁、带有哪些角色。
- 授权：拿着这个身份，判断某次访问是否被允许。

身份没建立之前谈不上授权。认证失败和授权失败对用户的提示也不该混用。登录被拒绝时，调用方要改凭证；已经登录但被拒绝时，要改的是角色或规则，重复提交同一口令没有意义。

把这两步拆开还有一个实现上的好处：认证可以集中在一个服务里做，授权可以留在持有资源的服务上。后面分布式那一篇就是按这个边界切的。单机工程里它们仍在同一个进程，只是过滤器不同。

## 权限模型里的三类数据

- 用户：登录名、口令摘要，以及他所隶属的角色。认证阶段读取它。
- 角色：角色名、说明，以及这个角色被授予的权限集合。授权阶段读取它。
- 权限：可以是一条菜单或一个 URL。前端用它决定画哪些入口，服务端用它决定放不放行。

用户和角色是多对多，角色和权限也是多对多。用户不直接挂权限，中间一定经过角色。这样调整一个人的能力时，改的是角色分配，而不是在用户记录上堆一长串 URL。

## 初识 Spring Security

Spring Security 用 AOP 的思路，把检查放在 Servlet 过滤器链上，而不是散落在每个控制器里。认证机制和到方法一级的授权都由这套过滤器完成。应用代码只提供“用户从哪来”和“哪些地址需要什么角色”。

### 创建工程

和 Web 工程相关的构件可以按职责记：

- `spring-security-core`：认证、授权的核心模型，任何用法都离不开它。
- `spring-security-web`：过滤器和 Web 层的基础设施。
- `spring-security-config`：解析 XML 或支撑 Java 配置。
- `spring-security-taglibs`：给 JSP 用的标签，用来按权限显示片段。

Maven 里引入 config 与 taglibs 后，core 和 web 会作为传递依赖出现。版本与 Spring 主版本对齐，下面的示例用 5.1 这一代：

```xml
<dependency>
    <groupId>org.springframework.security</groupId>
    <artifactId>spring-security-config</artifactId>
    <version>5.1.5.RELEASE</version>
</dependency>
<dependency>
    <groupId>org.springframework.security</groupId>
    <artifactId>spring-security-taglibs</artifactId>
    <version>5.1.5.RELEASE</version>
</dependency>
```

### 配置 web.xml

容器里只登记一个过滤器。名字必须是 `springSecurityFilterChain`，实现类是 Spring 的 `DelegatingFilterProxy`。它自己不做安全检查，只是按这个名字到 Spring 容器里找真正的过滤器。

```xml
<filter>
    <filter-name>springSecurityFilterChain</filter-name>
    <filter-class>org.springframework.web.filter.DelegatingFilterProxy</filter-class>
</filter>
<filter-mapping>
    <filter-name>springSecurityFilterChain</filter-name>
    <url-pattern>/*</url-pattern>
</filter-mapping>
```

### 一份最小的安全配置

`auto-config="true"` 让框架装上登录、注销等默认过滤器。`use-expressions="true"` 之后，`access` 里才能写 SpEL。

```xml
<security:http auto-config="true" use-expressions="true">
    <security:intercept-url pattern="/**"
        access="hasAnyRole('USER','ADMIN')"/>
</security:http>

<security:authentication-manager>
    <security:authentication-provider>
        <security:user-service>
            <security:user name="clerk" password="{noop}clerk-pass"
                           authorities="ROLE_USER"/>
            <security:user name="root" password="{noop}root-pass"
                           authorities="ROLE_ADMIN"/>
        </security:user-service>
    </security:authentication-provider>
</security:authentication-manager>
```

`{noop}` 是 `DelegatingPasswordEncoder` 的前缀，表示这段口令按明文比较，只适合演示。`hasAnyRole('USER')` 会自动补上 `ROLE_` 前缀，所以用户上的权限要写成 `ROLE_USER`。表达式若改用 `hasAnyAuthority`，则必须把完整字符串写上，框架不再加前缀。

内存用户启动后，访问任意地址都会被带到框架自带的登录页。用上面两个账号可以分别走通。

## 过滤器各自做什么

整条链是典型的拦截思想：每个过滤器只做一件事，做完交给下一个。默认自动配置里，和入门相关的过滤器按大致顺序如下。

### SecurityContextPersistenceFilter

链的最前面。它在请求开始时从 `SecurityContextRepository`（通常是 HttpSession）取出 `SecurityContext`，放进 `SecurityContextHolder`，请求结束前再写回去。后面的过滤器读到的当前用户，都来自这个上下文。上下文里放的是认证结果和权限，不是原始口令。

### WebAsyncManagerIntegrationFilter

如果控制器里使用了 Spring 的异步处理，这个过滤器把当前的 `SecurityContext` 传给 `WebAsyncManager`，避免异步线程里身份丢失。

### HeaderWriterFilter

向响应写入安全相关的头，例如阻止页面被嵌进外站框架。XML 里可以用 `headers` 元素调整，主要影响浏览器里的页面响应。

### CsrfFilter

跨站请求伪造指的是：浏览器会自动带上已经登录站点的 Cookie，另一个网站上的页面可以趁机让浏览器向你的站点发一个状态改变请求。过滤器的做法不是去猜请求从哪来，而是要求改变状态的请求带上一个只有本站页面才拿得到的同步令牌。令牌对不上就拒绝。GET 这类只读请求默认不检查。

### LogoutFilter

匹配注销地址（默认 `/logout`）。命中后清除认证信息、使会话失效，再按配置跳转。

### UsernamePasswordAuthenticationFilter

表单认证的入口。默认匹配 `/login`，且只接受 POST。它从请求里取出用户名和口令，交给 `AuthenticationManager`。

### DefaultLoginPageGeneratingFilter

配置里没有指定登录页时，由它生成一张简单的 HTML 表单，里面包含用户名、口令，以及 CSRF 隐藏域。

### DefaultLogoutPageGeneratingFilter

在没有自定义注销页时，生成一个发起注销的默认页面。

### BasicAuthenticationFilter

查看 `Authorization` 头。若以 `Basic ` 开头，就解析其中的用户名和口令并尝试认证。浏览器弹出的原生登录框走的是这条路径，和表单登录不是同一个过滤器。

### RequestCacheAwareFilter

内部用 `RequestCache` 记住认证之前用户原本想访问的地址。登录成功后可以回到那个地址，而不是总进首页。

### SecurityContextHolderAwareRequestFilter

包装 `ServletRequest`，让 `request.isUserInRole(...)` 这类 Servlet API 能读到 Spring Security 里的权限。

### AnonymousAuthenticationFilter

上下文里还没有认证对象时，放进一个匿名身份。这样后面的授权过滤器不必到处判断 null，未登录访问也可以走同一套决策，只是身份是匿名的。把某些地址设为 `permitAll`，实质是允许这个匿名身份通过。

### SessionManagementFilter

处理会话策略，例如限制同一用户同时存在多少个会话、检测会话固定。

### ExceptionTranslationFilter

放在授权过滤器之前，把链后部抛出的认证异常、权限不足异常翻译成跳转登录页或 403 响应。业务过滤器不用自己捕获这些异常。

### FilterSecurityInterceptor

链的末尾。它读取 `intercept-url` 或注解上的访问规则，对照 `SecurityContextHolder` 里的权限决定放行或拒绝。资源能不能访问，最终在这里拍板，而不是在页面上把菜单藏起来。

## 过滤器链是怎样装上来的

`web.xml` 里只写了一个 `DelegatingFilterProxy`，上面十几个过滤器并没有逐个登记。加载过程可以分成三层来看。

### DelegatingFilterProxy

代理在第一次请求时按过滤器名字 `springSecurityFilterChain` 向根 Spring 容器要 Bean。拿到的对象类型是 `FilterChainProxy`。之后每次 `doFilter` 都转给它。名字写错或 Bean 不在根容器里，启动后第一下访问就会失败。

### FilterChainProxy

`FilterChainProxy` 按请求 URL 选择一条 `SecurityFilterChain`。一份配置里可以有多条 `http` 块，例如静态资源一条、业务地址一条。选中之后，它按顺序调用那条链里的过滤器。调试时在这里能看到自动配置放进来的那一串过滤器实例。

### SecurityFilterChain

这是接口，默认实现持有一个请求匹配器和一个过滤器列表。`web.xml` 里那个名字对应的，其实是外层的 `FilterChainProxy`；真正“哪些 URL 走哪些过滤器”记录在一条条 `SecurityFilterChain` 上。

### 小结

理解这条链，是为了知道登录页、403 和匿名访问分别是哪个过滤器的行为，而不是为了替换框架内部类。下一步通常是换成自己的登录页。

调试时在 `FilterChainProxy` 上打断点，能看到本次请求选中了哪一条 `SecurityFilterChain`，以及列表里的过滤器实例。静态资源若命中了 `security="none"` 的那条链，列表是空的，请求不会进入 CSRF 和登录过滤器。业务地址则能看到上面这一串。不要在每个过滤器里打印完整请求体，口令会进日志。

## 使用自定义登录页

在 `http` 块里用 `form-login` 指明页面、提交地址、成功和失败的去向。登录页本身必须允许匿名访问，否则会被授权过滤器挡在外面，形成循环跳转。静态资源同样单独放行，避免样式表也要求登录。

```xml
<security:http pattern="/assets/**" security="none"/>
<security:http pattern="/login-failed.jsp" security="none"/>

<security:http auto-config="true" use-expressions="true">
    <security:intercept-url pattern="/sign-in.jsp" access="permitAll()"/>
    <security:intercept-url pattern="/**" access="hasAnyRole('USER')"/>
    <security:form-login
        login-page="/sign-in.jsp"
        login-processing-url="/sign-in"
        default-target-url="/home.jsp"
        authentication-failure-url="/login-failed.jsp"/>
    <security:logout logout-url="/sign-out"
                     logout-success-url="/sign-in.jsp"/>
</security:http>

<bean id="passwordEncoder"
      class="org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder"/>

<security:authentication-manager>
    <security:authentication-provider user-service-ref="accountService">
        <security:password-encoder ref="passwordEncoder"/>
    </security:authentication-provider>
</security:authentication-manager>
```

表单的 `action` 必须等于 `login-processing-url`，方法为 POST，字段名默认是 `username` 和 `password`。页面能打开、提交后却返回 403，多半不是口令错了，而是 POST 没有带上 CSRF 令牌。框架自带的登录页会放一个隐藏域，自己画的页面若漏了这一项，`CsrfFilter` 会在认证过滤器之前就把请求拒绝掉。

## CSRF 防护在链上的位置

`CsrfFilter` 只说明机制：它从会话仓库取出预期令牌，再和请求参数或请求头里的令牌比较。一致才放行。`HttpSessionCsrfTokenRepository` 负责把令牌放进会话，并在渲染页面时提供给视图。

处理自己的登录页有两种做法。

第一种是在配置里关掉 CSRF。过滤器不再检查。这样做等于放弃对状态改变请求的这项校验，演示可以，正式环境不建议，尤其是还使用 Cookie 会话的浏览器应用。

第二种是在表单里带上令牌。JSP 标签库提供：

```jsp
<form action="${pageContext.request.contextPath}/sign-in" method="post">
    <security:csrfInput/>
    <input type="text" name="username"/>
    <input type="password" name="password"/>
    <button type="submit">登录</button>
</form>
```

标签会渲染成隐藏字段，名字和值来自当前仓库。前后端分离、不能用标签时，改为由服务端把令牌发到前端，前端放进约定的请求头。不要在日志里打印令牌。

## 用数据库完成认证

### 认证在过滤器里的走向

`UsernamePasswordAuthenticationFilter` 确认地址和方法之后，把用户名和口令装进一个未认证的 `UsernamePasswordAuthenticationToken`，调用 `AuthenticationManager.authenticate`。默认字段名就是 `username` 和 `password`，地址要和 `login-processing-url` 一致。

`AuthenticationManager` 的常见实现是 `ProviderManager`，它按顺序询问各个 `AuthenticationProvider`。表单登录会落到 `DaoAuthenticationProvider`，其父类 `AbstractUserDetailsAuthenticationProvider` 做两件事：通过 `UserDetailsService` 按用户名加载用户，再用 `PasswordEncoder` 比对口令。比对成功后构造一个已认证的令牌，权限集合来自 `UserDetails.getAuthorities()`。

因此自定义加载逻辑返回的 `UserDetails` 必须带上权限。权限为空时，人虽然“登录了”，随后的 `FilterSecurityInterceptor` 仍会拒绝几乎所有地址。

过滤器本身的 `doFilter` 在父类 `AbstractAuthenticationProcessingFilter` 里。认证成功走 `successfulAuthentication`：把结果写入 `SecurityContext`，再决定跳转。认证失败走 `unsuccessfulAuthentication`，回到失败页。成功回调里还会通知记住我服务，那是后一篇的内容。

### 实现 UserDetailsService

让业务接口继承 `UserDetailsService`，框架只认这个接口。

```java
public interface AccountService extends UserDetailsService {
    void register(Account account);
    List<Account> listAll();
    void grantRoles(Integer accountId, Integer[] roleIds);
}
```

加载方法在用户不存在时抛出 `UsernameNotFoundException`，不要返回 null，否则提供者在取口令时会遇到空指针。口令列里已经是摘要，就不要再加 `{noop}`。

```java
@Override
public UserDetails loadUserByUsername(String loginName) {
    Account account = accountDao.findByLogin(loginName);
    if (account == null) {
        throw new UsernameNotFoundException(loginName);
    }
    List<GrantedAuthority> granted = new ArrayList<>();
    for (Role role : account.getRoles()) {
        granted.add(new SimpleGrantedAuthority(role.getCode()));
    }
    return new User(account.getLoginName(), account.getSecretHash(), granted);
}
```

配置里用 `user-service-ref` 指向这个实现，替换掉内存用户。角色码要和 `intercept-url` 里的表达式对得上，例如库存的是 `ROLE_USER`。

## 加密认证

口令不能明文入库。在容器里声明 `BCryptPasswordEncoder`，并挂到 `authentication-provider` 上。BCrypt 每次编码都会带随机盐，同一口令两次编码的字符串不同，比对必须走 `matches`，不能自己做字符串相等。

注册时先编码再保存：

```java
public void register(Account account) {
    account.setSecretHash(passwordEncoder.encode(account.getRawSecret()));
    accountDao.insert(account);
}
```

已经存在的明文口令需要离线做一次迁移：读出明文，编码后写回摘要列，然后应用只走编码器比对。迁移完成前混用明文和摘要，登录会不稳定。演示数据可以直接在库里更新成编码器生成的摘要，不要把摘要算法换成可以反推原文的编码。

比对时框架调用的是 `PasswordEncoder.matches(原文, 摘要)`。BCrypt 摘要里已经包含算法版本、代价因子和盐，所以数据库不需要另建盐字段。代价因子越大，单次登录越慢，暴力尝试的成本也越高；演示用默认强度即可，不必在登录接口上把强度调到让请求超时。

若一个系统里同时存在历史 MD5 摘要和新的 BCrypt 摘要，应使用 `DelegatingPasswordEncoder`，按前缀选择算法，并在用户下次成功登录时把旧摘要升级成 BCrypt。不要在查询接口里把摘要返回给前端，也不要写“万能口令”分支来绕过编码器。
