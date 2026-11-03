# Logstash 入门

## 介绍

Logstash 是跑在服务端的数据处理管道。它能同时从多个输入拿事件，做解析和改写，再写到指定的存储。在 Elastic Stack 里，这个存储通常是 Elasticsearch。

从数据流上看：原始事件若已经结构良好，采集器可以直接写入 Elasticsearch；需要切割字段、转换类型或按条件路由时，才把事件先送到 Logstash。Logstash 用 Java 编写，启动要拉起虚拟机，冷启动比 Beats 慢，所以不适合在每台业务机上都放一个 JVM 做采集。常见分工是 Beats 负责采集，Logstash 负责加工。

## 用途

输入可以是文件、标准输入、Beats、数据库或消息队列。过滤阶段识别字段并转成统一结构。输出除了 Elasticsearch，也可以是标准输出、文件或消息系统，便于先在控制台把管道调通，再切换正式出口。

## 部署安装

到与集群主版本一致的发行包解压。先确认 JDK 满足该版本要求（7.x 的后期发行包可以自带运行时，仍建议用 `java -version` 看一眼）：

```bash
tar -xf logstash-7.17.9-linux-x86_64.tar.gz
mv logstash-7.17.9 /opt/observe/logstash
```

最小例子从标准输入读一行，原样打到标准输出：

```bash
bin/logstash -e 'input { stdin {} } output { stdout {} }'
```

看到管道启动完成的日志后，在同一终端输入一行文本并回车，输出区会出现带时间戳的事件。这个过程慢，是 JVM 和管道编译的开销，不是输入插件卡住。

## 测试

输入 `ping-from-console` 后，stdout 里的 `message` 字段应当就是这一行。若没有任何事件，多半是管道还没打印出 “Pipeline started”，输入被缓冲在启动阶段。

## 配置详解

一份管道配置分成三段：

```ruby
input {
  stdin {}
}
filter {
}
output {
  stdout {}
}
```

`input` 声明事件从哪来，`filter` 做变换，`output` 声明写到哪。三段都可以并列多个插件。

### 输入

同一条管道可以同时打开多个输入，例如一边接收 Beats，一边读补录文件。事件进入管道后都变成字段集合，后面的过滤器和输出不再关心它原来是文件还是网络。

### 过滤

过滤器逐事件执行。常见用途是用分隔符或 Grok 把一行日志拆成字段、改类型、删掉无用字段、按条件打标签。结构稳定之后，Kibana 才能按字段聚合，而不是对整段 `message` 做全文搜索。

### 输出

输出插件决定落点。调试期用 `stdout { codec => rubydebug }` 把事件漂亮地打印出来；确认字段无误后，再改成 `elasticsearch`。两者也可以同时保留，但正式流量下把 rubydebug 开在生产管道里会拖慢输出。

## 读取自定义日志

Filebeat 的模块能处理常见的访问日志。业务自己打印的行，模块不认识，就需要 Logstash 按格式拆开。

### 日志结构

约定一行四段，用竖线分隔：

```text
2024-04-02 10:15:08|WARN|库存不足|sku=KB-19
```

段序固定为时间、级别、原因、参数。解析时按这个顺序取下标，顺序一变，下游字段就会错位。

### 编写配置文件

新建 `shelf-pipeline.conf`：

```ruby
input {
  file {
    path => "/var/log/shelf/app.log"
    start_position => "beginning"
    sincedb_path => "/var/lib/logstash/shelf.sincedb"
  }
}
filter {
  mutate {
    split => { "message" => "|" }
  }
}
output {
  stdout { codec => rubydebug }
}
```

`start_position => "beginning"` 只在 sincedb 里还没有这个文件的记录时生效。重复启动却看不到旧行，通常是 sincedb 已经把偏移记到了文件末尾。

```bash
./bin/logstash -f ./shelf-pipeline.conf
echo "2024-04-02 10:15:08|WARN|库存不足|sku=KB-19" >> /var/log/shelf/app.log
```

stdout 里的 `message` 会从字符串变成数组，四段各自占一个元素。`split` 不会自动给每段起名字，命名在下一步做。

### 输出到 Elasticsearch

把输出换成集群地址。过滤仍先拆开，便于在 Kibana 里确认数组内容，之后再加 `add_field` 把下标映射成业务字段。

```ruby
input {
  file {
    path => "/var/log/shelf/app.log"
    start_position => "beginning"
    sincedb_path => "/var/lib/logstash/shelf.sincedb"
  }
}
filter {
  mutate {
    split => { "message" => "|" }
  }
}
output {
  elasticsearch {
    hosts => ["http://127.0.0.1:9200"]
    index => "shelf-log-%{+YYYY.MM.dd}"
  }
}
```

重启管道后再追加两行。索引 `shelf-log-日期` 里应出现新文档。若 Elasticsearch 拒绝写入，先看 Logstash 日志里的映射冲突：同一字段先被写成文本、后又写成数字，就会被拒。调试期可以让索引名带上实验后缀，避免污染已经有映射的正式索引。
