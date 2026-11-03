# Eureka停更后的替换

Eureka 进入维护期之后，注册中心可以换到 Zookeeper 或 Consul。两者都能完成“服务上线登记、下线摘除”，但节点语义和 CAP 取舍不一样。

## Zookeeper替换Eureka

### Zookeeper是什么

Zookeeper 原本是分布式协调组件，靠临时节点和会话也能充当注册中心。关掉实验机防火墙后启动 ZK 进程，让它取代 Eureka Server，业务进程改为向 ZK 登记。

### 搭建Zookeeper注册中心

#### 引入依赖

Spring Cloud 的 ZK 发现起步依赖会传递一个较新的 ZK 客户端。实验环境的服务端还是 3.4 线，需要把传递依赖排除掉，再显式引入匹配的客户端。

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-zookeeper-discovery</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.apache.zookeeper</groupId>
            <artifactId>zookeeper</artifactId>
        </exclusion>
    </exclusions>
</dependency>
<dependency>
    <groupId>org.apache.zookeeper</groupId>
    <artifactId>zookeeper</artifactId>
    <version>3.4.14</version>
</dependency>
```

#### 修改配置文件

`connect-string` 指向 ZK 集群地址。服务名会变成 ZK 上的路径节点。

```yaml
spring:
  application:
    name: ledger-pay-node
  cloud:
    zookeeper:
      connect-string: zk-lab.internal:2181
```

#### 修改主启动类

注册发现用 `@EnableDiscoveryClient`，不要再写 Eureka 专用的客户端注解。

```java
@SpringBootApplication
@EnableDiscoveryClient
public class LedgerPayApp9104 {
    public static void main(String[] args) {
        SpringApplication.run(LedgerPayApp9104.class, args);
    }
}
```

应用本身监听 9104，这个端口写在 `server.port`，和 ZK 的 2181 不是一回事。

#### 启动

进程起来后，在 ZK 客户端里能看到 `/services/ledger-pay-node` 下面多出一个临时子节点，子节点数据里带主机和端口。

#### 思考

挂上去的是临时节点还是持久节点？

Eureka 有自我保护：心跳中断后不会立刻把实例从列表里抹掉，而是先留一段观察期。ZK 的做法不同。客户端和会话绑在一起，会话超时后临时节点被删掉，其它消费者下次拉列表就看不到它。所以 ZK 上的服务登记是临时节点，而不是重启后仍保留的持久节点。删除不是瞬时的，要等会话超时，这一点和“立刻感知下线”的直觉有差别。

## Consul替换Eureka

### 简介

Consul 由 HashiCorp 用 Go 编写，覆盖服务发现、健康检查、KV 配置和多数据中心。这些能力可以拆开用，也可以叠在一起做成一张服务网络。协议上它走 Raft，模型相对直接；发现入口同时提供 HTTP 和 DNS；健康检查的方式比较多；控制台开箱即有；Linux、macOS、Windows 都能跑。

### 功能

- 服务发现：HTTP API 或 DNS 两种查法
- 健康检查：HTTP、TCP、Docker、本机脚本都可以
- KV：用键值对存配置或开关
- 多数据中心：数据中心之间可以做 WAN 联合
- 自带 Web 控制台

### 安装

看版本：

```text
consul version
```

开发模式启动（数据不落盘，只适合本机）：

```text
consul agent -dev -client=0.0.0.0
```

起来之后用浏览器打开本机 8500 端口的控制台，能看到服务列表和健康状态。不要把 `-dev` 用到生产。

### 服务提供者注册Consul

依赖：

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-consul-discovery</artifactId>
</dependency>
```

配置：

```yaml
server:
  port: 9106
spring:
  application:
    name: ledger-consul-pay
  cloud:
    consul:
      host: 127.0.0.1
      port: 8500
      discovery:
        service-name: ledger-consul-pay
        prefer-ip-address: true
```

启动后控制台的服务页会出现 `ledger-consul-pay`，健康检查默认打应用的健康端点。检查失败会被标记为不健康，调用方应跳过它。

## 总结

| 组件 | 实现语言 | 健康检查 | 对外接口 | CAP | 与 Spring Cloud 集成 |
| --- | --- | --- | --- | --- | --- |
| Eureka | Java | 可按需开启 | HTTP | AP | 已提供起步依赖 |
| Consul | Go | 支持 | HTTP 与 DNS | CP | 已提供起步依赖 |
| Zookeeper | Java | 支持 | 自有客户端 | CP | 已提供起步依赖 |

### CAP理论

三个字母分别是：

- Availability：可用性，请求在合理时间内得到响应
- Consistency：一致性，读到的是最近一次成功写入
- Partition Tolerance：分区容错，网络被切开时系统仍能继续工作

讨论粒度是某一类数据在分区时怎么取舍，不是给整套架构贴一个标签。网络分区无法回避，所以 P 必须留下，剩下的选择是更偏向 C 还是更偏向 A。三者不能同时做到最好。

按这个标准，存储系统常被分成三类：

- CA：单机或单集群内部，强调一致和可用，扩展时经不起分区
- CP：分区时宁可拒绝写入，也要保证读到的一致
- AP：分区时继续提供服务，允许短时间读到旧值

有些数据本来就允许延迟一致。例如内容站点的点赞数、阅读数，可以先在内存里累加，隔几秒再落库。这种数据用 AP 更合适。

### AP架构

Eureka 走 AP。

集群之间复制注册表时，如果网络把节点分成两半，每一半仍接受注册和查询，只是彼此还没同步完，客户端可能拿到稍旧的实例列表。可用性留下来了，强一致被放宽。分区愈合后，注册表会逐渐对齐。

### CP架构

Zookeeper 和 Consul 走 CP。

出现分区时，少数派一侧不能再对外确认写入，否则两边都会以为自己持有最新成员列表。客户端在少数派上会看到失败或超时，直到重新连上多数派。一致性留下来了，分区期间的可用性被牺牲。

选注册中心时先问：短暂读到旧实例，和分区时直接拒绝登记，哪一个对这条业务更难接受。支付回调类服务更在意成员列表不要分裂；读多写少的内容服务往往更能接受短暂的旧列表。
