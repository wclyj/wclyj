# 使用goctl快速创建项目

## 前言

架构上的限流、熔断和缓存先放一边，落地时最先碰到的是项目骨架：目录怎么分、配置文件放哪、路由和逻辑写在哪个包。goctl 就是 go-zero 配套的代码生成器，用来把这些骨架一次铺好，再往生成的 logic 里填业务。

## 下载Go代码生成工具

模块模式下依赖默认从公共代理拉取。网络不稳定时，先把模块开关和代理写进环境：

```bash
go env -w GO111MODULE=on
go env -w GOPROXY=https://goproxy.cn,direct
```

然后安装命令行工具。较新的写法是：

```bash
go install github.com/zeromicro/go-zero/tools/goctl@latest
```

安装后执行 `goctl -v`。若提示找不到命令，把 `$(go env GOPATH)/bin` 加进 `PATH`。

## 编写业务代码

生成器只负责骨架，业务仍要手写，落点有三处：

- `.api` 文件描述对外路由、请求方法和请求体字段，是生成代码的输入
- `servicecontext.go` 集中存放依赖，例如数据库连接和缓存客户端，再传给各个 logic
- 每种方法对应一个 logic 文件，查询、创建、更新、删除的处理写在这里，不要改生成器标记过的路由注册代码，否则下次生成会被覆盖

## goctl用途

- 用 api 描述把请求和响应定下来
- 按同一份描述生成 Go 服务端，也能生成 Java、TypeScript 等调用端代码
- 按表结构生成 MySQL 的增删改查，并带上缓存读写
- 对 MongoDB 同样能生成带缓存的访问代码

## goctl使用说明

### 快速生成服务

单体风格的 HTTP 服务和对外的 RPC 服务命令不同：

```bash
# HTTP API，一个进程里挂多条路由
goctl api new demo

# RPC，生成 proto、server 与客户端桩代码
goctl rpc new demo
```

`demo` 换成项目名。命令会在当前目录下建同名文件夹。

### goctl参数说明

已有 `.api` 文件时，可以指定语言和输出目录，而不必从 `new` 开始：

```bash
goctl api go -api user/user.api -dir ./src
```

`go` 也可以换成 `java`、`ts` 等生成调用端。例如：

```bash
goctl api java -api order.api -dir client
goctl api ts -api order.api -dir web
```

`-api` 是描述文件，`-dir` 是输出目录。描述变更后重跑同一命令。

## 快速生成api服务

工具就绪后生成一个最小 HTTP 服务并跑起来：

```bash
goctl api new greet
cd greet
go run greet.go -f etc/greet-api.yaml
```

生成结果里会有 `etc` 配置、入口 `greet.go`，以及 `internal` 下的 handler、logic、svc 和 types。默认端口写在 yaml 中，常见是 8888。访问：

```text
http://localhost:8888/from/you
```

具体路径以生成出的 `.api` 为准，有的模板会带分组前缀。返回 JSON 且进程没有报错，就说明配置被读到、路由已注册。接下来改 `.api` 增加字段，再执行 `goctl api go`，最后只在 logic 里补判断和存储访问。
