# 多服务环境下的认证

单机篇把认证结果放进本进程的 HttpSession。服务一旦拆成多台，各进程的会话互不可见，不能再靠会话标识串起身份。这一篇用签名令牌代替会话：认证服务签发，资源服务只做校验。

## 分布式认证指什么

这里的分布式认证就是常说的单点登录。用户在互相信任的一组应用里只登录一次，之后访问其他应用不必再次输入口令。

每台机器维护自己的会话，会话内存不能直接拿来当共享方案。可行的流程分成两段，认证服务只部署一份，资源服务可以有多份。

- 用户认证：客户端把凭证交给认证服务。通过后，认证服务返回令牌。签发只发生在这一处。
- 身份校验：客户端访问某个资源服务时带上令牌。资源服务检查签名和有效期，还原出用户和角色，再做自己的授权。资源服务不保存口令，也不签发令牌。

令牌泄漏等同于在有效期内泄漏身份，所以有效期要短，传输只走 TLS，日志里不要打印完整令牌。

## JWT 的结构

令牌选用 JSON Web Token。它是一段用点号分成三截的字符串，本身不是加密容器。

- 头部：声明类型和签名算法，例如 `alg` 为 `RS256`。
- 载荷：声明。可以放用户名、角色、过期时间 `exp`、唯一编号 `jti`。不要放口令或密钥。载荷只是 Base64URL，任何人都能解码阅读。
- 签名：对“头部.载荷”按头部指定的算法计算。RS256 用私钥签名，校验方用公钥验证。签名对不上，说明内容被改过，或者不是这把私钥签发的。

三截都能被看到，防伪造靠的是签名，不是靠把载荷藏起来。若使用 HS256，签名和校验共用一把密钥，资源服务必须持有这把密钥，密钥扩散之后任何持有者都能签发。RS256 把私钥留在认证服务，公钥分发给资源服务，校验方无法自行签发。这和“把同一段盐同时用于生成和伪造”不是一回事：对称算法的密钥必须保密，非对称算法的公钥本来就可以公开。

有些文章把 JWT 说成“私钥加密、公钥解密”。对 RS256 而言，私钥做的是签名，公钥做的是验签，载荷并没有被加密。需要保密时要另用 JWE，或者只在 TLS 上传输，不把敏感字段放进载荷。

## RSA 在这里的用法

RSA 同时产生私钥和公钥。本方案里：

- 认证服务用私钥调用签名算法，生成令牌的第三截。
- 资源服务用公钥验证第三截，并读取载荷。
- 公钥可以分发，私钥只出现在认证服务的文件系统或密钥管理服务里，权限收到运行账号可读即可。

公钥加密、私钥解密是另一条用途，用来保护一小段密钥材料。不要和签名搞混。RSA 运算慢，所以只签令牌，不拿它加密业务报文。算法名称来自 Rivest、Shamir、Adleman 三人。

密钥长度至少 2048 位。生成时用足够的随机源，不要用可预测的短字符串当唯一熵。

## 令牌工具

依赖使用 jjwt 0.10 这一代，API 和 0.11 之后的构建器略有差别，示例对准 0.10。

```xml
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-api</artifactId>
    <version>0.10.7</version>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-impl</artifactId>
    <version>0.10.7</version>
    <scope>runtime</scope>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-jackson</artifactId>
    <version>0.10.7</version>
    <scope>runtime</scope>
</dependency>
```

载荷单独成对象，避免调用方拆 `Claims`：

```java
@Data
public class TokenBody<T> {
    private String tokenId;
    private T account;
    private Date expiresAt;
}
```

签发与校验：

```java
public final class TokenCodec {
    private static final String ACCOUNT_CLAIM = "acct";

    private TokenCodec() {}

    public static String sign(Object account, PrivateKey key, int minutes) {
        return Jwts.builder()
                .claim(ACCOUNT_CLAIM, JsonBoxes.write(account))
                .setId(UUID.randomUUID().toString())
                .setExpiration(DateTime.now().plusMinutes(minutes).toDate())
                .signWith(key, SignatureAlgorithm.RS256)
                .compact();
    }

    public static <T> TokenBody<T> read(String token, PublicKey key, Class<T> type) {
        Claims claims = Jwts.parser()
                .setSigningKey(key)
                .parseClaimsJws(token)
                .getBody();
        TokenBody<T> body = new TokenBody<>();
        body.setTokenId(claims.getId());
        body.setExpiresAt(claims.getExpiration());
        Object raw = claims.get(ACCOUNT_CLAIM);
        if (raw != null && type != null) {
            body.setAccount(JsonBoxes.read(raw.toString(), type));
        }
        return body;
    }
}
```

