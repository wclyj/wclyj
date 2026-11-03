# Nacos

## Spring Cloud Alibaba 简介

Netflix 那一套进入维护状态之后，注册、配置、限流需要有人继续做。Spring Cloud Alibaba 把阿里开源组件接到 Spring Cloud 的编程模型上，填的就是这块空缺。

### 维护模式

模块被标成维护，表示不会再加新能力。严重缺陷和安全问题仍会处理，社区较小的合并请求也可能被看。原计划是继续照看这些模块，直到 Greenwich 被广泛使用至少一年。

### 意味着什么

Spring Cloud 发版很快，有些问题还没修，下一版已经出来了。进入维护之后，Netflix 侧的功能面就停在当时：不再做新组件，以修缺陷和合并分支为主。新的能力改由别的实现提供。

因此选技术时要分开看：已经跑在 Eureka、Hystrix、Zuul 上的系统可以继续维持；新系统不必再把新功能押在这些模块上。

### 怎样进入官方项目

2018 年 10 月 31 日，该项目进入 Spring Cloud 孵化器，并在 Maven 仓库发出第一个版本。组件说明以仓库里的中文 README 和发行说明为准。

### 能做哪些事

- 限流与降级：可接到 Servlet、声明式 HTTP 客户端、`RestTemplate`、Dubbo 和消息消费上。规则能在运行时改，也能看指标。
- 登记与发现：对齐 Spring Cloud 的发现接口，早期版本还带客户端负载均衡。
- 外部配置：配置放在中心，变更后进程可以刷新。
- 消息：通过 Spring Cloud Stream 接 RocketMQ，把应用做成消息驱动。
- 对象存储：对接阿里云的对象存储，用来放文件。
- 任务调度：按 Cron 触发，也可把大量子任务分到多台 Worker。

### 用 BOM 锁版本

不要在每个依赖上各写一个版本。在父工程导入：

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>com.alibaba.cloud</groupId><artifactId>spring-cloud-alibaba-dependencies</artifactId>
            <version>2021.0.5.0</version>
            <type>pom</type><scope>import</scope>
        </dependency>
    </dependencies><!-- 版本只在父工程声明一次 -->
</dependencyManagement>
```

发行版号要和 Spring Boot 代际匹配。Boot 2.3 搭配 2.2.x 的 Alibaba 线，Boot 2.4 以后改看 2021.x。只升级其中一个，客户端注解和自动配置会对不上。

### 各组件分工

- Sentinel：从流量切入，做限流、熔断和系统保护。
- Nacos：服务发现、配置和服务管理。
- RocketMQ：高吞吐的消息与流平台。
- Dubbo：Java RPC。
- Seata：跨服务的事务。
- 对象存储 OSS：存放任意对象。
- SchedulerX：周期任务和固定时刻触发。

下面只展开 Nacos。其他组件各自有独立的接入方式。

## Nacos 简介

它同时做注册中心和配置中心，不是两个要额外部署的系统。

### 名字从哪来

Naming 取前两个字母，Configuration 取前两个字母，末尾的 s 来自 Service。连起来就是 Nacos，全称可以理解成动态的命名与配置服务。

### 它是什么

一个面向云上应用的动态发现、配置管理和服务管理平台。和旧组合的对应关系是：发现能力对位 Eureka，配置能力对位 Spring Cloud Config。一个进程把这两件事都做了。

### 用来替换什么

- 用它承接服务登记，调用方仍按服务名访问。
- 用它承接配置，应用启动时先拉远程文件，再完成本地启动。

### 安装

发行包从 Nacos 的发布页获取，文档站说明概念和部署。本地需要 JDK 8 或更新版本，以及 Maven（只在自己编译时需要）。解压后进入 `bin`：

- Windows：`startup.cmd -m standalone`
- Linux 或 macOS：`sh startup.sh -m standalone`

浏览器打开 `http://127.0.0.1:8848/nacos`。1.x 后期和 2.x 默认要登录，初始账号以发行说明为准，用过之后就改掉。控制台能看到服务列表和配置列表，说明进程起来了。

### 和同类的差别

它在阿里内部承载大量实例，经过大促流量。和只做注册或只做配置的组件比，运维对象更少。代价是注册和配置共享同一套集群，容量规划要一起看。

