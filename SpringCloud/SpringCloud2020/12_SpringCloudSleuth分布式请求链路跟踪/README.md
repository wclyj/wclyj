# SpringCloudSleuth分布式请求链路跟踪

## 概述

一次下单从浏览器进来，往往要经过网关、订单、库存、支付好几跳。任何一跳变慢或报错，用户看到的都只是“下单失败”。链路一多，靠日志里的时间戳对不上是谁拖了后腿。

Sleuth 在进程之间传递追踪上下文，并可以把采样到的跨度交给 Zipkin 做查询和依赖图。它解决的是“这次请求走过哪些服务、每段花了多久”，不是业务监控本身。

## 搭建

### zipkin

从 Finchley 这一代开始，不必再自己写一个 Zipkin Server 工程，直接运行官方打包好的可执行包即可。

#### 运行

```text
java -jar zipkin-server.jar
```

#### 打开

默认监听 9411。本实验把它改到 19411，避免和本机别的监控端口撞车。浏览器访问本机该端口下的 zipkin 页面，就能按追踪号检索。

### 名词解释

- Trace：一棵由多个 Span 组成的树，代表一整次用户请求，拥有全局唯一的 Trace Id
- Span：树里的一个节点，对应该次请求中的一段调用，例如“订单调用库存”

#### 完整的调用链路

一条 Trace 用 Trace Id 串起来。每个 Span 记录自己的 Span Id，并用 parent id 指向调用方。根 Span 没有父节点，后续的远程调用、数据库访问都挂在它下面。

依赖图把这些 Span 按服务名折叠：订单指向库存，库存指向仓储，箭头上可以看调用次数和失败占比。排障时先用 Trace Id 定位单次请求，再用依赖图看是不是某一对服务整体变慢。

### 引入依赖

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-zipkin</artifactId>
</dependency>
```

这个起步依赖同时带上 Sleuth 和向 Zipkin 上报的客户端。

### 修改yml

```yaml
spring:
  application:
    name: order-trace
  zipkin:
    # 指向本机 19411，实际填写时加上协议前缀
    base-url: 127.0.0.1:19411
  sleuth:
    sampler:
      # 0 到 1，1 表示每条请求都采样
      probability: 1
```

联调阶段把采样率设成 1，保证每条链路都能在 Zipkin 里看到。流量上来之后再降到 0.1 或更低，否则上报本身会成为负担。服务名会出现在 Span 的本地服务字段里，多个实例共用同一个 `spring.application.name` 即可。
