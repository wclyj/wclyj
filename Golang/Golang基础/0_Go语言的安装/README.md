# Windows下Go语言的安装

## 前言

准备把 Go 用到后续工作里，第一步是把工具链装好。编辑器、模块缓存和编译命令都依赖这一步，环境没配对时，后面的代码即使正确也跑不起来。

## 下载Go

官方下载页有时访问不稳定，可以从国内镜像站挑选与系统匹配的安装包。Windows 选带 `.msi` 的安装程序，版本号按团队要求锁定，不必盲目追最新。

安装包运行后会进入图形向导。安装目录要记下来，例如 `D:\Go`，因为下一步要把它写进环境变量。向导结束时，开始菜单里应能看到 Go 的相关项，安装目录下应有 `bin`、`src`、`pkg`。

## 配置环境变量

Windows 查找命令时会扫描 `Path` 里的目录。把 Go 的可执行文件目录加进去之后，任意文件夹都能敲 `go`。需要关心的变量有三项：

| 环境变量 | 说明 |
| -------- | ---- |
| GOROOT   | SDK 解压或安装后的根目录 |
| Path     | 追加该根目录下的 bin |
| GOPATH   | 工作区与模块缓存所在位置 |

打开“系统属性 → 高级 → 环境变量”。在用户变量或系统变量中新建 `GOROOT`，值填安装根目录。再编辑 `Path`，追加 `%GOROOT%\bin`。`GOPATH` 可指向一个专门的工作目录，例如 `D:\gows`；使用 Go Modules 之后，源码不必放在 `GOPATH/src` 下，但这个变量仍然影响默认缓存位置。

保存后新开一个命令提示符，执行：

```bash
go version
```

若打印出 `go version go1.x.x windows/amd64` 一类文字，说明 `Path` 已生效。若提示找不到命令，多半是终端还在使用旧环境，关掉窗口重开即可。

## 下载Jetbrain下的GoLang

命令行能编译之后，再用 JetBrains 的 GoLand 写代码。从 JetBrains 官网下载对应系统的安装包，安装后首次启动会要求选择主题和插件，Go 插件一般已内置。

进入 File → Settings → Go → GOROOT，指向刚才的安装根目录。同一页或 GOPATH 设置里，把项目工作目录也填上。索引完成后，标准库的 `fmt`、`net/http` 应能跳转定义。

## hello world

环境就绪后写一个最小程序，确认工具链和编辑器连在一起：

```go
package main

import "fmt"

func main() {
	fmt.Println("hi, gopher")
}
```

这几行各自的含义：

- 源文件后缀是 `.go`
- `package main` 表示当前文件属于 main 包；每个 Go 文件都必须属于某个包
- `import "fmt"` 引入格式化包，才能调用其中的输出函数
- `func main()` 是可执行程序的入口

在 GoLand 里点击运行，控制台应出现 `hi, gopher`。若报 GOROOT 无效，回到设置核对路径，而不是改源码。

## 编译和执行

也可以不依赖 IDE，在源文件所在目录用命令行编译：

```bash
# 编译 hello.go，Windows 上得到 hello.exe
go build hello.go
# 运行生成的可执行文件
hello.exe
```

开发时也可以一步到位：

```bash
go run hello.go
```

`go run` 会先编译再执行，所以第一次会比直接双击 exe 慢一点。交付到没有 Go 的机器时，应先 `go build` 得到二进制，再把二进制拷过去运行。
