# context：取消、超时和请求范围的数据

`net/http` 的服务端为每个请求启动一个 goroutine。处理函数往往再开出访问数据库或 RPC 的 goroutine。这些执行流需要同一份请求数据：调用方身份、校验令牌、截止时间。请求一旦被取消或超时，整条链上的 goroutine 都应该尽快返回，把连接和内存还回去。

## 没有 context 时怎么收场

### 一个停不下来的循环

```go
var crew sync.WaitGroup

func loop() {
	for {
		fmt.Println("tick")
		time.Sleep(time.Second)
	}
	crew.Done()
}

func main() {
	crew.Add(1)
	go loop()
	crew.Wait()
	fmt.Println("done")
}
```

`Done` 写在死循环后面，永远执行不到。主流程也没有办法通知它退出。

### 用全局布尔值

```go
var stop bool

func loop() {
	for {
		fmt.Println("tick")
		time.Sleep(time.Second)
		if stop {
			break
		}
	}
	crew.Done()
}

func main() {
	crew.Add(1)
	go loop()
	time.Sleep(3 * time.Second)
	stop = true
	crew.Wait()
	fmt.Println("done")
}
```

这样能停，但有两个问题。布尔值跨包时没有归属，谁都能改。`loop` 再启动下一层 goroutine 时，那个新任务看不见这份约定，除非再增加更多全局变量。对布尔的读写在多核上也不构成同步，编译器可能把它优化成只读一次。

### 用通道发停止信号

```go
func loop(stop <-chan struct{}) {
	for {
		fmt.Println("tick")
		time.Sleep(time.Second)
		select {
		case <-stop:
			crew.Done()
			return
		default:
		}
	}
}

func main() {
	stop := make(chan struct{})
	crew.Add(1)
	go loop(stop)
	time.Sleep(3 * time.Second)
	close(stop)
	crew.Wait()
	fmt.Println("done")
}
```

关闭通道能让所有接收方同时醒来，比往通道里发送一次更适合广播。缺点是每条调用链都要额外传这个通道，超时、截止时间和附带数据还得再配别的参数，约定不容易统一。

### 换成标准库的上下文

```go
func loop(ctx context.Context) {
	defer crew.Done()
	for {
		fmt.Println("tick")
		time.Sleep(time.Second)
		select {
		case <-ctx.Done():
			return
		default:
		}
	}
}

func main() {
	ctx, cancel := context.WithCancel(context.Background())
	crew.Add(1)
	go loop(ctx)
	time.Sleep(3 * time.Second)
	cancel()
	crew.Wait()
	fmt.Println("done")
}
```

`cancel` 被调用后，`ctx.Done()` 这个通道会被关闭。子任务再开子任务时，把同一个 `ctx` 传下去即可，不必再造新的停止通道：

```go
func loop(ctx context.Context) {
	crew.Add(1)
	go helper(ctx)
	defer crew.Done()
	for {
		fmt.Println("parent")
		time.Sleep(time.Second)
		if ctx.Err() != nil {
			return
		}
	}
}

func helper(ctx context.Context) {
	defer crew.Done()
	for {
		fmt.Println("helper")
		time.Sleep(time.Second)
		select {
		case <-ctx.Done():
			return
		default:
		}
	}
}
```

父任务返回前要把子任务也算进 `WaitGroup`，否则主流程可能先结束。

## 这个类型是做什么的

Go 1.7 把 `context` 放进标准库。它用来在一次请求涉及的多个 goroutine 之间传递取消信号、截止时间和少量请求域数据。这些数据可能要穿过好几层 API。

进入服务端的请求应当创建上下文；向外发出的调用应当接受上下文。函数链要么原样传递，要么用 `WithCancel`、`WithDeadline`、`WithTimeout`、`WithValue` 派生。父上下文被取消时，从它派生出去的全部都会被取消。

## 接口上的四个方法

```go
type Context interface {
	Deadline() (deadline time.Time, ok bool)
	Done() <-chan struct{}
	Err() error
	Value(key any) any
}
```

