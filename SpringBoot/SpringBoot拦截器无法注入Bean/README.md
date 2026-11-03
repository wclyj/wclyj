# SpringBoot 拦截器无法注入 Bean

## 前言

给图片服务加一道拦截器，想在进控制器之前读出令牌，并从缓存取出用户。字段上写了注入：

```java
@Autowired
private CacheClient cacheClient;
```

请求一进来就是空指针。不是工具类没写进容器，而是这个拦截器实例根本不是容器创建的。

`AutowiredAnnotationBeanPostProcessor` 会在初始化阶段扫描容器里的 Bean，看到 `@Autowired` 就按类型（或 `@Qualifier`）找匹配对象并写回字段。它只处理自己管辖的实例。注册拦截器时如果写成 `new`，对象在容器外面出生，后处理器看不到它，字段就一直是 null。

```java
@Configuration
public class MvcSetup implements WebMvcConfigurer {

    @Override
    public void addInterceptors(InterceptorRegistry registry)
    {
        registry.addInterceptor(new TokenGate()).addPathPatterns("/**");
    }
}
```

`TokenGate` 的生命周期在这行 `new` 就结束了，后面的 `@Autowired` 不会再被触发。

## 解决办法

更干净的做法是让拦截器自己成为 Bean，配置类再注入它：

```java
@Component
public class TokenGate implements HandlerInterceptor {
    private final CacheClient cacheClient;

    public TokenGate(CacheClient cacheClient) {
        this.cacheClient = cacheClient;
    }
}
```

```java
@Configuration
public class MvcSetup implements WebMvcConfigurer {
    private final TokenGate tokenGate;

    public MvcSetup(TokenGate tokenGate) {
        this.tokenGate = tokenGate;
    }

    @Override
    public void addInterceptors(InterceptorRegistry registry)
    {
        registry.addInterceptor(tokenGate).addPathPatterns("/api/**");
    }
}
```

如果拦截器必须手写 `new`（例如构造参数来自外部配置、暂时改不动注册方式），就从已经刷新好的上下文里按类型取出依赖。下面的持有者在容器回调 `setApplicationContext` 时把引用存下来：

```java
@Component
public class ContextHolder implements ApplicationContextAware {

    private static ApplicationContext context;

    public static <T> T bean(Class<T> type) {
        return context.getBean(type);
    }

    public static <T> T bean(String name, Class<T> type) {
        return context.getBean(name, type);
    }

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) {
        context = applicationContext;
    }
}
```

拦截器里改成显式查找，不要再依赖字段注入：

```java
@Override
public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) {
    CacheClient cacheClient = ContextHolder.bean(CacheClient.class);
    String token = request.getHeader("Authorization");
    request.setAttribute("loginUser", cacheClient.findUser(token));
    return true;
}
```

查找发生在请求阶段，此时上下文一定已经完成刷新。不要在拦截器的构造器里调用 `bean()`，那时候静态字段可能还是 null。持有者本身必须被组件扫描到，否则 `setApplicationContext` 不会执行。
