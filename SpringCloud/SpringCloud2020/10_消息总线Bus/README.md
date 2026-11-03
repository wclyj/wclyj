# 消息总线 Spring Cloud Bus

配置中心负责保存文件，总线负责告诉每个进程「文件已经变了」。只改仓库、不发通知时，已经启动的实例仍握着旧值。

## 概述

Bus 把各个实例接到同一套消息代理上，用来广播状态变化。它本身不存储配置，常和 Config 一起用：仓库提交之后，触发一次刷新端点，所有订阅者再去配置中心拉新值。

代理可以选 RabbitMQ，也可以选 Kafka。下面的例子走 AMQP。

Bus 把 Java 进程内的事件，交给消息中间件再投递出去。可以把它看成一条跨进程的通知通道：谁收到通知，谁就执行本地的刷新，而不是由某一个业务接口去逐个 HTTP 调用别人。

### 什么是总线

多个实例订阅同一个主题。主题上出现一条消息时，每个订阅者都能看到。配置变更、缓存作废这类「人人都要知道」的事件，适合丢到这条主题上，而不是点对点调用。

### 基本原理

每个配置客户端都听同一个默认主题。某个节点收到刷新动作后，把事件发到该主题。其余节点读到事件，再向配置中心拉取自己的 `name`、`profile` 和分支，替换本地已绑定的属性。

```text
仓库提交
   │
   ▼
配置中心  --发布刷新事件-->  消息代理主题
   ▲                              │
   │                              ▼
   └── 各客户端按需重新拉取  <── 全部订阅者
```

通知的是「该刷新了」，新内容仍从配置中心读取，不会把整份 YAML 塞进消息体。

## RabbitMQ 环境

本地或内网先有一个可用的代理，演示账号不要用在生产。

- 管理端口：`10.6.1.15:15672`
- AMQP 端口：`5672`
- 虚拟主机：`/lab`
- 用户：`bus.demo` / `bus-demo-pass`

确认能登录管理台后，再给配置中心和客户端加依赖。代理没启动时，应用会在连接阶段失败，刷新端点也不会可用。

## 全局广播

### 配置

客户端增加总线依赖：

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId><artifactId>spring-cloud-starter-bus-amqp</artifactId>
</dependency>
```

`profile-reader` 既指向配置中心，也写下代理地址：

```yaml
spring:
  application:
    name: profile-reader
  cloud:
    config:
      label: release
      name: checkout
      profile: qa
      uri: http://10.6.1.15:4180
  rabbitmq:
    host: 10.6.1.15
    port: 5672
    username: bus.demo
    password: bus-demo-pass
    virtual-host: /lab
```

这里会读取分支 `release` 上的 `checkout-qa.yml`。`uri` 是配置中心，不是代理端口。

### 设计思想

有两种触发位置。

第一种：打到某一个客户端的刷新地址，由它再发总线消息，带动其他实例。

第二种：打到配置中心的刷新地址，由中心统一发消息。

更稳妥的是第二种，原因有三点：

- 业务进程只处理业务。让某个订单服务兼任「刷新开关」，职责就混了。
- 各实例应是对等的。指定其中一台当入口，这台就变成特殊节点。
- 实例迁移后 IP 和端口常变。入口若绑在业务实例上，每次扩缩容都要改调用方。

配置中心地址相对稳定，适合作为唯一的刷新入口。

## 服务端挂上总线

配置中心模块同样引入 `spring-cloud-starter-bus-amqp`。

### 增加配置

服务名改成 `profile-hub`，Git 仓库换成自己的配置库，并只暴露总线刷新端点：

```yaml
spring:
  application:
    name: profile-hub
  cloud:
    config:
      server:
        git:
          uri: https://git.example.internal/team/app-profiles.git
          search-paths:
            - checkout
          username: profile-bot
          password: ${GIT_TOKEN}
      label: release
  rabbitmq:
    host: 10.6.1.15
    port: 5672
    username: bus.demo
    password: bus-demo-pass
    virtual-host: /lab

management:
  endpoints:
    web:
      exposure:
        include: busrefresh
```

`busrefresh` 对应执行器上的总线刷新操作，HTTP 方法是 POST。调用后，中心发一条消息，订阅该主题的客户端各自回源拉配置。端点名字在不同代的执行器里可能是 `bus-refresh` 或 `busrefresh`，以本机 `/actuator` 索引为准，不要照抄旧文档里的拼写。

上面把口令放在环境变量 `GIT_TOKEN`，避免写进仓库。

## 客户端接上总线

依赖与服务端相同，仍是 `spring-cloud-starter-bus-amqp`。

### 修改 yml

客户端要能连上代理，否则收不到广播。监控端点可以比服务端更宽，便于本机查看健康状态，但不要在公网暴露 `*`。

```yaml
spring:
  application:
    name: profile-reader
  cloud:
    config:
      label: release
      name: checkout
      profile: qa
      uri: http://10.6.1.15:4180
  rabbitmq:
    host: 10.6.1.15
    port: 5672
    username: bus.demo
    password: bus-demo-pass
    virtual-host: /lab

management:
  endpoints:
    web:
      exposure:
        include: health,info,busrefresh
```

和服务端的差别：服务端只开刷新入口；客户端至少要连上同一个虚拟主机。刷新请求仍发到服务端，而不是逐个打到客户端。

使用 `@RefreshScope` 或 `@ConfigurationProperties` 绑定的 Bean，才会在事件到达后重建。普通构造时写死的字段不会变。

## 测试

两端都连上代理之后，向配置中心发 POST：

```bash
curl -X POST "http://10.6.1.15:4180/actuator/busrefresh"
```

若该版本的路径仍带连字符，则改成 `/actuator/bus-refresh`。返回 404 时先打开 `http://10.6.1.15:4180/actuator` 看实际映射。

成功时，所有 `profile-reader` 实例都会收到通知并重新读取 `checkout-qa.yml`。改一处仓库、触发一次端点，各环境实例一起生效。

手工验证可以在配置里放一个 `promo.banner`，提交后调用端点，再请求客户端的读取接口，确认文案已换。

## 定点通知

有时只想更新一部分实例。例如十台里只更新一台灰度机，其余继续用旧文案。

形式：

```text
POST http://配置中心:端口/actuator/busrefresh/{destination}
```

`destination` 一般是 `服务名:端口`，也可以只写服务名来命中该名字下的全部实例。请求仍然发给配置中心，由消息里的目标字段过滤，而不是把刷新 POST 直接打到那台机器。

### 案例

`profile-reader` 有 `4191` 和 `4192` 两个端口。只让 `4191` 更新：

```bash
curl -X POST "http://10.6.1.15:4180/actuator/busrefresh/profile-reader:4191"
```

`4192` 不会处理这条目标不匹配的消息。服务名必须和 `spring.application.name` 一致，端口是该实例实际监听的端口。写错目标时请求往往仍返回成功，只是没有实例响应，需要看客户端日志里有没有收到并忽略。

## 踩坑

- 代理账号、虚拟主机不一致时，两端看起来都「启动成功」，刷新却没人收到。
- 只提交 Git、不调用端点，已运行的进程不会自己轮询仓库。
- 配置类没有刷新作用域时，日志显示收到了事件，接口仍返回旧值。
- 把刷新入口留在某一台业务机上，这台一下线，整套动态配置就停了。