- `Deadline` 返回这份上下文应当结束的时刻。第二个结果为假表示没有设截止时间。
- `Done` 返回一个通道，工作完成或上下文被取消时该通道关闭。多次调用返回同一个通道。
- `Err` 只在 `Done` 关闭之后才非空。主动取消得到 `context.Canceled`，超时得到 `context.DeadlineExceeded`。
- `Value` 按键取出请求域数据。同一个上下文、同一个键，多次读取结果相同。它不是用来传可选的函数参数的。

### 两个根

`Background` 和 `TODO` 都返回空的根上下文：不能取消、没有截止时间、也不携带值。

`Background` 用在 `main`、初始化和测试里，作为整棵树的根。`TODO` 用在暂时不知道该接哪一个父上下文的地方，方便以后搜索出来补上。业务函数里不要凭空再造一个 `Background`，否则上游的取消传不进来。

## 四个派生函数

### WithCancel

```go
func WithCancel(parent Context) (ctx Context, cancel CancelFunc)
```

返回的上下文拥有自己的 `Done` 通道。调用 `cancel`，或父上下文的 `Done` 关闭，都会关闭它。谁先发生听谁的。`cancel` 会释放关联的资源，操作结束后应该立刻调用，通常用 `defer`。

```go
func count(ctx context.Context) <-chan int {
	out := make(chan int)
	go func() {
		defer close(out)
		n := 1
		for {
			select {
			case <-ctx.Done():
				return
			case out <- n:
				n++
			}
		}
	}()
	return out
}

func main() {
	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()
	for n := range count(ctx) {
		fmt.Println(n)
		if n == 5 {
			break
		}
	}
}
```

`count` 里的 goroutine 只有在 `Done` 或发送成功时才有出口。调用方拿到 5 之后跳出循环，必须 `cancel`，否则那个 goroutine 堵在发送上，通道也没有人接收。

### WithDeadline

```go
func WithDeadline(parent Context, d time.Time) (Context, CancelFunc)
```

派生上下文的截止时间不会晚于 `d`。如果父上下文的截止更早，结果等价于父上下文，不会把期限放宽。到期、调用 `cancel`、父级关闭，三者任一发生都会结束这份上下文。

```go
func main() {
	deadline := time.Now().Add(50 * time.Millisecond)
	ctx, cancel := context.WithDeadline(context.Background(), deadline)
	defer cancel()
	select {
	case <-time.After(time.Second):
		fmt.Println("waited too long")
	case <-ctx.Done():
		fmt.Println(ctx.Err())
	}
}
```

50 毫秒先到，所以会打印超时错误，而不是 `waited too long`。即便知道它会自己到期，仍然要 `defer cancel()`。不然在到期前返回时，父级和定时器会比必要的时间活得更久。

### WithTimeout

```go
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)
```

它就是 `WithDeadline(parent, time.Now().Add(timeout))`。数据库和出站请求最常用这一份。

```go
func dial(ctx context.Context) {
	defer crew.Done()
	for {
		fmt.Println("dialing warehouse")
		time.Sleep(10 * time.Millisecond)
		select {
		case <-ctx.Done():
			fmt.Println("dial stopped")
			return
		default:
		}
	}
}

func main() {
	ctx, cancel := context.WithTimeout(context.Background(), 50*time.Millisecond)
	defer cancel()
	crew.Add(1)
	go dial(ctx)
	crew.Wait()
	fmt.Println("done")
}
```

超时到了不必再手动 `cancel` 才能让 `Done` 关闭，但 `defer cancel()` 仍要保留，用来提前结束或释放定时器。上面的 `Wait` 会在 `dial` 看到超时后返回，不会再额外睡五秒。

### WithValue

```go
func WithValue(parent Context, key, val any) Context
```

返回的派生上下文在父级的数据之外，多了一对键值。只放请求范围里、跨 API 必须带着走的数据，例如追踪号。可选的函数参数应该放在普通参数里，放进上下文会让签名失去说明力。