## 作为注册中心

### 提供者如何登记

依赖：

```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId><artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
</dependency>
```

`pay-ledger` 监听 9106，指向本机 Nacos：

```yaml
server:
  port: 9106
spring:
  application:
    name: pay-ledger
  cloud:
    nacos:
      discovery:
        server-addr: 127.0.0.1:8848
management:
  endpoints:
    web:
      exposure:
        include: health,info
```

启动类打开发现客户端：

```java
@SpringBootApplication
@EnableDiscoveryClient
public class LedgerApp {
    public static void main(String[] cli) {
        SpringApplication.run(LedgerApp.class, cli);
    }
}
```

一个用来确认打到了哪台机器的接口：

```java
@RestController
public class LedgerController {
    @Value("${server.port}")
    private String port;

    @GetMapping("/ledger/entry/{id}")
    public String entry(@PathVariable Integer id) {
        return "ledger port=" + port + ", id=" + id;
    }
}
```

启动后，控制台服务列表应出现 `pay-ledger`，实例 `127.0.0.1:9106`。

再复制一份运行配置，用参数换端口，而不是改文件：

```text
-Dserver.port=9107
```

第二进程起来后，同一服务下有 9106 和 9107。注意参数是 `server.port`，大小写写错会被当成普通系统属性，端口不会变。

### 调用方如何注册并负载均衡

Nacos 客户端会把实例列表交给 Spring 的负载均衡器。调用方同样只加发现依赖。

```yaml
server:
  port: 92
spring:
  application:
    name: checkout-api
  cloud:
    nacos:
      discovery:
        server-addr: 127.0.0.1:8848

remote:
  ledger-base: http://pay-ledger
```

声明带负载均衡的 `RestTemplate`。不加重注解时，`pay-ledger` 会被当成主机名。

```java
@Configuration
public class ClientConfig {
    @Bean
    @LoadBalanced
    public RestTemplate ledgerTemplate() {
        return new RestTemplate(); // 由 @LoadBalanced 选择实例
    }
}
```

```java
@RestController
public class CheckoutController {
    @Resource
    private RestTemplate ledgerTemplate;

    @Value("${remote.ledger-base}")
    private String ledgerBase;

    @GetMapping("/checkout/ledger/{id}")
    public String forward(@PathVariable Long id) {
        return ledgerTemplate.getForObject(
                ledgerBase + "/ledger/entry/" + id, String.class);
    }
}
```

访问 `http://127.0.0.1:92/checkout/ledger/13`，多次刷新应交替看到 `port=9106` 和 `port=9107`。默认是轮询。到这里为止还没有手写任何服务器列表。

### 注册中心怎么比

常见组件里，有的偏可用（AP），有的偏一致（CP）。Nacos 两种都支持，而且可以按服务切换，不是整个集群锁死一种。

#### 和 CAP 的关系

- C：同一时刻，各节点看到的数据相同。
- A：请求能得到响应。
- P：网络分区时系统仍要做取舍。

分区发生时不能同时把 C 和 A 都保住。Nacos 允许按场景选。它除了接 Dubbo、Spring Cloud，也能配合 Kubernetes 的服务模型。

#### 何时用 AP，何时用 CP

不需要在服务级保存额外元数据、实例由客户端注册并持续续约时，用 AP。Spring Cloud 和 Dubbo 的普通实例属于这种。AP 为了能读能写，放宽了瞬间一致，只接受临时实例。

要在服务级别编辑或保存信息时，用 CP。Kubernetes 服务、DNS 一类更适合 CP。此模式下实例可以持久化，集群用 Raft 复制。注册实例前必须先有服务；服务不存在会直接报错，而不是顺手建一个。

临时实例靠续约维持。持久实例不靠心跳删除，适合地址稳定、由运维登记的目标。

控制台或 API 可以指定服务的一致性模式。一个集群里允许一部分服务走 AP、另一部分走 CP，不要假设全局开关只有一个。

## 作为配置中心

配置写在 Nacos，应用按和 Config 类似的方式拉取。

### 基础接法

依赖：

```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId><artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
</dependency>
```

