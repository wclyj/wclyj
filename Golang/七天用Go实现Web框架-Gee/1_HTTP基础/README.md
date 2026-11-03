# 动手实现 Web 框架 Gee：第一天，认识 http.Handler

七日教程的第一篇。示例代码放在 day1 目录，笔记不附带仓库地址。

这一天做两件事：

- 看清 `net/http` 和 `http.Handler`
- 搭出 Gee 的骨架，实现大约五十行

## 标准库启动Web服务

`net/http` 已经封装了 HTTP 的基础能力，Gee 就建在它上面。先用标准库起一个服务。

```go
package main

import (
    "fmt"
    "log"
    "net/http"
)

func main() {
    http.HandleFunc("/", showPath)
    http.HandleFunc("/hi", showHeader)
    log.Fatal(http.ListenAndServe("127.0.0.1:9898", nil))
}

func showPath(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "path = %q\n", r.URL.Path)
}

func showHeader(w http.ResponseWriter, r *http.Request) {
    for key, vals := range r.Header {
        fmt.Fprintf(w, "Header[%q] = %q\n", key, vals)
    }
}
```

两条路由分别绑到两个函数。访问 `/` 会回显路径；访问 `/hi` 会把请求头逐条写回。

用 curl 检查：

```text
$ curl http://localhost:9999/
path = "/"
$ curl http://localhost:9999/hi
Header["User-Agent"] = ["curl/8.0.0"]
Header["Accept"] = ["*/*"]
```

`ListenAndServe` 的第一个参数是监听地址，`:9999` 表示所有网卡的 9999 端口。第二个参数是处理全部请求的 `Handler`。传入 `nil` 时，标准库使用默认的 `ServeMux`。这个参数就是后来把框架接进去的位置。

## 实现http.Handler接口

标准库里接口和启动函数的形状是：

```go
type Handler interface {
    ServeHTTP(ResponseWriter, *Request)
}

func ListenAndServe(addr string, h Handler) error
```

只要某个类型实现了 `ServeHTTP`，就可以交给 `ListenAndServe`。此后每个请求都会进这个方法。

```go
package main

import (
    "fmt"
    "log"
    "net/http"
)

type Engine struct{}

func (e *Engine) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    switch r.URL.Path {
    case "/":
        fmt.Fprintf(w, "path = %q\n", r.URL.Path)
    case "/hi":
        for key, vals := range r.Header {
            fmt.Fprintf(w, "Header[%q] = %q\n", key, vals)
        }
    default:
        fmt.Fprintf(w, "404 NOT FOUND: %s\n", r.URL)
    }
}

func main() {
    core := new(Engine)
    log.Fatal(http.ListenAndServe(":9999", core))
}
```

- `Engine` 目前是空结构体，只为了挂上方法。`Request` 里有路径、头部和正文；`ResponseWriter` 用来写状态码和正文。
- `main` 把 `core` 传给 `ListenAndServe`。在这之前，`HandleFunc` 只能一条一条登记路径。有了 `Engine`，所有请求先到同一入口，路由规则、日志和错误处理都可以集中写。
- curl 的结果与上一节相同，未知路径会得到 404。

## Gee框架的雏形

把上面的分支从 `switch` 换成一张路由表，目录先定成：

```text
gee/
  gee.go
  go.mod
main.go
go.mod
```

### go.mod

根模块通过 `replace` 指向本地的 `gee` 目录，这样还没发布的包也能被 import。

```text
module example

go 1.13

require gee v0.0.0

replace gee => ./gee
```

从 Go 1.11 起，相对路径的包不能直接写在 import 里，需要这种 replace。

### main.go

调用方式和 Gin 很像：`New` 得到引擎，`GET` 登记路由，`Run` 启动。

```go
package main

import (
    "fmt"
    "net/http"

    "gee"
)

func main() {
    r := gee.New()
    r.GET("/", func(w http.ResponseWriter, incoming *http.Request) {
        fmt.Fprintf(w, "path = %q\n", incoming.URL.Path)
    })
    r.GET("/hi", func(w http.ResponseWriter, incoming *http.Request) {
        for key, vals := range req.Header {
            fmt.Fprintf(w, "Header[%q] = %q\n", key, vals)
        }
    })
    r.Run(":9999")
}
```

这一版只有静态路径，还不认识 `/hi/:name`。动态段留到路由那一天。

### gee.go

```go
package gee

import (
    "fmt"
    "net/http"
)

type HandlerFunc func(w http.ResponseWriter, r *http.Request)

type Engine struct {
    routes map[string]HandlerFunc
}

func New() *Engine {
    return &Engine{routes: make(map[string]HandlerFunc)}
}

func (e *Engine) add(method, pattern string, h HandlerFunc) {
    e.routes[method+" "+pattern] = h
}

func (e *Engine) GET(pattern string, h HandlerFunc) {
    e.add(http.MethodGet, pattern, h)
}

func (e *Engine) POST(pattern string, h HandlerFunc) {
    e.add(http.MethodPost, pattern, h)
}

func (e *Engine) Run(addr string) error {
    return http.ListenAndServe(addr, e)
}

func (e *Engine) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    if h, ok := e.routes[r.Method+" "+r.URL.Path]; ok {
        h(w, r)
        return
    }
    fmt.Fprintf(w, "404 NOT FOUND: %s\n", r.URL)
}
```

- `HandlerFunc` 是给使用者写处理逻辑的函数类型。`routes` 的键把方法和路径拼在一起，例如 `GET /`、`GET /hi`、`POST /hi`。同一路径的不同方法可以指向不同函数。
- `GET` 只是把方法名填成 GET 再写入表。`Run` 把 `ListenAndServe` 包了一层，并把引擎自己当作 Handler。
- `ServeHTTP` 用请求方法和路径查表。查到就调用，查不到就写 404。

执行 `go run main.go` 后再 curl：

```text
$ curl http://localhost:9999/
path = "/"
$ curl http://localhost:9999/hi
Header["Accept"] = ["*/*"]
$ curl http://localhost:9999/missing
404 NOT FOUND: /missing
```

到这里，Gee 有了路由表、注册入口和启动函数。它还没有比 `net/http` 更强的匹配能力。动态路由和中间件会在后面几天加进来。
