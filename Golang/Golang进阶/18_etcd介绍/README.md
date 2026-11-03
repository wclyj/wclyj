# etcd：配置、发现与分布式锁

日志采集程序若把要跟踪的文件路径写死在本地，每加一项都得改配置再发布。把这些路径放到 etcd 里之后，采集端按本机地址拉取，并监听后续改动，就能在不重启的情况下增减采集项。Docker 和 Kubernetes 也把 etcd 当作集群状态的存储。

## 它是什么

etcd 用 Go 写成，是高可用的分布式键值存储，常用于配置共享、服务注册与发现。同类组件还有 ZooKeeper 和 Consul。

特点可以概括成：

- 每个节点都持有完整数据副本，读可以打到任意成员。
- 硬件或网络出问题时，剩余成员能再选出首领，避免单点。
- 一次读能看到跨多数派已经确认的最新写入。
- 对外是定义清楚的 API，传输用 gRPC。
- 可以启用 TLS，并校验客户端证书。
- 官方基准大约能到每秒一万次写入，具体数字取决于硬件和数据大小。
- 一致性来自 Raft：通过选举、日志复制处理成员故障。它和 ZooKeeper 使用的 ZAB 都追求多数派确认，但术语和恢复细节不同。脑裂时两边都不能独自提交，因为都凑不齐多数。

## 三种用法

### 服务发现

分布式系统里的进程需要找到对方正在监听的地址。服务发现要回答的就是：某个名字现在对应哪一个 TCP 或 UDP 端点。提供者启动时把自己的地址写到约定的前缀下，调用方按名字读取；提供者退出时删除或让租约过期，调用方的列表随之变短。

### 配置中心

启动时先读一次配置，同时对那个键注册 Watch。之后有人更新键，etcd 会把事件推给订阅者，进程就能换成新配置，不必重启。日志采集项的增删就是这种模式。

### 分布式锁

Raft 保证写入一旦被多数派确认，所有成员看到的是同一份值，因此可以在上面做锁。

- 独占：多个客户端同时创建同一个键，只有一个能创建成功。实现上是带“前值条件”的写入，也就是比较再交换。创建成功的一方视为拿到锁，用完删除键。
- 排队：所有等待者都要按全局顺序执行。对一个前缀发起“创建有序键”，etcd 会分配递增的名字。名字最小且仍存在的那个客户端获得锁，做完就删除自己的键，后面的人依次上前。

三个客户端同时抢同一把锁时，只有创建成功或序号最小的那个往下走，另外两个看着这个键，等它消失再重试。

## 和 ZooKeeper 怎么选

ZooKeeper 也能做发现、配置和锁。etcd 被单独拿出来，常常是因为运维和接口上的差别。

ZooKeeper 的不足：

- 部署和运维要熟悉它自己的一套概念，Paxos 这一类强一致算法本身也不好讲。官方客户端长期以 Java 和 C 为主。
- 它用 Java 实现，运行时依赖重。运维更希望负责一致性的那组机器尽量简单。
- 作为 Apache 基金会项目，发布节奏受基金会流程影响，演进相对慢。

etcd 这边：

- 单一二进制，用 Go 编译，接口直接是 HTTP 或 gRPC，Raft 的多数派模型相对好解释。
- 默认每次修改都落盘，而不是只留在内存。
- 支持 SSL 客户端证书。

它仍然年轻：迭代快意味着行为还在变，也不如老项目经过那么多年的生产锤炼。CoreOS、Kubernetes、Cloud Foundry 已经在生产里使用它，所以作为配置和发现的存储是站得住的。真正决定选型的是团队是否愿意维护 Raft 集群，而不是口号。

## 进程内的四块

- HTTP Server：处理用户 API，也处理其它成员发来的同步和心跳。
- Store：实现键值索引、节点状态、监听和事件，大多数对外功能在这里落地。
- Raft：一致性协议的实现，是能否提交写入的决策核心。
- WAL：预写日志。内存里虽有全部状态和索引，真正的持久化靠 WAL。数据在提交前先追加日志。快照用来避免日志无限增长；日志条目才是一次次写入的内容。

### 集群规模

选举需要多数票，所以成员数取奇数，常见是 3、5 或 7。两个节点时任何一个掉线都不再拥有多数，集群会不可用。

## 三个节点怎么启动

每个进程都要知道全部成员，并用同一个 token 标明“这是一伙的”，避免误加入别的集群。下面用 n1、n2、n3 表示三台机器，客户端端口 2379，对等端口 2380。

在 n1 上：

```bash
etcd --name n1 \
  --data-dir /var/lib/etcd/n1 \
  --initial-advertise-peer-urls http://10.0.0.1:2380 \
  --listen-peer-urls http://10.0.0.1:2380 \
  --listen-client-urls http://10.0.0.1:2379,http://127.0.0.1:2379 \
  --advertise-client-urls http://10.0.0.1:2379 \
  --initial-cluster-token lab-token \
  --initial-cluster n1=http://10.0.0.1:2380,n2=http://10.0.0.2:2380,n3=http://10.0.0.3:2380 \
  --initial-cluster-state new
```

n2、n3 只改 `--name`、数据目录和本机地址，`--initial-cluster` 三边保持一致。已有集群再加入新成员时，状态要改成 `existing`，并先通过成员 API 登记，不能三台都写 `new`。

