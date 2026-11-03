# 用注解创建队列和交换机

## 前言

消息拓扑如果只在管理后台手工点出来，环境一多就容易点错。名字差一个字母，发送端仍显示投递成功，消费端却永远空闲，排查时还要对照控制台和代码两套名称。把交换机、队列和绑定都写成 Spring Bean 之后，应用启动就会按代码把拓扑补齐，仓库即文档。

## 创建交换机

直连交换机按路由键精确匹配。`durable` 设为 true 时，代理重启后交换机定义还在。Bean 的名字后面绑定要用，单独起一个常量。

```java
public static final String ORDER_EXCHANGE = "order.direct";

@Bean(ORDER_EXCHANGE)
public DirectExchange orderExchange() {
    return new DirectExchange(ORDER_EXCHANGE, true, false);
}
```

第三个参数是 `autoDelete`：没有绑定残留时是否删掉交换机。业务交换机保持 false。

## 创建队列

队列同样做成 Bean。`QueueBuilder.durable` 等价于持久化、非独占、不自动删除，比重载一个只传名字的构造更一目了然。

```java
public static final String ORDER_CREATED = "order.created";
public static final String ORDER_MAIL = "order.mail";
public static final String ORDER_SMS = "order.sms";

@Bean(ORDER_CREATED)
public Queue createdQueue() {
    return QueueBuilder.durable(ORDER_CREATED).build();
}

@Bean(ORDER_MAIL)
public Queue mailQueue() {
    return QueueBuilder.durable(ORDER_MAIL).build();
}

@Bean(ORDER_SMS)
public Queue smsQueue() {
    return QueueBuilder.durable(ORDER_SMS).build();
}
```

## 绑定交换机和队列

绑定要把已经放进容器的队列和交换机拿回来，所以参数上加 `@Qualifier`，值是前面 `@Bean` 的名字。`DirectExchange` 的 `to(...)` 重载可以直接 `.with(路由键)`，不必再调用无参数的绑定形式。

三个路由键各自对应一个队列。短信那条如果误写成订单队列的键，短信消费者不会收到任何消息，这是手工绑定里最常见的笔误，代码评审时对着常量名看一眼就能发现。

```java
public static final String CREATED_KEY = "order.created";
public static final String MAIL_KEY = "order.mail";
public static final String SMS_KEY = "order.sms";

@Bean
public Binding createdBinding(@Qualifier(ORDER_CREATED) Queue queue,
                              @Qualifier(ORDER_EXCHANGE) DirectExchange exchange) {
    return BindingBuilder.bind(queue).to(exchange).with(CREATED_KEY);
}

@Bean
public Binding mailBinding(@Qualifier(ORDER_MAIL) Queue queue,
                           @Qualifier(ORDER_EXCHANGE) DirectExchange exchange) {
    return BindingBuilder.bind(queue).to(exchange).with(MAIL_KEY);
}

@Bean
public Binding smsBinding(@Qualifier(ORDER_SMS) Queue queue,
                          @Qualifier(ORDER_EXCHANGE) DirectExchange exchange) {
    return BindingBuilder.bind(queue).to(exchange).with(SMS_KEY);
}
```

删掉控制台上旧的同名对象再启动通知服务，能在管理界面看到交换机下挂着三条绑定。发送时用的路由键必须和这里的常量一致。

## 完整代码

消息体如果是对象，再补一个 JSON 转换器，生产者和消费者共用即可。

```java
@Configuration
public class OrderAmqpConfig {

    public static final String ORDER_EXCHANGE = "order.direct";
    public static final String ORDER_CREATED = "order.created";
    public static final String ORDER_MAIL = "order.mail";
    public static final String ORDER_SMS = "order.sms";
    public static final String CREATED_KEY = "order.created";
    public static final String MAIL_KEY = "order.mail";
    public static final String SMS_KEY = "order.sms";

    @Bean(ORDER_EXCHANGE)
    public DirectExchange orderExchange() {
        return new DirectExchange(ORDER_EXCHANGE, true, false);
    }

    @Bean(ORDER_CREATED)
    public Queue createdQueue() {
        return QueueBuilder.durable(ORDER_CREATED).build();
    }

    @Bean(ORDER_MAIL)
    public Queue mailQueue() {
        return QueueBuilder.durable(ORDER_MAIL).build();
    }

    @Bean(ORDER_SMS)
    public Queue smsQueue() {
        return QueueBuilder.durable(ORDER_SMS).build();
    }

    @Bean
    public Binding createdBinding(@Qualifier(ORDER_CREATED) Queue queue,
                                  @Qualifier(ORDER_EXCHANGE) DirectExchange exchange) {
        return BindingBuilder.bind(queue).to(exchange).with(CREATED_KEY);
    }

    @Bean
    public Binding mailBinding(@Qualifier(ORDER_MAIL) Queue queue,
                               @Qualifier(ORDER_EXCHANGE) DirectExchange exchange) {
        return BindingBuilder.bind(queue).to(exchange).with(MAIL_KEY);
    }

    @Bean
    public Binding smsBinding(@Qualifier(ORDER_SMS) Queue queue,
                              @Qualifier(ORDER_EXCHANGE) DirectExchange exchange) {
        return BindingBuilder.bind(queue).to(exchange).with(SMS_KEY);
    }

    @Bean
    public MessageConverter jsonBodies() {
        Jackson2JsonMessageConverter converter = new Jackson2JsonMessageConverter();
        converter.setCreateMessageIds(true);
        return converter;
    }
}
```