键必须可比较，并且不要用 `string` 或其它内置类型，否则不同包可能碰巧用了同一个字符串。习惯是在本包定义一个未导出的空结构体类型，或导出指针类型的键。空结构体作为键还可以避免装箱时的额外分配。

```go
type traceKey struct{}

func dial(ctx context.Context) {
	defer crew.Done()
	code, _ := ctx.Value(traceKey{}).(string)
	for {
		fmt.Printf("dialing trace=%s\n", code)
		time.Sleep(10 * time.Millisecond)
		select {
		case <-ctx.Done():
			return
		default:
		}
	}
}

func main() {
	ctx, cancel := context.WithTimeout(context.Background(), 50*time.Millisecond)
	defer cancel()
	ctx = context.WithValue(ctx, traceKey{}, "trc-88421")
	crew.Add(1)
	go dial(ctx)
	crew.Wait()
}
```

类型断言失败说明上游没放这个键，应当走默认值或记一条日志，不要直接当字符串用。`WithValue` 不会取消父级，超时仍然由前面的 `WithTimeout` 负责。

## 使用约定

- 上下文作为显式参数传递，不要塞进结构体再偷偷带着走。放进结构体的例外是那些生命周期确实等于该对象的东西，例如 `http.Request`。
- 参数列表里它排在第一位，名字用 `ctx`。
- 不要传 nil。暂时没有合适的父级就传 `context.TODO()`。
- `Value` 只放请求域数据，不放可选参数。
- 上下文可以被多个 goroutine 同时读，不需要再加锁。取消函数不要复制之后还指望各自独立，一份 `cancel` 会取消整棵派生树。

## 客户端超时：服务端偶尔很慢

### 服务端

用随机数模拟一半请求要睡很久：

```go
func home(w http.ResponseWriter, r *http.Request) {
	if rand.Intn(2) == 0 {
		time.Sleep(10 * time.Second)
		fmt.Fprint(w, "slow")
		return
	}
	fmt.Fprint(w, "fast")
}

func main() {
	http.HandleFunc("/", home)
	if err := http.ListenAndServe(":8000", nil); err != nil {
		panic(err)
	}
}
```

### 客户端

给请求套上 100 毫秒的上下文。`NewRequestWithContext` 会在超时或取消时中止这次 `Do`。

```go
type outcome struct {
	res *http.Response
	err error
}

func call(ctx context.Context) {
	client := &http.Client{
		Transport: &http.Transport{DisableKeepAlives: true},
	}
	req, err := http.NewRequestWithContext(ctx, http.MethodGet, "http://127.0.0.1:8000/", nil)
	if err != nil {
		fmt.Println("build request:", err)
		return
	}
	ch := make(chan outcome, 1)
	go func() {
		res, err := client.Do(req)
		ch <- outcome{res: res, err: err}
	}()
	select {
	case <-ctx.Done():
		fmt.Println("api timeout:", ctx.Err())
	case got := <-ch:
		if got.err != nil {
			fmt.Println("api error:", got.err)
			return
		}
		defer got.res.Body.Close()
		body, _ := io.ReadAll(got.res.Body)
		fmt.Println("api body:", string(body))
	}
}

func main() {
	ctx, cancel := context.WithTimeout(context.Background(), 100*time.Millisecond)
	defer cancel()
	call(ctx)
}
```

请求不频繁时可以关掉长连接，避免为了复用而留着空闲连接。请求很密时应当把 `Client` 做成全局变量并保留连接池。通道缓冲为 1，是为了超时之后那个 goroutine 仍能把结果送出去，不会堵死。更完整的做法是在超时分支里继续等待该 goroutine 结束再释放客户端，否则进程退出前可能还有请求挂着。

`http.Client.Timeout` 也能限制整次调用，但它不能和上游已经存在的取消信号合并。请求已经带着上下文时，优先把截止时间放在上下文里。
