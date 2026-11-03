# Eureka 集群

只有一台注册中心时，它一停，新的登记和地址查询一起停。已经缓存在调用方内存里的地址还能撑一会儿，过期后整个调用链会找不到人。集群用来拿掉这个单点。

## Eureka 工作原理

注册中心干两件事：收下服务提交的地址，再把地址按名字发给调用方。可以看成一张表，键是服务名，值是主机、端口和实例编号。

一次完整调用是这样的：

1. 先启动注册中心。
2. 再启动账单服务 `billing-api`。
3. 账单服务把 `billing-api` 这个名字和自己的 URL 登记进去。
4. 结算服务需要下单扣款时，拿 `billing-api` 去查实际地址。
5. 拿到地址之后，底层用 HTTP 客户端发起远程调用。
6. 调用方把结果缓存在本进程里，默认约 30 秒才向注册中心要一次新清单。

所以注册中心短暂抖动时，调用不一定立刻失败；缓存过期后如果仍连不上，才会大面积出错。远程调用要高可用，注册中心自己就得先高可用。

## 集群原理

两台注册中心互相把对方写成同伴。A 收到的登记会复制给 B，B 收到的也会复制给 A。调用方在地址列表里写上两台，任何一台活着都能查。

```text
billing-api ──登记──▶ registry-a:7101 ◀──同伴复制──▶ registry-b:7102
checkout-web ──查询──────────────────────────────▶ 两台任一即可
```

这不是主从选举。两台都接受写入，靠同伴复制对齐。

## 搭建集群

单机时常常关闭「向自己登记」和「从自己拉取」，因为进程自己就是登记簿：

```yaml
server:
  port: 7101
eureka:
  instance:
    hostname: registry-a.lab
  client:
    register-with-eureka: false
    fetch-registry: false
    service-url:
      defaultZone: http://registry-a.lab:7101/eureka/
```

改成两台之后，每台的 `defaultZone` 指向同伴，而不是自己。主机名也不能都叫 localhost，否则复制地址会缠在一起。在两台机器的 hosts 里分别解析 `registry-a.lab` 与 `registry-b.lab`。

`7101`：

```yaml
server:
  port: 7101
eureka:
  instance:
    hostname: registry-a.lab
  client:
    register-with-eureka: false
    fetch-registry: false
    service-url:
      defaultZone: http://registry-b.lab:7102/eureka/
```

`7102`：

```yaml
server:
  port: 7102
eureka:
  instance:
    hostname: registry-b.lab
  client:
    register-with-eureka: false
    fetch-registry: false
    service-url:
      defaultZone: http://registry-a.lab:7101/eureka/
```

提供者要登记、要拉取，所以这两个开关必须为 true。注册中心自己不对外提供业务，才关掉它们，避免自己把自己注成一个业务实例。

启动后打开 `http://registry-a.lab:7101`，副本列表里应出现 `7102`。再打开 `7102` 的面板，应能看到 `7101`。两边都能看见对方，集群才算接上。只在一边看得到，多半是 hosts 或 `defaultZone` 写反了。

## 服务注册进集群

账单服务的登记地址写成两台，用逗号隔开：

```yaml
eureka:
  client:
    register-with-eureka: true   # 业务进程需要登记
    fetch-registry: true
    service-url:
      defaultZone: http://registry-a.lab:7101/eureka/,http://registry-b.lab:7102/eureka/
  instance:
    instance-id: billing-api-8011
    prefer-ip-address: true
```

`fetch-registry` 在集群里必须打开，调用方才能拿到同伴同步过来的实例。面板上 `BILLING-API` 应同时出现在两台注册中心。

## 服务提供者集群

提供者也做成多实例。两台进程用同一个 `spring.application.name`，端口不同。名字一样，才会被看成一个服务下的两个实例。

`8011`：

```yaml
server:
  port: 8011
spring:
  application:
    name: billing-api
  datasource:
    type: com.alibaba.druid.pool.DruidDataSource
    driver-class-name: com.mysql.cj.jdbc.Driver
    url: jdbc:mysql://10.6.1.20:3306/shop_billing?characterEncoding=utf8&useSSL=false
    username: billing
    password: billing-demo
eureka:
  client:
    register-with-eureka: true   # 业务进程需要登记
    fetch-registry: true
    service-url:
      defaultZone: http://registry-a.lab:7101/eureka/,http://registry-b.lab:7102/eureka/
  instance:
    instance-id: billing-api-8011
    prefer-ip-address: true
```

`8012` 只改端口和实例编号，服务名保持 `billing-api`。若这台还要把调用链送到追踪系统，可以另加采集地址，与注册无关：

