# MVC 工程里的认证与授权

这一篇接在自定义登录和数据库用户之后，补上账号状态、注销、记住我、菜单显示和方法级授权。工程仍是传统的 Spring MVC 加 JSP，安全配置放在根容器里。

## 账号状态

`User` 有一个七个参数的构造器。除了用户名、口令和权限，还有四个布尔值。三参数构造器把它们都设为 true，所以平时感觉不到。

- `enabled`：账号是否可用
- `accountNonExpired`：账号是否仍在有效期内
- `credentialsNonExpired`：口令是否仍在有效期内
- `accountNonLocked`：账号是否未被锁定

任一值为 false，`DaoAuthenticationProvider` 都会拒绝登录，并抛出对应的异常子类。业务状态可以在组装 `UserDetails` 时映射过去。例如后台账号用一个枚举表示启用或停用：

```java
public static SecurityAccount from(AdminRecord record) {
    boolean enabled = record.getState() == AccountState.OPEN;
    return new SecurityAccount(
            record.getId(),
            record.getLoginName(),
            record.getSecretHash(),
            enabled,
            toAuthorities(record.getRoleCodes()));
}
```

停用的账号即使口令正确也不能进入系统。不要只在页面上隐藏登录按钮，检查必须发生在 `UserDetails` 的这些标志上。

## 注销

CSRF 打开时，注销只接受 POST。用链接发 GET 会被 `CsrfFilter` 拒绝，看起来像“点了退出却没反应”。在页头放一个小表单，把令牌一并提交：

```jsp
<form action="${pageContext.request.contextPath}/sign-out" method="post">
    <security:csrfInput/>
    <button type="submit">退出</button>
</form>
```

`logout-url` 与这里的 action 保持一致。成功后跳到登录页，会话里的 `SecurityContext` 被清掉。

## 记住我

### 框架在登录成功时做什么

认证成功后，`AbstractAuthenticationProcessingFilter` 会回调记住我服务。`AbstractRememberMeServices.loginSuccess` 先看请求里有没有约定的参数。参数名默认是 `remember-me`，值必须是 `true`、`on`、`yes` 或 `1` 之一。没有这个参数就直接返回，不会写 Cookie。

参数存在时，持久化实现 `PersistentTokenBasedRememberMeServices` 会生成一对系列号和令牌值，把摘要写入服务端，再把系列号和令牌放进 Cookie。

### 页面上的复选框

```jsp
<input type="checkbox" name="remember-me" value="true"/> 在这台设备上保持登录
```

`name` 和 `value` 写错，服务端会认为用户没有勾选。

只加复选框仍然不够。自动配置并不会启用 `RememberMeAuthenticationFilter`。要在 `http` 里显式打开，并给出令牌有效秒数：

```xml
<security:http auto-config="true" use-expressions="true">
    <security:remember-me token-validity-seconds="3600"/>
</security:http>
```

这个过滤器在后续请求里查看 Cookie。上下文中还没有认证对象时，调用自动登录。成功则写入 `SecurityContext`，用户不必再填表单。有效期设得太短时，关掉浏览器再打开仍要登录，那是过期，不是过滤器没装上。

### 两种实现的安全差别

最简单的实现把用户名等信息编码进 Cookie。Cookie 留在浏览器里，一旦被同机的其他程序读走，就等于带走了登录材料。即使做了编码，把和身份强相关的数据放在客户端也不可取。使用这个功能的用户应在离开时主动注销，让服务端作废会话。

另一种实现是持久化令牌：Cookie 里只放系列号和随机令牌，与口令无关。服务端保存令牌摘要和用户的对应关系。自动登录时用 Cookie 里的值去服务端核对。令牌被使用后可以轮换，旧值失效。这不能防止 Cookie 被窃后的重放窗口，但不再把口令材料放进浏览器，正式环境应采用这一档。

### 持久化到数据库

Spring Security 的 JDBC 实现要求固定表结构，列名不能改：

```sql
CREATE TABLE persistent_logins (
    username  varchar(64) NOT NULL,
    series    varchar(64) NOT NULL,
    token     varchar(64) NOT NULL,
    last_used timestamp   NOT NULL,
    PRIMARY KEY (series)
);
```

`series` 是主键，用来在轮换令牌时找到那一行。配置里指定数据源：

```xml
<security:remember-me data-source-ref="dataSource"
                      token-validity-seconds="3600"
                      remember-me-parameter="remember-me"/>
```

勾选并登录成功后，表中应多出一行。看不到行，先确认数据源就是应用在用的那个库，以及登录请求真的带了参数。注销时框架会删掉对应用户的行，旧 Cookie 不能再自动登录。

## 显示当前登录名

标签库从安全上下文读取主体，两种写法都可以，前者取 `UserDetails` 的用户名，后者取认证对象的 name：

```jsp
<span>
    <security:authentication property="principal.username"/>
</span>
<span>
    <security:authentication property="name"/>
</span>
```

匿名访问时主体是匿名用户的名字，不是空。页面若对未登录用户也渲染这段，会看到匿名标识，需要先用 `authorize` 标签包起来。

## 授权前的两个示例功能

