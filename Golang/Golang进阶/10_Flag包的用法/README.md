# 用 flag 解析启动参数

标准库 `flag` 负责把进程启动时附带的选项变成带类型的值。写小工具时，不必再手工切分字符串、判断正负号或补默认值。

## os.Args

如果只想原样查看启动时传入了什么，直接读 `os.Args`。它是一个字符串切片，下标 0 固定是本次执行的程序名或路径，后面才是用户附加的内容。

```go
package main

import (
	"fmt"
	"os"
)

func main() {
	for i, token := range os.Args {
		fmt.Printf("slot[%d]=%s\n", i, token)
	}
}
```

编译并带上几个位置参数：

```bash
go build -o optscan .
./optscan east 8080 debug
```

终端会打印：

```bash
slot[0]=./optscan
slot[1]=east
slot[2]=8080
slot[3]=debug
```

这种方式没有类型、没有默认值，也不会自动生成帮助文本。参数一多，就该换成 `flag`。

## flag 的基本用法

下面只覆盖日常会用到的声明方式和取值时机。每个选项在声明时就要写清名字、默认值和一句说明。

### 导入

```go
import "flag"
```

### 能直接声明的类型

`flag` 内置了布尔、有符号与无符号整数、浮点、字符串以及 `time.Duration`。命令行上的文本会按下面的规则转换：

| 类型 | 可接受的写法 |
| --- | --- |
| 字符串 | 普通文本 |
| 整数 | 十进制、前导零的八进制、`0x` 开头的十六进制，允许负号 |
| 浮点 | 常规十进制小数 |
| 布尔 | `1`/`0`、`t`/`f`、`true`/`false`，大小写不限 |
| 时长 | 带单位的间隔，例如 `300ms`、`-1.5h`、`2h45m`。单位有 `ns`、`us`、`µs`、`ms`、`s`、`m`、`h` |

## 声明选项

常用声明方式有两套：一套返回指针，一套把结果写进已有变量。

### 返回指针的形式

调用形如 `flag.String(名字, 默认值, 说明)`，得到对应类型的指针。

```go
region := flag.String("region", "cn-east", "部署区域")
port := flag.Int("port", 8080, "监听端口")
dryRun := flag.Bool("dry-run", false, "只打印计划")
grace := flag.Duration("grace", 0, "停机宽限")
```

此时 `region`、`port`、`dryRun`、`grace` 都是指针，读取时要解引用。整数、无符号整数、浮点也有同名函数，只是返回类型不同。

### 写入已有变量的形式

调用形如 `flag.StringVar(指针, 名字, 默认值, 说明)`。变量的生命周期由调用方掌握，适合字段很多、需要集中打印的场景。

```go
var region string
var port int
var dryRun bool
var grace time.Duration

flag.StringVar(&region, "region", "cn-east", "部署区域")
flag.IntVar(&port, "port", 8080, "监听端口")
flag.BoolVar(&dryRun, "dry-run", false, "只打印计划")
flag.DurationVar(&grace, "grace", 0, "停机宽限")
```

两套函数可以混用，但同一个名字不能声明两次。

## 调用 Parse

声明本身不会读命令行。必须执行 `flag.Parse()`，默认值才会被实际传入的文本覆盖。

四种写法都能识别：

- `-region east`：一个短横线，空格分隔
- `--region east`：两个短横线，空格分隔
- `-region=east`：一个短横线，等号连接
- `--region=east`：两个短横线，等号连接

布尔选项只写名字时表示真，例如 `-dry-run`。若要显式写成假，必须使用等号：`-dry-run=false`。

解析遇到第一个非选项参数就停下来，剩下的内容留给程序自己处理。单独的 `-` 不算选项。单独的 `--` 是终止符，它后面的内容一律当作位置参数，即便以短横线开头。

## 解析之后还能问什么

```go
flag.Args()  // 选项后面剩下的位置参数，类型是 []string
flag.NArg()  // 位置参数的个数
flag.NFlag() // 命令行里真正出现过的选项个数，不含那些只使用默认值的项
```

`NFlag` 统计的是用户写出来的选项，不是程序里声明过的选项。

## 串起来的例子

### 程序

```go
func main() {
	var region string
	var port int
	var dryRun bool
	var grace time.Duration
	flag.StringVar(&region, "region", "cn-east", "部署区域")
	flag.IntVar(&port, "port", 8080, "监听端口")
	flag.BoolVar(&dryRun, "dry-run", false, "只打印计划")
	flag.DurationVar(&grace, "grace", 0, "停机宽限")

	flag.Parse()
	fmt.Println(region, port, dryRun, grace)
	fmt.Println(flag.Args())
	fmt.Println(flag.NArg())
	fmt.Println(flag.NFlag())
}
```

传入 `-help` 时，程序按声明顺序打印用法后退出，说明末尾会带上默认值。

### 混合短横线与等号

```bash
$ ./optscan -region ap-south --port 9090 -dry-run=false -grace=45m
ap-south 9090 false 45m0s
[]
0
4
```

四个选项都被识别，位置参数为空，所以 `NArg` 是 0，`NFlag` 是 4。时长会被规范化成带秒的形式。

### 只给位置参数

```bash
$ ./optscan alpha beta
cn-east 8080 false 0s
[alpha beta]
2
0
```

没有任何选项出现时，变量保持默认值，`Args` 收集到两个位置参数。

## 容易踩坑的地方

- 忘记调用 `Parse`，打印出来的永远是默认值。
- 把布尔值写成 `-dry-run false`。空格后面的 `false` 会被当成位置参数，开关本身仍是真。
- 在 `Parse` 之后再声明新选项，那些选项不会参与本次解析。
- 选项名区分大小写，`-Port` 和 `-port` 是两个不同的名字。
