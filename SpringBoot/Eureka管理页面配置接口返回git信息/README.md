# Eureka 管理页面配置 /actuator/info 接口返回 git 信息

## 前言

注册中心列表里，每个实例都有一条指向自己的链接。点进去如果是空白页，多半不是页面坏了，而是这个服务根本没把 `info` 端点的内容准备好。Spring Boot Admin 以及 Eureka 自带的详情，都会来拉 `/actuator/info`。仓库用 Git 管理时，最有用的一块就是“这次部署对应哪次提交”。

要让该端点带上提交信息，先把 Actuator 放进依赖：

Maven 坐标写成 `org.springframework.boot:spring-boot-starter-actuator`。Gradle 工程直接：

```groovy
dependencies {
    implementation "org.springframework.boot:spring-boot-starter-actuator"
}
```

再在构建里加上提交号插件。`revision` 目标会在编译产物里写一份 `git.properties`，Actuator 启动时读它。下面把目标绑到 `initialize`，并允许在没有 `.git` 的打包机上跳过失败（例如只拿到源码压缩包的时候）：

```xml
<plugin>
    <groupId>
        pl.project13.maven
    </groupId>
    <artifactId>
        git-commit-id-plugin
    </artifactId>
    <version>4.9.10</version>
    <executions>
        <execution>
            <id>capture-revision</id>
            <phase>initialize</phase>
            <goals>
                <goal>
                    revision
                </goal>
            </goals>
        </execution>
    </executions>
    <configuration>
        <generateGitPropertiesFile>true</generateGitPropertiesFile>
        <failOnNoGitDirectory>false</failOnNoGitDirectory>
    </configuration>
</plugin>
```

Maven 面板里会出现 `git-commit-id:revision`。执行一次后，到 `target/classes` 查看 `git.properties`。内容类似（下面是虚构示例，不要把真实邮箱写进笔记或仓库）：

```properties
git.branch=main
git.commit.id.abbrev=c4e91ab
git.commit.id.full=c4e91ab0e2f14a6c8d0b77a1e5c90344aa11d002
git.commit.time=2024-11-02T09:30:00+0800
git.build.version=2.4.0
git.dirty=false
```

Boot 2 默认不一定把所有管理端点暴露到 Web。确认 `info` 在白名单里，并把 Git 模式开成 `full`，否则响应里可能只有分支名：

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info
  info:
    git:
      mode: full
```

重启注册中心，再点开刚才那条实例链接，页面上应能看到分支、短提交号和构建版本。若仍是空的，先直接请求 `http://主机:端口/actuator/info`：404 说明端点没暴露，200 但没有 `git` 节点说明 `git.properties` 没打进 classpath。
