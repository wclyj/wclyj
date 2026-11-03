# Go 中的接口

## 接口的介绍

手机、相机、U 盘都能插进电脑的 USB 口。调用方不关心内部电路，只要求插头符合同一规格。

Go 的接口是抽象类型：它只列出方法签名，不写方法体，也不放字段。谁拥有这些方法，谁就满足这份规格。接口因此成为多态的落点，调用方依赖规格，不依赖具体结构体。

## Go接口的定义

接口是类型，内容是一组方法。方法都不能带实现。一个具体类型不必写“implements”，只要方法集合覆盖了接口，赋值就能通过。

```go
type 接口名 interface {
    方法一(参数) 返回值
    方法二(参数) 返回值
}
```

约定：

- 接口名用 `type` 声明。习惯在名字后加 `er`，例如 `Writer`、`Stringer`，让人一眼看出这是行为。
- 方法名和接口名都大写时，别的包才能看见。
- 参数名、返回值名可以省略，只留类型。

下面定义 `Plug`，让耳机和读卡器各自实现。

```go
type Plug interface {
    On()
    Off()
}
```

具体类型用结构体：

```go
type Headset struct{ Brand string }

func (h Headset) On()  { fmt.Println(h.Brand, "开始工作") }
func (h Headset) Off() { fmt.Println(h.Brand, "停止工作") }
```

方法名必须和接口完全一致，包括大小写。接口里写 `On`，结构体上却定义 `on`，编译器会认为没有实现。

```go
func main() {
    var p Plug = Headset{Brand: "北辰"}
    p.On()
    p.Off()
}
```

读卡器是另一个实现：

```go
type Reader struct{ Brand string }

func (r Reader) On()  { fmt.Println(r.Brand, "开始工作") }
func (r Reader) Off() { fmt.Println(r.Brand, "停止工作") }

func main() {
    var p Plug = Reader{Brand: "川页"}
    p.On()
    p.Off()
}
```

笔记本只依赖 `Plug`，因此两种设备都能传进去：

```go
type Laptop struct{}

func (Laptop) Boot(p Plug)   { p.On() }
func (Laptop) Halt(p Plug)   { p.Off() }
```

```go
func main() {
    ear := Headset{Brand: "北辰"}
    card := Reader{Brand: "川页"}
    box := Laptop{}
    box.Boot(ear)
    box.Boot(card)
    box.Halt(ear)
    box.Halt(card)
}
```

运行会依次打印四行：两种设备各启动一次、再各停止一次。

## 空接口（Object类型）

一个方法都不写的接口叫空接口。它对实现者没有任何要求，所以任何值都能放进去。工程里常用它表示“类型此刻未知”。

```go
type AnyBox interface{}

func main() {
    var box AnyBox
    box = "你好，接口"
    fmt.Println(box)
}
```

也可以直接写 `interface{}`（Go 1.18 起还可以写成 `any`）：

```go
var box any
box = 20
box = "hello"
box = true
```

作为参数时，函数就能接收任意实参：

```go
func dump(v any) {
    fmt.Println(v)
}
```

### map的值实现空接口

映射的值用空接口，一张表里就能混放字符串、数字和布尔。

```go
person := map[string]any{
    "name": "林夏",
    "age":  15,
    "work": true,
}
```

### slice切片实现空接口

切片同理：

```go
row := make([]any, 3)
row[0] = "林夏"
row[1] = 1
row[2] = true
```

放进去容易，取出来要当具体类型用时，必须先做类型断言。

## 类型断言

接口值里其实有两样东西：动态类型，以及该类型的动态值。

断言写成：

```text
x.(T)
```

- `x` 是接口
- `T` 是你猜测的具体类型

双返回值形式里，第二个布尔值表示猜得对不对，猜错也不会 panic。

```go
var box any = "132"
text, ok := box.(string)
if ok {
    fmt.Println("字符串:", text)
} else {
    fmt.Println("不是字符串")
}
```

也可以在一个函数里按类型分发：

```go
func classify(v any) {
    if _, ok := v.(string); ok {
        fmt.Println("这是字符串")
    } else if _, ok := v.(int); ok {
        fmt.Println("这是整数")
    } else {
        fmt.Println("其他类型")
    }
}
```

分支一多，改用 `switch`。`x.(type)` 只能出现在这种类型开关里。

```go
func classify(v any) {
    switch v.(type) {
    case int:
        fmt.Println("整数")
    case string:
        fmt.Println("字符串")
    case bool:
        fmt.Println("布尔")
    default:
        fmt.Println("其他类型")
    }
}
```

若还要拿到转换后的值，写成 `switch n := v.(type)`。

## 结构体接收者

### 值接收者

方法的接收者是值时，结构体值和结构体指针都能赋给接口。指针会自动解引用再调用。反过来，若方法接收者是指针，接口变量里就必须放指针（或能取地址的值，但放进接口后的动态类型仍要匹配）。

## 结构体实现多个接口

一个类型可以同时满足多份规格。下面 `Pet` 负责改名，`Named` 负责读名，`Dog` 两个都实现。

```go
type Pet interface {
    Rename(string)
}

type Named interface {
    Title() string
}

type Dog struct{ Name string }

func (d *Dog) Rename(name string) { d.Name = name }
func (d Dog) Title() string       { return d.Name }

func main() {
    pet := &Dog{Name: "小黑"}
    var a Pet = pet
    var b Named = pet
    a.Rename("小齐")
    fmt.Println(b.Title())
}
```

`Rename` 要改字段，所以用指针接收者。`Title` 只读取，值接收者即可。因为 `pet` 是指针，两份接口都能接住它。

## 接口嵌套

接口可以嵌入别的接口，从而合成更大的规格。

```go
type Pet interface{ Rename(string) }
type Named interface{ Title() string }

type Creature interface {
    Pet
    Named
}

type Dog struct{ Name string }

func (d *Dog) Rename(name string) { d.Name = name }
func (d Dog) Title() string       { return d.Name }

func main() {
    pet := &Dog{Name: "小黑"}
    var c Creature = pet
    c.Rename("小齐")
    fmt.Println(c.Title())
}
```

嵌入是展平方法集，不是继承字段。`Creature` 同时拥有 `Rename` 和 `Title`。

## Golang中空接口和类型断言

空接口装进映射之后，不能直接当切片去按下标取：

```go
info := map[string]any{
    "name":  "lin",
    "age":   10,
    "hobby": []string{"画画", "跑步"},
}
fmt.Println(info["name"])
fmt.Println(info["age"])
fmt.Println(info["hobby"])
// info["hobby"][0] 无法编译：空接口没有下标运算
```

先断言成 `[]string`，成功后再用下标：

```go
if hobbies, ok := info["hobby"].([]string); ok {
    fmt.Println(hobbies[0])
}
```

断言失败时 `ok` 为假，不要使用第一个返回值。
