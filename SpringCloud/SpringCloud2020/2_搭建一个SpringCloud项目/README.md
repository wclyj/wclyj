# 搭建一个 Spring Cloud 项目

父工程只负责版本，业务代码放在子模块里。下面用一套最小可运行的骨架说明模块怎么拆、先启动谁。

## 概念

一套最基本的调用链包含三块：注册中心、提供接口的服务、发起调用的服务。注册中心保存名字与地址；提供者启动后登记自己；调用方按名字取地址，而不是把 IP 写进代码。

## 架构

```text
父工程 shop-cloud
├── registry-server     端口 7101
├── billing-api         端口 8011，注册名 billing-api
└── checkout-web        端口 8090，通过服务名访问 billing-api
```

父 POM 用 `dependencyManagement` 锁定 Spring Boot 与 Spring Cloud 的发行版，子模块不再各自写版本号，避免客户端和服务端不在同一代。

## 搭建步骤

1. 新建父工程，`packaging` 设为 `pom`，声明两个发行版导入。
2. 建 `registry-server`，只放注册中心依赖和它的端口。
3. 建 `billing-api`，加 Web 与发现客户端，启动类打开服务发现。
4. 建 `checkout-web`，同样注册上去，并用带负载均衡的 HTTP 客户端按名字调用。
5. 先起注册中心，再起 `billing-api`，最后起 `checkout-web`。

提供者示例：

```yaml
server:
  port: 8011
spring:
  application:
    name: billing-api
eureka:
  client:
    service-url:
      defaultZone: http://localhost:7101/eureka/
```

调用方不要写 `http://localhost:8011`，应写 `http://billing-api`，并在 `RestTemplate` 上打开负载均衡，这样后面临时加实例也不必改调用方。

## 配置约定

- 服务名用小写中划线，全公司保持一种风格。
- 端口按层段划分：注册中心 71xx，接口服务 80xx，边缘入口 80xx 里再错开。
- 本地配置只留端口、名字和注册地址；会随环境变的项以后再外置。

## 易错点

注册中心没起来就启动子服务，进程会连不上并反复重试。两个模块若误用了同一个 `spring.application.name`，会被当成同一个服务的两个实例。父 POM 没导入发行版时，传递依赖的版本容易互相打架，编译期不一定报错，运行期才出现方法找不到。