```yaml
server:
  port: 8012
spring:
  application:
    name: billing-api
  zipkin:
    base-url: http://10.6.1.20:9421
  sleuth:
    sampler:
      probability: 0.2
eureka:
  client:
    service-url:
      defaultZone: http://registry-a.lab:7101/eureka/,http://registry-b.lab:7102/eureka/
  instance:
    instance-id: billing-api-8012
    prefer-ip-address: true
```

启动后，注册名 `BILLING-API` 下面应有 `8011` 和 `8012`。调用方使用服务名：

```text
http://BILLING-API/invoice/42
```

`RestTemplate` 要打开负载均衡，否则它会把服务名当成 DNS 主机名，直接失败：

```java
@Configuration
public class HttpClientConfig {
    @Bean
    @LoadBalanced
    public RestTemplate billingClient() {
        return new RestTemplate(); // 配合 @LoadBalanced
    }
}
```

默认策略是轮询，连续两次请求会打到不同端口。这是客户端负载均衡，不是注册中心在转发流量。

## 用 Actuator 补全实例信息

面板上的实例链接依赖健康检查端点。Web 和 Actuator 一起引入：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId><artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId><artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

健康检查示例：`http://10.6.1.31:8012/actuator/health`。返回 UP 时，面板上的状态点才有意义。

### 服务名称修改

`instance-id` 是面板上这一条实例的显示名，不是服务名。服务名仍由 `spring.application.name` 决定。

```yaml
eureka:
  instance:
    instance-id: billing-api-8011
```

两台如果写成同一个 `instance-id`，后启动的会把前一台覆盖掉，面板上永远只剩一条。

### 设置服务的 IP 显示

默认可能显示主机名。希望链接里是 IP 时打开：

```yaml
eureka:
  instance:
    prefer-ip-address: true
```

容器里主机名无法从外面解析时，这项尤其有用。

## 服务发现

注入发现客户端，可以在代码里查看当前登记结果：

```java
@Resource
private DiscoveryClient catalog;
```

常用读法：

```java
List<String> names = catalog.getServices();
List<ServiceInstance> nodes = catalog.getInstances("BILLING-API");
String serviceId = nodes.get(0).getServiceId();
String host = nodes.get(0).getHost();
int port = nodes.get(0).getPort();
URI uri = nodes.get(0).getUri();
```

名字要和注册中心显示的一致。Eureka 通常把服务名转成大写，用小写去查会得到空列表。

## 自我保护

### 概念

自我保护针对的是「注册中心和客户端之间网络被切开」，不是针对「实例真的宕了」。进入该模式后，服务端保留登记表，不再因为心跳暂时缺失就把实例删掉。

面板出现自我保护的提示条，就说明当前节点认为丢心跳的比例已经高到不像是业务集体下线。用一句话说：某台服务此刻连不上，注册中心仍先留着它的记录。这是偏可用的一侧：宁可暂时查到一台可能已死的实例，也不在网络抖动时把整表清空。

### 原因

默认大约 90 秒收不到心跳，服务端会摘掉该实例。网络只是变慢、丢包或分区时，实例本身仍在干活。若按超时一律删除，调用方会拿到一份被掏空的地址表，故障被放大。

服务端在短时间丢失过多客户端时，改为进入保护：表留着，摘除动作停下来。设计取舍是保留可能过期的记录，也不误删仍健康的实例。网络恢复、心跳重新到达之后，表会自己修正。

### 禁止自我保护

本地调试时如果希望进程一停就从面板消失，可以关掉保护，并缩短租约。生产不建议这样，抖动会造成频繁摘除。

服务端：

```yaml
eureka:
  server:
    enable-self-preservation: false
    peer-node-read-timeout-ms: 2000
    peer-node-connect-timeout-ms: 2000
```

客户端把续约间隔和失效时间缩短，仅供试验：

```yaml
eureka:
  instance:
    lease-renewal-interval-in-seconds: 5
    lease-expiration-duration-in-seconds: 12
```

这样实例停止后，大约十几秒就会从两台面板消失。间隔太小会把心跳流量放大，不要照搬到生产。

## 关于停更

Eureka 2 的开源服务端没有继续做下去，1.x 仍可运行，但新功能不应再押在它上面。同角色的替换包括：

- ZooKeeper：偏协调，用临时节点表示在线。
- Consul：登记与健康检查绑在一起，也带 KV。
- Nacos：同时覆盖登记和配置，临时实例走类似续约的模型。

替换时调用方仍依赖「按服务名发现」这层抽象，业务 URL 不必跟着注册中心品牌改写。
