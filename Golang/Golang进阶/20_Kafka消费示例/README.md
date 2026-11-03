# 用 Go 收发 Kafka 消息

Kafka 是分布式的发布订阅系统，能扛住站点行为流这种写入量，并提供持久化、副本和横向扩展。下面用 Go 把一条日志送进去，再按分区取出来。

## 先把进程拉起来

安装与目录布局见日志采集那一节。单机试验时先启动协调服务，再启动 Kafka 本身。Windows 的脚本名如下，类 Unix 系统换成同名的 shell 脚本：

```bash
.\zookeeper-server-start.bat ..\..\config\zookeeper.properties
.\kafka-server-start.bat ..\..\config\server.properties
```

默认客户端端口是 `9092`。

## 客户端库

Go 侧使用 sarama。模块路径历史上是 `github.com/Shopify/sarama`，后来迁到 `github.com/IBM/sarama`，包名仍是 `sarama`。

```bash
go get github.com/IBM/sarama
```

1.20 起默认启用 zstd，Windows 上若缺少 C 编译器，可暂时锁在 1.19。能装编译器时直接用当前版本。

## 同步生产者

同步生产者会等到服务端按配置确认后才返回。下面要求首领和所有同步副本都落盘，分区用随机选择，并打开成功回执，这样 `SendMessage` 才能带回分区号和偏移。

```go
package main

import (
	"fmt"

	"github.com/IBM/sarama"
)

func main() {
	cfg := sarama.NewConfig()
	cfg.Producer.RequiredAcks = sarama.WaitForAll
	cfg.Producer.Partitioner = sarama.NewRandomPartitioner
	cfg.Producer.Return.Successes = true

	record := &sarama.ProducerMessage{
		Topic: "app_event",
		Value: sarama.StringEncoder("checkout latency spike"),
	}

	writer, err := sarama.NewSyncProducer([]string{"127.0.0.1:9092"}, cfg)
	if err != nil {
		fmt.Println("open producer:", err)
		return
	}
	defer writer.Close()

	part, off, err := writer.SendMessage(record)
	if err != nil {
		fmt.Println("publish:", err)
		return
	}
	fmt.Printf("partition=%d offset=%d\n", part, off)
}
```

`WaitForAll` 最慢也最不容易丢。只关心首领是否收到时，可以改成等待首领确认，吞吐更高，但首领宕机且副本还没跟上时可能丢消息。

## 按分区消费

消费者先问主题有哪些分区，再为每个分区单独开一个分区消费者。偏移用 `OffsetNewest` 表示只取启动之后的新消息；要补历史就改用 `OffsetOldest`。

```go
package main

import (
	"fmt"

	"github.com/IBM/sarama"
)

func main() {
	reader, err := sarama.NewConsumer([]string{"127.0.0.1:9092"}, nil)
	if err != nil {
		fmt.Printf("open consumer: %v\n", err)
		return
	}
	defer reader.Close()

	parts, err := reader.Partitions("app_event")
	if err != nil {
		fmt.Printf("list partitions: %v\n", err)
		return
	}
	fmt.Println("partitions", parts)

	for _, id := range parts {
		pc, err := reader.ConsumePartition("app_event", id, sarama.OffsetNewest)
		if err != nil {
			fmt.Printf("partition %d: %v\n", id, err)
			return
		}
		defer pc.AsyncClose()

		go func(src sarama.PartitionConsumer) {
			for msg := range src.Messages() {
				fmt.Printf("part=%d off=%d key=%s body=%s\n",
					msg.Partition, msg.Offset, msg.Key, msg.Value)
			}
		}(pc)
	}

	select {}
}
```

遍历分区切片时要用元素本身当作分区号。若写成 `for id := range parts`，得到的是下标，分区号和切片下标不一致时就会订到错误的分区。循环变量要作为参数传进闭包，避免所有协程共享最后一次的消费者。`main` 末尾若直接返回，进程会退出，消费协程来不及读到消息，所以这里用空的 `select` 停住。

## 转到 LogTransfer

采集端把日志推进 Kafka 之后，需要另一个进程把消息取出再写入 Elasticsearch。这个进程可以按职责拆成几块：

- `kafka`：订阅主题、读出消息
- `es`：把文本转成文档并写入
- `conf`：集中存放两边的地址和主题名

### 配置文件

用 ini 分区存放，避免把地址写死在代码里：

```ini
[kafka]
address=127.0.0.1:9092
topic=app_event

[es]
address=127.0.0.1:9200
```

### 对应的结构体

标签要和方括号里的区名、等号左边的键名一致，加载库才能填进字段。

```go
package conf

type TransferConf struct {
	Kafka KafkaConf `ini:"kafka"`
	ES    ESConf    `ini:"es"`
}

type KafkaConf struct {
	Address string `ini:"address"`
	Topic   string `ini:"topic"`
}

type ESConf struct {
	Address string `ini:"address"`
}
```

`MapTo` 之后，`Address` 和 `Topic` 就可以传给生产者、消费者和搜索客户端，换环境时只改配置文件。
