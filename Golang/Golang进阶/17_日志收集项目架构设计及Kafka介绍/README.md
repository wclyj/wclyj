# 日志采集架构与 Kafka

## 背景

每套业务都会写日志。机器少的时候，登录上去打开文件就能查。机器变成十几台、系统又是分布式部署时，逐台登录既慢又容易漏掉真正出错的那一台。

## 做法

把各台机器上的日志近实时收拢到一处，建好索引，用搜索代替逐文件翻找，并用一个网页把结果展示出来。

## 约束

一天的日志量可以到几十亿条，端到端延迟要控制在分钟级，架构必须能靠加机器扩展，而不是靠换一台更大的服务器。

## 业界常见的 ELK

一条经典链路是：业务机器上的 Logstash 代理读文件，送进 Elasticsearch 集群，Kibana 负责查询页面，浏览器作为入口。

这套组合能用，但有三个缺口：

- 每增加一种日志路径，都要改代理配置并重新下发，运维成本随采集项线性增长。采集项可以改由 etcd 集中管理。
- 代理本身的运行状态缺少统一观测，出了问题不知道是没读到文件还是没送出去。
- 代理是通用组件，业务上的过滤、脱敏和多租户不容易按自己的方式改。

## 自己的链路

配置中心用 etcd。它用 Go 实现，在“存键值、选主、监听变化”这些事上可以替代 ZooKeeper。

LogAgent 部署在业务机器上，读本地文件，把新行送进 Kafka。Kafka 既是缓冲队列，也把消息留在磁盘上。LogTransfer 再把 Kafka 里的记录取出、整理，写入 Elasticsearch。Kibana 对索引做检索和图表。主机指标另由 SysAgent 采集，也可以改用 Prometheus。

各组件的职责：

- LogAgent：采集端，只关心本机文件和投递。
- Kafka：高吞吐的分布式日志，最初由 LinkedIn 开发，现为 Apache 顶级项目。
- Elasticsearch：搜索引擎，提供 HTTP 接口。
- Kibana：对 Elasticsearch 的分析和展示。
- Hadoop：离线的分布式计算，适合对已经落地的大批数据做批处理。
- Storm：面向流的实时计算。链路里若要在入库前做聚合，可以插在队列之后。

做完这条链路，会实际碰到采集进程、消费进程、Kafka 与 ZooKeeper、搜索与可视化，以及用 etcd 做配置共享。

## 队列的两种模型

### 点对点

生产者把消息放进一条队列，消费者取走后队列里就没有了。一条消息只会被处理一次，不存在同一队列里的两个消费者重复拿到它。

### 发布订阅

生产者把消息发到主题，所有订阅者都能收到，像关注了同一栏目的人都会看到新文章。订阅者只有一个时，它会成为瓶颈。实践中把多个消费者编成一个组，组内分摊，组与组之间仍各拿一份。可以看成主题下面有多条队列：队列内部是点对点，队列之间是发布订阅。

## Kafka 是什么

Kafka 用 Scala 编写，2011 年开源，2012 年成为 Apache 顶级项目。设计之初是为了搬动社交网站内部的海量日志。它既可以单机运行，也可以多台组成集群，同时提供发布和订阅。特点是吞吐高、延迟低、副本能扛住节点故障。

### 架构里的名字

- Producer：生产者，数据的入口。
- 集群：一台或多台服务器。
- Broker：跑着 Kafka 进程的节点。集群内每个 broker 有不重复的编号。一台机器上可以跑多个进程，教学里先按“一个 broker 一台机器”理解。
- Topic：主题，消息的逻辑分类。一个 broker 上可以有多个主题，常见做法是一条业务线一个主题。
- Partition：主题的分区。分区用来并行，提高吞吐。同一主题的不同分区里数据不重复。在磁盘上，一个分区对应一个目录。
- Replication：分区的副本。首领负责读写，首领失效后从跟随者里再选一个。默认副本数上限是 10，且不能超过 broker 数量。同一分区的副本不会放在同一台机器上。
- Consumer：消费者，数据的出口。

### 一次写入怎么走完

生产者不直接写跟随者。顺序是：

1. 向集群查询目标分区的首领在哪一台。
2. 把消息发给首领。
3. 首领写入本地磁盘。
4. 跟随者从首领拉取。
5. 跟随者写完自己的磁盘后，向首领确认。
6. 首领按配置收齐确认，再应答生产者。

