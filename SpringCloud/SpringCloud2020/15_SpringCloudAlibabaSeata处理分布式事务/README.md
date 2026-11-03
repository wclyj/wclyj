# SpringCloudAlibabaSeata处理分布式事务

一份业务如果只打一个库，本地事务就够了。拆成多个服务、每个服务独占一个库之后，一次下单要跨库提交，本地事务盖不住全局一致性。Seata 用来补上这一层。

## 分布式事务

演进通常是：单机单库，然后一应用多库，再然后多应用多库。跨库、多数据源一旦要一起成功或一起失败，就出现分布式事务。

单体里的订单、库存、账户原本在一个库的一个事务里。拆开之后变成三个进程、三个数据源。每个进程内部仍可以用本地事务，但“订单已写入、库存已扣、账户没扣成”这种跨服务的中间态，没有任何一个本地事务能回滚别人的库。

## Seata简介

Seata 提供 AT、TCC、SAGA、XA 几种模式。AT 对业务侵入最小，下面的例子用它。模型可以收成一个全局事务号加三个角色：

- XID：全局事务的唯一编号，沿调用链传递
- TC（事务协调器）：独立服务，记录全局事务和分支，决定提交或回滚
- TM（事务管理器）：放在发起全局事务的业务方法上，负责开启和结束
- RM（资源管理器）：在每个参与方的数据源上，负责注册分支、上报状态、执行提交或回滚

### 处理过程

1. TM 向 TC 申请开启全局事务，成功后得到 XID
2. XID 随远程调用继续往下传
3. 每个 RM 把自己的分支注册到这个 XID 下面
4. TM 根据业务结果，通知 TC 对这个 XID 做全局提交或全局回滚
5. TC 再驱动各个分支完成提交或回滚

### 下载

取与客户端依赖相匹配的 Seata Server 发行包。本实验用 1.4 这一线，解压后先改 `conf` 里的存储和注册中心，再启动。不要拿 0.9 的客户端去连差异很大的服务端协议。

#### 修改file.conf

先把原始文件复制备份。服务端要改三块：事务分组的名字、事务日志改成数据库存储、数据库连接改成自己的库。文件模式重启会丢会话，实验一旦要看回滚，就用 db。

#### 修改service模块

`service` 段里把分组映射到集群名。客户端配置的事务组和这里必须同名，否则 TM 找不到 TC。

```text
vgroup_mapping.ledger_tx_grp = "default"
default.grouplist = "127.0.0.1:8191"
```

本实验把 Seata Server 的服务端口改到 8191，避免和机器上已有的 8091 冲突。

#### 修改store模块

`store.mode` 改为 `db`，并填写独立的 `seata_log` 库连接。`global_table`、`branch_table`、`lock_table` 使用脚本里的默认表名，连接账号不要复用业务库的高权限账号。

#### 创建一个seata数据库

建库 `seata_log`，执行发行包 `script/server` 目录下的存储建表脚本（不同版本目录名可能是 `server` 或 `mysql.sql`）。这三张表只给 TC 用，不要建在业务库里。

#### 修改seata-server的registry.conf配置文件

`registry.type` 改成 `nacos`，地址指向本实验的 8858，命名空间留空即公共空间，cluster 保持 `default`，与上面的分组映射一致。`config.type` 若仍用文件，就保持 `file`，让服务端继续读本地的 `file.conf`。改完先起 Nacos，再起 Seata Server，控制台里应能看到 `seata-server` 这个服务。

### 怎么玩

- 单库：Spring 的 `@Transactional`
- 跨服务：Seata 的 `@GlobalTransactional`

后者包住的方法就是 TM。它并不替代每个库自己的本地事务，而是在本地事务之外再加一层全局协调。库存、账户服务内部该加的本地事务仍然要加。

## 订单/库存/账户业务微服务准备

先起 Nacos 和 Seata Server，两个都健康再写业务。

### 分布式事务的业务说明

三个服务：

- 订单：插入订单，最后把状态改成完成
- 库存：按商品扣减可用数量
- 账户：按用户扣减余额

调用方向是订单服务发起，远程调库存，再远程调账户，最后改订单状态。跨三个库、两次远程调用。任何一步失败，前两步已经提交的本地变更都要能撤回。

一句话：下单、扣库存、减余额、改状态。

### 创建数据库