官方还提供过公网的发现服务，用来代替手写 `--initial-cluster`。现在更常见的做法是自己列出成员，避免依赖外部地址。

## 单机试用

### 下载

从项目的发行页选取对应系统的压缩包。Windows 解压后能看到 `etcd.exe` 和 `etcdctl.exe`；其它系统的可执行文件通常在解压目录或 `bin` 下。

### 启动

直接运行 `etcd`。默认在 `2379` 接受客户端，在 `2380` 接受同伴通信。单机时它自己就是首领。`etcdctl` 是命令行客户端。

### 指定 API 版本

旧的 etcdctl 默认走 v2 接口。要使用现在的键值 API，把环境变量设为 3：

```bash
# Windows
set ETCDCTL_API=3
# 类 Unix
export ETCDCTL_API=3
```

变量名是 `ETCDCTL_API`，不要写成 `SET_ETCDCTL_API`。设错时，同样的子命令会打到另一套接口上，表现成找不到键或语法不对。

### 写入、读取、删除

```bash
etcdctl --endpoints=http://127.0.0.1:2379 put /lab/region "east"
etcdctl --endpoints=http://127.0.0.1:2379 get /lab/region
etcdctl --endpoints=http://127.0.0.1:2379 del /lab/region
```

`put` 成功会打印确认。`get` 默认先输出键再输出值。`del` 输出删除的个数。

## 在 Go 里使用

当前模块路径是 `go.etcd.io/etcd/client/v3`。较老的教程写 `go.etcd.io/etcd/clientv3`，导入路径要和 `go.mod` 里的版本一致。

```bash
go get go.etcd.io/etcd/client/v3
```

### 写入和读取

```go
package main

import (
	"context"
	"fmt"
	"time"

	clientv3 "go.etcd.io/etcd/client/v3"
)

func main() {
	cli, err := clientv3.New(clientv3.Config{
		Endpoints:   []string{"127.0.0.1:2379"},
		DialTimeout: 5 * time.Second,
	})
	if err != nil {
		fmt.Printf("dial: %v\n", err)
		return
	}
	defer cli.Close()
	fmt.Println("etcd connected")

	ctx, cancel := context.WithTimeout(context.Background(), time.Second)
	_, err = cli.Put(ctx, "/lab/region", "east")
	cancel()
	if err != nil {
		fmt.Printf("put: %v\n", err)
		return
	}

	ctx, cancel = context.WithTimeout(context.Background(), time.Second)
	resp, err := cli.Get(ctx, "/lab/region")
	cancel()
	if err != nil {
		fmt.Printf("get: %v\n", err)
		return
	}
	for _, kv := range resp.Kvs {
		fmt.Printf("%s=%s\n", kv.Key, kv.Value)
	}
}
```

`Get` 的键要和 `Put` 的键相同。超时上下文用完要 `cancel`，避免定时器泄漏到截止时间才释放。

### 依赖冲突

若编译期出现与 `resolver.BuildOption` 相关的未定义错误，多半是 `google.golang.org/grpc` 的版本比这版客户端预期的新。可以在 `go.mod` 里把 grpc 钉到客户端所兼容的版本，例如：

```text
replace google.golang.org/grpc => google.golang.org/grpc v1.26.0
```

新版本的 etcd 客户端通常已经跟上较新的 grpc，优先升级客户端，而不是长期钉死旧的 grpc。`replace` 只作为编译不过时的退路。

### Watch

Watch 适合做热更新。它返回一个只读通道，键的新增、修改和删除都会变成事件。

```go
func main() {
	cli, err := clientv3.New(clientv3.Config{
		Endpoints:   []string{"127.0.0.1:2379"},
		DialTimeout: 5 * time.Second,
	})
	if err != nil {
		fmt.Printf("dial: %v\n", err)
		return
	}
	defer cli.Close()

	watchCtx, watchCancel := context.WithCancel(context.Background())
	defer watchCancel()
	ch := cli.Watch(watchCtx, "/lab/region")
	for pack := range ch {
		for _, ev := range pack.Events {
			fmt.Printf("type=%s key=%s val=%s\n", ev.Type, ev.Kv.Key, ev.Kv.Value)
		}
	}
}
```

另开一个终端执行 `put` 或 `del`，这个循环就会打印事件。`Watch` 从注册之后的修订版本开始推送，不会自动把当前值再发一遍，所以启动时仍要先 `Get` 一次。取消 `watchCtx` 会结束通道，循环随之退出。

服务端把通知推到客户端，走的是 gRPC 流，不是浏览器里的 WebSocket。客户端看起来像在读一个永远不关闭的 Go 通道。

## 用在日志采集上

采集进程启动时，用本机 IP 作为键的前缀去 `Get`，例如 `/logagent/10.0.0.8/`，前缀查询打开后能一次拿到这台机器的全部文件配置。每一条配置对应一个跟踪任务，负责读文件并把新行送进 Kafka。

随后 `Watch` 这个前缀。新增键就再启动一个跟踪任务，删除键就停掉对应任务，修改键就先停再按新路径启动。这样增加一个日志文件只需要写 etcd，不必登录机器改本地文件。

任务的创建和退出要带互斥，避免 Watch 事件和启动扫描同时改同一张任务表。etcd 暂时不可达时，保留上一份成功的配置继续采集，比直接退出更符合“采集端尽量不中断”的目标。
