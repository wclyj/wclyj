# 用 Zipkin 看清一次请求走过了谁

## 前言

Zipkin 用来收集分布式调用的计时。每个服务把片段上报，Zipkin 聚合成一条链路，用来回答「慢在哪一跳」，而不是代替业务日志。

站点页面往往要串起网关、内容接口、评论和对象存储。其中一跳变慢时，有两种找法：

- 把每一台机器的业务日志按时间对齐，靠人工对。
- 打开 Zipkin 的页面，按耗时排序，直接看到偏慢的服务。

各进程在互相调用时带上追踪上下文，Zipkin 收到后做聚合、存储和展示。页面上能看延迟、调用树和依赖。依赖图是按真实调用关系画出来的，不是手写的架构图。

追踪进程要先于业务进程启动。后启动的话，前面那一段调用没有收集端，页面上会缺记录。

## 安装 Zipkin

### 安装须知

Spring Boot 2 之后，不再建议自己写一个 Zipkin 服务端工程。使用官方可执行包即可，业务仓库里不必再放一个 `trace-server` 模块。

### 下载与运行

从 Zipkin 的发布页取可执行 jar（2.23 这一代即可，不必锁死某一个补丁号）。内存模式启动：

```bash
java -jar zipkin-server-exec.jar
```

若希望跨进程的片段先进入消息队列，再由收集端消费，可以打开 Rabbit 收集器：

```bash
java -jar zipkin-server-exec.jar --zipkin.collector.rabbitmq.addresses=10.6.1.15:5672
```

默认把数据放在内存。进程重启后，历史链路消失。页面地址是 `http://127.0.0.1:9411`。

### 存储方式

可选内存、MySQL、Elasticsearch。试验用内存；要留存几天的调用，再换外部存储。下面是改走 MySQL 时要准备的表，语句按当前库结构重写，不照搬旧脚本。库名用 `trace_store`。

三张表各自的职责：

- `span_record`：一次工作单元。保存 trace、span、父 span、名称、远端服务、开始时间和耗时。
- `span_note`：某个 span 上的事件或键值，例如状态码、主机。
- `span_daily_edge`：按天汇总的调用边，谁调用谁、次数、错误次数。

```sql
CREATE TABLE span_record (
  trace_hi BIGINT NOT NULL DEFAULT 0,
  trace_lo BIGINT NOT NULL,
  span_id BIGINT NOT NULL,
  span_name VARCHAR(128) NOT NULL,
  remote_name VARCHAR(128) NULL,
  parent_span BIGINT NULL,
  start_us BIGINT NULL,
  duration_us BIGINT NULL,
  PRIMARY KEY (trace_hi, trace_lo, span_id)
);

CREATE TABLE span_note (
  trace_hi BIGINT NOT NULL DEFAULT 0,
  trace_lo BIGINT NOT NULL,
  span_id BIGINT NOT NULL,
  note_key VARCHAR(128) NOT NULL,
  note_val VARBINARY(1024) NULL,
  note_type INT NOT NULL,
  note_ts BIGINT NULL
);

CREATE TABLE span_daily_edge (
  day_bucket DATE NOT NULL,
  caller_name VARCHAR(128) NOT NULL,
  callee_name VARCHAR(128) NOT NULL,
  call_total BIGINT NULL,
  fail_total BIGINT NULL,
  PRIMARY KEY (day_bucket, caller_name, callee_name)
);
```

表意对齐官方模型即可，字段名可以不同，但收集端若使用官方 MySQL 存储，仍要执行它随包提供的建表脚本。自己改了表名时，必须同时改存储实现，不能只改一半。

使用官方存储时的启动参数示例：

```bash
java -jar zipkin-server-exec.jar \
  --STORAGE_TYPE=mysql \
  --MYSQL_DB=trace_store \
  --MYSQL_USER=tracer \
  --MYSQL_PASS=tracer-demo \
  --MYSQL_HOST=10.6.1.20 \
  --MYSQL_TCP_PORT=3306
```

业务流量打过来之后，`span_record` 应出现新行。只有页面、没有表数据，多半是仍停在内存模式，或库名没有指到 `trace_store`。

## 接到业务进程

引入追踪与上报：

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId><artifactId>spring-cloud-starter-sleuth</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId><artifactId>spring-cloud-starter-zipkin</artifactId>
</dependency>
```

较新的发行版里，上报依赖改名为 `spring-cloud-sleuth-zipkin`。以依赖树里实际存在的构件为准，不要两个都写上一份旧坐标。

内容站点需要上报的进程：`portal`、`ops-console`、`notify-hub`、`gallery`、`finder`。每个进程的配置相同，只有应用名不同。

```yaml
spring:
  sleuth:
    sampler:
      probability: 0.2
  zipkin:
    base-url: http://10.6.1.20:9411
