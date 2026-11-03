# 分布式配置中心SpringCloudConfig

## 概述

### 面临的问题

单体拆开以后，每个进程都要自己的配置：端口、数据源、开关、下游地址。四个服务时手改四份文件还能忍受；四十个服务再乘上测试、预发、生产，文件会散落在各台机器上。改一个开关就要登录多台主机，而且很容易漏。

需要的是一份集中存放、按环境区分、最好还能在运行时推下去的配置。Spring Cloud 用 Config Server 承担这件事：配置只改一处，各客户端按自己的应用名和环境来取。

### 是什么

Config 给微服务提供外部化配置。服务端是一个独立进程，对接存储并对外提供读取、加密和解密。客户端在启动早期就把远程配置拉进环境，业务代码仍用 `@Value` 或配置属性类，不必关心文件放在哪台机器。

### 怎么玩

服务端连接配置仓库，把某个应用、某个环境的文件内容变成客户端能拉到的属性源。客户端声明自己的应用名、环境名和分支，启动时向服务端要一份快照。默认仓库是 Git，因此改配置和改代码一样有提交记录，回滚也是切提交。

### 能做什么

- 配置文件集中存放
- 按 dev、test、stage、prod 区分环境，支持多实例部署
- 运行中的进程不再在每台机器上维护一份本地副本，启动时向中心拉取
- 配合刷新机制，改完配置不必重启就能在客户端生效
- 以 REST 形式暴露配置，也可以用 POST 触发刷新

### 与Github整合部署

默认存储是 Git，也支持 SVN 和本地目录。团队协作更适合 Git，并用 SSH 或内网 HTTP 访问仓库。密钥不要写进仓库明文，敏感项走加密后再提交。

## Config服务端配置与测试

### 引入依赖

服务端除了 Config Server，还可以挂上 Bus，后面用来广播刷新。这里先把两个依赖一起引入。

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-bus-amqp</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-config-server</artifactId>
</dependency>
```

启动类上加 `@EnableConfigServer`。

### 修改YML

服务端监听 4410。`git.repo.runtime-config` 指向内网的配置仓库，搜索目录和默认分支用 `release`。Rabbit 用来承接 Bus。注册中心写两台，避免单点，地址填 registry-a 的 7101 和 registry-b 的 7102。

```yaml
server:
  port: 4410
spring:
  application:
    name: cfg-hub
  cloud:
    config:
      server:
        git:
          uri: ${git.repo.runtime-config}
          search-paths:
            - runtime-config
          default-label: release
  rabbitmq:
    host: mq.lab.internal
    port: 5672
    username: bususer
    password: bus-secret
eureka:
  client:
    register-with-eureka: true   # 配置中心自己也登记
    fetch-registry: true
    # defaultZone 指向 registry-a:7101 与 registry-b:7102
management:
  endpoints:
    web:
      exposure:
        include: bus-refresh
```

### 配置读取规则

服务端按下面两种路径把 Git 文件映射成 HTTP 资源：

- `/{label}/{application}-{profile}.yml`
  - 例：分支 release、应用 runtime、环境 dev，对应路径 `/release/runtime-dev.yml`
- `/{application}-{profile}.yml`
  - 不写分支时走默认分支，对应 `/runtime-dev.yml`

文件必须已经在仓库的 `search-paths` 下面，否则服务端返回空属性。

### 参数总结

- label：Git 分支或标签
- name：应用名，对应文件名前半段
- profile：环境，对应文件名里应用名后面的那段，如 dev、test、prod

## Config客户端配置与测试

### 引入依赖

客户端用 starter-config，不要用 config-server。需要跟随 Bus 刷新时再加 amqp。

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-bus-amqp</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-config</artifactId>
</dependency>
```

### bootstrap.yml

`application.yml` 是应用自己的配置，加载时机偏后。`bootstrap.yml` 由引导上下文读取，用来定位配置中心，优先级更高，远程属性默认不会被同名的本地项覆盖。

引导上下文是应用上下文的父级。它先连上 Config Server，把拉到的属性放进环境，然后再创建应用上下文。两套上下文约定不同，所以远程定位信息单独放在 bootstrap 文件里。客户端模块里原来的 application 文件要改名或拆出 bootstrap，否则进程会先用本地配置启动，根本不会去 4410 拉仓库。

```yaml
server:
  port: 4418
spring:
  application:
    name: cfg-reader
  cloud:
    config:
      label: release
      name: runtime
      profile: dev
      # 配置中心 cfg-hub，端口 4410，填写时带上协议前缀
      uri: 127.0.0.1:4410
eureka:
  client:
    register-with-eureka: true   # 客户端同样登记
    fetch-registry: true
    # defaultZone 指向 registry-a:7101 与 registry-b:7102
management:
  endpoints:
    web:
      exposure:
        include: refresh,health
```

这里 `name` + `profile` + `label` 会去拿 `release` 分支上的 `runtime-dev.yml`。

## 存在的问题

仓库里的值改了，客户端会不会自己变？

- 运维把 Git 上的 `runtime-dev.yml` 改掉并推送
- 再请求 4410，服务端立刻读到新内容
- 再请求 4418，客户端仍是启动时的旧值
- 只有重启 4418，本地环境才会重拉

也就是说服务端是“每次读取仓库”，客户端是“启动时复制一份”。中间缺一次通知。

## Config客户端之动态刷新

目标是改完仓库后，不必重启 4418。

### 引入了动态刷新

客户端加上 Web 和 Actuator：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

引导文件里只暴露 `refresh` 和 `health`。不要用通配把全部端点放开。

业务 Bean 上加 `@RefreshScope`。只有这个作用域里的 `@Value` 会在刷新时重建。

```java
@RestController
@RefreshScope
public class RuntimeBannerController {

    @Value("${runtime.banner}")
    private String banner;

    @GetMapping("/runtime/banner")
    public String banner() {
        return banner;
    }
}
```

仓库更新并确认 4410 已能读到新文件后，向 4418 的 `/actuator/refresh` 发一个 POST。返回体里会列出发生变化的键。再次访问 `/runtime/banner` 就能看到新文案。

这个办法的缺口：

- 客户端不止 4418，还有 4420、4426
- 每台都要手动 POST 一次
- 希望一次通知，所有实例一起换新值

单机刷新做不到广播。下一步是用 Spring Cloud Bus：刷新请求打到任意一个实例或配置中心，由消息中间件把 `refresh` 事件送给同一个总线上的全部客户端。