```sql
create database tx_order default charset utf8mb4;
create database tx_stock default charset utf8mb4;
create database tx_wallet default charset utf8mb4;
```

### 建立业务表

三个库各一张业务表。主键用自增 `id`，不要把列名写成类型名。

```sql
create table o_booking (
  id bigint not null auto_increment,
  user_id bigint,
  sku_id bigint,
  qty int,
  amount decimal(12, 2),
  state tinyint comment '0 drafting, 1 done',
  primary key (id)
);

create table o_stock (
  id bigint not null auto_increment,
  sku_id bigint,
  capacity int,
  consumed int,
  remain int,
  primary key (id)
);
insert into o_stock (sku_id, capacity, consumed, remain) values (7, 80, 0, 80);

create table o_wallet (
  id bigint not null,
  user_id bigint,
  credit decimal(12, 2),
  spent decimal(12, 2),
  remain decimal(12, 2),
  primary key (id)
);
insert into o_wallet (id, user_id, credit, spent, remain) values (1, 9, 5000, 0, 5000);
```

### 创建回滚日志表

AT 模式靠每个业务库里的 `undo_log` 保存前后镜像。三个业务库都要建，TC 用的 `seata_log` 不需要这张表。

```sql
create table undo_log (
  id bigint not null auto_increment,
  branch_id bigint not null,
  xid varchar(128) not null,
  context varchar(128) not null,
  rollback_info longblob not null,
  log_status int not null,
  log_created datetime(3) not null,
  log_modified datetime(3) not null,
  ext varchar(100),
  primary key (id),
  unique key uk_undo_xid_branch (xid, branch_id)
);
```

列名要和客户端预期一致，索引保证同一个分支不会插进两行回滚日志。

## 订单/库存/账户业务微服务准备

### 业务需求

顺序固定为：写订单（状态为草稿）→ 减库存 → 扣余额 → 把订单改成完成。任何远程失败都回滚草稿订单和已经扣过的库存、余额。

### 新建Order-Module表

#### 约定

- 实体放在 domain 包
- 返回给调用方的包装用一个简单的结果类，不另做展示对象
- 入参可以直接用实体，实验规模用不着再拆 DTO

#### 新建module2101

订单服务模块名 `booking-service`，端口 2101。库存 2102，账户 2103，后面两节按同一结构各建一个模块。

#### 引入POM

依赖清单如果已经锁定了 Seata 版本，就不要再排除后手写一个更老的 `seata-all`。版本不一致是连不上 TC 的常见原因。

```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-seata</artifactId>
</dependency>
```

同时还需要 Nacos 发现、OpenFeign、MyBatis 和数据源驱动，端口与注册配置见 yml。

#### 修改yml

事务组名 `ledger_tx_grp` 必须和服务端 `vgroup_mapping` 的键一致。

```yaml
server:
  port: 2101
spring:
  application:
    name: booking-service
  cloud:
    nacos:
      discovery:
        server-addr: 127.0.0.1:8858
    alibaba:
      seata:
        tx-service-group: ledger_tx_grp
  datasource:
    driver-class-name: com.mysql.cj.jdbc.Driver
    url: jdbc:mysql://127.0.0.1:3306/tx_order?useSSL=false&serverTimezone=UTC
    username: tx_user
    password: tx-pass
mybatis:
  mapper-locations: classpath:mapper/*.xml
logging:
  level:
    io.seata: info
```

Feign 的超时要大于账户服务可能的耗时，否则客户端先断开，Seata 会把这次调用当成失败去回滚。实验里把读超时调到 20 秒以上。

#### 增加file.conf

1.4 之前的客户端常在 `resources` 下放一份 `file.conf`，事务组写在这里。1.4 之后更推荐全部放进配置中心或 yml 的 `seata.*`。下面保留文件方式，便于和旧实验对照。分组、TC 地址、存储库都换成自己的值。

