# Spring Security 在单机 Spring Boot 中的用法

这一篇把前两篇的认证和授权搬到 Spring Boot 2.1 的单进程工程里。技术组合是 Boot、Spring Security、MySQL、MyBatis 和 JSP。单机指认证结果放在本进程的会话中，还没有跨服务的令牌。

## 第一版：默认安全配置

### 工程骨架

父 POM 固定 Boot 版本，先只引入 Web：

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>2.1.12.RELEASE</version>
</parent>

<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
</dependencies>
```

一个返回文本的控制器：

```java
@RestController
@RequestMapping("/catalog")
public class CatalogController {
    @GetMapping
    public String ping() {
        return "catalog-ok";
    }
}
```

启动类：

```java
@SpringBootApplication
public class DeskApplication {
    public static void main(String[] args) {
        SpringApplication.run(DeskApplication.class, args);
    }
}
```

此时没有安全依赖，`/catalog` 可以直接访问。

### 加入安全起步依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

重启后，所有地址都要求认证。Boot 的自动配置提供了一个内存用户，用户名是 `user`。口令在启动日志里打印一行 `Using generated security password`，每次重启都会变。浏览器会看到框架生成的登录页，通过后再回到 `/catalog`。

还没有数据库，也没有自己的 `WebSecurityConfigurerAdapter`。这是自动配置的默认行为，用来确认依赖已经生效。生产环境不要依赖这行日志里的口令。

## 第二版：自定义登录页

Boot 官方不鼓励在内嵌容器里用 JSP，但仍然可以。需要 Jasper 来编译 JSP，并且用能够加载 `src/main/webapp` 的方式启动。只执行启动类、由默认的可执行 JAR 运行时，JSP 路径经常找不到。

### 依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-tomcat</artifactId>
</dependency>
<dependency>
    <groupId>org.apache.tomcat.embed</groupId>
    <artifactId>tomcat-embed-jasper</artifactId>
</dependency>
<dependency>
    <groupId>javax.servlet</groupId>
    <artifactId>jstl</artifactId>
</dependency>
```

把打包方式改成 `war`，在 `src/main/webapp` 下放 `sign-in.jsp`、`home.jsp`、`login-failed.jsp` 和静态目录。IDE 若把工程识别成普通 JAR，`webapp` 不会被当作 Web 根，需要在 POM 里声明 `packaging` 为 `war`。

登录表单的 action 指向 `/sign-in`，退出表单指向 `/sign-out`。CSRF 若没有关闭，表单里仍要带令牌；这一版为了先打通页面，在配置里暂时关闭，后面接数据库时再决定是否打开。

### 视图解析

```properties
spring.mvc.view.prefix=/WEB-INF/views/
spring.mvc.view.suffix=.jsp
```

控制器返回逻辑名 `catalog-list` 时，才会转到对应的 JSP。直接放在 `webapp` 根下、用转发 URL 访问的页面，不经过这个前缀。

### 安全配置类

Boot 里用 Java 配置代替 XML。`WebSecurityConfigurerAdapter` 在 2.1 仍是入口。

```java
@Configuration
@EnableWebSecurity
@EnableGlobalMethodSecurity(securedEnabled = true)
public class DeskSecurityConfig extends WebSecurityConfigurerAdapter {

    @Bean
    public BCryptPasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.authorizeRequests()
                .antMatchers("/sign-in.jsp", "/login-failed.jsp",
                        "/assets/**").permitAll()
                .antMatchers("/catalog/**").hasRole("USER")
                .anyRequest().authenticated()
            .and()
                .formLogin()
                .loginPage("/sign-in.jsp")
                .loginProcessingUrl("/sign-in")
                .defaultSuccessUrl("/home.jsp", true)
                .failureUrl("/login-failed.jsp")
            .and()
                .logout()
                .logoutUrl("/sign-out")
                .invalidateHttpSession(true)
                .logoutSuccessUrl("/sign-in.jsp")
            .and()
                .csrf().disable();
    }
}
```

`hasRole("USER")` 匹配的是 `ROLE_USER`。`defaultSuccessUrl` 的第二个参数为 true 时，总是进首页，而不是回到登录前的地址。`logout().logoutSuccessUrl` 只能调用一次，重复设置时以后一次为准。

控制器改成返回视图名：

```java
@Controller
@RequestMapping("/catalog")
public class CatalogController {
    @GetMapping("/list")
    public String list() {
        return "catalog-list";
    }
}
```

用 Maven 的 `spring-boot:run` 或 IDE 的 Tomcat 插件启动，先看到自定义登录页。这一版还没接 `UserDetailsService` 时，登录仍会失败，因为自动配置的内存用户被适配器覆盖后不复存在。下一步必须接上用户来源。

## 第三版：数据库认证

库表沿用前几篇的账号、角色、账号角色关联。演示库可以叫 `desk_access`。

### 依赖

MyBatis 的起步依赖不要写两遍。驱动版本跟 Boot 2.1 管理的版本走即可，下面显式写出是为了和旧服务器上的 5.x 协议对齐：

