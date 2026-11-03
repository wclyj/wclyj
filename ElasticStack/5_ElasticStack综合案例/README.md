# Elastic Stack 综合案例

这一篇把前面的 Elasticsearch、Filebeat、Logstash、Kibana 串成一条行为日志管道。目标不是引入新组件，而是看清每段各自负责什么。

## 流程说明

业务进程持续写本地日志，一行代表一次会员动作。Filebeat 只负责把新行送出，不做字段解析。Logstash 按分隔符切开、转换类型，再写入 Elasticsearch。Kibana 读取这些文档，做成按时间计数的柱状图和按动作分类的饼图。

样例行：

```text
[INFO] 2024-06-18 09:12:04 [main] - ACT|1842|浏览货架|2024-06-18 09:02:11
[INFO] 2024-06-18 09:12:06 [main] - ACT|3901|领取礼券|2024-06-18 08:40:22
```

竖线后面的四段才是业务载荷：固定标记、会员编号、动作、动作发生时间。日志框架自己的级别和时间留在前缀里，解析时要知道下标从哪里算。

图表和仪表盘都是后建的，索引里不会预先出现这些对象。

## 应用介绍

实验里不接真实交易系统，用一个循环伪造动作。动作词表和会员编号范围都按书店场景来写。

```java
package lab.shelf;

import org.apache.commons.lang3.RandomUtils;
import org.joda.time.DateTime;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class ActionFeed {
    private static final Logger log = LoggerFactory.getLogger(ActionFeed.class);
    private static final String[] ACTIONS = {
            "浏览货架", "收藏书目", "加入书单", "提交订单", "使用礼券", "领取礼券", "检索书名"
    };

    public static void main(String[] args) throws InterruptedException {
        while (true) {
            Thread.sleep(RandomUtils.nextLong(200, 4_000));
            long memberId = RandomUtils.nextLong(1, 8_000);
            String action = ACTIONS[RandomUtils.nextInt(0, ACTIONS.length)];
            DateTime now = DateTime.now();
            String when = now
                    .minusHours(RandomUtils.nextInt(0, Math.max(now.getHourOfDay(), 1)))
                    .minusMinutes(RandomUtils.nextInt(0, Math.max(now.getMinuteOfHour(), 1)))
                    .toString("yyyy-MM-dd HH:mm:ss");
            log.info("ACT|{}|{}|{}", memberId, action, when);
        }
    }
}
```

打包后放到实验机，用脚本后台启动，标准输出和错误都进同一个文件，方便 Filebeat 只盯一个路径：

```bash
mvn -q -DskipTests package
```

```bash
#!/bin/bash
nohup java -Xms256m -Xmx512m -jar shelf-feed.jar >> /var/log/shelf/feed.log 2>&1 &
```

```bash
./run-feed.sh
tail -f /var/log/shelf/feed.log
```

能持续看到 `ACT|` 开头的行，再往下做采集。文件不增长时，先别查 Elasticsearch。

## 配置 Filebeat

新建 `shelf-feed.yml`，输入是日志文件，输出是本机 Logstash 的监听端口，而不是 Elasticsearch。这样原始行会先经过清洗。

```yaml
filebeat.inputs:
  - type: log
    enabled: true
    paths:
      - /var/log/shelf/feed.log
setup.template.settings:
  index.number_of_shards: 1
output.logstash:
  hosts: ["127.0.0.1:5044"]
```

Logstash 没在 5044 上监听时，Filebeat 会反复报连接失败。顺序是先让管道起来，再启动：

```bash
./filebeat -e -c shelf-feed.yml
```

## 配置 Logstash

### 先输出到控制台

第一版管道只证明 Beats 输入能收到事件。新建 `shelf-dashboard.conf`：

```ruby
input {
  beats {
    port => 5044
  }
}
output {
  stdout { codec => rubydebug }
}
```

```bash
./bin/logstash -f shelf-dashboard.conf
```

启动完成后再开 Filebeat。控制台里每条事件的 `message` 应是完整的一行日志。这一步不要加过滤器，方便区分“没收到”和“解析坏了”。

### 接通 Elasticsearch

确认有流量之后再清洗。`mutate` 的 `split` 会把 `message` 变成数组。给新字段赋值时，引用数组元素必须写成 `%{[message][1]}` 这种带方括号的字段路径。

```ruby
input {
  beats { port => 5044 }
}
filter {
  mutate {
    split => { "message" => "|" }
  }
  mutate {
    add_field => {
      "memberId" => "%{[message][1]}"
      "action"   => "%{[message][2]}"
      "actedAt"  => "%{[message][3]}"
    }
  }
  mutate {
    convert => {
      "memberId" => "integer"
      "action"   => "string"
      "actedAt"  => "string"
    }
  }
  mutate {
    remove_field => ["host"]
  }
}
output {
  elasticsearch {
    hosts => ["http://127.0.0.1:9200"]
    index => "shelf-act-%{+YYYY.MM.dd}"
  }
}
```

`memberId` 转成整数后才能做范围过滤；动作名保持字符串。重启 Logstash 后，文档里应同时存在拆开的数组和三个新字段。

### 问题 1：字段路径写错

日志里若出现 `Invalid FieldReference: message[1]`，说明 `%{message[1]}` 少了一层方括号。Logstash 把 `message` 当成普通字段名去找名为 `message[1]` 的键，而不是取数组下标。改成 `%{[message][1]}` 后这条警告会停。

### 问题 2：host 字段类型冲突

Filebeat 送来的 `host` 是对象，里面有主机名等子字段。索引模板若已经把 `host` 映射成 text，写入会被拒绝，错误类似 `failed to parse field [host] of type [text]`。实验管道用不到这个对象，直接 `remove_field => ["host"]`。若业务需要主机名，应在写入前把 `host.name` 抄到一个新的 keyword 字段，而不是让对象和文本抢同一个名字。

## 启动 Elasticsearch

写入之前集群必须已经在听 9200。用非 root 账号启动：

```bash
su - observe
cd /opt/observe/elasticsearch
./bin/elasticsearch
```

本机 `curl http://127.0.0.1:9200` 能返回版本 JSON，再看 Logstash 是否还在报连接拒绝。

## 启动 Kibana

```bash
./bin/kibana
```

浏览器打开 `http://<主机>:5601`。

### 添加索引模式

在索引模式页面使用 `shelf-act-*`。时间字段选 `@timestamp`（事件进入管道的时间）或你另外解析出的日期字段。选错时间字段时，图会全部堆在一个桶里或直接为空。

### 柱状图

Visualize 里新建 Vertical Bar，索引模式选 `shelf-act-*`。纵轴 Count，横轴 Date Histogram。时间范围改成最近 30 分钟，柱子应随 `ActionFeed` 的输出升高。保存为“动作计数”。

### 饼图

新建 Pie，同样的索引模式。Buckets 选择 Split Slices，聚合用 Terms，字段选 `action.keyword`。若字段列表里只有 `action` 而没有 keyword 子字段，说明它被动态映射成了 text 且没有多字段，需要在管道里把动作写成 keyword，或重建索引。更新之后每个动作一块扇区。可以在界面上为扇区指定标签，避免直接显示分析器切开的词。

### 数据表

再做一个 Data Table，列里放 `memberId`、`action`、`actedAt`。它适合和柱状图放在一起核对原始记录。

## 制作仪表盘

Dashboard 里新建一屏，把柱状图、饼图和数据表都加进去。时间选择器作用于整屏，点饼图上的某一块动作时，其余图会带上同样的筛选。到这里，一条从应用日志到可视化的链路就闭合了。
