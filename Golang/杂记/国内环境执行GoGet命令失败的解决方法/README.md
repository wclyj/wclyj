# 国内网络下 go get 拉依赖失败时怎么处理

## 前言

给项目加上 MySQL 驱动时，常会执行类似下面的拉取：

```bash
go get github.com/go-sql-driver/mysql
```

若模块代理仍指向默认的 `proxy.golang.org`，连接往往会在 TLS 握手阶段超时，报错里能看到 `dial tcp ...:443: connect` 以及超时说明。根因是默认代理从当前网络不可达，不是驱动仓库本身坏了。

## 解决方法

公共模块代理可以换成国内镜像。下面以 `goproxy.io` 为例，设好之后 `go` 命令会经镜像取模块。

### Go 1.13 及以上（优先用这种）

`go env -w` 会把配置写进本机，对之后所有终端生效：

```bash
go env -w GO111MODULE=on
go env -w GOPROXY="https://goproxy.io,direct"
go env -w GOPRIVATE=*.internal.example,example.com/secret
```

`GOPRIVATE` 可省略。写上之后，匹配到的私有模块不会走公共代理，也不会被校验和数据库拦截。多个模式用英文逗号隔开。

### Go 1.12 及更早

这个区间没有 `go env -w`，只能在当前 shell 导出变量。

**Bash（Linux 或 macOS）**

```shell
export GO111MODULE=on
export GOPROXY="https://goproxy.io,direct"
```

希望每次登录都生效，就把这两行放进 `~/.bashrc` 或 `~/.zshrc`。

**PowerShell（Windows）**

```shell
$env:GO111MODULE = "on"
$env:GOPROXY = "https://goproxy.io"
```

然后再执行 `go build` 或 `go test`，依赖会从 `goproxy.io` 下载。另一只常用镜像是 `goproxy.cn`，写法相同，把主机名换掉即可。