`parseClaimsJws` 会检查签名和 `exp`。过期或签名错误时抛出 `JwtException`，调用方转为 401，不要把异常栈回给浏览器。`jti` 用随机 UUID，便于以后做黑名单；本篇的资源服务只验签，还不维护黑名单。

JSON 工具只保留签发路径会用到的两个方法：

```java
public final class JsonBoxes {
    private static final ObjectMapper MAPPER = new ObjectMapper();

    private JsonBoxes() {}

    public static String write(Object value) {
        try {
            return MAPPER.writeValueAsString(value);
        } catch (JsonProcessingException ex) {
            throw new IllegalStateException(ex);
        }
    }

    public static <T> T read(String json, Class<T> type) {
        try {
            return MAPPER.readValue(json, type);
        } catch (IOException ex) {
            throw new IllegalArgumentException(ex);
        }
    }
}
```

### 读写 PEM 之前的密钥字节

实验里把 PKCS#8 私钥和 X.509 公钥的字节做 Base64 后放在文件中。这是教学用的文件布局，生产环境更常见的是 PKCS#12 或 KMS，不要把私钥提交进仓库。

```java
public final class RsaFiles {
    private RsaFiles() {}

    public static PublicKey readPublic(Path path) throws Exception {
        byte[] der = Base64.getDecoder().decode(Files.readAllBytes(path));
        return KeyFactory.getInstance("RSA")
                .generatePublic(new X509EncodedKeySpec(der));
    }

    public static PrivateKey readPrivate(Path path) throws Exception {
        byte[] der = Base64.getDecoder().decode(Files.readAllBytes(path));
        return KeyFactory.getInstance("RSA")
                .generatePrivate(new PKCS8EncodedKeySpec(der));
    }

    public static void createPair(Path publicPath, Path privatePath, int bits) throws Exception {
        KeyPairGenerator generator = KeyPairGenerator.getInstance("RSA");
        generator.initialize(Math.max(bits, 2048));
        KeyPair pair = generator.generateKeyPair();
        Files.write(publicPath, Base64.getEncoder().encode(pair.getPublic().getEncoded()));
        Files.write(privatePath, Base64.getEncoder().encode(pair.getPrivate().getEncoded()));
    }
}
```

生成时不要用一段短口令去播种 `SecureRandom`。短字符串的熵不够，不同环境若播种相同，有机会得到可预测的密钥对。`KeyPairGenerator` 默认的随机源已经够实验使用。密钥文件生成在认证服务的机器上，例如 `/var/lib/desk-auth/sign.pem` 与 `sign.pub`，权限设为仅运行用户可读。资源服务只拷贝公钥文件。

## 和单机过滤器的对应关系

单机认证发生在 `UsernamePasswordAuthenticationFilter`。它的父类在成功后把 `Authentication` 放进会话。

多服务、前后端分离时要改两处，仍然沿用过滤器这个扩展点，而不是在控制器里手写一套平行的会话。

- 登录过滤器改为读取 JSON 请求体，而不是表单字段。成功后用私钥签发令牌，写进响应头，不创建会话。
- 原来 `BasicAuthenticationFilter` 靠会话判断是否已登录。资源服务改为读取 `Authorization` 头，用公钥验签，再把权限放进 `SecurityContext`，后面的 `@Secured` 才能工作。

资源服务的过滤器只验签。把私钥配置进资源服务，会让每一台机器都能签发，失去“签发点唯一”的前提。

## 工程划分

父工程聚合三个模块：`desk-token`（令牌与密钥工具）、`desk-auth`（认证服务）、`desk-catalog`（一个资源服务）。父 POM 继承 Boot 2.1。

