# Go的流程控制

流程控制决定语句按什么条件、重复多少次往下走。Go 里最常用的是 `if` 和 `for`。`switch` 与 `goto` 用来减少重复分支，属于补充手段。

## if else

条件后面习惯不写圆括号，写了也能编译，但和社区写法不一致。

```go
func main() {
	var score = 60
	if score == 60 {
		fmt.Println("equal 60")
	} else if score > 60 {
		fmt.Println("above 60")
	} else {
		fmt.Println("below 60")
	}
}
```

还可以在条件前插入一条短声明。这样声明的名字只在整个 `if` / `else` 里可见：

```go
if mark := 80; mark >= 60 {
	fmt.Println("passed")
}
```

`mark` 出了这条 `if` 就不能再使用。适合“先算出一个临时结果，再立刻判断”的场景。

## for 循环结构

Go 只有 `for` 这一种循环关键字，计数、条件循环和无限循环都由它承担。

基本形式：

```text
for 初始语句; 条件表达式; 后置语句 {
	循环体
}
```

条件为真就继续，为假就离开循环。打印 1 到 10：

```go
for i := 1; i <= 10; i++ {
	fmt.Printf("%v ", i)
}
```

语言里没有 `while`。只保留条件，或连条件也不写，就得到持续循环：

```go
for {
	// 循环体，需自行 break
}
```

离开循环可以用 `break`、`goto`、`return`，也可以在不可恢复的情况下 `panic`。

## for range（键值循环）

`for range` 用来遍历数组、切片、字符串、map 和 channel。返回值的个数随类型变化：

- 数组、切片、字符串：下标和元素
- map：键和值
- channel：只给出通道里取出的值

遍历字符串时，下标按字节计，值则是一个 rune：

```go
var text = "早安gopher"
for idx, ch := range text {
	fmt.Printf("%v - %c ", idx, ch)
}
```

遍历切片：

```go
var langs = []string{"ruby", "kotlin", "rust", "go"}
for idx, name := range langs {
	fmt.Printf("%v %s ", idx, name)
}
```

若只要值，下标写成 `_`。遍历 map 时不要依赖顺序，每次运行的次序可能不同。

## switch case

`switch` 适合把一个值同许多常量比较，比一长串 `if` 更清楚。

```go
suffix := ".x"
switch suffix {
case ".html":
	fmt.Println("markup")
case ".md":
	fmt.Println("markdown")
case ".go":
	fmt.Println("source")
default:
	fmt.Println("unknown suffix")
}
```

被判断的值也可以在 `switch` 后面用短声明临时创建，作用域限于这条 `switch`：

```go
switch suffix := ".x"; suffix {
case ".html":
	fmt.Println("markup")
case ".md":
	fmt.Println("markdown")
case ".go":
	fmt.Println("source")
default:
	fmt.Println("unknown suffix")
}
```

一个分支可以列出多个匹配值，用逗号隔开：

```go
suffix := ".csv"
switch suffix {
case ".html":
	fmt.Println("markup")
case ".csv", ".md":
	fmt.Println("text document")
case ".go":
	fmt.Println("source")
default:
	fmt.Println("unknown suffix")
}
```

匹配成功后默认就结束整条 `switch`，不必再写 `break`，也不会掉进下一个分支。这和 C 的默认穿透不同。

若确实要继续执行紧挨着的下一个分支，在当前分支末尾写 `fallthrough`。它只穿透一层；每一层都写，才会层层往下走。

```go
suffix := ".csv"
switch suffix {
case ".html":
	fmt.Println("markup")
	fallthrough
case ".csv", ".md":
	fmt.Println("text document")
	fallthrough
case ".go":
	fmt.Println("source")
	fallthrough
default:
	fmt.Println("unknown suffix")
}
```

上面从 `.csv` 命中的那一支开始，会接着打印 source 和 unknown。`fallthrough` 不能写在分支中间，必须是该分支的最后一条语句。

## break：跳出循环

`break` 的用途有三处：

- 在循环里立刻结束本层循环，接着执行循环后面的语句
- 在 `switch` 里提前结束当前分支（默认已经会结束，显式写出只为可读）
- 多层循环时，配合标签指定要跳出哪一层

```go
var i = 0
for {
	if i == 5 {
		fmt.Println("leave loop")
		break
	}
	i++
	fmt.Println(i)
}
```

循环会打印 1 到 5，然后在 `i == 5` 时离开。标签写法是把名字放在 `for` 前面，再 `break 名字`，用来从内层直接跳出外层。

## go：跳转到指定标签

`goto` 按标签做无条件跳转。它能快速离开一段嵌套，但跳得太远会让阅读顺序断裂，只适合少数“集中到同一段收尾”的情况。

```go
var age = 16
if age > 18 {
	fmt.Println("adult")
} else {
	goto tail
}

fmt.Println("line-a")
fmt.Println("line-b")
tail:
fmt.Println("line-c")
fmt.Println("line-d")
```

`age` 不大于 18 时，会跳过中间两行，直接从 `tail` 继续。标签与 `goto` 必须在同一函数内，也不能跳进内层代码块去绕过变量声明。