### 选哪个分区

主题有多个分区时，生产者按下面的顺序决定：

- 发送时明确指定了分区，就进那个分区。
- 没指定分区但带了键，就对键做哈希，映射到固定分区。相同的键会进同一分区，从而在该分区内保持顺序。
- 分区和键都没有，就按时间片轮询，一小段时间写一个分区，下一段换下一个。

### 确认级别

生产者可以要求集群确认到什么程度，取值是 0、1 或 all。

- 0：发出去就不管，不等待任何应答。吞吐最高，丢失风险最大。
- 1：首领写入成功即可。首领在副本跟上之前宕机，这一条可能丢。
- all：要等所有同步副本都确认。最安全，也最慢。

往一个还不存在的主题写入时，Kafka 会按默认配置自动创建，分区数和副本数默认都是 1。生产环境应提前创建，避免默认值不符合容量规划。

### 主题与日志

主题是一类记录的集合，通常有多个订阅者。每个分区是一份有序、只追加的日志。新记录加在末尾，并分配一个分区内唯一的偏移量。Kafka 只保证同一分区内部有序，跨分区没有全局顺序。

可以配置保留时间。在这个时间之内，消息即使已被消费也不会删；超过之后清理，给新数据腾位置。例如保留两天，则两天内的记录都能按偏移再读一遍。数据写在磁盘上，保留窗口可以设得比较大，不必受内存大小限制。

### 分区目录里有什么

每个分区是一个目录，下面有多组 segment。一组里通常包含 `.log`、`.index` 和 `.timeindex`。`.log` 存放消息本身，另外两个是偏移和时间的索引，用来加快查找，避免从文件头扫描。

### 消费者组

同一业务的读取进程可以挂上相同的组标识。这些进程允许分散在不同的进程里，也可以分散在不同的机器上。

组内的实例共同分摊分区：一条记录只会被组里的一个实例读到。若每个实例各自使用不同的组，则每条记录会被每个组都读到，也就是广播。

举例：两个 broker，一个主题四个分区。组 A 有两个实例，组 B 有四个。组 A 里每个实例大约分到两个分区；组 B 里每个实例一个分区。组内一个分区最多分给一个实例，否则偏移量无法唯一记录“这个组读到了哪里”。分区只有四个时，组里第五个实例会闲着。实例上下线后，集群会重新分配分区。

### 能拿来做什么

- 消息队列：跨进程传递，上游只依赖队列，不依赖下游是否在线。典型用途是削峰和串联任务。同领域还有 ActiveMQ、RabbitMQ 等更传统的队列。
- 记录站点行为：浏览、搜索等事件进不同主题，供实时计算或导入仓库做离线报表。
- 传输监控指标：把分散进程的统计集中后再汇总展示。
- 聚合日志：相对 Flume、Scribe 这类专门的采集工具，Kafka 更强调吞吐，适合作为中间缓冲，而不是直接替代文件采集器。

## 安装与启动

发行包需要先有 JDK。解压后先改配置，再启动。Kafka 默认监听 `9092`。

日志目录在 `config/server.properties` 里，改成一块空间足够的磁盘：

```text
log.dirs=/var/lib/kafka-logs
```

Windows 在 `bin\windows` 下执行：

```bash
.\kafka-server-start.bat ..\..\config\server.properties
```

类 Unix 系统使用同名的 shell 脚本。看到监听端口打开，才算启动完成。

## ZooKeeper

这一版 Kafka 的发行包里带有配套的 ZooKeeper 配置，先改数据目录再启动。找到 `config/zookeeper.properties`，把数据目录指到独立路径，避免和 Kafka 日志混在一起。

```bash
.\zookeeper-server-start.bat ..\..\config\zookeeper.properties
```

它默认占用 `2181`。必须先启动它，再启动 Kafka，否则 Kafka 连不上协调服务会退出。较新的 Kafka 可以改用自带的共识控制器，不再依赖 ZooKeeper，那是另一套配置，本节的脚本仍按带 ZooKeeper 的发行包来写。

## LogAgent 怎么工作

进程做两件事：跟上本地文件的新增内容，把每一行交给 Kafka。读文件使用 `github.com/hpcloud/tail`，它能从末尾接着读，并在文件被滚动重建后重新打开。

```bash
go get github.com/hpcloud/tail
```