```xml
<groupId>lab.desk</groupId>
<artifactId>desk-security-parent</artifactId>
<packaging>pom</packaging>
<modules>
    <module>desk-token</module>
    <module>desk-auth</module>
    <module>desk-catalog</module>
</modules>
```

`desk-token` 依赖 jjwt、Jackson、日志和测试。不要依赖 `spring-boot-starter-web`，让它保持成一个纯工具包，认证服务和资源服务都引用它。

工具类就是上一节的 `TokenBody`、`TokenCodec`、`JsonBoxes`、`RsaFiles`。用一个测试方法调用 `createPair` 写出密钥文件，确认两个文件都非空，然后再启动服务。测试不要把私钥路径指到仓库目录。

## 认证服务

### 依赖与配置

认证服务需要 Web、Security、工具模块、MySQL 和 MyBatis。端口用 9101，库沿用单机篇的账号表。

```yaml
server:
  port: 9101
spring:
  datasource:
    url: jdbc:mysql://127.0.0.1:3306/desk_access?useSSL=false
    username: desk
    password: ${DESK_DB_PASSWORD}
desk:
  rsa:
    public-path: /var/lib/desk-auth/sign.pub
    private-path: /var/lib/desk-auth/sign.pem
```

配置类在启动时把两把密钥读进内存：

```java
@Data
@ConfigurationProperties(prefix = "desk.rsa")
public class DeskKeyProps {
    private String publicPath;
    private String privatePath;
    private PublicKey publicKey;
    private PrivateKey privateKey;

    @PostConstruct
    public void load() throws Exception {
        publicKey = RsaFiles.readPublic(Path.of(publicPath));
        privateKey = RsaFiles.readPrivate(Path.of(privatePath));
    }
}
```

启动类打开属性绑定，并扫描 Mapper：

```java
@SpringBootApplication
@MapperScan("lab.desk.auth.mapper")
@EnableConfigurationProperties(DeskKeyProps.class)
public class AuthServer {
    public static void main(String[] args) {
        SpringApplication.run(AuthServer.class, args);
    }
}
```

账号、角色、`UserDetailsService` 从单机篇搬过来。控制器改为 `@RestController`，认证结果用 JSON 返回，不再转发 JSP。Mapper 去掉只在单机演示里用的通用 Mapper 继承，保留按用户名查询即可。

### 登录过滤器

继承 `UsernamePasswordAuthenticationFilter`，覆盖尝试认证和成功回调。

```java
public class JsonLoginFilter extends UsernamePasswordAuthenticationFilter {
    private final AuthenticationManager authenticationManager;
    private final DeskKeyProps keys;

    public JsonLoginFilter(AuthenticationManager authenticationManager, DeskKeyProps keys) {
        this.authenticationManager = authenticationManager;
        this.keys = keys;
        setFilterProcessesUrl("/auth/session");
    }

    @Override
    public Authentication attemptAuthentication(HttpServletRequest request,
                                                HttpServletResponse response) {
        try {
            LoginCommand command = new ObjectMapper()
                    .readValue(request.getInputStream(), LoginCommand.class);
            UsernamePasswordAuthenticationToken token =
                    new UsernamePasswordAuthenticationToken(
                            command.getLoginName(), command.getSecret());
            return authenticationManager.authenticate(token);
        } catch (AuthenticationException ex) {
            throw ex;
        } catch (IOException ex) {
            response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
            throw new BadCredentialsException("无法读取登录请求");
        }
    }

    @Override
    protected void successfulAuthentication(HttpServletRequest request,
                                            HttpServletResponse response,
                                            FilterChain chain,
                                            Authentication auth) throws IOException {
        DeskAccount account = new DeskAccount();
        account.setUsername(auth.getName());
        account.setRoles(new ArrayList<>());
        auth.getAuthorities().forEach(granted -> {
            DeskRole role = new DeskRole();
            role.setCode(granted.getAuthority());
            account.getRoles().add(role);
        });
        String token = TokenCodec.sign(account, keys.getPrivateKey(), 30);
        response.setContentType("application/json;charset=UTF-8");
        response.setHeader("Authorization", "Bearer " + token);
        response.getWriter().write("{\"ok\":true}");
    }
}
```

