# HTTP：服务端、客户端，以及并发回顾

## 和这一节一起出现的题目

后面几节会碰到请求上下文、给函数补测试，以及用 pprof 看运行时。pprof 是工具链自带的剖析器，能给出 CPU 采样、堆内存和一张可交互的调用图。本节先把并发原语和 HTTP 的收发补齐。

## 并发原语回顾

### 互斥锁

`sync.Mutex` 是结构体，属于值类型。作为字段传递时要传指针，否则加锁加在副本上，互相看不见。

```go
var gate sync.Mutex
gate.Lock()
gate.Unlock()
```

多个 goroutine 同时改同一块内存时才需要它。只读的数据不必加锁。`Unlock` 最好用 `defer` 紧跟在 `Lock` 后面，避免中途返回忘记释放。

### 读写锁

读远多于写时用 `sync.RWMutex`。读锁可以同时被多个读者持有，写锁独占，持有期间读者和写者都要等。

```go
var shelf sync.RWMutex
shelf.RLock()
shelf.RUnlock()

shelf.Lock()
shelf.Unlock()
```

读锁的释放方法是 `RUnlock`。误写成别的名字会编译失败。写锁不能用读锁的方法去解，两边必须成对。

### 等待组

`sync.WaitGroup` 同样是值，传参时用指针。它用来等一组 goroutine 全部返回，而不是猜测要睡多久。

```go
var crew sync.WaitGroup
crew.Add(1) // 启动几条就加几
crew.Done() // 任务结束时减一
crew.Wait() // 计数回到零才继续
```

`Add` 放在 `go` 之前。计数减成负数会直接 panic。

### 只执行一次

`sync.Once` 保证某个函数在整个进程里只跑一遍，适合懒加载配置或模板。

```go
var once sync.Once
once.Do(func() {
	// 没有参数也没有返回值；要夹带数据就用闭包
})
```

### 并发安全的映射

内置的 `map` 不能同时读写。`sync.Map` 专门应付这类场景，零值就能用，不必 `make`。

```go
var box sync.Map
box.Store("region", "cn-east")
val, ok := box.Load("region")
box.LoadOrStore("region", "cn-north")
box.Delete("region")
box.Range(func(k, v any) bool { return true })
```

它不支持下标赋值。`LoadOrStore` 会返回键上最终的值和“这次是否已经存在”，不是单纯的读取。键和值的类型是 `any`，取出来后要做类型断言。

### 原子加减

对整数这类基本类型，`sync/atomic` 提供不用自己加锁的加减。

```go
var hits int64 = 10
atomic.AddInt64(&hits, 1)
```

传入的必须是变量地址。原子操作保证这一次加法不会被别的加法撕开，但不保证周围其它语句也成为同一个临界区。

### 协议分层

网络编程建立在分层协议上。OSI 由上到下是应用层、表示层、会话层、传输层、网络层、数据链路层、物理层。TCP/IP 在工程里通常收成应用层、传输层、网络层、数据链路层和物理层。写 HTTP 时主要接触最上面的应用层，传输层的连接由 `net/http` 代为完成。

## HTTP 服务端

`net/http` 同时包含服务端和客户端。HTTP 是浏览器和站点之间最常用的传输约定，最初用来交换 HTML 页面，现在也承载 JSON 接口。

### 最小的处理函数

处理函数的签名固定为 `ResponseWriter` 加请求指针。`ListenAndServe` 的第二个参数传 `nil` 时，使用默认的多路复用器，也就是前面 `HandleFunc` 注册的那些路径。

```go
func plaza(w http.ResponseWriter, r *http.Request) {
	w.Write([]byte("plaza ok"))
}

func main() {
	http.HandleFunc("/plaza", plaza)
	http.ListenAndServe("127.0.0.1:9090", nil)
}
```

访问该路径会得到正文 `plaza ok`。

### 把文件内容写回

页面放在磁盘上时，读出来再写进响应。读失败要把错误告诉调用方，并且不要继续把空内容当成成功页面。

```go
func page(w http.ResponseWriter, r *http.Request) {
	raw, err := os.ReadFile("./home.html")
	if err != nil {
		http.Error(w, err.Error(), http.StatusInternalServerError)
		return
	}
	w.Write(raw)
}

func main() {
	http.HandleFunc("/home", page)
	http.HandleFunc("/about", page)
	http.ListenAndServe("127.0.0.1:9090", nil)
}
```

