# Golang的数据类型

## 概述

类型分成两类。基本类型表达单个值：整数、浮点、布尔、字符串。复合类型把多个值组织起来：数组、切片、结构体、函数、map、channel、接口等。

选类型时先看取值范围和是否需要变长，再决定用基本类型还是再包一层复合类型。

## 整型

有符号整数按宽度分为 `int8`、`int16`、`int32`、`int64`，无符号对应 `uint8` 到 `uint64`。宽度写在类型名里，跨平台行为一致。

不写宽度的 `int` 跟机器字长走：

- 32 位系统上，`int` 等同 `int32`
- 64 位系统上，`int` 等同 `int64`

因此把 `int` 当成“永远是 8 字节”会在 32 位环境里出错。需要稳定宽度时显式写出 `int32` 或 `int64`。

占用字节数可以用 `unsafe.Sizeof` 查看：

```go
var sample = 12
fmt.Println(unsafe.Sizeof(sample))
```

64 位机器上这里通常打印 8。

### 类型转换

不同宽度的整数不能直接相加，必须显式转换：

```go
var lo int16 = 10
var hi int32 = 12
sum := int32(lo) + hi
fmt.Println(sum)
```

从宽类型转到窄类型会截断，高位被丢掉，结果可能变成负数：

```go
var wide int16 = 130
fmt.Println(int8(wide)) // -126
```

130 的低 8 位是 `10000010`，按补码解释就是 -126。金额、计数这类字段不要靠截断“碰巧”得到结果。

### 数字字面量语法

从 Go 1.13 起，整数字面量可以带进制前缀：

```go
mask := 0b00101101 // 二进制
perm := 0o377      // 八进制
```

`0x` 仍表示十六进制。前缀让位运算和权限位更易读，不必先在心里换算成十进制。

### 进制转换

`fmt` 的动词决定按哪种进制打印，值本身不变：

```go
number := 23
fmt.Printf("%v\n", number)
fmt.Printf("%d\n", number)
fmt.Printf("%o\n", number)
fmt.Printf("%b\n", number)
fmt.Printf("%x\n", number)
```

依次是默认、十进制、八进制、二进制、十六进制。

## 浮点型

浮点只有 `float32` 和 `float64`，格式遵循 IEEE 754。`float32` 的数量级大约到 `3.4e38`，常量是 `math.MaxFloat32`；`float64` 大约到 `1.8e308`，常量是 `math.MaxFloat64`。未标明类型的小数点字面量默认是 `float64`。

```go
pi := math.Pi
fmt.Printf("%f\n", pi)
fmt.Printf("%.2f\n", pi)
```

`%f` 默认保留 6 位小数，`%.2f` 保留 2 位。这只影响打印，不改变变量里的值。

### Golang中精度丢失的问题

二进制浮点无法精确表示很多十进制小数，各语言都有这个问题。`0.1` 在二进制里是无限循环，放进定长尾数后只能取近似值。

```go
raw := 1129.6
fmt.Println(raw * 100)
```

结果往往不是干净的 `112960`，而会带一串 `999` 或多余的末位。比较浮点时不要用 `==`，金额计算不要用 `float64`。需要十进制精确运算时，改用定点小数库，例如 `shopspring/decimal`，用字符串构造小数，再做乘除。

## 布尔类型

布尔只有 `true` 和 `false`，零值是 `false`。它不能和整数互相转换，`if` 的条件必须是布尔表达式。

```go
flag := false
if flag {
	fmt.Println("yes")
} else {
	fmt.Println("no")
}
```

## 字符串类型

字符串是基本类型，底层按 UTF-8 存放字节。字面量用双引号，可以直接写非 ASCII 字符：

```go
en := "hello"
zh := "你好"
```

双引号字符串会处理转义。多行原文用反引号，里面的换行和反斜杠都按字面保留：

```go
block := `第一行
第二行`
fmt.Println(block)
```

字符串不可变。下标读到的是字节，不是“第几个字”。

### 字符串常见操作

