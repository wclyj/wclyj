# Nacos 日志把磁盘写满

## 前言

注册中心不应该是最能写盘的那个进程，但运维侧经常看到 Nacos 目录几天就长出几个 GB。实例数不多时也会这样。下面分开看服务端访问日志、运行日志级别，以及客户端心跳打印。

## 事件背景

增长最快的常常是 `{nacos.home}/logs/access_log.yyyy-MM-dd.log`。里面是 HTTP 访问，不是业务审计。典型路径包括：

- 续约：`/nacos/v1/ns/instance/beat`
- 拉实例：`/nacos/v1/ns/instance/list`
- 探活：`/nacos/v1/ns/operator/metrics` 或服务状态检查

Nacos 服务端跑在内嵌 Tomcat 上，访问日志由容器打开。默认配置没有按天封顶，也没有总容量上限。客户端越多，续约和拉清单越频繁，这个文件就越厚。它和业务日志混在同一块盘上时，先满的往往是系统盘。

## 解决方案

访问日志开关在服务端 `conf/application.properties`，默认类似：

```properties
server.tomcat.accesslog.enabled = true
server.tomcat.accesslog.pattern=%h %l %u %t "%r" %s %b %D %{User-Agent}i
server.tomcat.basedir=
```

三项分别是：是否写入、行格式、文件放在哪。测试环境可以直接关掉：

```properties
server.tomcat.accesslog.enabled = false
```

改完要重启 Nacos 进程。生产若整段关掉，事后排「谁在打某个接口」会没有原始访问行。更稳的做法是留下日志，但只保留有限天数。

下面的脚本放在 `/etc/cron.daily/trim-nacos-access`，删除 10 天前的那一个文件，并把自己的动作记到单独的小文件，避免再去翻已经删掉的访问日志。

```bash
#!/bin/bash
audit=/var/log/nacos-access-trim.log
stamp=$(date -d "10 days ago" +%Y-%m-%d)
target=/opt/registry/nacos/logs/access_log.${stamp}.log

if [ ! -f "$audit" ]; then
  echo "trim access_log older than 10 days" >> "$audit"
fi
if [ -f "$target" ]; then
  rm -f "$target"
  echo "[$(date '+%F %T')] removed $target" >> "$audit"
fi
```

路径按实际安装目录改。`rm` 只针对拼出来的那一个日期文件，不要对整个 `logs` 目录做递归删除。脚本要用 `chmod +x`，并用 `run-parts --test` 看 cron.daily 是否会执行它。

这仍是外围补救：进程照样写，系统再去删。版本自己如果支持滚动和保留份数，应优先改日志配置。

## 日志级别动态调整

1.1.3 之前，服务端不少日志不能在运行中改级别。此版本之后可以用 HTTP 接口临时调整，不必重启。

把命名模块里 raft 日志调到 ERROR，把配置模块的 dump 调到 WARN：

```bash
curl -X PUT "http://10.4.1.20:8848/nacos/v1/ns/operator/log?logName=naming-raft&logLevel=error"
curl -X PUT "http://10.4.1.20:8848/nacos/v1/cs/ops/log?logName=config-dump&logLevel=warn"
```

`logName` 对应 `nacos-logback.xml` 里的记录器，不是文件名的随意缩写。改错名字不会报明显错误，只是没有文件变安静。级别在进程重启后可能回到配置文件中的值，长期设置仍要落在 XML 或 `application.properties`。

## 客户端日志

1.1.3 起客户端减少了续约与长轮询的默认打印。业务进程仍可以再收紧：

```yaml
logging:
  level:
    com.alibaba.nacos: WARN
```

或在启动参数里分开控制命名客户端和配置客户端，默认多是 INFO：

```text
-Dcom.alibaba.nacos.naming.log.level=warn
-Dcom.alibaba.nacos.config.log.level=warn
```

这组系统属性从 1.0.0 客户端就能用。它影响的是业务进程，不是 Nacos 服务端的 `access_log`。两边要分别处理。

## 更细的日志配置

`conf/nacos-logback.xml` 里每个功能一块 appender。以命名服务为例，滚动策略可以改成更小的单文件和更少的保留份数：

```xml
<appender name="naming-server" class="ch.qos.logback.core.rolling.RollingFileAppender">
    <file>${LOG_HOME}/naming-server.log</file><!-- 活动文件 -->
    <rollingPolicy
            class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
        <!-- 单文件与保留份数按磁盘预算改 -->
        <fileNamePattern>${LOG_HOME}/naming-server.%d{yyyy-MM-dd}.%i.log.gz</fileNamePattern>
        <maxFileSize>512MB</maxFileSize>
        <maxHistory>5</maxHistory>
        <totalSizeCap>3GB</totalSizeCap>
    </rollingPolicy>
    <encoder>
        <pattern>%d{HH:mm:ss.SSS} %-5level %msg%n</pattern>
        <charset>UTF-8</charset>
    </encoder>
</appender>
<!-- naming-server 只是其中一块，config 模块要单独改 -->
```

`totalSizeCap` 是这块日志的总量上限，`maxHistory` 是保留天数。改 XML 后需要重启才加载。不要只改 `maxFileSize` 而把历史份数留成几十份，总量仍然会撑满磁盘。

## 小结

访问日志、模块运行日志、客户端续约日志是三条线。测试环境可以关访问日志；生产更适合按天删除或让滚动策略自己封顶。运行级别能用接口临时调，客户端则用日志级别或 JVM 参数。磁盘告警先看 `access_log` 的日期文件大小，再决定是删历史还是降级别。