```

`probability` 取 0 到 1。0.2 表示大约五分之一的入口请求会记录；排障时临时调到 1，看完再调回去。全量采集在高峰会把收集端和存储打满。

`base-url` 指向 Zipkin，不是指向注册中心。改错端口时，业务仍正常，只是页面上永远没有新 trace。

浏览器打开 `http://10.6.1.20:9411`，能按服务名 `portal` 搜到调用，说明上报通了。

## Sleuth

### 定义

Sleuth 负责在日志和进程间调用里放上追踪号。名字取自「追查」。它不存储链路，只负责产生和传递标识。

### 原理

调用方把 span、trace 放进请求头。被调方用过滤器取出这些头，计算本段耗时，把结果放进有界队列，再异步 POST 到 Zipkin。这样上报失败不会把业务线程堵死。

相关角色：

- 出站拦截：给 Feign 或 `RestTemplate` 的请求补上 trace 与 span。
- 入站过滤：读请求头，包住控制器执行，拼出本段 span。
- Zipkin：收 JSON，落存储，给页面查询。

日志图案里加上 trace、span 之后，同一请求在 `portal` 和 `finder` 的业务日志可以用同一个 trace 对上。没有 Sleuth 时，Zipkin 即使运行，应用也不会主动上报。

### 采样率

使用默认的按比例采样时，内置比例常常是 0.1。用 `spring.sleuth.sampler.probability` 覆盖。设成 0 等于关闭；设成 1 等于来者不拒。网关已经采样的请求，下游要沿用上游的决定，否则一条链路会在中间断开。这个传递由请求头完成，不要在每个服务里用互不相关的随机数重新决定。

### 附加标记

可以给 span 加标签，但标签越多，单条记录越大。HTTP 场景默认识别状态码、主机和 URL。若还要把某个请求头放进链路，配置允许的头名称列表，不要把全部头都抄进去，以免把令牌写进追踪库。

## Zipkin

### 定义

它来自 Twitter 的分布式追踪思路，分成嵌入应用的上报端和独立部署的服务端。服务端默认内存存储。

它记录的是 RPC、HTTP、数据访问这类跨边界调用，不是 Log4j 里每一行业务日志。想检索全部应用日志，仍应走日志管道和检索引擎。两者可以共用同一个 trace 号，但系统是分开的。

### 几个名词

- Span：一段工作。一次 HTTP 或一次 SQL 都可以是一个 span，用编号标识，并带名称、时间戳、标签和父编号。
- Trace：同一入口下的 span 集合，像一棵树，有一个总编号。
- Annotation：这段工作上的事件。客户端/服务端各有开始和结束，一共四类。
- 二进制标签：额外的键值，不参与时间轴计算。

四类事件：

- `cs`：调用方发出。
- `sr`：被调方收到。
- `ss`：被调方发送响应。
- `cr`：调用方收到响应。

### 怎样读一条链路

一条 trace 由总编号串起。每个 span 用父编号指向自己的上游。例如 `portal` 调用 `finder`，`finder` 再调用数据库：根 span 的父编号为空，后两段的父编号指向各自的调用者。

延迟不要靠墙钟绝对时间相减到跨机器，优先用同一段上的事件差：

- `sr - cs`：请求在网络上的时间，还混有时钟偏差。
- `ss - sr`：被调方自己的处理时间。
- `cr - cs`：调用方视角的整段耗时。

举例（相对入口的毫秒）：

- 8 ms，`portal` 发出 `cs`。
- 11 ms，`finder` 收到 `sr`。网络大约 3 ms。
- 26 ms，`finder` 发出 `ss`。处理大约 15 ms。
- 27 ms，`portal` 收到 `cr`。整段大约 19 ms。

页面按相对时间排序。`ss` 和 `cr` 若落在同一毫秒，显示顺序可能挨在一起，不一定严格按字母。

### 服务端四个模块

- Collector：接收各应用上报。
- Storage：写入内存、MySQL、Cassandra 或 Elasticsearch。
- Query：按 trace 编号、服务名、时间范围读存储，给页面提供 JSON。
- Web：查询页面和依赖图。

四个模块可以同进程启动。上报地址不要写成 Query 的内部端口；可执行包默认把收集和页面放在 9411。

## 缺点

- 上报端和 Spring Cloud 的追踪起步依赖绑在一起，换非 Spring 进程时要另接上报库。
- 粒度停在接口和它调用的外部资源，看不到某个私有方法内部的循环。
- 采样率太低时，刚好那次慢请求可能没被抽中。

## 页面上各块在看什么

### 首页

按服务名、时间范围、耗时下限过滤。先选 `portal`，再把最小耗时调到 200 ms，就能把正常的短请求藏掉。查不到数据时，先确认时间范围覆盖「刚刚」，浏览器若停在别的时区，范围会错位。

### 单条 trace

点进一条记录后是调用树。每一行是一个 span，缩进表示谁调用谁。最宽的那一行就是最值得看的耗时。

### span 明细

相对时间是从这条 trace 开始算的。调用动作仍是 `cs`、`sr`、`ss`、`cr` 四种。