启动时要先连上配置中心，再创建普通 Bean。所以地址写在 `bootstrap.yml`（或 Boot 2.4+ 的 `spring.config.import`），不要只写在 `application.yml`。`bootstrap` 的加载早于 `application`。

`application.yml` 只负责选哪一份环境：

```yaml
spring:
  profiles:
    active: qa
```

`bootstrap.yml`：

```yaml
server:
  port: 3520
spring:
  application:
    name: runtime-profile
  cloud:
    nacos:
      discovery:
        server-addr: 127.0.0.1:8848
      config:
        server-addr: 127.0.0.1:8848
        file-extension: yaml
        group: QA_BUNDLE
        namespace: 7c2a91e0-11bd-4a6e-9f30-88ab12c04d55
```

数据编号的拼法：

```text
${spring.application.name}-${spring.profiles.active}.${file-extension}
```

上面会找命名空间 `7c2a91e0-...`、分组 `QA_BUNDLE` 下的 `runtime-profile-qa.yaml`。扩展名必须和 Nacos 里的文件后缀一致，一边写 `yml`、一边建 `yaml` 时会读不到，启动直接失败。

启动类：

```java
@SpringBootApplication
@EnableDiscoveryClient
public class ProfileApp {
    public static void main(String[] cli) {
        SpringApplication.run(ProfileApp.class, cli);
    }
}
```

读取接口加上刷新作用域，中心改文件后不必重启：

```java
@RestController
@RefreshScope
public class ProfileController {
    @Value("${runtime.banner}")
    private String banner;

    @GetMapping("/profile/banner")
    public String banner() {
        return banner;
    }
}
```

在 Nacos 创建该数据编号，内容例如：

```yaml
runtime:
  banner: qa-build-42
```

`runtime:` 下面要有空格，顶格写 `banner` 会变成另一棵树。先建配置再启动。访问 `http://127.0.0.1:3520/profile/banner` 应返回 `qa-build-42`。

若启动报找不到 `runtime.banner`，先核对三件事：后缀、分组、命名空间编号（用的是 ID，不是控制台上的中文名字）。有的版本文扩展名只认 `yaml`，把数据编号从 `.yml` 改成 `.yaml` 后重新发布即可。

改控制台里的文案再调接口，返回值应跟着变。这就是动态刷新，不需要再搭一条消息总线。

Boot 2.4 之后也可以不使用 bootstrap 文件，而在 `application.yml` 写：

```yaml
spring:
  config:
    import: nacos:runtime-profile-qa.yaml?group=QA_BUNDLE&refreshEnabled=true
```

两种写法选一种，不要两套地址各指一个环境。

### 分类配置

基础能力相当于 Config 加刷新。Nacos 另外用命名空间、分组和数据编号把多环境、多应用拆开。

#### 要解决的问题

系统通常至少有开发、测试、生产三套。大型系统里每个服务又各有这三套。若全部堆在默认空间的默认分组，数据编号一多就会改错文件。

#### 三者各管什么

类似包名和类名：命名空间是最外层隔离，分组和数据编号在空间内部再定位到一份文件。

默认是：空间 `public`，分组 `DEFAULT_GROUP`，集群 `DEFAULT`。

- 空间用来隔离。开发、测试、生产各建一个，彼此看不见配置，服务列表也分开。
- 分组用来把同一类微服务收在一起，例如交易一组、库存一组。默认组是 `DEFAULT_GROUP`。
- 服务是一个逻辑应用，下面可以有多个集群。集群常按机房划分，例如 `HZ` 和 `GZ`，调用时优先同机房。
- 实例是具体进程。

#### 三种加载方式

**只靠数据编号。** 空间和分组都用默认值，为 `dev` 和 `qa` 各建一个 Data ID：`runtime-profile-dev.yaml`、`runtime-profile-qa.yaml`。切换 `spring.profiles.active` 就切换文件。

**靠分组。** 创建配置时填分组。客户端写明：

```yaml
spring:
  cloud:
    nacos:
      config:
        server-addr: 127.0.0.1:8848
        file-extension: yaml
        group: QA_BUNDLE
```

同一数据编号可以在不同分组各有一份。适合「文件名不想带环境，环境放在分组上」。

**靠命名空间。** 在控制台新建空间，记下生成的空间 ID，而不是显示名。服务列表和配置列表都会多出空间切换。