```xml
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>5.1.49</version>
</dependency>
<dependency>
    <groupId>tk.mybatis</groupId>
    <artifactId>mapper-spring-boot-starter</artifactId>
    <version>2.1.5</version>
</dependency>
```

### 数据源

```properties
spring.datasource.url=jdbc:mysql://127.0.0.1:3306/desk_access?useSSL=false
spring.datasource.username=desk
spring.datasource.password=${DESK_DB_PASSWORD}
spring.datasource.driver-class-name=com.mysql.jdbc.Driver
```

口令放在环境变量或本地未提交的配置里，不要写进仓库。

启动类扫描 Mapper：

```java
@SpringBootApplication
@MapperScan("lab.desk.mapper")
public class DeskApplication {
    public static void main(String[] args) {
        SpringApplication.run(DeskApplication.class, args);
    }
}
```

### 角色与用户

角色实现 `GrantedAuthority`，`getAuthority` 返回带 `ROLE_` 的编码，后面的 `hasRole` 才能对上。

```java
@Data
public class DeskRole implements GrantedAuthority {
    private Integer id;
    private String code;
    private String label;

    @Override
    public String getAuthority() {
        return code;
    }
}
```

用户实现 `UserDetails`。四个状态方法先固定返回 true，需要停用账号时再把 `status` 映射到 `isEnabled`。集合字段不必参与 JSON，单机 JSP 场景影响不大，但接口若直接把实体写回响应，应忽略口令和角色的内部结构。

```java
@Data
public class DeskAccount implements UserDetails {
    private Integer id;
    private String username;
    private String password;
    private Integer status;
    private List<DeskRole> roles;

    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        return roles == null ? List.of() : roles;
    }

    @Override
    public boolean isAccountNonExpired() { return true; }

    @Override
    public boolean isAccountNonLocked() { return true; }

    @Override
    public boolean isCredentialsNonExpired() { return true; }

    @Override
    public boolean isEnabled() { return status == null || status == 1; }
}
```

### 查询

按账号主键取出角色：

```java
public interface DeskRoleMapper extends Mapper<DeskRole> {
    @Select("""
        SELECT r.id, r.code, r.label
        FROM desk_role r
        JOIN desk_account_role ar ON r.id = ar.role_id
        WHERE ar.account_id = #{accountId}
        """)
    List<DeskRole> findByAccountId(Integer accountId);
}
```

用户查询用嵌套查询把角色集合填上。列 `id` 既是主键，也是第二次查询的参数。

```java
public interface DeskAccountMapper extends Mapper<DeskAccount> {
    @Select("SELECT id, username, password, status FROM desk_account WHERE username = #{username}")
    @Results({
        @Result(id = true, property = "id", column = "id"),
        @Result(property = "roles", column = "id",
                many = @Many(select = "lab.desk.mapper.DeskRoleMapper.findByAccountId"))
    })
    DeskAccount findByUsername(String username);
}
```

### 认证服务

```java
@Service
public class DeskAccountService implements UserDetailsService {
    @Autowired
    private DeskAccountMapper accountMapper;

    @Override
    public UserDetails loadUserByUsername(String username) {
        DeskAccount account = accountMapper.findByUsername(username);
        if (account == null) {
            throw new UsernameNotFoundException(username);
        }
        return account;
    }
}
```

### 把编码器接到认证上

```java
@Autowired
private DeskAccountService accountService;

@Override
protected void configure(AuthenticationManagerBuilder auth) throws Exception {
    auth.userDetailsService(accountService).passwordEncoder(passwordEncoder());
}
```

库里的 `password` 列必须已经是 BCrypt 摘要。用库里的登录名提交表单，成功后进入 `home.jsp`，失败进入失败页。编码器漏挂时，明文和摘要永远对不上，表现是任何正确口令都被拒绝。

## 方法级授权

`@EnableGlobalMethodSecurity(securedEnabled = true)` 已经写在配置类上。给货品列表加上角色要求：

```java
@Secured("ROLE_ADMIN")
@GetMapping("/list")
public String list() {
    return "catalog-list";
}
```

只有 `ROLE_USER` 的账号再访问该地址，会抛出 `AccessDeniedException`。默认响应是 403 白页。用控制器通知把它转到自己的页面：

```java
@ControllerAdvice
public class DeskExceptionAdvice {
    @ExceptionHandler(AccessDeniedException.class)
    public String denied() {
        return "redirect:/denied.jsp";
    }

    @ExceptionHandler(RuntimeException.class)
    public String fallback() {
        return "redirect:/error.jsp";
    }
}
```

再次访问应落在 `denied.jsp`。若仍然是白页，多半是异常发生在过滤器而不是方法代理里：URL 规则已经拒绝的请求进不了 `@Secured`。把 `/catalog/**` 的 `hasRole` 放宽为 `authenticated`，让方法注解成为唯一的角色检查，就能观察到这条跳转。

到这里，单机工程具备了数据库认证、表单登录和方法级授权。多台机器不共享会话时，下一篇改为携带签名令牌，不再把 `SecurityContext` 放进 HttpSession。