请求体是 `{"loginName":"...","secret":"..."}`，字段名与表单默认的 `username`/`password` 不同，所以必须自己读流。口令只参与这一次认证，不放进 `TokenCodec.sign` 的对象。有效期示例是 30 分钟，过短会让客户端频繁登录，过长会拉大令牌泄漏后的窗口。

失败分支不要在响应里区分“用户不存在”和“口令错误”，统一成未认证即可，避免把账号是否存在泄露给调用方。

### 认证服务上的校验过滤器

认证服务自己的业务接口如果也要认令牌，就放一个验签过滤器。它和资源服务上的同名逻辑一致：没有 `Bearer ` 前缀则不建立身份；有前缀则验签。

```java
public class BearerVerifyFilter extends BasicAuthenticationFilter {
    private final DeskKeyProps keys;

    public BearerVerifyFilter(AuthenticationManager manager, DeskKeyProps keys) {
        super(manager);
        this.keys = keys;
    }

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain chain) throws IOException, ServletException {
        String header = request.getHeader("Authorization");
        if (header == null || !header.startsWith("Bearer ")) {
            chain.doFilter(request, response);
            return;
        }
        String raw = header.substring("Bearer ".length());
        try {
            TokenBody<DeskAccount> body =
                    TokenCodec.read(raw, keys.getPublicKey(), DeskAccount.class);
            DeskAccount account = body.getAccount();
            var authenticated = new UsernamePasswordAuthenticationToken(
                    account.getUsername(), null, account.getAuthorities());
            SecurityContextHolder.getContext().setAuthentication(authenticated);
            chain.doFilter(request, response);
        } catch (JwtException ex) {
            SecurityContextHolder.clearContext();
            response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
            response.setContentType("application/json;charset=UTF-8");
            response.getWriter().write("{\"ok\":false}");
        }
    }
}
```

没有令牌时继续沿过滤器链往下走，由后面的授权规则决定拒绝，而不是在这里先放行再额外写一个 403。在 `chain.doFilter` 之后再写响应体，会和已经提交的响应冲突，这是验签过滤器里容易写错的顺序。

### 安全配置

```java
@Configuration
@EnableWebSecurity
@EnableGlobalMethodSecurity(securedEnabled = true)
public class AuthSecurityConfig extends WebSecurityConfigurerAdapter {
    @Autowired private DeskAccountService accountService;
    @Autowired private DeskKeyProps keys;

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
        http.csrf().disable()
            .sessionManagement()
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            .and()
            .authorizeRequests()
                .anyRequest().authenticated()
            .and()
            .addFilter(new JsonLoginFilter(authenticationManager(), keys))
            .addFilter(new BearerVerifyFilter(authenticationManager(), keys));
    }
}
```

`STATELESS` 告诉框架不要创建会话。浏览器应用那种 CSRF 同步令牌依赖会话，纯令牌接口在只接受 `Authorization` 头、不使用 Cookie 时，CSRF 的威胁模型不同，示例因此关掉 CSRF。若同一服务还提供基于 Cookie 的页面，就不能照搬这一行。

登录请求 POST `/auth/session`，正文为 JSON。成功时响应头 `Authorization` 以 `Bearer ` 开头。随后访问认证服务上受保护的地址时带上同一头。签名错误或过期应得到 401。

## 资源服务

资源服务可以水平扩展。这里只做货品服务，端口 9102。配置里只有公钥路径，没有私钥项。启动时若代码误读私钥，让它在启动期失败，比在运行期悄悄具备签发能力更好。

```yaml
server:
  port: 9102
desk:
  rsa:
    public-path: /var/lib/desk-auth/sign.pub
```

资源侧的属性类只保留公钥字段，`@PostConstruct` 里只调用 `readPublic`。用户和角色类需要复制一份，因为验签后要还原权限；不必复制登录过滤器和 `UserDetailsService`，资源服务不接受口令。

安全配置去掉登录过滤器，只保留验签，并要求货品接口具备 `USER` 角色：