```yaml
spring:
  cloud:
    nacos:
      config:
        server-addr: 127.0.0.1:8848
        file-extension: yaml
        group: DEV_BUNDLE
        namespace: 3fa18c77-0b64-4d2e-91aa-6e55c0b7d441
```

三层一起用时，定位一份配置的键是：空间 ID + 分组 + 数据编号。少写空间时会落到 `public`，表现是读到别的环境或读到空。

发现和配置可以指向不同空间。例如实例注册在 `public`，配置读 `staging`。要分开写 `discovery.namespace` 和 `config.namespace`，不要以为配了一处就两处生效。

## 集群与持久化

### 三种跑法

- 单机：本机试验。
- 集群：生产，多节点避免单点。
- 多集群：多个数据中心，彼此再做同步。

默认单机把数据放在内嵌库 Derby。几个节点若各自用 Derby，数据并不共享，会出现「在 A 上建的配置，B 上看不见」。集群要改成集中存储，目前外置库使用 MySQL。

### 单机也改走 MySQL

0.7 之前单机只有内嵌库，不方便查看。之后可以接 MySQL：

1. 准备 MySQL 5.6.5 以上，或当前发行说明要求的版本。
2. 建库 `nacos_cfg`，执行发行包 `conf` 里的建表脚本。
3. 修改 `conf/application.properties`。

```properties
spring.datasource.platform = mysql
db.num=1
db.url.0=jdbc:mysql://127.0.0.1:3306/nacos_cfg?characterEncoding=utf8&connectTimeout=2000&socketTimeout=5000&autoReconnect=true
db.user.0=nacos
db.password.0=nacos-demo-pass
```

不同版本键名可能是 `db.user` 或 `db.user.0`，以手头 `application.properties` 里的注释为准。改完用单机模式启动，新写入进 MySQL。控制台若是空的，说明没再读 Derby 里的旧数据，这是预期。

### 生产形态

至少：1 台 Nginx，3 个 Nacos 节点，1 个 MySQL。客户端只访问 Nginx，由它转发到后面的节点。

节点端口规划为 `8841`、`8842`、`8843`，不要三台都占 8848 又忘了改脚本。

步骤：

1. 在 `10.20.6.40` 上把 MySQL 和库 `nacos_cfg` 准备好，三台 Nacos 都写同一套数据源。
2. 复制 `conf/cluster.conf.example` 为 `cluster.conf`，每行一个地址：

```text
10.20.6.41:8841
10.20.6.42:8842
10.20.6.43:8843
```

3. 启动时带上端口。新版本直接支持参数，不必改脚本：

```bash
sh startup.sh -m cluster -p 8841
sh startup.sh -m cluster -p 8842
sh startup.sh -m cluster -p 8843
```

若手头是要求改 `startup.sh` 的旧包，再在脚本里接收 `-p` 并传给 JVM 的 `-Dserver.port`。三台的 `cluster.conf` 内容要一致，否则节点互相不认识。

4. Nginx 做四层或七层转发，并带权重：

```nginx
upstream nacos_console {
    server 10.20.6.41:8841 weight=1;
    server 10.20.6.42:8842 weight=1;
    server 10.20.6.43:8843 weight=1;
}
server {
    listen 8088;
    location / {
        proxy_pass http://nacos_console;
    }
}
```

控制台从 `http://10.20.6.40:8088/nacos` 进入。业务只填这一个入口：

```yaml
spring:
  cloud:
    nacos:
      discovery:
        server-addr: 10.20.6.40:8088
```

`server-addr` 写成 Nginx 地址，而不是某一台 8841。某一台 Nacos 停机时，注册和拉配置仍可从另外两台完成。权重用来临时把流量从有问题的节点上挪走。

### 集群检查

- 三台日志里都能看到彼此加入。
- 在控制台发布一条配置，直连任意节点的 API 都能读到（因为后面是同一个 MySQL）。
- 停掉一台，客户端在下一次刷新后仍能注册。`server-addr` 若只写了宕掉的那台 IP，则没有负载均衡，会一起不可用。

内嵌库文件不要和 MySQL 混用。切过数据源之后，旧的 Derby 目录可以留档，但不要再把 `spring.datasource.platform` 改回去，否则新配置会写进另一套库。