对照前面的例子：11 ms 的 `sr` 减去 8 ms 的 `cs`，得到进入 `finder` 之前的网络时间；26 减 11 得到 `finder` 内部处理。若 `sr - cs` 特别大，而 `ss - sr` 很小，时间花在网络或排队，不在业务代码。

### 依赖图

依赖页把一段时间内的调用边画成图。点服务名可以看到谁在调用它、它又去调用谁。点具体的边能看到：

- 成功次数
- 失败次数

失败次数升高但耗时不变，优先查错误码；耗时升高但失败很少，优先查慢查询或下游容量。依赖图是采样后的统计，不是精确账单。

## 排障顺序

1. 确认 9411 已监听，业务配置的 `base-url` 与它一致。
2. 把出问题的那个服务采样调高，打一次能复现的请求。
3. 用日志里的 trace 号到页面搜索，而不是按时间瞎翻。
4. 看最长的 span 是下游还是自己；再决定查 SQL 还是查网络。
5. 查完把采样调回日常比例，避免收集端被单次排障的全量流量留下。


## 请求头里带什么

Sleuth 不会改 URL。它把追踪上下文放在 HTTP 头里，下游读头再生成自己的子 span。排障时在网关访问日志里应能看到类似下面的头（数值是举例）：

```text
X-B3-TraceId: 6f2c91ab88de4401
X-B3-SpanId: 19ac03e2
X-B3-ParentSpanId: 6f2c91ab88de4401
X-B3-Sampled: 1
```

`TraceId` 整条链路不变。`SpanId` 每一跳不同。`ParentSpanId` 指向调用方的 span。`Sampled` 为 0 时，下游不要再自己抽签改成 1，否则页面上会出现只有后半段的记录。

新一些的组件还会带 W3C 的 `traceparent`。同一进程里两套头应描述同一次调用。若网关只转发其中一套，下游会新开一条 trace，页面上就对不起来。

## 日志怎样和页面互相跳

没有追踪时，日志只有时间和线程名。接上之后，图案里应多出 trace 和 span。例如 `portal` 与 `finder` 各打一行：

```text
09:12:01.204 INFO [portal,6f2c91ab88de4401,6f2c91ab88de4401] render home
09:12:01.219 INFO [finder,6f2c91ab88de4401,19ac03e2] query keyword=ink
```

方括号里第二个字段相同，就可以拿 `6f2c91ab88de4401` 到 Zipkin 搜索。若日志里有号、页面没有，是上报没到 9411；页面有、日志没有，是日志图案没加 `%X{traceId}` 一类的占位。两处都要配，只配一边无法闭环。

业务代码一般不用手动 `MDC.put`。过滤器进入控制器之前会放好，离开时清掉。自己新开的线程池不会自动继承，异步任务里的日志会丢号。需要继承时使用框架提供的可传递线程池，或在提交任务前把当前 span 包进回调。

## 上报走 HTTP 还是队列

`base-url` 是 HTTP 上报，应用直接把 JSON 发给 9411。机器少、调用量不大时够用。Zipkin 重启的那几秒，在途的 span 会丢，业务请求本身不受影响，因为发送在后台队列。

调用量大、或希望收集端短暂不可用时不丢数据，改成先发到 RabbitMQ，由 Zipkin 的收集器消费。应用侧不再写 `base-url`，改为声明队列地址；启动 Zipkin 时打开 rabbit 收集器，地址与业务使用同一个虚拟主机。

不要两种上报同时打开，否则一条调用会被记两次，依赖图上的次数翻倍。切换之后用一次固定的探测请求验证：页面上该 trace 只有一棵树。

## 父编号对不上时

调用树出现多个根，通常不是存储坏了，而是中间有人没把头发全。常见原因：

- 网关用了自己的追踪过滤器，又把下游头改写了一遍。
- Feign 的拦截器被自定义客户端覆盖，出站头丢失。
- 消息消费重新起了一条 trace，消息体里却没有把上游 trace 带上。

先在下游打出入站头。下游的 parent 应等于上游的 span。对不上就查这一跳的客户端，而不是去删 Zipkin 的表。

时钟不准时，`sr` 可能早于 `cs`，相减得到负数。页面仍能展示，但网络延迟没有意义。先对时，再谈优化网络。

## 换成检索集群时

内存和 MySQL 适合试验和中小流量。span 每天上千万时，查询会拖慢页面，这时把 `STORAGE_TYPE` 换成 Elasticsearch，并给 Zipkin 一个独立索引，不要和业务日志写进同一个索引。保留天数在索引生命周期里设，例如 7 天。改存储不等于改应用依赖，应用仍然只认 9411。

自查可以压成四步：9411 有响应、业务日志里有 trace、页面能用该号搜到、最长的那一行能对应到具体下游。四步都过，再去看代码里的循环和 SQL。

采样在网关决定即可。下游看到 `Sampled: 0` 就不要上报，避免把没被抽中的调用补成残缺树。页面上的依赖次数会低于真实 QPS，用它看比例和失败，不用它做账单。