单独验证跟踪：

```go
package main

import (
	"fmt"
	"time"

	"github.com/hpcloud/tail"
)

func main() {
	cfg := tail.Config{
		ReOpen:    true,
		Follow:    true,
		Location:  &tail.SeekInfo{Offset: 0, Whence: 2},
		MustExist: false,
		Poll:      true,
	}
	follower, err := tail.TailFile("./app.log", cfg)
	if err != nil {
		fmt.Println("tail:", err)
		return
	}
	for {
		select {
		case line, ok := <-follower.Lines:
			if !ok {
				fmt.Println("reopen", follower.Filename)
				time.Sleep(time.Second)
				continue
			}
			fmt.Println("line", line.Text)
		case <-time.After(2 * time.Second):
		}
	}
}
```

`Whence: 2` 表示从文件末尾开始，只收集启动之后的新行。`ReOpen` 配合日志滚动：文件被改名、原路径上出现新文件时重新打开。`MustExist: false` 允许文件稍后再被创建。

若编译时提示找不到 `gopkg.in/fsnotify.v1`，是因为该库后来改了路径。打开报错所在的 `inotify.go` 与 `inotify_tracker.go`，把旧导入换成当前仍在维护的 fsnotify 路径，或直接改用仍引用旧路径的 tail 版本。不要只删掉导入，后面的类型名也要跟着变。

### 发送依赖

```bash
go get github.com/Shopify/sarama
```

sarama 在 1.20 之后默认启用 zstd，Windows 上没有 C 编译器时会报找不到 `gcc`。教学环境可以锁在 1.19：

```bash
go mod init example.com/lab/logagent
```

```text
go 1.21

require github.com/Shopify/sarama v1.19.0
```

然后执行 `go mod download`。能安装 C 工具链时，不必刻意退回旧版。

连通性试验：

```go
func main() {
	cfg := sarama.NewConfig()
	cfg.Producer.RequiredAcks = sarama.WaitForAll
	cfg.Producer.Partitioner = sarama.NewRandomPartitioner
	cfg.Producer.Return.Successes = true

	writer, err := sarama.NewSyncProducer([]string{"127.0.0.1:9092"}, cfg)
	if err != nil {
		fmt.Println("producer:", err)
		return
	}
	defer writer.Close()

	part, off, err := writer.SendMessage(&sarama.ProducerMessage{
		Topic: "app_event",
		Value: sarama.StringEncoder("disk latency high"),
	})
	if err != nil {
		fmt.Println("send:", err)
		return
	}
	fmt.Printf("partition=%d offset=%d\n", part, off)
}
```

`WaitForAll` 对应前面的确认级别 all。`Return.Successes` 不打开时，同步发送拿不到成功结果。

## 拆成三个文件

### kafka 包

只负责初始化和发送，主题由调用方传入，避免在发送函数里再写死。

```go
package kafka

import (
	"fmt"

	"github.com/Shopify/sarama"
)

var writer sarama.SyncProducer

func Init(brokers []string) error {
	cfg := sarama.NewConfig()
	cfg.Producer.RequiredAcks = sarama.WaitForAll
	cfg.Producer.Partitioner = sarama.NewRandomPartitioner
	cfg.Producer.Return.Successes = true
	var err error
	writer, err = sarama.NewSyncProducer(brokers, cfg)
	if err != nil {
		return err
	}
	fmt.Println("kafka ready")
	return nil
}

func Send(topic, body string) {
	part, off, err := writer.SendMessage(&sarama.ProducerMessage{
		Topic: topic,
		Value: sarama.StringEncoder(body),
	})
	if err != nil {
		fmt.Println("send:", err)
		return
	}
	fmt.Printf("sent partition=%d offset=%d body=%s\n", part, off, body)
}
```

### 跟踪文件的包

```go
package follow

import (
	"fmt"

	"github.com/hpcloud/tail"
)

var follower *tail.Tail

func Init(path string) error {
	cfg := tail.Config{
		ReOpen:    true,
		Follow:    true,
		Location:  &tail.SeekInfo{Offset: 0, Whence: 2},
		MustExist: false,
		Poll:      true,
	}
	var err error
	follower, err = tail.TailFile(path, cfg)
	if err != nil {
		fmt.Println("tail:", err)
	}
	return err
}

func Lines() <-chan *tail.Line {
	return follower.Lines
}
```