- `len(str)`：字节数，不是字符数
- `+` 或 `fmt.Sprintf`：拼接；大量拼接用 `strings.Builder`
- `strings.Split`：按分隔符切开
- `strings.Contains`：是否包含子串
- `strings.HasPrefix`、`strings.HasSuffix`：前后缀
- `strings.Index`、`strings.LastIndex`：子串第一次和最后一次出现的字节下标，没有则返回 -1
- `strings.Join`：把字符串切片用分隔符拼回去

## byte 和 rune类型

字符串由字符组成，字符字面量用单引号。Go 里和“字符”相关的类型有两个：

- `byte`，即 `uint8`，适合 ASCII 的一个字节
- `rune`，即 `int32`，表示一个 Unicode 码点

中文、日文等多字节字符要用 `rune`。UTF-8 下，一个汉字通常占 3 个字节，一个英文字母占 1 个字节。所以 `len("你好")` 是 6，而按 rune 遍历只有 2 个元素。

```go
package main

import "fmt"

func main() {
	var ch byte = 'a'
	fmt.Println(ch)
	fmt.Printf("%c\n", ch)

	s := "你好 go"
	for i := 0; i < len(s); i++ {
		fmt.Printf("%v(%c)\t", s[i], s[i])
	}

	for index, v := range s {
		fmt.Println(index, v)
	}
}
```

第一段循环按字节走，汉字会被拆开，`%c` 打出来的不是完整汉字。`range` 按 rune 走，下标仍是字节位置，所以中文之间的下标会跳 3。

### 修改字符串

不能给 `s[0]` 赋值。要改内容，先复制成 `[]byte` 或 `[]rune`，改完再转回 `string`。两次转换都会分配新内存。

只改 ASCII 时用字节切片：

```go
word := "big"
buf := []byte(word)
buf[0] = 'p'
fmt.Println(string(buf))
```

含汉字时按下标改必须用 rune，否则会把一个汉字的某个字节改坏：

```go
word := "你好go"
rs := []rune(word)
rs[0] = '我'
fmt.Println(string(rs))
```

## 基本数据类型转换

### 数值类型转换

不同数值类型运算前先转换。窄类型转到宽类型不会丢数值：

```go
var aa int8 = 20
var bb int16 = 40
fmt.Println(int16(aa) + bb)

var cc int8 = 20
var dd float32 = 3.5
fmt.Println(float32(cc) + dd)
```

整数转浮点是为了保留小数结果；浮点转整数会丢掉小数部分。方向上优先往信息量更大的类型转。

### 转换成字符串类型

`fmt.Sprintf` 按动词把各种值格式化成字符串：

```go
var i int = 20
var f float64 = 12.456
var t bool = true
var b byte = 'a'

str1 := fmt.Sprintf("%d", i)
fmt.Printf("类型：%v-%T \n", str1, str1)

str2 := fmt.Sprintf("%f", f)
fmt.Printf("类型：%v-%T \n", str2, str2)

str3 := fmt.Sprintf("%t", t)
fmt.Printf("类型：%v-%T \n", str3, str3)

str4 := fmt.Sprintf("%c", b)
fmt.Printf("类型：%v-%T \n", str4, str4)
```

需要控制进制和小数位时，用 `strconv` 更明确：

```go
var num1 int64 = 20
s1 := strconv.FormatInt(num1, 10)
fmt.Printf("转换：%v - %T", s1, s1)

var num2 float64 = 3.1415926
s2 := strconv.FormatFloat(num2, 'f', -1, 64)
fmt.Printf("转换：%v-%T", s2, s2)
```

`FormatFloat` 的四个参数依次是：原值、格式（`'f'` 十进制小数，`'e'` 科学计数）、保留的小数位（`-1` 表示尽量精确且去掉多余的 0）、位数 32 或 64。

### 字符串转换成int 和 float类型

解析失败时第二个返回值是错误，不能忽略，否则会把 0 当成合法结果：

```go
text := "10"
n, err := strconv.ParseInt(text, 10, 64)
if err != nil {
	fmt.Println(err)
	return
}
fmt.Println(n)

text2 := "3.141592654"
f, err := strconv.ParseFloat(text2, 64)
if err != nil {
	fmt.Println(err)
	return
}
fmt.Println(f)
```

`ParseInt` 的第二个参数是进制，第三个是结果位数。`ParseFloat` 的第二个参数只能是 32 或 64，不是进制。
