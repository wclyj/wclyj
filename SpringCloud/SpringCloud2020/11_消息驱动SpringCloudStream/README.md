# SpringCloud Stream 消息驱动

## 为什么引入消息驱动？

业务里一旦出现“下单后通知库存、通知积分、通知短信”，就会引入消息中间件。常见的有：

- ActiveMQ
- RabbitMQ
- RocketMQ
- Kafka

中台用一种，后台用另一种，客户端 API 完全不同。交换器、主题、分区、确认方式都要各写一套。四套产品都学一遍也不现实。

痛点是：希望业务代码不再关心具体中间件的类和方法，只通过一种绑定方式声明“我往哪发、我从哪收”，切换中间件时改配置而不是改调用。

Spring Cloud Stream 就是这层适配。它挡住客户端差异，应用只跟 Stream 的通道打交道，从而少写一套中间件专用代码。

## 消息驱动概述

### 是什么

它把不同中间件收成同一套编程模型，降低更换成本。

这和 Hibernate 的思路接近：底下可以是不同数据库，上面用 Session 和 HQL，调用方不必为每种方言写 JDBC。Stream 里，调用方不必为 Rabbit 的交换器或 Kafka 的主题各写一个客户端。

### 什么是SpringCloudStream

官方把它定义成构建消息驱动微服务的框架。

应用通过 inputs 或 outputs 和 Binder 交互。绑定关系由配置声明，Binder 再去连真正的中间件。

因此要搞懂的是通道和绑定，而不是某个中间件的原生 API。Stream 底层借用 Spring Integration 把应用和代理接起来，并给部分供应商做了自动配置。模型上它强调三件事：发布订阅、消费组、分区。

这一代绑定器主要覆盖 RabbitMQ 和 Kafka。

### SpringCloudStrem设计思想

#### 标准MQ

普通消息系统里：

- 生产者和消费者之间传递的是 Message
- 消息必须经过 Channel
- 谁来收发，由 `SubscribableChannel` 上的 `MessageHandler` 负责

应用不直接摸队列，只往通道送、从通道取。

#### 为什么用SpringCloudStream

RabbitMQ 有 exchange 和 queue，Kafka 有 topic 和 partition。两者的路由模型对不上。

项目先选了其中一个，后来要迁到另一个，如果业务代码直接依赖原生客户端，就等于把中间件焊进了领域服务。Stream 在中间放一个绑定器，相当于翻译：同一段发送代码，配置改成另一种 binder 类型，就能把消息送到另一套中间件。

#### SpringCloudStream为什么能屏蔽底层差异

没有绑定器时，Spring Boot 应用必须按中间件各自的连接、确认、序列化方式来写。这些差异来自产品设计，不是业务差异。把 Binder 定义成中间层之后，应用只看见统一的 Channel，中间件细节留在绑定器里。

#### Binder

- input：消费者一侧的通道
- output：生产者一侧的通道

绑定器还可以按配置切换。同一套接口，从 Rabbit 换到 Kafka，改的是 binder 类型和目的地名称，发送代码保持不变。服务因此可以把精力放回业务流程。

Stream 的通信遵循发布订阅。用一个主题做广播：在 Rabbit 里主题对应 Exchange，在 Kafka 里对应 Topic。

#### Stream标准流程套路

生产者和消费者都只和 Stream 对话，不持有中间件连接对象。

- Binder：连接中间件并抹平差异
- Channel：队列的抽象，负责把消息存下来再转发出去
- Source 与 Sink：以 Stream 自己为参照。从应用往外发是 Source（输出），应用接收是 Sink（输入）

#### 编码中的注解

旧版 API 用这三个注解把通道绑到应用上：

- `@EnableBinding`：声明本进程使用哪些通道
- `@Output` / `Source`：发送侧
- `@Input` / `Sink`：接收侧
- `@StreamListener`：在方法上订阅某个输入通道

新版函数式写法用 `Supplier`、`Consumer`、`Function` 代替这些注解。下面的例子仍用注解模型，方便和配置项 `output`、`input` 对照。

## 案例说明

实验前先起 RabbitMQ，管理端口按实际映射来，不要假定一定是 15672。三个进程：

- `mq-bridge-pub`，监听 6711，负责发
- `mq-bridge-sub-a`，监听 6712，负责收
- `mq-bridge-sub-b`，监听 6713，负责收

## 消息驱动之生产者

### 引入依赖

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-stream-rabbit</artifactId>
</dependency>
```

### 修改yml

绑定器单独起名，通道再引用它。目的地用自己的交换器名，不要沿用教程里的名字。

```yaml
spring:
  application:
    name: mq-bridge-pub
  cloud:
    stream:
      binders:
        labRabbit:
          type: rabbit
          environment:
            spring:
              rabbitmq:
                host: mq.lab.internal
                port: 5673
                username: stream
                password: stream-secret
      bindings:
        output:
          destination: noticeHub
          content-type: application/json
          binder: labRabbit
