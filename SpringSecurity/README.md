# Spring Security 笔记

笔记对准 Spring Security 5.1 与 Spring Boot 2.1，配置仍使用 `WebSecurityConfigurerAdapter`。新版本改由 `SecurityFilterChain` Bean 组装过滤器，概念相同。

认证回答“你是谁”，授权回答“你能做什么”。单机环境可以把结果放进 Session；多台机器不再共享 Session 时，笔记改用带签名的令牌。最后一篇把授权从“用户访问自己的系统”扩展到“一个系统代表用户访问另一个系统的资源”，也就是 OAuth 2.0 的四种授权方式。

## 目录

- [初识 Spring Security](./1_初识SpringSecurity/README.md)
- [在 MVC 工程中的认证与授权](./2_SpringSecurity在MVC项目中的使用/README.md)
- [单机 Spring Boot 工程](./3_SpringSecurity在单机环境下的使用/README.md)
- [多服务下的令牌认证](./4_SpringSecurity在分布式环境下的使用/README.md)
- [OAuth 2.0](./5_OAuth2.0介绍/README.md)

安全相关的段落只说明过滤链、令牌结构和角色分工。示例口令和关闭校验的写法不能直接搬进生产。
