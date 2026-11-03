# Go 的数组

## Array数组介绍

数组把同一类型的若干元素放在一起。元素可以是 `int`、`string` 这类内建类型，也可以是自己声明的结构。元素个数叫长度。在 Go 里长度属于类型的一部分，因此 `[3]int` 和 `[8]int` 不能互相赋值。元素在内存里连续摆放，按下标访问是一次地址偏移，所以很快。

切片可以变长，用起来更常见。要看懂切片的底层数组，得先把定长数组本身讲清楚。

## 数组定义

```text
var 名字 [长度]元素类型
```

```go
var ids [3]int
var titles [4]string
fmt.Printf("%T %T\n", ids, titles)

var scores [3]int
scores[0], scores[1], scores[2] = 7, 8, 9
fmt.Println(scores)

levels := [4]int{2, 4, 6, 8}
fmt.Println(levels)

auto := [...]int{5, 6}
fmt.Println(len(auto), auto)

sparse := [...]int{1: 10, 4: 40}
fmt.Println(sparse)
```

四种写法分别是：先声明再逐个赋值、字面量给全长度、`...` 让编译器数元素、只给指定下标（没写到的位置是零值）。

## 遍历数组

按下标走：

```go
sparse := [...]int{1: 10, 4: 40}
for i := 0; i < len(sparse); i++ {
    fmt.Print(sparse[i], " ")
}
```

用 `range` 同时拿到下标和值。不需要下标时用 `_` 丢掉。

```go
for _, n := range sparse {
    fmt.Print(n, " ")
}
```

## 数组的值类型

数组赋值和传参都会复制整份数据。改副本不会动原来那份。

```go
src := [...]int{4, 5, 6}
dst := src
dst[0] = 9
fmt.Println(src, dst) // [4 5 6] [9 5 6]
```

这和 Java 里数组变量保存引用不同。Go 的数组变量本身就装着全部元素。

## 切片定义(引用类型)

切片字面量不写长度。赋值时复制的是切片头（指针、长度、容量），底层数组仍共享，所以改一处两边都变。

```go
left := []int{4, 5, 6}
right := left
right[0] = 9
fmt.Println(left, right) // [9 5 6] [9 5 6]
```

## 二维数组

多维数组就是“元素仍是数组”。二维的声明写成：

```text
var 名字 [行数][列数]元素类型
```

```go
grid := [2][2]int{{1, 2}, {8, 9}}
fmt.Println(grid)
```

### 数组遍历

按下标扫每一行、每一列：

```go
grid := [2][2]int{{1, 2}, {8, 9}}
for r := 0; r < len(grid); r++ {
    for c := 0; c < len(grid[r]); c++ {
        fmt.Println(grid[r][c])
    }
}
```

两层 `range` 同样可以：

```go
for _, row := range grid {
    for _, cell := range row {
        fmt.Println(cell)
    }
}
```

### 类型推导

`...` 只能放在最外层。内层长度必须写死，编译器才知道每一行的类型。

```go
ok := [...][2]int{{1, 2}, {8, 9}}
fmt.Println(ok)
```

下面这样无法编译，因为列数无法从语法上单独推导：

```go
// bad := [2][...]int{{1, 2}, {8, 9}}
```

## 完整代码

```go
package main

import "fmt"

func main() {
    var ids [3]int
    var titles [4]string
    fmt.Printf("%T %T\n", ids, titles)

    var scores [3]int
    scores[0], scores[1], scores[2] = 7, 8, 9
    fmt.Println(scores)

    levels := [4]int{2, 4, 6, 8}
    auto := [...]int{5, 6}
    sparse := [...]int{1: 10, 4: 40}
    fmt.Println(levels, auto, sparse)

    for i := 0; i < len(sparse); i++ {
        fmt.Print(sparse[i], " ")
    }
    fmt.Println()
    for _, n := range sparse {
        fmt.Print(n, " ")
    }
    fmt.Println()

    base := 3
    copied := base
    base = 8
    fmt.Println(base, copied)

    src := [...]int{4, 5, 6}
    dst := src
    dst[0] = 9
    fmt.Println(src, dst)

    left := []int{4, 5, 6}
    right := left
    right[0] = 9
    fmt.Println(left, right)

    grid := [...][2]int{{1, 2}, {8, 9}}
    for r := 0; r < len(grid); r++ {
        for c := 0; c < len(grid[0]); c++ {
            fmt.Println(grid[r][c])
        }
    }
    for _, row := range grid {
        for _, cell := range row {
            fmt.Println(cell)
        }
    }
}
```