`ioutil.ReadFile` 从 Go 1.16 起被标记为过时，新代码用 `os.ReadFile`。

### 自定义 Server

需要限制读超时、写超时或头部大小时，不要用包级的 `ListenAndServe`，改成自己构造 `http.Server`。

```go
srv := &http.Server{
	Addr:           ":8080",
	Handler:        mux,
	ReadTimeout:    10 * time.Second,
	WriteTimeout:   10 * time.Second,
	MaxHeaderBytes: 1 << 20,
}
log.Fatal(srv.ListenAndServe())
```

没有读超时的服务端可能被慢客户端一直占着连接。

一次页面里，HTTP 负责请求和响应，HTML 描述结构，CSS 描述外观，JavaScript 在浏览器里响应用户操作。接口只返回 JSON 时，后三样可以不出现。

## HTTP 客户端

### 最直接的 Get

```go
func main() {
	res, err := http.Get("http://127.0.0.1:9090/lookup?name=lin&age=10")
	if err != nil {
		fmt.Println(err)
		return
	}
	defer res.Body.Close()
	raw, err := io.ReadAll(res.Body)
	if err != nil {
		fmt.Println(err)
		return
	}
	fmt.Println(string(raw))
}
```

不关闭 `Body` 会把连接留在池子里无法复用。查询参数出现在 URL 上，请求体一般是空的。

### 在服务端读查询参数

```go
func lookup(w http.ResponseWriter, r *http.Request) {
	fmt.Println(r.URL)
	q := r.URL.Query()
	fmt.Println("name", q.Get("name"))
	fmt.Println("age", q.Get("age"))
	fmt.Println("method", r.Method)
	raw, _ := io.ReadAll(r.Body)
	fmt.Println("body", string(raw))
}

func main() {
	http.HandleFunc("/lookup", lookup)
	http.ListenAndServe("127.0.0.1:9090", nil)
}
```

`Query` 返回的是 `url.Values`，同一个键出现多次时用 `Get` 只取第一个。中文参数也可以，客户端要先做百分号编码，`Get` 会负责编码：

```go
res, err := http.Get("http://127.0.0.1:9090/lookup?name=" + url.QueryEscape("林夏") + "&age=10")
```

更稳妥的做法是用 `url.Values` 组装，避免自己拼接时漏掉转义：

```go
q := url.Values{}
u, err := url.Parse("http://127.0.0.1:9090/lookup")
if err != nil {
	fmt.Println(err)
	return
}
q.Set("name", "林夏")
q.Set("age", "100")
u.RawQuery = q.Encode()
req, err := http.NewRequest(http.MethodGet, u.String(), nil)
if err != nil {
	fmt.Println(err)
	return
}
fmt.Println(req.URL)
```

### 自定义 Client

要改重定向策略或超时，就构造自己的 `http.Client`，不要依赖包级函数里那个没有超时的默认客户端。

```go
client := &http.Client{
	Timeout:       3 * time.Second,
	CheckRedirect: redirectPolicy,
}
res, err := client.Get("http://127.0.0.1:9090/lookup")

req, err := http.NewRequest(http.MethodGet, "http://127.0.0.1:9090/lookup", nil)
req.Header.Set("If-None-Match", `W/"plaza-9"`)
res, err = client.Do(req)
```

`Timeout` 覆盖从拨号到读完正文的整段时间。只给 `Do` 用的请求级超时，应放进请求的上下文。

### 自定义 Transport

代理、TLS 根证书、是否压缩、连接池，都在 `http.Transport` 上。

```go
tr := &http.Transport{
	TLSClientConfig:    &tls.Config{RootCAs: pool},
	DisableCompression: true,
}
client := &http.Client{Transport: tr}
res, err := client.Get("https://127.0.0.1:8443/health")
```

`Client` 与 `Transport` 可被多个 goroutine 共用。进程里创建一次再反复用，不要每个请求新造一个。

## 容易出错的地方

- 包级 `http.Get` 没有超时，对端不响应时调用会一直停着。
- 处理函数里忘记写状态码时，第一次 `Write` 会自动补 200。错误分支要先写状态码再写正文。
- 查询参数和路径参数不是一回事，路径参数需要自己的路由或第三方路由器。