```java
@Override
protected void configure(HttpSecurity http) throws Exception {
    http.csrf().disable()
        .sessionManagement()
            .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
        .and()
        .authorizeRequests()
            .antMatchers("/catalog/**").hasRole("USER")
            .anyRequest().authenticated()
        .and()
        .addFilter(new BearerVerifyFilter(authenticationManager(), keys));
}
```

处理器：

```java
@RestController
@RequestMapping("/catalog")
public class CatalogApi {
    @GetMapping
    public String list() {
        return "catalog-ready";
    }
}
```

用认证服务签发的令牌访问 `http://localhost:9102/catalog`。头正确且角色足够时返回正文。不带头或签名无法用这把公钥验证时，授权过滤器会拒绝。

把方法标成 `@Secured("ROLE_ADMIN")` 后，只有普通用户角色的令牌会得到 403。这是授权失败，不是令牌无效。需要在数据库里为该用户增加 `ROLE_ADMIN`，重新登录换一张新令牌。旧令牌的载荷里仍然是旧角色，改库不会让已签发的令牌自动更新。这是无状态令牌的特点：权限变化要等令牌过期或重新签发才生效。

资源服务的日志可以记录 `jti` 和用户名，不要记录完整令牌字符串。公钥文件更换时，所有仍用旧私钥签发的令牌都会验签失败，切换密钥要让认证服务和全部资源服务一起换，并给客户端一段重新登录的时间。

## 认证服务需要的账号模型

令牌里要能还原角色，账号和角色的结构就得在签发前组装好。下面的类型只服务认证，不承担页面展示。

```java
@Data
public class DeskRole implements GrantedAuthority {
    private Integer id;
    private String code;

    @Override
    public String getAuthority() {
        return code;
    }
}

@Data
public class DeskAccount implements UserDetails {
    private Integer id;
    private String username;
    private String password;
    private List<DeskRole> roles = new ArrayList<>();

    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        return roles;
    }
    @Override public boolean isAccountNonExpired() { return true; }
    @Override public boolean isAccountNonLocked() { return true; }
    @Override public boolean isCredentialsNonExpired() { return true; }
    @Override public boolean isEnabled() { return true; }
}
```

查询仍按登录名取账号，再按关联表取角色。MyBatis 可以用两次查询，也可以一条连接。关键是 `loadUserByUsername` 返回的对象里，`getAuthorities()` 已经有值，登录过滤器才能把角色放进令牌。

```java
public interface DeskAccountMapper {
    @Select("SELECT id, username, password FROM desk_account WHERE username = #{username}")
    DeskAccount find(String username);

    @Select("""
        SELECT r.id, r.code FROM desk_role r
        JOIN desk_account_role ar ON r.id = ar.role_id
        WHERE ar.account_id = #{id}
        """)
    List<DeskRole> findRoles(Integer id);
}
```

服务里把两次结果拼起来。找不到账号时抛 `UsernameNotFoundException`，让提供者统一变成认证失败，而不是在过滤器里判空。

登录过滤器在成功之后应结束这次请求，不要再调用 `chain.doFilter`。父类的成功回调负责写响应，链再往下走会把已经提交的响应头冲掉，客户端看到的是空头或 500。失败时同样直接写状态并返回，不要先把异常吞掉再继续过滤链，否则后面的授权过滤器会再写一个 403，调用方分不清是口令错了还是令牌没带。

验签过滤器则相反：身份建立之后必须调用 `chain.doFilter`，否则请求停在过滤器里，控制器不会执行，客户端只能看到空白 200。两条过滤器对“要不要继续链”的要求是反的，从登录过滤器复制成功回调到验签过滤器时最容易抄错。登录成功写完头就返回；验签成功放进上下文后把请求交出去。没有令牌时也继续链，让授权规则统一拒绝，避免这个过滤器自己再写一套 403 把已经开始的响应弄乱。

