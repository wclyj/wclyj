# 把 Sentinel 规则存进 Nacos

## 前言

控制台里点出来的流控、降级，默认留在 Sentinel 客户端进程的内存中。进程一重启，规则清空，得再配一遍。把规则放到 Nacos 之后，启动时拉回本地，重启不再丢；在 Nacos 里改 JSON，客户端也会更新。

## 安装依赖

在已经接入 Sentinel 的模块上增加 Nacos 数据源，版本跟随 Spring Cloud Alibaba 的发行版，不必单独写死一个过期号。

```xml
<dependency>
    <groupId>com.alibaba.csp</groupId><artifactId>sentinel-datasource-nacos</artifactId>
</dependency>
```

只有仪表盘依赖、没有这个数据源时，控制台能看监控，但不会从 Nacos 装载规则。

## 修改 yml

下面用 `storefront` 这个服务，端口 `8720`。注册与配置都指向 `10.4.1.20:8848`，命名空间用 `staging`。仪表盘改到 `8850`，避免和别的示例抢 `8080`。

```yaml
server:
  port: 8720

spring:
  application:
    name: storefront
  cloud:
    nacos:
      discovery:
        server-addr: 10.4.1.20:8848
        namespace: staging
      config:
        server-addr: 10.4.1.20:8848
        file-extension: yaml
        group: STAGING
        namespace: staging
    sentinel:
      transport:
        dashboard: 10.4.1.20:8850
        port: 8729
      datasource:
        flow:
          nacos:
            server-addr: 10.4.1.20:8848
            data-id: storefront-flow-rules
            group-id: SENTINEL_GROUP
            data-type: json
            rule-type: flow
            namespace: staging
        degrade:
          nacos:
            server-addr: 10.4.1.20:8848
            data-id: storefront-degrade-rules
            group-id: SENTINEL_GROUP
            data-type: json
            rule-type: degrade
            namespace: staging
```

`rule-type` 决定 JSON 按哪种规则解析：`flow` 是流控，`degrade` 是熔断降级。`data-id` 必须和 Nacos 里的数据编号完全一致。两个数据源的 `data-id` 不要写成同一个，否则后加载的类型会对不上。

`transport.port` 是客户端跟仪表盘说话的本地端口。被占用时 Sentinel 会往上试，写死一个空闲端口更容易排错。

## 创建配置文件

在命名空间 `staging`、分组 `SENTINEL_GROUP` 下建两条 JSON：

- `storefront-flow-rules`：流控
- `storefront-degrade-rules`：降级

格式选 JSON，和 `data-type` 一致。建在 `public` 而客户端写了 `staging` 时，启动日志会提示读不到数据源，规则为空。

### 流控 JSON

限制 `GET /catalog/sku/{skuId}` 每秒最多 3 次，超限直接失败。阈值类型用 QPS（`grade` 为 1）。

```json
[
  {
    "resource": "GET:/catalog/sku/{skuId}",
    "limitApp": "default",
    "grade": 1,
    "count": 3,
    "strategy": 0,
    "controlBehavior": 0,
    "clusterMode": false
  }
]
```

字段：

- `resource`：资源名，要和代码里的注解或网关 API 分组一致。
- `limitApp`：来源应用，`default` 表示不区分调用方。
- `grade`：0 按并发线程数，1 按 QPS。
- `count`：单机阈值。
- `strategy`：0 直接统计该资源，1 关联其他资源，2 按调用链入口。
- `controlBehavior`：0 立刻拒绝，1 预热，2 排队。
- `clusterMode`：是否走集群流控。单机演示保持 false。

这些字段对应流控规则对象。少写 `resource` 或把 `count` 写成字符串，数据源会解析失败，进程仍能启动，只是规则不生效。

### 降级 JSON

同一资源在时间窗内异常次数达到 4 次就熔断，窗口 8 秒。`grade` 为 2 表示按异常数，不是按比例。

```json
[
  {
    "resource": "GET:/catalog/sku/{skuId}",
    "grade": 2,
    "count": 4,
    "timeWindow": 8,
    "minRequestAmount": 5
  }
]
```

字段：

- `grade`：0 按慢调用，1 按异常比例，2 按异常数。
- `count`：对应策略的阈值。慢调用时它是毫秒数。
- `timeWindow`：熔断持续秒数，窗口内直接走降级逻辑。
- `minRequestAmount`：样本太少时不触发，避免一次失败就把接口打熔断。

## 测试

启动 `storefront` 之前，Nacos 里两条 JSON 已经存在。连续刷新商品详情，例如 `http://10.4.1.20:8720/catalog/sku/1008`。QPS 超过 3 之后应进入限流分支（演示里返回 429 或自定义降级页），而不是把请求打到下游。

打开仪表盘 `http://10.4.1.20:8850`。能看到 `storefront` 以及各一条流控、降级。规则来自 Nacos 时，在控制台里改完如果没有推回 Nacos，重启仍会回到 JSON 里的值。以后调整阈值，直接改数据编号对应的 JSON 即可。

客户端日志里应有数据源加载成功的记录。若只有仪表盘手动配置、Nacos 仍是空文件，重启后规则会再次消失，说明走的还是内存通道。

## 踩坑

- 资源名和注解不一致：JSON 合法，但没有任何请求命中。
- 分组、命名空间、Data ID 三者有一个不同，表现都是「规则没装上」。
- `data-type` 写成 `xml` 而内容是 JSON，解析异常会被数据源吞掉，界面上像是没配置。
- 只在仪表盘改规则、不回写 Nacos，持久化并没有发生。
