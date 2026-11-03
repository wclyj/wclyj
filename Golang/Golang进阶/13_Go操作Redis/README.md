# 用 Go 操作 Redis

Redis 常被用来做缓存、简易队列和排行榜。它是开源的内存数据库，把业务对象映射到字符串、哈希、列表、集合、有序集合等结构上。再配合复制、持久化和客户端分片，可以扩到数百 GB、每秒处理大量请求。

## 数据结构

除了字符串、哈希、列表、集合、有序集合，Redis 还提供位图、HyperLogLog，以及带半径查询的地理索引和流。选结构时先看操作：要范围排名就用有序集合，要左右弹出就用列表，只存一段文本就用字符串。

## 典型场景

- 缓存：把热点查询挡在 MySQL 前面，减轻主键库压力。
- 计数：关注数、播放量这类只增不减或偶尔回退的数字。
- 排行榜：有序集合按分数排序，取前几名是它的基本操作。
- 队列：列表可以从一侧推入、另一侧弹出。

## 和 Memcached 对比

Memcached 的值基本上是字符串。Redis 的结构更丰富，单机性能通常更好，并支持 RDB 快照和 AOF 日志两种持久化，也支持主从复制。只需要一个纯内存、可丢失的键值缓存时，两者都能用；一旦要排序、队列或落盘，就更适合 Redis。

## 准备一个本地实例

用容器起一个 5.0.7，名字和端口按自己的环境改：

```bash
docker run --name redis507 -p 6379:6379 -d redis:5.0.7
```

再开一个一次性的命令行客户端，走宿主机网络去连它：

```bash
docker run -it --network host --rm redis:5.0.7 redis-cli
```

## 选择客户端

除了 redigo，这里使用 `github.com/redis/go-redis/v9`。它能连单机、哨兵和集群。9 这个主版本的命令都要传入 `context.Context`，超时和取消因此能跟调用链走。

```bash
go get github.com/redis/go-redis/v9
```

## 建立连接

### 单机

```go
var rdb *redis.Client

func openStandalone() error {
	rdb = redis.NewClient(&redis.Options{
		Addr:     "127.0.0.1:6379",
		Password: "",
		DB:       0,
	})
	return rdb.Ping(context.Background()).Err()
}
```

`Ping` 失败多半是地址、密码或库号不对。客户端内部有连接池，应当做成进程级变量，不要每次命令都 `NewClient`。

### 哨兵

哨兵模式只给出主节点的逻辑名字和哨兵地址，真正的主库地址由哨兵告知，故障转移后客户端会跟着变。

```go
func openFailover() error {
	rdb = redis.NewFailoverClient(&redis.FailoverOptions{
		MasterName:    "app-master",
		SentinelAddrs: []string{"10.0.0.1:26379", "10.0.0.2:26379", "10.0.0.3:26379"},
	})
	return rdb.Ping(context.Background()).Err()
}
```

### 集群

集群客户端按槽位把键分到不同节点，地址列表里写上若干个入口即可。

```go
func openCluster() error {
	rdb = redis.NewClusterClient(&redis.ClusterOptions{
		Addrs: []string{":7000", ":7001", ":7002", ":7003", ":7004", ":7005"},
	})
	return rdb.Ping(context.Background()).Err()
}
```

涉及多个键的命令必须落在同一个槽，否则集群会拒绝。键名里用花括号可以指定参与哈希的那一段。

## 基本命令

### 字符串

```go
func stringDemo(ctx context.Context) {
	if err := rdb.Set(ctx, "points", 80, 0).Err(); err != nil {
		fmt.Printf("set points: %v\n", err)
		return
	}
	got, err := rdb.Get(ctx, "points").Result()
	if err != nil {
		fmt.Printf("get points: %v\n", err)
		return
	}
	fmt.Println("points", got)

	name, err := rdb.Get(ctx, "nickname").Result()
	switch {
	case err == redis.Nil:
		fmt.Println("nickname absent")
	case err != nil:
		fmt.Printf("get nickname: %v\n", err)
	default:
		fmt.Println("nickname", name)
	}
}
```

第三个参数是过期时间，`0` 表示不过期。键不存在时错误是 `redis.Nil`，要和网络错误分开处理。

### 有序集合

```go
func rankDemo(ctx context.Context) {
	key := "course_rank"
	rows := []redis.Z{
		{Score: 88, Member: "rust"},
		{Score: 96, Member: "kotlin"},
		{Score: 93, Member: "python"},
		{Score: 91, Member: "typescript"},
		{Score: 97, Member: "cpp"},
	}
	n, err := rdb.ZAdd(ctx, key, rows...).Result()
	if err != nil {
		fmt.Printf("zadd: %v\n", err)
		return
	}
	fmt.Printf("zadd wrote %d\n", n)

	latest, err := rdb.ZIncrBy(ctx, key, 6, "rust").Result()
	if err != nil {
		fmt.Printf("zincrby: %v\n", err)
		return
	}
	fmt.Printf("rust now %f\n", latest)

	top, err := rdb.ZRevRangeWithScores(ctx, key, 0, 2).Result()
	if err != nil {
		fmt.Printf("zrevrange: %v\n", err)
		return
	}
	for _, z := range top {
		fmt.Println("top", z.Member, z.Score)
	}

	band, err := rdb.ZRangeByScoreWithScores(ctx, key, &redis.ZRangeBy{
		Min: "93",
		Max: "100",
	}).Result()
	if err != nil {
		fmt.Printf("zrangebyscore: %v\n", err)
		return
	}
	for _, z := range band {
		fmt.Println("band", z.Member, z.Score)
	}
}
```

