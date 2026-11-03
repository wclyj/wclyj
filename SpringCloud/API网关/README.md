# API 网关

## 引入 pom 依赖

引入 gateway 启动器。按注册名转发时再加 Eureka 客户端。

## 开启 Eureka 服务名称转发

打开定位器，路径使用面板上的大写服务名。

```yaml
spring:
  cloud:
    gateway:
      discovery:
        locator:
          enabled: true
```