返回只读通道，调用方不能误关闭它。

### 入口

```go
func run(topic string) {
	for line := range follow.Lines() {
		if line == nil {
			continue
		}
		kafka.Send(topic, line.Text)
	}
}

func main() {
	if err := kafka.Init([]string{"127.0.0.1:9092"}); err != nil {
		fmt.Println("kafka:", err)
		return
	}
	if err := follow.Init("./app.log"); err != nil {
		fmt.Println("follow:", err)
		return
	}
	run("app_event")
}
```

用 `range` 读通道，文件被关掉时循环自然结束。不要在 `default` 里每隔一秒空转，那会在没有日志时占着 CPU，也会把“通道暂时没数据”和“跟踪失败”混在一起。

往 `app.log` 追加一行，生产者日志里应出现分区和偏移。再用控制台消费者核对：

```bash
.\kafka-console-consumer.bat --bootstrap-server=127.0.0.1:9092 --topic=app_event --from-beginning
```

`--from-beginning` 会把保留期内的历史也打出来。只看新消息时去掉它。

## 去掉硬编码

地址、主题和文件路径移到 ini。注意键名不要写成 `path:=` 这种带多余冒号的形式。

```ini
[kafka]
address=127.0.0.1:9092
topic=app_event

[follow]
filename=./app.log
```

解析使用 `gopkg.in/ini.v1`：

```bash
go get gopkg.in/ini.v1
```

可以按区和键读取：

```go
cfg, err := ini.Load("./conf/app.ini")
if err != nil {
	fmt.Println("ini:", err)
	os.Exit(1)
}
fmt.Println(cfg.Section("kafka").Key("address").String())
fmt.Println(cfg.Section("kafka").Key("topic").String())
fmt.Println(cfg.Section("follow").Key("filename").String())
```

这样入口里仍然充斥着字符串键名，字段一多就容易写错。更好的办法是定义结构体，让标签和 ini 对应。

```go
package conf

type App struct {
	Kafka  Kafka  `ini:"kafka"`
	Follow Follow `ini:"follow"`
}

type Kafka struct {
	Address string `ini:"address"`
	Topic   string `ini:"topic"`
}

type Follow struct {
	Filename string `ini:"filename"`
}
```

入口改成映射：

```go
func main() {
	var cfg conf.App
	if err := ini.MapTo(&cfg, "./conf/app.ini"); err != nil {
		fmt.Println("ini:", err)
		return
	}
	if err := kafka.Init([]string{cfg.Kafka.Address}); err != nil {
		fmt.Println("kafka:", err)
		return
	}
	if err := follow.Init(cfg.Follow.Filename); err != nil {
		fmt.Println("follow:", err)
		return
	}
	run(cfg.Kafka.Topic)
}
```

`MapTo` 需要指针。嵌入字段的标签必须等于方括号里的区名，键标签必须等于等号左边的名字，大小写按库的规则匹配。

## 偏移、重复与失败

采集端记下文件读到的位置，重启后才能接着上次继续。只从末尾跟随时，停机期间追加的行会被跳过。把位置写在本地磁盘，并在进程收到退出信号时刷出去。Kafka 侧的偏移是另一份账：消费者提交的是主题分区里的位置，不是文本文件的字节数。两份账不要混用。

确认级别选 all 时，首领要等同步副本写完才应答。采集端若把发送失败当成成功，这行日志就再也不会重试。失败应当把该行留在内存队列里，带上次数上限；超过上限再落本地死信文件，避免一条坏消息堵住后面的正常行。

下游按至少一次来设计。同一行可能因为超时重试出现两次，入库前用“机器加文件加行号”或业务主键做去重。不要假设分区之间有全局顺序，只在同一个键哈希到同一分区时，才能谈论先后。

## 这一版还缺什么

- 发送失败只打印，没有重试，进程退出前队列里的行会丢。
- 只跟踪一个文件。多个路径应做成切片，每个文件一个 goroutine。
- 偏移没有持久化，重启后从末尾读，停机期间的行会被跳过。若不能丢，就要记住读到的位置，或接受少量重复再在下游去重。
- 配置仍在本地文件。下一步是改成从 etcd 按 IP 拉取，并监听变更。变更事件和启动时的全量读取可能交错，更新任务表时要串行，避免同一个文件被跟踪两次。