`ZAdd` 对已存在的成员是更新分数，所以重复执行时返回的新增个数可能是 0。`ZRevRange` 按下标从高到低取，`ZRangeByScore` 按分数闭区间取。

一次运行可能打印出：

```text
zadd wrote 5
rust now 94.000000
top cpp 97
top kotlin 96
top rust 94
band rust 94
band kotlin 96
band cpp 97
```

### Pipeline

Pipeline 把多条命令攒在客户端，一次写进套接字。它减少的是网络往返，不是事务：命令到了服务器仍可能和别的客户端交错。

```go
pipe := rdb.Pipeline()
incr := pipe.Incr(ctx, "visit_counter")
pipe.Expire(ctx, "visit_counter", time.Hour)
_, err := pipe.Exec(ctx)
fmt.Println(incr.Val(), err)
```

这相当于一次送出：

```text
INCR visit_counter
EXPIRE visit_counter 3600
```

也可以把命令写在回调里：

```go
var incr *redis.IntCmd
_, err := rdb.Pipelined(ctx, func(pipe redis.Pipeliner) error {
	incr = pipe.Incr(ctx, "visit_batched")
	pipe.Expire(ctx, "visit_batched", time.Hour)
	return nil
})
fmt.Println(incr.Val(), err)
```

命令之间没有依赖、只是想省往返时再用。某一条失败不会自动撤销已经成功的那几条。

### 事务

Redis 执行单条命令时是原子的，但两条来自不同客户端的命令会交错。`MULTI` 到 `EXEC` 之间的命令会排队，期间不会插入别人的命令。客户端里对应 `TxPipeline`。

```go
pipe := rdb.TxPipeline()
incr := pipe.Incr(ctx, "tx_visit")
pipe.Expire(ctx, "tx_visit", time.Hour)
_, err := pipe.Exec(ctx)
fmt.Println(incr.Val(), err)
```

等价于：

```text
MULTI
INCR tx_visit
EXPIRE tx_visit 3600
EXEC
```

回调形式是 `TxPipelined`，写法和 `Pipelined` 相同，只是外层多了事务包裹。事务里的某条命令语法错误会导致整个 `EXEC` 失败；运行期错误（比如对字符串做列表操作）则只影响那一条。

### Watch

只把命令放进 `MULTI` 还不够时，用 `WATCH` 盯住若干键。从 `WATCH` 到 `EXEC` 之间，如果键被别的客户端改过，这次 `EXEC` 会失败，调用方决定重试还是放弃。这是乐观锁：先假设没人冲突，冲突了再重做。

```go
key := "quota"
err = rdb.Watch(ctx, func(tx *redis.Tx) error {
	n, err := tx.Get(ctx, key).Int()
	if err != nil && err != redis.Nil {
		return err
	}
	_, err = tx.TxPipelined(ctx, func(pipe redis.Pipeliner) error {
		pipe.Set(ctx, key, n+1, 0)
		return nil
	})
	return err
}, key)
```

回调里的读必须走传入的 `tx`，写必须放进 `TxPipelined`。在回调外面读，再在里面写，监视就失去意义。

多个 goroutine 一起做“读出再加一”时，失败要重试，并给重试次数封顶：

```go
func bump(ctx context.Context, key string) error {
	const attempts = 80
	txf := func(tx *redis.Tx) error {
		n, err := tx.Get(ctx, key).Int()
		if err != nil && err != redis.Nil {
			return err
		}
		n++
		_, err = tx.TxPipelined(ctx, func(pipe redis.Pipeliner) error {
			pipe.Set(ctx, key, n, 0)
			return nil
		})
		return err
	}
	for left := attempts; left > 0; left-- {
		err := rdb.Watch(ctx, txf, key)
		if err == nil {
			return nil
		}
		if err != redis.TxFailedErr {
			return err
		}
	}
	return errors.New("bump gave up after retries")
}
```

`TxFailedErr` 才表示乐观锁被打破，可以再试。别的错误是网络或语法问题，重试没有意义。

## 使用时记住

- 缓存要设过期时间，否则内存只会增。
- 键的命名带上业务前缀，避免不同模块互相覆盖。
- Pipeline 省的是往返，Watch 保证的是“我读过的键没被改过”，两者解决的不是同一个问题。
