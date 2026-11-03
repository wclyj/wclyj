# Go 的函数

## 函数定义

函数是一段可反复调用的代码，用来完成一件明确的事。Go 支持具名函数、匿名函数和闭包。

声明以 `func` 开头：

```go
func 名字(参数列表) (返回值列表) {
    // 函数体
}
```

名字由字母、数字、下划线组成，不能以数字开头，同一包内不能重名。

```go
func addPair(a int, b int) int {
    return a + b
}

// 调用：addPair(3, 4)
```

参数个数不固定时，在最后一个参数的类型前写 `...`。函数内部它是一个切片。

```go
func addAll(nums ...int) int {
    total := 0
    for _, n := range nums {
        total += n
    }
    return total
}

// 调用：addAll(1, 1, 2, 3, 5)
```

返回值可以有多个，也可以提前命名。命名之后，函数体里直接给这些变量赋值，裸 `return` 会把它们按声明顺序交出去。

```go
func split(a int, b int) (sum int, diff int) {
    sum = a + b
    diff = a - b
    return
}
```

## 函数类型和变量

### 定义函数类型

用 `type` 给一种函数形状起名字：

```go
type binOp func(int, int) int
```

凡是“收两个 `int`、还一个 `int`”的函数，都可以赋给 `binOp` 变量。

```go
func addPair(a int, b int) int { return a + b }

func main() {
    var op binOp
    op = addPair
    fmt.Println(op(2, 9))
}
```

### 方法作为参数

函数可以当参数传递，调用方决定具体算法。

```go
func apply(a int, b int, fn func(int, int) int) int {
    return fn(a, b)
}
```

也可以按运算符返回一个匿名函数：

```go
func pick(sign string) binOp {
    switch sign {
    case "+":
        return func(x, y int) int { return x + y }
    case "-":
        return func(x, y int) int { return x - y }
    case "*":
        return func(x, y int) int { return x * y }
    case "/":
        return func(x, y int) int { return x / y }
    default:
        return nil
    }
}

func main() {
    plus := pick("+")
    fmt.Println(plus(2, 9))
}
```

## 匿名函数

Go 不允许在函数内部再声明具名函数，但可以写没有名字的函数。形式是：

```go
func (参数) (返回值) {
    // 函数体
}
```

它没有名字，要么存进变量，要么立刻调用：

```go
func main() {
    func() {
        fmt.Println("立刻执行的匿名函数")
    }()
}
```

## Golang中的闭包

### 全局变量和局部变量

全局变量一直占着内存，任何包内代码都能碰到它，容易互相踩踏。

局部变量随函数栈帧一起出现和消失，影响范围只在当前函数。

### 闭包

闭包让某个变量的寿命跟着返回的函数走，又不必把它暴露成全局量。可以把它看成“函数加上它捕获的外部环境”。

常见写法是外层函数返回内层函数，内层继续使用外层的局部变量。

注意：被捕获的变量不会马上回收，闭包太多会抬高内存。只在确实要记住状态时再用。

下面这个版本每次都读到同一个 `n`，因为内层没有修改它，所以三次都打印 `11`：

```go
func counter() func() int {
    n := 10
    return func() int {
        return n + 1
    }
}

func main() {
    step := counter()
    fmt.Println(step())
    fmt.Println(step())
    fmt.Println(step())
}
```

要让状态累加，就在内层改外层变量：

```go
func accumulator() func(delta int) int {
    n := 10
    return func(delta int) int {
        n += delta
        return n
    }
}

func main() {
    step := accumulator()
    fmt.Println(step(10)) // 20
    fmt.Println(step(10)) // 30
    fmt.Println(step(10)) // 40
}
```

每次调用 `accumulator()` 都会得到独立的 `n`。两个闭包互不影响。

## defer语句

`defer` 把后面的调用推迟到当前函数即将返回时。多个 `defer` 按注册的相反顺序执行：先注册的后跑。

```go
fmt.Println("A")
defer fmt.Println("B")
fmt.Println("C")
fmt.Println("D")
```

输出是 `A`、`C`、`D`、`B`。

再注册一个就会插到更前面：

```go
fmt.Println("A")
defer fmt.Println("B")
defer fmt.Println("C")
fmt.Println("D")
```

输出是 `A`、`D`、`C`、`B`。

若要推迟一整段逻辑，用匿名函数包起来。函数体内部仍按书写顺序执行：

```go
func main() {
    fmt.Println("进入")
    defer func() {
        fmt.Println("清理-1")
        fmt.Println("清理-2")
    }()
    fmt.Println("离开")
}
```

输出：

```text
进入
离开
清理-1
清理-2
```

### defer执行时机

`return` 在实现上不是一步做完的。先把返回值准备好，再执行 RET 真正离开函数。`defer` 卡在这两步中间。

```mermaid
flowchart LR
    setVal[写入返回值] --> runDefer[按逆序执行 defer]
    runDefer --> retInst[执行 RET]
```

因此，命名返回值可以在 `defer` 里被改掉；已经复制到临时变量的非命名返回值则改不到。

## panic/revocer处理异常

Go 没有 try/catch。严重错误用 `panic` 抛出，用 `recover` 接住。

- `panic` 可以写在任何位置
- `recover` 只在 `defer` 出去的函数里生效

```go
func step1() { fmt.Println("step1") }

func step2() { panic("主动中断") }

func main() {
    step1()
    step2()
    fmt.Println("收尾")
}
```

程序会在 `step2` 停下，打印 panic，`收尾` 不会出现。

把 `recover` 放进 `defer` 后，当前函数可以恢复，调用方继续往下走：

```go
func step2() {
    defer func() {
        if cause := recover(); cause != nil {
            fmt.Println("接住:", cause)
        }
    }()
    panic("主动中断")
}

func main() {
    step1()
    step2()
    fmt.Println("收尾")
}
```

## 异常运用场景

读文件失败时，业务层可以选择把普通 `error` 升级成 `panic`，再由统一的 `defer` 做告警。能返回 `error` 的路径仍优先用 `error`，不要把流程控制都写成 panic。

```go
func load(name string) error {
    if name == "app.go" {
        return nil
    }
    return errors.New("找不到源文件")
}

func run() {
    defer func() {
        if recover() != nil {
            fmt.Println("通知值班")
        }
    }()
    if err := load("missing.go"); err != nil {
        panic(err)
    }
}

func main() { run() }
```

## 内置函数

| 内置函数 | 作用 |
| --- | --- |
| close | 关闭通道，通知接收方不再有新数据 |
| len | 取字符串、数组、切片、映射、通道的长度 |
| new | 为值类型分配内存并返回指针，例如 `new(int)` |
| make | 为切片、映射、通道做初始化，返回的就是该引用本身 |
| append | 向切片追加元素，容量不够时会换一块新数组 |
| panic / recover | 抛出与接住运行时恐慌 |