```text
transport {
  type = "TCP"
  server = "NIO"
  heartbeat = true
  thread-factory {
    boss-thread-prefix = "seataboss"
    worker-thread-prefix = "seataworker"
    boss-thread-size = 2
    worker-thread-size = 16
  }
  shutdown { wait = 5 }
  serialization = "seata" # 默认序列化，勿改成业务自定义名
  compressor = "none"
}
service {
  vgroup_mapping.ledger_tx_grp = "default"
  default.grouplist = "127.0.0.1:8191"
}
client {
  async.commit.buffer.limit = 2000
  lock {
    retry.internal = 20
    retry.times = 15
  }
  tm.commit.retry.count = 2
  tm.rollback.retry.count = 2
}
store {
  mode = "db"
  db {
    datasource = "druid"
    db-type = "mysql"
    driver-class-name = "com.mysql.cj.jdbc.Driver"
    url = "jdbc:mysql://127.0.0.1:3306/seata_log"
    user = "tx_user"
    password = "tx-pass"
    min-conn = 2
    max-conn = 8
    global.table = "global_table"   # TC 会话表
    branch.table = "branch_table"   # TC 分支表
    lock-table = "lock_table"       # TC 行锁表
    query-limit = 200
  }
}
lock { mode = "remote" }
transaction {
  undo.data.validation = true      # 回滚前校验后镜像
  undo.log.serialization = "jackson"  # 镜像的序列化方式
  undo.log.save.days = 3
  undo.log.table = "undo_log"      # 建在业务库，不是 seata_log
}
support.spring.datasource.autoproxy = true
```

客户端真正用到的是 `service` 里的分组和 TC 地址。`store` 段主要属于 TC；放在客户端文件里是旧教程的习惯，不要误以为业务进程会去连 `seata_log` 写全局表。

#### registry.conf 注册器

客户端这份文件告诉 RM/TM 去哪找 TC。实验只启用 Nacos，其它类型留注释即可，不必把未使用的地址都写进运行配置。

```text
registry {
  type = "nacos"
  nacos {
    serverAddr = "127.0.0.1:8858"
    namespace = ""
    group = "TX_CLUSTER"
    cluster = "default"
  }
}
config {
  type = "file"
  file { name = "file.conf" }
}
```

#### domain

```java
@Data
public class ApiEnvelope<T> {
    private int code;
    private String message;
    private T data;

    public static <T> ApiEnvelope<T> ok(String message) {
        ApiEnvelope<T> body = new ApiEnvelope<>();
        body.code = 200;
        body.message = message;
        return body;
    }
}

@Data
public class Booking {
    private Long id;
    private Long userId; // 下单人
    private Long skuId;
    private Integer qty;
    private BigDecimal amount;
    /** 0 drafting, 1 done */
    private Integer state;
}
```

#### Dao接口及实现

```java
@Mapper
public interface BookingMapper {
    int insertDraft(Booking booking);
    int markDone(@Param("userId") Long userId, @Param("fromState") Integer fromState);
}
```

```xml
<mapper namespace="lab.booking.mapper.BookingMapper">
    <insert id="insertDraft" useGeneratedKeys="true" keyProperty="id">
        insert into o_booking (user_id, sku_id, qty, amount, state)
        values (#{userId}, #{skuId}, #{qty}, #{amount}, 0)
    </insert>
    <update id="markDone">
        update o_booking
           set state = 1
         where user_id = #{userId}
           and state = #{fromState}
    </update>
</mapper>
```

`markDone` 用状态条件更新，避免把已经完成的行再改一次。

#### Service实现类

库存和账户通过 Feign 调用，接口只暴露扣减动作。

```java
@FeignClient("stock-service")
public interface StockClient {
    @PostMapping("/stock/consume")
    ApiEnvelope<Void> consume(@RequestParam("skuId") Long skuId,
                              @RequestParam("qty") Integer qty);
}

@FeignClient("wallet-service")
public interface WalletClient {
    @PostMapping("/wallet/spend")
    ApiEnvelope<Void> spend(@RequestParam("userId") Long userId,
                            @RequestParam("amount") BigDecimal amount);
}
```

```java
@Slf4j
@Service
public class BookingAppService {

    @Resource
    private BookingMapper bookingMapper;
    @Resource
    private StockClient stockClient;
    @Resource
    private WalletClient walletClient;

    @GlobalTransactional(name = "ledger-place-booking", rollbackFor = Exception.class)
    public void place(Booking booking) {
        log.info("draft booking");
        bookingMapper.insertDraft(booking);
        log.info("consume stock");
        stockClient.consume(booking.getSkuId(), booking.getQty());
        log.info("spend wallet");
        walletClient.spend(booking.getUserId(), booking.getAmount());
        log.info("mark booking done");
        bookingMapper.markDone(booking.getUserId(), 0);
    }
}
```

