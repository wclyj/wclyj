# Elasticsearch 入门与 Go 客户端

## 这一节在整条链路里的位置

日志从消息队列取出之后，要落到 Elasticsearch，再用 Kibana 做检索和图表。旁边还有两条监控支路：用采集库读取主机信息写入 InfluxDB，交给 Grafana；或者用 Prometheus 拉取性能指标，同样用 Grafana 展示。本节只展开 Elasticsearch 本身。

Elasticsearch 建立在 Lucene 之上，对外提供分布式的全文检索，接口风格接近 REST。它同时可以当成文档库：每个字段默认可被索引、被搜索。集群能横向加到上百个节点，承载 PB 级数据，并在较短时间内完成写入和查询，因此常被放在搜索条件复杂的系统中心。

Kibana 不存文档，只是把集群里的数据画成可交互的页面。

## 安装并确认进程

从发行渠道下载与操作系统匹配的压缩包，解压后在 `bin` 目录启动。单机学习时默认监听 `9200`。用浏览器或命令行访问本机该端口的根路径，能看到节点名、版本号和集群标识，就说明进程已经就绪。

## 用命令行调用 API

下面的例子都走 HTTP，本机地址写作 `127.0.0.1:9200`。请求体是 JSON 时要带上 `Content-Type: application/json`。源资料里常见的 `ContentType` 拼写不会被服务端当成内容类型。

### 看健康

```bash
curl -X GET "127.0.0.1:9200/_cat/health?v"
```

输出是一张表。`status` 为 `green` 表示分片都已分配；`yellow` 常见于单节点：副本没有第二台机器可放，主分片仍可用；`red` 表示有主分片丢失。

```text
epoch      timestamp cluster       status node.total node.data shards pri relo init unassign pending_tasks max_task_wait_time active_shards_percent
1590000001 08:00:01  lab-cluster   yellow          1         1      3   3    0    0        1             0                  -                 75.0%
```

### 列出索引

```bash
curl -X GET "127.0.0.1:9200/_cat/indices?v"
```

```text
health status index        uuid                   pri rep docs.count docs.deleted store.size pri.store.size
green  open   .kibana_1    PLvyZV1bRDWex05xkOrNNg   1   0          4            1     23.9kb         23.9kb
yellow open   parcel       o42mIpDeSgSWZ6eARWUfKw   1   1          0            0       283b           283b
```

以点开头的索引是 Kibana 自己的内部数据。`pri` 是主分片数，`rep` 是副本数。

### 创建索引

```bash
curl -X PUT "127.0.0.1:9200/parcel"
```

成功时返回：

```json
{"acknowledged":true,"shards_acknowledged":true,"index":"parcel"}
```

### 删除索引

```bash
curl -X DELETE "127.0.0.1:9200/parcel"
```

```json
{"acknowledged":true}
```

删除不可恢复，生产环境不要对带别名的业务索引直接执行。

### 写入文档

不指定标识时用 POST，由集群生成标识：

```bash
curl -H "Content-Type: application/json" -X POST "127.0.0.1:9200/parcel/_doc" -d "{\"code\":\"HZ-19\",\"weight\":12,\"fragile\":true}"
```

返回里 `result` 为 `created`，并带上 `_id`、`_version`。早期路径把类型名放在索引和标识之间；7.x 起统一用 `_doc`，自定义类型已被弃用。

想自己决定标识时改用 PUT，标识写在路径末尾：

```bash
curl -H "Content-Type: application/json" -X PUT "127.0.0.1:9200/parcel/_doc/box-4" -d "{\"code\":\"SZ-2\",\"weight\":3,\"fragile\":false}"
```

同一路径再 PUT 一次就是覆盖，版本号递增。

### 检索

查询条件放在 JSON 里，用 GET 或 POST 打到 `_search`，不要打到某条文档的地址上。

取出该索引下的文档：

```bash
curl -X GET "127.0.0.1:9200/parcel/_search"
```

按字段匹配：

```bash
curl -H "Content-Type: application/json" -X POST "127.0.0.1:9200/parcel/_search" -d "{\"query\":{\"match\":{\"code\":\"SZ\"}}}"
```

默认最多返回 10 条。`size` 与 `query` 同级，用来改变条数：

```bash
curl -H "Content-Type: application/json" -X POST "127.0.0.1:9200/parcel/_search" -d "{\"size\":2,\"query\":{\"match\":{\"code\":\"SZ\"}}}"
```

把 `size` 嵌进 `query` 对象里不会生效，服务端会忽略这个无法识别的字段或直接报错。

## 用 Go 写入

客户端主版本要和集群一致。集群是 7.x 时，导入带主版本后缀的客户端包。

```bash
go mod init lab-es
go get github.com/olivere/elastic/v7
```

`go.mod` 中会出现类似：

```text
require github.com/olivere/elastic/v7 v7.0.32
```

下面把一条包裹记录写入 `parcel` 索引。结构体字段的 json 标签决定文档里的键名。

```go
package main

import (
	"context"
	"fmt"

	"github.com/olivere/elastic/v7"
)

type Parcel struct {
	Code    string `json:"code"`
	Weight  int    `json:"weight"`
	Fragile bool   `json:"fragile"`
}

func main() {
	cli, err := elastic.NewClient(elastic.SetURL("http://127.0.0.1:9200"))
	if err != nil {
		panic(err)
	}
	fmt.Println("elasticsearch reachable")

	item := Parcel{Code: "CD-7", Weight: 6, Fragile: true}
	ack, err := cli.Index().
		Index("parcel").
		BodyJson(item).
		Do(context.Background())
	if err != nil {
		panic(err)
	}
	fmt.Printf("stored id=%s index=%s\n", ack.Id, ack.Index)
}
```

`NewClient` 默认会嗅探集群里的其它节点。只有一个地址、又不希望它改连别的节点时，加上 `elastic.SetSniff(false)`。写入使用的上下文若带超时，超时后调用会返回错误，文档是否已经落盘要以返回值为准，不要凭“没有 panic”判断。

更多查询构造可以查阅该客户端的文档，检索、更新和批量写入都有对应的链式方法。
