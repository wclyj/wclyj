# SpringBoot 解决时区问题

## 前言

刚写下的评论，页面却显示成约八小时前。库里的时间戳和机器上的 `date` 一致，偏差来自应用把同一瞬间按另一个时区解释。JVM 默认 UTC、会话在东八区时，展示正好差八小时。

## 解决方法

连接、JSON 和进程默认时区要写成同一个地区。

1. 在 JDBC 串上声明会话时区，避免驱动按自己的默认做转换：

```
jdbc:mysql://127.0.0.1:3306/appdb?serverTimezone=Asia/Shanghai&characterEncoding=utf8&useSSL=false
```

2. 让 Jackson 用同一时区输出时间：

```yaml
spring:
  jackson:
    time-zone: "Asia/Shanghai"
    date-format: "yyyy-MM-dd HH:mm:ss"
```

3. 启动类里把 JVM 默认时区钉住。`main` 要写在 `SpringApplication.run` 之前；`@PostConstruct` 再设一次，防止后续组件改回 UTC。

```java
@PostConstruct
public void pinZone() {
    TimeZone.setDefault(TimeZone.getTimeZone(ZoneId.of("Asia/Shanghai")));
}

public static void main(String[] arguments) {
    TimeZone.setDefault(TimeZone.getTimeZone(ZoneId.of("Asia/Shanghai")));
    SpringApplication.run(PortalApp.class, arguments);
}
```