`rollbackFor` 写 `Exception.class`，受检异常也会带动全局回滚。漏掉这个属性时，有些异常只会回滚本地，远端已经提交的分支会留下。

#### 业务类

```java
@RestController
public class BookingController {

    @Resource
    private BookingAppService bookingAppService;

    @GetMapping("/booking/place")
    public ApiEnvelope<Void> place(Booking booking) {
        bookingAppService.place(booking);
        return ApiEnvelope.ok("placed");
    }
}
```

实验用 GET 是为了浏览器能直接打。正式接口应改成 POST。

#### Config配置

AT 需要数据源代理，才能在业务 SQL 前后截获连接、写 undo 日志。关闭数据源自动配置后，自己注册 `DataSourceProxy`，并把 MyBatis 的工厂指到代理上。

```java
@Configuration
public class TxDataSourceConfig {

    @Bean
    @ConfigurationProperties("spring.datasource")
    public DataSource rawDataSource() {
        return DataSourceBuilder.create().type(DruidDataSource.class).build();
    }

    @Bean
    public DataSourceProxy bookingDataSource(DataSource rawDataSource) {
        return new DataSourceProxy(rawDataSource);
    }

    @Bean
    public SqlSessionFactory bookingSqlSession(DataSourceProxy bookingDataSource) throws Exception {
        SqlSessionFactoryBean factory = new SqlSessionFactoryBean();
        factory.setDataSource(bookingDataSource);
        factory.setMapperLocations(new PathMatchingResourcePatternResolver()
                .getResources("classpath:mapper/*.xml"));
        factory.setTransactionFactory(new SpringManagedTransactionFactory());
        return factory.getObject();
    }
}
```

```java
@Configuration
@MapperScan("lab.booking.mapper")
public class BookingMybatisConfig {
}
```

若使用较新的 Seata 自动代理，可以改为配置 `seata.enable-auto-data-source-proxy=true`，不必再手写 `DataSourceProxy`。手写和自动代理不要同时开启，否则会套两层代理。

#### 启动类

排除数据源自动配置，否则容器里会有一个没被 Seata 代理的数据源，undo 日志不会写。

```java
@SpringBootApplication(exclude = DataSourceAutoConfiguration.class)
@EnableDiscoveryClient
@EnableFeignClients
public class BookingApp2101 {
    public static void main(String[] args) {
        SpringApplication.run(BookingApp2101.class, args);
    }
}
```

### 新建Storage-Module

库存模块 `stock-service` 监听 2102，库用 `tx_stock`。它不开启全局事务，只作为 RM 参与调用方的 XID。

提供 `POST /stock/consume`：按 `sku_id` 把 `remain` 减去数量、`consumed` 加上数量，`remain` 不足时抛出异常。Mapper 的更新语句带 `remain >= #{qty}` 条件，影响行数为 0 就视为库存不足。

本模块同样要建 `undo_log`，数据源同样走代理，并配置同一个 `ledger_tx_grp`。不要在库存方法上再套 `@GlobalTransactional`，否则会再开一个全局事务，和订单传下来的 XID 脱节。

本地事务仍建议加在扣减方法上，保证“减库存”这一条 SQL 自己是原子的。全局回滚时，Seata 用 undo 日志把这一条补回去。

### 新建账户Account-Module

账户模块 `wallet-service` 监听 2103，库用 `tx_wallet`。`POST /wallet/spend` 按 `user_id` 减少 `remain`、增加 `spent`。余额不足抛异常。

结构与库存模块对称：Nacos 注册、同一事务组、`undo_log`、数据源代理、Feign 不需要（它是调用链末端）。超时实验就加在这个方法里，用睡眠模拟慢调用。

初始数据是用户 9，额度 5000。扣减必须带条件 `remain >= amount`，避免并发下扣成负数。

### 测试

#### 数据库初始情况

- `o_stock`：sku 7，总量 80，已用 0，剩余 80
- `o_wallet`：用户 9，额度 5000，已用 0，剩余 5000
- `o_booking`：空表

#### 正常下单

请求 2101 的 `/booking/place`，参数 `userId=9`、`skuId=7`、`qty=4`、`amount=120`。

期望：订单一行且 `state=1`，库存剩余 76，账户剩余 4880。三个库的 `undo_log` 在二阶段提交后被清理，不应长期堆积。

