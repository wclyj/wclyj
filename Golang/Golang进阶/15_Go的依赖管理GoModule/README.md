# 依赖管理：从 GOPATH 到 Go Module

## 老版本要手动打开

Go 1.11 和 1.12 里，模块模式默认不生效，需要自己设置环境变量才会按 `go.mod` 解析依赖。从 1.13 起，模块成为默认方式，新项目不必再把代码放进 `GOPATH/src`。

## 为什么早期的摆放方式不够用

最初所有第三方代码都堆在 `GOPATH` 下面。同一个导入路径只能留一份源码，两个项目若需要同一库的不同版本，后下载的会盖住先下载的，编译结果取决于机器上碰巧存在的那一份。

## godep 与 vendor

从 Go 1.5 起引入 vendor 机制：项目根目录下若有 `vendor`，工具链编译和测试时优先使用里面的包。`godep` 就是靠这个目录做锁定的第三方工具，社区里还有定位接近官方的 `dep`。两者都已被模块模式取代，这里只保留操作轮廓，方便读旧仓库。

### 安装

```bash
go get github.com/tools/godep
```

### 常用子命令

安装后直接输入 `godep` 可以列出子命令：

```bash
godep save      把依赖写进 Godeps.json，并复制源码
godep go        用已保存的依赖去执行 go 子命令
godep get       按锁定版本下载并安装
godep path      打印这套依赖使用的 GOPATH
godep restore   把锁定版本拉回 GOPATH
godep update    更新选定的包或 Go 版本
godep diff      对比当前依赖和上次保存的差异
godep version   查看工具自身版本
```

某个子命令的细节用 `godep help save` 这种形式查看。

### 一次保存做了什么

在能编译通过的项目里执行 `godep save`，会生成 `Godeps` 与 `vendor` 两个目录。`Godeps/Godeps.json` 记录导入路径和版本，`vendor` 里是对应源码。提交时这两个目录都要进版本库，别人拿到仓库才能复现构建。

### 查找顺序

Go 1.5 之后，解析某个导入路径时先看项目根下的 `vendor`，找不到再去 `GOPATH/src`。因此 vendor 里的副本会压过全局那一份。

### 日常流程

1. 先保证项目本身能编译。
2. 执行 `godep save`，冻结第三方的版本和源码。
3. 把 `Godeps` 与 `vendor` 一并提交。
4. 要换版本时，改 `Godeps.json` 里对应条目再重新保存，不要只改 vendor 里的文件。

## go module

`go module` 从 Go 1.11 开始由官方提供，Go 1.13 起成为默认的依赖管理方式。一个模块用根目录的 `go.mod` 声明自己的路径和依赖版本。

### GO111MODULE

这个环境变量决定是否启用模块，取值有三个，缺省是 `auto`。

1. `off`：关闭模块。编译只在 `GOPATH` 和 `vendor` 里找包。
2. `on`：强制开启。忽略 `GOPATH` 里的源码树，按 `go.mod` 下载依赖。
3. `auto`：项目不在 `GOPATH/src` 里面，且根目录存在 `go.mod` 时才开启。

设成 `on` 之后，项目可以放在任意目录。模块模式还会生成 `go.sum`，用来核对每个依赖的校验和。

### 代理

Go 1.11 之后可以用 `GOPROXY` 把下载转到可达的模块代理。Go 1.13 起默认指向官方公共代理，有的网络访问不到。值可以写成“代理地址”加逗号再加 `direct`：代理拉不到时，再直接向版本库地址请求。私有模块不要走公共代理。

### 子命令

```text
go mod download   把依赖拉到本地缓存，默认在 GOPATH/pkg/mod
go mod edit       用命令行增删 go.mod 里的条目
go mod graph      打印模块依赖图
go mod init       在当前目录创建 go.mod
go mod tidy       补上缺失的依赖，删掉没被引用的
go mod vendor     把依赖复制进 vendor
go mod verify     对照 go.sum 检查缓存是否被改过
go mod why        说明某个模块为什么会出现
```

### go.mod 长什么样

```text
module example.com/shop/catalog

go 1.21

require (
    github.com/gin-gonic/gin v1.9.1
    github.com/go-sql-driver/mysql v1.7.1
    github.com/jmoiron/sqlx v1.3.5
    google.golang.org/appengine v1.6.8 // indirect
)
```

- `module` 行是别人导入本项目时使用的前缀。
- `require` 列出直接或间接依赖和选定的版本。
- 行尾的 `indirect` 表示代码里没有直接导入它，它是被别的依赖带进来的。

#### 版本怎么写

模块使用语义化版本。`go get` 后面可以用 `@` 指定标签、分支或提交：

```text
go get example.com/gauge@v1.4.2
go get example.com/gauge@main
go get example.com/gauge@e3702bed2
```

`go.mod` 里因此会看到正式标签，也会看到“伪版本”：没有打标签的提交会被记成 `v0.0.0-日期-提交号`。`latest` 表示该模块当前最新的正式版本。

#### replace

有的路径在当前网络里克隆很慢，或者你想临时换成自己的分叉。可以在 `go.mod` 里把左侧模块替换成右侧：

```text
replace (
    golang.org/x/crypto v0.14.0 => example.com/mirror/crypto v0.14.0
    golang.org/x/text v0.13.0 => example.com/mirror/text v0.13.0
)
```

替换只影响本模块的构建，不会改写上游。本地未发布的库也可以指到相对目录，方便联调。

### go get 的升级粒度

在模块目录里执行 `go get` 会下载依赖，并可以指定升到哪里。

1. `go get -u` 升到最新的次版本或修订号。按 `x.y.z` 理解，`y` 是次版本，`z` 是修订号。
2. `go get -u=patch` 只动修订号，次版本保持不变。
3. `go get 包路径@版本` 精确锁定到给出的版本。

只想把 `go.mod` 里已经列出的依赖填进缓存时，用 `go mod download`，它不会改版本选择。

### 删掉代码后记得整理

从源码里去掉某个导入，`go.mod` 不会自动删掉对应行。执行 `go mod tidy`，它会对照真正的导入关系：缺的补上，没人用的去掉，并更新 `go.sum`。

### 用 go mod edit 改文件

手改 `go.mod` 之后可能缩进混乱，可以格式化：

```bash
go mod edit -fmt
```

追加一条依赖：

```bash
go mod edit -require=golang.org/x/text@v0.13.0
```

只从文件里删掉一条、先不下载：

```bash
go mod edit -droprequire=golang.org/x/text
```

更多开关用 `go help mod edit` 查看。改完后仍建议跑一次 `go mod tidy`，让文件和源码一致。

## 在项目里启用

### 已有项目

1. 在项目根目录执行 `go mod init 模块路径`，生成 `go.mod`。模块路径应与仓库地址一致，这样别人能按它找到你。
2. 执行 `go get` 或 `go mod tidy`，工具会扫描源码、记录依赖，并写出 `go.sum`。

### 新建项目

1. 建好目录后执行 `go mod init 模块路径`。
2. 要么手工补 `require`，要么直接在代码里导入，再执行 `go get` 让工具补全。

## 常见问题

- `go.sum` 冲突时不要随手删掉整份文件，先 `go mod tidy`，再看差异是不是预期的版本变化。
- 公司私有仓库要配置 `GOPRIVATE`，否则工具会把私有路径也拿去问公共代理。
- `replace` 指向本机目录时，别人的机器上没有那个目录，提交前要确认是否只用于本地调试。