响应头名用标准的 `Authorization`，值以 `Bearer ` 加一个空格开头。漏掉空格或写成 `bearer` 小写以外的自定义方案时，验签过滤器按前缀判断会认为没有令牌。调试时打印头名和前缀长度即可，不要打印头的全部内容。空格属于前缀的一部分，`substring` 时要把这 7 个字符一起去掉，多删或少删一位都会导致签名校验失败，看起来像密钥不对。前缀比对使用 `startsWith("Bearer ")`，不要先 `trim` 再比，否则值中间的空格布局会被改掉。多个空格或换行混进头里时，先拒绝这张令牌，让客户端按原样重传，不要在服务端尝试修复字符串。修复字符串会把本应失败的校验变成一次不确定的解析，联调时更难判断密钥是否配对。密钥文件的权限收成只有运行用户可读，同机的其他账号不应能把私钥复制走。资源机器只给运行用户读取公钥的权限，不要把认证服务的整个家目录拷过去，否则私钥会留在不该出现的磁盘上，增加泄漏面，轮换密钥时也容易漏掉一份。

登录命令和入库实体分开，避免客户端多传一个 `roles` 字段就被信任：

```java
@Data
public class LoginCommand {
    private String loginName;
    private String secret;
}
```

`JsonLoginFilter` 只从 `LoginCommand` 取登录名和口令。角色一律来自数据库查询结果，不来自请求体。否则调用方可以在 JSON 里自己写 `ROLE_ADMIN`。

## 用接口工具走一遍

认证服务起来之后，用一次 POST 验证签发。请求体不要放进 URL。

```http
POST /auth/session HTTP/1.1
Host: 127.0.0.1:9101
Content-Type: application/json

{"loginName":"clerk","secret":"<口令>"}
```

成功时状态 200，响应头里有 `Authorization: Bearer ...`，正文是 `{"ok":true}`。把这枚令牌原样带到货品服务：

```http
GET /catalog HTTP/1.1
Host: 127.0.0.1:9102
Authorization: Bearer <刚拿到的令牌>
```

角色足够时正文是 `catalog-ready`。把令牌最后几个字符改掉再请求，应得到 401，说明验签不是只看前缀。用过期时间很短的令牌等它过期后再访问，同样是 401，这和 403 要分开：401 表示令牌不被接受，403 表示令牌有效但权限不够。

货品服务上如果误配置了私钥路径，代码仍不应去调用 `sign`。检查配置类的字段列表比检查日志更直接：资源模块的 `DeskKeyProps` 没有 `privateKey` 属性，启动期就读不到私钥文件。

多实例部署时，每台货品服务只需要同一份公钥。认证服务水平扩展则每台都必须用同一把私钥，否则一台上签发的令牌在另一台的公钥下验不过。私钥的份数增加，泄漏面也增加，所以认证服务通常保持少量实例，资源服务按流量扩展。

## 模块之间不要互相引用实现

`desk-catalog` 只依赖 `desk-token` 和自己的验签过滤器，不要依赖 `desk-auth`。一旦货品服务的 POM 引入认证服务，私钥配置、数据源和登录过滤器会一起被扫进容器，资源节点就具备了签发能力。公共代码停留在令牌的读写和密钥文件的读取。账号表的 Mapper 可以在两个服务里各写一份查询，或者再拆一个只含实体的模块；不要为了省一个类而把整个认证应用当成库来依赖。

父 POM 把 Boot 版本锁定在 2.1 后，子模块不要再各自声明一套 Spring 版本。jjwt 的版本集中写在 `desk-token`，避免认证服务和资源服务各引一个小版本，运行时出现两套 `Jwts` 类。

密钥文件的测试只做两件事：`createPair` 能写出非空文件，以及用私钥签出的字符串能被刚写出的公钥读回同一个用户名。测试结束后删除临时目录。不要把生成的私钥留在 `src/test/resources`，那会被打包进 JAR。

过滤器的注册顺序要让登录过滤器排在验签过滤器之前。`addFilter` 按过滤器自身的 `@Order` 或父类顺序插入，不是按调用 `addFilter` 的先后。`UsernamePasswordAuthenticationFilter` 的位置本来就在 `BasicAuthenticationFilter` 之前，继承它们的子类会落在同一槽位。若两个自定义过滤器都继承同一个父类，后加的会把前一个挤掉。那时应改用 `addFilterBefore` / `addFilterAfter`，显式锚在某个已知过滤器上。

