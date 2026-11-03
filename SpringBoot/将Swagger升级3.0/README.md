# SpringBoot 项目将 Swagger 升级 3.0

## 前言

springfox 3 已经把文档模型对齐到 OpenAPI 3。升级不只是改版本号，启动类注解、UI 地址和安全放行路径都变了。

## 新特性

- 依赖树里不再夹带一组互相打架的旧传递版本，Guava 等冲突要自己再核对一次。
- 启动类上的 `@EnableSwagger2` 退出，改由 `@EnableOpenApi` 触发。
- 新增 `springfox-boot-starter`，单独引它即可，不必再拼 `swagger2` 和 `swagger-ui` 两个构件。

## 引入依赖

用 starter 取代原先散落的依赖：

坐标使用 `io.springfox:springfox-boot-starter:3.0.0`。Gradle：

```groovy
dependencies {
    implementation "io.springfox:springfox-boot-starter:3.0.0"
}
```

启动类注解同步更换：

```text
旧：@EnableSwagger2
新：@EnableOpenApi
```

## 修改配置文件

文档类型改为 `OAS_30`，仍然只收集标了 `@ApiOperation` 的方法：

```java
@Configuration
public class OpenApiConfig {

    @Bean
    public Docket openApi() {
        Docket docket = new Docket(DocumentationType.OAS_30);
        return docket.apiInfo(meta())
                .select()
                .apis(RequestHandlerSelectors.withMethodAnnotation(
                        ApiOperation.class))
                .paths(PathSelectors.regex("/.*"))
                .build();
    }

    private ApiInfo meta() {
        ApiInfoBuilder info = new ApiInfoBuilder();
        return info.title("服务接口说明")
                .description("OpenAPI 3 文档")
                .version("3.0")
                .build();
    }
}
```

3.x 的页面地址是 `/swagger-ui/index.html`，不再使用 `/swagger-ui.html`。安全配置要把新页面、旧书签、分组数据和 webjars 一并匿名放行，否则页面能开，规格 JSON 仍是 401：

```java
.antMatchers(
        "/swagger-ui.html",
        "/swagger-ui/**",
        "/swagger-resources/" + "**",
        "/v2/api-docs",
        "/v3/api-docs/**",
        "/webjars/**"
).permitAll()
```

本地验证访问 `http://localhost:8601/swagger-ui/index.html`。

需要在页面上填令牌时，给 `Docket` 补一个请求头形式的 `ApiKey`，并限定只对业务路径生效：

```java
@Bean
public Docket openApi() {
    java.util.function.Predicate<RequestHandler> marked =
            RequestHandlerSelectors.withMethodAnnotation(ApiOperation.class);
    Docket docket = new Docket(DocumentationType.OAS_30);
    return docket.apiInfo(meta())
            .select()
            .apis(marked)
            .paths(PathSelectors.ant("/api/**"))
            .build()
            .securitySchemes(List.of(new ApiKey("Authorization", "Authorization", "header")))
            .securityContexts(List.of(SecurityContext
                    .builder()
                    .securityReferences(List.of(new SecurityReference(
                            "Authorization",
                            new AuthorizationScope[]{new AuthorizationScope("global", "全部接口")})))
                    .forPaths(PathSelectors.ant("/api/**"))
                    .build()));
}
```

重启后页面右上角会出现授权入口。填入 `Bearer` 加上令牌，随后对 `/api/**` 的试用请求会自动带上 `Authorization` 头。分组接口若仍要匿名调试，不要把它们划进 `forPaths`。

## 参考

以 springfox 3 的发行说明和 Spring Security 对静态资源的放行配置为准，核对 UI 路径与 `Docket` 的文档类型是否都已经切到 OpenAPI 3。
