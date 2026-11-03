# Go语言中的变量和常量

## Go语言中变量的声明

变量名由字母、数字和下划线拼成，第一个字符不能是数字。关键字和预定义标识符也不能拿来当名字。

名字必须先声明再使用，同一个作用域里不能把同一个名字声明两次。声明了却从不读取，编译也会失败。只声明、不赋初值时，打印出来的是该类型的零值，而不是随机垃圾。

## 如何定义变量

### 方式1：

类型可以省略，由右边的字面量推断：

```go
var city = "hangzhou"
```

### 方式2：带类型

左边写出类型，右边的值必须能赋给它：

```go
var city string = "hangzhou"
```

### 方式3：类型推导方式定义变量

函数体内可以用 `:=` 同时声明并赋值，比 `var` 更短。

这种短声明只能出现在函数内部，包一级的全局变量仍然要写 `var`。

```go
变量名 := 表达式
```

### 方式4：声明多个变量

类型相同、一次声明多个名字：

```go
var 左值, 右值 类型
```

类型不同时，用括号把每行拆开：

```go
var (
	标题 类型
	分数 类型
)
```

例如两个字符串先声明再赋值：

```go
var left, right string
left = "east"
right = "west"
fmt.Printf(left)
fmt.Printf(right)
```

### 总结

几种写法可以放在同一个 `main` 里对照：

```go
package main

import "fmt"

func main() {
	fmt.Println("start")
	fmt.Print("X", "Y", "Z")
	fmt.Println()
	var n = 7
	fmt.Printf("%d", n)

	var nick = "ada1"
	var nick2 string = "ada2"
	nick3 := "ada3"

	fmt.Println(nick)
	fmt.Println(nick2)
	fmt.Println(nick3)
	fmt.Printf("n1=%v n2=%v n3=%v \n", nick, nick2, nick3)
}
```

## 如何定义常量

常量在编译期就定死，适合圆周率、状态码这类运行中不会改的量。写法和变量相近，把 `var` 换成 `const`，而且定义时必须给出值，不能先声明再赋值。

```go
// 常量允许暂时不被引用，编译器不会因此报错
const ratio = 2.5

const (
	North = "N"
	South = "S"
)

// 同一组里如果某一行不写值，就沿用上一行的字面量
const (
	North = "N"
	East
	West
)
```

## Const常量结合iota的使用

`iota` 是常量组里的行计数器，只能写在 `const` 表达式中。

每遇到一个新的 `const`，计数从 0 重新开始；组内每向下增加一行，计数加一。可以把它理解成这一组声明的行号。

```go
const solo = iota // solo = 0
const (
	row0 = iota // row0 = 0
	row1        // row1 = 1
	row2        // row2 = 2
)
```

空白标识符 `_` 可以吃掉某一行，让后面的常量跳号：

```go
const (
	row0 = iota // row0 = 0
	_
	row2        // row2 = 2
)
```