为了观察授权，准备两个控制器：订单列表和货品列表，侧栏各放一个入口。此时还没有方法级限制，只是普通的请求映射。

## 动态菜单不是授权

侧栏的每个入口用标签按角色显示：

```jsp
<security:authorize access="hasRole('USER')">
    <a href="${pageContext.request.contextPath}/orders">订单</a>
</security:authorize>
<security:authorize access="hasRole('STOCK')">
    <a href="${pageContext.request.contextPath}/catalog">货品</a>
</security:authorize>
```

用只有 `ROLE_USER` 的账号登录，侧栏只剩订单。直接在地址栏请求货品列表的 URL，页面仍然打开。标签只影响 HTML 是否输出，请求到达服务器时 `FilterSecurityInterceptor` 并没有一条针对该 URL 的规则。

结论：按角色隐藏菜单改善的是界面，不是权限边界。真正的限制必须写在 URL 规则或方法注解上。

## 方法级授权

### 注解支持要放在正确的容器

Spring MVC 里存在父子容器。安全配置和 `UserDetailsService` 放在父容器（根容器），控制器由子容器扫描。注解是否生效，取决于创建那个 Bean 的容器有没有打开对应的支持。注解放在控制器上，就在 MVC 的配置文件里开启；注解放在 Service 上，就在根配置里开启。只在其中一个文件里打开，另一个容器里的 Bean 不会被代理。

安全配置本身不要放到子容器。子容器里的 Bean 对父容器可见性是单向的，过滤器链在根容器启动，找不到子容器里的认证管理器。

### 三类注解

演示可以都打开，工程里选定一类即可：

```xml
<security:global-method-security
    jsr250-annotations="enabled"
    pre-post-annotations="enabled"
    secured-annotations="enabled"/>
```

- `secured-annotations`：Spring Security 自己的 `@Secured`
- `jsr250-annotations`：标准的 `@RolesAllowed`，需要对应的 API 包
- `pre-post-annotations`：`@PreAuthorize` / `@PostAuthorize`，里面写 SpEL

控制器上三选一：

```java
@Controller
@RequestMapping("/catalog")
public class CatalogController {

    @PreAuthorize("hasAnyAuthority('ROLE_STOCK','ROLE_ADMIN')")
    @RequestMapping("/list")
    public String list() {
        return "catalog-list";
    }
}
```

`@Secured({"ROLE_STOCK","ROLE_ADMIN"})` 和 `@RolesAllowed` 表达同一条要求，只是注解来源不同。`@PreAuthorize` 能写更复杂的表达式，例如同时检查方法和参数。没有相应角色的用户再访问 `/catalog/list`，会得到 403，而不是一张空菜单。

表达式里的 `hasRole('STOCK')` 会补 `ROLE_`，`hasAuthority('ROLE_STOCK')` 不会。角色码在数据库、注解和 URL 规则三处要用同一种写法。

## 权限不足时的页面

默认的 403 是容器或框架的空白错误页，需要换成自己的提示。

### 在安全配置里指定

`ExceptionTranslationFilter` 捕获 `AccessDeniedException` 后，按这里的页面转发：

```xml
<security:http auto-config="true" use-expressions="true">
    <security:access-denied-handler error-page="/denied.jsp"/>
</security:http>
```

### 在 web.xml 里指定

```xml
<error-page>
    <error-code>403</error-code>
    <location>/denied.jsp</location>
</error-page>
```

这是容器级的错误页，不经过 Spring MVC 的视图解析。适合静态提示页。

### 用异常解析器

过滤器和拦截器的差别顺便记一笔：过滤器在 Servlet 容器里，先于 Spring MVC；拦截器在 DispatcherServlet 内部。方法注解抛出的 `AccessDeniedException` 发生在控制器调用期间，可以被 Spring 的异常解析器看到。过滤器直接写回的 403 则不一定进得了解析器。

实现 `HandlerExceptionResolver`：

```java
@Component
public class AccessFailureResolver implements HandlerExceptionResolver {
    @Override
    public ModelAndView resolveException(HttpServletRequest request,
                                         HttpServletResponse response,
                                         Object handler,
                                         Exception ex) {
        ModelAndView mv = new ModelAndView();
        if (ex instanceof AccessDeniedException) {
            mv.setViewName("redirect:/denied.jsp");
        } else {
            mv.setViewName("redirect:/error.jsp");
        }
        return mv;
    }
}
```

用 `@ControllerAdvice` 更短，效果等价于注册了一个解析器：

```java
@ControllerAdvice
public class AccessFailureAdvice {
    @ExceptionHandler(AccessDeniedException.class)
    public String denied() {
        return "redirect:/denied.jsp";
    }

    @ExceptionHandler(RuntimeException.class)
    public String other() {
        return "redirect:/error.jsp";
    }
}
```

两种方式都只覆盖进入 MVC 的异常。URL 规则在 `FilterSecurityInterceptor` 里拒绝时，仍然要靠 `access-denied-handler` 或容器错误页。页面上隐藏入口、方法注解、过滤器规则，是三层不同的东西，缺了后两层，直接请求 URL 仍然能进到功能。
