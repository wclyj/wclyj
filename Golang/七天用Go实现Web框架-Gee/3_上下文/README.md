# Go语言动手写Web框架 - Gee第二天 上下文Context

这一天做两件事：把路由表从引擎里拆出去，再设计一个跟单次请求绑定的上下文。框架代码大约一百多行，其中大部分是新的 `Context`。

- 路由单独成文件，后面才能加动态路径，而不必改引擎的每个方法
- `Context` 同时握住请求和响应，并提供 JSON、HTML 等快捷写法
- 处理函数的参数从原来的两个标准库对象，收成一个上下文

## 使用效果

调用方看到的是引擎上的 `GET` / `POST`，处理函数只跟 `*Context` 打交道：

```go
func main() {
	app := gee.New()
	app.GET("/", func(c *gee.Context) {
		c.HTML(http.StatusOK, "<h1>Hi</h1>")
	})
	app.GET("/hi", func(c *gee.Context) {
		c.String(http.StatusOK, "hi %s, path %s\n", c.Query("name"), c.Path)
	})
	app.POST("/signin", func(c *gee.Context) {
		c.JSON(http.StatusOK, gee.H{
			"user": c.PostForm("user"),
			"code": c.PostForm("code"),
		})
	})
	app.Run(":7777")
}
```

- 查询字符串和表单都从上下文读，不必自己拆 `URL` 和 `Request.Form`
- 写响应时指定状态码和正文类型，头由对应方法补上

## 设计Context

### 必要性

1. Web 处理说到底是读 `*http.Request`、写 `http.ResponseWriter`。这两个类型的方法很碎：状态码、内容类型、正文要分三次设置。每次返回 JSON 都手写编码器，既重复也容易漏掉头部。框架应当把常见响应收成一次调用。

直接写标准库时，返回 JSON 要自己设头、写状态、再编码：

```go
payload := map[string]interface{}{
	"user": "lee",
	"code": "7788",
}
w.Header().Set("Content-Type", "application/json")
w.WriteHeader(http.StatusOK)
enc := json.NewEncoder(w)
if err := enc.Encode(payload); err != nil {
	http.Error(w, err.Error(), 500)
}
```

收进上下文之后，同样的事情变成：

```go
c.JSON(http.StatusOK, gee.H{
	"user": c.PostForm("user"),
	"code": c.PostForm("code"),
})
```

2. 少写几行只是原因之一。动态路由里的路径参数、中间件放进去的用户信息，都得有地方放。这些数据跟某一次请求共存亡，不该做成全局变量。上下文在请求进来时创建，写完响应后就可以丢掉。路由函数和以后的中间件都只接收这一个参数，查找参数、设置状态、提前中断都走它。

3. 标准库里也有 `context.Context`，用来传递取消信号和截止时间。这里的 `gee.Context` 更接近一次 HTTP 交换的工具箱：里面是请求、响应和框架自己的附加数据。名字相同，职责不要混。一个请求一个实例，中间件和处理函数看到的是同一个指针。

### 具体实现

`H` 是 `map[string]interface{}` 的别名，构造 JSON 时少写一长串类型。`Context` 保存原始对象，并缓存方法与路径，避免处理函数反复从 `Request` 里取。