```

`output` 是通道名，要和代码里注入的通道一致。`destination` 才会变成 Rabbit 上的交换器。

### 业务类

#### 发送消息的接口

```java
public interface NoticePublisher {
    String publish();
}
```

#### 发送消息的接口实现类

```java
@EnableBinding(Source.class)
public class NoticePublisherImpl implements NoticePublisher {

    @Resource
    private MessageChannel output;

    @Override
    public String publish() {
        String token = UUID.randomUUID().toString();
        output.send(MessageBuilder.withPayload(token).build());
        return token;
    }
}
```

`Source.OUTPUT` 对应配置里的 `output`。这里不再写 Dao，发送动作就是往通道丢一条消息。

#### Controller

```java
@RestController
public class NoticeController {

    @Resource
    private NoticePublisher noticePublisher;

    @GetMapping("/notice/push")
    public String push() {
        return noticePublisher.publish();
    }
}
```

#### 测试

打开 Rabbit 管理台，能看到新建的交换器 `noticeHub`。它来自 `bindings.output.destination`。

访问本机 6711 的 `/notice/push`。管理台里该交换器出现入站消息，说明生产路径已经打通。此时还没有队列绑定的话，消息会被丢弃，这是发布订阅在没有订阅者时的正常现象。

## 消息驱动之消费者

### 引入依赖

与生产者相同，仍是 `spring-cloud-starter-stream-rabbit`。

### 修改yml

消费者把通道名改成 `input`，目的地必须和生产侧的 `noticeHub` 一致，否则两边不在同一个交换器上。

```yaml
spring:
  application:
    name: mq-bridge-sub
  cloud:
    stream:
      binders:
        labRabbit:
          type: rabbit
          environment:
            spring:
              rabbitmq:
                host: mq.lab.internal
                port: 5673
                username: stream
                password: stream-secret
      bindings:
        input:
          destination: noticeHub
          content-type: application/json
          binder: labRabbit
          group: lane-north
```

`group` 先写上，后面解释它为什么能消掉重复消费。

### 业务类

```java
@Component
@EnableBinding(Sink.class)
public class NoticeListener {

    @Value("${server.port:6712}")
    private String port;

    @StreamListener(value = Sink.INPUT)
    public void onNotice(Message<String> message) {
        System.out.println("sub got " + message.getPayload() + " on " + port);
    }
}
```

6712 和 6713 使用同一份监听代码，靠端口区分实例。

## 分组消费

同时运行：

- RabbitMQ
- 注册中心（若服务还要互相发现）
- 6711 生产
- 6712 消费
- 6713 消费

### 运行后有两个问题

- 两个消费者都收到了同一条消息，出现重复处理
- 消费者不在线时，消息是否还在

### 消费

6712 和 6713 若各自独占一条队列，又都绑定到同一个交换器，就会各收一份。订单集群里这是错的：一张订单不能被两台机器同时扣库存。

Stream 用消费组解决。同一个 `group` 里的多个实例是竞争关系，一条消息只会被其中一台拿走。不同组各自有队列，组与组之间仍然是广播，会重复收到。

集群部署为了可用会起多个实例。多数业务希望“发给这个微服务一次，就被处理一次”。两个实例应用名相同，并不自动共享组，必须显式写同一个 `group`。

### 分组

#### 原理

把 6712 和 6713 放进 `lane-north`。组内竞争，组间互不影响。配置片段：

```yaml
bindings:
  input:
    destination: noticeHub
    content-type: application/json
    binder: labRabbit
    group: lane-north
```

连续打 6711 的 `/notice/push` 六次。6712 大约处理其中三条，6713 处理另外三条，总和是六而不是十二。两边谁多谁少取决于投递时的竞争，不保证严格交替，但不会双倍处理。

若把 6713 的 group 改成 `lane-south`，则每条消息两台都会收到。需要广播时用不同组，需要集群互斥时用同一组。

## 消息持久化

分组除了消重，还决定队列是否 durable、消费者离线时消息是否留下。

### 案例

- 停掉 6712 和 6713。去掉 6712 的 group，保留 6713 的 `lane-north`
- 先由 6711 发送 4 条
- 再启动 6712：没有组，收不到刚才那 4 条
- 再启动 6713：有组，启动后把积压的消息消费掉

没有组时，Stream 使用匿名、自动删除的队列，订阅者一走队列就没了，离线期间的消息无处可留。写了组名之后，队列按组保留，消费者重新连上就能把积压读走。所以生产配置里，只要这个服务需要“至少处理一次”，`group` 就不要省。
