# Swagger-ui.html 页面出现 404

## 前言

文档页突然打不开，浏览器只有 404。依赖、`Docket` 和接口上的 `@ApiOperation` 都还在，另一套用同样依赖的服务却正常。差别只在近期为了校验登录，给这个服务加了 MVC 拦截器。

## 排查过程

日志里有一句很关键：

日志大意是：名为 dispatcherServlet 的分发器没有为 `/swagger-ui.html` 找到处理器映射。

普通控制器靠 `@RequestMapping` 注册处理器；`swagger-ui.html` 不是控制器方法，而是打在 springfox 依赖里的静态文件，路径在 `classpath:/META-INF/resources/`。没有映射，说明资源处理器没把这个位置挂上，而不是注解写错。

拦截器注册如下，路径是全部请求：

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

容易误判成“拦截器把静态文件拦掉了”。拦截器只能决定进不进控制器，不会删除资源映射。真正会让 Boot 放弃默认资源映射的，是存在一个 `WebMvcConfigurationSupport` 类型的 Bean：`WebMvcAutoConfiguration` 上有 `@ConditionalOnMissingBean(WebMvcConfigurationSupport.class)`，一旦有人继承了这个支持类，自动配置整体后退，`WebMvcAutoConfiguration.WebMvcAutoConfigurationAdapter` 里的 `addResourceHandlers` 也不会执行。

仅仅 `implements WebMvcConfigurer` 不会触发上面的条件，默认映射理论上还在。但项目里如果另外写过 `addResourceHandlers`，并用 `/**` 只指向 `classpath:/static/`，这条更宽的模式会抢在 springfox 自带的映射前面，文档页和 webjars 就都落空。补上文档页实际所在的位置即可：

```java
@Override
    public void addResourceHandlers(ResourceHandlerRegistry registry)
    {
        registry.addResourceHandler("/assets/**")
                .addResourceLocations("classpath:/" + "static/");
        registry.addResourceHandler("/swagger-ui.html")
                .addResourceLocations("classpath:/META-INF/" + "resources/");
        registry.addResourceHandler("/webjars/**")
                .addResourceLocations("classpath:/META-INF/" + "resources/webjars/");
    }
```

这里故意不用 `/**` 去覆盖整个站点，避免再次把默认链盖住。重启后文档页应能打开。若打开 HTML 之后脚本 404，再核对 `/webjars/**` 是否被安全配置要求登录。拦截器若对未登录请求直接重定向，也要把 `/swagger-ui.html`、`/webjars/**`、`/swagger-resources/**` 放进 `excludePathPatterns`，否则资源映射对了，仍会被拦到登录页。