```go
type H map[string]interface{}

type Context struct {
	Writer http.ResponseWriter
	Req    *http.Request
	Path   string
	Method string
	StatusCode int
}

func newContext(w http.ResponseWriter, req *http.Request) *Context {
	ctx := &Context{Writer: w, Req: req}
	ctx.Path = req.URL.Path
	ctx.Method = req.Method
	return ctx
}

func (c *Context) PostForm(key string) string {
	return c.Req.FormValue(key)
}

func (c *Context) Query(key string) string {
	return c.Req.URL.Query().Get(key)
}

func (c *Context) Status(code int) {
	c.StatusCode = code
	c.Writer.WriteHeader(code)
}

func (c *Context) SetHeader(key, value string) {
	c.Writer.Header().Set(key, value)
}

func (c *Context) String(code int, format string, args ...interface{}) {
	c.SetHeader("Content-Type", "text/plain; charset=utf-8")
	c.Status(code)
	c.Writer.Write([]byte(fmt.Sprintf(format, args...)))
}

func (c *Context) JSON(code int, obj interface{}) {
	c.SetHeader("Content-Type", "application/json; charset=utf-8")
	c.Status(code)
	if err := json.NewEncoder(c.Writer).Encode(obj); err != nil {
		http.Error(c.Writer, err.Error(), http.StatusInternalServerError)
	}
}

func (c *Context) Data(code int, body []byte) {
	c.Status(code)
	c.Writer.Write(body)
}

func (c *Context) HTML(code int, html string) {
	c.SetHeader("Content-Type", "text/html; charset=utf-8")
	c.Status(code)
	c.Writer.Write([]byte(html))
}
```

要点：

- `Query` 只看 URL 上的查询串，`PostForm` 走 `FormValue`，会在需要时解析表单
- `Status` 既记下状态码，又真正写到响应里；状态码必须在写正文之前发出
- `String`、`JSON`、`HTML` 各自带上合适的内容类型，调用方不用再记头部名字
- `JSON` 编码失败时返回 500。注意此时可能已经写过 200 的状态行，严谨的实现应先编码到缓冲区，成功后再写头部和正文

## 路由(Router)

路由从引擎中拆出。表的键是“方法-路径”，值是处理函数。这一天还没有通配符，键必须完全一致。

```go
type router struct {
	handlers map[string]HandlerFunc
}

func newRouter() *router {
	table := make(map[string]HandlerFunc)
	return &router{handlers: table}
}

func (r *router) addRoute(method, pattern string, handler HandlerFunc) {
	log.Printf("route %s %s", method, pattern)
	r.handlers[method+"-"+pattern] = handler
}

func (r *router) handle(c *Context) {
	key := c.Method + "-" + c.Path
	if handler, ok := r.handlers[key]; ok {
		handler(c)
		return
	}
	c.String(http.StatusNotFound, "missing: %s\n", c.Path)
}
```

和前一天相比，`handle` 的参数从响应和请求换成了上下文。找不到路由时，也通过上下文写 404，错误页面和正常页面走同一套输出。

## 框架入口

引擎只保留一个路由表，并实现 `ServeHTTP`，从而能交给 `http.ListenAndServe`。

```go
type HandlerFunc func(*Context)

type Engine struct {
	router *router
}

func New() *Engine {
	return &Engine{router: newRouter()}
}

func (e *Engine) addRoute(method, pattern string, handler HandlerFunc) {
	e.router.addRoute(method, pattern, handler)
}

func (e *Engine) GET(pattern string, handler HandlerFunc) {
	e.addRoute(http.MethodGet, pattern, handler)
}

func (e *Engine) POST(pattern string, handler HandlerFunc) {
	e.addRoute(http.MethodPost, pattern, handler)
}

func (e *Engine) Run(addr string) error {
	return http.ListenAndServe(addr, e)
}

func (e *Engine) ServeHTTP(w http.ResponseWriter, req *http.Request) {
	c := newContext(w, req)
	e.router.handle(c)
}
```

`ServeHTTP` 是标准库回调的入口。每个请求先包成上下文，再交给路由。后面给上下文加参数表、中间件链时，改的是 `newContext` 和 `handle`，`GET` / `POST` 的签名可以保持不变。

用 `go run` 启动后，可以用 curl 核对四种结果：

```text
curl -i http://localhost:7777/
curl "http://localhost:7777/hi?name=lee"
curl "http://localhost:7777/signin" -X POST -d "user=lee&code=7788"
curl "http://localhost:7777/nope"
```

期望依次是：HTML 正文、带名字的纯文本、包含 `user` 与 `code` 的 JSON，以及 404 文本。若 POST 读到空字符串，检查请求头是否为表单类型，以及字段名是否和 `PostForm` 的键一致。