会话策略设为 `STATELESS` 之后，`SecurityContext` 只活在当前请求的线程里。请求结束，`SecurityContextPersistenceFilter` 不会把身份写进 HttpSession。这要求每一个需要身份的请求都自己带令牌，不能依赖浏览器自动带上的会话 Cookie。用浏览器直接打开货品地址而没有头，失败是符合配置的，不是过滤器漏了。

最后核对清单：

- 认证服务能用数据库账号签发，响应头里是 Bearer 令牌，载荷解码后看不到口令
- 货品服务只有公钥配置项
- 不带令牌访问货品接口被拒绝
- 篡改令牌中间一段后验签失败
- 只有 `ROLE_USER` 的令牌访问 `@Secured("ROLE_ADMIN")` 得到 403
- 给账号补上管理员角色后必须重新登录，旧令牌仍然 403

六条都符合，这条无状态链路才算接上。任意一条靠“先放行再在控制器里判断”绕过，后面加服务时还会把同一段判断复制一遍。

JSON 工具如果还要给别的接口用，可以补上集合和字典的读取，但令牌路径只依赖对象这一对方法。集合解析用 Jackson 的 `constructCollectionType`，避免把 `List` 读成 `List<LinkedHashMap>` 后再手工转。解析失败应抛出受检范围之外的运行时异常，让验签过滤器统一变成 401，而不是返回 null 继续放行。

```java
public static <T> List<T> readList(String json, Class<T> element) {
    try {
        return MAPPER.readValue(json,
                MAPPER.getTypeFactory().constructCollectionType(List.class, element));
    } catch (IOException ex) {
        throw new IllegalArgumentException(ex);
    }
}
```

角色列表若以 JSON 数组形式放进令牌，读取时用这个方法，元素类型是 `DeskRole`。不要把整个 `UserDetails` 的 Spring 代理序列化进载荷，代理类带有无法还原的内部字段，换一个进程就读不回来。签发前拷贝成只含用户名和角色码的普通对象，就是登录过滤器里 new 一个新的 `DeskAccount` 的原因。

时钟也要纳入联调。`exp` 按签发机器的时间计算，验签机器如果慢了几分钟，会把仍应有效的令牌判成过期。实验虚拟机先做时间同步，再怀疑密钥。反过来，验签机器时钟超前会把过期令牌继续当成有效，等于把有效期悄悄延长。两边都对时，比把有效期调到好几天来“避免过期”更合适。

跨域时，浏览器不会把自定义的 `Authorization` 头自动放进简单请求。前端要先发 OPTIONS 预检，资源服务再允许这个头。允许头写成星号、却没有把 `Authorization` 列进实际响应，浏览器会在预检阶段就失败，服务端日志里看不到 GET。这和令牌签名错误不是同一层：预检失败时业务过滤器还没运行。示例若暂时由网关统一处理跨域，资源服务就不要再写第二套允许来源，两套规则叠在一起时，错误的那套会把对的覆盖掉。

货品服务和认证服务的端口不要复用。9101 只签发，9102 只验签。用同一个端口换进程启动，客户端会把上一枚令牌送到错误的过滤器上，现象是登录接口突然开始要 Bearer，或者货品接口突然返回登录 JSON。启动脚本里把端口和模块名打在同一行日志里，联调时先核对这行。

令牌放在响应头而不是响应体，是为了和后续请求使用的 `Authorization` 头对齐，调用方截取同一段字符串即可。若团队约定把令牌放进 JSON 字段，验签过滤器也要改成读正文，两种位置不要并存，否则有的客户端带头、有的客户端带字段，过滤器只认一种就会把另一种当成未登录。示例统一用响应头。正文里的 `{"ok":true}` 只表示签发动作完成，不携带第二份令牌，避免日志中间件把正文落盘时把令牌一起记下。需要审计时记 `jti` 和用户名，足够追到是哪一次签发。

载荷里的角色列表变长时，令牌字符串会跟着变长。网关或浏览器对请求头有长度上限，角色多到上百个时可能被中间件直接丢掉，表现成验签过滤器看不到头。演示只用少量角色码。真要携带很长的权限清单，应改成令牌里只放用户 id，资源服务再查自己的权限表，那就是另一种有状态设计，和本篇“验签即可授权”的取舍不同，不要在同一枚令牌里两套都塞满。