#### 超时异常，没加@GlobalTransaction

在账户扣减方法里先睡 15 秒。Feign 读超时若仍是默认的 1 秒左右，订单服务会认为调用失败。此时若拿掉 `@GlobalTransactional`：

- 订单行可能停在草稿，`state` 仍是 0
- 库存可能已经扣掉
- 账户可能因为 Feign 重试被扣了不止一次

三个库各自提交，没有人负责补偿，数据对不齐。

#### 超时异常，添加@GlobalTransaction

把注解加回去：

```java
@GlobalTransactional(name = "ledger-place-booking", rollbackFor = Exception.class)
```

同样让账户睡 15 秒并让 Feign 超时。全局事务失败后，订单草稿消失，库存和余额回到下单前。这是 AT 用各库的 `undo_log` 做了反向补偿，不是数据库本身的两阶段提交。

若注解加了仍不回滚，先查三件事：事务组名是否和服务端一致、业务库有没有 `undo_log`、数据源是不是代理对象。

## 一部分补充

### Seata

2019 年初由蚂蚁和阿里联合开源。名字展开是 Simple Extensible Autonomous Transaction Architecture。1.0 之后接口和配置键有一波调整，新项目按 1.4 及以后的配置键来写，不要混用 0.9 的 `spring.cloud.alibaba.seata` 老前缀和 1.5 的 `seata.*` 前缀而不看文档对照表。

### 再看TC/TM/RM三大组件

- TC 就是 Seata Server 进程
- TM 就是标了 `@GlobalTransactional` 的那个业务方法所在进程
- RM 是每个参与事务的数据库一侧，由数据源代理扮演

一次下单里，订单服务既是 TM 也是 RM（它自己写了订单库），库存和账户服务只是 RM。

### 分布式事务的执行流程

1. TM 向 TC 注册全局事务，得到 XID
2. 业务按顺序访问库存库、账户库，每个 RM 向 TC 登记分支并上报
3. TM 结束方法，一阶段结束，通知 TC 提交或回滚
4. TC 汇总分支状态后做决定
5. TC 通知各 RM 提交或回滚，二阶段结束

### AT 为何不用改业务 SQL

默认的 AT 不要求业务写 try/confirm/cancel。应用仍写普通的增删改 SQL。代理在这些 SQL 前后自动记录镜像和行锁。业务感知到的只是多了一个注解。代价是：必须使用支持本地 ACID 的关系库，并且通过 JDBC 访问；锁和回滚日志也有额外写入。

### AT模式

#### 前提

- 数据库本身能做本地事务
- Java 应用经 JDBC 访问该库

不满足这两条时不要选 AT，可以评估 TCC 或 SAGA。

#### 整体机制

它是两阶段提交的一种变体，但一阶段已经把业务数据提交到库里，而不是把锁一直拿到二阶段。

- 一阶段：业务 SQL 和回滚日志放在同一个本地事务里提交，然后释放本地锁和连接
- 二阶段提交：异步删掉快照和锁，很快
- 二阶段回滚：按一阶段留下的前置镜像把数据改回去

#### 一阶段加载

代理拦截业务 SQL 的过程：

1. 解析语句，找出将要改动的行，读出改动前的值，记为 before image
2. 执行业务 SQL
3. 再读出改动后的值，记为 after image，并写上行锁

这三步在同一个本地事务里。中途失败则本地事务回滚，不会留下半条 undo 日志。

#### 二阶段提交

业务数据在一阶段已经落库。二阶段若决定提交，TC 只要求 RM 删除对应的 undo 记录和行锁，不必再执行一次业务 SQL。

#### 二阶段回滚

回滚时用 before image 覆盖当前行。覆盖前先拿当前行和 after image 比较。两者一致，说明提交后没有别人改过，可以安全还原。两者不一致，说明发生了脏写，自动回滚可能覆盖别人的更新，此时应转人工核对，而不是强行覆盖。

#### 总结

AT 把“全局原子”拆成“本地提交 + 可补偿日志”。读的时候没有准备阶段的长时间锁，吞吐比传统 XA 好；写路径上多了镜像和锁表。全局回滚的正确性依赖 undo 日志完整、数据源确实被代理，以及事务组能找到 TC。这三处任一缺失，表现都是“注解加了，数据仍然脏”。
