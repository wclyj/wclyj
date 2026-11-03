# Nacos 关闭心跳日志

## 前言

把应用日志接到集中检索之后，磁盘和索引很快被同一类 INFO 刷满。客户端的命名模块会周期打印当前订阅到的地址，空列表也会打。下面是 `ledger-console` 在 `10.8.3.21:7410` 上的节选，服务名换成了目录查询和媒体存储：

```text
[ledger-console:10.8.3.21:7410] 2024-04-11 09:21:06.441 INFO 22041 [com.alibaba.nacos.client.naming.updater] current ips:(0) service: DEFAULT_GROUP@@catalog-query -> []
[ledger-console:10.8.3.21:7410] 2024-04-11 09:21:11.448 INFO 22041 [com.alibaba.nacos.client.naming.updater] current ips:(0) service: DEFAULT_GROUP@@catalog-query -> []
[ledger-console:10.8.3.21:7410] 2024-04-11 09:21:16.455 INFO 22041 [com.alibaba.nacos.client.naming.updater] current ips:(1) service: DEFAULT_GROUP@@media-store -> [{"clusterName":"DEFAULT","healthy":true,"ip":"10.8.3.21","port":7418,"serviceName":"DEFAULT_GROUP@@media-store","weight":1.0}]
[ledger-console:10.8.3.21:7410] 2024-04-11 09:21:21.461 INFO 22041 [com.alibaba.nacos.client.naming.updater] current ips:(0) service: DEFAULT_GROUP@@catalog-query -> []
```

这类行来自订阅刷新，不是业务异常。实例一多，每秒都有几条，采集进来几乎不能用来排障，只会抬高存储费用。目标是把这条链路的级别抬高，让普通刷新不再落盘，真正的失败仍保留。

日志器名称是 `com.alibaba.nacos.client.naming`，刷新线程打在它的 `updater` 子记录器上。父级别设成较严之后，子记录器若没有单独覆盖，会跟着变严。

## 解决方法

在业务服务自己的配置里改日志级别，不必改 Nacos 服务端。下面把命名客户端收到 ERROR 才输出。YAML 的层级要逐级缩进，写平了不会生效。

```yaml
logging:
  level:
    com.alibaba.nacos.client.naming: ERROR
```

若只想压住地址刷新、仍保留注册失败的 INFO，可以只收紧更新器：

```yaml
logging:
  level:
    com.alibaba.nacos.client.naming.updater: ERROR
    com.alibaba.nacos.client.naming: INFO
```

改完后重启该业务进程（例如 `ledger-console`），再观察同一时间窗口，周期出现的 `current ips` 应消失。注册失败、连接被拒绝仍会以 ERROR 出现。

不要把根日志器改成 ERROR，否则业务自己的 INFO 也会没掉。这只影响客户端打印，服务端访问日志要到 Nacos 的 `conf` 里另配。级别写在 `application.yml` 或远端配置都可以，远端修改后需能刷新日志系统才不必重启。

### 为什么空列表也在刷

`catalog-query` 被订阅之后，更新器按固定节奏把「当前地址」打出来。提供方还没注册、或已经下线时，列表是空的，INFO 仍会输出。这不是故障，只是一次订阅刷新。实例和订阅者一多，行数近似于「订阅数 × 刷新次数」。

### 和访问日志不是一回事

客户端这几行出现在业务进程的标准输出里，记录器前缀是 `com.alibaba.nacos.client.naming`。Nacos 服务端目录下的 `access_log` 是 Tomcat 访问日志，记的是 `/instance/beat` 这类 HTTP。关掉客户端 INFO 不会让服务端访问文件变小，反过来也一样。

### 怎样确认已经关掉

1. 改级别前先数一分钟里 `current ips` 出现了多少次，记下服务名。
2. 只把 `naming.updater` 调到 ERROR，重启 `ledger-console`。
3. 再数一分钟。刷新行应消失；故意写错 `server-addr` 时，连接失败仍要以较高级别留下。
4. 若行还在，多半是日志配置被 profile 或日志中心覆盖，或改到了另一个没有被加载的文件。

级别名用 `ERROR` 或 `error` 都可以。不要写成 `ERR`。包名少写一段 `client` 时，规则匹配不到更新器，看起来就像「配置没生效」。
